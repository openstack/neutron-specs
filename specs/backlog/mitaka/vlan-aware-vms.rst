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
VLAN tagged frames over its vNIC. The main point of that is to overcome the
limitations of the current one vNIC per network model. A VLAN (or other
encapsulation) aware VM can differentiate between traffic of many networks by
different encapsulation types and IDs, instead of using many vNICs. This
approach scales to higher number of networks and enables dynamic handling of
network attachments (without hotplugging vNICs).


Problem Description
===================

Currently VLANs in VMs are not integrated with Neutron.  There are various use
cases where it would be useful to allow a VM to be attached to multiple Neutron
networks using VLANs as a local encapsulation to differentiate the traffic for
each network as it goes in/out of a single VIF.

Use cases:

* Some applications have requirements to connect to many (say, hundreds) of
  Neutron networks.  It is more practical to use a single or other small number
  of VIFs and VLANs to differentiate traffic for each network than to have
  hundreds of VIFs per VM.

* Cloud workloads are often very dynamic.  It may be more efficient and/or less
  complex to add/remove VLANs than to hotplug interfaces in a VM.

* A VM could be moved from one network to another without detaching the VIF
  from the VM.

* A VM may be running many containers. Each container may have requirements to
  be connected to different Neutron networks. Assigning a VLAN (or other
  encapsulation) id for each container is more efficient and scalable than
  requiring a vNIC per container.

* There are legacy applications that expect to use VLANs as a way to connect to
  multiple networks.  Neutron should provide a way to expose that model to the
  VM decoupled from how the network is actually implemented.

The proposal may look related in some sense to the VLAN-transparent
approach (bp/nfv-vlan-trunks), but differs in that it doesn't provide a
way to send opaque data through the links, but instead be aware of the
encapsulating, provided by the VM (or an appliance inside a VM).


Proposed Change
===============

The proposal keeps the port resource unchanged.

::

    $ neutron port-create NETWORK

The new API extension looks like this:

#. One new trunk resource with member_actions for subport operations:

    * One trunk resource
    * Subports encoded in an attribute of the trunk, added and removed by
      member_actions

The proposal is to add an API extension to turn already existing ports
into trunk and subports.  The best way to think about this is that
Neutron needs a method to describe a logical topology such that you have a
regular VIF port (the parent) and that a parent may also have child ports.
These child ports are used to connect the VIF port to further networks.
Both the parent and child ports have all of the attributes that a port
has today (however some subport attributes must behave differently,
eg. binding:host_id) and can be created on any Neutron network.
All needed information beyond legacy port attributes is contained in
newly introduced resource(s) referring legacy ports.

The first part of the API extension is the logical topology portion.
First, we include a method to indicate that a port may have child
ports.  This isn't strictly necessary, but is included for convenience.
It allows Neutron to go ahead and reject the port creation if subports
aren't supported by the backend.

::

    # trunk-create may refer to 0, 1 or more subport(s).
    $ neutron trunk-create --port-id PORT \
                          [--subport PORT[,SEGMENTATION-TYPE,SEGMENTATION-ID]] \
                          [--subport ...]

All ports referred must exist.

Note that if a given plugin does not implement support for
bp/vlan-aware-vms, this request will be rejected.

If the backend does not implement the extension any trunk operation will fail
(with 404). If a plugin does implement the extension, trunk operations may
return a conflict error if the port status is not compatible with the request.

In addition to the logical topology piece, we need to define the type
of encapsulation (segmentation) used between the hypervisor and VM for
targeting packets to a logical subport, as well as for determining the
source logical port for a packet arriving from the VM. The encapsulation
here is local, used between the VM and hypervisor only.  It has nothing
to do with how the Neutron network the port is attached to is implemented.
SEGMENTATION-TYPE is an enum of strings, currently having one valid value
'vlan'. SEGMENTATION-ID is an unsigned integer.  SEGMENTATION-TYPE and
SEGMENTATION-ID are optional in both trunk-create and trunk-subport-add
in the generic case, allowing neutron server to choose the type and id
if needed. On the other hand backends are allowed to reject requests
with unspecified segmentation details and make the user supply them.
Further segmentation types may be introduced by later specs.

::

    # trunk-add-subport adds 1 or more subport(s)
    $ neutron trunk-subport-add TRUNK \
                                PORT[,SEGMENTATION-TYPE,SEGMENTATION-ID] \
                               [PORT,...]

All ports referred must exist.

