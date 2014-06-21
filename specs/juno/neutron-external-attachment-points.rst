====================================
External Attachment Points Extension
====================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/neutron-external-attachment-points


This blueprint introduces the concept of external attachment points into Neutron
via an extension so external devices (e.g. baremetal workloads) can gain access
to Neutron networks. External attachment points will specify an attachment ID to
locate the port in the physical infrastructure (e.g. Switch1/Port5) as well as the
neutron network that it should be a member of. These can be referenced in port creation
requests from externally managed devices such as ironic instances.


Problem description
===================

There is no well-defined way to connect devices not managed by OpenStack
directly into a Neutron network. Even if everything is manually configured
to setup connectivity, the neutron DHCP agent will not issue addresses to
the device attached to the physical port since it doesn't have a corresponding
Neutron port. A neutron port can be created to match the MAC of an
external device to allow DHCP, but there is nothing to automate the process
of configuring the network attachment point of that device to put it in the
correct VLAN/VXLAN/etc.


Proposed change
===============

To integrate these external devices into Neutron networks, this blueprint
introduces the external attachment point extension. It will add a new mixin
class that will handle the database records for the external attachent points.
The actual configuration of the network devices the attachment points reside on
will be dependent on the plugin. The reference implementation will include an
OVS-based attachment gateway.

The new resource type this extension will introduce is the external attachment
point. It iss responsible for capturing the external attachment identification
information (e.g. Switch1/Port5).

External attachment points are created by admins and then assigned to a tenant for use.
The tenant can then assign the external attachment point to any Neutron networks that
he/she can create Neutron ports on. When it is assigned to a network, the backend will
provision the infrastructure so that attachment point is a member of the Neutron network.

A port creation request may reference an external attachment ID. This will prevent the
external attachment from being deleted or assigned to a different network while any Neutron
ports are associated with it.


Relational Model
----------------

::

      +-----------------+
      | Neutron Network |
      +-----------------+
              | 1
              |
              | M
 +---------------------------+
 | External Attachment Point |
 +---------------------------+


Example CLI
-----------

Create external attachment point referencing a switch port (admin-only).

::

 neutron external-attachment-point-create --attachment_type 'vlan_switch' --attachment_id switch_id=00:12:34:43:21:00,port=5,vlan_tag=none

Assign external attachment point to a tenant (admin-only).

::

 neutron external-attachment-point-update <external_attachment_point_id> --tenant-id <tenant_id>

Assign external attachment point to neutron network (admin-or-owner).

::

 neutron external-attachment-point-update <external_attachment_point_id> --network-id <network_id>


Create a neutron port referencing an external attachment point to prevent the attachment point from being deleted/re-assigned.

::

 neutron port-create --external-attachment-point-id <external_attachment_point_id>



Use Cases
---------

*Ironic*
  Ironic will be responsible for knowing the MAC address and switch port of each
  instance it manages. This will either be through manual configuration or through
  LLDP messages received from the switch. Using this information, it will create
  and manage the life cycle of the external attachment point associated with each
  server. Since this process is managed by Ironic and since Neutron ports can't be
  assigned to a different network after creation, the external attachment object
  never needs to be assigned to the tenant.


*L2 Gateway*
  This is a generic layer 2 attachment into the Neutron network. This could be
  a link to an access point, switch, server or any arbitrary set of devices that
  need to share a broadcast domain with a Neutron network. In this workflow an
  admin would create the attachment point with the switch info and either assign
  it to a neutron network directly or assign it to a tenant who would assign
  it to one of his/her networks as necessary.


Alternatives
------------

There isn't a very good alternative right now. To attach an external device to
a Neutron network, the admin has to lookup the VLAN (or other segment
identifier) for that network and then manually associate a port to that VLAN.
The end device then has to be configured with a static IP in an exclusion range
on the Neutron subnet or manually create a neutron port with the matching MAC.
Tenants then cannot assign the physical device to a different network,
or see any indication that the device is actually attached to the network.


Data model impact
-----------------

For plugins that leverage this extension, it will add the two following tables:
external attachment points and external attachment points port bindings.

The external attachment point table will contain the following fields:

* id - standard object UUID
* name - optional name
* description - optional description of the attached devices
  (e.g. number of attached servers or a description of the L2 neighbors
  on other side)
