..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Add Subnet Allocation to IPAM
=============================

https://blueprints.launchpad.net/neutron/+spec/subnet-allocation

:Author: Carl Baldwin <carl.baldwin@hp.com>
:Co-Author: Ryan Tidwell <ryan.tidwell@hp.com>

The current implementation of IPAM does not provide a mechanism to set up an
address space from which subnets can be allocated.  This will be needed in
order to automatically allocate addresses for subnets instead of requiring
subnet details at the time of creation.

This becomes very important with IPv6 where floating IPs are not implemented.
A tenant may want to create a network with addresses that are routable:  not
just that the addresses are globally unique -- because all IPv6 addresses
should be -- but that the operator can actually route them.

Problem Description
===================

IPAM in Neutron cannot allocate subnets.  Subnet details must be specified by
the End User at the time of subnet creation.

End Users may want to offload the burden of keeping track of subnets and which
addresses are in use.  In this case, the End User should be able to set up a
private address space from which these are automatically allocated.  For IPv4,
this will often be a portion of the RFC1918 address space but doesn't need to
be.  It might be part of a corporate address space which has been delegated to
the cloud.  For IPv6, the End User may want Neutron to automatically calculate
a useable ULA subnet using a pseudo-random algorithm in harmony with RFC4193
[#]_.  This implies that the algorithm for the selection of subnets within the
space is pluggable in some way.

.. [#] http://tools.ietf.org/html/rfc4193

Deployers will set up external networks and may have a chunk of routable
addresses that could be leased or delegated to tenants for use on their
networks.

Neutron needs an API for creating and managaging address spaces and making them
available to tenants.

Proposed Change
===============

Overview
--------

The purpose of this blueprint is to enable creation and management of subnet
allocation pools. This will require changes to the core API and will also require a
modification to subnet creation to allow specifying an address space id and a
prefix length in lieu of actual subnet address details.

A reference implementation for this new feature will be added to implement this
along with the included reference IPAM implementation.

A subnet pool can be shared or not shared.  Only admins can create shared pool.

A quota mechanism will be added for shared pools.  Quotas will be expressed in
terms of the number of minimum atomically allocatable address units.  To keep
the math simple, the unit size will be hard-coded at /32 for IPv4 and /64 for
IPv6.  Counting the total number of addresses with IPv6 will make things
cumbersome since even an unsigned 64 bit integer is not sufficient to express
numbers this large.  It would also require extra complexity around presentation
in order to present these numbers to a user in a way that makes any sense at
all.  The implementation will share code between IP versions.  The only
difference will be the prefix size constant.

The resource quotas are applied to is not the SubnetPool, but rather IP
addresses.  As such, the current quota engine is not able to perform this
operation so management and enforcement should occur in a custom fashion.

Operators may want to charge for allocations (hopefully not with IPv6) but the
mechanism by which they can do this is beyond this bp's scope.

A quota mechanism will be used for the number of SubnetPools which can be
created per tenant just to avoid excessive abuse of the API.

Address Scoping
~~~~~~~~~~~~~~~

There should be one SubnetPool for each address scope.  The pool simply tells
us what addresses are available within the scope for allocation. It is out of
this bp's scope to implement anything more.  However, in the future, there may
be more use cases for allowing routing across scopes.  For example:

#. NAT may be used in IPv4 even without external networks.
#. Two scope owners may agree that routing should be allowed.  This might
   be useful when RBAC is implemented to allow tenants to cross-plug networks.
#. A tenant has some globally routable addresses and wants to work it out with
   the cloud operator to route it externally.

Neutron has been implicitly using a single scope per tenant.  Address
uniqueness is not explicitly enforced in this scope.  Most tenants likely
already do this on their own.  However, Neutron has not been enforcing
uniqueness within a tenant's address scope.

Overlapping addresses will be allowed for IPv4 pools. By default, the pool will
allow overlapping. This can be changed by setting a flag when creating the pool.
Overlapping will not be allowed under any circumstances inside IPv6 pools. However,
there will be no enforcement of uniqueness across pools.

Multinetting on a network using subnets from different pools will be allowed.

Data Model Impact
-----------------

New objects will be added:

SubnetPool

.. csv-table::
    :header: Attribute,Type,Description

    id, UUID, Unique identifier for the pool.
    name, String, Name of the pool
    tenant id, UUID, The tenant owning the space.
    shared, boolean, Whether the pool is shared.
    version, 4 or 6,
    allow overlap, boolean, Whether the pool allows overlapping
    min prefix len, Integer, Largest subnet that can be allocated.
    default prefix len, Integer, The default subnet size to allocate

SubnetPoolRanges

.. csv-table::
    :header: Attribute,Type,Description

    Subnet Pool,UUID, Unique identifier of the SubnetPool
    first ip,Integer,
    last ip,Integer,

SubnetAllocation

.. csv-table::
    :header: Attribute,Type,Description

    id, UUID, Unique identifier for the allocation.
    Subnet Pool, UUID, Unique identifier of the SubnetPool
    IP Version, 4 or 6,
    prefix, Integer, The allocated network address prefix
    prefix len, Integer, The prefix length

The pool_id field will be added to Subnet:

Subnet

.. csv-table::
    :header: Attribute,Type,Description

    All fields as currently implemented, ,
    pool id, UUID, ID of the pool the subnet was allocated from

When the subnet is either not allocated from a pool or is migrated during upgrade,
pool_id will be 'null'.

No data migration is necessary.  The standard script to create the initial
empty tables will be provided.

Needing an availability table like in existing address IPAM is not anticipated.
This will be computed dynamically.  Subnet allocation will be performed much
less often than port allocation and won't be as contentious in the database.
This is an implementation detail to be worked out later.  The availability
table or some other clever solution may be necessary after all.

Avoiding overlap in the reference implementation will be a little trickier than
with simple IP address allocation.  It can't use a simple unique constraint on
the database because we want to support allocating subnets of different sizes,
especially for IPv4.  It could store allocations in terms of the minimum
allocation size.  So, a larger subnet allocation would be stored using multiple
rows in the database.  If SubnetPools were always of reasonable size, storing
availability and allocations in the same table by prepopulating the entire
table might be feasible.  This is an implementation detail that can be worked
out during code review.

If we stored allocations in terms of the minimum allocation size then we will
have problems if that size is updated after the pool was created.  For example,
if the minimum size is increased, then what happens to existing allocations
that are already smaller and don't align with the new size?

REST API Impact
---------------

Subnet details become optional in subnet-create.  Instead, an address space can
be chosen along with a prefix length indicating the size of the subnet desired.
This table summarizes changes to the subnet creation API.

.. csv-table::
    :header: Attribute Name,Type,Access,Default Value,Validation Conversion,Description

    cidr,-,-,allocated automatically,-,-
    address pool,string (UUID),same as cidr,none,Must exist and tenant must have access,Pool to allocate from
    allocation pools,-,-,-,-,-Use 0.0.0.0/NN
    gateway ip,-,-,-,-,Use 0.0.0.0

Allocation pools and gateway ips can still be specified as they are with
current subnet creation.  Since the actual subnet address is not known, they
must be specified using 0 as a wildcard prefix (0.0.0.0/NN) for the subnet where NN is the prefix length
chosen.  The actual network prefix will be filled in when it has been
allocated.  For example, if I send this in a successful subnet create call::

  cidr = 0.0.0.0/25
  address pool = <some id>
  gateway_ip = 0.0.0.1
  allocation_pool = 0.0.0.64 - 0.0.0.126

I might get this back::

  cidr = 10.10.10.128/25
  address pool = <some id>
  gateway_ip = 10.10.10.129
  allocation_pool = 10.10.10.192 - 10.10.10.254

In some cases, the tenant is perfectly content with a default prefix length determined by the pool.
This has utility with IPv6 where the tenant just needs to be allocated a /64.  In such cases,the
pool is configured with a default prefix length and tenants have no need to supply a  prefix length
when requesting a subnet from the pool. For example, a subnet create call using the default prefix
length of the pool (/25 in this case) would look like this:

Request::

  cidr = <not specified>
  address pool = <some id>
  gateway_ip = 0.0.0.1
  allocation_pool = 0.0.0.192 - 0.0.0.254

Allocation::

  cidr = 10.10.10.128/25
  address pool = <some id>
  gateway_ip = 10.10.10.129
  allocation_pool = 10.10.10.192 - 10.10.10.254


The following explains how cidr and address pool can be used
together.  The basic rules are that cidr and prefix are mutually exclusive and
one must be specified.  If a prefix length is specified, a pool must be
specified too. The exception to this is if an option global default pool is
defined in neutron.conf.

.. csv-table::
    :header: cidr, address pool, action

    -, -, Error when no global default pool defined, else subnet with default prefix length is allocated
    -, specified, Allocate using the pool's default prefix length
    specify specific CIDR, -, Same as before.  Uses Implicit tenant address pool.
    specify specific CIDR, specified, Tries allocating specified subnet from pool.
    specify wildcard CIDR, specified, Subnet with requested prefix length from pool.
    specify wildcard CIDR, -, Error when no global default pool defined, else allocate subnet from global pool

New errors from the API are possible:  SubnetPoolNotFound,
PrefixLengthTooBig/Small, NoAddressesAvailable.

New methods need to be added to create and manipulate SubnetPools.

.. csv-table::
    :header: Attribute, Access, Type, Required, CRUD, Default Value, Validation Constraints, Notes

    id, "RO, all", string(UUID), N/A, R, generated, N/A, UUID representing the address space
    name, "RW, owner", string, Yes, CR, N/A, name of the pool
    shared, "RO, all (if True); RW, admin", bool, No, CRU, False, True/False, whether other tenants see it
    version, "RW, owner", integer, Yes, CR, N/A, 4/6, The IP version
    allow overlap, "RW, admin", bool, No, CR, True if version=4, False if version=6, True/False, allow overlapping subnets
    min prefix len, "RW, owner", integer, Yes, CRU, N/A, viable prefix lengths for IP version, The IP version
    default prefix len, "RW, owner", integer, Yes, CRU, determined by min_prefix_len and version, > min_prefix_len & < max version prefix, default prefix allocation len
    ranges, "RW, owner", list(2-tuples or CIDR's), Yes, CRU, N/A, valid non-overlapping ranges

Basically, if shared is True then all tenants can read all fields.  If it is
not true then only the owner can see the pool.  Only the owner is able to
write the fields in any case.  Only admin can write to the shared field.

For IPv4, the prefix lengths should be between 8 and 30.  For IPv6, likely
between 32 and 64.

There are certain parts of the IPv6 address space that are simply not specified
for any kind of use (i.e. outside 2000::/3, fc00:/7 for ULA, and other specified
scopes)[#]_.  Some validation is called for.  The following address spaces should
be allowed:

.. [#] http://www.iana.org/assignments/ipv6-address-space/ipv6-address-space.xhtml

.. csv-table::
    :header: Addresses, Description

    2000::/3,Global unicast
    fc00::/7,ULA addresses which can be routable within sites.

If ranges are updated after the initial creation, nothing will be done about
existing subnet allocations that happen to fall outside of the new ranges.

This bp will start with the same shared model that we have now for networks.
However, an RBAC mechanism can be added later similar to the one proposed for
networks [#]_.

.. [#] https://review.openstack.org/#/c/132661/

Security Impact
---------------

There is an new API.  With any new API, there is the potential for new attacks
on the system.  For example, if someone could obtain control over an address
space, they could shrink it down to nothing and prevent further allocations.

As long as only admins can create shared pools and quotas are in place on the
shared pools, there should be no new vulnerabilities introduced. With that said,
particular attention should be paid during code review to guard against the
introduction of new ones.

Someone with unlimited control of an address space could potentially fill it up
as another way to prevent any further allocations.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

Performance Impact
------------------

Allocation is performed on subnet create.  This may involve a couple of
significant database queries.  Subnet create is not nearly as common as a port
create so this is not expected to be a problem.  Use of optimistic locking
techniques should mitigate the impact.

The new API methods for creating and updating pools aren't expected to be
called often enough to have any significant impact.

We should consider atypical use cases in addition to the typical.  If the
implementation performs very poorly, it could be used as a denial of service
attack and pose a Security Risk. This should be addressed during code review.

IPv6 Impact
-----------

This new feature must work for IPv6 equally as well as IPv4.  This is intended
to enhance the IPv6 experience in Neutron.  There will be no effect on existing
IPv6 features in Neutron.

Other Deployer Impact
---------------------

By default, the cloud system will work like it does today except that tenants
will have the ability to create their own pools without any deployer action.

The deployer may use the shared pools feature to create pools of addresses that
will be available to tenants for use on their networks but is not required to.
They will likely want to use this feature if they want to route to tenant
networks either globally or within the datacenter.

External IPAM may be used with this new API.  Development of external IPAM
drivers is out of the scope of this blueprint.

Developer Impact
----------------

None

Community Impact
----------------

This change was discussed at the Kilo design summit in the pluggable IPAM
session.  It has also been discussed with the IPv6 subteam.  This has been
recognized as a need for the community, especially for IPv6 routing. With
this API, the IPv6 subteam can just use prefix delegation as a mechanism
for handing out allocations, after a user has made a request to this API.

Alternatives
------------

N/A

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  `ryan-tidwell <https://launchpad.net/~ryan-tidwell>`_

Other contributors:
  `carl-baldwin <https://launchpad.net/~carl-baldwin>`_
  L3 Subteam

Work Items
----------

* subnetpools REST API and corresponding DB support
* Adjustments to subnets API and DB schema
* Minimal Horizon enablement to support basic subnet allocation (v4 & v6)
* Tempest test
* Functional Test
* API test

Dependencies
============

This blueprint depends on the work in the neutron-ipam [#]_.  That blueprint
adds some of the frame-work necessary to implement this new feature.

.. [#] https://blueprints.launchpad.net/neutron/+spec/neutron-ipam

Testing
=======

Unit test all new code of course.  This means that the code structure must be
testable.  Will use TDD.

Tempest Tests
-------------

* Create v4 subnetpool
* Create v6 subnetpool
* Allocate v4 subnet from subnetpool
* Allocate v6 subnet from subnetpool

Functional Tests
----------------

* Verify quota enforcement
* Lower tenant quota, verify previously allocated resources intact, verify enforcement of new value
* Assert applicable subnet allocation details not leaking cross-tenant with shared pools
* Create subnetpools with allow_overlap=True and allow_overlap=False, verify allocation of subnets

API Tests
---------

* Verify defaults on subnetpool creation of v4 pools
* Verify defaults on subnetpool creation of v6 pools
* Verify allow_overlap is constrained to False for v6 pools
* Create v4 subnetpool, verify shared and allow_overlap values
* Create v6 subnetpool, verify shared and allow_overlap values
* Allocate v4 subnet from subnetpool, assert success and subnet details
* Allocate v6 subnet from subnetpool, assert success and subnet details


Documentation Impact
====================

User Documentation
------------------

Update networking API reference
Update admin guide

Developer Documentation
-----------------------

N/A

References
==========

.. [#] https://etherpad.openstack.org/p/neutron-ipam
