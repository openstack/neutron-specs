..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================================================
Layer 3 service plugin to support hardware based routing (SVI) on Brocade VDX
=============================================================================

https://blueprints.launchpad.net/neutron/+spec/brocade-l3-svi-service-plugin

This blueprint exploits the L3 SVI functionality in the Brocade VDX
switch via an L3 router service plugin.


Problem description
===================

Brocade Hardaware supports SVI (Switch Virtual Interface) which provides ASIC
level routing/gateway functionality in the switch for configured VLANs. This
Service plugin provides support for this feature which enables line rate
routing/gateway functionality.

Please see definition of SVI in the references section of this document.


Proposed change
===============

This proposal will introduce a Layer 3 service plugin which will interface
with the Brocade VDX switch using NETCONF. NETCONF interface on the switch
will be used to program the SVI on the switch.

This plugin works in conjuction with Brocade ML2 Driver which will continue
to manage networks, subnets, and ports.

We plan to support the full API.


Alternatives
------------

The alternative approach is to use the open source agent based layer 3 router
plugin and update the L3 agent to interact with Brocade Hardware.

Data model impact
-----------------

In the Brocade specific DB a table will be added to track the SVI created on
a tenant basis. Mapping of the SVI to the VLAN will also be provided.

new table: brocade_db.svi (id, vlan, tenant_id)
id - standard uuid
vlan - svi's associated vlan
tenant_id: tenant this svi belongs to

REST API impact
---------------

none

Security impact
---------------

none

Notifications impact
--------------------

none

Other end user impact
---------------------

none

Performance Impact
------------------

There are no changes to any existing code patterns, hence there is no
significant change in performance profile. Instead of software L3 service
this blueprrint provides the same service using the VDX hardware. NETCONF
requests to the switch may add a very slight change to the execution time of
the API but we expect this change to be minor and will affect only setups
depolying this hardware based functionality.

Other deployer impact
---------------------

none

Developer impact
----------------

none


Implementation
==============

Assignee(s)
-----------

Shiv Haris
sharis@brocade.com
IRC - shivharis

Work Items
----------

* L3-SVI Service Plugin
* DevStack related enhancements to support this plugin


Dependencies
============

none


Testing
=======

Complete unit test coverage of the code will be provided with mocked hardware.

Brocade will provide Third-Party tempest code coverage of this functionality. This
will be implemented as a CI (continuous integration) Testing. L3 routing tests will
be enabled.



Documentation Impact
====================

Documentation to configure and deploy this service plugin will be provided
in the Openstack wiki.


References
==========

SVI General definition:
Wikipedia: http://en.wikipedia.org/wiki/Switvh_virtual_interface






