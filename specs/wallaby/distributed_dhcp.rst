..
     This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Distributed DHCP for Openvswitch Agent
======================================

RFE: https://bugs.launchpad.net/neutron/+bug/1900934

Neutron DHCP agents and the scheduled network instances are relatively
simple in function. But the configuration is complex, and it depends
on external process (dnsmasq) and namespace. When the user's demand is
merely unique, for example, they only need the DHCP response during the
virtual machine booting process, then the existing DHCP agent and its
configuration procedure for network and port make things complicated.

This spec describes how to implement a DHCP extension for Neutron
openvswitch agent to achive a simple and efficient solution for virtual
machine DHCP function by leveraging the openflow with openvswitch.

Problem Description
===================

Response the DHCP request is the main function for the scheduled network
instance of DHCP agent. Except this, it has other functions, like
isolated metadata and DNS lookup. But there are alternatives for these
extended functions, such as config drive for metadata and Designate
for DNS.

Then, the use frequency of the DHCP agent and its scheduled instance are
relatively low. It will be used only during the VM booting if no DNS lookup.
If you use config drive, aslo the scheduled DHCP instance is not useful.

And we have more problems for large scale clusters:

* The scheduled network instances of DHCP port increase the consuming of
  L2 agent's capacity and performance.
* The DHCP provisioning block sometimes causes virtual machine booting
  failure.
* Full sync in unknown reason causes the message queue, neutron-server and
  DB in high load.
* DVR local router creation for the scheduled DHCP port is default behavior,
  which increased resource load for L2 and L3 agents implicitly [1]_.
* Down a DHCP node causes long time recovery due to it has tons of scheduled
  network instance.

And it is hard to find the balance between ``how many DHCP agents should the
deployment have`` and ``how many resources could one agent handle``. Too few
DHCP agent will finally make each one agent has a huge mount of resources.
Too many DHCP agents will increase maintenance pressure for the operators,
and make the centralized components overloaded.

There is a way to schedule one network to all DHCP agents on all compute
nodes. Firstly, this may work for tiny deployment with extremely few resources.
For large-scale deployment, it is basically impossible. Because there will be
tens of thousands of Networks in Neutron, this will directly lead to a surge of
resource pressure on each node, consume too many IP in user network,
the results basically are unable to operate, virtual machine startup failure
and unable to obtain IP and so on.

Proposed Change
===============

A new extension of Neutron openvswitch agent will be added to achieve the
``Distributed DHCP``.

.. note:: This extension is only for openvswitch agent, other mechanism drivers
          will not be considered, because this new extension will rely on the
          openflow protocol and principle. For OVN, it has supported similar
          DHCP local response mechanism.

Solution Proposed
-----------------

As we know Neutron openvswitch agent has the entire information of the ports
which are pluged to the ovs bridge (If the port information was not
synchronized by the ovs-agent, there is a simple cache pull mechanism which
will fill the information). Neutron has the all conditions to support
distributed DHCP natively:

* ovs-agent is based on the python SDN controller ryu/os-ken
* ovs-agent is fully distributed
* ovs-agent has the entire resource information

So we can assume the Neutron openvswitch agent is a local SDN controller which
will try to response the VM's DHCP request. The basical data pipeline can be
described as this:

::

                            +---------+                +---------------------+
    +-------+ DHCP Request  |         |    packet-in   |  +----------------+ |
    |  VM   +--------------->  Flows  +---------------->  |   os-ken app   | |
    |       <---------------+         <----------------+  |                | |
    +-------+ DHCP Response |         |    packet-out  |  | DHCP Responder | |
                            |         |                |  +----------------+ |
                            | br-int  |                |      OVS-agent      |
                            +---------+                +---------------------+

After this we will have:

* Higher level availability than DHCP agent with it's schedlued ``dnsmasq``,
  DHCP requests are directly processed in the
  computing nodes, it is completely distributed.
* No DHCP agent and its scheduling mechanism anymore
* No extra external process for DHCP anymore
* The (Neutron openvswith) agent downtime will no longer affect the
  address acquisition and virtual machine startup in other nodes.
