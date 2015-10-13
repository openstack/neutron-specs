..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
VLAN aware VMs
==========================================

Launchpad blueprint:

 https://blueprints.launchpad.net/neutron/+spec/vlan-aware-vms

This blueprint proposes how to incorporate VLAN aware VMs into
OpenStack. In this document a VLAN aware VM is a VM that sends and receives
VLAN tagged frames over its vNIC. Pros are better scalability and more
dynamic handling of Neutron networks connected to VMs.


Problem Description
===================

Currently VLANs in VMs are not integrated with Neutron. In certain OVS
based plugins VMs can only send untagged traffic. This is because OVS
currently does not support QinQ. Even if OVS did support QinQ we would need
something to integrate VM tagged traffic with rest of Neutron.

Use cases:

* There are old legacy applications that are VLAN aware. These applications
  can more easily be adapted to OpenStack.

* Some applications handle a large number (+100) of Neutron networks. It is
  more optimized to use VLANs instead of using one vNIC per network.

* VLANs are more dynamic. It's usually less complex to add/remove a VLAN
  than to add/remove a vNIC, from a VM perspective.

* Service VMs could use this to serve VMs from other Tenants if admin gives
  adequate permissions to the Service VMs.

* This proposal gives an explicit way to control VLAN restrictions on trunk
  ports. One can choose which VLANs can pass which port, thus restricting a
  VLAN to a subset of VMs. The approach requires an underlying mechanism
  to be VLAN-aware, not VLAN-agnostic.

* VLAN-aware mechanism can be used to expose particular VLANs from within
  the Neutron infrastructure to an external networks. The usecase like the
  one above with inter-VM VLAN restrictions.

* The existing hardware can effectively work with VLANs and QinQ, but QinQ
  still is not supported by Neutron, while the VLAN-transparent (or data
  agnostic) way can lead to issues with the hardware.

* A VM may be running many containers. Each container may have requirements
  to be connected to different Neutron networks. Assigning a VLAN id for
  each container is a nice way to handle this.

The proposal may look related in some sense to the VLAN-transparent
approach [1], but differs in that it doesn't provide a way to send opaque
data through the links, but instead be aware of the encapsulating, provided
by the VM (or an appliance inside a VM).


Proposed Change
===============

The proposal is to extend Neutron with a new first-class resource, so to
say «trunk-port», that would be able to discriminate the VM's traffic
by VLAN tags. The trunk port is supposed to be attached to the VM and
receive both untagged and tagged traffic. To the trunk port can be
attached 0 .. k Neutron ports, or «subports». The trunk port can not have
a network attached. The subport must have a network attached. Each subport,
being attached to a trunk port, has additional extended attributes, VID,
VID type and parent ID, becomes «bound» and can not be attached to a VM
directly.

The tenant traffic that comes via trunk port to the subport from the VM,
is being remapped — packet tags are replaced with internal Neutron ones,
unique to the Neutron infrastructure. No tagged tenant traffic passes
the Neutron network.

The tenant does not need any special privileges to manage its trunk or
subports. Only admin can create and attach subports for different tenants.

Example of logical model::

                   +---------+
                   | S1      |------------ N1 provider:network_type vlan
                   | VLAN 10 |                provider:segmentation_id 102
                   +---------+
   +------+       /
   |      |+---+ /
   |      ||   |/  +---------+
   |  VM  || T |---| S2      |------- N2 provider:network_type vlan
   |      ||   |\  | VID: -- |           provider:segmentation_id 103
   |      |+---+ \ +---------+
   +------+       \
                   +---------+
                   | S3      |------------ N3 provider:network_type gre
                   | VLAN 20 |                provider:segmentation_id X
                   +---------+

T = Trunk port, S1-3 = Subports, N1-3 = Networks

In above example a VM connects to 3 Neutron networks. T is a trunk port and
represents the NIC. T has three subports, the untagged traffic goes to S2
and tagged to S1 and S3. On the S1 and S3 tenant tags are stripped and
the traffic is forwarded to the Neutron network.

On the other side the reverse procedure is applied, and tenant tags are
placed back.

+---------+--------------+
|VM sends |Frame goes to |
+=========+==============+
|Untagged | N2           |
+---------+--------------+
|VID 10   | N1           |
+---------+--------------+
|VID 20   | N3           |
+---------+--------------+

Example of commands:

::

 neutron trunk-port-create --name T

