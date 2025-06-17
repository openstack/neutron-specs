..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================
Distributed DNS Forwarder with Openvswitch Agent
================================================

RFE: https://bugs.launchpad.net/neutron/+bug/2112446

Neutron DHCP agents (using dnsmasq) can act as DNS forwarders or
provide DNS resolution for instances by using resolvers configured
on the host where the DHCP agent runs. However, when users deploy Distributed DHCP
alongside the Open vSwitch (OVS) agent, they are responsible for ensuring
that instances can reach DNS servers or for setting up their own forwarding mechanism.

This spec describes how to implement a DNS forwarder extension for Neutron
openvswitch agent to provide a simple, fully distributed DNS forwarding capability
for virtual machines by leveraging OpenFlow rules and os-ken applications.

Problem Description
===================

*   DNS plays a critical role in ensuring high availability for services such as databases,
    VPN clusters, or any failover scenarios where DNS records are updated dynamically
    whether within OpenStack or through external systems. In addition, DNS resolution is essential
    in OpenStack environments to allow instances on external or project networks to those services,
    even if they don't have a direct path to DNS servers. For instance, a web server running in
    an isolated VXLAN network might need to connect to a database service (provided by the cloud provider)
    on the same network using an internal domain name.

*   Distributed DHCP for Openvswitch Agent is good for large-scale deployments, but it lacks
    built-in DNS resolution capabilities, unlike the DHCP agent that uses dnsmasq. [1]_

Proposed Change
===============

A new extension of Neutron openvswitch agent will be added to achieve the
``Distributed DNS Forwarder with Openvswitch Agent``.

.. note:: This extension is only for openvswitch agent, other mechanism drivers
          will not be considered, because this new extension will rely on the
          openflow protocol and principle. For OVN, it has supported similar
          DNS resolution mechanism with ovn-controller.

Solution Proposed
-----------------

Developing a new OS-Ken application in OVS-agent which will act as an intermediary
between instance and a DNS server, so it can response DNS query to instance without
direct connection to the actual DNS server.

The basic data pipeline can be described as this:

.. code-block:: none

                            +--------------+                +---------------------+                +---------------------+
    +-------+ DNS Query     |              |    packet-in   |  +----------------+ | DNS Query      |                     |
    |  VM   +--------------->  Flows       +---------------->  |   os-ken app   | +---------------->   Local resolvers   |
    |       <---------------+ (capture     <----------------+  |                | <----------------+        or           |
    +-------+ DNS Answer    | DNS query    |    packet-out  |  | DNS Forwarder  | | DNS Answer     |   Real DNS servers  |
                            |   here)      |                |  +----------------+ |                |                     |
                            |   br-int     |                |      OVS-agent      |                |                     |
                            +--------------+                +---------------------+                +---------------------+

After this we will have:

*   Higher level availability than DNS resolution with ``dnsmasq`` in DHCP Agent.
*   OpenStack with distributed DHCP extension [1]_ can now support DNS lookup.
*   Great combination with Distributed metadata datapath for Openvswitch Agent [2]_.

DNS Forwarder
+++++++++++++

We will extract DNS payload from the packet, then send it to upstream DNS server.

Server side changes
+++++++++++++++++++
None

OpenvSwitch Agent side changes
++++++++++++++++++++++++++++++

For Neutron openvswitch agent, we will add a new agent extension which will
process the basical flow installation for any DNS query.

There will be some basic flows which will check if the DNS query is matched
with destinations, then submit it to the controller. The default destinations are
`169.254.169.254:53` and `[fd00::254]:53`. We will make it configurable.

.. code-block:: none

  table=60, priority=102,udp,nw_dst=169.254.169.254,tp_dst=53 actions=CONTROLLER:0
  table=60, priority=102,udp6,nw_dst=fd00::254,tp_dst=53 actions=CONTROLLER:0

For the new extension of openvswitch agent, it will register a local
``packet_in_handler`` which will do the following works:

*   Listen on the EventOFPPacketIn event
*   Verify each packet if it's matched above rules
*   Extract DNS payload from the packet
*   Sent DNS payload to upstream DNS server
*   Assemble the DNS answer response and ``send_msg`` to ``in_port``

The response DNS answer packet structure will be:

.. code-block:: none

  +------------------------------------------+
  |           *Source Mac Address            |
  |   169.254.169.254/fd00::254 MAC Address  |
  +------------------------------------------+
  |       *Destination Mac Address           |
  |        Neutron Port Mac Address          |
  +------------------------------------------+
  |           *Source IP Address             |
  |        169.254.169.254/fd00::254         |
  +------------------------------------------+
  |         *Destination IP address          |
  |             Neutron Port IP              |
  +------------------------------------------+

*   ``Source Mac Address`` will be `169.254.169.254/fd00::254` MAC Address.
*   ``Destination Mac Address`` will be the port's MAC.
*   ``Source IP Address`` will be `169.254.169.254/fd00::254`.
*   ``Destination IP address`` will be the port IP(v4/v6) address.

.. note:: Unlike DHCP Request which typically rely on broadcast packets, this solution
          works more like the metadata service which is need to have a route
          to `169.254.169.254/fd00::254` via somewhere to sent out DNS query packet.
          These packets will then be captured on the br-int bridge.
          The necessary route can be provided by the DHCP agent, distributed DHCP, or via cloud-init.
          Additionally, the specified next hop must be reachable, meaning the instance
          should be able to resolve its MAC address using ARP and in most cases,
          this next hop is the default gateway.

Potential configurations
++++++++++++++++++++++++

A new extension alias name ``dns_forwarder`` will be added for neutron
openvswitch agent:

.. code-block:: ini

  [agent]
  extensions = ...,dns_forwarder

Config section ``[dns_forwarder]`` will be added for neutron openvswitch agent
and register some common options:

.. code-block:: python

  dns_forwarder_opts = [
    cfg.ListOpt('upstream_dns_server_ports',
                default=['1.1.1.1:53', '[2606:4700:4700::1111]:53'],
                help=_('Comma-separated list of the Upstream DNS server
                       'in ip:port format which will be used as resolvers.')),
    cfg.IntOpt('upstream_dns_query_timeout', default=5,
               help=_("Query timeout in seconds for each Upstream DNS servers")),
    cfg.ListOpt('client_dns_server_ports',
                default=['169.254.169.254:53', '[fd00::254]:53'],
                help=_('Comma-separated list of the Client DNS server
                       'in ip:port format which will be used inside client instances.')),
  ]

Data Model Impact
-----------------
None


REST API Impact
---------------
None

Upgrading
---------
This feature does not affect any current features, so we simply enable it to use.

Implementation
==============

Assignee(s)
-----------

Primary assignees:

* Dai, Dang Van <daikk115@gmail.com>

Work Items
----------

* Implement a new DNS Responder OS-Ken application.

* Implement some new configuration options.

* Implement relevant unit and functional tests.

* Write documentation.


Testing
=======

* Unit/functional tests.


Documentation
=============

* User Documentation: How to setting up internal DNS resolution with OVS.

References
==========

.. [1] https://specs.openstack.org/openstack/neutron-specs/specs/wallaby/distributed_dhcp.html#upgrading
.. [2] https://specs.openstack.org/openstack/neutron-specs/specs/yoga/distributed-metadata-data-path.html
