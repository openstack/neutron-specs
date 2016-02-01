..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
MTU selection and advertisement
===============================

https://blueprints.launchpad.net/neutron/+spec/mtu-selection-and-advertisement

This proposes a list of fixes to the MTU problems of Openstack to
ensure that VMs run with the correct MTU on both virtual and physical
Openstack networks.

Problem Description
===================

tl;dr: Your Neutron network's MTU for any L3 encap is probably not
1500 bytes.  This causes a multitude of creeping horrors, listed in
excruciating detail here, and while there are existing workarounds
they don't work in a variety of deployment types, probably including
the one you're running right now.  If you are willing to accept this
as a given, skip to the next section.

Neutron networks have unpredictable MTUs, which causes a long list of
corner case behaviour.

The MTU of a tenant network running on machines that themselves have
MTUs of 1500 bytes on all their interfaces (the typical default value)
may be, among other values:

* 1500 bytes, where the encap mechanism used is VLANs, or the network
  is a flat network.  Note that, independent of your preferred encap
  type, one of these two applies to all your externally connected
  network

* 1492 bytes, where the encap mechanism used is GRE tunnels.  Note
  that this implies that your tenant network on one side of the router
  doesn't have the same MTU as the external network on the other side

* smaller, if the encap is something else L3 like VXLAN

* larger, if the cloud has been deployed on an infrastructure using
  jumbo frames

If the maximum transfer unit size is exceeded, packets are dropped,
either on the network (if the network cannot pass traffic as big as
the sender expects to send) or at the desination (if the desination's
configured MTU is smaller than that of the sender or the network).  In
practice, this means that both the sender and receiver - be they VMs,
services or routers - *must* agree on MTU size, and the network
segment *must* pass packets of the agreed size.

Also, note that L3 mechanisms for path MTU discovery *only* take place
for packets passing through a L3 element such as a router.  They do
*not* take place for traffic transiting an L2 segment, which means
that large packets are dropped without the sender finding out.  This
leads to inconsistent behaviour with different transfer types:
typically, bulk transfers of data do not work but some connections
such as small HTTP requests do.

Aside from the size of the absolute maximum transfer unit that the
segment will pass, there is also typically a preferred transfer unit
size to use.  Some tunnelling protocols allow fragmentation of packets
after encapsulation, which means that if a certain transfer unit size
is exceeded the encapsulated packet undergoes fragmentation and, while
the packet arrives, the network performance drops off dramatically.

Finally, MTU is frequently inconsistent between different endpoints in
a segment.  Between two VMs on the same host, it is quite common to be
able to communicate with transfer units that are bigger than between
two VMs on different hosts.  The MTU for the segment should be the
minimum that works between any pair of hosts, but this is not always
true.

A recommended and widely deployed workaround is to advertise a
functional sub-1500 MTU to all VMs over DHCP, which works providing:

* DHCP MTU advertisements are respected

* there are no provider networks in operation (as their MTU size is
  typically 1500 and may be completely different, and in either case
  VMs will not communicate with external systems)

* the external network is configured for small MTUs at the nexthop

* any service MTUs are also appropriately configured.

This also does not function for IPv6-only VMs - they respond to an RA
attribute, not DHCPv4, and the RA attribute will generally only work
to *reduce* network MTU - or for the occasional VMs that do not speak
IP at all.

Occasionally, tenants also have opinions about the MTU they desire -
either that it is larger than the default 1500, or, for deployments
that have been deployed in a way that makes large MTU sizes
functional, that they nevertheless wish to have a 1500 MTU.

Proposed Change
===============

This blueprint proposes bringing consistency to MTU operations, such
that (in order of desirability):

* the best MTU to use is automatically determined

* the plugin can ensure that advertisement mechanisms (e.g. DHCP, RA,
  config drive) can access the correct MTU for the network, and are
  used to propagate it to guests when this functionality is enabled

