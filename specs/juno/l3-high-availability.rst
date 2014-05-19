..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Neutron/L3 High Availability VRRP
=================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/l3-high-availability

The aim of this blueprint is to add High Availability Features on
virtual routers.

High availability features will be implemented as extensions and drivers.
A first driver on the agent side will be based on Keepalived.

A new scheduler will be also added in order to be able to spawn multiple
instances of a same router on many agents for the redundancy.

The DVR blueprint will leverage this proposal as a Service node specifically
for SNAT traffic. See the reference for the DVR BP at the end of this
specification


Problem description
===================

Currently we are able to spawn more than one l3 agent, and a l3 agent is able
to handle more than one external network, however each l3 agent is a SPOF.

If an l3 agent fails, all virtual routers of this agent will be lost,
and consequently all VMs connected to these virtual routers will be isolated.

Proposed change
===============

For the Neutron server side:

The idea of this blueprint is to schedule a virtual router to at least two l3
agents, but this limit could be increased by changing a parameter in the
neutron configuration file.

For the Neutron L3 agent side:

The current router interfaces management in the l3 agent will be abstracted in
order to introduce the possibility to add drivers for that purpose. As a first
implementation of a driver, an HA Keepalived driver will be added. All the IPs
will be converted to VIPs.

In order to hide the HA traffic from the tenant point of view a HA network will
be added and all the virtual router instances will be connected through a HA
port to this network.

Flows::

         +----+                          +----+        
         |    |                          |    |        
 +-------+ QG +------+           +-------+ QG +------+ 
 |       |    |      |           |       |    |      | 
 |       +-+--+      |           |       +-+--+      | 
 |     VIPs|         |           |         |VIPs     | 
 |         |      +--+-+      +--+-+       |         | 
 |         +      |    |      |    |       +         | 
 |  KEEPALIVED+---+ HA +------+ HA +----+KEEPALIVED  | 
 |         +      |    |      |    |       +         | 
 |         |      +--+-+      +--+-+       |         | 
 |     VIPs|         |           |         |VIPs     | 
 |       +-+--+      |           |       +-+--+      | 
 |       |    |      |           |       |    |      | 
 +-------+ QR +------+           +-------+ QR +------+ 
         |    |                          |    |        
         +----+                          +----+        


As a phase 2 of the keepalived driver implementation, the Keepalived driver
will start a conntrackd instance in order to not lose the established
connections when switching from the active to standby.

Alternatives
------------

The first driver is going to be based on Keepalived. We could use some
alternative drivers based on other protocols for ex: Common Address Redundancy
Protocol (CARP).

By default a config parameter will be added in order to specify whether the
virtual routers will be HA or not. In addition, an admin-only API is introduced
which will allow admins to migrate existing routers to HA mode.

Data model impact
-----------------

Two new columns will be added to the router_extra_attributes table in order to
specify whether the virtual router will be HA or not and to specify the virtual
router id.

+------------+-------+---------+---------+------------+---------------------+
|Attribute   |Type   |Access   |Default  |Validation/ |Description          |
|Name        |       |         |Value    |Conversion  |                     |
+============+=======+=========+=========+============+=====================+
|ha          |bool   |RW, admin|False    |N/A         |Set router as HA     |
|ha_vr_id    |int    |RW, admin|N/A      |N/A         |HA virtual router id |
+------------+-------+---------+---------+------------+---------------------+

The ha_vr_id will be limited to 255 due to VRRP protocol. This limit will have
to be removed when introducing a new driver without this limitation.

A new table will be introduced to specify the association between a router,
the agents and the HA ports that are going to be used for the HA
administrative traffic.

