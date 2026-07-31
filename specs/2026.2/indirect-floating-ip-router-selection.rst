..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Indirect Floating IP Router Selection
======================================

https://bugs.launchpad.net/neutron/+bug/2072505

When a floating IP (FIP) is associated with a port, Neutron selects a
**hosting router** that will own the DNAT/SNAT rules for that FIP. Today,
``get_router_for_floatingip()`` in ``neutron/db/l3_db.py`` only accepts a
router that has **both** a router interface on the port's fixed-IP subnet
**and** an external gateway on the FIP's external network. If no such router
exists, association fails with HTTP 404 and the message *"External network …
is not reachable from subnet …"*.

This check protects the common single-router model but blocks deliberate
multi-router topologies where tenant subnets reach a gateway router across
one or more transit networks (or via static routes through a gateway VM).
The datapath for such topologies can work but today's control-plane validation
prevents it.

This spec proposes relaxing that validation by allowing callers to name the
hosting router explicitly, while documenting datapath prerequisites,
backend scope, and lifecycle limitations.


Problem Description
===================

Current behavior
----------------

On FIP create or update (when ``port_id`` / fixed IP is set), the L3 plugin
calls ``get_router_for_floatingip()`` which:

#. Queries routers with a router interface on the fixed IP's subnet **and**
   an external gateway on the FIP's external network.
#. Prefers the router whose interface IP is the subnet gateway IP; otherwise
   returns the first match.
#. If no match is found, raises ``ExternalGatewayForFloatingIPNotFound``
   (HTTP 404).

This is intentional for the typical case where one Neutron router connects
a tenant subnet directly to an external network. It does **not** consider:

