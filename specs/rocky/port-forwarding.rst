..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================
Port Forwarding API
===================

https://blueprints.launchpad.net/neutron/+spec/port-forwarding

Port forwarding is a common feature in networking and more specifically in PaaS
and SaaS cloud systems which aim at reusing the same public IP for different
clients that use different VMs for their services.

This is especially relevant for deployments which lack a large number of public
IPs they can assign.

Common use case for this feature is a client requesting a specific service,
where the serving platform (PaaS, SaaS) allocates a VM to run the service and
then allocates a client port to access this service.
This means that various clients use the same public IP, but the TCP/UDP
destination port is used to distinguish between the end point VMs.

Example, a mapping for web servers:

* client1  172.24.4.2:4001 TCP  => maps to 10.0.0.2 port 80 TCP (VM1)
* client2  172.24.4.2:4002 TCP  => maps to 10.0.0.3 port 80 TCP (VM2)

This spec will focus on port forwarding based on Floating IPs. A future spec
will be submitted for port forwarding based on routers external gateway
interface.


Problem Description
===================

* In environments constrained with limited IPs, operators would like
  to reuse public IPs instead of assigning to each VM its own public
  IP (Floating IP).

* Docker supports a port-mapping feature and hence a big eco-system of
  automation orchestration and management plugins leverage it.
  We would like to make Neutron compatible for these tools and systems
  and provide a similar API [#foot1]_.


Proposed Change
===============
Introduce port forwarding API and implementation to Floating IPs.

The user can define various port forwarding rules on the Floating IPs
containing the internal/client port and the external/destination port,
connected with the VM they want to expose. And users would be allowed to create
port forwarding rules only on a "free" Floating IP, i.e. a Floating IP which is
not directly associated with a Fixed IP of a tenant's VM.

We will have four deployment variants:

a) Legacy Router: For the generic Router deployment, the port forwarding rules
would be installed in the Router namespace on the network node and forwarding
functions would be performed.

b) HA Router: For the HA Router deployment, the port forwarding rule will be
installed in both the ACTIVE and the BACKUP Router namespaces on the network
nodes they are located on.

c) DVR: As [#foot2]_ has been merged, we now have the ability to create
centralized Floating IPs in a DVR supported deployment.
This helps in mapping the Compute nodes where the destination VM is present
with the centralized FIP for Port forwarding. This mechanism will centralize
a floating IP not only when it is associated with a port bound to a host with
the ``dvr_no_external`` option enabled, but also when port forwarding
attributes are added to it.

d) DVR + HA: If the created router is not an HA router, then we can proceed
with option (c). While if the router is an HA router, then we will use the
centralized HA router to install the port forwarding rules.

In all deployment variants, the port forwarding entry NATs a specific
Floating IP:Port and protocol to a specific Neutron port (and a private IP that
is attached to this port). That means it will maintain a mapping like
"FIP:extport protocol" to "Neutron Port Fixed IP:intport protocol". So if a
Neutron Port contains multiple Fixed IPs, then it will be allowed to create
multiple port forwarding entries for a particular Neutron port, with different
external ports, such as:

* FIPX:EXTPORTX PROTOCOLA => Neutron PortQ Fixed IPA:INTPORTA PROTOCOLA (VM1)
* FIPX:EXTPORTY PROTOCOLA => Neutron PortQ Fixed IPB:INTPORTA PROTOCOLA (VM1)

However, the same Fixed IP:intport socket cannot be mapped with different
protocols.

If the Neutron port is deleted, the port forwarding entries that match this
port are also deleted. Same is applicable in case the Floating IP is deleted.


Data Model Impact
-----------------

The following new table is added as part of the port forwarding feature::

    CREATE TABLE port_forwarding (
        id CHAR(36) NOT NULL PRI KEY,
        floating_ip_id VARCHAR(36) NOT NULL,
        external_port INT NOT NULL,
        internal_neutron_port_id VARCHAR(36) NOT NULL FOREIGN KEY,
        protocol CHAR(4) NOT NULL,
        socket VARCHAR(20) NOT NULL,
        CONSTRAINT floating_ip_id_external_port_constraint UNIQUE (floating_ip_id, external_port),
        CONSTRAINT internal_neutron_port_id_socket_constraint UNIQUE (
            internal_neutron_port_id, socket)
    );

