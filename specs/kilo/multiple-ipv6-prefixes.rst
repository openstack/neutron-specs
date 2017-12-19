..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================================
Support Multiple IPv6 Prefixes and Addresses for IPv6 Network
=============================================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/multiple-ipv6-prefixes

IPv6 standards allow for multiple IPv6 prefixes and addresses to be
associated with a given interface. This blueprint defines how multiple
IPv6 prefixes and/or addresses per Neutron port will be supported in
OpenStack, and describes how this differs from the IPv4 implementation.

Some possible use cases include:

* Allow a mix of public (Global Unicast Address, or GUA) and private
  (e.g. Unique Local Address, or ULA) IPv6 addresses on a tenant network.
* IPv6 renumbering: Support for overlap interval during which old and new
  IPv6 prefixes co-exist (e.g. on router port).
* Floating-IP-like support: Allow for a tenant to allocate an address from
  a pool of pre-determined public IPv6 addresses, and associate that
  address with a VM or set of VMs. Note: Unlike the IPv4 Floating IP
  feature, IPv6 support would not involve NAT translation from fixed IPs
  to floating IPs. This design specification is a pre-requisite for
  IPv6 Floating IPs; more details will be covered in a separate blueprint.


Problem Description
===================

* Considerations for IPv6

  There are some differences between IPv6 and IPv4 which need to be
  considered when deciding how multiple IPv6 prefixes and addresses
  should be configured and supported in Neutron, and how closely the
  IPv6 implementation will follow the IPv4 paradigm:

  * IPv6 addresses are more abundant. That means:

    * We can afford to be less conservative about assigning addresses to a
      port from each of the subnets within a network.
    * No need to do NAT translation. Globally accessible addresses can
      be assigned directly to the port.

  * IPv6 provides two main mechanisms for assigning IP addresses:

    * SLAAC (Stateless Address Autoconfiguration):
      This mechanism [Ref-2]_ requires minimal configuration of hosts and
      routers. With SLAAC, routers periodically send out Router Advertisement
      (RA) messages via multicast to all IPv6-enabled hosts on that link.
      (In OpenStack, generation of RA messages by virtual routers is provided
      by a router advertisement daemon, or radvd process.) The RA messages
      contain one or more prefixes that identify subnets associated with
      the link. Hosts on the link then generate unique IPv6 addresses per
      SLAAC subnet and per interface on the link by forming a unique
      "interface identifier" (typically based on the interface's MAC
      address), and combining that interface identifier with each prefix.
      To obtain an router advertisement quickly, hosts on that
      link may also prompt the router by sending a Router Solicitation
      message [Ref-3]_.
    * DHCPv6: Similar to DHCPv4, a DHCP server can be configured to assign
      specific IPv6 addresses to each node on a link.

    With the current Neutron implementation, a subnet is configured for
    SLAAC vs. DHCPv6 stateful/stateless address mode when the subnet is
    created by setting the "ipv6_ra_mode" and "ipv6_address_mode" attributes
    in a subnet-create API call. These attributes were added with the
    blueprint listed in [Ref-4]_. Attribute values are described in [Ref-5]_.

    For the purposes of this design specification, the term "SLAAC-enabled"
    subnets will be used to refer to subnets that are configured for either
    SLAAC or DHCPv6-stateless address modes.

    The important aspect of SLAAC addressing to consider here is that
    RA messages are multicast (on the all nodes multicast channel) to all
    IPv6-capable nodes on a link. What this means in an OpenStack instance
    is that whenever a subnet is created with SLAAC address generation mode,
    all IPv6-capable hosts/VMs which are created on that network will
    receive periodic RA messages which include the associated prefix for that
    subnet (along with prefixes for all other SLAAC-enabled subnets on that
    network). The host will therefore generate an address using that
    subnet/prefix, regardless of whether the associated port-create API calls
    (and any follow-up port-update API calls) for that VM happen to specify
    that the subnet should be associated with that port.

    Theoretically, it would be possible to selectively enable/disable
    delivery of RA prefix assignments on a per-port, per-subnet basis by
    configuring radvd so that RA messages are delivered per subnet via
    unicast, but this implementation would be rather complex and would not
    scale well in a large network because of the added messaging and
    processing required, and would go against the automatic nature of SLAAC.