* the plugin can ensure that network services attached to a network
  operate with the correct MTU (e.g. router ports have an appropriate
  interface MTU configured) so that, even in the case of differing MTU
  sizes on different networks, normal L3 behaviour will accommodate
  the variance without breaking tenant communications.

* the tenant can discover the MTU on the network, so that - in the
  case that the tenant has to write configuration files on a config
  drive to instruct the VM to configure MTU - it can tell which MTU
  value to use.  Note that some VMs will not request DHCPv4 MTU, a
  strict reading of the RFC prohibits VMs from accepting a large MTU
  over IPv6 RA, and there is no config drive mechanism at present for
  avertising an MTU, so this possibility must be accounted for

* the tenant can request a specific MTU on a network - this is so
  that, in the case that the tenant has VMs with preconfigured MTUs
  (for instance, ones that do not respect any of the available forms
  of MTU advertisement, they can ask for a specifc - typically 1500 -
  MTU for networks that have these VMs on)

* the plugin can confirm that it can deliver a certain MTU on the
  network, if a request has been made explictly

For any given network, Neutron will internally calculate two values.
These are:

* the maximum MTU that will work.  This may be indeterminate (for
  instance, if the encap fragments).

* the optimum MTU to use.  This may be the maximum MTU, or it may be
  smaller.  Packets greater than this optimum MTU may see degraded
  throughput, though if they are smaller than the maximum MTU they
  will be passed.

The tenant may, optionally (and in Phase III; see below), propose an
MTU for the network by setting the 'mtu' flag on a net-create.  The
plugin in use has two options:

1. It can create the network, setting the MTU of that network to the
   size given (meaning that the L2 segment will pass packets of at
   least that size; services started on that network will have the
   same MTU as the network; and the MTU advertised, if advertisement
   is turned on, is the MTU selected).  The 'mtu' attribute of the
   network, stored in the DB and returned on the network object, will
   contain the MTU requested.

2. It can decline to create the network, because it cannot satisfy the
   MTU request, if the MTU is greater than the maximum MTU.
   Alternatively, Neutron will decline the request if the plugin does
   not support MTU control.  This allows us to make the 'mtu'
   interface core while supporting plugins that do not recognise it.

When the MTU is *not* provided, supporting plugins will use the
preferred value.  Non-supporting plugins will leave it indeterminate.

If the network is also flagged as VLAN transparent (per the VLAN
transparency spec, not implemented yet, at [1]), the network must pass
a VLAN tagged packet with content of the MTU size or less (meaning the
content of the packet before VLAN tagging must be 8 bytes smaller than
the MTU on the Neutron network).  This distinction matters in
tunnelling protocols such as GRE - traditionally, VLAN tags are not
counted against MTU, but the additional header fields count against
GRE tunnel payload, so additional space needs to be left if the tagged
packet might pass through a tunnel.

A configuration item for Neutron, 'advertise_mtu', defaulting to false
if absent (for backward compatibility), enables MTU advertisement.  If
set to true, various MTU advertisement mechanisms will be enabled and
configured with the MTU.  The list of advertisements include:

* DHCPv4 will advertise the MTU when requested by the guest with an
  'MTU' option

* IPv6 RAs, where sent by Neutron, will contain the MTU (though,
  typically, guests will only respect MTU sizes less than 1500 unless
  specially configured)

The following advertisement method will be added when 3rd party
support is available:

* cloud-init files will include the interface MTU for each interface.

If advertisement is turned off, it is the responsibility of the
Openstack application to ensure that its VMs have appropriate MTUs
configured.

Regardless of the advertisement setting, Neutron network appliances
such as routers, and advanced service appliances, will have their
interfaces configured to the selected MTU if the MTU is known.  This
implies that the plugging mechanism for the service understands and
correctly implements MTU setting.

For any given MTU that is advertised to the network, the plugin must
be certain that, even in the event of a change of underlying network
topology (e.g. failover to a backup path, or using a different encap
type, or switching from a single-host to multi-host setup when a new
instance is attached to the network), packets of the specified size
transmitted from *any* endpoint will *always* be guaranteed to reach
all desinations.  It is its task to ensure that the MTU is small
enough to work on the selected infrastructure, both inter-host and
intra-host.