The ``socket`` column will store the string like 'Fixed IP:Port'.

.. note:: This table lacks ``project_id``, as the owner of this
          ``port_forwarding`` must be the owner of associated Floating IP. So
          there is a project_id check for preventing association of
          Floating IP to internal Neutron Port if their ``project_id`` are
          different. Also, allow the association that Floating IP/internal
          Neutron Port exists on a shared network for admin users in different
          project_id cases, such as FloatingIP from a shared public network
          created by a admin user and a Neutron Port from a particular
          internal tenant network created by the tenant user, then admin user
          want a association of them which have different project_ids. For
          general users, there is only the same project_id case.

Sub Resource Extension
----------------------

Neutron ``floatingips`` will be extended with a sub resource
``port_forwarding``, it will contain some fields to expose the
``port_forwarding`` assigned to the ``floatingip`` resource.

For this new feature, a new service plugin will be introduced, and the
following methods will be added:

* 'create_floatingip_port_forwarding()'
* 'delete_floatingip_port_forwarding()'
* 'get_floatingip_port_forwarding()'
* 'get_floatingip_port_forwardings()'

For update operation, we will extend the function in the future if possible.
But for now, we will just support create/delete/get functions.

So the attributes map of new sub resource would be like:

.. code-block:: python

    SUB_RESOURCE_ATTRIBUTE_MAP = {
        'port_forwarding': {
            'parent': {'collection_name': 'floatingips',
                        'member_name': 'floatingip'},
            'parameters': {
                'external_port': {'allow_post': True, 'allow_put': False,
                                     'convert_to':
                                         convert_validate_port_value,
                                     'is_visible': True},
                'internal_port': {'allow_post': True, 'allow_put': False,
                                     'convert_to':
                                         convert_validate_port_value,
                                     'is_visible': True},
                'internal_ip_address': {'allow_post': True,
                                            'allow_put': False,
                                            'validate': {
                                            'type:ip_address_or_none': None},
                                            'is_visible': True},
                'protocol': {'allow_post': True, 'allow_put': False,
                               'validate': {
                                  'type:values': constants.IPTABLES_PROTOCOL_MAP.keys()},
                               'is_visible': True,
                               'convert_to': converters.convert_to_protocol},
                'internal_port_id': {'allow_post': True,
                                        'allow_put': False,
                                        'validate': {'type:int':None},
                                        'is_visible': True},
            }
        }
    }

REST API Impact
---------------

The idea is to extend the Floating IP Rest API with a new extension
``floating_ip_port_forwarding`` with the below defined attributes.

.. list-table:: Floating IP extension

  * - Attribute Name
    - Type
    - CRUD
    - Default Value
    - Description
  * - port_forwardings
    - List
    - R
    - None
    - The associated 'port-forwarding' sub resource with the particular
      Floating IP resource.

The Floating IP extension definition would be expanded as :

.. code-block:: python

   RESOURCE_ATTRIBUTE_MAP = {
        'floatingips': {
            'port_forwardings': {'allow_post': False,
                                    'allow_put': False,
                                    'is_visible': True, 'default': None}
        }
   }

This new field will be exposed in the response during GET/POST/PUT requests of
Floating IP resource. That means users can not change the forwarding resources
through CRU FloatingIP, only can create the forwardings one by one with the new
port forwarding API which will be introduced below.

For example, GET a Floating IP:

GET /v2.0/floatingips/<floatingip-uuid>

::

   {
       "floatingip": {
            "floating_network_id": "376da547-b977-4cfe-9cba-275c80debf57",
            "router_id": "d23abc8d-2991-4a55-ba98-2aaea84cc72f",
            "fixed_ip_address": "",
            "floating_ip_address": "172.24.4.228",
            "project_id": "4969c491a3c74ee4af974e6d800c62de",
            "tenant_id": "4969c491a3c74ee4af974e6d800c62de",
            "status": "ACTIVE",
            "port_id": "",
            "id": "2f245a7b-796b-4f26-9cf9-9e82d248fda7",
            "port_forwardings": [
                {
                    "internal_ip_address": "10.0.0.3",
                    "protocol": "tcp",
                    "internal_port": "22",
                    "external_port": "7001"
                },
                {
                    "internal_ip_address": "192.168.4.32",
                    "protocol": "tcp",
                    "internal_port": "22",
                    "external_port": "7002"
                }
            ]
       }
   }

