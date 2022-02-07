..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
Off-path SmartNIC DPU Port Binding with OVN
===========================================

https://blueprints.launchpad.net/neutron/+spec/off-path-smartnic-dpu-port-binding-with-ovn

Off-path SmartNIC DPUs introduce an architecture change where network
agents responsible for NIC switch configuration and representor
interface plugging run on a separate SoC with its own CPU, memory and
that runs a separate OS kernel. The side-effect of that is that
hypervisor hostnames no longer match SmartNIC DPU hostnames which are
seen by ovs-vswitchd and OVN [3]_ agents while the existing port binding
code relies on that. The goal of this specification is to introduce
changes necessary to extend the existing hardware offload code to cope
with the hostname mismatch and related design challenges while reusing
the rest of the code. To do that, PCI(e) add-in card tracking is
introduced for boards with unique serial numbers so that it can be used
to determine the correct hostname of a SmartNIC DPU which is responsible
for a particular VF. Additionally, more information is suggested to be
passed in the "binding:profile" during a port update to facilitate
representor port plugging.

Nova spec: https://review.opendev.org/c/openstack/nova-specs/+/787458


Problem Description
===================

Terminology
-----------

* Data Processing Unit (DPU) - an embedded system that includes a CPU, a NIC
  and possibly other components on its board which integrates with the main
  board using some I/O interconnect (e.g. PCIe);
* Off-path SmartNIC DPU architecture [1]_ [2]_ - an architecture where NIC
  cores are responsible for programming a NIC Switch and are bypassed when
  rules programmed into the NIC Switch are enough to make a decision on where
  to deliver packets. Normally, NIC cores only participate in packet forwarding
  for the "slow path" only and the "fast path" is handled in hardware like an
  ASIC;
* On-path SmartNIC DPU architecture [1]_ [2]_ - an architecture where NIC cores
  participate in processing of every packet going through the NIC as a whole.
  In other words, NIC cores are always on the "fast path" of all packets;
* NIC Switch (or eSwitch) - a programmable embedded switch present in various
  types of NICs (SR-IOV-capable NICs, off-path SmartNICs). Typically relies
  on ASICs for packet processing;
* switchdev [4]_ - in-kernel driver model for switch devices which offload the
  forwarding (data) plane from the kernel.
* Representor ports [5]_ - a concept introduced in the switchdev model which
  models netdevs representing switch ports. This applies to NIC switch ports
  (which can be physical uplink ports, PFs or VFs);
* devlink [6]_ - a kernel API to expose device information and resources not
  directly related to any device class, such as chip-wide/switch-ASIC-wide
  configuration;
* PCI/PCIe Vital Product Data (VPD) - a standard capability exposed by PCI(e)
  endpoints which, among other information, includes a unique serial number
  (read-only, persistent, factory-generated) of a card shared by all functions
  exposed by it. Present in PCI local bus 2.1+ and PCIe 4.0+ specifications.


Detailed overview
-----------------

The related Nova specification [7]_ is an integral part of the cross-project
effort and its problem description is assumed to be a part of this
specification.

There are also relevant changes to OVS [8]_ and OVN itself [9]_ [10]_ that
are needed in order to facilitate the implementation of this specification
in Neutron. The result of upstream OVN discussions has been that a separate
``ovn-vif`` [11]_ component is going to be introduced and the corresponding
change [10]_ is in the late stages of the review process at the time of writing.
Support in other ML2 mechanism drivers is possible, but we would like to target
OVN first and leave other ML2 drivers out of scope for Yoga cycle.

In the context of off-path SmartNIC DPUs, the main problems that need to be
addressed in this change are:

* SmartNIC DPU hostname selection such that the correct host is selected for
  representor port plugging. This corresponds to selecting the right chassis
  (transport node) in OVN terms;
* VF representor selection corresponding the VF and a PF it is associated with
  selected at the hypervisor host side by Nova.

Some practical challenges are as follows:

* If PCIe is used to access the NIC at a SmartNIC DPU host, PCI addresses seen
  by the hypervisor host and the SmartNIC DPU host differ as they have their
  own root complexes and see isolated PCIe topologies (while communicating
  with the same NIC). Therefore, PCI addresses seen from the SmartNIC DPU host
  side have no meaning at the hypervisor host side;
* A NIC Switch is not exposed to PFs at the hypervisor host side. Meanwhile,
  PF representor numbers are tied to a particular NIC Switch (to a controller
  number) at the SmartNIC DPU host side. There may be multiple controllers per
  SmartNIC DPU host in a general case so simply passing PF logical numbers from
  the host side is not enough to identify a PF representor at the transport
  node side;
* VF logical numbering seen via devlink port attributes depends on a device
  driver implementation at the hypervisor host and does not always match what
  is seen at the SmartNIC DPU host via the NIC Switch. It is possible to
  retrieve VF logical numbers tied to PFs at the hypervisor side based on the
  VF PCI address assignment process governed by the PCI SIG SR-IOV
  specification;