An Openstack network typically has physical networks for (a) external
connectivity and (b) provider network use.  These networks may or may
not have the same MTU size as each other and are quite likely to have
an MTU size different from that of the virtual tenant networks.  Link
MTU for physical networks is specified by configuration and will be
used as the maximum and preferred MTU sizes for provider and external
networks on those segments.

Alternatives
------------

Common workarounds for this problem are:

* using config, adding a DHCP option to propagate a 'safe' MTU to all
  VMs, and setting the MTU in various (somewhat driver-specific)
  config parameters to ensure that L2 and L3 network elements work.
  This tends to work for v4 and not v6, as the equivalent option is
  not generally set; it's not fully functional, as routers are unaware
  that their MTU is incorrect; and it's only available to VMs that
  actually support DHCP.  It also does not work if tenant and provider
  networks have different MTUs, or internal networks and the external
  network.

* instructing users, in documentation, to set a 'safe' MTU in all VMs
  (plus the same config option tweaks).  This has the same caveats as
  above, in that the VMs may agree with each other but do not agree
  with routers and this can result in unexpected behaviour.  It does
  not work with provider networks or other configurations where
  different Neutron networks have different MTUs.

* it is possible to use large MTU interfaces and configure the MTU on
  network components such as in-kernel switches and bridges, but again
  routers are not always correctly configured and provider network MTUs
  are commonly wrong.

There are some MTU-related configuration variables, but they serve as
a workaround to MTU issues that arise with software networking
constructs.  If the MTU of the network were known explicitly there
would be no need for this configuration, as the relevant components
could have their MTU explcitly configured to be that of the virtual
network.

When the driver sets the MTU on the network or the tenant provides it,
the values for network_device_mtu and veth_mtu are derived dynamically
for that network and any configuration values are ignored.

Data Model Impact
-----------------

'mtu' (positive integer) attribute added to the data model for
networks.  This stored the determined preferred value for 'mtu',
unless overridden by the user.  It may be indeterminate, in which case
it is stored as NULL.  When NULL, advertisement mechanisms will not
send MTU.

In Phase I, mtu is stored in the database but not exposed over the
API.  It is set at net-create to the preferred MTU provided by the
plugin (if any), and read to provide advertisement via DHCP and RA.

In Phase II, the item becomes visible over the API to all users as a
readonly attribute.  NULL is treated as unset.

In Phase III, the item is writeable over the API, providing the
administrator has explicitly enabled it in policy.json.  It can only
be set in a net-create call and it is passed to the plugin; ML2 will
validate it aginst the plugin-provided max-MTU value and reject any
value that is too high.

REST API Impact
---------------

Phase II only:

The 'mtu' attribute can be read by any user from the network object.
The value is constant over the lifetime of the network.

Phase III only:

For net-create, the 'mtu' attribute may be provided.  It will be
validated against the max calculated MTU within the plugin and may be
rejected as too large.  Its value is immutable after net-create, so
net-update must not accept it.

Security Impact
---------------

There may be some visibility into cloud implementation now that the
user can detect maximum and preferred MTU values, but this is not a
security risk in and of itself.  Consideration should be given to
whether this weakens any other security assumptions.

Notifications Impact
--------------------

None.

IPv6 Impact
-----------

RAs will carry MTU when advertisement is enabled.  An MTU greater
than 1500 will not be accepted by RFC-conforming IPv6 stacks in VMs.
Linux VMs will generally take it if their MTU is larger than 1500 when
the RA is received.  This information should be documented.

Other End User Impact
---------------------

VMs may now receive MTU advertisements in DHCPv4 and IPv6 RA.

Performance Impact
------------------

The current issues with MTU occasionally lead to very low network
performance when large tenant packets are passed over tunnels where
the tunnel packet size exceeds the PMTU of the tunnel.  Correct MTU
behaviour should rectify this, particularly if network plugins prefer
to create networks with MTUs that do not cause packet fragmentation.

