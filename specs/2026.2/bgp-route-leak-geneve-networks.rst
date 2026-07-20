=============================================
BGP Route Leaking for Geneve Tenant Networks
=============================================

https://bugs.launchpad.net/neutron/+bug/2161353

This spec extends the OVN BGP integration [1]_ to allow operators to leak
routes from private geneve tenant networks into the underlay BGP fabric. It
introduces a ``leak_routes`` attribute on the subnet resource that causes
the network's host routes to be advertised via BGP and installs a static
route on ``bgp-lr-main`` to direct ingress traffic through the Neutron
router.


Problem Description
===================

The OVN BGP integration introduced in the 2025.2 cycle provides dynamic
advertisement of floating IPs and router gateway addresses to the underlay
fabric via BGP. However, there is no mechanism to advertise tenant network
prefixes directly.

Without route leaking, reaching tenant workloads from the external fabric
requires NAT via floating IPs. Route leaking eliminates this NAT overhead by
making tenant network addresses directly routable from the underlay. Both
ingress and egress traffic traverse the external flat network to reach the
Neutron router, following the same path used by floating IPs.

The ovn-bgp-agent project provides similar functionality through its
``expose_tenant_networks`` configuration option, which advertises all tenant
network ports' IPs hosted on the given compute node to the fabric. This spec
brings an equivalent capability into the native OVN BGP integration in Neutron,
with the key difference that route leaking is controlled per-subnet via an API
attribute rather than being a per-node configuration toggle affecting all tenant
networks on that node.


Proposed Change
===============

This spec proposes a new attribute on the subnet resource that enables route
leaking from a geneve tenant network into the underlay BGP routing domain
managed by the BGP service plugin.

API Extension
-------------

A new boolean attribute ``leak_routes`` is introduced on the subnet resource:

* Default value: ``False``
* Supported operations: Update only (not available at create time)
* Constraint: only allowed on subnets belonging to geneve networks
* Constraint: the subnet must be attached to a router with an external gateway
* Requires: the ``ovn-bgp`` service plugin must be enabled
* Authorization: admin-only by default (policy-controlled)

CLI examples:

.. code-block:: text

    # Enable route leaking on a subnet (subnet must already be attached
    # to a router with an external gateway)
    openstack subnet set --leak-routes my-subnet

    # Disable route leaking
    openstack subnet set --no-leak-routes my-subnet

    # Show the attribute
    openstack subnet show my-subnet -c leak_routes

REST API:

.. code-block:: text

    PUT /v2.0/subnets/{subnet_id}
    {
        "subnet": {
            "leak_routes": true
        }
    }

    PUT /v2.0/subnets/{subnet_id}
    {
        "subnet": {
            "leak_routes": false
        }
    }

    GET /v2.0/subnets/{subnet_id}
    {
        "subnet": {
            "name": "my-subnet",
            "leak_routes": true,
            ...
        }
    }


OVN Topology Change
-------------------

When ``leak_routes`` is set to ``True`` on a subnet, the BGP service plugin
performs two actions:

1. **LRP/LSP attachment (advertisement trigger)**: Create a logical router
   port (LRP) on ``bgp-lr-main`` connected to the network's logical switch.
   The LRP uses **IPv6 unnumbered** addressing (link-local only) so that no
   IP address is consumed from any tenant subnet allocation pool. A
   corresponding logical switch port (LSP) of type ``router`` is created on
   the tenant LS. This connection's sole purpose is to make OVN treat the LS
   as "connected" to ``bgp-lr-main``, triggering host route advertisement.
   No data traffic flows through this link.

   The LRP/LSP pair is created when the **first** subnet on a network is
   leaked and removed when the **last** leaked subnet on that network has
   leaking disabled or is detached from its router.

2. **Static route (data-plane path)**: A static route is added to
   ``bgp-lr-main`` for the leaked subnet's CIDR (e.g., ``10.0.0.0/24``),
   with the next-hop set to the Neutron router's LRP IP address on the
   provider (external) switch. This directs ingress traffic from the underlay
   through the provider network into the Neutron router, which then forwards
   it to the tenant LS via its internal interface.

