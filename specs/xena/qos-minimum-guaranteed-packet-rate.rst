..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
QoS minimum guaranteed packet rate
==================================

https://bugs.launchpad.net/neutron/+bug/1922237

Similarly to how bandwidth can be a limiting factor of a network interface,
packet processing capacity tend to be a limiting factor of the soft switching
solutions like OVS. In the same time certain applications are dependent on not
just guaranteed bandwidth but also on guaranteed packet rate to function
properly. OpenStack already supports bandwidth guarantees via the
`minimum bandwidth QoS policy rules`_. This specification is aiming for adding
support for a similar minimum packet rate QoS policy rule.

To add support for the new QoS rule type both Neutron and Nova needs to be
extended. This specification only focuses on the Neutron impact. For the high
level view and for the Nova specific impact please read the
`Nova specification`_.


Problem Description
===================

Currently there are several parameters, quantitative and qualitative,
that define a VM and are used to select the correct host
and network backend devices to run it. The packet processing capacity of the
soft switching solution (like OVS) could also be such a parameter that affects
the placement of new workload.

This spec focuses on managing the packet processing capacity of the soft switch
(like OVS) running on the hypervisor host serving the VM. Managing packet
processing capacity in other parts of the networking backend (like in
Top-Of-Rack switches) are out of scope.

Guaranteeing packet processing capacity generally involves enforcement of
constraints on two levels.

* placement: Avoiding over-subscription when placing (scheduling) VMs and their
  ports.

* data plane: Enforcing the guarantee on the soft switch

This spec addresses placement enforcement only. Data plane enforcement
can be developed later. When the `packet rate limit policy rule`_ feature is
implemented then a basic data plane enforcement can be applied by adding both
minimum and maximum packet rate QoS rules to the same QoS policy where maximum
limit is set to be equal to the minimum guarantee.

.. _`packet rate limit policy rule`: https://bugs.launchpad.net/neutron/+bug/1912460

Use cases
---------
I as an administrator want to define the maximum packet rate, in kilo packet
per second (kpps), my OVS soft switch capable of handle per compute node, so
that I can avoid overload on OVS.

I as an end user want to define the minimum packet rate, in kilo packet per
second (kpps) a Neutron port needs to provide to my VM, so that my
application using the port can work as expected.

I as an administrator want to get a VM with such ports placed on a
compute node that can still provide the requested minimum packet rate for the
Neutron port so that the application will get what it requested.

I as an administrator want that the VM lifecycle operations are
rejected in case the requested minimum packet rate guarantee of the Neutron
ports of the server cannot be fulfilled on any otherwise eligible compute
nodes, so that the OVS overload is avoided and application guarantees are kept.


Proposed Change
===============

.. note::
   For the high level solution including the impact and interactions between
   Nova, Neutron and Placement see the `Nova specification`_.

This feature is similar to the already supported
`minimum bandwidth QoS policy rules`_. So this spec describes the impact in
relation to the already introduced concepts and implementation while also
pointing out key differences.

The solution needs to differentiate between two deployment scenarios.

1) The packet processing functionality is implemented on the compute host CPUs
   and therefore packets processed from both ingress and egress directions are
   handled by the same set of CPU cores. This is the case in the
   non-hardware-offloaded OVS deployments. In this scenario OVS represents a
   single packet processing resource pool. Which can be represented with a
   single new resource class, ``NET_PACKET_RATE_KILOPACKET_PER_SEC``.

2) The packet processing functionality is implemented in a specialized hardware
   where the incoming and outgoing packets are handled by independent
   hardware resources. This is the case for hardware-offloaded OVS. In this
   scenario a single OVS has two independent resource pools one for the
   incoming packets and one for the outgoing packets. Therefore these needs to
   be represented with two new resource classes
   ``NET_PACKET_RATE_EGR_KILOPACKET_PER_SEC`` and
   ``NET_PACKET_RATE_IGR_KILOPACKET_PER_SEC``.

