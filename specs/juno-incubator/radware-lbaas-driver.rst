====================
Radware LBaaS Driver
====================

https://blueprints.launchpad.net/neutron/+spec/radware-lbaas-driver

Neutron/LBaaS driver for Radware ADC.


Problem description
===================

The new driver would allow using Radware ADC appliances
 as backends for Neutron/LBaaS functionality.


Proposed change
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

Data model impact
-----------------

None.

REST API impact
---------------

None.

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

* Unit tests

* Radware QA

* Third party CI


Documentation Impact
====================

None.

References
==========

None.
