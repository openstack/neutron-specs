====================
Radware LBaaS Driver
====================

https://blueprints.launchpad.net/neutron/+spec/radware-lbaas-driver

Neutron/LBaaS driver for Radware ADC.


Problem Description
===================

The new driver would allow using Radware ADC appliances
 as backends for Neutron/LBaaS functionality.


Proposed Change
===============

The driver (v2 driver based on v2 object model and API) will implement the
interfaces in the LoadBalancerBaseDriver, using vDirect REST API,
a JSON HTTP interface for configuring Radware appliances.
The following managers will be implemented:

* LoadBalancerManager
* ListenerManager
* PoolManager
* MemberManager
* HealthMonitorManager

Alternatives
------------

None.

Data Model Impact
-----------------

None.

REST API Impact
---------------

None.

Security Impact
---------------

None.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None.

Performance Impact
------------------

None.

IPv6 Impact
-----------

None.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

None.

Community Impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee: https://launchpad.net/~avishayb


Work Items
----------

* Radware driver code
* Unit tests
* Voting CI

Dependencies
============

* https://review.openstack.org/#/c/101084/


Testing
=======

Tempest Tests
-------------

None.

Functional Tests
----------------

Functional Tests sohuld be added for driver

API Tests
---------

None.

Documentation Impact
====================

User Documentation
------------------

None.

Developer Documentation
-----------------------

None.

References
==========

None.
