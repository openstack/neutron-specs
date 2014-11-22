..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================
VLAN trunking networks for NFV
==============================

https://blueprints.launchpad.net/neutron/+spec/nfv-vlan-trunks

Problem Description
===================

Many commonly used Neutron plugin configurations create networks that do
not permit VLAN tagged traffic to transit the network.  This includes ML2
when using the OVS driver or the VLAN driver.  Some plugins, conversely,
are totally fine at passing all forms of ethernet frame.  It's impossible
to tell (via the API) which is in use as a tenant and it's also impossible
to indicate to a plugin what kind of network is required.

VLANs are required for many NFV functions.  This spec proposes making it
possible to request a VLAN transparent network - that is, one where VLAN
tagged traffic will be forwarded to other hosts - when one is requested,
and know that a network is in fact a trunk network.

It is a common requirement of NFV VMs that they talk over many
separate L2 channels.  The number of channels is far in excess of the
number of ports that the VM has and can change over time.  VLAN tags
are often used for separating these channels, so that the number of
ports can be static while still allowing flexibility in the number of
channels.

In Neutron, different network plugins create networks with different
properties, and there is no specific definition of the meaning of the
word 'network'.  Mostly, but not always, networks are L2 broadcast
domains with learning-switch behaviour, although as long as antispoof
filters are in place the network can be implemented either as an L2 or
an L3 domain.  Generally, as long as networks meet a minimum
requirement that IPv4 and IPv6 unicast traffic, ARPs and NDs, work as
expected, no-one will criticise a plugin.  VLAN transparency is not
a part of this current minimum requirement, and some plugins do not,
by oversight or design, allow VLAN packets to flow.  It is not possible
to control this behaviour or detect it via the API, and this proposal aims
to change that.

(Other blueprints propose solutions to decomposing trunks into
networks from its individual VLANs, address management on trunk
networks, and so on.  This makes no attempt to address anything other
than the simple L2 properties of networks.  This spec addresses different
use cases to those that deal with management of ports by Openstack -
specifically, the case where two VLAN-aware VMs wish to talk to each
other over a number of VLANs, possibly with tags that change over time,
and without informing Openstack at each stage of which VLANs and
addresses are in use.)


Proposed Change
===============

This proposal suggests that a request-discover mechanism be put into
place, so that:

* existing plugins that will pass VLAN tagged traffic can identify
  themselves as such
* existing plugins that cannot pass VLAN tagged traffic can also
  identify themselves as such
* legacy plugins that have not been adapted to report their behaviour
  are identifiable
* future plugins can have selective behaviour, where a network may or
  may not pass VLAN traffic depending on user request - which permits
  the plugin to make decisions in favour of efficiency where the
  functionality is not required (such as using an L3 domain).

Additionally, port firewall behaviour - currently undefined for VLAN
tagged packets - will be defined as applying consistently to the outermost
encap of the packet.  That is, antispoof and security groups detailing
matching IP ports will apply to packets with an IP ethertype, but will
*not* apply to packets with a VLAN ethertype containing an IP ethertype.
This is because it is highly unlikely that a VLAN will have the same
address and be serving the same services as the VM's untagged interface
on the same vNIC, and so using the same filtering is almost certain to
render the VLAN tagging useless for practical purposes.

Request
-------

During net-create, the user may at their option request that a VLAN
transparent trunk network is created by passing a 'vlan-transparent'
boolean property on the net-create request, request, set to 'true'.
(Setting the property to 'false' is equivalent to not specifying the
property in the description below.)

Plugins that have not been adapted to understand the property will ignore
this flag and create the network regardless.

Plugins that are aware of the meaning of this property but do not
understand the flag, or cannot deliver VLAN transparent trunk networks,
will refuse to create a network and return an appropriate error to
the user.

Plugins that are aware of this flag and capable of delivering a VLAN
transparent network will do so.

Plugins may change behaviour based on this flag:

- they may create a VLAN transparent network (at potentially higher
  resource cost) if the flag is set, and save on resources
  when it is not set
- they may refuse to attach certain port types to the network
  (e.g. some external attachment ports may not themselves be VLAN
  transparent and therefore should not be attached to transparent
  networks)

Plugins may also only implement one sort of network, either transparent
or not, and decline to create a network when a network is requested
that cannot be implemented by the controller.

In the case that the flag is absent an aware plugin is under no
obligation to deliver a VLAN transparent network (and an unaware
plugin will, naturally, ignore its absence); the returned network may
or may not be VLAN transparent.

Note that the flag cannot be changed by a net-update.  VLAN transparency
must be declared at creation and cannot be changed after this point,
as this would potentially affect the placing of the network and even the
choice of endpoints that VMs plug to.

Response
--------

After network creation, a network may have a property
'vlan-transparent'.

