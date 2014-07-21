..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================================
Big Switch - Convert L3 functions to L3 Service plugin
======================================================


https://blueprints.launchpad.net/neutron/+spec/bsn-l3-service-plugin

Move the L3 functions from the Big Switch monolithic plugin to a separate L3
service plugin so the fabric can provide L3 functionality for ML2 or the Big
Switch Plugin from one common code base.

Problem description
===================
There is no way to use the Big Switch L3 features when using the Big Switch
ML2 driver.


Proposed change
===============

Remove the L3 functions from the Big Switch core plugin and put them into an
L3 service plugin. This will allow the Big Switch backend to provide L3 service
when using the Big Switch plugin or the ML2 driver.


Alternatives
------------

N/A

Data model impact
-----------------

Database migration for L3 tables will be needed for new service plugin.


REST API impact
---------------
N/A

Security impact
---------------
N/A

Notifications impact
--------------------
N/A

Other end user impact
---------------------
N/A

Performance Impact
------------------
N/A

Other deployer impact
---------------------
An L3 service plugin configuration will be required for the BSN core plugin.

Developer impact
----------------

N/A

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  kevinbenton

Work Items
----------

* Separate code into L3 service plugin module
* Re-organize BSN L3 tests into separate code from current unit test files

Dependencies
============

This will be completed after the L3 offload functionality is done since that
will require more L3 factoring and is a higher priority.[1]


Testing
=======

The current unit tests and 3rd party CI will cover this change since no
new features are being added.


Documentation Impact
====================
The service plugin will have to be referenced in the configuration guide for
the Big Switch plugin and the Big Switch ML2 agent.


References
==========

1. https://blueprints.launchpad.net/neutron/+spec/bsn-blended-l3-agent
