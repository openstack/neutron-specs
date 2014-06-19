..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
bulk-get-device-details
==========================================

https://blueprints.launchpad.net/neutron/+spec/bulk-get-device-details

Introduce a rpc call to get device details for multiple devices instead of one
by one.


Problem description
===================

An agent needs to communicate with Neutron server in order to get some infomation
about the devices (to handle devices added for example).
Currently there's an rpc call that can be used by the agent to get the details
of a device, get_device_details . This call needs to be issued for each device.
See neutron/agent/rpc.py .

The purpose of this blueprint is to add a call to get the details of multiple
devices. This way instead of making N calls for N devices only one call will be
made.


Proposed change
===============
Introduce a rpc call to get device details for multiple devices instead of one
by one.


Alternatives
------------

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

Performane will be improved by this change

Other deployer impact
---------------------

None

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Rossella Sblendido

Primary assignee:
  <rossella-s>


Work Items
----------


Dependencies
============


Testing
=======

Unit tests will be added

Documentation Impact
====================

No impact

References
==========
None
