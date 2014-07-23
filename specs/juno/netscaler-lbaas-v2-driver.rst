================================================================================
NetScaler LBaaS V2 Driver
================================================================================

NetScaler ADC driver for LBaaS v2 model

https://blueprints.launchpad.net/neutron/+spec/netscaler-lbaas-driver

Problem description
===================

Driver for the Neutron LBaaS plugin for using the Citrix NetScaler
loadbalancing devices to provide Neutron LBaaS functionality in OpenStack
based on the LBaaS V2 model described in
https://review.openstack.org/#/c/89903/.

Proposed change
===============

The driver will implement the interfaces according to the driver interfaces
mentioned in the spec https://review.openstack.org/100690
for the blueprint
https://blueprints.launchpad.net/neutron/+spec/lbaas-objmodel-driver-changes

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

Primary assignee: https://launchpad.net/~vijay-venkatachalam


Work Items
----------

* NetScaler driver code
* Unit tests
* Voting CI

Dependencies
============

* https://review.openstack.org/#/c/101084/
* https://review.openstack.org/#/c/105610/

Testing
=======

* Unit tests
* NetScaler QA
* NetScaler CI

Documentation Impact
====================

None.

References
==========

None.

