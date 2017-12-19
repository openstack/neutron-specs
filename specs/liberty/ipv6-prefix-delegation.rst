..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================
Support IPv6 Prefix Delegation
==============================

https://blueprints.launchpad.net/neutron/+spec/ipv6-prefix-delegation

:Author: John Davidge
:Author: Dane LeBlanc

The IPv6 Prefix Delegation (PD) mechanism (described in RFC3769 [1]_
and RFC3633 [2]_) provides a way of automatically configuring IPv6
prefixes and addresses on routers and hosts. This mechanism can be used
in Neutron to automate the selection of IPv6 prefixes for tenants.

With prefix delegation, a prefix delegating router (referred to as a
prefix delegation server, or PD server in this document) is configured with
a set of prefixes. The prefix delegation process begins when a router
acting as a prefix delegation requester (referred to as a prefix delegation
client, or PD client in this document) requests configuration information
through DHCP messages. When the PD server receives the request, it selects
an available prefix or prefixes for delegation to the PD client.

It should be noted that the prefixes that are delegated by the PD server
are routable, that is, the PD server provides a downstream route to
the prefixes that have been delegated.

This blueprint proposes using IPv6 Prefix Delegation to support automatic
assignment of IPv6 prefixes to tenants/subnets, using an external (outside
of the OpenStack cloud) router to delegate prefixes from a pre-configured
pool of routable prefixes.

Note that it would be possible to have a PD server application that is
managed by Neutron that is running within a Neutron router namespace, and
that is configured using the IPAM Subnet Allocation API [3]_. This
support is out the scope of this blueprint, and will be covered in a
separate blueprint.


Problem Description
===================

In the current Neutron implementation, tenants must supply a prefix when
creating subnets. This is not a big deal for IPv4 private subnets, since
IP addresses on the private subnets can overlap. For IPv6 subnets that use
Global Unicast Address (GUA) format, addresses are globally routable
(unlike IPv6 Unique Local Addresses, or ULAs), so care needs to be taken
to ensure that prefixes chosen by one tenant do not overlap with prefixes
chosen by another tenant.

An OpenStack administrator may want to simplify the process of subnet prefix
selection for the tenants by automatically supplying prefixes for IPv6 subnets
from one or more large pools of pre-configured IPv6 prefixes. This would also
eliminate any potential conflicts in prefix selection.

The IPv6 Prefix Delegation mechanism provides a way of automatically
configuring IPv6 prefixes and addresses on routers and hosts, and can be
used in Neutron to automate the selection of IPv6 prefixes for tenants.
Prefix Delegation would also provide a pathway for adding IPv6 automatic
renumbering for IPv6 addresses in an OpenStack cloud as a future enhancement.

This blueprint proposes using IPv6 Prefix Delegation to support automatic
assignment of IPv6 prefixes to tenants/subnets, using an external router
to delegate prefixes from a pre-configured pool of routable prefixes.


Proposed Change
===============

Prefix Delegation Topology
--------------------------

The prefix delegation topology being proposed in the blueprint is
displayed in the following diagram:

::

                   +-------+-------+
                   |  PD Server    |
                   |  (Running on  |
                   |   external    |
                   |    router)    |
                   +-------+-------+
                           |
                           |
             +-------------+--------------+
             |                            |
      +------+-------+             +------+-------+
      |Neutron Router|             |Neutron Router|
      | (Running PD  |             | (Running PD  |
      | Client and   |             |  Client and  |
      |   RADVD)     |             |    RADVD)    |
      +------+-------+             +------+-------+
 Router Port |                Router Port |
             | /64 Prefix #1              | /64 Prefix #2
             |                            |
      +------+------+               +-----+-------+
      |             |               |             |
 +----+-----+ +-----+----+     +----+-----+ +-----+----+
 | Tenant A | | Tenant A |     | Tenant B | | Tenant B |
 |    VM    | |    VM    |     |    VM    | |    VM    |
 +----------+ +----------+     +----------+ +----------+

Note that in this topology, since the PD server is an entity outside of the
OpenStack cloud, the PD client needs to be authenticated by the PD
server (in order to prevent a rogue requester from depleting available
prefixes), and the identity of the PD server needs to be authenticated
by the PD requesters. This is described in Section 3.6 of RFC3769 [1]_.

The PD Client functionality could be provided by one of several open-source
utilities that support PD client, for example:

* WIDE DHCPv6 (dhcp6c)
  License: BSD
