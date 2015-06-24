..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================================
Support external physical bridge mapping in linuxbridge
=======================================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/phy-net-bridge-mapping

This proposal implements physical bridge mapping for linuxbridge-agent.
According to this proposal, linuxbridge-agent will be able to be aware of
external user-defined physical bridges and attach tap devices on it.


Problem Description
===================

The linuxbridge-agent currently creates a bridge for each physical network
used as a flat network, moving any existing IP address from the interface
to the newly created bridge. This is very helpful in some cases, but there are
other cases where the ability to use a pre-existing (user-defined) bridge.

For instance, the same physical network might need to be bridged for other
purposes, or the agent moving the system's IP might not be desired or cause
some unexpected error for the pre-defined environment.

According to some enterprise end-users' feedbacks, they do need this feature
to make neutron compatible with their pre-defined virtual network environment
due to some security regulations.


Proposed Change
===============

Add a bridge_mappings configuration variable, similar to that used
by the openvswitch-agent, alongside the current physical_interface_mappings
variable.

When a physical bridge for a flat network is needed, the bridge mappings would
be checked first. If a bridge mapping for the physical network exists, it would
be used.

If not, the interface mapping would be used and a bridge for the interface
would be created automatically. Sub-interfaces and bridges for VLAN networks
would continue to work as they do now, created by the agent using the interface
mappings.

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

Add an option called bridge_mappings in
/etc/neutron/plugins/linuxbridge/linuxbridge_conf.ini.

Developer Impact
----------------

None.

Community Impact
----------------

This change is a value-added feature for linuxbridge-agent, and it won't
affect the current development cycle.

The implementation is simple and straightforward.

Alternatives
------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  nick-ma-z

Other contributors:
  None

Work Items
----------

* Implement the feature
* Provide unit tests

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

None. This new feature can be fully covered by unit test cases.

API Tests
---------

None.


Documentation Impact
====================

User Documentation
------------------

Add an option called bridge_mappings in
/etc/neutron/plugins/linuxbridge/linuxbridge_conf.ini.

Developer Documentation
-----------------------

None.


References
==========

See related bug: https://bugs.launchpad.net/neutron/+bug/1105488
