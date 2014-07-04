..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================================
Enhance floating IP router lookup to consider extra routes
==========================================================

https://blueprints.launchpad.net/neutron/+spec/floating-ip-extra-route

Allow assigning a floating IP to a port that is not directly connected to the
router (a port on the ``internal-net`` in the diagram below).

Example setup::

  +––––––––––+  +–––––––+  +––––––––––––––––+  +–––––––+  +––––––––––––+
  |public-net+––+router1+––+intermediate-net+––+router2+––+internal-net+––+
  +––––––––––+  +–––––––+  +––––––––––––––––+  +–––––––+  +––––––––––––+

Problem description
===================

When a floating IP is associated with an internal address, there is a
verification that the internal address is reachable from the router. Currently
it only considers internal addresses that are directly connected to the router,
and does not allow association to internal addresses that are reachable by
routes over intermediate networks.

An example use case would be a setup where a compute instance that implemets a
multi-homed gateway and functions as a VPN or firewall gateway protecting an
internal network. In this setup the port of an ``internal-net`` server to which
we wish to assign a floating IP will not be considered reachable, even if there
exists an extra route to it on ``router1``.

Proposed change
===============

Extend the verification to allow the floating IP association if the internal
address belongs to a network that is reachable by an extra route, over other
gateway devices (such as routers, or multi-homed compute instances).

Iterate over routers belonging to the same tenant as the ``internal-net``,
and select routers that have an extra route that matches the fixed IP on the
``internal-net`` port which is the target of the floating IP assignment. Sort
the candidate routers according to the specificity of the route (most specific
route is first). Use the router with the most specific route for the floating
IP assignment.

Alternatives
------------

The proposal above takes a minimalist approach that trusts the tenant to ensure
that there exists a path between the router and the internal port (the target
of the floating IP association). As discussed in [#third]_, a follow-on
enhancement can add such validation.

Also, the tenant is trusted to maintain the extra route for the life of the
floating IP association. A future enhancement can add a validation before an
extra route deletetion that there is no floating IP association that becomes
unreachable without the deleted route.

Data model impact
-----------------

None.

REST API impact
---------------

None.

Security impact
---------------

None. There is validation that the router that is selected for the floating IP
assignement belongs to the same tenant as the internal port that is the target
for the floating IP assignment.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

Performance Impact
------------------

The perfomance impact is expected to be minimal.

The added router verification code runs only over routers that belong to the
same tenant as the ``internal-net`` port. For each such router that has an
extra route that matches the internal address, there will be an extra query to
validate that the router is indeed on the ``public-net``.

Other deployer impact
---------------------

None. The change enables floating IP assignment in network topologies, where it
was not possible in the past. As such, it will not affect existing deployments.

Developer impact
----------------

Plugins that override L3_NAT_db_mixin._get_router_for_floatingip() should be
modified to call L3_NAT_db_mixin._find_routers_via_routes_for_floatingip() and
iterate over the resulting list of routers. The iteration should apply any
plugin specific requirements and select an appropriate router from the list.
(The blueprint implementation already contains this change for
neutron/plugins/nec/nec_router.py).

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  https://launchpad.net/~ofer-w

Other contributors:
  https://launchpad.net/~zegman

Work Items
----------

Enhance floating IP router lookup: https://review.openstack.org/#/c/55987/

Dependencies
============

None.

Testing
=======

A new unit test: neutron.tests.unit.test_extension_extraroute:
ExtraRouteDBTestCaseBase.test_floatingip_reachable_by_route() will be added to
define the topology in the example setup diagram above. It will test the
association of a floating IP to the internal network, including tenant ID match
and multiple fixed IPs for the internal network.

Documentation Impact
====================

None.

References
==========

.. [#first] http://lists.openstack.org/pipermail/openstack-dev/2013-December/021579.html
.. [#second] http://lists.openstack.org/pipermail/openstack-dev/2014-January/025940.html
.. [#third] http://lists.openstack.org/pipermail/openstack-dev/2014-February/026194.html
