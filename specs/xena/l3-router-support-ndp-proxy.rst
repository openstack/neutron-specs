..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
L3 router support NDP proxy
===========================

Launchpad Bug:
https://bugs.launchpad.net/neutron/+bug/1877301

The IPv6's NDP (Neighbor Discovery Protocol) is similar to IPv4's ARP (Address
Resolution Protocol) in functional, but the NDP works at L3. The NDP proxy [1]_
is also similar to ARP proxy [2]_. But, the NDP proxy only works on IPv6, it can
proxy specific IPv6 address's NA (Neighbor Advertisement) that to response the
NS (Neighbor Solicitation) which comes from the upstream router. The Linux
kernel already implement NDP proxy. With these functionalities, we can
implement some APIs so that users/tenants can advertise specific IPv6
addresses. It looks a little like IPv4's floating IP, but this solution doesn't
do any NAT action.


Problem Description
===================

As the IPv6 more and more popularize, we should provide a simple method to make
the IPv6 VMs more easily and flexibly connect to external network.

* In some use cases, such as a web site which has some DB services, MQ services
  and portal services etc, and they work on IPv6 VMs with same subnet. The
  administrator of the web site just would like to publish portal services to
  external. So the administrator needs some methods to just advertise a part of
  address to external.

* The current solution of publishing IPv6 address which the Neutron support,
  such as BGP (Border Gateway Protocol) [3]_, PD (Prefix delegation) [4]_, is
  more complex, they require do some complex configurations in upstream router.
  This maybe not suit some company as their bare metal hosted in thrid-party
  IDC, they don't have the privilege of the upstream router. So we should
  provide a solution with not depth cooperation of upstream router.


Proposed Change
===============

Overview
--------

Server Side
~~~~~~~~~~~

* Implement a service plugin named ``ndp_proxy`` to support the feature of this
  spec description. Administrator can enable the feature by append the plugin
  to ``service_plugin`` in Neutron configuration file.

* Implement an API extension, In the extension we should define a named
  ``ndp_proxy`` resource and its APIs.

* Add a new parameter ``enable_ndp_proxy`` to router. If it is set to True, the
  ndp proxy will be enabled in router namespace.

Agent Side
~~~~~~~~~~

Implement an extension of Neutron L3 agent to set relational rules for ndp
proxy. The extension behavior like below:

* If the router's ``enable_ndp_proxy`` field is true, the router's namespace
  should have a default ip6tables rule to drop all packets input from the
  router's external gateway device. By this way, we can eliminate affection of
  extra route and neighbor in upstream router. So, if a router's
  ``enable_ndp_proxy`` set as True, the Prefix Delegation [4]_ and BGP Dynamic
  Routing [3]_ solution will have no effect, we should explain this in user
  document.

  .. note:: If a router's ``enable_ndp_proxy`` set as false when the router
            have ndp proxies, these ndp proxes will have no effect, the Prefix
            Delegation [4]_ and BGP Dynamic Routing [3]_ solution will recover.
            In particular, the DVR router will ignore ``enable_ndp_proxy``.

* For each ndp proxy object, the extension should set a neighbor proxy entry
  at the router's external gateway device and set an ip6tables rule to permit
  the relational packets through. By this, the router only permits those IPv6
  addresses which enabled the ``ndp_proxy`` to communication with external
  network. This makes the ndp proxy's behavior is more similar to IPv4's
  floating ip.

Data Model Impact
-----------------

The following new tables are added as part of the ndp proxy feature.

For ndp proxy resource::

    CREATE TABLE ndp_proxies (
        id CHAR(36) NOT NULL PRI KEY,
        name VARCHAR(255),
        router_id VARCHAR(36) NOT NULL FOREIGN KEY,
        port_id VARCHAR(36) NOT NULL FOREIGN KEY,
        ip_address VARCHAR(64) NOT NULL,
        project_id VARCHAR(255),
        standard_attr_id bigint(20) NOT NULL,
    );

The ``router_id`` column will store the id of router which the ndp proxy belong
to. The ``port_id`` column will store the id of Neutron internal port which the
``ip_address`` locate in. The ``ip_address`` column will store the IPv6 address
which we need to proxy.

