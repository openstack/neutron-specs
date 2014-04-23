..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
Neutron OVS DVR - Distributed Virtual Router
============================================

https://blueprints.launchpad.net/neutron/+spec/neutron-ovs-dvr

Neutron Distributed Virtual Router implements the L3 Routers across the
Compute Nodes, so that tenants intra VM communication will occur without
hiting the Network Node. (East-West Routing)

Also Neutron Distributed Virtual Router implements the Floating IP namespace
on every Compute Node where the VMs are located. In this case the VMs with
FloatingIPs can forward the traffic to the External Network without reaching
the Network Node. (North-South Routing)

Neutron Distributed Virtual Router provides the legacy SNAT behavior for
the default SNAT for all private VMs. SNAT service is not distributed, it
is centralized and the service node will host the service.

Problem description
===================

Today Neutron L3 Routers are deployed in specific Nodes (Network Nodes) where
all the Compute traffic will flow through.

Problem 1: Intra VM traffic flows through the Network Node

In this case even VMs traffic that belong to the same tenant on a different
subnet has to hit the Network Node to get routed between the subnets.
This would affect Performance.

Problem 2: VMs with FloatingIP also receive and send packets through the
Network Node Routers

Today FloatingIP (DNAT) translation done at the Network Node and also the
external network gateway port is available only at the Network Node. So any
traffic that is intended for the External Network from the VM will have to
go through the Network Node.

In this case the Network Node becomes a single point of failure and also the
traffic load will be heavy in the Network Node. This would affect the
performance and scalability.

This would also help the Neutron networks to be on par with the Nova parity.

Proposed change
===============

The proposal is to distribute the L3 Routers across the Compute Nodes when
required by the VMs.

In this case there will be Enhanced L3 Agents running on each and every
compute node ( This is not a new agent, this is an updated version of the
existing L3 Agent). Based on the configuration in the L3 Agent.ini file,
the enhanced L3 Agent will behave in legacy(centralized router) mode or
as a distributed router mode.

Also the FloatingIP will have a new namespace created on the specific compute
Node where the VM is located
Each Compute Node will have one new namespace for FIP, per external network
that will be shared among the tenants.
An External Gateway port will be created on the
Compute Node for the External Traffic to Flow through.

Default SNAT functionality will still be centralized and will be running on a
Service Node.

The Metadata agent will be distributed as well and will be hosted on all compute
nodes and the Metadata Proxy will be hosted on all the Distributed Routers.

The existing DHCP server will still run on the Service Node. There are future
plans to distributed the DHCP. ( This will be addressed in a different blueprint)

This implementation is specific to ML2 with OVS driver.

Alternatives
------------
An alternative is to use a Kernel Module. But we did not pursue this since
there was a dependency for the Kernel Module to be part of the upstream
linux distribution before we push this patch.


Data model impact
-----------------

There are couple of minor data model changes that will be
addressed by this blueprint.

1. Router object Data Model.

::
    +----------------+--------------+------+-----+---------+
    |     Field      |    Type      | Null | Key | Default |
    +----------------+--------------+------+-----+---------+
    | tenant_id      | string(256)  | Yes  |     | NULL    |
    | id             | string(36)   | NO   | PRI |         |
    | name           | string(256)  | YES  |     | NULL    |
    | status         | string(16)   | YES  |     | NULL    |
    | admin_state_up | boolean      | YES  |     | NULL    |
    | gw_port_id     | string(36)   | YES  | MUL | NULL    |
    | enable_snat    | boolean      | NO   |     |         |
    | distributed    | boolean      | YES  |     | NULL    |
    +----------------+--------------+------+-----+---------+

Add "distributed" flag to the router object data model. This
will enable the agent to take necessary action based on the
router model.

2. SNAT Agent to host mapping data model.

A new table for the service node enhanced L3 agent to
track the SNAT service on each node.

::
    +------------------+--------------+------+-----+---------+
    | Field            | Type         | Null | Key | Default |
    +------------------+--------------+------+-----+---------+
    | id               | string(36)   | NO   | PRI |         |
    | router_id        | string(36)   | YES  | MUL | NULL    |
    | host_id          | string(255)  | YES  |     | NULL    |
    | l3_agent_id      | string(36)   | YES  | MUL | NULL    |
    +------------------+--------------+------+-----+---------+

REST API impact
---------------

router-create    Create a router for a given tenant.

::
    router-create --name another_router --distributed=true

Admin can only set this attribute. The tenants need not be aware about
this attribute in the router table. So it is not visible to the tenant.

Request

::
    POST /v2.0/routers
    Accept: application/json

    {
    "router":{
    "name":"another_router",
    "admin_state_up":true,
    "distributed":true}
    }