The CLI has the following new commands:

* trunk-create
* trunk-delete
* trunk-list
* trunk-show [should not print subports]
* trunk-subport-add
* trunk-subport-delete
* trunk-subport-list

Importantly, after creating trunks and maybe subports, VMs are still booted
with a reference to plain old ports:

::

    $ nova boot --nic port-id=...

Nova need not know about trunks and subports.

Example logical model::

            +-----------+
            |           |
            | +-------+ |   +------+
            | | port1 +-----+ net1 |
            | +-------+ |   +------+
 +-----+    |           |
 |     |    | +-------+ |   +------+
 | vm0 +------+ port0 +-----+ net0 |
 |     |    | +-------+ |   +------+
 +-----+    |           |
            | +-------+ |   +------+
            | | port2 +-----+ net2 |
            | +-------+ |   +------+
            |           |
            +-----+-----+
                  ^
                  |
  +----------------+--------------+
  |                               |
  | Ports combined into one VIF   |
  | by turning port0 into a trunk |
  | and the other ports into      |
  | subports of the trunk.        |
  |                               |
  +-------------------------------+

* The traffic of port0 is untagged in vm0.
* The traffic of port1 is encapsulated, for example as vlan id 100.
* The traffic of port2 is encapsulated, for example as vlan id 200.

In the above example, a VM connects to 3 Neutron networks.  port0 is
a regular port that handles untagged traffic.  port0 has two subports,
each using VLAN encapsulation with a different VLAN ID.  Packets targeted
at a subport will be tagged with the appropriate VLAN ID before being
sent to the VIF.  Packets received on the VIF tagged with a VLAN ID
associated with a subport will be treated as if the logical source was
that subport and the packet will be sent to the Neutron network that
subport is attached to.

Subports have independent, arbitrary MAC addresses just like any other Neutron
port.  If the application requires having subports use the same MAC address as
their parent, the port create API supports specifying the MAC address, so it
can be specified to match the parent's MAC address.  A parent and a subport
having the same MAC address are not allowed to be on the same net.

Nested subports are not supported.

Constraints
-----------

* A top level port that may have child ports added must be marked by creating
  a trunk referring to the port in its `port_id` attribute.

* Every subport of a given parent should have unique (segmentation_type,
  segmentation_id) among its siblings.  Otherwise, it would not be possible
  to properly differentiate the source and destination logical port.

* The parent port handles "untagged" traffic.  The parent will receive
  all the packets that do not match a subport, which is no
  different than how a regular port already behaves today.  This also
  means that every VM has a port for untagged traffic.  It doesn't
  necessarily have to be used though and could be attached to a dummy
  Neutron network if desired.  You could also have a security group
  set on the parent port to drop all incoming and outgoing traffic.
  A future enhancement could include the ability to create a Neutron
  port not yet attached to a network, though it's unclear how valuable
  that actually is.

* A port with a parent logically inherits its binding:host_id from its parent.

* An attempt to update binding:host_id of a subport (by booting a Nova VM with
  a subport UUID, for example) must result in an error.

* A normal user can only connect subports to its own parent ports. Admin
  can connect subports to parent ports of different owners.

* Until OVS supports QinQ, an OVS based Neutron backend cannot support having a
  subport attached to a "vlan_transparent" network.  That would require the
  abilty to transport VLAN tagged traffic over the VLAN tagged subport interface
  (nested VLANs), which is not yet possible.

The following deletion constraints exist:

* Deletion of a trunk automatically deletes all of its subports.

* Deletion of a (child) port referred by a subport is forbidden. The subport
  must be deleted first.

* Deletion of a (parent) port referred by a trunk is forbidden. The trunk
  must be deleted first.

Nova changes
------------

The Neutron OVS agent uses OVS in such a way that it will likely need
to create a new OVS bridge per top level parent port.  That requires
an enhancement to the existing OVS and vhost-user VIF types in Nova to
allow Neutron to tell Nova the name of the bridge to use for the port.
Currently, Nova only supports a single bridge name provided by the Nova
configuration file.

Alternatives
------------

For alternatives please see the history of the spec, including:

* v1: https://review.openstack.org/#q,I015762abe197b91916374143410b1f182908909e,n,z
  patch sets 1 through 10
* v2: https://review.openstack.org/#q,I015762abe197b91916374143410b1f182908909e,n,z
  patch sets 11 through 21
