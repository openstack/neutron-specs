..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
QoS minimum bandwidth allocation in Placement API
=================================================

https://bugs.launchpad.net/neutron/+bug/1578989

This spec describes how to model, from Neutron, new resource providers
in Placement API to describe bandwidth allocation.

Problem Description
===================

Currently there are several parameters, quantitative and qualitative,
that define a Nova server and are used to select the correct host
and network backend devices to run it. Network bandwidth is not yet
among these parameters. This allows situations where a physical network
device could be oversubscribed.

This spec addresses managing the bandwidth on the first physical device
ie. the pyhsical interface closest to the nova server. Managing bandwidth
further away, for example on the backplane of a Top-Of-Rack switch or
end-to-end, is out of scope here.

Guaranteeing bandwidth generally involves enforcement of constraints on
two levels.

* placement: Avoiding oversubscription when placing (scheduling) nova servers
  and their ports.

* data plane: Enforcing the guarantee on the physical network devices.

This spec addresses placement enforcement only. (Data plane enforcement
is covered by [4]_.) However the design must respect that users are
interested in the joint use of these enforcements.

Since the placement enforcement itself is a Nova-Neutron cross-project
feature this spec is meant to be read, commented and maintained together
with its Nova counterpart: `Network bandwidth resource provider` [2]_.

This spec is based on the approved Neutron spec `Add a spec for strict
minimum bandwidth support` [3]_. The aim of the current spec is not
to redefine what is already approved in [3]_, but to specify how it is
going to be implemented in Neutron.

Use Cases
---------

The most straightforward use case is when a user, who has paid for a
premium service that guarantees a minimum network bandwidth, wants to
spawn a Nova server. The scheduler needs to know how much bandwidth is
already in use in each physical network device in each compute host and
how much bandwidth the user is requesting.

Data plane only enforcement was merged in Newton for SR-IOV egress
(see `Newton Release Notes` [6]_).

Placement only enforcement may be a viable feature for users able
to control all traffic (e.g. in a single tenant private cloud). Such
placement only enforcement can be also used together with the bandwidth
limit rule. The admin can set two rules in a QoS policy, both with
the same bandwidth values and then each server on such chosen compute
host will be able to use at most as much bandwidth as it has guaranteed.

Proposed Change
===============

1. The user must be able to express the resource needs of a port.

   1. Extend ``qos_minimum_bandwidth_rule`` with ingress direction.

      Unlike enforcement in the data plane, Placement can handle both
      directions by the same effort.

   2. Mark ``qos_minimum_bandwidth_rule`` as supported QoS
      policy rule for each existing QoS driver.

      Placement enforcement is orthogonal to backend mechanisms. A user
      can have placement enforcement for drivers not having data plane
      enforcement (yet).

   Due to the fact that we exposed (and likely want to expose further)
   partial results of this development effort to end users, the meaning
   of a ``qos_minimum_bandwidth_rule`` depends on OpenStack version,
   Neutron backend driver and the rule's direction. A rule may be enforced
   by placement and/or on the data plane. Therefore we must document, next
   to the already existing support matrix in the `QoS devref` [10]_, which
   combinations of versions, drivers, rule directions and (placement and/or
   data plane) enforcements are supported.

   Since Neutron's choice of backend is hidden from the cloud user, the
   deployer must also clearly document which subset of the above support
   matrix is applicable for a cloud user in a particular deployment.

2. Neutron must convey the resource needs of a port to Nova.

   Extend port with attribute ``resource_request`` according to section
   'How required bandwidth for a Neutron port is modeled' below. This
   attribute is computed, read-only and admin-only.

   Information available at port create time (ie. before the port
   is bound) must be sufficient to generate the ``resource_request``
   attribute.

   The port extension must be decoupled from ML2 and kept
   in the QoS service plugin. One way to do that is to use
   ``neutron.db._resource_extend`` like ``trunk_details`` uses it.

3. Neutron must populate the Placement DB with the available resources.

   Report information on available resources to the Placement service
   using the `Placement API` [1]_. That is information about the physical
   network devices, their physnets, available bandwidth and supported
   VNIC types.

   The cloud admin must be able to control (by configuration) what is
   reported to Placement. To ease the configuration work autodiscovery
   of networking devices may be employed, but the admin must be able to
   override its results.

Which devices and parameters will be tracked
--------------------------------------------

Even inside a compute host many networking topologies are possible.
For example:

