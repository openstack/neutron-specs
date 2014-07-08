..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================
Support for VDP in Neutron for Network Overlays
===============================================

https://blueprints.launchpad.net/neutron/+spec/vdp-network-overlay


Problem description
===================

A very common requirement in today's data centers is the need to support
more than 4K segments. Openstack achieves this by using Host based
Tunneling and uses tunneling protocols like GRE, VXLAN etc. A topology,
more common in Massively Scalable Data Centers (MSDC's) is when the compute
nodes are connected to external switches. We will refer to all the external
switches and the other inter-connected switches as a 'Fabric'. This is shown
below:

asciiflows::

                               XXXXXXXXXXXXX
                        XXXXXXX             XXXXXXXXX
                       X +–––––––+         +––––––––+ X
                      X  |spine 1| ++  ++  |spine n |  X
                    X    +–––––––+         +––––––––+   X
                   X                                     X
                  X                                       X
                 +––––––––+                               X
                 + Leaf i |     SWITCH FABRIC            X
                 +––––––––+                              X
                    X                                   X
                     X                                 X
                       X+–––––––+            +–––––––+X
                        | Leaf x|XXXX XX XXXX|Leaf z |
                        |  +----+            |   +---+
                        |  | VDP|            |   |VDP|
                        +––+–––++            +––++---+
       +––––––––––––––+        |                |
       |   Openstack  |        |                |
       |   Controller |        |                |
       |   Node       | +––––––+––––––––+    +––+––––––––––––+
       |              | | OVS           |    | OVS           |
       |              | +–––+-–-––––––––+    +–––+–-–––––––––+
       |              | |   |LLDPad(VDP)|    |   |LLDPad(VDP)|
       |              | |   +---––––––––+ ++ |   +--–––––––––+
       |              | | Openstack     |    | Openstack     |
       +––+––––+––––+–+ | Compute node 1|    | Compute node n|
               |        +––––––––––+––––+    +––––––––––+––––+
               |                   |                    |
               +–––––––––––––––––––+––––––––––––––––––––+


In such topologies, the fabric can support more than 4K segments.
Tunneling starts at the Fabric or more precisely the leaf switch
connected to the compute nodes. This is called the Network based Overlay.
The compute nodes send a regular dot1q frame to the fabric. The
fabric does the appropriate encapsulation of the frame and associates
the port, VLAN to a segment (> 4K) that is known in the fabric. This
fabric segment can be anything based on the overlay technology used
at the fabric. For VXLAN, it can be VNI. For TRILL, it can be the FGL
value and for FabricPath it can be the internal qinq tag. So,
the VLAN tag used by the compute nodes can only be of local significance,
that is, it's understood only between the compute nodes and the
immediately connected switch of the fabric. The immediate question is
how the fabric knows to associate a particular VLAN from a VM to a
segment. An obvious way is to configure this statically in the switches
beforehand. But, a more useful and interesting way is to do this
automatically. The fabric using 'some' method knows the dot1q tag that
will be used by the launched instance and assigns a segment and
configures itself.
There are many solutions floating around to achieve this.
The method that is of interest in this document is by using VSI
Discovery Protocol (VDP), which is a part of IEEE 802.1QBG standard [1].
The purpose of VDP is to automatically signal to the network when a vNIC is
created, destroyed or modified. The network can then appropriately provision
itself, which alleviates the need for manual provisioning. The basic flow
involving VDP is given below. Please refer the specification link of the
blueprint for the diagram.

1. In the first step, the network admin creates a network. (non-VDP
specific)

2. In the second step, the server admin launches a VM, assigning it's
vNIC to the appropriate network that was already created by the network
admin.  (non-VDP specific).

3. When a VM is launched in a server, the VDP protocol running in the
server signals the UP event of a vNIC to the connected switch passing the
parameters of the VM. The parameters of interest are the information
associated with a vNIC (like the MAC address, UUID etc) and the network
ID associated with the vNIC.

4. The switch can then provision itself. The network ID is the common
entity here. The switch can either contact the central database for the
information associated with the Network ID or these information can be
configured statically in the switch. (again, non-VDP specific).

In Openstack, the first two steps are done at the Controller. The third
step is done as follows:
The Network ID (step 3 above) that we will be using will be the Segment ID
that was returned when the Network was created. In Openstack, when VLAN
type driver is used, it does the VLAN translation. The internal VLAN is
translated to an external VLAN for frames going out and the reverse is
done for the frames coming in. What we need is a similar mechanism, but
the twist here is that the external VLAN is not pre-configured, but is
given by the VDP protocol. When a VM is launched, its information will
be sent to the VDP protocol running in the compute. Each compute will
be running a VDP daemon. The VDP protocol will signal the information
to the fabric (Leaf switch) and the switch will return the VLAN to be
used by the VM. Any frames from the VM then needs to be tagged
with the VLAN sent by VDP so that the fabric can associate it with the
right fabric segment.

To summarize the requirements:

1. Creation of more than 4K networks in Openstack. With the current model,
one cannot use a type driver of VLAN because of the 4K limitation. With
today's model, we need to use either GRE or VXLAN.

2. If a type driver of VXLAN or GRE is used that may mean host based
overlay. Frames will be tunneled at the server and not at the network.

3. Also the programming of flows should be done after communicating the
vNIC information to lldpad. lldpad communicates with the leaf switches and
return the VLAN to be used by this VM as described earlier.

Proposed change
===============

The following changes can be considered as a solution to support Network
based overlays using VDP. External components can still communicate with
LLDPad.

1.  Create a new type driver for network-based overlays. This will be very
similar to the existing VLAN type driver but without the 4K range check. The
range can be made configurable.
This type driver will be called "network_overlay" type driver.

2. In the computes (ovs_neutron_agent.py), the following is needed:

  2.a. An external bridge (br-ethx) is also needed for this model, so no
       change required in the init. 'enable_tunneling' will be set to false.
  2.b. A configuration parameter is required that specifies whether the
       network overlay uses VDP based mechanism to get the VLAN.
  2.c. Another condition needs to be added in the places where
       provisioning/reclaiming of the local VLAN and adding/deleting the
       flows is done. Change will be to communicate with VDP and program
       the flow using the VLAN returned by VDP.

This is sample code change:

def port_bound(...)

...

        if net_uuid not in self.local_vlan_map:
           self.provision_local_vlan(net_uuid, network_type,
                                     physical_network, segmentation_id)
        else:
            if network_type == constants.TYPE_NETWORK_OVERLAY and
               self.vdp_enabled():

               self.send_vdp_assoc(...)

def provision_local_vlan(...)

...

        elif network_type == constants.TYPE_FLAT:

...

        elif network_type == constants.TYPE_VLAN:

...

        elif network_type == constants.TYPE_NETWORK_OVERLAY:
            if self.vdp_enabled():
               self.send_vdp_assoc(....)
               br.add_flow(...) Using VLAN return from prev step

            else:

              ...


def reclaim_local_vlan(self, net_uuid):

...

        elif network_type == constants.TYPE_FLAT:

...

        elif network_type == constants.TYPE_VLAN:

...

        elif network_type == constants.TYPE_NETWORK_OVERLAY:
            if self.vdp_enabled():
                self.send_vdp_disassoc(...)
                br.delete_flows(...)

            else:

                ...

3. Save the assigned VDP vlan and other useful information (such as
segmentation_id, compute node name and port id) into the ML2 database for
troubleshooting and debugging purposes. A new RPC method from OVS neutron
agent to neutron server will be added.

Alternatives
------------

VDP Protocol runs between the server hosting the VM's (also running Openstack
agent) and the connected switch. Mechanism driver support is only present at
the Neutron Server. Current type drivers of VXLAN, GRE use host-based overlays.
Type driver of VLAN has the 4K limitation. So a new type driver is required.
Refer [4] for more detailed information.
Duplicating the OVS Neutron agent code for VDP alone is also an alternative.
But, this makes it tough to manage and was also not recommended.
This approach mentioned in this spec and explained in detail in [4]
require the minimal changes with the existing infra structure and that will
also serve the needs without impacting other areas.

