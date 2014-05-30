..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================================
Cisco Dynamic Fabric Automation (DFA) ML2 Mechanism Driver
==========================================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/ml2-mechnism-driver-for-cisco-dfa

The purpose of this blueprint it to build an ML2 Mechanism Driver for DFA.

Problem description
===================

Cisco Dynamic Fabric Automation (DFA) helps enable network automation and
provisioning, to simplify both physical servers and virtual machine
deployments and moves across the fabric. It uses network admin-defined
profile templates for physical and VM projects.
When a server admin provisions VMs and physical servers, instances of
network policies are automatically created and applied to the network leaf
switch. As VMs move across the fabric, the network policy is automatically
applied to the leaf switch. More details of cisco DFA can be found in [1].

The following sections describe the proposed changes in neutron, a new
ML2 mechanism driver, and make it possible to use openstack in such a
topology.

This diagram depicts openstack deployment in Cisco DFA topology.

asciiflows::

                               XXXXXXXXXXXXX
                        XXXXXXX             XXXXXXXXX
                       X +–––––––+         +––––––––+ X
                      X  |spine 1| ++  ++  |spine n |  X
  +–––––––––+       X    +–––––––+         +––––––––+   X
  |DCNM     |      X                                     X
  |(Data    |     X                                       X
  | Center  |    +––––––––+                               X
  | Network +––––+ Leaf i |     SWITCH FABRIC            X
  | Manager)|    +––––––––+                              X
  +–––––+–––+       X                                   X
  |                  X                                 X
  |                    X+–––––––+            +–––––––+X
  +                     | Leaf x|XXXX XX XXXX|Leaf z |
  |                     |  +----+            |   +---+
  REST API              |  | VDP|            |   |VDP|
  +                     +––+–––++            +––++---+
  |    +––––––––––––––+        |                |
  |    |   Openstack  |        |                |
  |    |   Controller |        |                |
  |    |   Node       | +––––––+––––––––+    +––+––––––––––––+
  |    |  +–––––––––+ | | OVS           |    | OVS           |
  |    |  |Cisco DFA| | +–––+-–-––––––––+    +–––+–-–––––––––+
  +–––––––+Mechanism| | |   |LLDPad(VDP)|    |   |LLDPad(VDP)|
       |  |Driver   | | |   +---––––––––+ ++ |   +--–––––––––+
       |  |         | | | Openstack     |    | Openstack     |
       +––+––––+––––+–+ | Compute node 1|    | Compote node n|
               |        +––––––––––+––––+    +––––––––––+––––+
               |                   |                    |
               +–––––––––––––––––––+––––––––––––––––––––+

As shown in the diagram above, each openstack compute node is connected
to a leaf node and controller node is connected to a DCNM (Data Center
Network Manager). The DCNM is responsible for provisioning, monitoring and
troubleshooting of data center network infrastructures [2].

Proposed change
===============

The requirements for ML2 Mechanism Driver to support cisco DFA are as follows:

1. DCNM exchanges information with openstack controller node using REST API.
To support this, a new client should be created to handle the exchange.

2. When a project is created/deleted the same needs to be created/deleted on
the DCNM.

3. New extension, dfa:cfg_profile_id, needs to be added to network resource.
Configuration profile can be thought as extended network attributes of the
switch fabric.

4. Periodic should be sent to DCNM to get all supported
configuration profile and save it into a database.

5. The network information, such as subnet, segmentation ID, and configuration
profile should be sent to DCNM, when network is created, deleted or updated.

6. Mechanism Driver needs to exchange information with OVS neutron agent when a
VM is launched. Currently only plugin can have RPC call to ovs neutron agent.
Mechanism driver need to setup RPC to exchange the information with ovs agent.

7. The DFA topology represent a network based overlay and it support more than
4k segment. So new type driver is needed to handle this type of network and
detailed is explained in [4].

8. Data path (OVS flow) should be programmed on compute nodes where instances
are launched. The VLAN for the flow is obtained from an external process, that
is LLDPad, running on computed node. More detail of process explained in [4].