1. OVS agent: physical network - OVS bridge - single physical NIC (or a bond):
   1-to-1 mapping between physical network and physical interface

2. SR-IOV agent: physical network - one or more PFs:
   1-to-n mapping between physical network and physical interface(s)
   (See `Networking Guide: SR-IOV` [7]_.)

Each Neutron agent (Open vSwitch, Linux Bridge, SR-IOV) has a
configuration parameter to map a physical network with one or more
provider interfaces (SR-IOV) or a bridge connected to a provider interface
(Open vSwitch or Linux Bridge).

OVS agent configuration::

    [ovs]
    # bridge_mappings as it exists already.
    bridge_mappings = physnet0:br0,physnet1:br1

    # Each right hand side value in bridge_mappings:
    #   * will have a corresponding resource provider created in Placement
    #   * must be listed as a key in resource_provider_bandwidths

    resource_provider_bandwidths = br0:EGRESS:INGRESS,br1:EGRESS:INGRESS

    # Examples:

    # Resource provider created, no inventory reported.
    resource_provider_bandwidths = br0
    resource_provider_bandwidths = br0::

    # Report only egress inventory in kbps (same unit as in the QoS rule API).
    resource_provider_bandwidths = br0:1000000:

    # Report egress and ingress inventories in kbps.
    resource_provider_bandwidths = br0:1000000:1000000

    # Later we may introduce auto-discovery (for example via ethtool).
    # We reserve the option to make auto-discovery the default behavior
    # when it is implemented.
    resource_provider_bandwidths = br0:auto:auto

SR-IOV agent configuration::

    [sriov_nic]
    physical_device_mappings = physnet0:eth0,physnet0:eth1,physnet1:eth2

    resource_provider_bandwidths = eth0:EGRESS:INGRESS,eth1:EGRESS:INGRESS

How required bandwidth for a Neutron port is modeled
----------------------------------------------------

The required minimum network bandwidth needed for a port is modeled
defining a QoS policy along with one or more QoS minimum bandwidth rules
[4]_. However neither Nova nor Placement know about any QoS policy
rule directly. Neutron translates the resource needs of a port into a
standard port attribute describing the needed resource classes, amounts
and traits.

In this spec we assume that a single port requests resources from a
single RP. Later we may allow a port to request resources from multiple RPs.

The resources needed by a port are expressed via the new attribute
``resource_request`` extending the port as follows.

Figure: resource_request in the port

.. code-block:: python

    {"port": {
        "status": "ACTIVE",
        "name": "port0",
        ...
        "device_id": "5e3898d7-11be-483e-9732-b2f5eccd2b2e",
        "resource_request": {
            "resources": {
                "NET_BW_IGR_KILOBIT_PER_SEC": 1000,
                "NET_BW_EGR_KILOBIT_PER_SEC": 1000 },
            "required": ["CUSTOM_PHYSNET_NET0", "CUSTOM_VNIC_TYPE_NORMAL"]}
    }}

The ``resource_request`` port attribute will be implemented by a new
API extension named ``port-resource-request``.

If a nova server boot request has a port defined and this port has a
``resource_request`` attribute, that means the Placement Service must
enforce the minimum bandwidth requirements.

A host will satisfy the requirements if it has a physical network
interface RP with the following properties. First, inventory of the
new ``NET_BANDWIDTH_*`` resource classes and there is enough bandwidth
available as shown in the 'Networking RP model' section. If a host doesn't
have an inventory of the requested network bandwidth resource class(es),
it won't be a candidate for the scheduler. Second, the physical network
interface RP must have all the traits associated with it as listed in the
``required`` field of the ``resource_request`` attribute.

We propose two kinds of custom traits. First to express and request support
for certain ``vnic_types``. This trait uses prefix ``CUSTOM_VNIC_TYPE_``.
The ``vnic_type`` is then appended in all upper case.
For example:

* ``CUSTOM_VNIC_TYPE_NORMAL``
* ``CUSTOM_VNIC_TYPE_DIRECT``

Second we'll use traits to decide if a segment of a network (identified
by its physnet name) is connected on the compute host considered in
scheduling.  This trait uses prefix ``CUSTOM_PHYSNET_``. The physnet name
is then appended in all upper case, any characters prohibited in traits must
be replaced with underscores.

For example:

* ``CUSTOM_PHYSNET_PUBLIC``
* ``CUSTOM_PHYSNET_NET1``