Nothing should make performance worse.

Other Deployer Impact
---------------------

The deployer may provide three new configuration variables:

advertise_mtu - default false, and when false, backward compatible; no
effort is made to advertise MTU to VMs via network methods.  When
true, VMs will receive DHCP and RA MTU options when the network's
preferred MTU is known.

path_mtu - for L3 mechanism drivers, determines the maximum
permissible size of an unfragmented packet travelling from and to
addresses where encapsulated Neutron traffic is sent.  Drivers should
calculate maximum viable MTU for validating tenant requests based on
this value (typically path_mtu - max encap header size).  If not
supplied, the path MTU is indeterminate and no calculations will take
place (i.e. MTU requests will be declined for all drivers requiring
it).  (Note that the path MTU is specified and not probed, as it is
impossible to programmatically determine the smallest MTU on the
*worst* possible path between two endpoints, only the path MTU
currently in use.)

segment_mtu - for L2 mechanism drivers (i.e. VLAN), determines the
segment MTU.  If not supplied, the segment MTU is indeterminate and no
calculations will take place (i.e. MTU requests will be declined for
all drivers requiring it).  (Note that the segment MTU is determined
from config and *not* by probing interfaces, as multiple interfaces
can be involved in a network segment.)

Additionally, a new attribute, physnet_mtus, will be added to the
ML2 configuration to accommodate an optional per-physical network
MTU setting.  Any physical networks without a corresponding
physnet_mtus setting will cause the use of the global segment_mtu
value in MTU calculations for that physical network (if the
segment_mtu is unset then no calculation takes place).

Example:
  physnet_mtus = physnet1:1550, physnet2:1500

Or, to set MTU for physnet2 and leave physnet1 as default:
  physnet_mtus = physnet2:1550

This can be used to declare the MTU of physical networks, including
the external network and those used for provider networks.

The values listed above mean that it is possible to calculate the
maximum MTU that a specific network can transit based on the driver or
drivers used to construct it.

The veth_mtu property was previously used to configure the MTU of a
veth interface used in constructing OVS-driver network elements.  This
is no longer required when the network MTU is set.  It will be ignored
in those circumstances, but will be applied if the network mtu cannot
be calculated (e.g. if the path or segment MTUs are required and not set).

The network_device_mtu was previously used to configure the MTU of
interfaces in namespaces that attach to Neutron networks.
After this change, if the driver (or the tenant) has set the MTU of
the network, this will now be used directly.  If indeterminate,
network_device_mtu will still be used.

Developer Impact
----------------

Plugins should add support for the proposed MTU interface.  This blueprint
does not mandate the timetable but offers the facility for them to do so.
We will change at least the ML2 plugin as a sample to allow for testing but
do not guarantee to add support to all plugins.

Drivers should add support for the proposed MTU interface.  We will
change the OVS, LB, VLAN, VXLAN and GRE drivers to add support.

Community Impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ijw-ubuntu

Work Items
----------

This work will be conducted in three distinct phases.

Phase I will involve creating mechanisms to allow calculation of
suitable MTUs within the ML2 plugin and the OVS, Linuxbridge, VLAN,
GRE and VXLAN drivers; the advertisement of those MTUs to VMs; and the
use of those MTUs by the namespace implementations of services
(Neutron routers, DHCP, Metadata).  Note that, per the problem
statement, this *does* require per-network MTU values.  At this point,
backward compatibility mechanisms will be put in place so that drivers
that do *not* support MTU calculation fall back to their current
behaviour.

Phase II will involve enabling the read-only 'mtu' flag on the network
object, so that VMs with issues understanding advertisement mechanisms
can be configured out-of-band to the correct MTU.

Phase III will involve allowing the user to specify their own
selection of MTU, the validation of that MTU against the driver's
permitted values, and the appropriate advertisement of the MTU.  This
will be done initially as experimental code, where the 'mtu'
advertisement will take a value on net-create commands but the actual
writing of the value is disabled in the default policy.json.  This
permits interested parties to use the functionality without enabling
it for the majority of users.