Response

::
    {
    "router":{
    "status":"ACTIVE",
    "external_gateway_info":null,
    "name":"another_router",
    "admin_state_up":true,
    "distributed":true,
    "tenant_id":"6b96ff0cb17a4b859e1e575d221683d3",
    "id":"8604a0de-7f6b-409a-a47c-a1cc7bc77b2e"}
    }


router-show    Show information of a given router.

Request

::
    GET /v2.0/routers/a9254bdb-2613-4a13-ac4c-adc581fba50d
    Accept: application/json

Response

::
    {
    "routers":[{
    "status":"ACTIVE",
    "external_gateway_info":{
    "network_id":""
    },
    "name":"router1",
    "admin_state_up":true,
    "distributed":true,
    "tenant_id":"33a40233088643acb66ff6eb0ebea679",
    "id":"a9254bdb-2613-4a13-ac4c-adc581fba50d"}]
    }

router-update    Create a router for a given tenant.

Admin can only update a centralized router to a distributed router.

Note: Admin can only update a centralized router to a distributed
router and not the other way around. For the first release we are
targeting only from centralized to distributed.

Admin only context:

::
    neutron router-update router1 --distributed=True


Admin only CLI commands:

::
    l3-agent-list-hosting-snat   List L3 agents hosting a snat service.

This command will list the agent with the router-id and SNAT IP.

::
    l3-agent-snat-add            Associate a snat namespace to an L3 agent.

This command will allow an admin to associate a SNAT namespace to an agent.
This command will take the router ID as an argument.

::
    l3-agent-snat-remove         Remove snat association from an L3 agent.

This command will allow an admin to remove or disassociate a SNAT service from
the agent.


Security impact
---------------

Need to make sure the existing FWaaS and the Security Group Rules
are not affected by the DVR.


Notifications impact
--------------------

None


Other end user impact
---------------------

Yes this change will have some impact on the python-neutronclient

The Admin level API proposed above will have to be implemented in
the CLI.

Also there is an impact with Horizon to address the admin level API
mentioned above.

Performance Impact
------------------

* Improves Performance.

Inter VM traffic between the tenant's subnet need not reach the
router in the Network node to get routed and will be routed locally
from the Compute Node. This would increase the performance substantially.

Also the Floating IP traffic for a VM from a Compute Node will directly hit
the external network from the compute node, instead of going through the
router on the network node.

Other deployer impact
---------------------

Global Configuration to enable Distributed Virtual Router.

#neutron.conf

[default]
# To enable distributed routing this flag need to be enabled.
# It can be either True or False.
# If False it will work in a legacy mode.
# If True it will work in a DVR mode.

#router_distributed = True


# ovs_neutron_plugin.ini

# This flag need to be enabled for the L2 Agent to address
# DVR rules

#enable_distributed_routing = True


# l3_agent.ini
#
# This flag is required by the L3 Agent as well to run the L3
# agent in a Distributed Mode.
#
#distributed_agent = True
#

This will be disabled by default.

NOTE: This is for backward compatibility. For migration the admin
might have to run the db-migration script and also re-start the
agents with the right configuration to take effect.

If Cloud admin wanted to enable the feature this can be configured.


It currently uses the existing OVS binary in Linux Distribution. So
there should not be any new binaries.


Developer impact
----------------

Multinode Devstack setup may be required to develop and test.

Services Impact - Some of the services such as the VPN and FW should be
refactored to accomodate the distributed virtual routers. The respective
services team will be working with the DVR team to refactor the services.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <swaminathan-vasudevan>

Other contributors:
  <rajeev-grover>
  <mbirru>
  <michael-smith6>
  <vivekanandan-narasimhan>

Work Items
----------

1. L3 Plugin Extension for DVR

2. ML2 Plugin/OVS Agent for DVR

3. L3 Enhanced Agent for DVR

4. L3 Agent Scheduler for DVR

5. L3 Driver/iplib for DVR


Dependencies
============
OVS (2.01 and above), L2-Pop.


Testing
=======
Yes. Since we are implementing the Distributed Nature of
routers, there need to be multinode setup for testing this
feature so that the rules and actual namespace creation for
the routers can be validated.

Single node infrastructure to test the feature may still be
possible, but we need to validate.

Continuous integration testing to test the dvr at the gate
will be considered.

Documentation Impact
====================

Yes. There will be documentation impact and so documentation
has to be modified to address the new deployment scenario.

References
==========


* https://etherpad.openstack.org/p/Distributed-Virtual-Router

* https://wiki.openstack.org/wiki/Meetings/Distributed-Virtual-Router

* https://blueprints.launchpad.net/neutron/+spec/ovs-distributed-router