If a nova server boot request has a network defined and this network has
a ``qos_minimum_bandwidth_rule``, that boot request is going to fail as
documented in the 'Scoping' section of [2]_ until Nova is refactored to
create the port earlier (that is before scheduling). See also `SPEC:
Prep work for Network aware scheduling (Pike)` [11]_.

For multi-segment Neutron networks each static segment's physnet trait
must be included in the ``resource_request`` attribute in a format that
we can only specify after Placement supports request matching logic
of ``any(traits)``. See `any-traits-in-allocation_candidates-query` [9]_.

Reporting Available Resources
-----------------------------

Some details of reporting are described in the following sections of [2]_:

* Neutron agent first start

* Neutron agent restart

* Finding the compute RP

Details internal to Neutron are the following:

Networking RP model
~~~~~~~~~~~~~~~~~~~

We made the following assumptions:

* Neutron supports the ``multi-provider`` extension therefore a single
  logical network might map to more than one physnet. Physnets of
  non-dynamic segments are known before port binding. For the sake of
  simplicity in this spec we assume each segment directly connected to a
  physical interface with a mimimum bandwidth guarantee is a non-dynamic
  segment. Therefore those physnets can be included in the port's
  ``resource_request`` as traits.

* Multiple SRIOV physical functions (PFs) can give access to the same
  physnet on a given compute but those PFs always implement the same
  ``vnic_type``. This means that using only physnet traits in Placement
  and in the port's resource request does not select one PF unambiguously
  but it is not a problem as both PFs are equivalent from resource
  allocation perspective.

* Two different backends (e.g. SRIOV and OVS) can give access to the same
  physnet on the same compute host. In this case Neutron selects the
  backend based on ``vnic_type`` of the Neutron port specified by the
  end user during port create. Therefore physical device selection during
  scheduling should consider the ``vnic_type`` of the port as well. This
  can be done via the ``vnic_type`` based traits previously described.

* Two different backends (e.g. OVS and LinuxBridge) can give access to
  the same physnet on the same compute host while they are also
  implementing the same ``vnic_type`` (e.g. ``normal``). In this
  case the backend selection in Neutron is done according to
  the order of ``mechanism_drivers`` configured by the admin in
  ``neutron.conf``. Therefore physical device selection during scheduling
  should consider the same preference order. As the backend order is
  just a preference but not a hard rule supporting this behavior is *out
  of scope* in this spec but in theory it can be done by a new weigher
  in nova-scheduler.

Based on these assumptions, Neutron will construct in Placement a RP tree
as follows:

Figure: networking RP model

.. code::

  Compute RP (name=hostname)
   +
   |
   +-------+Network agent RP (for OVS agent), uuid = agent_uuid
   |          inventory: # later, model number of OVS ports here
   |             +
   |             |
   |             +------+Physical network interface RP,
   |             |       uuid = uuid5(hostname:br0)
   |             |         traits: CUSTOM_PHYSNET_1, CUSTOM_VNIC_TYPE_NORMAL
   |             |         inventory:
   |             |         {NET_BW_IGR_KILOBIT_PER_SEC: 10000,
   |             |          NET_BW_EGR_KILOBIT_PER_SEC: 10000}
   |             |
   |             +------+Physical network interface RP,
   |                     uuid = uuid5(hostname:br1)
   |                       traits: CUSTOM_PHYSNET_2, CUSTOM_VNIC_TYPE_NORMAL
   |                       inventory:
   |                       {NET_BW_IGR_KILOBIT_PER_SEC: 10000,
   |                        NET_BW_EGR_KILOBIT_PER_SEC: 10000}
   |
   +-------+Network agent RP (for LinuxBridge agent), uuid = agent_uuid
   |             +
   |             |
   |             +------+Physical network interface RP,
   |                     uuid = uuid5(hostname:virbr0)
   |                       traits: CUSTOM_PHYSNET_1, CUSTOM_VNIC_TYPE_NORMAL
   |                       inventory:
   |                       {NET_BW_IGR_KILOBIT_PER_SEC: 10000,
   |                        NET_BW_EGR_KILOBIT_PER_SEC: 10000}
   |
   +-------+Network agent RP (for SRIOV agent), uuid = agent_uuid
                 +
                 |
                 +------+Physical network interface RP,
                 |       uuid = uuid5(hostname:eth0)
                 |         traits: CUSTOM_PHYSNET_2, CUSTOM_VNIC_TYPE_DIRECT
                 |         inventory:
                 |         {VF: 8, # VF resource is out of scope
                 |          NET_BW_IGR_KILOBIT_PER_SEC: 10000,
                 |          NET_BW_EGR_KILOBIT_PER_SEC: 10000}
                 |
                 +------+Physical network interface RP,
                 |       uuid = uuid5(hostname:eth1)
                 |         traits: CUSTOM_PHYSNET_2, CUSTOM_VNIC_TYPE_DIRECT
                 |         inventory:
                 |         {VF: 8, # VF resource is out of scope
                 |          NET_BW_IGR_KILOBIT_PER_SEC: 10000,
                 |          NET_BW_EGR_KILOBIT_PER_SEC: 10000}
                 |
                 +------+Physical network interface RP,
                         uuid = uuid5(hostname:eth2)
                           traits: CUSTOM_PHYSNET_3, CUSTOM_VNIC_TYPE_DIRECT
                           inventory:
                           {VF: 8, # VF resource is out of scope
                            NET_BW_IGR_KILOBIT_PER_SEC: 10000,
                            NET_BW_EGR_KILOBIT_PER_SEC: 10000}

