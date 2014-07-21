..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================
Big Switch - Plugin support for OVS agent
=========================================

https://blueprints.launchpad.net/neutron/+spec/bsn-ovs-agent

The Big Switch plugin needs to support deployments that use OVS vSwitches not
connected directly to the backend controller. For this scenario, the plugin
needs to pass a port with a vlan network type and a VLAN to the OVS agent for
the vswitch to be configured with the appropriate VLAN translation rule to
match the fabric's expectation for that segment.

Problem description
===================

The Big Switch plugin currently assumes all of the vswitches will have a direct
connection to the backend controller for the provisioning of rules. Under this
condition there was no need to leverage the OVS agent since connectivity and
segmentation was completely controlled by the rules installed in the vswitches
by the controller.

However, some deployments have incompatible vswitches that can't be changed out
so they need to be controlled by the OpenvSwitch agent. In this scenario, a VLAN
needs to be passed to the openvswitch agent to configure the appropriate VLAN
rewrite rule to match the fabric's configured VLAN for that segment at that
compute node's connection point.


Proposed change
===============

Add support for the the OVS agent to the Big Switch plugin by including the
appropriate RPC classes and by populating RPC port messages with the
appropriate VLAN to configure.

These changes should be contained in the Big Switch Plugin and won't require
any modifications to the OVS Agent.


Alternatives
------------

Modify the Big Switch agent to have the ability to install the VLAN translation
rules.  However, this approach requires extra work because it would require the
same modifications to the plugin in addition to the modifications to the agent.


Data model impact
-----------------

N/A

REST API impact
---------------


Security impact
---------------

N/A

Notifications impact
--------------------

None. All compute nodes already receive port notifications for security group
operations.


Other end user impact
---------------------

N/A

Performance Impact
------------------

N/A


Other deployer impact
---------------------

For compute nodes using OVS without a connection to the controller, the OVS
agent will need to be installed instead of the Big Switch agent.

Developer impact
----------------

N/A


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Kevin Benton (kevinbenton)

Work Items
----------

* Setup RPC code with appropriate callbacks and libraries as necessary
* Add unit tests to make sure port messages are dispatched appropirately
* Add 3rd party CI test scenario

Dependencies
============

This depends on the provider net extension being integrated into the plugin.[1]


Testing
=======

Since this is leveraging the existing agent, the only component that needs to be
tested is the dispatch of correct RPC messages. Unit tests can cover this.
The BSN 3rd party CI system will be modified to test this as well.

Documentation Impact
====================

Deployment guide for Big Switch plugin needs to be updated with option for an
out-of-control OVS bridge.

References
==========

1. https://blueprints.launchpad.net/neutron/+spec/bsn-provider-net