* attachment_id - an identifier for the backend to identify the port
  (e.g. Switch1:Port5).
* attachment_type - a type that will control how the attachment_id should be
  formatted (e.g. 'vlan_switch' might require a switch_id, port_num, and an optional_vlan_tag).
  These will be enumerated by the backend in a manner that allows plugins to
  add new types.
* network_id - the ID of the neutron network that the attachment point should be a member of
* status - indicates status of backend attachment configuration operations (BUILD, ERROR, ACTIVE)
* status_description - provides details of failure in the case of the ERROR state
* tenant_id - the owner of the attachment point


The external attachment point port binding table will contain the following fields:

* port_id - foreign-key reference to associated neutron port
* external_attachment_point_id - foreign-key reference to external attachment point

This will have no impact on the existing data model. Neutron ports associated with
external attachment points can be deleted through the normal neutron port API.

Three attachment_type formats will be included.

* vlan_switch
  * switch_id - hostname, MAC address, IP, etc that identifies the switch on the network
  * switch_port - port identifier on switch (e.g. ethernet7)
  * vlan_tag - 'untagged' or a vlan 1-4095
* ovs_gateway
  * host_id - hostname of node running openvswitch
  * interface_name - name of interface to attach to network
* bonded_port_group
  * ports =  a list of port objects that can be any attachment_types


REST API impact
---------------

The following is the API exposed for physical ports.

.. code-block:: python

    RESOURCE_ATTRIBUTE_MAP = {
        'external_attachment_points': {
            'id': {'allow_post': False, 'allow_put': False,
                   'enforce_policy': True,
                   'validate': {'type:uuid': None},
                   'is_visible': True, 'primary_key': True},
            'tenant_id': {'allow_post': True, 'allow_put': True,
                          'required_by_policy': True,
                          'is_visible': True},
            'name': {'allow_post': True, 'allow_put': True,
                     'enforce_policy': True,
                     'validate': {'type:string': None},
                     'is_visible': True, 'default': ''},
            'description': {'allow_post': True, 'allow_put': True,
                            'enforce_policy': True,
                            'validate': {'type:string': None},
                            'is_visible': True, 'default': ''},
            # the attachment_id format will be enforced in the mixin
            # depending on the attachment_type
            'attachment_id': {'allow_post': True, 'allow_put': False,
                                 'enforce_policy': True,
                                 'default': False,
                                 'validate': {'type:dict': None},
                                 'is_visible': True,
                                 'required_by_policy': True},
            'attachment_type': {'allow_post': True, 'allow_put': False,
                                   'enforce_policy': True,
                                   'default': False, 'validate': {'type:string': None},
                                   'is_visible': True,
                                   'required_by_policy': True},
            'network_id': {'allow_post': True, 'allow_put': True,
                           'required_by_policy': True,
                           'is_visible': True},
            'ports': {'allow_post': False, 'allow_put': False,
                      'required_by_policy': False,
                      'is_visible': True},
            'status': {'allow_post': False, 'allow_put': False,
                       'required_by_policy': False, 'is_visible': True},
            'status_description': {'allow_post': False, 'allow_put': False,
                                   'required_by_policy': False, 'is_visible': True}
        },
        'ports': {
            'external_attachment_id': {'allow_post': True, 'allow_put': True,
                                       'is_visible': True, 'default': None,
                                       'validate': {'type:uuid': None}},
        }
    }


The following is the default policy for external attachment points.

.. code-block:: javascript

    {
        "create_external_attachment_point": "rule:admin_only",
        "delete_external_attachment_point": "rule:admin_only",
        "update_external_attachment_point:tenant_id": "rule:admin_only",
        "get_external_attachment_point": "rule:admin_or_owner",
        "update_external_attachment_point": "rule:admin_or_owner",
    }


Security impact
---------------

There should be no security impact to Neutron. However, these ports will
not have security group support so users won't have a way of applying
firewall rules to them.

Notifications impact
--------------------

N/A

Other end user impact
---------------------

The neutron command-line client will be updated with the new external
attachment point CRUD commands.

Performance Impact
------------------

None to plugins that don't use this extension.

