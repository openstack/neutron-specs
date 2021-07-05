..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
Allow multiple external gateways on a router
============================================

RFE: https://bugs.launchpad.net/neutron/+bug/1905295

This spec proposes to allow a Neutron router to have multiple external
gateways, so we can represent router designs where the router has
multiple legs towards the outside world for reasons of higher capacity
(typically load balanced between these legs) and higher availability
(tolerating the failure of a router uplink or a router gateway port).

Problem Description
===================

Below we describe one cloud design that required us to allow multiple
external gateways on a single router.  However the feature proposed in
this spec is in no way specific to this design, this spec only strives
to allow multiple external gateways for any design that needs it.

The Neutron router could be an active-active HA router.  Compute hosts are
connected to the Neutron router via an MLAG, the two sides of the router
present a single IP to the computes.  There is a single logical router
one hop away from the Neutron router towards the external world (from
here on: datacenter gateway) which is also an active-active HA router.
The Neutron router and the datacenter gateway are interconnected by 2x2
point-to-point links (from each side to each side).  The Neutron router
in this setup has 4 links towards the external world.

::

  +--+   +--+
  |R3|   |R4| Datacenter gateway
  +--+   +--+
   | \   / |
   |  \ /  |
   |   X   |  2x2 point-to-point links
   |  / \  |
   | /   \ |
  +--+   +--+
  |R1|   |R2| Neutron router
  +--+   +--+
   | \   / |
   |  \ /  |
   |   X   |  MLAG
   |  / \
   | /
  +--+
  |C1|   ...  Compute hosts
  +--+

Proposed Change
===============

Overall Picture
---------------

The information in an ``external_gateway_info`` structure is used to
create a gateway port.  A gateway port is a router leg, where beyond
packet forwarding, we perform SNAT (if enabled) and DNAT (for floating
IPs and port forwardings) and which influences the default route of
that router.  Given multiple ``external_gateway_info`` structures
we can straightforwardly create multiple gateway ports.  We can also
straightforwardly install the directly connected routes of the additional
external gateways.

With only one external gateway - regarding packet forwarding - naturally
only the internal-internal and internal-external directions exist.
However with multiple external gateways the external-external direction
also becomes possible. We don't have a use case for external-external
forwarding. However for the sake of simplicity of implementation this
spec proposes to support that, that is to allow packet forwading from
one external gateway to another.  If required, a follow-up spec may
propose finer control of external-external packet forwarding.

However the default route of a router cannot be multiplied without
complex consequences (e.g. load balancing between multiple nexthops).
So for the sake of simplicity for each router we retain one external
gateway as special.  The special one is used to set the default route of
a router as before, while the other external gateways have no influence
on the default route.

NATting and reservation of floating IPs can be performed for each external
gateway as usual, but please see the 'Out of Scope' section.

The first implementation targets centralized routers on l3-agent.

For the special, first gateway port we keep adding the same routes as before.

The usual set of implicitly managed routes in a Neutron router are:

* One directly connected route for each router interface's each subnet.
* One directly connected route for each gateway port's each subnet.
* One default route to the single gateway port's subnet's ``gateway_ip``.

Keeping to this logic, when we add further gateway ports, we also add a
directly connected route for each additional gateway port's each subnet.
Therefore the traffic destined to these subnets will be sent already
via the respective gateway ports.  Also traffic may reach the gateway
ports from outside.  However the majority of traffic will be sent via
the default route.

Out of Scope
------------

First of all, overall support for active-active HA routers by the l3-agent
is not targeted here.

We intentionally only add a default route for the first, special gateway
port('s subnet's ``gateway_ip``). We do not add default routes for the
other gateway ports(' subnet's ``gateway_ip``). Further management
of routes (to make the additional external gateways actually used)
is left to:

* Managing additional routes via the extraroutes API.
* Future improvements to neutron-dynamic-routing so a Neutron router
  can also receive (not just advertise) additional routes. [1]

Since the extraroutes API is already available, we believe that some
use of the multiple external gateways can be realized as soon as this
feature is merged.

The advertisement of any routes with the newly introduced additional
external gateways as nexthops (for tenant networks or floating IPs)
is also out of scope here.  We believe that advertising such routes via
routing protocols clearly makes sense, however:

* This can be covered by a later spec in neutron-dynamic-routing and
* basic use of the changes proposed here is possible without routing protocols.

Backwards Compatibility
-----------------------

