..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================
Change the segment ID of a VLAN provider network
================================================

Launchpad bug:
https://bugs.launchpad.net/neutron/+bug/1806052

This spec proposes a method to change the segmentation ID of a VLAN-type
network, which is the external VLAN ID of the network. The implementation
described is limited to only the OVS back-end and to single-segment
provider networks.


Problem Description
===================

Currently, Neutron does not allow the segmentation ID parameter of a provider
network to change, regardless of the status of the network (active or not) or
if ports exist on the network (or not). In order to change the segmentation ID,
the administrator needs to delete the network, first un-binding any bound
ports, then create a new network with the required segmentation ID, and
finally bind the ports back to the network.


Proposed Change
===============

The proposed solution is to be able to change the segmentation ID of a VLAN-
type network, using one single command ("openstack network set") in an atomic
operation.

In order to implement this, several changes are needed:

* The CLI, to allow to update the segmentation ID in an existing network.
* The Neutron server, to allow writing those changes in the database.
* The networking back-end. In this spec only OVS is covered and the
  implementation will be limited to it.


OpenStack Client
----------------

Currently, the OpenStack Client allows updating the segmentation ID of an
existing network. No changes are needed.


Neutron server
--------------

Currently, when a network update message is sent to the Neutron server, if any
provider attribute is present in this message, for example 'segmentation ID',
this command is rejected. The new implementation will allow updating the
segmentation ID:

* If the segmentation ID is the only provider parameter to be updated. No other
  provider parameters (network type, physical network) are allowed.
* If the network has only one segment. In multi provider networks the server
  doesn't know which one of the segments should be updated.
* If the provider segment type is 'vlan'. No other types are allowed.
* If all ports bound to this network belong to a compatible network driver. A
  compatible network driver is one which has implemented this change. In
  this spec, the OVS agent implementation is described. For example, with OVS,
  ports with VIF type 'vhostuser' and 'ovs' are accepted, and 'unbound' or
  'binding_failed' are also allowed because they are not yet bound to any
  network driver.

If the above conditions are all met, the Neutron server will execute the
following actions (in order):

* Validate the new segment, to know if the segmentation ID falls inside the
  minimum and maximum VLAN tags.
* Reserve a new provider segment. This will ensure the segmentation ID is not
  used in this network.
* Create a new provider segment in the database.
* Release the old provider segment.
* Delete the old provider segment from the database.

All these operations will be executed inside a write context, to provide
concurrency and enable rollback in case of failure. If any of these database
operations fail, including subsequent ones, the context will prevent the old
segment from being deleted and the new one will not be created.


Mechanism driver
----------------

The agent mechanism driver interface will be extended with a new function. This
function will return a list of provider network attributes that can be modified
in a network with ports bound to this networking back-end.

By default, this function will return an empty list.


Neutron OVS agent
-----------------

In the OVS agent, VLAN provider networks can access the provider network
outside the host through the physical bridges. Each provider network has a
single physical bridge assigned. A physical bridge can host the traffic of
multiple provider networks, but each provider network can only send and receive
traffic through one physical bridge. There is a N:1 (network:physical bridge)
relationship.

Inside the integration bridge, the tenant networks are isolated using virtual
networks. These virtual networks have their own VLAN ID mapping, different from
the provider networks VLAN ID mapping. Both VLAN maps are related through the
"LocalVlanManager", which is in charge of correlating both mappings. When a
packet from a tenant exits the integration bridge to the corresponding physical
bridge, a flow changes the VLAN tag from the tenant VLAN ID to the provider
segmentation ID. The opposite process happens in the physical bridge::

  br_int:
  cookie=0xb46613524bd0de42, duration=23646.484s, table=0, n_packets=0,
    n_bytes=0, priority=3,in_port="int-br-phy",dl_vlan=1074
    actions=mod_vlan_vid:1,resubmit(,60)

  br_phy:
  cookie=0x43209f9280b7fc88, duration=23666.003s, table=0, n_packets=10,
    n_bytes=788, priority=4,in_port="phy-br-phy",dl_vlan=1
    actions=mod_vlan_vid:1074,NORMAL

When a network segmentation ID is updated, the only change needed is to update
the new segmentation ID in both flows. This could be done by deleting both
flows and then creating them again with the new segmentation ID. For example::

  phys_br.reclaim_local_vlan(port=phys_port, lvid=lvid)
  phys_br.provision_local_vlan(
      port=phys_port, lvid=lvid,
      segmentation_id=new_segmentation_id,
      distributed=self.enable_distributed_routing)
  self.int_br.reclaim_local_vlan(port=int_port,
      segmentation_id=old_segmentation_id)
  self.int_br.provision_local_vlan(
      port=int_port, lvid=lvid,
      segmentation_id=new_segmentation_id)


Limitations
-----------

As commented, this solution will be available only for:

* Single segment networks.
* VLAN type networks.
* Only OVS (kernel OVS or DPDK OVS) bound ports to this network.


Data Model
----------

No changes.


REST API
--------

A new ML2 RPC call, requested from the agent side, is needed. This new call
will provide information about the updated network. With this information the
agent will know if the network has changed the segmentation ID (as commented,
the "LocalVlanManager" singleton stores this information).


CLI impact
----------

No changes.


Operation Impact
----------------

The segmentation ID change implies two actions:

* To change the segmentation ID of a Neutron network without unbinding the
  attached ports. Once this process is done in Neutron, the traffic from those
  ports will be tagged with the new VLAN ID and transmitted through the
  provider network.
* To change the provider network segmentation. This process is external to
  Neutron (and OpenStack).

Because those processes cannot be done at the same time, the administrator
should decide, depending on the site network infrastructure, the order of
execution. Until both the segmentation ID of the Neutron network and the
VLAN ID of the external provider network are synchronized, there will be a
networking disruption.


References
==========

None.
