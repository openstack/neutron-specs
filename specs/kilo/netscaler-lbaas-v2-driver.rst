..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================================
Neutron LBaaS v2 driver for Citrix NetScaler appliances
=======================================================

https://blueprints.launchpad.net/neutron/+spec/netscaler-lbaas-v2-driver

Neutron LBaaS v2 driver for Citrix NetScaler appliances.


Problem Description
===================

Enable NetScaler appliances to be LBaaS backends.


Proposed Change
===============

The driver will implement the LBaaS v2 driver interface, as a shim to a
middle ware NetScaler Control Center, ultimately configuring LB in
NetScaler appliances. This implementation will be similar to the
current v1 driver.

This driver will include TLS and L7 functionality included in LBaaS v2.


Data Model Impact
-----------------

None

REST API Impact
---------------

None

Security Impact
---------------

Driver communicates with infrastructure hardware via https.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

None

IPv6 Impact
-----------

Will support ipv6 at the same level as neutron lbaas.

Other Deployer Impact
---------------------

NetScaler Control Center virtual appliance must be deployed prior to using this driver.

Developer Impact
----------------

None

Community Impact
----------------

None

Alternatives
------------

N/A

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  https://launchpad.net/~vijay-venkatachalam

Work Items
----------

* Driver shim

* Unit tests for shim

* 3rd-party CI


Dependencies
============

* LBaaS v2

Testing
=======

Tempest Tests
-------------

Third-party CI will run existing LB tempest tests with NetScaler Control Center as middleware
and NetScaler VPX appliances as backends.

Functional Tests
----------------

Third-party CI will run existing LB functional tests with NetScaler Control Center as middleware
and NetScaler VPX appliances as backends.

API Tests
---------

Third-party CI will run existing LBaaS API tests with NetScaler Control Center as middleware
and NetScaler VPX appliances as backends.


Documentation Impact
====================

User Documentation
------------------

None

Developer Documentation
-----------------------

None

References
==========

* LBaaS v2 - https://review.openstack.org/#/c/138205/