* Current Implementation

  * Port creation with explicit list of fixed IPs:

    With the current Neutron implementation, when a port is created with
    an explicit list of fixed IPs, IPs from the following subnets on the
    associated network will be associated with the port:

    * All subnets/IPs that are explicitly listed

    What is missing is that SLAAC-enabled subnets are not implicitly
    included for association with the port, even though IPv6-enabled VMs
    will be automatically generating SLAAC addresses for those subnets
    based on RA messages that they receive.

  * Port creation without explicit list of fixed IPs:

    With the current Neutron implementation, when a port is created without
    an explicit list of fixed IPs, IPs from the following subnets on the
    associated network are associated with the port:

    * One subnet from all IPv4 subnets
    * One subnet from all IPv6 DHCPv6-stateful subnets
    * All SLAAC-enabled subnets

    What could be considered here is whether all DHCPv6-stateful subnets
    should also be implicitly included for allocation and association with
    the port (along with SLAAC-enabled subnets), since IPv6 addresses are
    more plentiful. However, this would be quite a shift from the IPv4 model,
    so it would be safer and less confusing to stay with the current
    implementation.

  * Port update:

    With the current Neutron implementation, when a port is updated,
    IPs from the following subnets on the associated network are
    associated with the port:

    * All subnets/IPs explicitly listed in the optional fixed_ips list

    What's arguably broken here is that any existing association with
    addresses from SLAAC-enabled subnets that are not explicitly included
    in the fixed_ips list of a port update request will be forgotten
    (deleted), even though the port will continue to use the SLAAC addresses
    as long as the source of IPv6 RA messages (radvd process) continues to
    send RA messages to all IPv6-enabled hosts via the all-nodes multicast
    channel.

  * Router interface create

    Layer 3 (L3) agent limits the number of IP addresses that is initially
    associated with an internal router interface (gateway port) to a maximum
    of one IP address per family (i.e. up to one IPv4 address and up to
    one IPv6 address).

    For externally-facing gateway ports, the number of IP addresses is
    further restricted to a single IP address, either IPv4 or IPv6 [Ref-1]_.

    The change described in [Ref-1]_, which is currently abandoned, was
    an attempt to relax this restriction so that up to one IPv4 address
    and up to one IPv6 address would be allowed per external gateway port.

    This blueprint will include the change described in [Ref-1]_, and will
    also relax the restrictions for internal router ports by allowing
    multiple IPv6 addresses per internal router port.


Proposed Change
===============

* Port Create API Call

  The definition/syntax of the Neutron port-create API request/response
  messages will not be changed for this blueprint. However, the way in
  which this API call will be used, and the way that Neutron handles this
  call when IPv6 subnets are included in the associated network, will
  represent a small shift from the IPv4 model. The change in behavior will
  revolve around the notion that if a network includes a SLAAC-enabled
  subnet, then all ports on the network will get an address from that subnet
  automatically. The changes can be summarized:

  * Current IPv4 behavior will not be changed.

  * Port-create operations with an explicit list of fixed IPs:
    When an explicit list of fixed IPs is included (via fixed_ips dictionary),
    the following changes will apply:

    * All SLAAC-enabled subnets on the associated network will now be
      implicitly associated with the port, regardless of whether or not
      these subnets are explicitly included in the fixed_ips list.
      (DHCPv6-stateful subnets will continue to be included in this case
      only when they are explicitly included in fixed_ips.)
    * A port-create request can optionally include SLAAC-enabled
      subnets explicitly listed in fixed_ips, but these entries are
      superfluous.

  * Port-create operations without an explicit list of fixed IPs:
    There will be no change from the current design. When a port-create
    operation does not include an explicit list of fixed IPs, the following
    subnets on the associated network will be associated with the port:

    * One subnet from all IPv4 subnets
    * One subnet from all IPv6 DHCPv6-stateful subnets
    * All SLAAC-enabled subnets

  * Response to API call will list all fixed IPs associated with the
    port, whether the association was implicitly or explicitly included in
    the port-create request.