* During port binding, the OVN mechanism driver needs to handle the vnic type
  VNIC_REMOTE_MANAGED which it currently does not.

The following diagram illustrates various components that play a role in the
context of this specification::

                           ┌────────────────────────────────────┐
                           │  Hypervisor                        │    LoM Ports
                           │  ┌───────────┐       ┌───────────┐ │   (on-board,
                           │  │ Instance  │       │  Nova     │ ├──┐ optional)
                           │  │ (QEMU)    │       │ Compute   │ │  ├─────────┐
                           │  │           │       │           │ ├──┘         │
                           │  └───────────┘       └───────────┘ │            │
                           │                                    │            │
                           └────────────────┬─┬───────┬─┬──┬────┘            │
                                            │ │       │ │  │                 │
                                            │ │       │ │  │ Control Traffic │
                               Instance VF  │ │       │ │  │ PF associated   │
                                            │ │       │ │  │ with an uplink  │
                                            │ │       │ │  │ port or a VF.   │
                                            │ │       │ │  │ (used to replace│
                                            │ │       │ │  │  LoM)           │
       ┌────────────────────────────────────┼─┼───────┼─┼──┼─┐               │
       │   SmartNIC DPU Board               │ │       │ │  │ │               │
       │   (transport node)                 │ │       │ │  │ │               │
       │  ┌──────────────┐ Control traffic  │ │       │ │  │ │               │
       │  │   App. CPU   │ via PFs or VFs  ┌┴─┴───────┴─┴┐ │ │               │
       │  ├──────────────┤  (DC Fabric)    │             │ │ │               │
       │  │ovn-controller├─────────────────┼─┐           │ │ │               │
       │  ├──────────────┤                 │ │           │ │ │               │
       │  │ovs-vswitchd  │     Port        │ │NIC Switch │ │ │               │
       │  ├──────────────┤   Representors  │ │  ASIC     │ │ │               │
       │  │    br-int    ├─────────────────┤ │           │ │ │               │
       │  │              ├─────────────────┤ │           │ │ │               │
       │  └──────────────┘                 │ │           │ │ │               │
       │                                   │ │           │ │ │               │
       │                                   └─┼───┬─┬─────┘ │ │               │
     ┌─┴──────┐Initial NIC Switch            │   │ │       │ │               │
    ─┤OOB Port│configuration is done via     │   │ │uplink │ │               │
     └─┬──────┘the OOB port to create        │   │ │       │ │               │
       │       ports for control traffic.    │   │ │       │ │               │
       └─────────────────────────────────────┼───┼─┼───────┼─┘               │
                                             │   │ │       │                 │
                                          ┌──┼───┴─┴───────┼────────┐        │
                                          │  │             │        │        │
                                          │  │   DC Fabric ├────────┼────────┘
                                          │  │             │        │
                                          └──┼─────────────┼────────┘
                                             │             │
                                             │         ┌───┴──────┐
                                             │         │          │
                                         ┌───▼──┐  ┌───▼───┐ ┌────▼────┐
                                         │OVN SB│  │Neutron│ │Placement│
                                         └──────┘  │Server │ │         │
                                                   └───────┘ └─────────┘

Proposed Change
===============

Identifying a SmartNIC DPU Host and VF Representor
--------------------------------------------------

The related Nova specification [7]_ introduces add-in-card (board) serial
number collection via PCIe VPD and forwarding of that information in port
updates to Neutron, therefore, it becomes possible to match that against what
is seen at the SmartNIC DPU host side. Those serial numbers are unique,
read-only, and assigned at the manufacturing time (per the PCI and PCIe specs).
The board serial number can be stored in the OVN SB database as an external-id
for a particular OVN chassis::

  external_ids:ovn-board-serial-number=UNIQUEBOARDSERIAL

Note that there can be other means of accessing the NIC from the SmartNIC DPU
host side (e.g. platform devices or other I/O types) so querying the serial
number can be done either by extracting PCIe VPD via sysfs or by using
devlink-info API and getting ``board.serial_number`` (which does not depend
on a particular I/O interconnect type). However, this is a concern for OVN
itself since Neutron does not have any agents running at the SmartNIC DPU side
- it only needs to look up a chassis by a ``ovn-board-serial-number``
regardless of how it got collected at the SmartNIC DPU host.

The responsibility of placing it there can be given to:

* A deployer who will be responsible of configuring the ovn-controller agent
  to be responsible for a particular transport node;
* ovn-controller itself based on some default behavior of looking up a card
  serial number based on the switchdev-capable devices available on the
  SmartNIC DPU host.

Regardless of the chosen method of populating this value, Neutron will then
be able to lookup which OVN chassis should handle representor plugging and flow
programming for a particular port update that has a board serial included.