For router's ``enable_ndp_proxy`` parameter::

    CREATE TABLE router_ndp_proxy_state (
        router_id VARCHAR(36) NOT NULL FOREIGN KEY,
        enable_ndp_proxy BOOL NOT NULL,
    );

New Resource Extension
----------------------

New resource extension ``ndp_proxy`` will be added. It will define the
``ndp_proxy`` entry's CURD APIs.

For this new feature, a new service plugin will be introduced, and the
following methods will be added:

* 'create_ndp_proxy()'
* 'delete_ndp_proxy()'
* 'update_ndp_proxy()'
* 'get_ndp_proxy()'
* 'get_ndp_proxies()'

So the attributes map of new resource would be like:

.. code-block:: python

    RESOURCE_ATTRIBUTE_MAP = {
        'ndp_proxy': {
            'id': {'allow_post': False,
                   'allow_put': False,
                   'validate': {'type:uuid': None},
                   'is_visible': True,
                   'primary_key': True},
            'name': {'allow_post': True,
                     'allow_put': True,
                     'validate': {'type:string': 255},
                     'is_filter': True,
                     'is_sort_key': True,
                     'is_visible': True, 'default': ''},
            'project_id': {'allow_post': True,
                           'allow_put': False,
                           'required_by_policy': True,
                           'validate': {'type:uuid': None},
                           'is_visible': True},
            'router_id': {'allow_post': True,
                          'allow_put': False,
                          'validate': {'type:uuid': None},
                          'is_visible': True},
            'port_id': {'allow_post': True,
                        'allow_put': False,
                        'validate': {'type:uuid': None},
                        'is_visible': True},
            'ip_address': {'allow_post': True,
                           'allow_put': False,
                           'default': None,
                           'validate': {
                               'type:ip_address_or_none': None},
                           'is_visible': True}
            'description': {'allow_post': True,
                            'allow_put': True,
                            'default': '',
                            'validate': {'type:string': 1024},
                            'is_visible': True}
        }
    }

.. note:: The ``ip_address`` parameter is optional, if not set it when user
          send post request, the new service plugin will select a IPv6 address
          from the ``port_id`` represented port.

REST API Impact
---------------

Extend router API
~~~~~~~~~~~~~~~~~

The idea is to extend the ``router`` Rest API with a new extension
``router_ndp_proxy`` with the below defined attribute.

.. list-table:: Router extension

  * - Attribute Name
    - Type
    - CRUD
    - Default Value
    - Description
  * - enable_ndp_proxy
    - Boolean
    - CRU
    - False
    - Whether the router enable ndp proxy function.

The ``router`` extension definition would be expanded as :

.. code-block:: python

   RESOURCE_ATTRIBUTE_MAP = {
        'routers': {
            'enable_ndp_proxy': {
                'allow_post': True, 'allow_put': True,
                'convert_to': converters.convert_to_boolean_if_not_none,
                'is_visible': True,
                'is_filter': True,
            }
        }
   }

Default, only admin user can update ``enable_ndp_proxy`` parameter. And, a new
config option ``enable_ndp_proxy_by_default`` will be introduced. If it set as
``True``, the ``enable_ndp_proxy`` will be set as ``True`` default.

For example, GET a ``router``:

GET /v2.0/routers/<router-uuid>

