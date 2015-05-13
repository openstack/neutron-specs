..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Enable spoofchk control for SR-IOV ports
==========================================

https://blueprints.launchpad.net/neutron/+spec/sriov-spoofchk

Allow user to control setting of MAC spoof checking for the SR-IOV ports.

Problem Description
===================

Support for SR-IOV ports appeared in Neutron Juno and allows allows VMs
to access virtual network via SR-IOV VFs. SR-IOV ports in Linux allow
specification of whether source MAC spoof checking should be enabled or
disabled for them. This can be done, for example, using the ip-link(8) tool:

  ip link set eth0 vf 2 spoofchk off

This command disables spoof checking for Virtual Function 2 on
Physical Device 'eth0'.

This feature is useful for bonding configurations inside guests.
For example, MAC spoof checking should be disabled for 802.3ad (Dynamic
link aggregation) bonds. Please see the 'Configuring QoS Features with
Intel Flexible Port Partitioning' whitepaper for more details, link is
available in the 'References' section.

Proposed Change
===============

The proposal is to leverage the port security extension. Specifically, for
'direct' types of ports port_security_enabled = False would mean that
spoof checking should be disabled.

Actual setting for the VF will be done by the sriovnicagent using the ip-link(8)
tool.

Default value for the spoof checking will be enabled, so the change will not
affect a default behavior.

Data Model Impact
-----------------

None

REST API Impact
---------------

None

Security Impact
---------------

That could have a security impact if user disables spoof checking for
a specific port, however it is expected and user can manually decide if
that's applicable for his configuration.

As spoof checking is enabled by default, there would be no security impact
with the default setting.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

And user will have a facility to control spoof checking settings on specific
Neutron ports.

* python-neutronclient does not need to be modified

Performance Impact
------------------

It'll have a slight impact on the sriovagent as it'll have to run one more
external command (ip-link(8)).

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

None

Developer Impact
----------------

None

Community Impact
----------------

This changes has not been discussed so far.

Alternatives
------------

Another possible way of providing control to user is to introduce
a dedicated attribute for spoof checking instead of 'port_security_enabled'
from the portsecurity extension. It could be named 'spoofchk' and be placed
either into portsecurity extension or into portbindings extension.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  novel

Work Items
----------

 * Modify sriovagent to be able to enable or disable spoof checking based on
   user setting
 * Verify on API level and in binding handling routines that sriovagent is
   not disabled as it being turned off would not allow to meet user's
   expectation.

Dependencies
============

None

Testing
=======

Tempest tests are not planned currently as SR-IOV hardware is not always
available. 3rd party CI testing could be considered, though probably the
feature is relatively minor for that.

Tempest Tests
-------------

None

Functional Tests
----------------

None

API Tests
---------

API tests would be added to cover specifics of port_security for 'direct'
type of ports.

Documentation Impact
====================

User Documentation
------------------

User documentation will be updated with information about spoof checking
control for 'direct' ports and its security considerations.

Developer Documentation
-----------------------

None

References
==========

 * http://www.intel.com/content/www/us/en/network-adapters/10-gigabit-network-adapters/config-qos-with-flexible-port-partitioning.html
 * https://www.kernel.org/doc/Documentation/networking/bonding.txt