For the new sub resource 'port_forwarding', a new url will be introduced:

* /v2.0/floatingips/<floatingip-uuid>/port_forwardings

List Port Forwardings
~~~~~~~~~~~~~~~~~~~~~

GET /v2.0/floatingips/<floatingip-uuid>/port_forwardings

::

   {
       "port_forwardings": [
            {
                "id": "ae34051f-aa6c-4c75-abf5-50dc9ac99ef3",
                "external_port": "7003",
                "internal_port": "22",
                "internal_ip_address": "10.0.0.10",
                "protocol": "tcp",
                "internal_port_id": "b930d7f6-ceb7-40a0-8b81-a425dd994ccf"
            },
            {
                "id": "915a14a6-867b-4af7-83d1-70efceb146f9",
                "external_port": "7004",
                "internal_port": "22",
                "internal_ip_address": "10.0.0.11",
                "protocol": "tcp",
                "internal_port_id": "0c56df5d-ace5-46c8-8f4c-45fa4e334d18"
            }
       ]
   }

.. list-table:: Response Parameters
   :header-rows: 1

   * - Parameter
     - Style
     - Type
     - Description
   * - port_forwardings
     - plain
     - xsd:list
     - A list of *port_forwarding* objects

More parameters see :ref:`show_port_forwarding`

.. _show_port_forwarding:

Show Port Forwarding
~~~~~~~~~~~~~~~~~~~~

GET /v2.0/floatingips/<floatingip-uuid>/port_forwardings/<port-forwarding-id>

::

   {
       "port_forwarding": {
            "id": "ae34051f-aa6c-4c75-abf5-50dc9ac99ef3",
            "external_port": "7003",
            "internal_port": "22",
            "internal_ip_address": "10.0.0.10",
            "protocol": "tcp",
            "internal_port_id": "b930d7f6-ceb7-40a0-8b81-a425dd994ccf"
        }
   }

.. list-table:: Response Parameters
   :header-rows: 1

   * - Parameter
     - Style
     - Type
     - Description
   * - port_forwarding
     - plain
     - xsd:dict
     - A *port_forwarding* object
   * - id
     - plain
     - xsd:string
     - The ID of  *port_forwarding* object
   * - external_port
     - plain
     - xsd:string
     - The exposed external protocol port number
   * - internal_port
     - plain
     - xsd:string
     - The port forwarding mapped internal protocol port number
   * - internal_ip_address
     - plain
     - xsd:string
     - The IP Address from the fixed ips of a particular port
   * - protocol
     - plain
     - xsd:string
     - The traffic protocol type, such as TCP or UDP. Default value is 'TCP'.
   * - internal_port_id
     - plain
     - xsd:string
     - The Neutron internal Port ID.

Create Port Forwarding
~~~~~~~~~~~~~~~~~~~~~~

POST /v2.0/floatingips/<floatingip-uuid>/port_forwardings

.. list-table::Request Parameters
   :header-rows: 1

   * - Parameter
     - Style
     - Type
     - Description
   * - port_forwarding
     - plain
     - xsd:dict
     - A *port_forwarding* Object
   * - external_port (mandatory)
     - plain
     - xsd:string
     - The exposed external protocol port number
   * - internal_port (mandatory)
     - plain
     - xsd:string
     - The port forwarding mapped internal protocol port number
   * - protocol (optional, default = 'TCP')
     - plain
     - xsd:string
     - The traffic protocol type, such as TCP or UDP. Default value is 'TCP'.
   * - internal_port_id (mandatory)
     - plain
     - xsd:string
     - The Neutron internal Port ID.
   * - internal_ip_address (optional, default = The first fixed ip address of
       port)
     - plain
     - xsd:string
     - The IP Address from the fixed ips of a particular port

::

   {
       "port_forwarding": {
            "external_port": "7233",
            "internal_port": "22",
            "internal_port_id": "b930d7f6-ceb7-40a0-8b81-a425dd994ccf"
        }
   }