* v3: https://review.openstack.org/#q,I8563e54cc984b60f2be2eb7cc2e3c6fc60e6de20,n,z
  patch sets 1 through 5
* v4: https://review.openstack.org/#q,I8563e54cc984b60f2be2eb7cc2e3c6fc60e6de20,n,z
  patch sets 6 and later (the current)

Data Model Impact
-----------------

Neutron port: no changes.

Neutron DB table: trunks

* id: integer primary key
* uuid
* name
* tenant_id
* port_id: foreign key to ports.id

Neutron DB table: subports

* id
* port_id: foreign key to ports.id
* trunk_id: foreign key to trunks.id
* segmentation_type
* segmentation_id

Neutron DB constraints:

* (segmentation_type, segmentation_id, trunk_id) must be unique in the
  subports table
* Ports referred by trunks.port_id must not be referred by subports.port_id
  and vice versa.

REST API Impact
---------------

trunk resource:
  * id
  * name
  * tenant_id
  * port_id

::

    POST /v2.0/trunks
         ...
    PUT /v2.0/trunks/TRUNK-ID/add_subports
    PUT /v2.0/trunks/TRUNK-ID/delete_subports
         {'subports':
             [{'port_id': PORT_ID,
               'segmentation_type': SEGMENTATION_TYPE,
               'segmentation_id': SEGMENTATION_ID},
              ...
             ]}}
    GET /v2.0/trunks/TRUNK-ID/subports

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

The performance of existing functionality should be unaffected.  The data path
for normal ports is unchanged.

In some cases, this change may improve performance.  Without this change,
connecting a VM to many Neutron networks required a VIF per network.  With this
change, you could connect to 1000 Neutron networks with very little overhead vs.
having to attach 1000 virtual interfaces to your VM before.  The tradeoff is
that some additional burden is shifted into the VM to deal with that
segmentation on a single interface.

IPv6 Impact
-----------

None.  Both parent ports and subports have all of the same attributes as Neutron
ports do today, including IPv6 addresses if desired.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

* New neutron resources are to be used
* Requires modifications to Neutron plugins to support this model
* Requires development of new tests.

Community Impact
----------------

Implementation
==============

Assignee(s)
-----------

* Kevin Benton
* Peter V. Saveliev
* Russell Bryant
* Bence Romsics

Work Items
----------

* API extension and DB schema updates
* Unit tests for API+DB changes
* Tempest tests for creating port topology
* Tempest scenario test(s) for doing functional validation
* Neutron Plugin support.
  * networking-ovn (OVN supports this model already)
  * ml2+ovs
* Expose subport information in nova metadata service (likely will require a new
  spec)

Dependencies
============

The change needed for the existing OVS VIF type has `its own blueprint
<https://blueprints.launchpad.net/nova/+spec/neutron-ovs-bridge-name>`_.

Testing
=======

Tempest and functional tests will be created.

Full-stack and in-tree API tests
--------------------------------

* Create parent ports
* Create subports
* Bind parent ports
* Delete subports
* Delete parent ports

Functional Tests
----------------

Tests to be implemented:

* Boot VM with one parent port and no children.
  Verify connectivity.
* Boot VM with one parent port and multiple subports.
  Verify connectivity to each logical port.
* Boot VM with multiple parent ports and subports and verify connectivity.
* Add subport to running VM with a parent port.
* Remove subport from running VM with parent port.
* Delete VM with parent port including subports.

API Tests
---------

Tests to be implemented:

* Check that subport only can be connected to a parent port.
* Check that an invalid segmentation_type is rejected
* Check that an invalid segmentation_id is rejected


Documentation Impact
====================

The use of parent ports and subports should be documented as a way to create a
logical multi-port topology using a single VIF on a VM.

Possible scenarios for use cases should be provided with
CLI examples.

User Documentation
------------------

Update networking API reference.
Update admin guide.

Developer Documentation
-----------------------

The parent port resource description, subports behaviour, API reference.

References
==========

* `OVS VIF type: pass bridge name from neutron to nova
  <https://blueprints.launchpad.net/nova/+spec/neutron-ovs-bridge-name>`_
* Subport details may be exposed to VMs via the Nova metadata service in a
  later spec like this: `Virtual guest device role tagging
  <https://review.openstack.org/#q,I16845bd36878bbd9d7a877dc556b2650bc6f0fad,n,z>`_.
* `nfv-vlan-trunks
  <https://blueprints.launchpad.net/neutron/+spec/nfv-vlan-trunks>`_
