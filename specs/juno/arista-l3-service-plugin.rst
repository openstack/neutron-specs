==========================================================================
Layer 3 service plugin to support hardware based routing on Arista devices
==========================================================================

https://blueprints.launchpad.net/neutron/+spec/arista-l3-service-plugin

This blueprint is to implement a L3 service plugin to support hardware based
Layer 3 routing in Arista devices.


Problem description
===================

This service plugin implements neutron L3 routing features on Arista hardware.
It communicates with Arista hardware fabric over JSON RPC to automate
the provisioning of routing on Arista devices (both at leaf and spine layer).


Proposed change
===============

This proposal is to introduce a new Layer 3 service plugin that uses JSON RPC to
to communicate with Arista hardware fabric to provide full L3 routing functionality.

This plugin works in conjuction with Arista ML2 Driver, which manages the
networks, subnets, and ports.
This service plugin and ML2 Driver leverage the resources and topology knowledge
to facilitate the automation of L2 and L3 provisioning in Arista hardware.

Alternatives
------------

The alternative approach is to use the open source agent based layer 3 router
plugin and update the L3 agent to interact with Arista Hardware.

Data model impact
-----------------
None

REST API impact
---------------
n.a

Security impact
---------------
n.a

Notifications impact
--------------------
n.a.

Other end user impact
---------------------
n.a.

Performance Impact
------------------

This service plugin is event based triggered. No polling is deployed.
There are no changes to any existing code patterns. Note that the events here
means invocation of the APIs.
This plugin uses JSON RPC to communicate with Arista hardware. Arista ML2 Driver
uses the same mechanism and implements bulk operations to support scale
deployements. This plugin will leverage the same mechanisms.

Other deployer impact
---------------------

Configuration knobs will be provided to the adminstrators to take advantage of
various deployment topologies supported by Arista fabric. These configuration
knobs will be documented on OpenStack wiki.

Developer impact
----------------
n.a.


Implementation
==============

Assignee(s)
-----------

Sukhdev Kapur
sukhdev@arista.com
IRC - Sukhdev

Work Items
----------

* L3 Service Plugin
* DevStack related enhancements to support this plugin

Dependencies
============

None

Testing
=======

Complete unit test coverage of the code will be provided.

For full tempest test coverage, Arista third party testing is implemented
and is described at the following wiki.
With implementation of this Plugin, the test list will be updated. For example,
Floating IP related tests will be removed and L3 routing function tests will be
included. The updated list of tests will be published.

https://wiki.openstack.org/wiki/Arista-third-party-testing


Documentation Impact
====================
Documentation to configure and deploy this service plugin will be provided
in the Openstack wiki.


References
==========

n.a.
