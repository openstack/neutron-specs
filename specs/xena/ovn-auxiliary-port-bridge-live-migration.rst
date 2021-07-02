==========================================================================
OVS auxiliary bridge to reduce live migration networking disruption in OVN
==========================================================================

Launchpad bug: https://bugs.launchpad.net/neutron/+bug/1933517

The goal of this spec is to propose a change in the VM - backend (OVN)
connectivity to improve the live migration process.


Problem Description
===================

The live migration is a very sensitive process that implies to configure
a destination host to execute a virtual machine that is already running.
When the destination hypervisor is prepared, the source virtual machine
is paused and immediately unpaused in the destination host.

Since [1]_ and [2]_, Nova is capable of plugging the port and create the
interface in the destination host during the pre-live migration. But the TAP
device is created by libvirt when the VM is unpaused. This is when the port
interface is assigned with an OpenFlow port ID.

When the port is created, the network backend detects the new port attached
and informs to the OVN Northbound and this event is received by Neutron. The
network backend is commanded to configure the needed rules for this new port.

The problem with this sequence of actions is that when the virtual
machine is unpaused, the network backend is not ready to continue any
network communication until the OVN controller (in the compute node) has set
all needed OpenFlow rules and chassis configuration, using the OpenFlow port
ID (``interface.ofport``) assigned when the TAP interface is created.


Proposed Change
===============

The spec scope is limited to the OVN backend. This spec does not consider the
case of HW offloaded ports, that have other plug process. This spec is only
considering ``os_vif.objects.vif.VIFOpenVSwitch`` VIF types within the related
network backend.

This spec proposes to create an intermediate OVS bridge between the integration
bridge and the virtual machine tap interface. This OVS bridge will be connected
to the integration bridge with a patch port. The TAP interface will be plugged
into the intermediate bridge when the virtual machine is created or unpaused
(that happens during the live migration). This architecture is very similar to
OVS hybrid plugging, but with an OVS bridge in between.

The advantage of this approach is that the integration bridge port, in this
case the patch port, that is created during the pre-live migration process, has
a valid OpenFlow port ID (``interface.ofport``), needed to provide the correct
OpenFlow rules in the integration bridge.

After the port is plugged, Nova commands to libvirt to copy the guest memory
to the destination host. If "live_migration_permit_post_copy" is used [3]_,
the virtual machine on the destination host will be activated before all its
memory has been copied. However there is a period of time where the port
is bound to the destination host and the virtual machine is still running on
the source host. During this period of time, the virtual machine won't be
able to communicate, same as without this feature.

When the virtual machine is unpaused in the destination host, the OVN
backend is ready to immediately continue transmitting packets from the guess.


Implementation
--------------

The VIF plug and unplug process is executed by Nova, using the different
backend implementations provided by ``os-vif`` library. The bridge creation
and deletion will be done during the plug and unplug processes [4]_, as in OVS
hybrid plug strategy.

The proposed architecture and naming is the following one, implemented in
[5]_:

::

        ┌────────┐
        │tap-xxx │           TAP interface port
  ┌─────┴────────┴─────┐
  │      pbr-xxx       │     Port bridge
  └─────┬────────┬─────┘
        │pbp-xxx │           Port bridge patch port (port bridge side)
        └────┬───┘
        ┌────┴───┐
        │ipb-xxx │           Port bridge patch port (integration bridge side)
  ┌─────┴────────┴─────┐
  │    integration     │
  │      bridge        │
  └────────────────────┘


For each new VM TAP interface, an OVS bridge is created along with the patch
port to connect this bridge with the integration bridge. This port bridge will
have a default OpenFlow rule, allowing all traffic from the TAP interface port
to the patch port:

.. code-block:: console

   cookie=0x0, duration=84162.020s, table=0, n_packets=444223, n_bytes=832399534, priority=0 actions=NORMAL


OVN links the OVS ports (Open vSwitch DB) with the Logical Switch Ports (OVN
NB database) using the Neutron DB port ID. The OVN NB Logical Switch Port,
created by Neutron, will have this port ID in the ``lsp.name`` field. The OVS
port ``ipb-xxx`` (the integration bridge side patch port) is created with
``interface.external_ids={iface-id: port_id}``. The ovn-controller will detect
this new port and assign to OVN SB Port Binding the corresponding chassis.
The ``interface.external_ids`` information is added by ``os-vif`` during the
port plug process [5]_, called by Nova.

Note: OVS and OVN ports are represented in ``os-vif`` with the same VIF type,
``objects.vif.VIFOpenVSwitch``. The ``os-vif`` implementation could be used
both for OVS and OVN backends. However, the scope of this spec is limited to
OVN backend only.


Nova - Neutron events
---------------------

Nova and Neutron have a mechanism to communicate the state of the VIFs [6]_.
This is a unidirectional API that allows Neutron to inform Nova when a VIF has
been plugged or unplugged. Nova expects from Neutron a ``network-vif-plugged``
event when the port has been bound or plugged. Depending on the backend, Nova
will expect a ``network-vif-plugged`` when the port has been bound or when
the port has been plugged [7]_. For example, in OVS with hybrid plug, Nova
expects the event when the port has been plugged to the OVS. In OVN, Nova is
expecting this event when the port has been bound during the live migration
process [8]_.