::

    {
        "router": {
            "admin_state_up": true,
            "availability_zone_hints": [],
            "availability_zones": [
                "nova"
            ],
            "created_at": "2018-03-19T19:17:04Z",
            "description": "",
            "distributed": false,
            "enable_ndp_proxy": false,
            "external_gateway_info": {
                "enable_snat": true,
                "external_fixed_ips": [
                    {
                        "ip_address": "172.24.4.6",
                        "subnet_id": "b930d7f6-ceb7-40a0-8b81-a425dd994ccf"
                    },
                    {
                        "ip_address": "2001:db8::9",
                        "subnet_id": "0c56df5d-ace5-46c8-8f4c-45fa4e334d18"
                    }
                ],
                "network_id": "ae34051f-aa6c-4c75-abf5-50dc9ac99ef3"
            },
            "flavor_id": "f7b14d9a-b0dc-4fbe-bb14-a0f4970a69e0",
            "ha": false,
            "id": "f8a44de0-fc8e-45df-93c7-f79bf3b01c95",
            "name": "router1",
            "revision_number": 1,
            "routes": [
                {
                    "destination": "179.24.1.0/24",
                    "nexthop": "172.24.3.99"
                }
            ],
            "status": "ACTIVE",
            "updated_at": "2018-03-19T19:17:22Z",
            "project_id": "0bd18306d801447bb457a46252d82d13",
            "tenant_id": "0bd18306d801447bb457a46252d82d13",
            "service_type_id": null,
            "tags": ["tag1,tag2"],
            "conntrack_helpers": []
        }
    }

Set a router's ``enable_ndp_proxy`` parameter to True:

Post /v2.0/routers/<router-uuid>

Request body::

    {
        "router": {
            "enable_ndp_proxy": true
        }
    }


For the new resource ``ndp_proxy``, some new URLs will be introduced:

List NDP Proxies
~~~~~~~~~~~~~~~~

GET /v2.0/ndp_proxies

The response body::

   {
       "ndp_proxies": [
            {
                "project_id": "ad239fa5-ceb7-8b81-abf5-b930d7f6ceb7",
                "id": "ae34051f-aa6c-4c75-abf5-50dc9ac99ef3",
                "name": "test01",
                "port_id": "fa450s1f-aa6c-4c75-abf5-50dc9ac9df67",
                "ip_address": "2001:217::19",
                "created_at":"2020-05-21T11:33:21Z",
                "updated_at":"2020-06-18T08:31:48Z",
                "description": ""
            },
            {
                "project_id": "ad239fa5-ceb7-8b81-abf5-b930d7f6ceb7",
                "id": "915a14a6-867b-4af7-83d1-70efceb146f9",
                "name": "test02",
                "port_id": "4323401f-aa6c-4c75-abf5-50dc9ac99ef3",
                "ip_address": "2001:218::12"
                "created_at":"2020-05-21T11:33:21Z",
                "updated_at":"2020-06-18T08:31:48Z",
                "description": ""
            }
       ]
   }

Show NDP Proxy
~~~~~~~~~~~~~~

GET /v2.0/ndp_proxies/<ndp-proxy-id>

The response body::

   {
       "ndp_proxy": {
            "project_id": "ad239fa5-ceb7-8b81-abf5-b930d7f6ceb7",
            "id": "ae34051f-aa6c-4c75-abf5-50dc9ac99ef3",
            "name": "test01",
            "port_id": "b930d7f6-ceb7-40a0-8b81-a425dd994ccf"
            "ip_address": "2001:217::19",
            "created_at":"2020-05-21T11:33:21Z",
            "updated_at":"2020-06-18T08:31:48Z",
            "description": "Some descriptions"
        }
   }


Create NDP Proxy
~~~~~~~~~~~~~~~~

POST /v2.0/ndp_proxies

The request body::

   {
       "ndp_proxy": {
            "name": "test01",
            "router_id": "5823deb7-8b81-ceb7-40a0-b930d7f6ceb7",
            "port_id": "b930d7f6-ceb7-40a0-8b81-a425dd994ccf",
            "ip_address": "2001:217::19",
            "description": "Some descriptions"
        }
   }

There are some constraints here:

* The router's ``enable_ndp_proxy`` parameter must be set as True.

* The subnet that the ``ip_address`` allocated from must be added to the
  router.

* The network of the ``port_id`` represented port belong to must have same
  ``ipv6_address_scope`` [5]_ with the network of router's external gateway.

