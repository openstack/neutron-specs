..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Reference IPAM driver
==========================================

https://blueprints.launchpad.net/neutron/+spec/reference-ipam-driver

This specification proposes moving Neutron's IPAM code, currently baked in the
database logic, into an appropriate driver which will be called by Neutron
through the interface being defined in [1]_.

Decoupling IPAM logic from the Neutron management layer has been on the list
of "debt repayment" tasks for a long time. It is also a necessary step towards
the implementation of a pluggable IPAM framework, which has been sanctioned
as one of the priorities for the Kilo release cycle [2]_

The following specification provides both a long-term picture for the evolution
of the proposed reference IPAM driver and a short-term detailed description of
the changes proposed as part of this blueprint. The discussion on the long-term
evolution is in the section titled "The Big Picture"; readers concerned
exclusively with changes that will be performed as a part of this specification
can safely skip this section.

Problem Description
===================

Neutron's IPAM logic is baked into the "base" DB class [3]_. This has serious
manageability shortcomings, and is also incompatible with the activity promoted
by [1]_. The latter point makes it pretty much compulsory to transform this
IPAM logic into a driver which gets called through the IPAM interface.

Also, this IPAM logic is currently the first source of "lock wait timeout"
issues in neutron DB because of deeply nested (and long) transactions.

Proposed Change
===============

This specification is complementary to other specifications proposed for the
Kilo release cycle [1]_, [4]_. Thanks to these effort, IPAM will become
"pluggable", meaning that deployers will be able to pick the solution that
best suits them to perform IP address management.
On the other hand, the current logic, used in many production deployments,
will become an IPAM driver as well. More precisely this would be the IPAM
driver that would be used in upstream gate tests.
For this reason we will refer to this IPAM driver as the "reference" driver,
without hopefully abusing this term.

At the end of the Kilo release cycle Neutron will have a driver which does
IP address management exactly in the same way as Neutron does today. As a part
of this blueprint we will also try and fix some of its well known shortcomings,
leveraging community contributions already available for review where possible.

The reference driver will be implemented by:

 * Defining a simple data model extrapolating it from the current "baked-in"
   IPAM data model.

 * Moving out IPAM routines from the "base" DB class [3]_ into the newly
   created IPAM driver

 * Hooking the IPAM driver into the newly developed IPAM interface [1]_.

Driver data model
---------------------------

The data model for the reference IPAM driver will be based on the same
principles as the current IPAM logic, namely:

* Availability ranges created starting from subnets' allocation pools.

* Allocation of IP addresses performed from availability ranges.

* Recalculation of availability ranges only when strictly necessary.

* Allocation outside of allocation pools will be allowed.

However, the reference IPAM driver will not include, at least in its first
iteration, support for subnet pools [8]_. This driver assumes that every
tenant has a single subnet pool whose address range is the entire IP address
space.
Overlapping allocations from this "default" pool are also allowed to ensure
a backward compatible behavior.

Details about schema changes are discussed in the appropriate section.

Code refactoring
----------------

Code refactoring activities are mostly about moving code which is now part
of neutron.db.db_base_plugin_v2.NeutronDbPluginV2.

More specifically the following methods will become part of the IPAM driver:
 * _allocate_ips_for_port
 * _update_ips_for_port
   For the two methods above there will be a slighty different semantic and
   signature to reflect the operations exposed by the new IPAM interface.
 * _test_fixed_ips_for_port
 * _check_unique_ip
 * _allocate_fixed_ips
 * _allocate_specific_ip
 * _generate_ip
 * _try_generate_ip
   The locking query logic used in the above two methods will be replaced with
   a lock free algorithm.
 * _store_ip_allocation
   For this method, some logic will stay in neutron's DB base class, as it will
   be used to provide IP address information for ports to API consumers.
 * _allocate_pools_for_subnet
 * _rebuild_availability_ranges
 * _check_ip_in_allocation_pool
 * _delete_ip_allocation

Plugging into IPAM interface
----------------------------