* Virtual machine startup will no longer be affected by port's DHCP
  configuration, which reduces the probability of VM spawning failure.
* DHCP request and reponse for VM will achieve a high success rate.

DHCP(v4/v6) protocol options
++++++++++++++++++++++++++++

We will not repeat the DHCPv4 [2]_ and DHCPv6 [3]_ in details. But for
this spec, we will ensure the following features of the related protocol:

* DHCPv4 all message types
* DHCPv4 host Configurations
* DHCPv6 types Solicit, Advertise, Confirm, Renew, Rebind, Release and Reply
* DHCPv6 Options

For Neutron, we have some Port attributes like dns_domain, dns_name and
extra_dhcp_opts, and Subnet dns_nameservers, host_routes and gateway_ip
and so on, all these options will be added to the final DHCP
response like dnsmasq do.

Server side changes
+++++++++++++++++++

Based on the new added config options, some DHCP related DB options,
APIs, notifications and RPCs will be changed to no operation or be skipped.
The config options only controls Neutron itself, the DHCP protocol will have
no effect on. The final goal is to support the full features of DHCP protocol
defined. The changes are:

* disable DHCP scheduling mechanism and its failover forever
* disable DHCP provisioning block
* disable DHCP related RPCs and notificatons

OpenvSwitch Agent side changes
++++++++++++++++++++++++++++++

For Neutron openvswitch agent, we will add a new agent extension which will
process the basical flow installation for each port's further DHCP request.

There will be two basic flows which will direct DHCPv4 and DHCPv6 to independent tables.
``table 77`` is for DHCPv4, ``table 78`` is for DHCPv6. The flows are:

::

  table=60, priority=101,udp,nw_dst=255.255.255.255,tp_src=68,tp_dst=67 actions=resubmit(,77)
  table=60, priority=101,udp6,ipv6_dst=ff02::1:2,tp_src=546,tp_dst=547 actions=resubmit(,78)

For table 77, each DHCP request will be checked to verify the source mac and in_port in order
to avoid the DHCP spoofing. If the DHCP request is matched, then submit it to the controller,
aka the Neutron openvswitch agent. Any unmatched packets will be dropped. One example for a
VM's port is:

::

  table=77, priority=100,udp,in_port="tapcc4f2da4-c5",dl_src=fa:16:3e:46:58:fe,tp_src=68,tp_dst=67 actions=CONTROLLER:0
  table=77, priority=0 actions=drop

For table 78, DHCPv6 match and drop flows structure are basically same to DHCPv4:

::

  table=78, priority=100,udp6,in_port="tapcc4f2da4-c5",dl_src=fa:16:3e:46:58:fe,tp_src=546,tp_dst=547 actions=CONTROLLER:0
  table=78, priority=0 actions=drop

For the new extension of openvswitch agent, it will add a local
``packet_in_handler`` which will do the following works:


* Listen on the EventOFPPacketIn event
* Verify each packet to be DHCPv4 or DHCPv6
* According to the openflow inport number to retrieve the port's information
* Assemble the DHCP(v4/v6) response and ``packet_out`` to ``in_port``.

The response DHCP packet structure will be:

::

  +------------------------------------------+
  |           *Source Mac Address            |
  |The gateway Port's MAC or A fake fixed MAC|
  +------------------------------------------+
  |       *Destination Mac Address           |
  |        Neutron Port Mac Address          |
  +------------------------------------------+
  |           *Source IP Address             |
  |      Gateway IP address from Subnet      |
  +------------------------------------------+
  |         *Destination IP address          |
  |             Neutron Port IP              |
  +------------------------------------------+

* ``Source Mac Address`` will be the internal subnet gateway port's Mac address.
  But actually this is not necessary for the DHCP protocol, we can use a fake
  fixed mac address to avoid some DB/RPC query.
* ``Destination Mac Address`` will be the port's MAC.
* ``Source IP Address`` will be the internal subnet's gateway IP.
* ``Destination IP address`` will be the port's first IP(v4/v6) address,
  the secondary IPs will be ignored.

Potential configurations
++++++++++++++++++++++++

Config option ``disable_traditional_dhcp`` for neutron server side will be
added which is aiming to control:

* to disable DHCP scheduling for networks
* to disable DHCP provisioning block
* to disable DHCP RPC/notification
* to disable all DHCP related API/attibutes network, subnet and port.

A new extension alias name ``dhcp`` will be added for neutron
openvswitch agent:

::

  [agent]
  extensions = ...,dhcp

Config section ``[dhcp]`` will be added for neutron openvswitch agent
and register some common options to determine DHCP protocol related
parameters, the final ``[dhcp]`` section for openvswitch agent will be:

::

  dhcp_opts = [
    cfg.BoolOpt('enable_dhcp_ipv6', default=False,
                help=_("Whether enable DHCP for IPv6")),
    cfg.IntOpt('dhcp_renewal_time', default=0,
               help=_("DHCP renewal time T1 (in seconds). If set to 0, it "
                      "will default to half of the lease time.")),
    cfg.IntOpt('dhcp_rebinding_time', default=0,
               help=_("DHCP rebinding time T2 (in seconds). If set to 0, it "
                      "will default to 7/8 of the lease time.")),
  ]

The Neutron basic workflow
--------------------------

1. User creates a VM in a network
2. Nova plug the VM's NIC port to ovs-bridge
3. Ovs-agent process the port and install the DHCP related flows
4. L2 provisioning block released (No DHCP provisioning block)
5. VM booting and send DHCP request out
6. Match the flows and ``packet_in`` to ovs-agent
7. Ovs-agent directly send DHCP(v4/v6) response to VM's port
8. VM booting success


Data Model Impact
-----------------
None


REST API Impact
---------------

With the new config options, the following APIs will be disabled [4]_:

* add_network_to_dhcp_agent
* remove_network_from_dhcp_agent
* list_networks_on_dhcp_agent
* list_dhcp_agents_hosting_network

For the option ``enable_dhcp`` of ``Subnet``, this agent extension will set
the flows based on that. If it is False, ports under this subnet will have
no flows installed in table 77 and 78. The DHCP request will hit the final
DROP action.

Upgrading
---------

For native ml2/ovs deployment, this feature will be easily to upgrade to
enforce. A simple way is to run all agents as they are. But disable the DHCP
provisioning block. After enable the ``dhcp`` extension for ovs-agent, the
DHCP request will be handled by it earlier than ``dnsmasq``.

If you need a pure deployment without DHCP agents, the following is an overview
about how to migrate to use this new feature:

* Upgrading the Neutron code and restart neutron-server processes.
* Setup the ovs-agent with ``dhcp`` extension.
* Disable all DHCP agents to make sure no more scheduled network are created.

::

  openstack network agent set --disable <dhcp_agent_id>

.. note:: This action will remove all scheduled network instances from
          the admin state DOWN DHCP agent.

* Set the ``disable_traditional_dhcp = True`` option for neutron-server to
  disable the scheduling related API/RPCs.
* (optional, just in case) Remove all scheduled network from all DHCP agents,
  this step is to pure all DHCP namespace and DHCP woker process ``dnsmasq``.
* After no more scheduled network, stop all DHCP agents and remove it from DB.

.. note:: This feature does not support DNS lookup. If your running deployments
          are using the DNS lookup function from ``dnsmasq``, consider use
          ``designate`` as an alternative.

Implementation
==============

Assignee(s)
-----------

* LIU Yulong <i@liuyulong.me>


Work Items
----------

* Config options for neutron server to control DHCP related codes.
* Create agent extension.
* Testing.
* Documentation.

Dependencies
============

None

Testing
=======

Functionality
-------------

We will add fullstack test case to verify this new agent extension:

* Create two fake fullstack VMs in two test namespaces
* Use DHCP(v4 and v6) to config the fake VM ports
* Ping (-4/6) from one fake  VM to another

References
==========

.. [1] https://review.opendev.org/c/openstack/neutron/+/364793
.. [2] https://tools.ietf.org/html/rfc2131
.. [3] https://tools.ietf.org/html/rfc8415
.. [4] https://review.opendev.org/c/openstack/neutron/+/772255/8/neutron/extensions/dhcpagentscheduler.py#106-126