The response body::

   {
       "ndp_proxy": {
            "id": "ad239fv5-aa6c-4c75-abf5-50dc9ac99ef3",
            "name": "test01",
            "project_id": "ad239fa5-ceb7-8b81-abf5-b930d7f6ceb7",
            "router_id": "5823deb7-8b81-ceb7-40a0-b930d7f6ceb7",
            "port_id": "b930d7f6-ceb7-40a0-8b81-a425dd994ccf",
            "ip_address": "2001:217::19",
            "created_at":"2020-05-21T11:33:21Z",
            "updated_at":"2020-06-18T08:31:48Z",
            "description": "Some descriptions"
        }
   }


Update NDP Proxy
~~~~~~~~~~~~~~~~

PUT /v2.0/ndp_proxies/<ndp-proxy-id>

The request body::

   {
       "ndp_proxy": {
            "name": "test02",
            "description": "New descriptions"
        }
   }

The response body::

   {
       "ndp_proxy": {
            "id": "ad239fv5-aa6c-4c75-abf5-50dc9ac99ef3",
            "name": "test02",
            "project_id": "ad239fa5-ceb7-8b81-abf5-b930d7f6ceb7",
            "router_id": "5823deb7-8b81-ceb7-40a0-b930d7f6ceb7",
            "port_id": "b930d7f6-ceb7-40a0-8b81-a425dd994ccf"
            "ip_address": "2001:217::56",
            "created_at":"2020-05-21T11:33:21Z",
            "updated_at":"2020-06-18T08:31:48Z",
            "description": "New descriptions"
        }
   }

Delete NDP proxy
~~~~~~~~~~~~~~~~

DELETE /v2.0/ndp_proxies/<ndp-proxy-id>

This operation does not accept a request body and does not return a response
body.

Addtionally, if a router or port will be deleted, the ndp proxies which
relational with them will be delete cascade.

Effects on Existing Router APIs
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Before remove the subnet from a router, neutron should check whether has any
  ndp proxy related to the subnet (whether the ndp proxy's ``ip_address`` was
  allocated from the subnet). If has any ndp proxy related to the subnet, the
  subnet can't be removed.

L3 Agent Impact
---------------

Neutron needs to implement a new l3 agent extension to cooperate with the
server API to set relational rules (neigh proxy and ip6tables, distributed
router needs to set some extra route rules) in router's namespace.

We assume user has a below scenario, then respectively describe the
implementions of the feature about DVR router and Legacy router:

* A subnet of which cidr is ``2001::1:0/112``
* A VM belong to the subnet and it's IPv6 address is ``2001::1:8``
* A distributed/legacy router which set external gateway and connect to the
  subnet.

.. _legacy_router_impact:

Legacy router Impact
~~~~~~~~~~~~~~~~~~~~

Assume the router's external gateway device is ``qg-733bd76b-62``:

* If the router's parameter ``enable_ndp_proxy`` is true, the extension need to
  set the kernel parameter ``proxy_ndp`` as ``1`` in the router's
  qrouter-namespace, create a custom chain named ``neutron-l3-agent-NDP`` and
  set a default iptables rule in it to drop all packets input from the router's
  external gateway device. The executed commands like below::

    sysctl -w net.ipv6.conf.qg-733bd76b-62.proxy_ndp=1
    ip6tables -N neutron-l3-agent-NDP
    ip6tables -A neutron-l3-agent-NDP -i qg-733bd76b-62 -j DROP

.. note:: If the router have no external gateway, the parameter
          ``enable_ndp_proxy`` will be ignored.

* When add the IPv6 subnet to the router (the router's ``enable_ndp_proxy``
  already set as true), Before user advertise the subnet's address with ndp
  proxy, the subnet should drop all external traffic. So, the following cmd
  should be executed::

    ip6tables -I neutron-l3-agent-FORWARD -i qg-733bd76b-62 --destination 2001::1:0/112 -j neutron-l3-agent-NDP

  By this, we can eliminate the effect of extra route and neighbor entries in
  upstream router.

* For each ndp proxies, the extension should add a neigh proxy entry to the
  router external gateway device, and add ip6tables rule to permit the
  relational packets pass. The executed commands like below::

    ip -6 neigh add proxy 2001::1:8 dev qg-733bd76b-62
    ip6tables -I neutron-l3-agent-NDP -i qg-733bd76b-62 --destination 2001::1:8 -j ACCEPT