We propose to store and expose the external gateways a bit redundantly
to make backwards compatibility easier.  Where possible (API, DB, RPC)
keep the current scalar external gateway (or gateway port) attribute of
a router as is.  But also add a new router attribute for the plural:
external gateways or gateway ports which contain a list of (not just
the rest but) all such objects.  So the object in the scalar attribute
is present in this list too - preferably as the first element of the list.

This is just a high level approach that aims to leave all code unchanged
where we only need to keep backwards compatible behavior.

DB Impact
---------

The current DB schema contains the router - external gateway relations
somewhat redundantly.

First in the ``routerports`` table there is no constraint on the number
of ports with type ``network:router_gateway`` belonging to one router.
Today we only store at most one ``network:router_gateway`` port for each
router, but the DB schema allows more. [2]

Second the ``gw_port_id`` column of the ``routers`` table is a scalar.
Today it stores the same port uuid as we have in the ``routerports``
table. [3]

We propose:

* To keep the SQL schema as is, but start storing multiple
  ``network:router_gateway`` ports in the ``routerports`` table.
* To also keep ``routers.gw_port_id`` scalar and there store the one
  special, backwards compatible external gateway (so this one is present
  both in the ``routerports`` table and in ``routers.gw_port_id``).
* To extend the ``neutron.db.models.l3.Router`` class with new attribute
  ``gw_ports`` that map to all relevant ``network:router_gateway`` ports
  stored in the ``routerports`` table.

REST API Impact
---------------

Introduce a new API extension called ``multiple-external-gateways``.

This extension adds a new router attribute: ``external_gateways``.
Which is a list of ``external_gateway_info`` structures, for example:

::

    [
      {"network_id": ...,
       "external_fixed_ips": [{"ip_address": ..., "subnet_id": ...}, ...],
       "enable_snat": ...},
      ...
    ]

The first element in the list is special:

* It is always the same as the original ``external_gateway_info``.
* It is the one that sets the default route of the Neutron router.

The order of the the rest of the list is irrelevant and ignored.
Duplicates in the list (that is multiple external gateways with the same
``network_id``) are not allowed.

Updating ``external_gateway_info`` also updates the first element of
``external_gateways`` and it leaves the rest of ``external_gateways``
unchanged.  Setting ``external_gateway_info`` to an empty value also
resets ``external_gateways`` to the empty list.

The ``external_gateways`` attribute cannot be set in
``POST /v2.0/routers`` or ``PUT /v2.0/routers/{router_id}`` requests,
instead it can be managed via sub-methods:

* ``PUT /v2.0/routers/{router_id}/add_external_gateways``

  Accepts a list of ``external_gateway_info`` structures.  Adding an
  external gateway to a network that already has one raises an error.

* ``PUT /v2.0/routers/{router_id}/update_external_gateways``

  Accepts a list of ``external_gateway_info`` structures.  The external
  gateways to be updated are identified by the ``network_ids`` found
  in the PUT request.  The ``external_fixed_ips`` and ``enable_snat``
  fields can be updated.  The ``network_id`` field cannot be updated.

* ``PUT /v2.0/routers/{router_id}/remove_external_gateways``

  Accepts a list of potentially partial ``external_gateway_info``
  structures.  Only the ``network_id`` field from
  ``external_gateway_info`` structure is used.  The ``external_fixed_ips``
  and ``enable_snat`` keys can be present but their values are ignored.

The add/update/remove PUT sub-methods respond with the whole router
object just as ``POST/PUT/GET /v2.0/routers``.

RPC Impact
----------

The ``sync_routers`` message already has a ``gw_port`` field.  Extend the
message to also include a ``gw_ports`` field containing all gateway ports.
Bump the rpc version of this message.

Upgrade Impact
--------------

The ``sync_routers`` RPC message will have a new version.

Client Impact
-------------

Relevant changes in osc and openstacksdk.

Testing
-------

* Unit tests.
* Fullstack tests for l3-agent.
* Tempest tests in neutron-tempest-plugin.

Assignee(s)
-----------

* Bence Romsics <bence.romsics@gmail.com>

References
==========

[1] https://review.opendev.org/c/openstack/neutron-specs/+/783791
[2] https://opendev.org/openstack/neutron/src/commit/b7c4a11158786431c262cfcc2fc4bc46ab6bacd2/neutron/db/models/l3.py#L24
[3] https://opendev.org/openstack/neutron/src/commit/b7c4a11158786431c262cfcc2fc4bc46ab6bacd2/neutron/db/models/l3.py#L54