For plugins that use this extension it will be limited since most of
this code is only called during external attachment CRUD operations.

Other deployer impact
---------------------

The level of configuration required to use this will depend highly
on the chosen backend. Backends that already have full network control
may not require any additional configuration. Others may require lists
of objects to specify associations between configuration credentials
and network hardware.

Developer impact
----------------

If plugin developers want to use this, they will need to enable the extension
and use the mixin module.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  kevinbenton

Other contributors:
  kanzhe-jiang

Work Items
----------

* Complete DB mixin model and extension API attributes
* Update python neutron client to support the new external attachment point commands
* Implement extension in ML2 in a multi-driver compatible way
* Implement an experimental OVS-based network gateway reference backend
  (allows physical or virtual ports to be used as external attachment points)


Dependencies
============

N/A

Testing
=======

Unit tests will be included to exercise all of the new DB code and the API.
Tempest tests will leverage the reference OVS-based network gateway implementation.

Documentation Impact
====================

New admin and tenant workflows need to be documented for this extension.
It should not impact any other documentation.


References
==========

The following are notes from the mailing list[1] regarding Ironic use, but they
are not all requirements that will be fulfilled in the initial implementation:

* Bare metal instances are created through Nova API with specifying networking
  requirements similarly to virtual instances. Having a mixed environment with
  some instances running in VMs while others in bare metal nodes is a possible
  scenario. In both cases, networking endpoints are represented as Neutron
  ports from the user perspective.

* In case of multi-tenancy with bare metal nodes, network access control
  (VLAN isolation) must be secured by adjacent EOR/TOR switches

* It is highly desirable to keep existing Nova workflow, mainly common for
  virtual and bare metal instances:

  * Instance creation is requested on Nova API, then Nova schedules the
    Ironic node

  * Nova calls Neutron to create ports in accordance with the user
    requirements. However, the node is not yet deployed by that time and
    networking is not to be "activated" at that point.

  * Nova calls Ironic for "spawning" the instance. The node must be connected
    to the provisioning network during the deployment.

  * On completion of the deployment phase, "user" ports created in step 2 are
    to be activated by Ironic calling Neutron.

 * It is a realistic use case that a bare metal node is connected with multiple
   NICs to the physical network, therefore the requested Neutron ports need to
   be mapped to physical NICs (Ironic view) - attachment points (Neutron view)

 * It is a realistic use case that multiple Neutron ports need to be mapped to
   the same NIC / attachment, e.g. when the bare metal node needs to be
   connected to many VLANs. In that case, the attachment point needs to be
   configured to trunking (C-tagging) mode, and C-tag per tenant network needs
   to be exposed to the user. NOTE(kevinbenton): this exact workflow will not
   be supported in this initial patch because attachment points are a 1-1
   mapping to a Neutron network.

 * In the simplest case, port-to-attachment point mapping logic could be
   placed into Ironic. Mapping logic is the logic that needs to decide which
   NIC/attachment point to select for a particular Neutron port withing a
   specific tenant network. In that case, Ironic can signal the requested
   attachment point to Neutron.

 * In the long-term, it is highly desirable to prepare the "Neutron port" to
   "attachment point" mapping logic for specific situations:

   * Different NICs are connected to isolated physical networks, mapping to
     consider network topology / accessibility

   * User wants to apply anti-affinity rules on Neutron ports, i.e. requesting
     ports that are connected to physically different switches for resiliency

   * Mapping logic to consider network metrics, interface speed, uplink
     utilization.cThese aspects argue to place the mapping logic into Neutron,
     that has the necessary network visibility (rather than Ironic)

 * In some cases, Ironic node is configured to boot from a particular NIC in
   network boot mode. In such cases, Ironic shall be able to inform the mapping
   logic that a specific Neutron port (boot network) must be placed to that
   particular NIC/attachment point.

 * It is highly desirable to support a way for automating NIC-to-attachment
   point detection for Ironic nodes. For that purpose, Ironic agent could
   monitor the link for LLDP messages from the switch and register an
   attachment point with the detected LLDP data at Ironic. Attachment point
   discovery could happen either at node discovery time, or before deployment.

1. http://lists.openstack.org/pipermail/openstack-dev/2014-May/thread.html#35298
