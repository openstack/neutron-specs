..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
[OVN] - IPv6 Distributed routing
================================

RFE: https://bugs.launchpad.net/neutron/+bug/1998609

This RFE intends to implement distributed routing support for IPv6 only or
dual-stack usage scenarios. Distributed routing is already a reality for IPv4
FIP addresses and the benefits of implementing DVR for IPv6 are the same as
for IPv4 FIP addresses.


Problem Description
===================

When we are using an IPv6 only or dual-stack deployment, the traffic for IPv6
addresses remains centralized by the gateway port of the router. DVR support
for IPv4 FIP addresses has already implemented in neutron using the
enable_distributed_floating_ip flag in the [ovn] section - ml2_conf.ini file.

This issue affects both End Users and Deployers:

* For End Users - (Network performance) Since traffic is centralized, all
  network applications need to go through the gateway router port chassis
  resident <-> compute node (via tunneling - e.g. Geneve) before reaching
  the endpoint. In case the gw router port is on the same chassis as the VM
  compute node, this is not applicable.
* For Deployers - (Additional settings for dynamic routing) As the IPv6 GUA
  subnet is associated behind the router's gw port, it is necessary to
  advertise this subnet dynamically to the border network element (which
  routes the external/internal traffic of IPv6 prefixes). It could be
  statically, but it is not feasible when we talk about large scale
  deployments. So, we need to enable additional settings like BGP with the
  neutron-dynamic-routing, for example.

If the compute node where the VM is running knows the path to send packets
to external networks (with the help of routing protocols configured on each
compute node, e.g. using FRR), outgoing traffic may work directly to the
border. However, the incoming traffic will always be forwarded to the
router's gw port, as it is the only one that knows how to respond the
Neighbor Solicitation to the IPv6 GUA address of the VM.

DVR Use case:

* The provider networks must be stretched over the Underlay Network and each
  Compute Node would have the bridge for external traffic. In an L3 Leaf-Spine
  Underlay, for example, the network to be reached is for the Underlay Network
  be able to stretch an L2 domain(VLAN) via VXLAN as dataplane and BGP EVPN as
  Control Plane. In this solution, the Leaf switches would need to work as HW
  VTEP Gateways to initiate and terminate the VXLANs tunnels and use BGP EVPN
  to learn and advertise the MAC Addresses from the Compute Node's provider
  network.

.. code::

  - E/W PATH:
    Compute Node VM <-> OVS br-int <-> br-provider <-> external-bridge-mapping
    <-> FRR <-> BGP EVPN type2 <-> E/W spine/leaf <-> [reverse flow to VM]
  - N/S PATH:
    Compute Node VM <-> OVS br-int <-> br-provider <-> external-bridge-mapping
    <-> FRR <-> BGP EVPN type2 <-> N/S Border Leaf <-> external network


Proposed Change
===============

To solve the problem described above, the proposal is to introduce a new NAT
rule for the IPv6 GUA addresses that are allocated to VMs (OVN understands
IPv6 GUA like a FIP). Even though it is a global address, the ovn-controller
running on the chassis needs this rule to start responding to Neighbor
Advertisements for IPv6 just like it does with GARPs for IPv4 FIP's.

To enable the IPv6 DVR NAT rule management, a new config option should be
enabled via [ovn] section in ml2_conf.ini file.

.. code::

  * ``enable_distributed_ipv6 = True``

This option is similar to a configuration option to enable DVR for FIP's:
enable_distributed_floating_ip.

OVN Impact
----------

OVN support of distributed routing for an IPv6 GUA follows the same idea of
the IPv4 case [1]_. For the ovn-controller, the external_ip and local_ip can
contain IPv4 or IPv6 addresses. Therefore, the contract rule for creating the
IPv6 NAT rule remains the same used for FIP IPv4 addresses.

NAT rule fields (OVN):

.. code::

  type        : dnat_and_snat
  logical_ip  : The same VM IPv6 GUA address
  external_ip : The same VM IPv6 GUA address
  logical_port: VM logical port
  external_ids: Managed by Neutron
  external_mac: VM MAC address

The external_ip and logical_ip used in the NAT rule are the same (e.g. VM
GUA). With this entry, the OVN should add logical flows to respond to IPv6
Neighbor Solicitation requests for the IPv6 GUA directly by ovn-controller
on the chassis where the VM resides.

We know that there is no NAT for IPv6, so this special rule that translates
the GUA address to itself is only used to create the flows in the chassis
where the VM resides. Without this rule, the compute node does not know how
to respond to that IPv6 GUA and centralize the communication through the
router's GW port (via Geneve tunnel if the router resides on another host).

Neutron Impact
--------------

The NAT rules for FIP IPv4 are managed by Neutron floating ips database.
In the case of the NAT rule for IPv6 GUA addresses, we need to create a
"fake translation rule" for the address associated with a VM. IPv6 addresses
are allocated to VMs during port creation and managed via ipv6_address_mode
(static, SLAAC, DHCPv6 Stateful, or DHCPv6 Stateless).

