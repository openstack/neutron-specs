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


Problem description
===================

The ML2 plugin does not adequately support hierarchical network
topologies. A hierarchical network might have different network
segment types (VLAN, VXLAN, GRE, proprietary fabric, ...) at different
levels, and might be made up of one or more top-level static network
segments along with dynamically allocated network segments at lower
levels. For example, traffic between ToR and core switches could
encapsulate virtual networks using VXLAN segments, while traffic
between ToR switches and compute nodes would use dynamically allocated
VLANs segments.

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
particularly important in allowing neutron deployments to scale beyond
the 4K limit per physical network for VLANs. VLAN allocations can be
managed at lower levels of the hierarchy, allowing many more than 4K
virtual networks to exist and be accessible to compute nodes as VLANs,
as long as each link from ToR switch to compute node needs no more
than 4K VLANs simultaneously.

Note that the diagram above shows static VXLAN segments connecting ToR
and core switches, but this most likely isn't the current 'vxlan'
network_type where tunnel endpoints are managed via RPCs between the
neutron server and L2 agents. It would instead be a network_type
specific to both the encapsulation format and the way tunnel endpoints
are managed among the switches. Each network_type value should
identify a well-defined standard or proprietary protocol, enabling
interoperability where desired.


Proposed change
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
and the configuration of deployments that use hierarchical networks,
but does not change any REST APIs in any way.

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
a mechanism driver can only partially establish a binding, it will
instead call continue_binding(segment_id, next_segments_to_bind).

As with set_binding(), the segment_id passed to continue_binding()
indicates the segment that this driver is binding to. The new_segments
parameter specifies the set of network segments that can be used by
the next stage of binding for the port. It will typically contain a
dynamically allocated segment that the next driver can use to complete
the binding.

Currently, mechanism drivers try to bind using the segments from the
PortContext.network.network_segments property. These are the network's
static segments. The new PortContext.segments_to_bind property should
now be used instead by all drivers. For the initial stage of binding,
it will contain the same segments as
PortContext.network.network_segments. But for subsequent stages, it
will contain the segment(s) passed to PortContext.continue_binding()
by the previous stage driver as next_segments_to_bind.

The ML2 plugin currently tries to bind using all registered mechanism
drivers in the order they are specified in the mechanism_drivers
config variable. To support hierarchical binding, a new
port_binding_drivers configuration variable is added that specifies
the sets of drivers that can be used to establish port bindings.

::

  port_binding_drivers = [openvswitch, hyperv, myfabric+{openvswitch|hyperv}]

With this example, ML2 will first try binding using just the
openvswitch mechanism driver. If that fails it will try the hyperv
mechanism driver. If neither of these can bind, it will try to bind
using the myfabric driver. If the myfabric driver partially bindings
(calling PortContext.continue_binding()), then ML2 will try to
complete the binding using the openvswitch driver, and if that can't
complete it, the hyperv driver.

When port_binding_drivers has the default empty value, the
mechanism_drivers value is used instead, with each registered driver
treated as a separate single-item chain.


Alternatives
------------

We originally considered supporting dynamic VLANs by allowing the
(single) bound mechanism driver to provide arbitrary data to be
returned to the L2 agent via the get_device_details RPC. This approach
was ruled out because it requires a single mechanism driver to support
both the ToR switch and the specific L2 agent on the compute
node. This would require separate drivers for each possible
combination of ToR mechanism (switch) and compute node mechanism (L2
agent). The approach described in this specification avoids this
combinatorial explosion by binding separate mechanism drivers at each
level.


Data model impact
-----------------

Whether the data model is impacted remains to be determined during
implementation. If the ML2 plugin only needs to store the details of
the final stage of the binding, then no change should be needed. But
if it needs to store details of all levels, the ml2_port_bindings
table schema will need to be modified.

If we need to persist all levels of the binding, we must store the
driver name and the bound segment ID for each level, but the vif_type
and vif_details only apply to the lowest level. The driver and segment
columns in the ml2_port_bindings table currently each store a single
string, and we need to make sure DB migration preserves already
established bindings. We could add columns to store a list of
additional binding levels, store partial binding data in a separate
table, or possibly redefine the current strings' content to be
comma-separated lists if items.


REST API impact
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
network_type values are being introduced through this specific BP.


Security impact
---------------

None.


Notifications impact
--------------------

None.


Other end user impact
---------------------

None.


Performance Impact
------------------

None.


Other deployer impact
---------------------

No change is required when deploying non-hierarchical network
topologies. To support hierarchical network topologies, the new
port_binding_drivers configuration variable will need to be set to
specify the allowable binding driver chains, and all relevant drivers
will need to be listed in the mechanism_drivers and type_drivers
configuration variables. Additionally, when VLANs are used for the
host-level bindings, L2 agent configurations will be impacted as
described below. Since its multi-level networks at least initially
will involve propriery switches, vendor-specific documentation and
deployment tools will need assist the adminstrator.

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


Developer impact
----------------

Mechanism drivers that support hierarchical bindings will use the
additional driver API call(s). Other drivers will only need a very
minor update to use PortContext.segments_to_bind in place of
PortContext.network.network_segments.


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

This specification should not require much code change, so it can
probably be implemented as a single patch that contains:

1. Update ML2 DB schema to support multi-level bindings, including
   migration (if necessary).

2. Update ML2 driver API.

3. Implement multi-level binding logic.

4. Add unit test for multi-level binding.

5. Update existing drivers to use PortContext.segments_to_bind

If it does turn out that details of each partial binding need to be
persistent, that might be implemented as a separate patch.


Dependencies
============

Usage, and possibly testing, depend on implementation of portions of
https://blueprints.launchpad.net/neutron/+spec/ml2-type-driver-refactor,
in order to support dynamic segment allocation.


Testing
=======

A new unit test will cover the added port binding functionality. No
new tempest tests are needed. Third party CI will cover specific
mechanism drivers that support dynamic segment allocation.


Documentation Impact
====================

The new configuration variables and ML2 driver API changes will need
to be documented. Configuration for specific mechanism drivers
supporting multi-level binding will be documented by those drivers'
vendors.


References
==========