Alternatives
------------

None

Data model impact
-----------------

* Configuration profiles - keeps supported config profile by DCNM.
* Configuration profile and network binding - keeps mapping between network
  and config profile.
* Project - keeps mapping between project name and project ID.

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

1. As it is mentioned before, when a network is created, list of configuration
profile needs to be displayed. From horizon GUI or CLI, the information can
be requested using list_config_profiles API, which will be added to the
python-neutronclient.

2. Configuration parameters regarding DCNM (such as ip address,...) should be
added to mechanism driver config file.

Performance Impact
------------------

There are two options to query configuration profiles from DCNM, periodic and
on demand.
The on demand request may cause performance issue on create_network. As the
reply to a request, has to be processed and it also include database access.
On the other hand with periodic approach the information may not be available
for duration of pooling time. Concerning performance, a periodic task can
query and process the information.

There are create/update/delete_<resource>_post/precommit methods in the ML2
mechanism driver. All the access to database for DFA mechanism driver is done
in the precommit methods and postcommit methods are handling DCNM requests.

Other deployer impact
---------------------

1. New configuration options for DCNM, that is ip address and credentials.
2. Enabling notification in keystone.conf.
3. Adding new config parameter to ml2_conf.init to define RPC parameters
   (i.e. topic's name) for neutron ovs agent and mechanism driver.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Nader Lahouti (nlahouti)

Work Items
----------

1. Change the setup.cfg to introduce 'cisco_dfa' as mechanism driver.

2. Events notifications should be enabled in keystone.conf.
The mechanism driver relies on notification for project events (create/delete).
The ‘cisco_dfa’ mechanism driver listens to these events and after processing
them it sends request to the DCNM to create/delete project.
'cisco_dfa' keep the projects info in a local database, as they will be used
when sending delete request to DCNM.

3. Spawn a periodic task to send request to DCNM. The reply contains
configuration profiles. The information will be saved in a database.
If connection to DCNM fails invalidate the cached information.

4. Define new extension to network resource for configuration profile. The
extensions will be added to supported aliases in the cisco_dfa mechanism
driver class.

NOTE: ML2 plugin currently does not support extension in the mechanism driver.
A new blueprint is opened to address this issue [5].

5. When an instance is created cisco_dfa needs to send information (such as
instance’s name), to the external process (i.e. LLDPad) on the compute nodes.
For that purpose RPC is needed to call an API on ovs_neutron_agent side.
Then the API pass the information to LLDpad through a shared library
(provided by LLDPad)

Dependencies
============

1. Changes in ovs_neutron_agent to program the flow [4].
2. Need implementation of RPC for mechanism driver in ovs_neutron_agent.
3. Support for extensions in ML2 mechanism drivers [5]

Testing
=======

It is needed to have a setup as shown in the beginning of this page.
It is not mandatory to have physical switches for that topology.
The whole setup can be deployed using virtual switches (i.e. instead of
having physical switch fabric, it can be replaced by virtual switches).
For each module added to the mechanism driver, unit test is provided.
Functional testing with tempest will be provided. The third-party Cisco DFA CI
report will be provided to validate this ML2 mechanism driver.

Documentation Impact
====================

Describe cisco DFA mechanism driver and configuration details.

References
==========
[1] http://www.cisco.com/go/dfa
http://www.cisco.com/c/en/us/solutions/collateral/data-center-virtualization/unified-fabric/white_paper_c11-728337.html
http://www.cisco.com/c/en/us/solutions/data-center-virtualization/unified-fabric/dynamic_fabric_automation.html#~Overview

[2] http://www.cisco.com/c/en/us/products/cloud-systems-management/prime-data-center-network-manager/index.html

[3] https://blueprints.launchpad.net/horizon/+spec/horizon-cisco-dfa-support

[4] https://blueprints.launchpad.net/neutron/+spec/vdp-network-overlay

[5] https://blueprints.launchpad.net/neutron/+spec/neutron-ml2-mechanismdriver-extensions
http://summit.openstack.org/cfp/details/240