* Port Update API Call

  Similar to the port-create API call, the definition/syntax of the Neutron
  port-update API request/response messages will not change, but the way in
  which this API call is used and the way that Neutron handles these
  requests will change for IPv6 subnets/addresses. The change in behavior
  will revolve around the notion that if a network includes a SLAAC-enabled
  subnet, then all ports on the network will get an address from that subnet
  automatically. The following changes will apply:

  * The association of IP addresses/subnets for each SLAAC-enabled subnet
    on the associated network will now be implicitly preserved (retained)
    when the port-update operation does not explicitly include the
    SLAAC-enabled subnet in the fixed_ips list of changed/new
    subnets/addresses.

  * The response to the API call will include addresses for all SLAAC-enabled
    subnets on the network in the list fixed IPs associated with the
    port, whether the association was implicitly or explicitly included in
    the port-update request.

* Subnet Create API Call

  When a SLAAC-enabled subnet is created after ports have already been
  created on a given network, and that subnet relies on a
  provider source for RA messages (as opposed to an OpenStack reference
  RADVD process; this is selected via "ipv6_ra_mode" and "ipv6_address_mode"
  attributes in the subnet-create API call), then the associated IPv6-capable
  hosts on the network will be expected to automatically generate SLAAC
  addresses based on prefixes advertised by the provider's source of RA
  messages. In order to keep in sync with these generated addresses, the
  subnet create processing will need to be modified to add entries in the
  IPAllocations table for all existing ports on the network in order to
  associate the ports with addresses on the SLAAC-enabled subnet.

* Limits on Number of IP Addresses per Router Interface

  * Externally-facing gateway ports: As discussed in the Problem
    Definition section, this blueprint will relax the restrictions imposed
    by the Neutron L3 agent on externally-facing gateway ports by allowing
    up to one IPv4 address and up to one IPv6 address per port.

  * Internal router ports: This blueprint will relax restrictions imposed
    on internal router ports by allowing multiple IPv6 addresses on each
    internal router port. For IPv4 subnets, no change will be made, that
    is, separate router interfaces will be required for each IPv4 subnet
    that requires a gateway IP address.

    The handling for the router interface create API call will be changed
    so that IPv6 subnets on a given network will share one internal router
    port per router to which they are attached. That is, when the router
    interface create API is called for an IPv6 subnet, a check will be
    performed to see if there is already an internal router interface
    on that router that supports a gateway address for one of the other
    IPv6 subnets in the network. If so, the new IPv6 gateway IP address
    will be added to that existing interface, rather than having a new
    router interface created.

Data Model Impact
-----------------

No data model changes are anticipated.

REST API Impact
---------------

As discussed in earlier sections, the definition/syntax of the port-create,
port-update, and subnet-delete API calls will not be changed, but the way
in which these API calls are used and the way that Neutron handles these
API calls will be modified for IPv6 subnets as compared to IPv4 subnets.
In particular, ports will be automatically associated with addresses from
all SLAAC-enabled subnets within the port's network as part of port create
and port update operations, and the port create and port update responses
will automatically include those SLAAC addresses in the list of fixed_ips.

Security Impact
---------------

There may be some risk associated with opening up an OpenStack cloud to
IPv6 RA messages (e.g. spoofed RA messages may modify the network), but this
is a concern for basic SLAAC implementation, and not something that is being
introduced with this blueprint.

Notifications Impact
--------------------

No changes anticipated.

Other End User Impact
---------------------

Nothing more than what was discussed in the "REST API Impact" section.

Performance Impact
------------------

No significant performance impact expected.

IPv6 Impact
-----------

Yes, this effects the Neutron reference IPv6 implementation.

Other Deployer Impact
---------------------

Deployers will need to understand that IPv6 addresses will be automatically
and implicitly generated on every port on a network for every SLAAC-enabled
subnet in that network. This should not come as no surprise, since SLAAC is
an autoconfiguration mechanism.

Developer Impact
----------------

Developers of L3 services may need to incorporate some of the behavior
changes proposed in the blueprint with respect to port creation when
IPv6 subnets are involved to ensure that management plane behavior properly
reflects the way that SLAAC works. The implementation for this blueprint
will be incorporated into common framework database classes, so services
that derive from these classes should automatically inherit this behavior.

Community Impact
----------------

This blueprint is a natural progression from the addition of RADVD and
IPv6 SLAAC support to the Neutron reference design. It is an attempt to
reconcile some inconsistencies between the way that IPv6 SLAAC works
and what IP addresses are implicitly associated when a port is created
without an explicit list of fixed IPs in the current design.

