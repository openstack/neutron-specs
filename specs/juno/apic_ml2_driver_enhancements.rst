================================================================
Apic ML2 driver enhancements
================================================================

https://blueprints.launchpad.net/neutron/+spec/apic-driver-enhancements

This blueprint keeps track of some improvements needed by the APIC ML2 Driver.
Specifically, the target enhancements will be:

Refactoring of the apic-manager:
Making it more robust and it's operations idempotent,
being also able to "fix" any incoherence that may happen between the APIC
and the Neutron world;

Name mapping:
Defining configurable naming strategy for APIC. This allows the end-user
to see names instead of uuids for APIC resources in APIC GUI;

External Gateway:
Enable router external gateway feature for the APIC plugin;

Dynamic Topology Discovery:
Enable the ability for the APIC plugin to automatically discover
the physical topology (eg. compute host attached interfaces).
This allows for less manual configuration from the user.

Support for VPC/LACP (bonding):
Enable the use of openstack servers connected in a
redundant topology with VPC/LACP (port bonding).

Problem description
===================

This blueprint is targeting a number of small enhancements
to solve different problems.

High Availability:
Current APIC ML2 driver does not support a High Availability
and Synchronization strategy for APIC, making it hard to recover
from failures of the Neutron server. Also, the driver doesn't
leverage APIC management address redundancy/fallback mechanism,
which translates into plugin failures every time the single APIC
controller address in it's configuration is not reachable.

Name Mapping:
At the current state, creating objects in the neutron model is
translated into an APIC model always named by the object UUID.
A alternative name mapping strategy should be provided to the user.

L3 External Gateway Support:
Today, the APIC L3 driver doesn't support the external gateway
extension for routers.

Dynamic Topology Discovery:
The APIC drivers discovers the topology by leveraging some static
Neutron configuration. Ideally we want to be able to automatically
discover the topology by the use of a new agent.

Support for VPC/LACP (bonding):
The current APIC driver does not support VPC/LACP or bonding of
ports. This is a desirable configuration for HA deployments
that we want to support.

Proposed change
===============

High Availability:
The changes for this point can be summarized into two major items:

First, modifying the current APIC driver configuration so that the
user can specify as many APIC controllers as he likes. The APIC
client will then rotate among these addresses every time there's a
connection failure.

To improve the manager's operations idempotence and reliability, a
transactional model is introduced which leverages the ability of
the APIC to run operations on a Model's full subtree transactionally.

Name Mapping:
A new DB table will be added to keep track of the "APIC Name" used
by a specific neutron object's ID. Every time the driver is called,
the mapper will be used to retrieve the actual names in the backend.

L3 External Gateway Support:
Every time a port with device_owner "router:gateway" is created, the
mechanism driver will run all the needed operations in order to
guarantee external connectivity to the proper networks. Some
configuration may be needed to describe the external router.

Dynamic Topology Discovery:
An APIC specific agent is proposed. It will be installed as a 3rd
party agent on all openstack nodes that need to be connected to the
APIC fabric. The agent will listen to LLDP advertisements, and on
discovering a topology change, it will update the APIC with the
appropriate interface mapping.

Support for VPC/LACP (bonding):
VPC pairs are defined in the plugin configuration. When an interface
from a switch that is in a VPC pair is detected, the appropriate
fabric configuration to enable that interface to work as a part of
VPC/LACP bond is created on the APIC.

Alternatives
------------

None

Data model impact
-----------------

Following new models are created by this driver. Note that these are
specific to this driver:

* ApicName: Tracks the name mapping.
* ApicKey: Tracks persistent config on APIC that needs to be in sync with driver
* HostLink: Tracks connectivity of host links.

There are also changes on the existing model. Again, everything
is APIC specific:

* NetworkEPG: Deleted;
* PortProfile: Deleted;
* TenantContract: Becomes RouterContract, since the contract will now be considered by Router and no by Tenant.

A database migration is included to create the tables for these
models.

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

There is potentially a new configuration (for VPC pairs), if
that feature is used.
Also, if dynamic topology discovery is used, the agent should
be added to all the compute node, while the service will be
likely installed on the cloud controller.

Performance Impact
------------------

The transactional model for the APIC manager does a lot of work
"locally", and only 1 rest call at the end of each transaction.
This will improve the responsiveness of the plugin.

Other deployer impact
---------------------

The configuration of host connectivity is no longer required in
the configuration file.


Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Ivar Lazzaro (mmaleckk)

Mandeep Dhami (mandeep-dhami)

Work Items
----------

* APIC Mechanism driver package;
* APIC L3 driver package;
* Unit Tests Code (APIC Specific).

Dependencies
============

None

Testing
=======

Unit Test

Documentation Impact
====================

Configuration Reference guide will be updated.

References
==========
