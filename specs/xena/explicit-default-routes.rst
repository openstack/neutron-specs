..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Manage default route(s) explicitly
==================================

RFE: https://launchpad.net/bugs/1921126

This spec proposes to allow explicit management of the default route(s)
of a Neutron router.  This is mostly useful for a user to install
multiple default routes for Equal Cost Multipath (ECMP) and treat all
these routes uniformly.

Problem Description
===================

Today the ``extraroute`` API allows explicit addition of default route(s),
where by default route we mean a route with destination=0.0.0.0/0
and an arbitrary (connected) nexthop.  But the router already has an
implicit default route derived from its external gateway settings (with
its nexthop being the gateway port's first subnet's ``gateway_ip``).
To the best of my knowledge the interaction of the default routes out of
these two sources is problematic.  For example our api-ref has warnings
about it [1]_. Other times people not familiar with these warnings consider
this behavior outright buggy [2]_.

We also already have a spec merged for ECMP [3]_. ECMP makes as much
sense for the default route as for any other route.  But today if you
want to manage multiple default route entries in order to have ECMP over
the default route, then you have to manage a mishmash of the implicit
default route and some other default routes via the ``extraroute`` API.

Proposed Change
===============

This spec proposes to separate the implicit management of a router's
default route from the explicit.  A router either has the one traditional
implicit default route or all of its default route(s) are explicitly
managed via the extraroute API.  But never a mixture of these two.

We propose to have a new API extension: ``default-route-management``.
Which introduces a new router attribute: ``explicit_default_routes``.
Which is a boolean and defaults to ``False``.
It can be set at router create and update.

When ``explicit_default_routes`` is set to ``False`` (implicit mode) the
default route of a router is derived from its external gateway as before.
This default route is not visible through the ``extraroute`` API (again
as before).  For the sake of backward compatibility we should not reject
default routes on the ``extraroute`` API.  However we could document
that in implicit mode managing default routes over the ``extraroute``
API is not recommended.

Updating a router's ``explicit_default_routes`` (in both directions)
is rejected except when a router's ``routes`` attribute contains:

* either no default routes,
* or exactly one default route with the same nexthop as in implicit mode.

When ``explicit_default_routes`` is set to ``True`` (explicit mode)
the Neutron router is not going to have any default routes installed
implicitly. However a router can be updated from implicit to explicit
mode without traffic loss if the one extra route allowed above (that is:
exactly one default route with the same nexthop as in implicit mode) is
present before switching to explicit mode. In this case the default route
is preserved through the update.

In explicit mode default routes can be managed via the ``extraroute`` API.
And they can be managed properly - for example all of the default routes
set are visible via the ``extraroute`` API.

DB Impact
---------

Extend the ``routers`` table with boolean column
``explicit_default_routes`` defaulting to ``False``.

REST API Impact
---------------

New API extension: ``default-route-management`` introducing new router
attribute: ``explicit_default_routes``.

Client Impact
-------------

Relevant changes in osc and openstacksdk.

Testing
-------

* Unit tests.
* Tempest tests in neutron-tempest-plugin.

Assignee(s)
-----------

* Bence Romsics <bence.romsics@gmail.com>

References
==========

.. [1] https://docs.openstack.org/api-ref/network/v2/#extra-routes-extension

.. [2] https://bugs.launchpad.net/neutron/+bug/1901992

.. [3] https://specs.openstack.org/openstack/neutron-specs/specs/wallaby/l3-router-support-ecmp.html