* When remove a ndp proxy, the extension should remove related neigh proxy
  entry from the router external gateway device, and remove the related
  ip6tables rule to re-forbid relational packets pass. The executed commands
  like below::

    ip -6 neigh del proxy 2001::1:8 dev qg-733bd76b-62
    ip6tables -D neutron-l3-agent-NDP -i qg-733bd76b-62 --destination 2001::1:8

* When router's parameter ``enable_ndp_proxy`` set to false, the extension
  needs to set the kernel parameter ``proxy_ndp`` as ``0`` in the router's
  qrouter-namespace namespace, delete the custom chain named
  ``neutron-l3-agent-NDP``. The executed commands like below::

    sysctl -w net.ipv6.conf.qg-733bd76b-62.proxy_ndp=0
    ip6tables -X neutron-l3-agent-NDP

HA router Impact
~~~~~~~~~~~~~~~~

The implementation of the feature for HA router is same as Legacy router,
except failover. When HA router's state has changed, the extension should
refresh the ndp proxy rules, because the ndp proxy rules may be lost after
multible failover.

.. note:: When HA router's state has changed, the ``keepalived`` will sends
          unsolicited neighbor advertisement automatically. So, we don't need
          to write extra code to do this. During failover, the traffic will be
          breaked, but it will recover soon if the failover accomplished.

DVR Router Impact
~~~~~~~~~~~~~~~~~

For DVR [6]_ router, it's a little different from legacy and HA router. We need
to consider two scenes: If the ``neutron-l3-agent`` in compute node set as
``dvr`` mode, the ``neutron-l3-agent`` will create a fip-namespace to process
north-south traffic, so we need to apply related rules in qrouter-namespace and
fip-namespace. If the ``neutron-l3-agent`` in compute node set as
``dvr_no_external`` mode, the north-south traffic will be processed by
snat-namespace in network node, so we need to apply related rules in
qrouter-namespace and snat-namespace.

For dvr mode
^^^^^^^^^^^^

Assume the fip-namespace's fg-dev port is ``fg-84920cf6-5e``; the port connect
fip-namespace to qrouter-namespace is ``fpr-ea902fe0-9`` and it's IPv6
address is ``fe80::8493:5bff:fe9b:8d93``; the port connect qrouter-namespace
to fip-namespace is ``rfp-ea902fe0-9`` and it's IPv6 address is
``fe80::a0a7:c5ff:fe2c:bade``. The topology like below::

                      +---------------+
                      |               |
                      |upstream router|
                      |               |
                      +-------+-------+
                              |
                +----------------------------+
                |             | compute node |
                |             |              |
                |     +-------+--------+     |
                |     | fg-84920cf6-5e |     |
                |     |                |     |
                |     |  fip-namespace |     |
                |     |                |     |
                |     | fpr-ea902fe0-9 |     |
                |     +--------+-------+     |
                |              |             |
                |              |             |
                |     +--------+--------+    |
                |     |  rfp-ea902fe0-9 |    |
                |     |                 |    |
                |     |qrouter-namespace|    |
                |     |                 |    |
                |     +-----------------+    |
                |                            |
                +----------------------------+

.. note:: We don't need to set ip6tables rules for dvr router. Because just by
          adding or removing the related route in fip-namespace (as description
          below) can be the switch to enable/disable the IPv6 traffic.

* Due to the current Neutron don't support dvr with IPv6, the qrouter-namespace
  has no default route about IPv6. We should add the default route firstly if
  the router set external gateway, the default route's next-hop shoule be the
  fpr-dev device's IPv6 address, The executed command like this (Executed in
  qrouter-namespace)::

    ip route add default via fe80::8493:5bff:fe9b:8d93 dev rfp-ea902fe0-9

* Because of the device which directly connects to upstream router is located
  in fip-namespace, the proxy entry should be set in fip-namespace. So, the
  below cmd should be executed in all fip-namespace::

    sysctl -w net.ipv6.conf.fg-84920cf6-5e.proxy_ndp=1