OVS packet processing capacity
------------------------------
Neutron OVS agent needs to provide a configuration options for the
administrator to define the maximum packet processing capability of the OVS
per compute node. This means the following new configuration options are added
to OVS agent configuration::

    [ovs]
    # Comma-separated list of <hypervisor>:<packet_rate> tuples, defining the
    # minimum packet rate the OVS backend can guarantee in kilo (1000) packet
    # per second. The hypervisor name is used to locate the parent of the
    # resource provider tree. Only needs to be set in the rare case when the
    # hypervisor name is different from the DEFAULT.host config option value as
    # known by the nova-compute managing that hypervisor or if multiple
    # hypervisors are served by the same OVS backend.
    # The default is :0 which means no packet processing capacity is guaranteed
    # on the hypervisor named according to DEFAULT.host.
    # resource_provider_packet_processing_without_direction = :0
    #
    # Similar to the resource_provider_packet_processing_without_direction but
    # used in case the OVS backend has hardware offload capabilities. In this a
    # case the format is
    # <hypervisor>:<egress_packet_rate>:<ingress_packet_rate> which allows
    # defining packet processing capacity per traffic direction. The direction
    # is meant from the VM perspective. Note that the
    # resource_provider_packet_processing_without_direction and the
    # resource_provider_packet_processing_with_direction
    # are mutually exclusive options.
    # resource_provider_packet_processing_with_direction = :0:0
    #
    # Key:value pairs to specify defaults used while reporting packet rate
    # inventories. Possible keys with their types: allocation_ratio:float,
    # max_unit:int, min_unit:int, reserved:int, step_size:int
    # packet_processing_inventory_defaults = {
    #   'allocation_ratio': 1.0, 'min_unit': 1, 'step_size': 1, 'reserved': 0}


.. note::
   Note that while bandwidth inventory is defined per OVS bridge in the
   ``[ovs]resource_provider_bandwidths`` configuration option, the packet
   processing capacity is applied globally to the whole OVS instance.

.. note::
   Note that to support both OVS deployment scenarios there are two mutually
   exclusive configuration options. One to handle the normal OVS deployments
   with directionless resource inventories and one to handle hardware-offloaded
   OVS deployments with direction aware resource inventories.

The ``configurations`` field of the OVS agent heartbeat RPC message is extended
to report the packet processing capacity configuration to the Neutron server.
If the hypervisor name is omitted from the configuration it is resolved to the
value of ``[DEFAULT]host`` in the RPC message.

The Neutron server reports the packet processing capacity as the new
``NET_PACKET_RATE_KILOPACKET_PER_SEC`` or
``NET_PACKET_RATE_[E|I]GR_KILOPACKET_PER_SEC`` resource inventory on the OVS
agent resource provider (RP) to Placement in a similarly way how the bandwidth
resource is reported today. Now that the OVS agent RP has resource inventory
the Neutron server also needs to report the same ``CUSTOM_VNIC_TYPE_`` traits
on the OVS agent RP as reported on the bridge RPs. These are the vnic types
this agent configured to support. Note that ``CUSTOM_PHYSNET_`` traits are
not needed for the packet rate scheduling as this resource is not split
between the available physnets.

.. note::
    Regarding the alternative of reporting the resources on the OVS bridge
    level instead, please see the `Nova specification`_

Minimum packet rate QoS policy rule
-----------------------------------

A new Neutron API extension is needed to extend the QoS API with the new
minimum_packet_rate rule type. This rule has a similar structure and semantic
as the minimum_bandwidth rule::

    qos_apidef.SUB_RESOURCE_ATTRIBUTE_MAP = {
        'minimum_packet_rate_rules': {
            'parent': qos_apidef._PARENT,
            'parameters': {
                qos_apidef._QOS_RULE_COMMON_FIELDS,
                'min_kpps': {
                    'allow_post': True,
                    'allow_put': True,
                    'convert_to': converters.convert_to_int,
                    'is_visible': True,
                    'is_filter': True,
                    'is_sort_key': True,
                    'validate': {
                        'type:range': [0, db_const.DB_INTEGER_MAX_VALUE]}
                },
                'direction': {
                    'allow_post': True,
                    'allow_put': False,
                    'is_visible': True,
                    'validate': {
                        'type:values': (
                            constants.ANY_DIRECTION,
                            constants.INGRESS_DIRECTION,
                            constants.EGRESS_DIRECTION
                        )
                    }
                }
            }
        }
    }

The direction, ingress or egress of the rule is considered from the Nova
server's perspective. So an 1 kpps ingress guarantee means the system ensures
that at least 1000 packets can enter the VM via the given port per second.

To support the two different OVS deployment scenarios we need to allow that the
``direction`` field of the new QoS policy rule to be set to ``any`` to
indicate that this QoS rule is valid in the normal OVS case where the resource
accounting is directionless.

Mixing direction aware and directionless minimum packet rate rules in the same
QoS policy will always result in a NoValidHost error during scheduling as no
single OVS instance can have both direction aware and directionless inventory
at the same time. Therefore such rule creation will be rejected by the Neutron
server with HTTP 400.