Above trunk-port-create returns id of the trunk port (T-id).

::

 neutron port-create --name S1 N1 --subport:parent_id <T-id>
 --subport:vid 10 --subport:vid_type VLAN

::

 neutron port-create --name S2 N2 --subport:parent_id <T-id>

By default the subport works with untagged (unmatched) traffic.

::

 neutron port-create --name S3 N3 --subport:parent_id <T-id>
 --subport:vid 20 --subport:vid_type VLAN

By default subports inherit the trunk-port MAC address, but this can be
overriden, if needed.

.. use :maxwidth: 240

DHCP Example:

.. actdiag::

   actdiag {
     send_dhcp -> t_dhcp -> n_dhcp -> dhcp ->
     rn_dhcp -> rt_dhcp -> receive_dhcp
     lane vm {
       label = "VM"
       send_dhcp [label = "Send DHCP on VLAN"];
       receive_dhcp [label = "DHCP reveived on VLAN"];
     }
     lane trunk {
       label = "Trunk port/Subport"
       t_dhcp [label = "Convert to network_type format"];
       rt_dhcp [label = "Convert to VLAN format"];
     }
     lane network {
       label = "Neutron network"
       n_dhcp [label = "Transmit over network"];
       rn_dhcp [label = "Transmit over network"];
     }
     lane dhcp {
       label = "DHCP Agent"
       dhcp [label = "Reply on DHCP"];
     }
   }


Constraints
-----------

* A trunk port has no parent and no VID.

* A subport has to have exactly one parent and it is a trunk port.

* Every subport of a given VID type should have unique VID among its
  siblings.

* There can be only one subport for "untagged" traffic. The subport will
  receive all the packets that are not matched by classifiers. In the
  general case it will be untagged packets.

* When the trunk port is bound, all the subports are marked as bound.
  A subport, being a normal Neutron port, can be rebound to the VM directly.

* A normal user can only connect subports to its own trunk ports. Admin
  user can connect subports to trunk ports with different owners.


Nova changes
------------

Since the trunk port is a first class resource, Nova should be changed
to be able to start VMs with trunk-port instead of normal port, if needed.

Alternatives
------------

One alternative is to extend the port so that it can be connected to
multiple networks directly. The main disadvantage with that is that it
would change the port in such way that it affected other services in
neutron that uses port information from ports connected to VMs. An example
of this is the DHCP agent. Another benefit of using solution with trunk
port and subports is that it probably is easier support service VMs that
connect to multiple Tenants. There will be one subport on each Tenant
network instead of one trunk port connected to networks from different
networks.

An option to associate the VID on the subport could be to associate it with
the network and let the tenants decide VID per network. A drawback with
this would be in the case a trunk port is used to connect to several
tenants. Different tenants could select the same VID for networks connected
to the service VM.

Another option could be to not have a separate VID associated with the
network but use the segmentation ID instead. This would limit this feature
to VLAN based Neutron networks. Also, if VIDs can't be chosen freely by the
users, more logic is needed in users to determine which VIDs to use.

Another alternative is to use a L2-gateway together with a true trunk
Neutron network (a Neutron network that can carry VLAN tagged
traffic).

Things to consider would be:
  * How to manage a VMs IP addresses for different VLANs on a trunk Neutron
    network. DHCP support for these IP addresses.
  * How to control the broadcast domains for the different VLANs. Possibly
    add support to select which VLANs a Neutron port is member of.
  * The most straight forward solution in plugins using OVS requires
    addition of QinQ support in OVS. Although VLAN in top of GRE and VXLAN
    type networks could probably be implemented using OF metadata.

Benefits would be that user do not have to manage each VLAN
separately. This means easier usage when trunking many VLANs between points
without caring about the content. Also less Neutron networks would be
consumed.

Another benefit is that L2 gateway is a
better solution when generic tranlation between VLAN, VXLAN etc. is exposed
to Tenant. See next section regarding adding this support to proposed
proposal. Different implementations of the L2 gateway could support
different things. To handle this a API to request/query support of L2
gateways could be added.

Current proposal is to always translate to/from VLAN for VM side. The
network_type for a neutron network could be VLAN, VXLAN or other and will
be translated to VLAN when send to VM. A more generic alternative could be
to be able to specify what to translate to/from. It would be possible for a
user to select how a network would be represented in a VM (VLAN, VXLAN or
other). This would be more complex solution and also require more
parameters, like IP addresses for VXLAN. All information required to setup
the specified network_type needs to exist in the API parameters.