Custom traits will be used to indicate which physical network a given
Physical network interface RP is connected to, as previously described.

Custom traits will be used to indicate which ``vnic_type`` a backend
supports so different backend technologies can be distinguished, as previously
described.

The current purpose of agent RPs is to allow us detecting the deletion of an
RP. Later we may also start to model agent-level resources and capabilities.

Report directly or indirectly
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Considering only agent-based MechanismDrivers we have two options:

* direct: The agent reports resource providers, traits and inventories
  directly to the Placement API.

* indirect: The agent reports resource providers, traits and
  inventories to Neutron-server which in turn reports the information
  to the Placement API.

Both have pros and cons. Direct reporting involves fewer components
therefore it's more efficient and more reliable. On the other hand
freshness of the resource information may be important information in
itself. Nova has the compute heartbeat mechanism to ensure scheduler
considers the live Placement records only. In case freshness of Neutron
resource information is needed the only practical way is to build
on the Neutron-agent heartbeat mechanism. Otherwise the reporting
and heartbeat mechanism would take different paths. If resource
information is reported through the agent heartbeat mechanism then
freshness of resource information is known by Neutron-server and other
components (for example a nova scheduler filter) could query it from
Neutron-server.

When Placement and nova-scheduler choose to allocate the requested
bandwidth on a particular network resource provider (that represents
a physical network interface) that choice has its implications on:

* Neutron-server's choice of a Neutron backend for a port.
  (vif_type, vif_details)
* Neutron-agent's choice of a physical network interface.
  (Only in some cases like when multiple SR-IOV PFs back one physnet.)

The later choices (of neutron-server and neutron-agent) must respect the
first (in the allocation), otherwise resources could be used somewhere
else than allocated.

The choice in the allocation can be easily communicated to Neutron
using the chosen network resource provider UUID if this UUID is known
to both Neutron-server and Neutron-agent. If available resources are
reported directly from Neutron-agent to Placement then Neutron-server
may not know about resource provider UUIDs. Therefore indirect reporting
is recommended.

Even when reporting indirectly we must keep the (Neutron reported part
of the) content of the Placement DB under the as direct as possible control
of Neutron agents. It is best to keep Neutron-server in a basically
proxy-like role.

Content and format of resource information reported (from agent to server)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We propose to extend the ``configurations`` field of the agent heartbeat
RPC message.

Beyond the agent's hardcoded set of supported ``vnic_types`` the following
agent configuration options are the input to extend the heartbeat message:

* ``bridge_mappings`` or ``physical_device_mappings``
* ``resource_provider_bandwidths``
* If needed further options controlling inventory attributes like:
  ``allocation_ratio``, ``min_unit``, ``max_unit``,
  ``step_size``, ``reserved``

Based on the input above the ``configurations`` dictionary of the
heartbeat message shall be extended with the following keys:

* (custom) ``traits``
* ``resource_providers``
* ``resource_provider_inventories``
* ``resource_provider_traits``

The values must be (re-)evaluated after the agent configuration is
(re-)read. Each heartbeat message shall contain all items known by
the agent at that time. The extension of the ``configurations`` fields
intentionally mirrors the structure of the placement API (and does not
directly mirror the agent configuration format, though can be derived
from it). The values of these fields shall be formatted so they can be
readily pasted into requests sent to the Placement API.

Agent resource providers shall be identified by their already existing
Neutron agent UUIDs as shown in the 'Networking RP model' section above.