Once Nova creates the intermediate bridge [1]_ [2]_, Neutron can send the
``network-vif-plugged`` event when the port is plugged and the network backend
is configured.

This spec proposes to use the existing OVN event infrastructure to capture the
patch port creation event, using ``LogicalSwitchPortCreateUpEvent``, and in
the handler method push the event to Nova. Along with this change, Neutron will
inform Nova about the backend used in the port adding a field in
``port.vif_details`` called ``backend``. The value will be "ovn". This
port update will be done in [9]_, in the mech driver port binding method.


Neutron configuration
---------------------

As explained in the previous section, this spec proposes to change the event
emission from Neutron. That will depend on if ``os-vif`` is configured to
create this intermediate bridge between the integration bridge and the TAP
device.

Because Neutron cannot read the ``os-vif`` configuration, this spec proposes
to add the same config flag in the ML2 OVN plugin section: "per_port_bridge".

If enabled, Neutron will send the ``network-vif-plugged`` when the port is
plugged, not when the port binding is updated.


QoS
---

With this feature, the Logical Switch Port is now the patch port between the
port bridge and the TAP device, not the TAP device port. In OVS, QoS rules
can only be applied to external ports. However, in OVN the QoS rules are
applied using the "match" field in the QoS register [10]_. This is a string
in the same expression language used in the Logical_Flow table. The traffic
shaping applied to the TAP device is the same when using the patch port
reference because the traffic flow is the same in both ports.


Performance
-----------

The intermediate bridge will have one single rule, with action NORMAL. That
means all traffic coming from the TAP port will be sent to the patch port and
vice versa.

The datapath will collapse this bridge resulting in an identical flow as
without the bridge. In other words, the new intermediate bridge won’t have any
effect on packet processing performance.


OVN DPDK
--------

Same as with other bridges (physical bridges, tunnel bridges, external
bridges), each port bridge will be connected to the integration bridge with a
patch port. That keeps one single datapath (“netdev” in DPDK case), as
commented in the previous section.


Upgrade Impact
--------------

This feature could be enabled in ``os-vif`` using the configuration variable
"per_port_bridge" [11]_. If this option is enabled, any OVN port will be
plugged and unplugged using the implementation described in this spec.
Currently this implementation does not handle mixing both plugging methods
(connecting the TAP port directly to the integration bridge or creating the
port bridge as described in this spec). If this option is enabled, the compute
node should not host any virtual machine. At this point, as described in the
proposed change, any virtual machine created or migrated to this host, will
be plugged using the intermediate port bridge. In case of migration, that will
solve or mitigate the connectivity problems when the virtual machine is
unpaused in the destination host.

This spec does not cover any scenario mixing TAP ports using both plug
strategies on the same host (same as with OVS hybrid plugging strategy).


Testing
=======

Tempest Tests
-------------

Live migration tempest tests are covered in the Nova CI. In order to test this
feature, a set of tests from ``tempest.api.compute.admin.test_live_migration``
should be executed on ``neutron-ovn-tempest-slow`` CI job.


Documentation Impact
====================

User Documentation
------------------

Document the new architecture and the configuration flag to change the event
emission from Neutron.


Implementation
==============

Assignee(s)
-----------

Sean K Mooney (smooney@redhat.com, IRC: sean-k-mooney)
Rodolfo Alonso Hernandez (ralonsoh@redhat.com, IRC: ralonsoh)


References
==========
.. [1] https://review.opendev.org/c/openstack/nova/+/602432
.. [2] https://review.opendev.org/c/openstack/nova/+/797428
.. [3] https://docs.openstack.org/nova/pike/admin/live-migration-usage.html
.. [4] https://github.com/openstack/os-vif/blob/e93736711920afe64a850f564dddefbd704975cd/vif_plug_ovs/ovs.py
.. [5] https://review.opendev.org/c/openstack/os-vif/+/798055
.. [6] https://wiki.openstack.org/wiki/Nova/ExternalEventAPI
.. [7] https://github.com/openstack/nova/blob/e7a7fd51d12d045ff134d55a4a1749c1feee0386/nova/network/model.py#L464-L481
.. [8] https://github.com/openstack/neutron/blob/13e96bf6bd740a09283d9e2849fc2b6adbbf29e3/neutron/plugins/ml2/drivers/ovn/mech_driver/mech_driver.py#L796-L815
.. [9] https://github.com/openstack/neutron/blob/2acb96c37468120ba79de7d8de48ffb942e78a2b/neutron/plugins/ml2/drivers/ovn/mech_driver/mech_driver.py#L840-L960
.. [10] https://www.ovn.org/support/dist-docs/ovn-nb.5.html
.. [11] https://github.com/openstack/os-vif/blob/b837c1a74f37191692a820711e431a75516a4abf/vif_plug_ovs/ovs.py#L100
