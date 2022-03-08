..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================
Distributed Router Advertisement on Openvswitch Agent
=====================================================

RFE: https://bugs.launchpad.net/neutron/+bug/1961011

Neutron L3 router will run ``radvd`` to send out RA (Router Advertisement)
packets to notify the guests about the subnet ManagedFlag, LinkMTU and prefix
of IPv6 subnet. This is fine to work for small scale clouds, or long stand
clouds with no changes. But for large scale cloud environments, management
complexity increases.

Neutron openvswitch agent can be considered as a distributed SDN controller,
this spec describes how to make RA work on each compute node in distributed
mode without any extra processes.

Problem Description
===================

``radvd`` is spawned in L3 router namespaces, which will multicast a
RA on the subnet from the router interface, reaching all attached VMs.
Current radvd config looks like this:

::

  interface qr-8fd65326-c4
  {
     AdvSendAdvert on;
     MinRtrAdvInterval 30;
     MaxRtrAdvInterval 100;
     AdvLinkMTU 1500;
     AdvManagedFlag on;
     prefix fda7:a5cc:3460:3::/64
     {
          AdvOnLink on;
          AdvAutonomous off;
     };
  };

In order to do such work, L3 agent needs to do:
* start a process manager for ``radvd``
* render the config for ``radvd``
* monitor the extra process in case of unexpected exit
* respawn the process if something changes

For a DVR router, ``radvd`` will be run in every local router namespace
on the compute node as well.

In certain cases, this increases the number of server processes, increases
the resource consumption of the server, increases the failure point and reduces
the processing performance of the L3 agent.

There is a defect of current RA mechanism. It forces the subnet to have
association with a Neutron router, if you want Neutron to manage the IPv6
related configurations [2]_. Then if a subnet does not connect to a router,
no radvd answers the RS request, no RA back. VMs under isolated networks
will get failed to find out how to do with IPv6 address, how to configure
IPv6 address, how to do with DHCPv6.

So, this agent extension will make isolated networks can work for IPv6.
VM under it will get RA about the IPv6 subnets prefixes, autonomous address
configuration flag, managed address configuration flag and other configuration
flag and so on. VMs under the isolated networks are going to work with IPv6
based on these flags.

Proposed Change
===============

Adds an agent extension for Neutron openvswitch agent to send RA out to local
guests in a more natural and graceful style.

.. note:: This extension is only for openvswitch agent, other mechanism drivers
          will not be considered, because this new extension will rely on the
          openflow protocol and principle.

Solution Proposed
-----------------

The new agent extension will use os-ken(ryu) [1]_ to assemble the RA packets,
and then directly do packet-out to the ofport (VM port). Mission accomplished,
very simple! Some main code procedure will be:

* new extension initialized with a subnet cache list
* start a periodic loop to send RA to each port from those subnets
* in ``handle_port`` method each VM port's subnet will be added to the cache
* in ``delete_port`` remove the VM port from subnet's port cache
* get subnet information by the resource cache RPC
* looping of sending RA to each port

Openvswitch agent will pull ports, subnets and networks information by
"resource cache" related RPC.

.. note:: For some cases, neutron router will interact with the upstream physical
          router. This extension will not handle such configurations and
          deployment architecture.

The attribute ``ipv6_ra_mode`` of subnet is None, then Neutron will not
generate RA to VMs. ``ipv6_ra_mode = None`` will be used to indicate if
there are external (physical) routers.

Implementation
==============

Assignee(s)
-----------

* LIU Yulong <i@liuyulong.me>


Work Items
----------

* Adding ovs-agent extension to do the RA work.
* Make L3-agent radvd configurable.
* Testing.
* Documentation.

Dependencies
============

None

Testing
=======

Test cases to verify the RA can work for ports.

References
==========

.. [1] https://ryu.readthedocs.io/en/latest/library_packet_ref/packet_icmpv6.html#ryu.lib.packet.icmpv6.nd_router_advert
.. [2] https://docs.openstack.org/neutron/latest/admin/config-ipv6.html