.. note::
    For the alternative about having only direction aware QoS rule types see
    the `Nova specification`_

The above definition allows the ``min_kpps`` value to be updated with a ``PUT``
request. This request will be rejected with HTTP 501 if the rule is already
used in a bound port similarly to the minimum bandwidth rule behavior.

Neutron provides the necessary information to the admin clients via
the ``resource_request`` of the port to allow the client to keep the placement
resource allocation consistent and therefore implement the schedule time
guarantee. Nova will use this information to do so. However neither Neutron nor
Nova can enforce that another client, which can bound ports also properly
allocates resources for those ports. This is out of scope.

This results in the following new API resources and operations:

* ``GET /v2.0/qos/policies/{policy_id}/minimum_packet_rate_rules``

  List minimum packet rate rules for QoS policy

  Response::

    {
      "minimum_packet_rate_rules": [
          {
              "id": "5f126d84-551a-4dcf-bb01-0e9c0df0c793",
              "min_kpps": 1000,
              "direction": "egress"
          }
      ]
    }

* ``POST /v2.0/qos/policies/{policy_id}/minimum_packet_rate_rules``

  Create minimum packet rate rule

  Request::

    {
      "minimum_packet_rate_rule": {
          "min_kpps": 1000,
          "direction": "any",
      }
    }

  Response::

    {
      "minimum_packet_rate_rule": {
          "id": "5f126d84-551a-4dcf-bb01-0e9c0df0c793",
          "min_kpps": 1000,
          "direction": "any"
      }
    }

* ``GET /v2.0/qos/minimum_packet_rate_rules/{rule_id}``

  Show minimum packet rate rule details

  Response::

    {
      "minimum_packet_rate_limit_rule": {
          "id": "5f126d84-551a-4dcf-bb01-0e9c0df0c793",
          "min_kpps": 1000,
          "direction": "egress"
      }
    }

* ``PUT /v2.0/qos/minimum_packet_rate_rules/{rule_id}``

  Update minimum packet rate rule

  Request::

    {
      "minimum_packet_rate_rule": {
          "min_kpps": 2000
      }
    }

  Response::

    {
      "minimum_packet_rate_rule": {
          "id": "5f126d84-551a-4dcf-bb01-0e9c0df0c794",
          "min_kpps": 2000,
          "direction": "any",
      }
    }

* ``DELETE /v2.0/qos/packet_rate_limit_rules/{rule_id}``

  Delete minimum packet rate rule

.. note::
    This spec intentionally does not propose the addition of the old style
    ``/v2.0/qos/policies/{policy_id}/minimum_packet_rate_rules/{rule_id}`` APIs
    as Neutron team prefers the
    ``/v2.0/qos/alias_bandwidth_limit_rules/{rule_id}`` style APIs in the
    future. The old style APIs only kept for backwards compatibility for
    already existing QoS rules. However as this new API will not have an old
    style counterpart the 'alias' prefix is removed from the resource name.

To persist the new QoS rule type a new DB table
``qos_minimum_packet_rate_rules`` is needed::

        op.create_table(
            'qos_minimum_packet_rate_rules',
            sa.Column('id', sa.String(36), nullable=False,
                      index=True),
            sa.Column('qos_policy_id', sa.String(36),
                      nullable=False, index=True),
            sa.Column('min_kpps', sa.Integer()),
            sa.Column('direction', sa.Enum(constants.ANY_DIRECTION,
                                           constants.EGRESS_DIRECTION,
                                           constants.INGRESS_DIRECTION,
                                           name="directions"),
                      nullable=False,
                      server_default=None),
            sa.PrimaryKeyConstraint('id'),
            sa.ForeignKeyConstraint(['qos_policy_id'], ['qos_policies.id'],
                                    ondelete='CASCADE')
        )

This also means a new ``QosMinimumPacketRateRule`` DB model and OVO are added.

Request packet rate resources
-----------------------------

Today the ``resource_request`` field of the Neutron port is used to express the
resource needs of the port. The information in this field is calculated from
the QoS policy rules attached to the port. So far only the minimum bandwidth
rule is used as a source of the requested resources. However both the structure
and the actual logic calculating the value of this field needs to be changed
when the new minimum packet rate rule is added as an additional source of the
requested resources.

Currently ``resource_request`` is structured like::

    {
        "required": [<CUSTOM_PHYSNET_ traits>, <CUSTOM_VNIC_TYPE traits>],
        "resources":
        {
            <NET_BW_[E|I]GR_KILOBIT_PER_SEC resource class name>:
            <requested bandwidth amount from the QoS policy>
        }
    },