* Static routes on a router that reach a subnet via a nexthop on another
  connected subnet ([LP#2072505]_).
* Transit topologies where the VM subnet reaches a gateway router across an
  intermediate router and transit network.

The selected ``router_id`` is stored on the FIP and used by L3 backends
(for example, OVN programs ``dnat_and_snat`` NAT rules on that logical
router). The current validation therefore conflates **"which router should
host NAT?"** with **"is the subnet directly connected to that router?"** —
which is stricter than what the datapath requires.


Use cases
---------

Gateway VM with static routes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Reported in [LP#2072505]_ and originally in [LP#1616317]_.

An operator deploys:

* An external network and an "outer" internal network connected by a
  Neutron router.
* An "inner" internal network **not** connected to that Neutron router.
* A Nova instance ("gateway") with interfaces on both inner and outer
  networks, forwarding between them.
* A "worker" VM on the inner network.

A static route on the Neutron router sends traffic for the inner subnet to
the gateway VM as nexthop. Outbound connectivity from the worker works, but
associating a FIP to the worker fails because no router has an interface on
the inner subnet **and** a gateway on the external network.

Two-hop transit topology
~~~~~~~~~~~~~~~~~~~~~~~~

A common case is a tenant VM on a private subnet that reaches the public
external network through **two Neutron routers** and a **transit network**.
No single router has both a router interface on the tenant subnet and an
external gateway on the public network, so FIP association fails today even
when routing and NAT are configured correctly on the datapath.

Example topology:

::

   public
     |
   ext-router
     |
   transit-net (192.168.200.0/24)
     |
   internal-router
     |
   tenant-net (192.168.210.0/24)
     |
   tenant-vm  <-- FIP from public

* **internal-router** has a router interface on ``tenant-net``.
* **ext-router** has an external gateway on ``public`` and a router interface
  (or port) on ``transit-net``.
* **internal-router** is also connected to ``transit-net``; routes on both
  routers forward traffic between ``tenant-net`` and ``ext-router``.

Traffic flow (simplified):

``tenant-vm → internal-router → transit-net → ext-router → public``

For egress SNAT from ``tenant-net`` through ``ext-router`` without a direct
router interface on that subnet, ML2/OVN deployments typically also require
``ovn_router_indirect_snat=True`` on the gateway router ([927560]_). The
missing piece is control-plane selection of ``ext-router`` as the FIP hosting
router when associating a public FIP with ``tenant-vm``.

Longer chains (three or more routers) follow the same pattern; this spec uses
the two-hop case as the reference topology for tests and documentation.


Proposed Change
===============

Overview
--------

Introduce a new API extension ``floating-ip-router-writable`` that makes
``floatingip.router_id`` writable on FIP create and update. When the
caller supplies ``router_id``, Neutron:

#. Validates that the router exists and is visible to the caller (existing
   RBAC / project scope).
#. Validates that the router has an external gateway on the FIP's external
   network.
#. **Skips** the direct subnet-adjacency check in
   ``get_router_for_floatingip()``.
#. Stores the supplied ``router_id`` on the FIP for backend programming.

When ``router_id`` is **not** supplied, behavior remains unchanged: the
existing direct-connectivity check applies.


REST API Impact
---------------

The ``floating-ip-router-writable`` extension makes ``router_id`` writable on
the existing ``floatingip`` resource. Today ``router_id`` is returned by the
API but computed by the server and cannot be set by the client.

**Changed attribute on ``floatingip``:**

.. list-table::
   :header-rows: 1

   * - Attribute
     - Type
     - CRUD
     - Description
   * - ``router_id``
     - string (UUID)
     - CRU
     - Hosting router for the FIP. When omitted on create/update, Neutron
       computes ``router_id`` using the existing direct-connectivity logic.

* ``POST /v2.0/floatingips``

  Create a floating IP and optionally name the hosting router explicitly.

  Request::

    {
      "floatingip": {
        "floating_network_id": "<external-network-uuid>",
        "port_id": "<vm-port-uuid>",
        "router_id": "<gateway-router-uuid>"
      }
    }

  ``router_id`` is optional. When provided, Neutron validates the router and
  stores it on the FIP instead of running the direct subnet-adjacency check.

* ``PUT /v2.0/floatingips/{floatingip_id}``

  Update a floating IP association and optionally set or change the hosting
  router.

  Request::

    {
      "floatingip": {
        "port_id": "<vm-port-uuid>",
        "router_id": "<gateway-router-uuid>"
      }
    }

  ``router_id`` may be supplied together with ``port_id`` when associating a
  FIP with a port, or updated on an already-associated FIP.

When ``router_id`` is provided:

* The router must exist and be visible to the caller.
* The router must have an external gateway on the same network as
  ``floating_network_id`` (on create) or the FIP's current external network
  (on update).
* The implementation **does not** verify L3 reachability between the fixed IP
  subnet and the named router.

**Error responses:**

* ``404 Not Found`` if the named router does not exist or is not visible, or
  if ``router_id`` was omitted and no directly-connected router exists.
* ``409 Conflict`` if the router has no external gateway on the FIP's
  external network.


Datapath prerequisites (operator responsibility)
------------------------------------------------

Selecting a hosting router is **necessary but not sufficient** for a working
FIP. The operator must ensure:

* **Routes** on every router in the path forward traffic correctly between
  the VM subnet and the gateway router (static routes, default routes, or
  transit subnets as designed).
* **ML2/OVN:** ``ovn_router_indirect_snat=True`` on gateway routers that
  must SNAT traffic from subnets not directly connected ([927560]_) - to
  make sure this is true, Neutron `ovn-router` service plugin may load new api
  extension `floating-ip-router-writable` only when ``ovn_router_indirect_snat``
  is set to ``True``. We may also consider introducing new config option
  named ``ovn_router_indirect_nat`` which would be similar to the already
  deprecated ``ovn_router_indirect_snat`` but would  cover both SNAT and
  DNAT (FIP) cases.
* **OVN version:** OVN build including the [FDP-744]_ fix where catch-all
  SNAT and FIP NAT must coexist.
* **Gateway VM forwarding** (LP#2072505 use case): the gateway instance must
  actually forward; Neutron cannot verify this.


Backend scope
-------------

ML2/OVN (primary, in scope for initial implementation)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This is the primary deployment target. OVN FIP NAT rules key on the fixed IP
and VM port chassis residence, not on direct subnet attachment to the
hosting router.

For the **gateway VM use case** (where a forwarding VM bridges the inner and
outer networks instead of a Neutron router), DNAT is performed in the
external router pipeline. Traffic that reaches the external router will be
DNATed correctly. However, **distributed FIP is not possible** in this
configuration: the gateway VM's traffic (without a distributed FIP of its
own) traverses the ext-router pipeline on the gateway chassis node, so the
``is_chassis_resident`` check used for distributed FIP will fail. Indirect
FIPs in the gateway-VM topology therefore operate in **centralised mode
only**.
This limitation will have to be well documented.

ML2/OVS without DVR
~~~~~~~~~~~~~~~~~~~

May work when routes are correct and FIP traffic traverses the central
router namespace. Behavior should be documented; full test coverage is
**not** required for the initial merge if the feature is gated (see
Configuration).

ML2/OVS with DVR
~~~~~~~~~~~~~~~~

**Known limitation.** DVR FIP traffic uses a FIP namespace on the compute
node and may bypass intermediate routers or gateway VMs in the path
([MEETING]_, [LP#1571676]_). The initial implementation should document
that indirect FIPs are unsupported with DVR. Additionally, api extension
``floating-ip-router-writable`` should not be loaded when DVR is enabled.

Resolving DVR datapath gaps is **out of scope** for this spec unless a
follow-up is accepted by the drivers team.


Lifecycle and operational gaps
------------------------------

Indirect FIPs inherit weaker lifecycle protection than directly-connected
FIPs:

.. list-table::
   :header-rows: 1

   * - Action
     - Direct FIP
     - Indirect FIP (this spec)
   * - Remove external gateway from hosting router while FIP exists
     - Blocked (409)
     - Blocked (409)
   * - Remove intermediate router interface on transit path while FIP exists
     - N/A
     - **Not blocked** — datapath may break silently
   * - Change static routes on transit routers
     - N/A
     - **Not validated**

This has to be documented in the release notes and admin guide.


Alternatives Considered
=======================

The following alternatives are considered candidates for the implementation
of this spec.

Unlike the proposed solution, the alternatives below do not make
``router_id`` writable on the ``floatingip`` resource. If one of them is
chosen instead, Neutron should still expose a lightweight **shim API
extension** — for example ``floating-ip-indirect`` — so that clients can
discover through ``GET /v2.0/extensions`` whether indirect FIP association is
supported on the deployment. The shim is advertisement only: it signals that
the relaxed resolver (config option or route-based validation) is active, not
how to select the hosting router.


Alternative 1: Bounded BFS — ``max_indirect_fip_hops``
------------------------------------------------------

The main point of this alternative is to add a new configuration option,
``max_indirect_fip_hops`` (default ``0``), instead of extending the FIP API.
When the option is ``0``, behavior is unchanged. When it is positive, Neutron
automatically discovers the hosting router at FIP association time — the
caller does not supply ``router_id``.

**Bounded BFS** means a *breadth-first search* over the L3 topology graph,
limited to a configured hop count:

* **Graph model:** subnets are nodes; two subnets are adjacent when a Neutron
  router has interfaces on both (adjacency is keyed on ``subnet_id``, not
  CIDR).
* **Breadth-first:** starting from the VM's fixed-IP subnet, expand outward
  one "layer" at a time — first routers on that subnet, then other subnets
  those routers touch, then routers on those subnets, and so on.
* **Bounded:** stop after ``max_indirect_fip_hops`` intermediate routers.
  For example, ``1`` allows a gateway router one transit hop away (as in the
  two-hop topology above: ``tenant-net → internal-router → transit-net →
  ext-router``); ``2`` allows two intermediate routers. This bounds the search
  so unrelated fabrics (peered networks, distant redundant gateways) are not
  pulled into every FIP association.

At each layer, Neutron collects routers that have an external gateway on the
FIP's external network. It returns a router ID **only if exactly one** such
gateway router is found within the hop bound; otherwise it fails closed with
404 (no guessing). The walk considers topological adjacency only — not static
routes or actual reachability.

**Pros:**

* **Config-only resolver** — the new ``max_indirect_fip_hops`` option selects
  the hosting router; no writable ``router_id`` on the ``floatingip`` resource
  (a shim extension such as ``floating-ip-indirect`` would be needed for API
  discoverability).
* Bounded blast radius (peered networks or distant redundant gateways stay outside
  the search radius).
* Subnet-ID keyed adjacency avoids confusion from overlapping CIDRs.

**Cons:**

* Multiple DB queries per association.
* Ambiguous topologies (two reachable gateway routers) fail entirely.
* Same lifecycle and route-validation gaps as the proposed solution.
* Does not cover the gateway-VM / static-route-only inner subnet case unless
  that subnet also appears in the adjacency graph.


Alternative 2: Static route-based validation
--------------------------------------------

This is original proposal from [LP#2072505]_.

Extend ``get_router_for_floatingip()`` to consider router ``extraroutes``:
accept a router if the fixed IP's address/subnet matches a route
**destination** and the route **nexthop** is on a subnet to which the router
is directly connected. As with Alternative 1, a shim extension such as
``floating-ip-indirect`` would be needed so clients can discover that this
relaxed validation is enabled.

**Pros:**

* Semantically closer to "is there a route?" for the gateway-VM use case.
* Uses existing extraroute data.

**Cons:**

* **Router discovery problem** — from an inner port alone, finding the
  relevant router may require scanning many routers.
* Poor fit for the two-hop transit topology (routes use IP nexthops, not
  router adjacency).
* Worst-case performance without careful indexing.
* High implementation and maintenance cost.


DB Impact
=========

No schema change required. The ``floatingips.router_id`` column already
exists.


Security Impact
===============

* Writable ``router_id`` is gated by new Oslo policy rules:

  * ``create_floatingip:router_id`` — project member or admin (default).
  * ``update_floatingip:router_id`` — project member or admin (default).

* By default, project members may set ``router_id`` only to routers visible
  to their project.
* Allowing indirect FIPs does not bypass port or network RBAC; it only
  relaxes which router may host NAT for an already-authorized port
  association.
* Misconfiguration (wrong ``router_id``) creates a FIP that appears ACTIVE
  but does not forward traffic — an operational risk, not a tenancy bypass.


Performance Impact
==================

Explicit ``router_id``: ~2 additional DB lookups (router existence + gateway
check) compared to the failure path today — negligible for low-frequency FIP
operations.


Implementation
==============

Work Items
----------

* **neutron-lib:** API definition for ``floating-ip-router-writable`` extension
  (writable ``router_id`` on create/update).
* **Neutron:** Extension implementation, policy rules, API reference.
* **Neutron:** Update ``get_router_for_floatingip()`` / ``_get_assoc_data()``
  to honor request-supplied ``router_id``.
* **Neutron:** Unit and API tests (including negative cases: wrong external
  network, missing gateway, RBAC).
* **neutron-tempest-plugin:** Scenario connectivity test to cover at least
  the simple use case described in the spec above.
* **Documentation:** API reference, admin guide (datapath prerequisites,
  backend limitations, lifecycle gaps), release notes.
* **OpenStackClient / SDK:** Follow-up patches to expose ``--router`` on FIP
  create/update (not blocking for core Neutron merge).


Testing
=======

* Unit tests for explicit ``router_id`` validation and fallback to legacy
  behavior when omitted.
* ML2/OVN functional tests:

  - Two-router transit topology (analogous to ext-router /
    internal-router lab setup).
  - Negative: gateway router without external network on FIP's network.
  - Negative: strict mode without ``router_id`` still returns 404.
  - Negative: FIP can't be associated with port which belongs to the VM
    connected to the isolated network.

* Document ML2/OVS/DVR test gaps or exclusions explicitly in test suite
  configuration.


Documentation Impact
====================

* Neutron API reference: ``floating-ip-router-writable`` extension and writable
  ``router_id``.
* Admin guide: two-hop transit prerequisites (``ovn_router_indirect_snat``,
  routing, OVN version), ML2/OVS/DVR limitations, lifecycle caveats.
* Release notes: new extension; behavior change when ``router_id`` is supplied.


References
==========

.. [LP#2072505] https://bugs.launchpad.net/neutron/+bug/2072505
.. [LP#1616317] https://bugs.launchpad.net/neutron/+bug/1616317
.. [LP#2051935] https://bugs.launchpad.net/neutron/+bug/2051935
.. [LP#1571676] https://bugs.launchpad.net/neutron/+bug/1571676
.. [MEETING] https://meetings.opendev.org/meetings/neutron_drivers/2024/neutron_drivers.2024-07-19-14.00.log.html
.. [FDP-744] https://issues.redhat.com/browse/FDP-744
.. [927560] https://review.opendev.org/c/openstack/neutron/+/927560
