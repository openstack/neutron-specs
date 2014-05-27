..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
Provider Segment Support for Cisco Nexus Switches
=================================================
https://blueprints.launchpad.net/neutron/+spec/ml2-cisco-nexus-mechdriver-providernet

This blueprint is to implement ML2 provider segment support for the Cisco 3K, 5K
and 7K Nexus switches.

The core cisco plugin supports this feature. Much of the work required is a
port of this code to the ML2 cisco_nexus mechanism driver (MD).

ML2 already supports the "provider" extension.

Problem Description
===================
Need to keep ToR provider segment information made by a network administrator
configured when the ML2 cisco_nexus mechanism driver is enabled.

Currently the ML2 cisco_nexus mechanism driver does not distinguish between
provider vs. tenant network segments. Update and delete port events result in
adding and removing vlan configuration on the nexus switches. More information
needs to be available to the ML2 MDs so that different actions can be taken
based on what network type is being created.

Proposed Change
===============
Provider segment support is already available under the core cisco plugin. This
code will be ported over to the ML2 cisco_nexus MD.

The ML2 type driver refactoring blueprint [1] is a requirement.
This blueprint will make available to the MDs the network segment type
(tenant vs. provider).

Data Model Impact
-----------------
None

History: The core cisco plugin required a database table be created which held
provider network information (see 'cisco_provider_networks' references under
plugins/cisco). This table was created because the provider vs. tenant
information was only available during network events. During these events
the OVS plugin would overwrite the "provider:xxx" information with either
provider or tenant data. When port events are processed, distinguishing between
provider vs. tenant networks is no longer possible. The core cisco database
table was created for this reason.

With "ML2 Type drivers refactor to allow extensiblity" blueprint [1] implemented
this information should be available for all events (network, subnet, port).
Therefore this DB table will no longer be needed.

REST API Impact
---------------
None

Security Impact
---------------
None

Notifications Impact
--------------------
None

Other End User Impact
---------------------
None

Performance Impact
------------------
None

Other Deployer Impact
---------------------
In addition to the core cisco provider segment control plane code being ported
over the core cisco provider segment configuration variables will also be
ported. These variables will be created/accessed by the ML2 cisco_nexus MD.
Port of these core cisco plugin variables:
provider_vlan_name_prefix - VLAN Name prefix for provider vlans.
provider_vlan_auto_create - Provider VLANs are automatically created as needed
on the Nexus switch.
provider_vlan_auto_trunk - Provider VLANs are automatically trunked as needed
on the ports of the Nexus switch.

Developer Impact
----------------
None

Community Impact
----------------
None

Alternatives
------------
None

Implementation
==============

Assignee(s)
-----------
Rob Pothier <rpothier@cisco.com>

Work Items
----------
- Provider segment control plane code added to ML2 cisco_nexus MD.
- Provider segment ML2 cisco_nexus unit tests.

Dependencies
============
[1] ML2 Type Driver refactor part 3
- https://review.openstack.org/#/c/115151

Testing
=======

Tempest Tests
-------------
For tempest test coverage, third party testing is provided for tenant networks.
The Cisco CI reports on all changes affecting this driver. The testing is run
in a setup with an OpenStack deployment (devstack) connected to a Cisco Nexus 5K
physical switch. Although provider segment networks are not specifically tested,
changes made will be exercised by creating the tenant networks. i.e. tests
will verify that the provider segment code added does not impact the proper
actions taken for tenant network test events.

Functional Tests
----------------
Complete unit test coverage of the code is included. Tests include changing
the three configuration variables above and testing for correct results after
issuing create network, subnet and port events.

API Tests
---------
Not applicable.

Documentation Impact
====================

User Documentation
------------------
Deployment documentation on how to configure and deploy this feature
will be documented in the Openstack wiki.

Developer Documentation
-----------------------
None needed beyond documentation changes listed above.

References
==========
ML2 Cisco Nexus WIKI:
  https://wiki.openstack.org/wiki/Neutron/ML2/MechCiscoNexus

Bug review used to commit provider segment support for the core Cisco plugin:
  https://review.openstack.org/#/c/36231/