The current structure only allows describing one group of resources and traits.
However as described above the packet processing resource inventory is reported
on the OVS agent RP while the bandwidth resources are reported on the OVS
bridge RP. This also means that requesting these resources needs to be
separated as one group of resources always allocated from a single RP in
placement.

Therefore the following structure is proposed for the ``resource_request``
field::

    {
        "request_groups":
        [
            {
                "id": <some unique identifier string of the group>
                "required": [<CUSTOM_VNIC_TYPE traits>],
                "resources":
                {
                    NET_PACKET_RATE_[E|I]GR_KILOPACKET_PER_SEC:
                    <amount requested via the QoS policy>
                }
            },
            {
                "id": <some unique identifier string of the group>
                "required": [<CUSTOM_PHYSNET_ traits>,
                             <CUSTOM_VNIC_TYPE traits>],
                "resources":
                {
                    <NET_BW_[E|I]GR_KILOBIT_PER_SEC resource class name>:
                    <requested bandwidth amount from the QoS policy>
                }
            },
        ]
    }

Each dict in the list represents one group of resources and traits that needs
to be fulfilled from a single resource provider. E.g. either from the bridge
RP in case of bandwidth, or the OVS agent RP in case of packet rate. This
solves the problem of the RP separation. However in the other hand a port
still needs to allocate resources from the same overall entity, e.g. from
OVS, by allocating packet processing capacity from the OVS agent RP and
bandwidth from one of the OVS bridge RPs. In other words it is invalid to
fulfill the packet processing request from OVS and the bandwidth request
from an SRIOV PF managed by the SRIOV agent. In the placement model the OVS
bridge RPs are the children of the OVS agent RP while the PF RPs are not. So
placement already aware of the structural connections between the resource
inventories. Still by default, when two groups of resources are requested
Placement only enforces that they are fulfilled from two RPs in the same RP
tree but it does not enforce any subtree relationship between those RPs. To
express that the two groups of resources should be fulfilled from two RPs in
the same subtree (in our case from the subtree rooted by the OVS agent RP)
placement needs extra information in the request. Placement supports a
``same_subtree`` parameter that can express what we need. Neutron needs to add
a new top level key ``same_subtree`` to the ``resource_request``
dict. I.e.::

    {
        "request_groups":
        [
            {
                "id": "port-request-group-due-to-min-pps",
                # ...
            },
            {
                "id": "port-request-group-due-to-min-bw",
                # ...
            },
        ],
        "same_subtree":
        [
            "port-request-group-due-to-min-pps",
            "port-request-group-due-to-min-bw"
        ]
    }

The ``id`` field is a string that is assumed to be unique for each group of
each port in a neutron deployment. To achieve this a new UUID will be generated
for each group by combining the ``port_id`` and UUIDs of the QoS rules
contributing to the group via the UUID5 method. This way we get a unique and
deterministic id for the group so we don't need to persist the group id in
Neutron.

We discussed and rejected another two alternatives:

* Ignore the same subtree problem for now. `The QoS configuration`_ already
  requires the admin to create a setup where the different mechanism drivers
  support disjunct set of ``vnic_type`` s via the ``vnic_type_prohibit_list``
  config option. A port always requests a specific ``vnic_type`` and the
  supported ``vnic_type`` s are disjunct therefore the port's resource request
  always selects a specific networking backend via the ``CUSTOM_VNIC_TYPE_``
  traits. Support for packet rate resource is only added to the OVS backend,
  therefore if the scheduling of a port's resource request, containing both
  packet rate and bandwidth resource, succeeds then we know that the packet
  rate resource is fulfilled by the OVS agent RP. Therefore the ``vnic_type``
  matched the OVS backend. The bandwidth request also fulfilled from a backend
  with the same ``vnic_type`` so it is also fulfilled from the OVS backend.
  If we go with this direction then special care needs to be taken to document
  the above assumption carefully so that future developers can check if any of
  the statements become invalid due to new feature additions.

.. _`The QoS configuration`: https://docs.openstack.org/neutron/latest/admin/config-qos-min-bw.html#neutron-server-config

* Make an assumption in Nova that every request group from a single port
  always needs to be fulfilled from the same subtree. Then the ``same_subtree``
  key is not needed in the ``resource_request`` but Nova will implicitly assume
  that it is there and generate the placement request accordingly.

The selected alternative is more future proof. When the IP allocation pool
handling are transformed to use the resource request, that resource will come
from a sharing resource provider and therefore Nova cannot implicitly assume
same_subtree for the whole resource_request.