* dibbler (http://klub.com.pl/dhcpv6/)
  License: GNU GPL

For the initial implementation we have chosen to use dibbler, as it has some
important features missing in other clients which make it suitable for this
use case. A plugin framework will be included to allow the addition of other
client options in the future.

A new dibbler-client will be started in the Neutron router namespace whenever a
subnet attached to that router requires prefix delegation.

Detecting When a Prefix has Been Allocated
------------------------------------------

OpenStack/Neutron needs to be aware of prefix allocations as they occur. For
example, Neutron needs to update the IP allocation database with the gateway
IP address that has just been assigned to the internal router port when the
prefix delegation client is allocated a prefix.

Dibbler supports the configuration of a custom script that gets executed
whenever a prefix allocation has been made. Such a custom script can provide
the necessary hook.

Workflow
--------

* Initial Setup

  * Admin configures the default_ipv6_subnet_pool configuration setting
    (described in the IPAM Subnet Allocation specification, [3]_)
    in the neutron.conf configuration file to indicate that prefix
    delegation should be used for the default subnet pool.

* Allocation of a Prefix

  * User/tenant creates an IPv6 subnet while supplying neither an address
    pool nor a prefix. Normally, this indicates to the subnet pool allocation
    infrastructure [3]_ that the prefix should be automatically
    allocated from the default IPv6 allocation pool. However, in this case
    (because of the configuration described in the Initial Setup section),
    allocation for the default IPv6 allocation pool needs to be done
    through prefix delegation. The subnet pool ID for this subnet
    is populated with a special constant to indicate that prefix delegation is
    required for this subnet.
  * User/tenant creates a Neutron router (if one does not already exist).
  * User/tenant attaches the IPv6 subnet to the Neutron router. This causes
    a PD client application to get spawned in the Neutron router space.
  * For a SLAAC/DHCP-stateless subnet, the RADVD configuration is modified
    with a "::/64" entry for the new router interface, and RADVD is signalled
    to reconfigure itself. This new configuration indicates to RADVD that it
    should monitor for an IPv6 address assignment on the new router
    interface, at which time RAs can be initiated to advertise the new
    prefix.
  * Some PD delegation clients provide the capability of running a user
    defined script whenever a prefix delegation has been received. If the
    PD delegation client DOES NOT provide this capability, then a polling
    script will be spawned at this time to periodically poll to detect
    when an IPv6 address has been configured on the internal router interface.
  * The configuration of the PD delegation client is modified to initiate
    a PD request for the new router interface, and the PD delegation
    client is signalled to reconfigure itself, or initialise a new instance
    of itself where dynamic reconfiguration is not available.
  * The PD delegating server receives the PD request, selects an available
    prefix from its block of prefixes, and sends back a
    PD response back to the PD delegation client, indicating the prefix
    to be used.
  * The PD delegating client assigns an address to the router interface
    from the delegated prefix.
  * If the PD delegating client provides the capability of running a user
    defined script whenever a prefix has been delegated, then that script
    will be run to update the OpenStack databases for the subnet's new prefix.
    Otherwise, the polling script will eventually detect the assignment of
    the new IPv6 address to the router interface, and it will update
    the OpenStack databases for the new prefix.
  * For SLAAC or DHCP-stateless subnets, RADVD will assign IPv6 addresses
    from the delegated prefix as new ports are created on the subnet.

Limitations and Future Enhancements
-----------------------------------

* Only /64 prefixes will be delegated in initial release (no sub-delegating).
  Sub-delegation of short prefixes (large address space) into longer
  prefixes by Neutron routers could be added as a future enhancement.

* Only a single pool of /64 prefixes will be supported by the PD server
  in the initial release. Support of multiple pools of prefixes can
  be added in a future release (e.g. to add support of per-tenant prefix
  pools).

* Limits on the number of prefixes that each tenant is allowed to use
  at any time will be maintained using the OpenStack/Neutron subnet quota.

* Prefix delegation will be limited to SLAAC and DHCPv6-stateless subnets
  for this proposal.

Data Model Impact
-----------------

* If an OpenStack provider is upgrading an OpenStack instance from a version
  of Neutron that does not support automatic prefix delegation to a version
  of Neutron that does, no additional migration will be required.
* If an OpenStack provider is upgrading an OpenStack instance from a version
  of Neutron that supports automatic prefix delegation to a newer version,
  then a migration script might be required to re-trigger delegation
  requests for all existing automatic-prefix subnets, effectively causing
  a renumbering.

REST API Impact
---------------

As described in the Subnet Pool Allocation specification [3]_, the
subnet create API will need to allow for the absence of both subnet
prefix and subnet pool ID. Normally, this indicates to Neutron (and the
subnet pool allocation infrastructure) that the prefix for this subnet
should be allocated from a default allocation pool configured for that IP
family. When prefix delegation is configured (see "Initial Setup" above),
however, it is understood that the allocation of IPv6 prefixes in this case
should be done through prefix delegation. In this case, the subnet pool ID
is populated with special constant to mark subnets as requiring prefix
delegation.

As described in the "Allocation of a Prefix" section, the prefix allocation
process will then get triggered after a subnet that is marked for prefix
delegation (i.e. subnet pool ID is populated with the special prefix
delegation constant) is attached to a router.

While the subnet is awaiting assignment of a prefix via prefix delegation,
the response for the subnet-list and subnet-show API/CLI will list the cidr
and allocation_pools for the subnet using a temporary ::/64 prefix.

Security Impact
---------------

When the PD server is an entity outside of the OpenStack cloud (e.g. an
ISP edge router), then the  PD client needs to be authenticated by the PD
server (in order to prevent a rogue requester from depleting available
prefixes), and the identity of the PD server needs to be authenticated
by the PD requesters. This is described in Section 3.6 of RFC3769 [1]_.

Limits on the number of prefixes that each tenant is allowed to use
at any time will be maintained using OpenStack/Neutron subnet quota.


Notifications Impact
--------------------

None.

Other End User Impact
---------------------

Horizon will need to be updated to support this feature for subnet creation.
This Horizon work can be done as part of the changes being made to support
subnet allocation pools.

Performance Impact
------------------

If the PD client does not provide for either asynchronous notification for
the allocation of each prefix, or the configuration of a custom script that
is called upon allocation of prefixes, then a polling script will need to be
spawned whenever a subnet is created that requires prefix delegation.
The extent that these polling scripts have on performance would depend on
the scale of subnets that are being created at any point in time, but it
shouldn't be too significant if the polling interval is a few seconds
or more.

IPv6 Impact
-----------

Yes, this change will modify the Neutron IPv6 implementation.

Other Deployer Impact
---------------------

Deployers wishing to use prefix delegation would need to configure an
external router to act as a PD server, and will need to configure a
pool of available IPv6 prefixes.

Developer Impact
----------------

It may be worth considering adding a mechanism to the pluggable IPAM
infrastructure that would allow for a feature such as prefix delegation
to report prefixes that have been delegated. This would mainly be for
informational purposes, e.g. for displaying what prefixes have been
delegated through prefix delegation.

Community Impact
----------------

This feature is complementary to the IPAM Subnet Allocation feature
[3]_. This implementation could be expanded in the future to support
an internal (Neutron-managed) PD server, and thereby used as part of the
underlying implementation for the IPv6 portion of the IPAM Subnet
Allocation.

Alternatives
------------

Instead of using the default_ipv6_subnet_pool Neutron configuration to
indicate that prefix delegation should be used whenever a subnet create
request is made with neither a subnet prefix nor subnet pool ID, a boolean
attribute could be added to the subnet create API to indicate that
prefix delegation should be used for this subnet. The advantage to such an
API change would be that the user would be able to select between prefix
delegation and a default allocation pool on a per-subnet basis. However,
this benefit probably doesn't outweigh the cost of adding yet another
attribute to the API.

Implementation
==============

Assignee(s)
-----------
Primary assignee:
  Dane LeBlanc
  launchpad-id: leblancd

Other contributors:
  John Davidge
  launchpad-id: john-davidge
  Robert (Bao) Li
  launchpad-id: baoli

Work Items
----------

* Implement PD client configuration.
* Coding/UT for subnet-create and router-interface-add
* Coding/UT for subnet-delete and router-interface-delete
* Write Functional tests
* Write Tempest tests


Dependencies
============

The change in behavior for the subnet create API described in this proposal
will need to build off changes in that API that will be made for IPAM
Subnet Allocation [3]_.


Testing
=======

Tempest Tests
-------------

In order to test functionality with an external router serving as the
PD server, a third party CI system will be needed that incorporates
a router that supports prefix delegation. The Tempest test to run on this
third party CI system would be:

* Configure the external router for prefix delegation service and
  configure a block of IPv6 prefixes.
* Create Neutron virtual routers for 2 separate tenants
* Create a subnet for each tenant, confirm subnet allocated for each
  from the PD server's block of prefixes.

Functional Tests
----------------

Since this feature depends upon a PD server that is running outside of
OpenStack, this feature will require a functional test that runs an
open-source PD server application in a neutron namespace, configured
with a pool of IPv6 prefixes. After the PD server application is running,
the test sequence would be similar to the sequence described in the
previous section.

PD server applications that can be considered for this testing include:

* ISC DHCP server running in V6 mode
  License: ISC license
  (http://www.isc.org/downloads/software-support-policy/isc-license/)
* dibbler (http://klub.com.pl/dhcpv6/)
  License: GNU GPL (should be evaluated to see if appropriate for OpenStack)

API Tests
---------

This feature will require an API test for testing subnet create API with
neither prefix nor allocation pool ID specified.


Documentation Impact
====================

Minor document changes will be required to reflect the configuration
of the default IPv6 subnet pool for prefix delegation.

User Documentation
------------------

Specify any User Documentation which needs to be changed. Reference the guides
which need updating due to this change.

Developer Documentation
-----------------------

The Neutron API docs will need updating to reflect API behavior changes
for subnet create.


References
==========

.. [1] RFC 3769: `Requirements for IPv6 Prefix Delegation
   <http://tools.ietf.org/html/rfc3769>`_

.. [2] RFC 3633: `IPv6 Prefix Options for Dynamic Host Configuration
   Protocol (DHCP) version 6
   <http://tools.ietf.org/html/rfc3633>`_

.. [3] Neutron Blueprint: `Add support for subnet allocation
   <https://blueprints.launchpad.net/neutron/+spec/subnet-allocation>`_

Related Information
-------------------

- RFC 4862: `IPv6 Stateless Address Autoconfiguration
  <http://tools.ietf.org/html/rfc4862>`_
- RFC 4861: `Neighbor Discovery for IP version 6 (IPv6)
  <https://datatracker.ietf.org/doc/rfc4861>`_
- RFC 4291: `IP Version 6 Addressing Architecture
  <http://tools.ietf.org/html/rfc4291>`_