This means that port creation and deletion events manage IPv6 addresses and
MAC addresses allocated to a VM. In this case, the creation of the NAT rule
for IPv6 GUA can follow the port creation and deletion process provided by
ovn_client in the OVN driver.

The port structure provides the IPv6 address (subnet), the VM MAC address,
and the logical_port but to create a NAT rule it is necessary to allocate
this rule to the router, being necessary to search for the id of the router
associated with the port of the VM.

IPv6 GUA NAT rule creation:

.. code::

  {
    args = {'type': 'dnat_and_snat',
            'logical_ip': IPv6_ADDRESS,
            'external_ip': IPv6_ADDRESS,
            'logical_port': PORT_ID,
            'external_ids': EXTERNAL_IDS,
            'external_mac': EXTERNAL_MAC}
    _nb_idl.add_nat_rule_in_lrouter(gw_lrouter_name, args)
  }

IPv6 GUA NAT rule deletion:

.. code::

  {
    args = {'type': 'dnat_and_snat',
            'logical_ip': IPv6_ADDRESS,
            'external_ip': IPv6_ADDRESS}
    _nb_idl.delete_nat_rule_in_lrouter(gw_lrouter_name, args)
  }

For IPv4 FIP NAT rules, the external_ids has important fields such as the
fip_key, for example, but for IPv6 the external_ids information is not
relevant. So, we can set common port information in that field to keep
tracking. The neutron dnat_and_snat rule is composed with the following
information in the external_ids:

.. code::

  { external_ids = OVN_PORT_EXT_ID_KEY, OVN_DEVID_EXT_ID_KEY,
              OVN_NETWORK_NAME_EXT_ID_KEY, OVN_ROUTER_NAME_EXT_ID_KEY}
  }

The common port information is stored in the OVN "NAT:external_ids"
dictionary:

.. code::

  $ ovn-nbctl list NAT
   _uuid               : 4ab3ef03-c832-4635-bce9-779d1092fc89
   allowed_ext_ips     : []
   exempted_ext_ips    : []
   external_ids        : {"neutron:device_id"=
                                   "a467f63b-8159-4e87-83ce-d6b55ed4401f",
                          "neutron:network_name"=
                                 neutron-b3110178-9798-4951-a681-2e12301001f8,
                          "neutron:port_id"=
                                   "9910cdd5-d764-403c-ae35-565930107c7a",
                          "neutron:router_name"=
                                 neutron-a00e21fa-4332-4e26-8e34-73a24cbf46d1}
   external_ip         : "2001:db8:1234:1::33"
   external_mac        : "fa:16:3e:1d:31:57"
   external_port_range : ""
   logical_ip          : "2001:db8:1234:1::33"
   logical_port        : "9910cdd5-d764-403c-ae35-565930107c7a"
   options             : {}
   type                : dnat_and_snat


IPv6 DVR events
---------------

Neutron will configure the NAT rule for the IPv6 distributed routing
reacting to the following events:

* VM port creation: with this event, neutron create a NAT rule for the
  IPv6 GUA address allocated to the VM. If the VM does not have a router
  associated with the VM subnet, do not create the IPv6 NAT rule.
* VM port deletion: same behavior as the VM port creation, this event is
  received and neutron must delete the NAT rule attached to the router. If the
  VM does not have a router associated with the VM subnet, do not delete
  the IPv6 NAT rule.
* Router port creation: if this event is received, it is necessary to check
  if the added port is in the same subnet associated with the previously
  created VM port. This means neutron must create the IPv6 NAT rule if the
  router port is associated with the VM subnet.
* Router port deletion: same behavior as the router port creation, this means
  neutron must delete the IPv6 NAT rule if the router port is associated
  with the VM subnet.

FIP IPv4 Impact
---------------

Until now, the only NAT rule of type 'dnat_and_snat' was used exclusively
for FIP's, therefore, the functions that search the OVN database for NAT
rules of type 'dnat_and_snat', need to filter for exclusive FIP rules.
One idea would be to create a new constant in external_ids to set the IP
version, but it might be enough just to look for rules that have the
OVN_FIP_EXT_ID_KEY fieldset.

The change in neutron OVN NAT lookup for FIP's may require changes in
existing tests for floating ips.

DB Impact
---------

No new column entries or tables are needed to implement this RFE. IPv6 and
MAC address information is already recorded on ports associated with VMs.

However, to ensure consistency of information between neutron and OVN, it
is necessary to implement a check for IPv6 NAT rules during ovn_db_sync.
For VMs already created before enabling 'enable_distributed_ipv6' flag,
neutron will notify in the log about the config differences when it is
restarted. The NAT rules for IPv6 GUA will be automatically added or
removed when executing a SYNC_MODE_REPAIR.


Implementation
==============

Assignee(s)
-----------

Primary assignees:
  Roberto Bartzen Acosta <rbartzen@gmail.com>

Testing
=======

* Unit/functional tests.


Documentation Impact
====================

User Documentation
------------------

* Information about the IPv6 distributed routing support.

References
==========

.. [1] https://mail.openvswitch.org/pipermail/ovs-discuss/2022-December/052126.html