In the case that this property does not exist, the plugin
is a legacy plugin and no determination is possible about whether the
network is capable of passing VLAN tagged packets.

In the case that the property exists and the flag is set to true, then
the plugin is a VLAN aware plugin and (regardless of the request)
has created a network capable of passing VLAN tagged packets.

In the case that the property exists and the flag is set to false,
then the plugin is a VLAN aware plugin, the request did not pass the
'vlan-transparent' flag, and for its own reasons the plugin
elected to create a network without VLAN transparency (typically
because it's being efficient or it's simply not capable of doing so).

Per the description in 'request' above, it is possible that the correct
response is an error indicating inability to act upon the request.

Firewalling
-----------

VLAN tagging will be treated as an opaque encapsulation.

VLAN tagged packets will *not* be firewalled by security groups or other
port based security such as antispoofing, as the packet is not an IP
packet.  (For those drivers capable of passing VLAN tagged packets, this
is - at least sometimes - a change of behaviour.)  This is a deliberate
choice: security groups generally filter packets they understand and pass
packets they don't, so it's consistent (IP-in-MPLS would behave like this)
and it's very likely IP-in-VLAN packets will have different addresses to
any untagged addresses on the same interface, and to each other, and so
Openstack port security will not serve any useful purpose.

This requires validation within the IPTables firewall driver to ensure that
this is the behaviour seen at the moment.

Alternatives
------------

There exists a complementary port-based VLAN spec that permits supplying a
set of networks to a nominated port as a VLAN trunk.  It addresses other
use cases and is not a direct alternative.

Data Model Impact
-----------------

'vlan-transparent' property added to networks, a tri-state boolean.

REST API Impact
---------------

On networks:

+-----------------+--------+--------------------+---------+------------+
|Attribute        |Type    |Access              |Default  |Validation/ |
|Name             |        |                    |Value    |Conversion  |
+=================+========+====================+=========+============+
|vlan-transparent |tristate|write on create, all|absent   |boolean     |
|                 |        |readonly after, all |         |or absent   |
+-----------------+--------+--------------------+---------+------------+

Security Impact
---------------

In current implementations that do pass VLANs, tagged packets'
contents are firewalled.  This is not explicitly documented, but is
one behaviour a user might reasonably expect.

This change proposes treating a VLAN tag as an opaque encapsulation,
and thus VLAN tagged packets would *not* be firewalled by their
content.  Also, security groups only offer IP-based firewalling, so
it would not be possible to block VLAN tagged packets.  That said,
guest OSes can be expected to ignore tagged packets when not
configured for receipt of VLANs, so there should be no impact.

This is in keeping with the behaviour for other non-IP packet types.

Notifications Impact
--------------------

None

IPv6 Impact
-----------

None

Other End User Impact
---------------------

The python-neutronclient should be adjusted to take account of the new
option for net-create and the new property in net-show.

Performance Impact
------------------

May make some plugins more efficient at using network resources.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

None.

Community Impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

ijw-ubuntu

Work Items
----------

* Implement new net-create parameter
* Implement new network property, including database migration script
* Change a sampling of plugins, including the ML2 plugin, to implement
  use of the properties
* Implement a new attr on ML2 drivers indicating whether a driver, when in
  use and potentially responsible for a segment of a network, is capable of
  passing VLAN packets on that network.  The driver may implement settings of
  true or false, or the property may be absent in legacy drivers.  In the case
  that any driver does not have the attribute or has the attribute set to
  false, ML2 will decline to create VLAN transparent networks.
* Of the core drivers, the VLAN and OVS drivers will be marked as not
  supporting VLAN transparent networks and the LB, VXLAN and GRE drivers
  will be marked as supporting VLAN transparent networks.  Other drivers
  will have legacy behaviour.
* Other plugins will have legacy behaviour until updated and no change is
  required of them.

This is *not* changing any driver implementation, merely reporting the
known behaviour of existing drivers.  Thus agents are unaffected.

Dependencies
============

None

Testing
=======

Tempest Tests
-------------

Tempest should confirm that (for a known good networking setup) VLAN
transparent networks can be requested from ML2 and that they work.
Such testing should ideally be host to host, to test both the soft switch
and the hardware configuration.

API Tests
---------

API tests should confirm that the property behaves as described
on net-create, net-update and net-show.

Functional Tests
----------------

Functional tests should confirm that ML2 approves and denies creations correctly.

Documentation Impact
====================

User Documentation
------------------

Requires documentation of the new flag.

Developer Documentation
-----------------------

Requires documentation of plugin and MechanismDriver behaviour and
the requirements for supporting VLAN transparent networks for plugins.

References
==========

 - https://etherpad.openstack.org/p/juno-nfv-bof
 - https://review.openstack.org/#/c/92541/ (composite port support, which
   is independent of the status of an individual network)
