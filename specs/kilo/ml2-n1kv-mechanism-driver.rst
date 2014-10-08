..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================
ML2 Mechanism Driver for Cisco Nexus1000V switch
================================================

https://blueprints.launchpad.net/neutron/+spec/ml2-n1kv-mechanism-driver

The purpose of this blueprint is to add support for Cisco Nexus1000V switch
in OpenStack neutron as a ML2 mechanism driver.

Problem Description
===================

Cisco Nexus 1000V for KVM is a distributed virtual switch that works with
the Linux Kernel-based virtual machine (KVM) open source hypervisor.

* Virtual Supervisor Module (VSM): Controller of the Cisco Nexus1000V
  distributed virtual switch based on Cisco NX-OS software.

* Virtual Ethernet Module (VEM): Data plane software installed on the Compute
  nodes. All VEMs are controlled by VSM.

* Network Profiles: Container for one or more networks. VLAN and VXLAN
  type of network profiles will be supported in the initial version.

* Policy Profiles: Policy profiles are the primary mechanism by which network
  policy is defined and applied to VM ports in a Nexus 1000V system.

* VM Network: VM Network refers to a combination of network-segment and
  policy-profile. It maintains a count of ports that use the above
  combination.

This blueprint proposes the first step in migration of the Cisco Nexus1000V
monolithic plugin to a mechanism driver for ML2 plugin. This mechansim driver
will interact with the VSM via REST APIs to dynamically configure and manage
networking for instances created via OpenStack.

Proposed Change
===============

The diagram below provides a high level overview of the interactions between
Cisco Nexus1000V switch and OpenStack components.

Flows::

         +--------------------------+
         | Neutron Server           |
         | with ML2 Plugin          |
         |                          |
         |             +------------+
         |             |   N1KV     |
         |             | Mechanism  |                 +--------------------+
 +-------|             |  Driver    |                 |                    |
 |       |           +-+--+---------+    REST API     |     Cisco N1KV     |
 |   +---+           | N1KV Client  +-----------------+                    |
 |   |   +-----------+--------------+                 |     Virtual        |
 |   |                                                |     Supervisor     |
 |   |                                                |     Module         |
 |   |   +--------------------------+                 |                    |
 |   |   |        N1KV VEM          +-----------------+                    |
 |   |   +--------------------------+                 +-+------------------+
 |   |   |                          |                   |
 |   +---+       Compute 1          |                   |
 |       |                          |                   |
 |       +--------------------------+                   |
 |                                                      |
 |                                                      |
 |       +--------------------------+                   |
 |       |        N1KV VEM          +-------------------+
 |       +--------------------------+
 |       |                          |
 +-------+       Compute 2          |
         |                          |
         +--------------------------+

The Cisco Nexus1000V mechanism driver will handle all the postcommit events
for network, subnets and ports. This data will be used to configure the VSM.
The VSM and VEM will be responsible for port bring up on the compute nodes.

The mechanism driver will initialize a default network profile and a policy
profile on the VSM. All networks will have a binding to this default
network profile. All ports will have a binding to this default policy profile.
Dynamic binding of network and policy profiles will be implemented in the
future once extensions are supported for mechanism drivers.

Data Model Impact
-----------------

This mechanism driver introduces four new tables which are specific to the
N1KV mechanism driver.

 * NetworkProfile: Stores network profile name and type.
 * N1kvNetworkBinding: Stores the binding between network profile and
   network.
 * PolicyProfile: Stores policy profile name and UUID.
 * N1kvPortBinding: Stores the binding between policy profile and
   port.

A database migration is included to create the tables for these models.

A script to migrate data for users of monolithic plugin will be provided in
the next iteration of this driver when we add support for extensions.

No existing models are changed.

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

The performance of Cisco N1KV ML2 mechanism driver will depend on the
responsiveness of the VSM.

Other Deployer Impact
---------------------

The deployer must provide the following in order to be able to connect to a
VSM.

* IP address of VSM.
* Admin credentials (username and password) to log into VSM.
* Add "cisco_n1kv" as a ML2 driver.
* Add the "vlan" type driver.
* Add the "vxlan" type driver.

These should be provided in:
/opt/stack/neutron/etc/neutron/plugins/ml2/ml2_conf_cisco.ini

Example:
[ml2_cisco_n1kv]

# N1KV Format.
# [N1KV:<IP address of VSM>]
# username=<credential username>
# password=<credential password>


The Nexus1000V monolithic plugin will be deprecated once the Nexus1000V ML2
mechanism driver achieves feature parity with the monolithic plugin.
Feature parity denotes full support for existing Nexus1000V specific resource
and attribute extensions.


Developer Impact
----------------

None.

Community Impact
----------------

This change adds a new mechanism driver for ML2 plugin aligning with the
community direction.

Alternatives
------------

Alternative is to use the existing monolithic Nexus1000V plugin.
We add this blueprint in order to interoperate with other mechanism drivers
and align with community direction.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <abhraut>

Other contributors:
  <sthillma>

Work Items
----------

Work Items can be roughly divided into the following tasks:
* Mechanism driver to handle network/subnet/port CRUD requests.
* N1KV Client to perform HTTP requests to the VSM.
* Unit test cases to test the mechanism driver and client code.
* Tempest test cases to peform functional testing.


Dependencies
============

Following third party library used:

 * requests: Requests is a python library for making HTTP requests which is
   well documented at http://docs.python-requests.org/en/latest/
   Link to code -> https://github.com/kennethreitz/requests


Testing
=======


Tempest Tests
-------------

Third party testing will be provided. The Cisco CI will report on all changes
affecting this mechanism driver. The testing will run on a setup with an
OpenStack deployment connected to a VSM and VEM.

Functional Tests
----------------

New functional tests will be added by mocking the VSM responses.

API Tests
---------

None.


Documentation Impact
====================


User Documentation
------------------

Configuration details for this mechanism driver.

Developer Documentation
-----------------------

None.

References
==========

http://www.cisco.com/go/nexus1000v
