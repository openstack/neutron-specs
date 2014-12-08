..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
ML2: Hierarchical Port Binding
==========================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/ml2-hierarchical-port-binding

This blueprint extends ML2 port binding to support hierarchical
network topologies.

Note to reviewers: A similar blueprint was accepted for the Juno
release and successfully implemented, but the review was not completed
early enough in the cycle for the code to be merged. This
specification has been significantly updated to reflect decisions made
during implementation and in response to review comments, as well as
to improve understandability. It describes the final state of the code
reviewed during Juno. Other than rebasing, no significant changes should
be required to the previously reviewed code.

Problem Description
===================

The ML2 plugin does not adequately support hierarchical network
topologies. A hierarchical virtual network uses different network
segments, potentially of different types (VLAN, VXLAN, GRE,
proprietary fabric, ...), at different levels. It might be made up of
one or more top-level static network segments along with dynamically
allocated network segments at lower levels. For example, virtual
network traffic between ToR and core switches could be encapsulated
using VXLAN segments, while traffic for those same virtual networks
between the ToR switches and the compute nodes could use dynamically
allocated VLANs segments.

::

                          +-------------+
                          |             |
                          | Core Switch |
                          |             |
                          +---+-----+---+
                  VXLAN       |     |       VXLAN
                  +-----------+     +------------+
                  |                              |
           +------+-----+                 +------+-----+
           |            |                 |            |
           | ToR Switch |                 | ToR Switch |
           |            |                 |            |
           +---+---+----+                 +---+----+---+
       VLAN    |   |   VLAN           VLAN    |    |    VLAN
       +-------+   +----+                +----+    +------+
       |                |                |                |
  +----+----+      +----+----+      +----+----+      +----+----+
  |         |      |         |      |         |      |         |
  | Compute |      | Compute |      | Compute |      | Compute |
  | Node    |      | Node    |      | Node    |      | Node    |
  |         |      |         |      |         |      |         |
  +---------+      +---------+      +---------+      +---------+

Dynamically allocating segments at lower levels of the hierarchy is
particularly important in allowing Neutron deployments to scale beyond
the 4K limit per physical network for VLANs. VLAN allocations can be
managed at lower levels of the hierarchy, allowing many more than 4K
virtual networks to exist and be accessible to compute nodes as VLANs,
as long as each link from ToR switch to compute node needs no more
than 4K VLANs simultaneously.

Support for allocating dynamic segments was merged to ML2 during
Juno. But there is currently no way for these dynamic segments to be
allocated by one mechanism driver supporting the ToR switch, and used
by a different mechanism driver supporting the L2 agent on the compute
node. What is needed is the ability for an initial mechanism driver to
partially bind to a static segment of a port's virtual network, and
provide a dynamic segment that can be bound by a different mechanism
driver at the next level, continuing until the binding is complete.

Note that the diagram above shows static VXLAN segments connecting ToR
and core switches, but this most likely isn't the current 'vxlan'
network_type where tunnel endpoints are managed via RPCs between the
Neutron server and L2 agents. It would instead be a network_type
specific to both the encapsulation format and the way tunnel endpoints
are managed among the switches. Each network_type value should
identify a well-defined standard or proprietary protocol, enabling
interoperability where desired, and coexistance within heterogeneous
deployments.

Proposed Change
===============

ML2 will support hierarchical network topologies by binding ports to
mechanism drivers and network segments at each level of the
hierarchy. For example, one mechanism driver might bind to a static
VXLAN segment of the network, causing a ToR switch to bridge that
network to a dynamically allocated VLAN on the link(s) to the compute
node(s) connected to that switch, while a second mechanism driver,
such as the existing OVS or HyperV driver, would bind the compute node
to that dynamic VLAN.

Supporting hierarchical network topologies impacts the ML2 driver APIs
and ML2's port binding data model, but does not change Neutron's REST
APIs in any way.

A new function and property will be added to the PortContext class in
the driver API to enable hierarchical port binding.

::

  class PortContext(object):
      # ...

      @abc.abstractmethod
      def continue_binding(self, segment_id, next_segments_to_bind):
          pass

      @abc.abstractproperty
      def segments_to_bind(self):
          pass