Data model impact
-----------------

New database for network overlay type driver will be created. It contains,
segment-id, physical network and allocated flag.

REST API impact
---------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

Adding configuration parameters in config file for:

1. network_overlay type driver - This will have a parameter that shows if VDP
is used.

2. VDP - This has all the VDP related configuration (such as vsiidtype) [1][2]


Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Nader Lahouti (nlahouti)

Other contributors:
  Paddu Krishnan (padkrish)

Work Items
----------

* Add a new type driver for network-based overlays. This will be very
  similar to the existing VLAN type driver but without the 4K range check. The
  range can be made configurable.

* In the computes (ovs_neutron_agent.py), add functionality for network
  overlays using VDP.

Dependencies
============

VDP running as a part of lldpad daemon [2].

Testing
=======


For testing, it is needed to have the setup as shown in the begining of this
page. It is not mandatory to have physical switches for that topology.
The whole setup can be deployed using virtual switches (i.e. instead of
having physical switch fabric, it can be replaced by virtual switches).

The regular test for type driver applies here. Integration with VDP is
required for programming the flows.

The implementation of VDP is needed in the switches (physical or virtual).


Documentation Impact
====================

Changes in the OVS neutron agent and configuration details.

References
==========

[1] [802.1QBG-D2.2] IEEE 802.1Qbg/D2.2

[2] http://www.open-lldp.org

[3] https://blueprints.launchpad.net/neutron/+spec/vdp-network-overlay

[4] https://docs.google.com/document/d/1jZ63WU2LJArFuQjpcp54NPgugVSaWNQgy-KV8AQEOoo/edit?pli=1
