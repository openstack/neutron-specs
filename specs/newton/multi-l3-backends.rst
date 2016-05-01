..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Flavors for L3 Service Plugin
=============================

https://blueprints.launchpad.net/neutron/+spec/multi-l3-backends

This blueprint covers the introduction of the flavor framework into
the in-tree L3 service plugin. By adding support for flavors, operators
will be able to run multiple L3 drivers in the same deployment.

RFE: https://bugs.launchpad.net/neutron/+bug/1461133


Problem Description
===================

The in-tree L3 service plugin only has support for the Neutron software
routers (legacy/ha/dvr/ha+dvr). If an operator wants to use another
type of router, the service plugin must be changed which will preclude
the use of the reference plugin routers. This current model fails to
address any use cases where a specific routing backend should be used
for only a subset of all of the logical routers in a Neutron deployment.

More use cases can be found in the following etherpad:
https://etherpad.openstack.org/p/neutron-modular-l3-router-plugin-use-cases


Proposed Change
===============

Add flavor support to the in-tree L3 service plugin so requests can be
dispatched to different drivers depending on the flavor associated with
a given router. This will have no impact on other existing L3 service
plugins that exist outside of Neutron.


block diagram::

     +---------------------------------------------------------------+
     |   Neutron Server                                              |
     |                                                               |
     |  +----------------------------------------------------------+ |
     |  | L3 router service plugin                                 | |
     |  |                                                          | |
     |  | +------------------------------------------------------+ | |
     |  | | router driver controller                             | | |
     |  | +------------------------------------------------------+ | |
     |  |                                                          | |
     |  | +-----------+  +----+  +---+  +------+  +----------+     | |
     |  | |single_node|  |dvr |  |ha |  |ha+dvr|  | vendor X | ... | |
     |  | |           |  |    |  |   |  |      |  | driver   |     | |
     |  | +--+--------+  ++---+  ++--+  ++-----+  +----------+     | |
     +--+----|------------|-------|------|---------|---------------+-+
             |            |       |      |         |
             |RPC         |       |      |         | vendor
             |            |       |      |         | specific
             |            |       |      |         | communication
             |            |       |      |         |
             |            |       |      |         V
             V            |       |      |      +------------------+
         +---------+      |       |      |      | Proprietron 9000 |
         |l3 agent |<-----+-------+------+      +------------------+
         +---------+


Within the flavor framework there are flavors and service providers. Flavors
are user-facing via the flavors API and they allow users to choose the type
of router they would like (e.g. 'slow and cheap'). Each flavor is associated
to one or more service profiles. These service profiles define a driver and
some metadata to pass to that driver when it is used.

When a user creates a router with a specific flavor the flavor framework will
look up the service profiles for that flavor and create the router using one
of the associated drivers. Once a driver is selected for a router, the
router/driver association is stored in the DB so any future operations on the
router can lookup the driver without going through the flavor framework.

If a user does not choose a flavor, their router will have a NULL flavor. The
chosen driver will then depend on the operator configured service providers, or
in the backward compat case, it will be determined based on the HA and distributed
configuration flags.

Currently, the flavor framework only supports one service profile
per flavor because it lacks the logic to schedule between profiles [1]
So initially we will only support one service profile per flavor but this
will not prevent future support for scheduling among multiple providers.

The scheduling among multiple providers is not be confused with router
scheduling: the one referenced in the flavors framework is actually used
to determine which service provider to use for a given flavor if it has
multiple service profiles associated with it. Router scheduling is used
in agent-based implementations of L3 where multiple nodes can host the
routing function and thus the server needs to determine which one to
choose based on a specified policy (e.g. random vs load-based policy).

For the sake of this effort, routing scheduling is to be considered
implementation specific and left to the driver. In other words, the
L3 plugin should be made unaware of routing scheduling issues as a
driver may or may not need to schedule the routing function to a node
depending on how the L3 function is actually implemented.

This means that it is within the scope of this blueprint to move the
scheduling implementation details within the default drivers and leave
other drivers in charge of scheduling if they need it.

Responsibility of DB operations
-------------------------------

