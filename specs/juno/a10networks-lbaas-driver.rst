..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
A10 Networks LBaaS Driver
==========================================

https://blueprints.launchpad.net/neutron/+spec/a10networks-lbaas-driver

Resubmitting Icehouse BP.  Neutron/LBaaS driver for A10 Networks appliances.


Problem description
===================

The new driver would allow using A10 Networks ADC appliances (hardware or
software) as backends for Neutron/LBaaS functionality.


Proposed change
===============

The driver will implement the interfaces in the lbaas abstract_driver, using
axAPI verison 2.1, a JSON HTTP interface for configuring A10 appliances.  The
currently implemented methods are:

* create_vip
* update_vip
* delete_vip
* create_pool
* update_pool
* delete_pool
* stats
* create_member
* update_member
* delete_member
* update_pool_health_monitor
* create_pool_health_monitor
* delete_pool_health_monitor

Among the current LBaaS functionality (as of Icehouse), the only unsupported
feature is APP_COOKIE persistence.

Driver will support  the upcoming Juno LBaaS object model changes.  Juno TLS support will be included in a future blueprint.

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

Primary assignee: https://launchpad.net/~dougwig


Work Items
----------

* A10 driver code
* Unit tests
* Voting CI

Dependencies
============

Driver likely affected by LBaaS model and TLS changes for Juno:

* https://blueprints.launchpad.net/neutron/+spec/lbaas-api-and-objmodel-improvement

* https://blueprints.launchpad.net/neutron/+spec/lbaas-ssl-termination


Testing
=======

* Unit tests

* A10 QA

* Existing LBaaS tests provide complete coverage, if driver is installed
  and configured (as our CI will do.)

* Not testable in gate, requires hardware.  Third party CI will be in place.


Documentation Impact
====================

None.

References
==========

* Github repo: https://github.com/a10networks/a10_lbaas_driver

* axAPI reference and examples: http://www.a10networks.com/products/axseries-aXAPI.php

* Old description doc: https://docs.google.com/file/d/0B2tCOk4L0wErdEpfdGtPMXpqM0k/edit