Neutron-agents shall generate UUIDs for physical network interface
resource providers. Version 5 (name-based) UUIDs should be used by
hashing names like ``HOSTNAME:OVS-BRIDGE-NAME`` for ovs-agent and
``HOSTNAME:PF-NAME`` for sriov-agent since this way the UUIDs will
be stable through an agent restart.

Please note that the agent heartbeat message contains traits and their
associations with resource providers, but there's no traits directly
listed in the agent configurations. This is possible because both physnet
and ``vnic_type`` traits we'll use can be inferred from already known
pieces of information.

Synchronization of resource information reported
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Ideally Neutron-agent, Neutron-server and Placement must have the same
view of resources. We propose the following synchronization mechanism
between Neutron-server and Placement:

Each time Neutron-server learns of a new agent it diffs the heartbeat
message (for traits, providers, inventories and trait associations)
with all objects found in Placement under the agent RP. It creates
the objects missing from Placement. It deletes those missing from the
heartbeat. It updates the objects whose attributes are different in
Placement and the heartbeat.

At subsequent heartbeats received Neutron-server diffs the new and
the previous heartbeats. If nothing changed no Placement request
is sent. If a change in heartbeats is detected Neutron sends the
appropriate Placement request based on the diff of heartbeats using
the last seen Placement generation number. If the Placement request is
successful Neutron stores the new generation number. If the request
fails with generation conflict Neutron falls back to diffing between
Placement and the heartbeat.

Progress or block until the Compute host RP is created
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Neutron-server cannot progress to report resource information until
the relevant Nova-compute host RP is created. (The reason being the
Nova-compute host RP UUID is unpredictable to Neutron.) We believe
that while waiting for the Nova-compute host RP a Neutron-server can
progress with its other functions.

Port binding changes
--------------------

The order of relevant operations are the following:

1. Placement DB is populated with both compute and network resource
   information.

2. Triggered by a nova server boot Placement selects a list of candidates.