This has not been discussed on mailing lists, but the blueprint and
associated patch set have been up for review since mid-Juno. This was added
as an etherpad agenda item to discuss at the Atlanta summit during the IPv6
design session, but unfortunately the design session ended before we got to
this item.

Alternatives
------------

As an alternative to what is being proposed, it would be possible to
avoid the automatic nature of RA prefix assignments by configuring radvd
so that RA messages are delivered on a per-port, per-subnet basis via
unicast. Ports that explicitly include a given SLAAC-enabled subnet would
then be sent an RA unicast that includes an advertisement of that subnet,
and ports that don't explicitly include the SLAAC subnet would not receive
an RA unicast (or would receive an RA that doesn't include the subnet).
However, this approach would be rather complex and error prone, and
would go against the automatic nature of SLAAC. Also, this approach would
not add much value given that IPv6 addresses can already be selectively
assigned on a subnet by using DHCPv6-stateful.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Dane LeBlanc
  launchpad-id: leblancd

Other contributors:
  Robert (Bao) Li
  launchpad-id: baoli

Work Items
----------

* Port-create API handling coding and unit test
* Port-update API handling coding and unit test
* Subnet-delete API handling coding and unit test
* Router/Gateway interface coding and unit test
* Tempest test creation


Dependencies
============

None

Testing
=======

Tempest Tests
-------------

Testing for API behavior will be incorporated into the Neutron in-tree
API tests, so no out-of-tree Tempest API tests will be added.

A Tempest network scenario test will be added to check dataplane
connectivity when multiple addresses per port are in use:

  - Create an internal router port with 1 IPv4, 1 SLAAC, and 1 DHCPv6 subnet.
  - Create an instance using all three subnets.
  - From the instance, ping the gateway IP address for each of the three
    subnets.
  - From the instance, ping an external gateway port.

Functional Tests
----------------

No functional tests needed.

API Tests
---------

* API tests for port create on a network with 1 IPv4 and 1 SLAAC
  address/subnets.
* API test for port create on a network with 1 IPv4, 1 DHCPv4
  address/subnets.
* API test for port create on a network with 2 DHCPv4 and 2 SLAAC
  address/subnets.
* API test for port update: Start with a SLAAC address, add DHCPv4 address.


Documentation Impact
====================

User Documentation
------------------

The Neutron API v2 reference documentation may need to be modified to reflect
the expected behavior with respect to implicit IPv6 address assignments for
port-create, port-update, and subnet delete (although the documentation
doesn't go into this much detail):

https://wiki.openstack.org/wiki/Neutron/APIv2-specification#Create_Port
https://wiki.openstack.org/wiki/Neutron/APIv2-specification#Update_Port
https://wiki.openstack.org/wiki/Neutron/APIv2-specification#Delete_Subnet

Developer Documentation
-----------------------

None needed beyond Neutron API v2 documentation changes listed above.


References
==========

.. [Ref-1] Neutron Blueprint: `Allow multiple subnets on gateway
   <https://blueprints.launchpad.net/
   neutron/+spec/allow-multiple-subnets-on-gateway-port>`_

.. [Ref-2] RFC 4862: `IPv6 Stateless Address Autoconfiguration
   <http://tools.ietf.org/html/rfc4862>`_

.. [Ref-3] RFC 4861: `Neighbor Discovery for IP version 6 (IPv6)
   <https://datatracker.ietf.org/doc/rfc4861>`_

.. [Ref-4] Neutron Blueprint: `Two Attributes Proposal to Control IPv6 RA
   Announcement and Address Assignment
   <https://blueprints.launchpad.net/neutron/+spec/ipv6-two-attributes>`_

.. [Ref-5] `OpenStack Neutron IPv6 Address Mode Attributes Table
   <https://www.dropbox.com/s/9bojvv9vywsz8sd/IPv6%20Two%20Modes%20v3.0.pdf>`_

Related Information
-------------------

-  DevStack Blueprint: `Add IPv6 support
   <https://blueprints.launchpad.net/devstack/+spec/ipv6-support>`_
-  Neutron Blueprint: `Provider Networking - upstream SLAAC support
   <https://blueprints.launchpad.net/neutron/+spec/ipv6-provider-nets-slaac>`_
-  RFC 4291: `IP Version 6 Addressing Architecture
   <http://tools.ietf.org/html/rfc4291>`_
