..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================
Strict minimum bandwidth support (egress)
=========================================

Problem Description
===================

RFE link: https://bugs.launchpad.net/neutron/+bug/1578989

Newton release added minimum bandwidth support to QoS policy
rules. While this improvement works at the hypervisor level,
it's a best effort measure that will not prevent over-subscription
on a specific network link bandwidth (When SUM i port[i].min_bw >
max available bandwidth within a provider network).

NFV/Telcos are interested in these types of rules to make sure
functions don't over commit compute nodes.

Cloud Service Providers could make use of this feature to provide bandwidth
guarantees for streaming.

Proposed Change
===============

To provide a strict constraint we need scheduling cooperation with
Nova through integration with the new Nova placement API [1][2].

Neutron L2 agents, or the specific plugin/backend will be responsible
for registering per compute resources into the placement API. L2 agents
are proposed to take this responsibility, since it mimics the design
decision in Nova where nova-compute reports resources to such API.
This has the benefit of distributing the workload across all the nodes
instead having a central service responsible of syncing to the placement
API.

The resources will have the following form, please note INGRESS is
provided as a reference but it's out of the scope of this spec:

  NIC_BW_EGRESS.<physical-network>
  NIC_BW_INGRESS.<physical-network>

physical-network will be the "physnet" in the reference implementation,
or "tunneling" in the case of requesting bandwidth on the tunneling
path.

While modeling the tunneling bandwidth is not a simple thing, because it
could vary based on host routers and interfaces, we will provide this
simple model to start with.

Nova will use those details to count down and manage the inventories
at its side.

When Nova creates, updates or gets a port, Neutron will show the
resource counts needed by the port in the response, removing any
dependency from Nova into the Neutron QoS policy rules::

    {"port": {
        "status": "ACTIVE",
        "name": "",
        ...
        "device_id": "5e3898d7-11be-483e-9732-b2f5eccd2b2e",
        "resources": { "NIC_BW_EGRESS.ext-net": 1000, /*kbps*/
                       "NIC_BW_INGRESS.ext-net": 1000}
    }}


This extra dictionary could be incorporated to the QoS extension itself,
but it makes sense to make something agnostic to the QoS extension that
other resource aware extensions could make use of.

There is a common architecture in network configuration that we need to
address. It's very common to bond several interfaces, and pass each
specific physnet or even tunneling through a VLAN on such bonding.

To properly handle that case is where we need to make virtual physnets
handling the nesting, for example by providing a configuration
like this per-agent::

    [qos]

    virtual_physnets = bond0phys:bond0, bond1phys:bond1

    nested_physnets = bond0phys:physnetA, bond0phys:physnetB, \
                      bond0phys:tunneling, \
                      bond1phys:physnetC, bond1phys:physnetD


In such case the <physical-network> part of the resources to nova would
be substituted by bond0phys for physnetA / physnetB / tunneling and
by bond1phys for physnetC and D.

The extra level of indirection is provided by virtual_physnets so
interface name changes won't de-sync the nova resources and allocations.

In this specific case, the total bandwidth will be collected from bond0 and
bond1 and reported to NIC_BW_{EGRESS,INGRESS}.{bond0phys,bond1phys}, see
work item 1.

Dependencies
------------

We depend on several Nova changes which will allow the definition
of custom resource classes [4].

Work Items
----------

1) Collecting available bandwidth from physnets on the agents
   with the ability to override bandwidth in config.
   Asignee: Rodolfo Alonso <rodolfo.alonso.hernandez@intel.com>

2) Write an API for sending collected bandwidth information to the
   Nova placement API. For the in-tree drivers this information can be
   sent right away from the agents to avoid the neutron-server bottleneck
   problem. But out-of-tree drivers would have the freedom to use such API from
   the QoS driver itself by, for example periodically syncing with the backend
   to grab bandwidth information. Or the out-of-tree drivers could choose to
   send the placement information from the backend itself to nova.
   Assignee: Miguel Lavalle

3) Nova placement API client (custom resource classes [4])
   Initial nova placement API client has already been merged [3]
   Assignee: Miguel Lavalle (he's handling it for Routed Networks)

4) Making the resources field available on port create/update/get for nova.
   Assignee: Slawek Kaplonski

References
==========
[1] http://docs.openstack.org/developer/nova/placement.html
[2] http://docs.openstack.org/developer/nova/placement_dev.html
[3] https://review.openstack.org/#/c/414726
[4] https://review.openstack.org/#/q/topic:bp/custom-resource-classes
[5] https://github.com/openstack/nova/tree/master/nova/scheduler/client