Because ``bgp-lr-main`` already has
``options:dynamic-routing-redistribute=connected-as-host,nat`` set at the
router level, no additional per-LRP redistribute option is needed. OVN
automatically redistributes host routes (/32 for IPv4, /128 for IPv6) for
ports on the newly attached logical switch, populates the Advertised_Route
table in the Southbound DB, and the per-chassis FRR instances advertise them
to the BGP peers.

Traffic Flows
~~~~~~~~~~~~~

**Ingress** (external fabric → tenant VM):

1. External peer sends traffic to a tenant VM IP (e.g., 10.0.0.5).
2. The host route for 10.0.0.5/32 was advertised by ``bgp-lr-main``.
3. ``bgp-lr-main`` matches the static route for 10.0.0.0/24, forwards to the
   Neutron router's external LRP IP via the provider LS.
4. The Neutron router routes the packet to the tenant LS via its internal
   interface (the subnet gateway).
5. The packet reaches the VM.

**Egress** (tenant VM → external fabric):

Traffic egresses through the Neutron router's default gateway as usual (the
same path as without route leaking). The VM uses the Neutron router as its
default gateway.

When ``leak_routes`` is set back to ``False``, the static route is removed.
If this was the last leaked subnet on the network, the LRP and LSP are also
removed, and OVN withdraws all host routes from the BGP VRF.


Lifecycle Interactions
----------------------

**Router interface removal**: If the router interface for a leaked subnet is
removed (e.g., ``openstack router remove subnet``), the BGP plugin
automatically tears down the OVN state (removes the static route and, if
applicable, the LRP/LSP) and resets ``leak_routes`` to ``False``. The subnet
retains its other attributes.

**External gateway removal**: If the router's external gateway is removed
while a subnet is leaked, no automatic teardown occurs. The static route on
``bgp-lr-main`` remains but traffic will blackhole because the next-hop is
unreachable. When the external gateway is restored, the data path recovers
automatically. This is considered an operational misconfiguration.


Limitations and Constraints
===========================

Multi-Subnet Route Advertisement
--------------------------------

When the LRP/LSP connects ``bgp-lr-main`` to the tenant LS, OVN advertises
host routes for **all** ports on that logical switch — regardless of which
subnet they belong to. However, only subnets with ``leak_routes=True`` will
have a static route on ``bgp-lr-main`` and a working ingress data-plane path.

Traffic destined to IPs on non-leaked subnets will be advertised (attracting
packets from the external fabric) but will be **blackholed** at
``bgp-lr-main`` because no static route exists for those prefixes.

Operators should therefore leak **all** subnets on a network or be aware of
this asymmetry. A future extension may enforce this constraint automatically.

Overlapping Subnet Ranges
-------------------------

In standard Neutron, multiple isolated tenant networks may use the same subnet
CIDR (e.g., 10.0.0.0/24) because they exist in separate routing domains.
However, once routes are leaked into the shared underlay BGP domain via
``bgp-lr-main``, all leaked host routes coexist in the same routing table.
If two networks share the same subnet range, ports from both networks could
have identical IP addresses, resulting in conflicting host routes and ambiguous
routing. Overlapping CIDRs must therefore be prevented.

**Validation rules:**

1. When ``leak_routes`` is set to ``True`` on a subnet, its CIDR is checked
   for overlap against all other subnets that currently have
   ``leak_routes=True``. If any overlap is detected, the operation is rejected
   with a ``Conflict`` (HTTP 409) error.

2. Overlap is defined as any intersection between CIDR ranges, not just exact
   match. For example, 10.0.0.0/24 overlaps with 10.0.0.0/16.

This is a **global constraint** — it spans all projects because the underlay
routing domain is shared. The error message must clearly indicate which
existing subnet conflicts.