3. Scheduler chooses exactly one candidate and allocates it in a single
   transaction. (In some complex nova server move cases the conductor
   may allocate but that's unimportant here.)

4. Neutron binds the port.

In steps (2) and (3) the selection includes the choice of RPs representing
network backend objects (beyond the obvious choice of compute host). This
naturally conflicts with Neutron's current port binding mechanism.

To solve the conflict we must make sure:

* Placement must produce candidates whose ports later can be bound by
  Neutron. (At least with roughly the same probability as scheduler is
  able today.)

* The choices made by Placement and made by Neutron port binding must
  be the same. Therefore the selection must be coordinated.

  If more than one Neutron backend can satisfy the resource requirements
  of a port on the same host then it cannot happen that Placement chooses
  one, but Neutron binds another.

  One way to do that is for Neutron-server to read (from Placement)
  the allocation of the port currently being bound and let it influence
  the binding. However this introduces a slow remote call in the middle of
  port binding therefore it is not recommended.

  Another way is to pass down part of the allocation record in the
  call/message chain leading to the port binding PUT request. In the
  port binding PUT request use the binding_profile attribute. That way
  we would not need a new remote call, just to add an argument/payload
  to already existing calls/messages.

  The Nova spec ([2]_) proposes that the resources requested for a port
  are included in a numbered request group (see `Granular Resource Request
  Syntax` [8]_). A numbered request group is always matched by a single
  resource. In general Neutron needs to know which resource provider
  matched the numbered request group of the port.

  To express the choice made by placement and nova-scheduler we propose
  to add an ``allocation`` entry to ``binding_profile``::

      {
        "name": "port with minimum bw being bound",
        "id": ...,
        "network_id": ...,
        "binding_profile": { "allocation": RP_UUID }
      }

  If a port has the ``resource_request`` attribute then it must
  be bound with ``binding_profile.allocation`` supplied. Otherwise
  ``binding_profile.allocation`` must not be present.

  Usually ML2 port binding tries the mechanism drivers in their
  configuration order until one succeeds to set the binding. However
  if a port being bound has ``binding_profile.allocation`` then only a
  single mechanism driver can be tried - the one implicitly identified by
  ``RP_UUID``.

  In case of hierarchical port binding ``binding_profile.allocation``
  is meant to drive the binding only on the binding level that represents
  the closest physical interface to the nova server.

Out of Scope
------------

Minimum bandwidth rule update:
When a minimum bandwidth rule is updated, the ML2 plugin will list the bound
ports with this QoS policy and rule attached and will update the Allocation
value. The `consumer_id` of each Allocation is the `device_id` of the port.
This is out of scope in this spec and should be done during the work related
to `os-vif migration tasks` [5]_.

Trunk port:
Subports of a trunk port are unknown to Nova. Allocating resources for
subports is a task for Neutron only. This is out of scope too.

Testing
-------

* Unit tests.

* Functional tests.

  * Agent-server interactions.

* Fullstack.

  * Handling agent failure cases.

* Tempest API tests.

  * Port API extended with ``resource_request``.
  * Extensions of ``binding_profile``.

* Tempest scenario tests.

  * End-to-end feature test.

In test frameworks where we cannot depend on Nova we can mock it away by:

* Creating and binding the port as Nova would have done it, including.

  * Setting its ``binding_profile``.
  * Setting its ``binding_host_id`` as if Placement and Scheduler would
    have chosen the host.

Upgrade
-------

* When upgrading a system with ``minimum_bandwidth`` rules to support
  both data plane and placement enforcement we see two options:

  1. It is the responsibility of the admin to create the
     allocations in Placement for all ports using ``minimum_bandwidth``
     rules. Please note: this assumes that bandwidth is not overallocated
     at the time of upgrade.

  2. Add tooling for 1. as described in the 'Upgrade impact' section of [2]_.

* The desired upgrade order of components is the following:
  Placement, Nova, Neutron

  If for some reason the reverse Neutron-Nova order is desired, then the
  Neutron port API extension of ``resource_request`` must not be turned on
  until both components are upgraded.

* Neutron-server must be able to handle agent heartbeats both with
  and without resource information in the ``configurations``.

Work Items
----------

These work items are designed so Neutron end-to-end behavior can be
prototyped and tested independently of the progress of related work in
Nova. But part of it depends on already available Placement features.

* Extend agent heartbeat configuration with resource provider information.
* (We already have it): Persist extended agent configuration reported via
  heartbeat.
* (We already have it): Placement client in neutron-lib for the use of
  neutron-server.
* Neutron-server initially diffs resource info reported by agent against
  Placement.
* Neutron-server diffs consequent agent heartbeat configurations.
* Neutron-server turns the diffs into Placement requests (with generation
  handling).
* Extend rule ``qos_minimum_bandwidth_rule`` with direction ``ingress``.
* Extend port with ``resource_request`` based on QoS rule
  mimimum-bandwidth-placement.
* Make the reported agent configuration queriable so neutron-server can
  infer which backend is implied in the RP allocated (as in
  ``binding_profile.allocation``).
* In binding a port with ``binding_profile.allocation`` replace the list of
  tried mechanism drivers with the one-element list of the inferred backend.
* (We already have it): Send ``binding_profile`` to all agents.
* In sriov-agent force the choice of PF as implied by the RP allocated.

For each of the above:

* Tests.
* Documentation: api-ref, devref, networking guide.
* Release notes.

References
==========

.. [1] `Placement API`:
       https://docs.openstack.org/nova/latest/user/placement.html

.. [2] `SPEC: Network bandwidth resource provider`:
       https://review.openstack.org/502306

.. [3] `Add a spec for strict minimum bandwidth support`:
       https://review.openstack.org/396297

.. [4] `[RFE] Minimum bandwidth support (egress)`:
       https://bugs.launchpad.net/neutron/+bug/1560963

.. [5] `BP: os-vif migration tasks`:
       https://blueprints.launchpad.net/neutron/+spec/os-vif-migration

.. [6] `Newton Release Notes`:
       https://docs.openstack.org/releasenotes/neutron/newton.html

.. [7] `Networking Guide: SR-IOV`:
       https://docs.openstack.org/neutron/queens/admin/config-sriov.html#enable-neutron-sriov-agent-compute

.. [8] `Granular Resource Request Syntax`:
       https://specs.openstack.org/openstack/nova-specs/specs/rocky/approved/granular-resource-requests.html

.. [9] `any-traits-in-allocation_candidates-query`:
       https://blueprints.launchpad.net/nova/+spec/any-traits-in-allocation-candidates-query

.. [10] `QoS devref`:
        https://docs.openstack.org/neutron/latest/contributor/internals/quality_of_service.html#agent-backends

.. [11] `SPEC: Prep work for Network aware scheduling (Pike)`:
        https://specs.openstack.org/openstack/nova-specs/specs/pike/approved/prep-for-network-aware-scheduling-pike.html
