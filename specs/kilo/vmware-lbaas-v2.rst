====================
VMWware LBaaS Driver
====================

https://blueprints.launchpad.net/neutron/+spec/vmware-lbaas-v2

Neutron/LBaaS driver for VMWare NSX-v loadbalancer.


Problem Description
===================

The new driver would allow using VMWare NSX-v loadbalancer
for Neutron/LBaaS functionality.


Proposed Change
===============

The driver (v2 driver based on v2 object model and API) will implement the
interfaces in the LoadBalancerBaseDriver, using vDirect REST API,
a JSON HTTP interface for configuring VMWare appliances.
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

Is part of Openstack for NSX-v solution.
See [1]

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

Primary assignee: https://launchpad.net/~ksamoray


Work Items
----------

* VMWare driver code
* Unit tests
* 3rd party CI

Dependencies
============

None.

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

Is covered by CI, running unit, tempest, scenario tests

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

[1] NSX - http://www.vmware.com/products/nsx/
[2] NSX-vSphere API - https://www.vmware.com/support/pubs/nsx_pubs.html

