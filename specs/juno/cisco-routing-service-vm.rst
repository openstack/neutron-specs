..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================================================
Neutron routing service implemented using Cisco VM (and physical devices)
=========================================================================

https://blueprints.launchpad.net/neutron/+spec/cisco-routing-service-vm

Problem description
===================
There is need to support Neutron's routing service implemented in Cisco devices.
This blueprint targets that use case. In particular it implements routing using
the Cisco CSR1kv VM devices.

Proposed change
===============
- The implementation is done as a separate routing service plugin.
- Introduction of binding table to associate Neutron router with hosting device
- RPC Notifications and callbacks for interactions with Cisco configuration
  agent (the latter defined and implemented in BP/patch:
  https://blueprints.launchpad.net/neutron/+spec/cisco-config-agent)

Alternatives
------------
This could have been done as part of refactoring the existing routing service
plugin. However, this path was not taken in order to reduce impact as there is a
broader discussion in the community about L3 router modularization, flavor
framework etc.

Data model impact
-----------------
The basic l3 routing data models are extended via the regular Neutron extension
mechanism. The base classes will not be modified.

Three new DB tables: **hostingdevices**, **HostedHostingPortBinding** and
**routerhostingdevicebindings**

::

 class HostingDevice(model_base.BASEV2, models_v2.HasId, models_v2.HasTenant):
    """Represents an appliance hosting Neutron router(s).

       When the hosting device is a Nova VM 'id' is uuid of that VM.
    """
    # complementary id to enable identification of associated Neutron resources
    complementary_id = sa.Column(sa.String(36))
    # manufacturer id of the device, e.g., its serial number
    device_id = sa.Column(sa.String(255))
    admin_state_up = sa.Column(sa.Boolean, nullable=False, default=True)
    # 'management_port_id' is the Neutron Port used for management interface
    management_port_id = sa.Column(sa.String(36),
                                   sa.ForeignKey('ports.id',
                                                 ondelete="SET NULL"))
    management_port = orm.relationship(models_v2.Port)
    # 'protocol_port' is udp/tcp port of hosting device. May be empty.
    protocol_port = sa.Column(sa.Integer)
    cfg_agent_id = sa.Column(sa.String(36),
                             sa.ForeignKey('agents.id'),
                             nullable=True)
    cfg_agent = orm.relationship(agents_db.Agent)
    # Service VMs take time to boot so we store creation time
    # so we can give preference to older ones when scheduling
    created_at = sa.Column(sa.DateTime, nullable=False)
    status = sa.Column(sa.String(16))

 class HostedHostingPortBinding(model_base.BASEV2):
    """Represents binding of logical resource's port to its hosting port."""
    logical_resource_id = sa.Column(sa.String(36), primary_key=True)
    logical_port_id = sa.Column(sa.String(36),
                                sa.ForeignKey('ports.id',
                                              ondelete="CASCADE"),
                                primary_key=True)
    logical_port = orm.relationship(
        models_v2.Port,
        primaryjoin='Port.id==HostedHostingPortBinding.logical_port_id',
        backref=orm.backref('hosting_info', cascade='all', uselist=False))
    # type of router port: router_interface, ..._gateway, ..._floatingip
    port_type = sa.Column(sa.String(32))
    # type of network the router port belongs to
    network_type = sa.Column(sa.String(32))
    hosting_port_id = sa.Column(sa.String(36),
                                sa.ForeignKey('ports.id',
                                              ondelete='CASCADE'))
    hosting_port = orm.relationship(
        models_v2.Port,
        primaryjoin='Port.id==HostedHostingPortBinding.hosting_port_id')
    # VLAN tag for trunk ports
    segmentation_tag = sa.Column(sa.Integer, autoincrement=False)

 class RouterHostingDeviceBinding(model_base.BASEV2):
    """Represents binding between Neutron routers and their hosting devices."""
    router_id = sa.Column(sa.String(36),
                          sa.ForeignKey('routers.id', ondelete='CASCADE'),
                          primary_key=True)
    router = orm.relationship(l3_db.Router)
    # If 'auto_schedule' is True then router is automatically scheduled
    # if it lacks a hosting device or its hosting device fails.
    auto_schedule = sa.Column(sa.Boolean, default=True, nullable=False)
    # id of hosting device hosting this router, None/NULL if unscheduled.
    hosting_device_id = sa.Column(sa.String(36),
                                  sa.ForeignKey('hostingdevices.id',
                                                ondelete='SET NULL'))
    hosting_device = orm.relationship(hd_models.HostingDevice)"

REST API impact
---------------
The l3 routing service REST API remains intact. No new REST API is introduced.

Security impact
---------------
We create a virtual management network which is a Neutron provider Network that
CSR1kv VMs will have a VIF on as well as the config agent used to apply
configurations inside them. This configuration agent must be able to communicate
with the Neutron server for RPC. The regular Neutron management network is used
for that communication. The virtual management network and the Neutron
management network need not be the same and are preferably kept separate for
improved security isolation.

Security issues coupled to metadata service do not apply since this
implementation does not support metadata service via Neutron router.

Notifications impact
--------------------
None to existing. This l3 routing service plugin uses its own RPC for
interactions with Cisco configuration agents. These agents are defined in
https://blueprints.launchpad.net/neutron/+spec/cisco-config-agent.

Other end user impact
---------------------
End users creating Neutron routers will experience a somewhat longer delay from
the time the Neutron router create request has returned until the Neutron router
is operational (i.e., forwarding packets). This is due to the time needed to
spin up a CSR1kv VM.

Performance Impact
------------------
The Neutron and Nova services should not be significantly impacted by this
routing service plugin. The RPC is basically equivalent to the l3agent RPC
in terms of resulting traffic. Compared to the Linux namespace router
implementation a few more Neutron resources (ports, network, subnets) are
created by this implemenation when a Neutron router is created. We don't expect
these extra operations to amount to significantly increased load of the Neutron
server.

Other deployer impact
---------------------
We believe this will have no impact on community L3 router service plugin, not
in its current implemenation, nor when DVR is merged. The deployer will have to
specify that the Cisco routing service plugin be used instead of the community
one. A configuration agent must also be deployed.

Developer impact
----------------
None.

Implementation
==============
The below figure shows the new router service plugin and the components it
interacts with. The hosting device boxes are objects in the database and are
just to show that the router service plugin knows about devices it has placed a
Neutron router inside. The RouterHostingDeviceBindings box represents the
database table where this binding is stored.

Novaclient is used to interact with Nova to requeat that CSR1kv VMs are created
and deleted.

The plugin to config agent RPC is analogous to the l3 agent RPC. I.e., plugin
sends router-update/delete notifications to the config agent. The latter
periodically does get_routers() callback to hte plugin to fetch latest router
configurations for updated routers.

The router information sent to the config agent includes information about the
device that is hosting the Neutron router. The config agent can thereby remotely
configure that device via a virtual management network (which is a Neutron
provider Network).

All CSR1kv VMs that are spun up using Nova are owned by a special tenant. This
is also the case for the Neutron resources created for the CSR1kv VM instances.
These resources include: Neutron port on the virtual management network, Neutron
Networks with trunking capability and Neutron Ports on those networks.

A CSR1kv VM instance will only host one Neutron Router. As a consequence, a
CSR1kv VM instance will be allocated solely to the tenant owning the Neutron
Router it hosts. This limitation is imposed not for performance reasons but to
reduce the scope of the implementation. It allows us to simplify the scheduling.
VPN and/or Firewall service instances may be hosted in a CSR1kv VM instance if
those instances are bound to the Neutron Router hosted in the CSR1kv VM
instance.

::

                       ............
                       .   Nova   .
                       .   api    .
                       .  server  .
                       ............
                             ^
                             |
 ............................|.............     .............     ..............
 . Neutron                   |            .     . Some      .     . Nova       .
 . Server                    |            .     . Server    .     . Compute    .
 .                           |            .     .           .     .            .
 .                           |            .     .           .     .  +-------+ .
 .  +------------------------|---------+  .     .           .     .  |Hosting| .
 .  | Cisco Router         Nova        |  .     .           .  +---->|Device | .
 .  | Service Plugin       client      |  .     .           .  |  .  |Svc VM | .
 .  |                                  |  .     .           .  |  .  +-------+ .
 .  |                                  |  . RPC .  +------+ .  |  ..............
 .  |   +-------+   +-------------+    |  NOTIFIC. |Config|<---+
 .  |  +-------+|   |    Router   |    |<--------->|Agent | .    Device specific
 .  | +-------+||   |HostingDevice|    | CALLBACKS |      | .    protocol (e.g.,
 .  | |Hosting||+   |  Bindings   |    |  .     .  +------+ .    Netconf, REST)
 .  | |Device |+    +-------------+    |  .     .           .
 .  | +-------+                        |  .     .           .
 .  +----------------------------------+  .     .           .
 .                                        .     .           .
 .                                        .     .           .
 ..........................................     .............

Assignee(s)
-----------
Primary assignee: bob-melander

Other contributors: hareesh-puthalath, skandasw

Work Items
----------
* Implement new router DB mixin derived from extraroute DB mixin.
* Implement new mixin to manage CSR1kv VMs using Nova.
* Implement RPC notifications and callbacks for interaction with config agent.
* Implement new routing service plugin that uses above mixins and RPC.
* Create template configuration .ini file for the routing service plugin.
* Create bash scripts that exemplify how to setup the routing service plugin.
* Extend existing L3 routing unit test suite to cover new functionality.

Dependencies
============
Devstack must be updated to support this plugin.
The router service plugin uses Cisco config agent:
https://blueprints.launchpad.net/neutron/+spec/cisco-config-agent

Testing
=======
Unit tests will be added for all new functionality.
Tempest support for 3rd party CI is ongoing work and will be ready for Juno.
Functional and scenario tests for community L3 implementation will be useful
also for this implementation.

Documentation Impact
====================
Will require new documentation in Cisco sections.

References
==========