The L3 service plugin will remain responsible for the CRUD operations on
the router DB objects. This will ensure consistency across common fields
between different flavors/drivers. If a driver needs to perform validation
on input before the record is created in the DB, it can subscribe a
validation callback to the PRECOMMIT events for the router.
Additionally, it is the responsibility of the driver to adjust its own records
based on the PRECOMMIT events if the driver requires storing extra information
about the router.

Each driver may potentially require distributed coordination in order to
dispatch and implement router operations, and as such transactionality of
these operations is left as a driver specific issue. This aspect may be
revised in the future according to findings/experiments developed in the
context of [3]

Comparison to ML2
-----------------

This differs from ML2 in an important aspect. All L3 drivers will not
be called for each operation. Only a single driver will be responsible
for all operations regarding a given router. The driver that is called
is determined by the driver associated to the router when it is created.


Associating a Router to a Driver
--------------------------------

This association is performed one of two ways: either via an explicit
flavor request from the user, or via the distributed/ha attributes and
distributed/ha configuration.

The driver manager will load up four default drivers to represent the
single node, ha, distributed, and HA+distributed router types. The driver
manager will then select the appropriate one in the absense of a flavor
request from the user. In other words, the manager chooses a driver at
creation time based on the user selected flavor, or it will fall back
on HA/DVR attributes, if presented. If everything is ommitted, the
API behavior will remain unchanged and the centralized software router
is created. In other words if a custom driver is loaded up next to the
default providers, a user must specify the flavor in order to select
the requested behavior.

In the event a user requests flags incompatible with the flavor
(e.g. neutron router-create --flavor-id=<ha-id> --distributed=True),
an error will be returned (e.g. invalid input). This can be potentially
solved either client side or server side; however the server side is
to be preferred as it makes the behavior consistent for API users too.

Operators will be able to override the default drivers by explicitly
defining other service_providers in their configuration.

It is up to the operator if he/she wants to actually expose any of the
drivers as flavors to the end user to pick from. The flavor_id associated
with a router is nullable, so an operator can maintain backwards compatiblilty
with the current model by not defining any flavors
(a.k.a 'a tasteless deployment').


Data Model Impact
-----------------

A flavor_id will be added to the Router table with a foreign key constraint to
the flavors table.[2] It will be nullable to preserve compatibility with the
current behavior.

No other modifications to the existing tables are required since the service
type framework already exists and allows a driver to be associated with
a resource_id.


REST API Impact
---------------

All router objects will now have a nullable 'flavor_id' attribute
that indicates the router's flavor. This attribute can also be used
in create/update calls to request a specific flavor. From an API
extension standpoint, the proposed framework will not include any
capability to allow drivers to bring their own extensions to
the L3 models, not now and not in the future. In other words, there
will not be a supported programmatic API for extension pluggability.
Having said that, the existing mechanisms to plug into the Neutron
frameworks can still be exploited as they are available at the
time of the driver development.


RPC API Impact
--------------

The L3 agent based solutions rely on a crucial RPC that allows agents
to sync their state with the Server. This API (sync_state) is heavily
specialized across the hierarchy l3->dvr->ha->ha+dvr. Resolving the
mess is a long term objective of this effort, however, for now the
default software drivers will share this RPC detail and therefore will
continue to rely on the existing behavior of the sync_state operation.

As soon as the first iteration of the framework is complete, strategies
to address this aspect will be explored, so that each driver that needs
to share this RPC call can do so by means of composition rather than
inheritance.


Backward compatibility
----------------------
Flavors and service providers are defined by operators. However, we want
the L3 reference plugin to continue to work without action by the operator
when they upgrade. To that effect, the current L3 plugin driver will
automatically register the single_node, ha, dvr, and dvr+ha drivers as
service providers so the driver manager can work the same as the current
system.


Work Items
----------

* add flavor framework support into the existing L3 plugin
* create separate drivers for single_node, HA, DVR, and HA+DVR routers
* decompose the giant mixin containing logic for all types in the main plugin
  and move type-specific logic into each driver
* add API tests to exercise flavors


References
==========
1. https://github.com/openstack/neutron/blob/33eec87d7822c0915bd45f2c9d2de0b6dc455771/neutron/db/flavors_db.py#L263-L273
2. https://specs.openstack.org/openstack/neutron-specs/specs/liberty/neutron-flavor-framework.html
3. https://bugs.launchpad.net/neutron/+bug/1552680