+------------+-------+---------+---------+------------+----+---------------+
|Attribute   |Type   |Access   |Default  |Validation/ |Key |Description    |
|Name        |       |         |Value    |Conversion  |    |               |
+============+=======+=========+=========+============+====+===============+
|port_id     |UUID   |RW, admin|N/A      |N/A         |PRI |HA port id     |
+------------+-------+---------+---------+------------+----+---------------+
|router_id   |UUID   |RW, admin|N/A      |N/A         |    |               |
+------------+-------+---------+---------+------------+----+---------------+
|l3_agent_id |UUID   |RW, admin|N/A      |N/A         |    |               |
+------------+-------+---------+---------+------------+----+---------------+
|priority    |int    |RW, admin|50       |N/A         |    |               |
+------------+-------+---------+---------+------------+----+---------------+
|state       |enum   |RW, admin|N/A      |N/A         |    |active/standby |
+------------+-------+---------+---------+------------+----+---------------+


REST API impact
---------------

router-create    Create a router for a given tenant.

::
    router-create --name another_router --ha=true

Admin can only set this attribute. The tenants need not be aware about
this attribute in the router table. So it is not visible to the tenant.

Request

::
    POST /v2.0/routers
    Accept: application/json

    {
    "router":{
    "name":"another_router",
    "admin_state_up":true,
    "ha":true}
    }


Response

::
    {
    "router":{
    "status":"ACTIVE",
    "external_gateway_info":null,
    "name":"another_router",
    "admin_state_up":true,
    "ha":true,
    "tenant_id":"6b96ff0cb17a4b859e1e575d221683d3",
    "id":"8604a0de-7f6b-409a-a47c-a1cc7bc77b2e"}
    }


router-show    Show information of a given router.

Request

::
    GET /v2.0/routers/a9254bdb-2613-4a13-ac4c-adc581fba50d
    Accept: application/json

Response

::
    {
    "routers":[{
    "status":"ACTIVE",
    "external_gateway_info":{
    "network_id":""
    },
    "name":"router1",
    "admin_state_up":true,
    "ha":true,
    "tenant_id":"33a40233088643acb66ff6eb0ebea679",
    "id":"a9254bdb-2613-4a13-ac4c-adc581fba50d"}]
    }

router-update    Create a router for a given tenant.

Admin can only update the HA mode of a router.

Admin only context:

::
    neutron router-update router1 --ha=True

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

There will be no network performance impact. Spawning a new virtual router may
be a bit longer due to the delay of starting the Keepalived/Conntrackd
processes.

Other deployer impact
---------------------

Since this implementation relies on Keepalived, Keepalived will have to be
deployed on each l3 node. The required version of Keepalived is the
version 1.2.0 in order to have the IPV6 support.

In addition, conntrackd will be required to be run on each node.

There is no plan to migrate automatically the original virtual routers to
the HA virtual routers when updating a previous Openstack installation.
So after a migration and with the l3_ha configuration parameter set to "True",
the new routers created will be HA while the older ones will be unchanged.
Cloud admins can migrate existing virtual routers to be HA routers by using
the new API. This API is not exposed to tenants.

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Sylvain Afchain <sylvain-afchain>

Other contributors:
  Assaf Muller <amuller>

Work Items
----------

1. HA L3 Extension, DB bases
2. HA L3 Scheduler
3. Keepalived manager
4. L3 agent driver abstraction introduction, Keepalived driver
5. Conntrackd support


Dependencies
============

None


Testing
=======

The code will be covered by unit tests.
When multi-nodes test will be available, tempest test will be introduced.

A document explaining how to test all the patches during the review
process will be updated here :

https://docs.google.com/document/d/1P2OnlKAGMeSZTbGENNAKOse6B2TRXJ8keUMVvtUCUSM


Documentation Impact
====================

Document deployer impacts.


References
==========

https://review.openstack.org/#/q/topic:bp/l3-high-availability,n,z
https://git.openstack.org/cgit/openstack/neutron-specs/tree/specs/juno/neutron-ovs-dvr.rst
https://wiki.openstack.org/wiki/Neutron/L3_High_Availability_VRRP