For Kilo, Phase I must be completed, and Phase II is highly desirable.
Phase III has a lower priority.

The tasks for phase I are as follows:

Change the data model in the database to have an 'mtu' flag on
networks, adding upgrade and downgrade scripts

Change the config file reader so that the new format of the
bridge-mappings item is understandable.

Add the path_mtu, segment_mtu and advertise_mtu properties to the
configuration file reader

Change the OVS, LB, VLAN, GRE and VXLAN drivers to correctly calculate
their maximum and preferred MTU values, using bridge-mappings,
path_mtu and segment_mtu as required.

Change the ML2 plugin to correctly determine the maximum and preferred
MTU values for a network based on the capabilities of the drivers in
use for the network, and store that value on the network object in the
database.

Correct the network control and plugging code to correctly set the MTU
on all software network elements that respect it

Add code to the neutron namespaces to correctly set the MTU

Add automatic advertisement, if the MTU is set on a network, from DHCP,
when DHCP is enabled.

Add automatic advertisement, if the MTU is set on a network, from RA,
when RA is enabled.


The tasks for phase II are as follows:

Expose the 'mtu' flag for read on networks


The tasks for phase III are as follows:

Expose the 'mtu' flag for write on net-create

Validate the value passed against the maximum MTU the driver is
capable of

Change the core Neutron code to refuse to create a network when an
'mtu' is provided and a plugin does not know its maximum MTU.

Dependencies
============

None.

Testing
=======

API Tests
---------

Phase I:

None

Phase II:

Test presence and value of new 'mtu' attribute; confirm read-only;
confirm expected value

Phase III:

Test writing to 'mtu' attribute; test it is respected; test it is
correctly validated against maximum and preferred MTU

Validate that max MTU is correctly calculated for VLAN based on
segment mtu for tenant networks and the physical network MTU for
provider networks

Validate that max MTU is correctly calculated and used for the
external network based on the external bridge physical MTU.

Verify physical MTUs default to 1500.

Verify network elements are correctly configured based on the new
config parameters and the old config parameters no longer affect
behaviour.

Verify that a driver and plugin decline a network create with an
untransmissibly large MTU

Functional Tests
----------------

Phase I:

Confirm that VLAN, VXLAN, GRE, OVS and Linuxbridge drivers are
correctly calculating their MTU from the inputs - the configuration
parameters path_mtu, segment_mtu and physnet_mtus

Confirm that, using the above drivers, ML2 correcly calculates the MTU
to use for a variety of network types, including virtual, provider and
external networks

Confirm that the MTU is being correctly advertised by checking the
dnsmasq and RADVD config in use on networks when advertise_mtu is set;
confirm that it is *not* advertised when advertise_mtu is not set

Phase II:

Confirm that the 'mtu' value exposed on a network is correct per
selected network configuration, for a variety of configurations

Phase III:

Ensure appropriate application of MTU to all elemtns when
MTU-specified and a non-specified networks are used.

Tempest Tests
-------------

Tempest scenario tests to ensure:

Phase I:

* When advertisement is on, a packet of that size can be transmitted
  and received across a network between VMs that recognise the
  advertisement type

Phase II:

 * When VMs are manually configured on startup with the 'mtu' value
   seen on the network, with 'advertise_mtu' disabled, a packet of
   that size can be transmitted and received across a network between
   VMs

Phase III:

 * an accepted MTU is transmissible

Documentation Impact
====================

User Documentation
------------------

The new 'mtu' attribute should be documented.
The changed behaviour of DHCP and RA should be documented.

The new 'path_mtu', 'segment_mtu' and 'advertise-mtu' options should
be documented.  The options they replace should be deprecated.

Developer Documentation
-----------------------

Plugin and driver design documents should note the new functionality
to manage MTU.

References
==========

[1] https://blueprints.launchpad.net/neutron/+spec/nfv-vlan-trunks
