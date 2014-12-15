..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================
Allow Filtering of Routers by Network ID
========================================

https://blueprints.launchpad.net/neutron/+spec/filter-routers-by-network-id

Currently, there is no way to list all the routers which are connected to a
single network. This spec proposes an optional 'network_id' filter to the
router-list REST API, which (when used) will allow one to get a list of all the
routers which belong to a specific network.

Problem Description
===================

On a deployed setup with a lot of networks and routers, it's very difficult to
get a list of all the routers which are connected to a specific network, and
includes manually filtering out all the irrelevant routers.

Proposed Change
===============

This spec proposes adding a new parameter to the REST API for router-list,
'network_id', which will return routers which are logically connected to the
specified network. This is done by querying a list of all the ports which are
connected to the specified network, and then filtering from them only the ports
that belong to some router.

Data Model Impact
-----------------

None. All the data is already present on the database, we simply need to filter
them.

REST API Impact
---------------

* Filtering routers by network_id will be allowed by GET to the address:
  /v2.0/routers.json?network_id=network_id

Security Impact
---------------

None

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

The new flow of filtering for routers by a network_id will only occur of
'network_id' is provided in the GET query. If it isn't nothing is changed in
the flow and no impact should be had.

In case 'network_id' is provided, an SQL query which joins relevant rows of
both the Router and Port models is created and executed. This is done only when
a tenant asks for the information.

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

None

Developer Impact
----------------

None

Community Impact
----------------

None

Alternatives
------------

* The user is able to produce a list of all the ports, and for each use
  'port-show' and see if the 'network_id' field matches the network_id the user
  is interested in, and if the 'device_owner' is a router. This is quite
  cumbersome.
* A similar implementation can be automated in neutronclient, but will only add
  more throughput (a list of all the ports must be sent from the server to the
  client, and once the data is there the neutronclient code will need to
  manually filter it. The proposed fix does this at the DB level.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  jschwarz (jschwarz@redhat.com)

Work Items
----------

* Implementation (patch 111580)


Dependencies
============

None

Testing
=======

Tempest Tests
-------------

None

Functional Tests
----------------

None

API Tests
---------

* A unit test which adds 2 routers, 2 subnets and 2 ports, and makes sure that
  using router-list using the network-id parameter only produce the wanted
  routers should be added.
* A similar test for external networks should also be added.


Documentation Impact
====================

The new field, 'network_id', should be documented.

User Documentation
------------------

None

Developer Documentation
-----------------------

None

References
==========

https://review.openstack.org/#/c/111580/
