========================================
Ensure routes order in subnet and router
========================================

As route may be recursive, some routes must be installed after the depending
route. So host_routes of subnets and routes of routers must be ordered
correctly.

Problem description
===================

Assume we are going to setup 2 routes in subnet 192.168.0.0/24 whose default
gateway is 192.168.0.1:

1. 192.168.0.0/16 direct connected
2. 10.0.0.0/8 via 192.168.1.1

The second one depends the first one, so it must be installed after the first
one. Currently there is no way to arrange the routes order in neutron, by
default it is ordered alphabet, where the second one goes first, leads to some
error.

Proposed change
===============

Adds an order column to host_routes and route_routes table, which stores the
order of routes.

When create or update subnet host routes or router routes, the order column is
automatically assigned when inserting the data to database based on user input
order.

When retrieve routes, they are returned as the same order as the origin user
input.

Alternatives
------------

None.

Data model impact
-----------------

* subnetroutes
  - add order column
* routerroutes
  - add order column

REST API impact
---------------

None, but the subnet host routes and route routes will retain the order same as
the input when they are created or updated.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

Performance Impact
------------------

None.

Other deployer impact
---------------------

None.

Developer impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  zangmingjie

Work Items
----------

Single patch.

Dependencies
============

None.

Testing
=======

 * Unit tests, the routes should be returned ordered.
 * Tempest API tests

Documentation Impact
====================

None

References
==========

None