Response parameters

see :ref:`show_port_forwarding`

::

   {
       "port_forwarding": {
            "id": "f8a44de0-fc8e-45df-93c7-f79bf3b01c95",
            "external_port": "7233",
            "internal_port": "22",
            "internal_ip_address": "192.168.43.33",
            "protocol": "tcp",
            "internal_port_id": "b930d7f6-ceb7-40a0-8b81-a425dd994ccf"
        }
   }

Delete Port Forwarding
~~~~~~~~~~~~~~~~~~~~~~

DELETE /v2.0/floatingips/<floatingip-uuid>/port_forwardings/<port-forwarding-id>

This operation does not accept a request body and does not return a response
body.

Effects on Existing Floating IP APIs
------------------------------------

Slight adjustment to existing Floating IP APIs:

* Create a Floating IP just the same with current behavior.

* Associate a Neutron internal port with a Floating IP, if the requested url
  is the same as previous, the Floating IP will work as 1:1 DNAT like current
  behavior. If the request with the new url, that means the request needs a
  port-forwarding function towards the Floating IP.

* Get a Floating IP resource will be the same as before if the Floating IP
  resource had already associated with a neutron port for 1:1 DNAT. If a
  Floating IP resource contains more than 1 port-forwarding sub resource, it is
  better to show the ``port-forwardings`` summary in the Floating IP response
  body to distinguish which Floating IP resource is available for different
  requirements, such as 1:1 DNAT, port-forwarding.

Command Line Client Impact
--------------------------
Openstack Client would have additional options ``portforwarding`` for
Floating IP CLI, which would define the port-forwarding characteristics.

Security Impact
---------------
Port forwarding is similar in nature to centralized DNAT, so should not pose
additional security implications. But if users actually want to use Port
forwarding, they must make sure to allow the associated ingress Security Group
towards the internal ip which is used by the neutron port of their VMs.

Notifications Impact
--------------------
Depends on the implementation spec

Other End User Impact
---------------------
None

Performance Impact
------------------
Performance testing must be conducted to see what is the overhead
of enabling this feature, of course that if the feature is
disabled no performance impact should be noticed.

IPv6 Impact
-----------
IPv6 is not supported

Other Deployer Impact
---------------------
Deployer will be able to leverage port forwarding for a unique way to reach a
private VM/Container without wasting new public IPs

Developer Impact
----------------
Future SNAT distribution plans should take port forwarding into consideration.
Kuryr can leverage port forwarding for feature compatibility with Docker port
mapping.

Alternatives
------------
Users can use an external VM that provide this NAT capability or assign a new
Floating IP for each VM.


Implementation
==============

Assignee(s)
-----------

Primary assignees:
  reedip <reedip.banerjee@nectechnologies.in>

Other contributors:
  gal-sagie <gal.sagie@gmail.com>

  tian-mingming <tian.mingming@h3c.com>

  zhaobo <zhaobo6@huawei.com>

Work Items
----------

1) API Implementation
2) DB Implementation
3) Reference implementation
4) Tests
5) Documentation


Dependencies
============
None


Testing
=======

Tempest Tests
-------------
Need to add tempest tests

Functional Tests
----------------
Need to add functional tests

API Tests
---------
Need to add API tests

Fullstack Tests
---------------
Need to add Fullstack tests.


Documentation Impact
====================

User Documentation
------------------
Needs user documentation

Developer Documentation
-----------------------
Needs devref documentation


References
==========

.. [#foot1] https://docs.docker.com/engine/userguide/networking/default_network/binding/
.. [#foot2] https://review.openstack.org/#/c/485333/20
.. [#foot3] https://ask.openstack.org/en/question/75190/neutron-port-forwarding-qrouter-vms/
.. [#foot4] http://www.gossamer-threads.com/lists/openstack/dev/34307
.. [#foot5] http://openstack.10931.n7.nabble.com/Neutron-port-forwarding-for-router-td46639.html
.. [#foot6] http://openstack.10931.n7.nabble.com/Neutron-port-forwarding-from-gateway-to-internal-hosts-td32410.html
.. [#foot7] https://blueprints.launchpad.net/neutron/+spec/neutron-ovs-dvr