Note that the current neutron API does not define the exact structure of the
field, it is just a dict, so from Neutron perspective there is no need for a
new API extension to change the structure. However Nova needs to know which
format will be used by Neutron. We have alternatives:

* A new Neutron API extension: It can signal the change of the API. Nova
  already use to check the extension list to see if some feature is enabled in
  Neutron or not.

* A new top level ``resource_request_version`` field in the
  ``resource_request`` dict can signal the current and future versions. Nova
  would need to check if the field exists in ``resource_request`` and
  conditionally parse the rest of the dict.

* Let Nova guess based on the structure. The new top level ``request_groups``
  key can be used in Nova to detect the new format.

The API extension is the selected alternative as that feels like the standard
way in Neutron to signal API change.

Port binding
------------

Today Nova sends the UUID of the resource provider the port's
``resource_request`` is fulfilled from in the ``allocation`` key of the
``binding:profile``. The port binding logic uses this information to bind the
port to the same physical device or OVS bridge the port's resources are
allocated from. This is necessary as it is allowed to have multiple such
devices that are otherwise equivalent from Neutron perspective, i.e. they are
connected to the same physnet and supporting the same ``vnic_type``. When the
port has two groups of resource requests (one for bandwidth and the other for
packet rate) the resource allocation is fulfilled from more than one RPs. To
support that we need to change the structure of the ``allocation`` key. As
every group of resources in the ``resource_request`` now has a uniq identifier
Nova can send a mapping of <group.id>: <RP.uuid> back in the ``allocaton``
key of ``binding:profile`` so that Neutron gets informed about which RP
provided which group of resources. This means the following structure in the
``allocation`` key::

    {
        <uniq id of the group requesting min_pps>:
            <OVS agent RP UUID>,
        <uniq id of the group requesting min_bw>:
            <OVS bridge RP UUID or SRIOV PF RP UUID>,
    }

Only those group id keys are present in this dict that are present in the
``resource_request`` field as ``id``.

*Alternatively* we could  ignore the problem for now. Only OVS supports packet
rate inventory for now and the packet rate inventory is global for the whole
OVS instance. A single ``binding:host_id`` always maps to a single OVS
instance, so Neutron can always assume that the minimum packet rate resource
are allocated from the OVS agent resource provider that belongs to the
compute node the port is requested to bound to. So the UUID of the packet
rate resource provider is not needed of the port binding logic. Therefore the
``allocation`` key can be kept to only communicate the UUID of the bandwidth
resource provider if any.

QoS policy change on bound port
-------------------------------

Neutron supports changing the QoS policy on a bound port even if this means
that resource allocation change is needed due to changes in the resource
request indicated by the minimum_bandwidth QoS rule. This implementation needs
to be extended to handle changes not just in minimum_bandwidth but also in
minimum_packet_rate rule.

Upgrade
-------
A database schema upgrade is needed to add the new
``qos_minimum_packet_rate_rules`` table.

The changes in the Neutron server - OVS agent communication means that during
rolling upgrade upgraded OVS agents might send the new
packet processing capacity related keys in the hearthbeat while old agents
will not send it. So Neutron server needs to consider this new key as optional.

The manipulation of the  new minimum_packet_rate QoS policy rule and changes in
the ``resource_request`` and ``allocation`` fields of the port requires a new
API extension. We need to support upgrade scenarios where the Neutron is
already upgraded to Xena but Nova still on Wallaby version. As the old Nova
cannot support the new ``resource_request`` format, Neutron needs to make this
new API extension optional with a new configuration option in the neutron
server configuration. This configuration should be deprecated already at
introduction so that we can remove it during the Y release.

Testing
-------
* Unit tests.

* Functional tests: agent-server interactions.

* Tempest scenario tests: End-to-end feature test.

Documentation
-------------

* Update the generic `QoS admin guide`_
* A new admin guide, similar to the one for `minimum bandwidth`_

.. _QoS admin guide: https://docs.openstack.org/neutron/latest/admin/config-qos.html
.. _minimum bandwidth: https://docs.openstack.org/neutron/latest/admin/config-qos-min-bw.html

References
==========

* The `Nova specification`_

* The already supported `minimum bandwidth QoS policy rules`_ serving as a
  pattern for the new minimum packet rate QoS policy rule.


.. _`Nova specification`: https://review.opendev.org/c/openstack/nova-specs/+/785014
.. _`minimum bandwidth QoS policy rules`: https://docs.openstack.org/api-ref/network/v2/?expanded=#qos-minimum-bandwidth-rules