The new continue_binding() method can be called from within a
mechanism driver's bind_port() method as an alternative to the
existing set_binding() method. As is currently the case, if a
mechanism driver can complete a binding, it calls
PortContext.set_binding(segment_id, vif_type, vif_details, status). If
the mechanism driver can only partially establish a binding, it will
instead call continue_binding(segment_id, next_segments_to_bind). If
the mechanism driver cannot bind at all, it simply returns without
calling either of these.

As with set_binding(), the segment_id passed to continue_binding()
indicates the segment that this driver is binding to. The new_segments
parameter specifies the set of network segments that can be used by
the next stage of binding for the port. It will typically contain a
dynamically allocated segment that the next driver will use to
continue or complete the binding.

Currently, mechanism drivers try to bind using the segments from the
PortContext.network.network_segments property. These are the network's
static segments. The new PortContext.segments_to_bind property should
now be used instead by all drivers. For the initial stage of binding,
it will contain the same segments as
PortContext.network.network_segments. But for subsequent stages, it
will contain the segment(s) passed as next_segments_to_bind to
PortContext.continue_binding() by the previous stage driver.

The ML2 plugin currently tries to bind using all registered mechanism
drivers in the order they are specified in the mechanism_drivers
config variable. To support hierarchical binding, only a minor change
is needed to avoid any possibility of binding loops in misconfigured
or misbehaving deployments. At each stage of binding, any drivers that
have already bound at a higher level using any of the current level's
set of segments to bind are excluded. This approach allows the same
driver to partially bind at multiple levels of a hierarchical network
using different segments, but not using the same segment. Also, if a
limit on the total number of binding levels is exceeded, binding will
fail and an error will be logged.

Finally, to enable mechanism drivers to see the details of
hierarchical (as well as normal) bindings, the PortContext's
bound_segment, original_bound_segment, bound_driver, and
original_bound_driver properties will be replaced with a new set of
properties:

::

  class PortContext(object):
      # ...

      @abc.abstractproperty
      def binding_levels(self):

      @abc.abstractproperty
      def original_binding_levels(self):

      @abc.abstractproperty
      def top_bound_segment(self):

      @abc.abstractproperty
      def original_top_bound_segment(self):

      @abc.abstractproperty
      def bottom_bound_segment(self):

      @abc.abstractproperty
      def original_bottom_bound_segment(self):


The binding_levels and original_binding_levels properties return a
list of dictionaries describing each binding level if the port is/was
fully or partially bound. Keys for BOUND_DRIVER and BOUND_SEGMENT are
defined, and additional keys can be added in future versions if
needed. The first entry describes the top-level binding, which always
involves one of the port's network's static segments. When fully
bound, the last entry describes the bottom-level binding that supplied
the port's binding:vif_type and binding:vif_details attribute values.

In the presence of hierarchical bindings, some mechanism drivers that
used the old bound_segment, original_bound_segment, bound_driver,
and/or original_bound_driver properties need to access the top-level
binding, while other drivers need to access the bottom-level
binding. Therefore, the old properties will be replaced by new sets of
properties providing access to each of these.

Data Model Impact
-----------------

In order to store multiple levels of binding information for access by
mechanism drivers, changes to the ML2 database schema are
required. The driver and segment columns will be moved from the
existing ml2_port_bindings and ml2_dvr_port_bindings tables to a new
ml2_port_binding_levels table. This table will have port_id, host, and
level columns as primary keys. No separate DVR-specific table will be
needed.

A DB migration will be provided that moves existing binding data to
the new table on upgrade. Downgrades will preserve existing
single-level binding data, but there is no sensible way to preserve
existing multi-level bindings on downgrade.

REST API Impact
---------------

No REST API changes are proposed in this specification.

Using the existing providernet and multiprovidernet API extensions,
only the top-level static segments of a network are accessible. There
is no current need to expose dynamic segments through REST APIs. The
portbindings extension could potentially be modified in the future to
expose more information about multi-level bindings if needed.

As mechanism drivers for specific network fabric technologies are
developed, new network_type values may be defined that will be visible
through the providernet and multiprovidernet extensions. But no new
network_type values are being introduced in this specific BP.