Geneve-Only Restriction
-----------------------

The ``leak_routes`` attribute is only valid on subnets belonging to networks
with ``provider:network_type = geneve``. Attempting to set it on subnets of
other network types is rejected with a ``BadRequest`` (HTTP 400) error.

Router Attachment Required
--------------------------

Setting ``leak_routes=True`` requires that the subnet is already attached to
a router with an external gateway. If the subnet is not attached to any router
or the router has no external gateway, the operation is rejected with a
``BadRequest`` (HTTP 400) error.

The attribute is not available at subnet creation time because the subnet
cannot be attached to a router at creation.

BGP Service Plugin Dependency
-----------------------------

Using ``leak_routes=True`` requires the ``ovn-bgp`` service plugin to be
loaded. If the plugin is not enabled, the API extension is not advertised and
the attribute is not available.


Data Model Changes
==================

A new table tracks which subnets have route leaking enabled:

.. code-block:: python

    class SubnetBGPLeakRoutes(model_base.BASEV2):
        __tablename__ = 'subnet_bgp_leak_routes'

        subnet_id = sa.Column(
            sa.String(36),
            sa.ForeignKey('subnets.id', ondelete='CASCADE'),
            primary_key=True,
        )

The primary key is ``subnet_id``. The ``network_id`` is derivable via a join
to the ``subnets`` table when needed (e.g., to determine if other subnets on
the same network are still leaked). The ``CASCADE`` ensures cleanup if the
subnet is deleted.

An Alembic migration creates the ``subnet_bgp_leak_routes`` table.


Implementation
==============

The implementation consists of three parts:

1. **API extension**: A new extension (``ovn-bgp``) that declares the
   ``leak_routes`` attribute on the subnet resource. The extension is
   advertised only when the ``ovn-bgp`` service plugin is loaded.

2. **BGP service plugin handlers**: The BGP service plugin subscribes to
   subnet and router-interface events:

   * ``SUBNET PRECOMMIT_UPDATE``: when ``leak_routes`` transitions to
     ``True``, validate that the subnet is attached to a router with an
     external gateway, the network is geneve, and no subnet CIDR overlap
     exists. Write the ``bgp_leak_routes`` DB entry. When transitioning to
     ``False``, remove the DB entry.
   * ``SUBNET AFTER_UPDATE``: issue OVN commands. On ``False → True``: create
     the LRP/LSP (if first leak on this network) and add the static route on
     ``bgp-lr-main``. On ``True → False``: remove the static route; if last
     leaked subnet on the network, remove the LRP/LSP.
   * ``ROUTER_INTERFACE BEFORE_DELETE``: if the subnet being detached has
     ``leak_routes=True``, remove the DB entry within the same transaction.
   * ``ROUTER_INTERFACE AFTER_DELETE``: tear down OVN state (static route,
     and LRP/LSP if last leaked subnet on network).

3. **OpenStackClient support**: Add ``--leak-routes`` / ``--no-leak-routes``
   options to ``openstack subnet set`` command.


Assignee(s)
-----------

* Jakub Libosvar <jlibosva@redhat.com>


Testing
=======

Tempest Tests
-------------

* End-to-end: enable ``leak_routes`` on a subnet, create a port, verify host
  route appears in the BGP VRF on the chassis
* End-to-end: disable ``leak_routes``, verify host routes are withdrawn
* End-to-end: add/remove ports on a leaked network, verify dynamic host route
  advertisement/withdrawal
* End-to-end: remove router interface on a leaked subnet, verify automatic
  teardown
* Negative: attempt ``--leak-routes`` on subnet not attached to a router
* Negative: attempt ``--leak-routes`` on subnet attached to router without
  external gateway
* Negative: attempt overlapping subnets across two leaked subnets


References
==========

.. [1] Core OVN BGP Integration spec,
   https://specs.openstack.org/openstack/neutron-specs/specs/2025.2/ovn-bgp-integration.html