The reference driver proposed in this document will implement the inferface
defined in [1]_.

The get_subnet operation will be a no-op in the proposed driver.

* allocate_subnet will setup the availability ranges; allocation pools
  should be stored in the base DB class as they are also retrieved via
  the API.

* remove_subnet will ensure no allocation is still active for the subnet
  and clear the availability ranges

* allocate will implement a logic similar to allocate_ips_for_port, with
  the only exception that it will be called not for a port but for every
  IP to configure on a port. Recalculation of availability ranges will
  still occur here.

* deallocate will simply keep removing IP allocation info. It is likely to
  be a no-op for this first implementation of the reference driver.

The Big Picture
---------------

This specification so far discussed how the driver will look in Kilo.
From an architectural perspective it will simply be a class which is loaded
at startup, and where IPAM calls are dispatched.

This still does not address two important aspects of IPAM in Neutron:

* independence from the core plugin

* coordination among multiple servers (or even workers)

In Kilo indeed the calls to the IPAM interface will still be in the base DB
class, which is also the base class for the great majority of Neutron plugins.
This means that the reference IPAM driver will actually be called by the
plugin, and not by the neutron management layer. This applies to any other
IPAM driver as well.
This is an architectural flow, even if not a major one, as it means that:

* The ability of running a specific IPAM driver depends on the core_plugin
  currently configured.

* IPAM interface hooks might need to be explicitly added to plugin modules.
  Indeed with the current codebase, we'll need to insert calls to the IPAM
  interface at least in the base DB class and in the ML2 plugin class [5]_.

* As a plugin will still be free to choose how to use the IPAM interface,
  the interactions defined in [1]_ are therefore plugin-specific. This could
  lead to unexpected behaviors when running specific combinations of plugins
  and IPAM driver.
  For instance, a plugin might redefine core methods such as create_port, and
  bypass calls to the IPAM driver. As another example, a plugin might decide
  to override a method implementation by calling the IPAM driver from within
  a DB transaction, thus creating the conditions for eventlet yields leading
  to deadlocks in the database.

Therefore IPAM driver calls should be performed by the management layer, and
the result of IPAM operation should then be sent to the plugin layer. This
however will require extensive changes in the plugin interface, and is out
of scope for this release cycle. Ideally this would fit very well in a world
where a "v3" plugin interface is defined which allows to have distinct
interfaces for operating on different resource.

When multiple servers or workers (or both) are deployed there will also be
several instances of the reference IPAM driver. With the current implementation
this leads to well known race conditions such as [7]_. The first iteration
of the IPAM reference driver will likely suffer of the same issues.
In subsequent iteration the community might evaluate scalable and reliable
solution which might include either distributed coordination among driver
instances or a "conductor-style" implementation where drivers forward
operations to a centralized IPAM service.

Data Model Impact
-----------------

Ideally there should not be any change in the schema. However, the IPAM
driver will be in charge of managing the IPAvailabilityRanges table.

For IPAllocation and IPAllocationPools, ideally these tables should be
split, since they both have a role in the REST layer and in the IPAM
system.

While splitting this tables provides the best possible separation of concerns
between the plugin and the IPAM driver, it has also some drawbacks. The most
notable ones are duplication of information (eg: an IP address will be both
on the plugin and on the IPAM side), and an increased number of DB operations
for allocating addresses.

For this reason we will assume, at least for this first implementation, that
te Neutron DB layer and the IPAM driver can share database tables. In this
conditions no database schema changes will be needed.

REST API Impact
---------------

The proposed change won't alter a bit the syntax and semantics of the
Neutron API.
Any API change behavior resulting from this implementation will be an
unintentional side effect and should therefore be treated as a high
priority bug.

Security Impact
---------------

We do not expect the reference IPAM driver to be less secure than the current
logic, since most of code will remain unchanged in the driver.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None.

Performance Impact
------------------

The reference driver will contain logic for preventing lock timeout and
reducing the scope of locking queries. This might result in a performance
improvement under heavy load, but we expect this improvement to not be
relevant. The changes in logic from the 'baked IPAM' are aimed mostly
towards reliability.