VF logical numbers are relevant in the context of a particular PF. In turn,
PF logical numbers are tied to a particular controller. Since a NIC Switch is
not exposed to the hypervisor host, it cannot determine the controller logical
number as visible by the SmartNIC DPU host and while PF numbers could be
inferred indirectly from their PCI address function numbers, this information
is not enough to identify a PF representor at the SmartNIC DPU host side.
To address that, a PF MAC seen by the hypervisor host can be passed to the
SmartNIC DPU host. While port representors have a different MAC address from
the port they represent, it is possible to use generic in-kernel API to
retrieve a MAC address of a function via its representer (as of kernel
5.9 [12]_, subject to the switchdev-aware device driver support, for example
[13]_). While Neutron will not be responsible for doing this lookup (OVN will),
it needs to accept and forward this information to OVN.

As a result, the ``binding:profile`` attribute for a port updated by Nova is
expected to contain the following information::

  binding:profile={
      "pci_vendor_info",
      "pci_slot",
      "physical_network",
      "card_serial_number": "UNIQUEBOARDSERIAL",
      "pf_mac_address": "de:ad:be:ef:ca:fe",
      "vf_num": 42
  }

With this information both the right OVN chassis hostname and port
representor at the SmartNIC DPU host side can be identified.

The OVN mechanism driver also needs to be changed (where relevant) to use the
hostname looked up based on a serial number instead of relying on the
hypervisor hostname passed in ``binding_host_id``.

After receiving a port update from Nova, Neutron needs to create relevant
Logical Switch Ports in the OVN database (the final approach to communicating
the necessary information to OVN is to be determined based on the outcome of
``ovn-vif`` discussions in upstream OVN)::

  Logical_Switch_Port
    options:requested-chassis=fqdn-of-a-smartnic-dpu-host
    options:plug-type=representor
    options:plug-mtu-request=1500
    options:plug:representor:pf-mac=de:ad:be:ef:ca:fe
    options:plug:representor:vf-num=0

If ``vf-num`` is not specified, then a PF representor should be plugged instead
of a VF representor.


Port Binding and Port Capabilities
----------------------------------

Currently, to support hardware offload, the ``bind_port`` method of the OVN
mechanism driver is able to bind ports of type ``direct`` if they also have
the "switchdev" capability supplied in their ``binding:profile`` attribute::

  binding:profile='{"capabilities": ["switchdev"]}'

This capability is discovered by Nova from PCI(e) endpoints via devlink -
when an NIC Switch is visible via a PF, the "switchdev" capability is added to
both PFs and VFs tracked in Nova. SmartNIC DPUs do not expose the NIC Switch to
the hypervisor host, therefore, this capability is not discovered. To address
that, the related Nova specification [7]_ relies on a new port type for the
purposes of working with SmartNIC DPUs called ``VNIC_REMOTE_MANAGED``. The
``VNIC_SMARTNIC`` VNIC type already present in neutron-lib has been considered
but it was found later that the fact that scheduling happens after resource
request creation makes it to run into a conflict with the Ironic use-case).

The OVN mechanism driver needs to be extended to also handle
``VNIC_REMOTE_MANAGED`` ports. It will allow the port binding code to pick the
right code-path and trigger the representor port plugging logic in OVN.

Implementation
==============

Primary Assignees
-----------------

* Dmitrii Shcherbakov (lp: ~dmitriis, oftc || libera: dmitriis)
* Frode Nordahl (lp: ~fnordahl, oftc || libera: fnordahl)

Dependencies
============

* OVS [8]_ and OVN [9]_ [10]_ [11]_ changes.
* A related Nova specification [7]_ depends on this spec;

References
==========

.. [1] https://netdevconf.info/0x14/pub/slides/39/Netdev%200x14%20--%20Taking%20Control%20of%20your%20SmartNIC%20v1.pdf
.. [2] https://homes.cs.washington.edu/~arvind/papers/ipipe.pdf
.. [3] https://man7.org/linux/man-pages/man7/ovn-architecture.7.html
.. [4] https://www.kernel.org/doc/Documentation/networking/switchdev.txt
.. [5] https://lwn.net/Articles/692942/
.. [6] https://www.kernel.org/doc/html/latest/networking/devlink/index.html
.. [7] https://review.opendev.org/c/openstack/nova-specs/+/787458
.. [8] https://github.com/openvswitch/ovs/commit/bfee9f6c011518c7690d3ce3b290a2b7189a377d
.. [9] https://patchwork.ozlabs.org/project/ovn/list/?series=267834&state=3&archive=both
.. [10] https://patchwork.ozlabs.org/project/ovn/list/?series=269965&state=*&archive=both
.. [11] https://github.com/fnordahl/ovn-vif
.. [12] https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2a916ecc405686c1d86f632281bc06aa75ebae4e
.. [13] https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f099fde16db3d2594a54ba8c94ce9fa3557aa3e1