Security Impact
---------------

None.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

Hierarchical network topologies should enable OpenStack deployments to
scale to larger numbers of networks.

Performance Impact
------------------

Hierarchical network topologies may improve tenant network throughput
by utilizing low-overhead encapsulation techiniques such as VLAN at
the compute node level, while specialized network switches handle the
more advanced encapsulation needed to scale to very large numbers of
networks.

IPv6 Impact
-----------

No impact, since this blueprint is at layer 2.

Other Deployer Impact
---------------------

No change is required when deploying non-hierarchical network
topologies. When deploying hierarchical network topologies, additional
mechanism drivers will need to be configured. Additionally, when VLANs
are used for the bottom-level bindings, L2 agent configurations will
be impacted as described in the next few paragraphs.

Mechanism drivers determine whether they can bind to a network segment
by looking at the network_type and any other relevant information. For
example, if network_type is 'flat' or 'vlan', the L2 agent mechanism
drivers look at the physical_network and, using agents_db info, make
sure the L2 agent on that host has a mapping for the segment's
physical_network. This is how the existing mechanism drivers work, and
this will not be changed by this BP.

With hierarchical port binding, where a ToR switch is using dynamic
VLAN segments and the hosts connected to it are using a standard L2
agent, the L2 agents on the hosts will be configured with a mapping
for a physical_network name that corresponds to the scope at which the
switch assigns dynamic VLANs.

If dynamic VLAN segments are assigned at the switch scope, then each
ToR switch should have a unique corresponding physical_network
name. The switch's mechanism driver will use this physical_network
name in the dynamic segments it creates as partial bindings. The L2
agents on the hosts connected to that switch must have a (bridge or
interface) mapping for that same physical_network name, allowing any
of the normal L2 agent mechanism drivers to complete the binding.

If dynamic VLAN segments are instead assigned at the switch port
scope, then each switch port would have a corresponding unique
physical_network name, and only the host connected to that port should
have a mapping for that physical_network.

Since multi-level network deployments will typically involve a
particular vendor's propriery switches, that vendor should supply
documentation and deployment tools to assist the administrator.

Developer Impact
----------------

Mechanism drivers that support hierarchical bindings will use the
additional driver API call(s). Other drivers will only need a very
minor update to use PortContext.segments_to_bind in place of
PortContext.network.network_segments, and the new properies for
accessing the top or bottom bound segment or driver.

Community Impact
----------------

This change provides new capabilities for ML2 mechanism drivers to
support hierarachical network topologies. This supports the
community's interest in allowing Neutron to scale to larger
deployments. It has been discussed at ML2 sub-team meetings, and at
the Juno and Kilo summits.

Alternatives
------------

The alternative is for vendors that need to support hierarchical
network topologies to do so through monolithic plugins rather than
ML2, or through ML2 mechanism drivers that require customized L2
agents.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  rkukura

Other contributors:
  asomya

Work Items
----------

As was done during Juno, the implementation will consist of a series of three patches:

* The driver API changes
* The DB schema changes
* The binding logic changes

Dependencies
============

Use of this feature requires the dynamic segment feature that was
merged during Juno.

Testing
=======

Unit tests will verify all the new driver API methods and properties,
and that hierarchical port bindings can be established with a test
mechanism driver. Real mechanism drivers using hierarchical bindings
will be tested in the correspinding 3rd party CI, where the required
switch hardware is available.

Tempest Tests
-------------

No new tempest tests are required because there is no change in
application level behavior. 3rd party CI testing using the existing
tempest tests will adequately exercise the hierarchical port binding
functionality.

Functional Tests
----------------

No new functional tests are required because there are no system level
interactions that cannot be tested in unit tests.

API Tests
---------

There are no REST API changes to test.

Documentation Impact
====================

Deployer documentation is the responsibility of 3rd parties that
provide mechanism drivers that establish hierarchical bindings.

User Documentation
------------------

No user documentation changes are required.

Developer Documentation
-----------------------

The primary developer documentation is derived from the doc strings in
the driver API source file, which describe the APIs and their behavior
in detail. If an ML2 driver writers guide is ever written, it will
need to cover hierarchical port binding.

References
==========

None.