IPv6 Impact
-----------

No change expected in IPv6-related IPAM features such as SLAAC.
Any change deriving from the implementation of this spec is an unintentional
side effect and should be treated as a high priority bug.

Other Deployer Impact
---------------------

The reference driver will be the default value for the IPAM driver.
In order to avoid potential mayhem is a deployer is running multiple servers,
the same IPAM driver should be used on every server.

Developer Impact
----------------

None

Community Impact
----------------

This will be good for the community as it will provide a single place in the
neutron code tree where IPAM SMEs can contribute.

Note: so far this work is targeting the neutron code tree, but in the future
the IPAM reference driver might be moved outside in line with what's being
proposed for the plugin/driver split blueprint [6]_

Alternatives
------------

Under the hypothesis that there will be a pluggable IPAM interface there is really
no alternative. Unless wrapping every IPAM interface call with a logic like the
following might be considered an alternative:

::
  if cfg.CONF.use_pluggable_ipam:
      self.ipam_driver.do_this(subnet_id, ...)
  else:
      <previous logic in db base class>


The author of this documentation has high hopes that nobody will consider the
snippet above a viable alternative. Even if it can be a short-time measure to
allow for independently develop the pluggable interface, it's something that
should be avoided if possible.

Implementation
==============

The minimum goal for Kilo is to have a driver which manages IPs in the same
way as the "baked" logic does today.

Assignee(s)
-----------

Primary assignee:
  salv-orlando

Work Items
----------

* Implement the driver

* Write unit tests and functional tests for it

* Combine with pluggable interface, face the gate

* According to the strategy chosen in syncing up with effort [1]_, theree might
  be vestigial code to remove

Dependencies
============

* Pluggable IPAM interface [1]_

* Introduction of Subnet Pool concept. This is part of [4]_

Testing
=======

Testing is going to be tricky because the driver can't be fully tested until
the pluggable interface is in place. On the other hand for introducing the
pluggable interface one needs a driver in place. In order to break this
deadlock one possibility is to introduce the driver, but limit only to unit and
functional testing until the IPAM interface is in place.

The following sections will define strategies for both the cases in which the
driver is merged with the pluggable IPAM interface in place and the case in
which the driver is merged without it.

Tempest Tests
-------------

Currently defined scenario tests provide enough coverage for the reference
driver, since it's going to be functionally equivalent to what the current
"baked in" code does today.

Obviously coverage by tempest scenario tests will be possible only when the
pluggable IPAM interface will be in place and dispatching calls to the
reference IPAM driver.

Functional Tests
----------------

Functional testing in the classical neutron sense won't be possible until
the pluggable IPAM interface merges.
However, functional tests intended as exercising the driver trough its
interface should always be possible.

On the other hand, if the IPAM pluggable interface is already in place,
this will allow us to perform functional testing by exercising the neutron
API thus leveraging the current framework.

API Tests
---------

As there will be no API change, no changes to API tests will be needed.

Documentation Impact
====================

User Documentation
------------------

No change.

Developer Documentation
-----------------------

The reference IPAM driver should come with appropriate developer documentation

References
==========

.. [1] Proposed IPAM interface: https://review.openstack.org/#/c/134339
.. [2] Kilo priorities: http://specs.openstack.org/openstack/neutron-specs/priorities/kilo-priorities.html
.. [3] Base DB plugin module: http://git.openstack.org/cgit/openstack/neutron/tree/neutron/db/db_base_plugin_v2.py
.. [4] Pluggable IPAM specification: https://review.openstack.org/#/c/97967/
.. [5] ML2 plugin: http://git.openstack.org/cgit/openstack/neutron/tree/neutron/plugins/ml2/plugin.py
.. [6] Plugin decomposition specification: https://review.openstack.org/#/c/134680/
.. [7] https://bugs.launchpad.net/neutron/+bug/1332923
.. [8] Subnet allocation specification: https://review.openstack.org/#/c/135771/
