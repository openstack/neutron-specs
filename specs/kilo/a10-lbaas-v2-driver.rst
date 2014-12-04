..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Example Spec - The title of your blueprint
==========================================

https://blueprints.launchpad.net/neutron/+spec/a10-lbaas-v2-driver

Neutron LBaaS v2 driver for A10 Networks appliances.


Problem Description
===================

Enable A10 Networks appliances to be LBaaS backends.


Proposed Change
===============

The driver will implement the LBaaS v2 driver interface, as a shim to an
open-source pypi package, similar to the current v1 driver.

This driver will include TLS and L7 functionality included in LBaaS v2.

This driver will not support APP_COOKIE persistence.


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

Pypi package 'a10-neutron-lbaas' must be installed prior to using this driver.

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
  https://launchpad.net/~dougwig

Work Items
----------

* Driver shim (this spec is likely longer than the driver will be.)

* Unit tests for shim being shimmy

* 3rd-party CI


Dependencies
============

* LBaaS v2

Testing
=======

Tempest Tests
-------------

Third-party CI will run existing LB tempest tests with A10 appliances.

Functional Tests
----------------

Third-party CI will run existing LB functional tests with A10 appliances.

API Tests
---------

Third-party CI will run existing LB API tests with A10 appliances.


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
