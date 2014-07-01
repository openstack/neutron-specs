..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Huawei ML2 mechanism driver
==========================================

https://blueprints.launchpad.net/neutron/+spec/huawei-ml2-mechanism-driver

* HW-SDN MD : Huawei SDN Mechanism Driver
* HW-SDN CR : Huawei SDN Controller

The purpose of this blueprint is to build an ML2 Mechanism Driver for Huawei
software define network (SDN) controller, which proxies RESTful calls
(formatted for Huawei SDN controller) from ML2 plugin of Neutron to Huawei
SDN controller.

Huawei SDN controller enables network automation and provision to simplify
virtual machine deployments and move on a larger layer 2 network. When a
cloud administrator provisions VMs, instances of network flow rules are
automatically created and applied to the OpenvSwitch(OVS) which hosted on
each compute node. As VMs move across the compute nodes, the network flow
rules are automatically applied to each OVS.

Problem description
===================

Huawei SDN controller requires information of OpenStack Neutron based networks
and ports to manage virtual network appliances and OVS flow rules.

In order to recieve such information from neutron service, a new ML2 mechanism
driver is needed to post the _postcommit data to Huawei SDN controller.

The following sections describe the proposed changes in Neutron, a new ML2
mechanism driver, and make it possible to use OpenStack in Huawei SDN
topology. The following diagram depicts the OpenStack deployment in Huawei
SDN topology.

Huawei SDN Topology::

  +-----------------------+              +----------------+
  |                       |              |                |
  |     OpenStack         |              |                |
  |     Controller        |              |                |
  |     Node              |              |                |
  |                       |              |                |
  | +---------------------+              |   Huawei SDN   |
  | |Huawei SDN mechanism | REST API     |   controller   |
  | |driver               |--------------|                |
  | |                     |              |                |
  +-+--------+-----+------+              +--+----------+--+
             |     |                        |          |
             |     |                        |          |
             |     +--------------+         |          |
             |                    |         |          |
  +----------+---------+      +---+---------+------+   |
  |                    |      |                    |   |
  |      OVS           |      |      OVS           |   |
  +--------------------+ ---- +--------------------+   |
  | OpenStack compute  |      | OpenStack compute  |   |
  | node 1             |      | node n             |   |
  +----------+---------+      +--------------------+   |
             |                                         |
             |                                         |
             +-----------------------------------------+

As shown in the diagram above, each OpenStack compute node is connected
to Huawei SDN controller, which is responsible for provisioning, monitoring
and troubleshooting of cloud network infrastructures. The Neutron API requests
will be proxied to SDN controller, then network topology information can be
built. When a VM instance starts to communicate with another, the first packet
will be pushed to Huawei SDN controller by OVS, then the flow rules will be
calculated and applied to related compute nodes by SDN controller. Finally,
OVS follows the rules to forward packets to the destination instance.

Proposed change
===============

The requirements for ML2 mechanism driver to support huawei SDN controller
are as follow:

1. SDN controller exchanges information with OpenStack controller node by
using REST API. To support this, we need a specific client.

2. OpenStack controller (Neutron configured with ML2 plugin) must be
configured with SDN controller access credentials.

3. The network, subnet and port information should be sent to
SDN controller when network or port is created, updated, or deleted.

4. SDN controller address should be set on OVS. SDN controller will detect
port change and calculate flow tables based on network information sent from
OpenStack controller. These flow tables will be applied on OVS on related
compute nodes.

Huawei Mechanism driver handles the following postcommit operations.

Network create/update/delete
Subnet  create/update/delete
Port    create/delete

Supported network types include vlan and vxlan.

Huawei SDN mechanism driver handles VM port binding within the mechanism
driver.

'bind_port' function verifies the supported network types (vlan, vxlan)
and calls context.set_binding with binding details.

Huawei SDN Controller manages the flows required on OVS, so we don't have
an extra agent.

Sequence flow of events for create_network is as follow:

::

 create_network
 {
   neutron    ->  ML2_plugin
   ML2_plugin ->  HW-SDN-MD
   HW-SDN-MD  ->  HW-SDN-CR
   HW-SDN-MD  <-- HW-SDN-CR
   ML2_plugin <-- HW-SDN-MD
   neutron    <-- ML2_plugin
 }

Port binding task is handled within the mechanism driver, So OVS mechanism
driver is not required when this mechanism driver is enabled.

Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

Recently a feature of enabling OVS secure mode was added to the OVS agent.
Huawei SDN controller doesn't rely on the OVS agent but secure mode will
be enabled when deploying Huawei SDN controller and OVS.

Notifications impact
--------------------

None

Other end user impact
---------------------

This change doesn't take immediate effect.

1. Configuration parameters regarding SDN (such as ip address,...) should be
added to the mechanism driver configuration file.

Update /etc/neutron/plugins/ml2/ml2_conf_huawei.ini, as follow:

::

 [ml2_Huawei]
 nos_host = 128.100.1.7
 nos_port = 8080

2. An SDN controller account should be created for OpenStack to access, also
this account should be added to the mechanism driver configuration file.

Update /etc/neutron/plugins/ml2/ml2_conf_huawei.ini, as follow:

::

 [ml2_Huawei]
 nos_username = admin
 nos_password = my_password

Performance Impact
------------------

There are create/update/delete_<resource>_postcommit functions to proxy
those requests to SDN controller in the ML2 mechanism driver. All those
processes require database access in SDN controller, which may impact the
Neutron API performance a little.

Other deployer impact
---------------------

This change doesn't take immediate effect.

1. Add new configuration options for SDN controller, which are ip address
and credentials.

Update /etc/neutron/plugins/ml2/ml2_conf_huawei.ini, as follow:

::

 [ml2_Huawei]
 nos_host = 128.100.1.7
 nos_port = 8080
 nos_username = admin
 nos_password = my_password

2. Configure parameters of section ml2_type_vxlan in ml2_conf.ini, setting
vni_ranges for vxlan network segment ids and vxlan_group for multicast.

Update /etc/neutron/plugins/ml2/ml2_conf.ini, as follow:

::

 [ml2_type_vxlan]
 vni_ranges = 1001:2000
 vxlan_group = 239.1.1.1

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  yangxurong

Work Items
----------

1. Change the setup.cfg to introduce 'huawei' as the mechanism driver.
2. An REST client for SDN controller should be developed first.
3. Mechanism driver should implement create/update/delete_resource_postcommit.
4. Test connection between two new instances under different subnets.

Dependencies
============

None

Testing
=======

1. The whole setup can be deployed using OVS and SDN controller can be deployed
in VM.
2. For each module added to the mechanism driver, unit test is provided.
3. Functional testing with tempest will be provided. The third-party Huawei CI
report will be provided to validate this ML2 mechanism driver.

Documentation Impact
====================

Huawei SDN mechanism driver description and configuration details will be added.

References
==========

https://review.openstack.org/#/c/68148/