* For each ndp proxies the extension add a proxy entry to the fg-dev in
  fip-namespace, the proxy entry just add to one namespace which hosted in the
  node of the ndp proxy object's port belong to, and add a route in the
  namespace so that the packets of which destination is the ndp proxy's
  `ip_address` can be forwarded to qrouter-namespace. This cmds like below
  (Executed in fip-namespace)::

    ip -6 neigh add proxy 2001::1:8 dev fg-84920cf6-5e
    ip route add 2001::1:8 via fe80::a0a7:c5ff:fe2c:bade dev fpr-ea902fe0-9

* When remove a ndp proxy, the extension should remove related neigh proxy
  entry from the fg-dev, and remove the related route, This cmds like below
  (Executed in fip-namespace)::

    ip -6 neigh del proxy 2001::1:8 dev fg-84920cf6-5e
    ip route del 2001::1:8 via fe80::a0a7:c5ff:fe2c:bade dev fpr-ea902fe0-9

* Because setting of ip6tables rules is not required, so for the change of
  ``enable_ndp_proxy``, the agent extension needn't do any thing.

* If an instance was migrated and it's port was related to a ndp proxy entry.
  The extension should delete related rules in old host and create them in new
  host. Additionally, the extension should send a NA (Neighbour Advertisement)
  to fresh the upsteam router's neighbor entry so that the external traffic can
  forward to new host's fip-namespace immediately.


For dvr_no_external mode
^^^^^^^^^^^^^^^^^^^^^^^^

Assume the snat-namespace's qg-dev is ``qg-87059c6c-a9``, the sg-dev is
``sg-68bcdb7b-a2``; the qrouter-namespace's qr-dev is ``qr-50457f9b-98``. The
topology like below::

                    +---------------+
                    |               |
                    |upstream router|
                    |               |
                    +-------+-------+
                            |
                +-----------------------+
                |network    |           |
                | node      |           |
                |   +-------+------+    |
                |   |qg-87059c6c-a9|    |
                |   |              |    |
                |   |snat-namespace|    |
                |   |              |    |
                |   |sg-68bcdb7b-a2|    |
                |   +-------+------+    |
                |           |           |
                +-----------------------+
                            |
                +-----------------------+
                |compute    |           |
                | node      |           |
                |  +--------+--------+  |
                |  | qr-50457f9b-98  |  |
                |  |                 |  |
                |  |qrouter-namespace|  |
                |  |                 |  |
                |  +-----------------+  |
                |                       |
                +-----------------------+

For this mode, in qrouter-namespace we just need to add a default route, so
that the north-south traffic can be redirected to snat-namespace (The current
neutron code already completed this demand). For snat-namespace we just treat
is as legacy router's qrouter-namepsace, about it's rules process we can refer
to :ref:`legacy_router_impact`.

OVN backend impact
------------------

The ``ndp_proxy`` for OVN L3 backend is not covered by this proposal. If it was
proved to be feasible, we should implement the feature base on OVN backend in
the future. But for now, we will just implement this base on Neutron L3 agent.


Implementation
==============

Assignee(s)
-----------

yangjianfeng

Work Items
----------

1) API Implementation
2) DB Implementation
3) Reference implementation
4) Tests
5) Documentation


Testing
=======

Tempest Tests
-------------
The functions that the spec proposed need the external hardware router's
support (need to add a direct route entry at upstream router). This is
difficult for ``tempest`` to do this. So, we can skip the scenario tests
firstly.

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

API reference
-------------
Needs API reference documentation


References
==========

.. [1] https://tools.ietf.org/html/rfc4389
.. [2] https://tools.ietf.org/html/rfc1027
.. [3] https://docs.openstack.org/neutron/train/admin/config-bgp-dynamic-routing.html#ipv6
.. [4] https://specs.openstack.org/openstack/neutron-specs/specs/liberty/ipv6-prefix-delegation.html
.. [5] https://docs.openstack.org/neutron/latest/admin/config-address-scopes.html
.. [6] https://docs.openstack.org/neutron/latest/admin/deploy-ovs-ha-dvr.html