Data Model Impact
-----------------

New objects:

::

       +--------------+
       |              |
       |   TrunkPort  |
       |              |
       +--------------+
              | 1
              |
              | N
       +--------------+
       | Neutron port |
       |              |
       |  + extended  |
       |  attributes  |
       +--------------+

TrunkPort
  * port_id - id of port.
  * trunk_type - type of port, currently only trunk or unset.

Neutron port
  * parent_id - port id of parent.
  * vid - VID used to reach attached network.
  * vid_type - VID type (VLAN) of the tenant traffic.

REST API Impact
---------------

Subport extended attributes for port resource

+----------+-------+---------+---------+------------+---------------------+
|Attribute |Type   |Access   |Default  |Validation/ |Description          |
|Name      |       |         |Value    |Conversion  |                     |
+==========+=======+=========+=========+============+=====================+
|subport:  |string |CR, all  |''       |uuid of     |ID of the trunk      |
|parent_id |(UUID) |         |         |parent      |port that subport is |
|          |       |         |         |            |connected to.        |
+----------+-------+---------+---------+------------+---------------------+
|subport:  |integer|CR, all  |''       |uint or     |VID that will be     |
|vid       |       |         |         |''          |used to access this  |
|          |       |         |         |            |subport from trunk   |
|          |       |         |         |            |port. VID has to be  |
|          |       |         |         |            |unique among subport |
|          |       |         |         |            |siblings. Empty VID  |
|          |       |         |         |            |means unmatched      |
|          |       |         |         |            |traffic              |
+----------+-------+---------+---------+------------+---------------------+
|subport:  |string |CR, all  |'VLAN'   |enum (*)    |VID type of the      |
|vid_type  |       |         |         |            |tenant traffic that  |
|          |       |         |         |            |will come from VM.   |
|          |       |         |         |            |                     |
|          |       |         |         |            |                     |
|          |       |         |         |            |                     |
|          |       |         |         |            |                     |
+----------+-------+---------+---------+------------+---------------------+

(*) As the blueprint is focused on VLAN tagging, only 'VLAN' vid_type
is defined here. Any extension is a subject of further separate blueprints.

Security Impact
---------------

Only admin can create and attach subports for different tenants.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

Tenants should be aware that OpenStack does nothing to enable VMs
to handle the tagged traffic, but just provides tagged packets. It
is totally up to the user to set VMs up properly.

Performance Impact
------------------

Performance of legacy functionality should be unaffected. Data path for non
trunk ports are unchanged.

IPv6 Impact
-----------

None. Tenants should be able to specify IPv6 address per subport.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

Extented attributes on the Neutron port are to be used.

A new first-class resource is to be provided.

Requires the testing framework changes.


Community Impact
----------------

The new first-class resource, the trunk port, impacts not
only Neutron, but Nova also, providing additional ways to
start VM.

Bound ports (subports) differ in the behaviour from normal
ports and should not be treated as normal ports.

Implementation
==============

Assignee(s)
-----------

Kevin Benton
Peter V. Saveliev

Work Items
----------

* Security Groups support.
* VLAN support.
* Unit test.
* Tempest test.
* Scenario test.

Dependencies
============

[WIP] Requires Nova changes and may require ML2 changes.

Testing
=======

Tempest and functional tests will be created.

Tempest Tests
-------------

Tempest tests to be implemented:

* Create trunk ports
* Create subports
* Bind trunk ports
* Delete subports
* Delete trunk ports

Functional Tests
----------------

Tests to be implemented:

* Boot VM with one trunk port with subport for untagged traffic.
  Verify connectivity.
* Boot VM with one trunk port with multiple subports.
  Verify connectivity.
* Boot VM with multiple trunk ports and subports.
* Add subport to running VM with trunk port.
* Remove subport from running VM with trunk port.
* Delete VM with trunk port including subports.

API Tests
---------

Tests to be implemented:

* Check that subport only can be connected to trunk port.
* Check of valid port type.


Documentation Impact
====================

The trunk port, being the first-class resource, should be
documented. The subports behaviour should be described.

Possible scenarios for usecases should be provided with
CLI examples.

User Documentation
------------------

Update networking API reference.
Update admin guide.

Developer Documentation
-----------------------

The trunk port resource description, subports behaviour, API reference.

References
==========

None

