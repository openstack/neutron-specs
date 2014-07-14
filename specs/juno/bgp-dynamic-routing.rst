..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================
Dynamic Routing for External Networks
=====================================

`Blueprint Link
<https://blueprints.launchpad.net/neutron/+spec/bgp-dynamic-routing>`_

The goal of this blueprint is to add a dynamic routing capability to OpenStack
deployment. This feature would allow Neutron to exchange routes with an external
router using standard routing protocols.

This document aims to define a foundation supporting different
route discovery/advertise protocols.

Problem description
===================
This is a new feature and this section will describe use cases where dynamic
routing may be useful.


**Expose External Networks Dynamically**

Allow Neutron to dynamically announce to and/or from external uplink
routers.

An OpenStack cloud would run a routing protocol (for example, BGP) against at
least one router in each uplink network provider. By announcing external network
hosting floating IP prefixes to those peers, the Neutron network would be
reachable by the rest of the internet via both paths. If the link to an uplink
provider broke, the failure information would propagate to routers further up
the stream, keeping the cloud reachable through the remaining healthy link.
Likewise, in such a case, Neutron would eliminate the routes learned through the
faulty link from its forwarding table, redirecting all cloud-originated traffic
through the healthy link.


**Sample Topology**

.. nwdiag::

  nwdiag {

    network external {
      gateway_router[color = red];
      tenant_router1;
      tenant_router2;
    }

    group {
      color = lightgreen;
      label = "OpenStack Deployment";
      gateway_router;
      tenant_router1;
      tenant_router2;
    }

    network isp_net1 {
      gateway_router;
      uplink_router1;
    }

    network isp_net2 {
      gateway_router;
      uplink_router2;
    }

  }

Please note that in this diagram, the red gateway router is actually a linux
machine with a software BGP speaker and a software router. This blueprint
proposes to configure this BGP speaker through the Neutron API (see below).

In this scenario each external network's address range will be advertised as a
route. Individually allocated floating IP addresses will not be exported as host
routes to reduce the workload of the upstream router(s).

This also would be a valid scenario for an internal cloud, where the uplink
router is actually one's provider router and the routes are advertised using
iBGP.

**Learn Uplink Routes Dynamically**

With a dynamic routing system, having tenant routers connected directly to
uplink routers, will let you discover dynamically uplink routes.


.. nwdiag::

  nwdiag {

    uplink_router1;
    uplink_router2;

    network external {
      uplink_router1;
      uplink_router2;
      tenant_router1;
      tenant_router2;
    }

    group {
      color = lightgreen;
      label = "OpenStack Deployment";
      tenant_router1;
      tenant_router2;
    }
  }


**Routed Model for Floating IPs on External Network**

This use case will allow an external network with a large public IP space to
possibly span more than one L2 network. This could allow for improved scale and
isolation between AZs while maintaining the appearance of a large external
network where floating IPs and networks can float freely.

This use case may also include announcing public networks behind Neutron routers
to an upstream router. It does not require learning routes from the upstream
router.

**Sample Topology**

.. nwdiag::

  nwdiag {
    network external {
      tor_switch1;
      tor_switch2;
    }

    tor_switch1 -- l2_network1;
    tor_switch2 -- l2_network2;

    group {
      color = lightgreen;
      tor_switch1;
      l2_network1;
    }

    group {
      color = yellow;
      tor_switch2;
      l2_network2;
    }
  }

This use case is covered in detail in `Pluggable External Net Blueprint
<https://blueprints.launchpad.net/neutron/+spec/pluggable-ext-net>`_

The topology of this use case can be seen as a generalization of the previous
one, with a multi-homed OpenStack installation and leverage the fact that a
floating IP can be seen as a /32 network.

Proposed change
===============


Overview
--------

A new system that dynamically advertises and discovers routes from other peers
outside the OpenStack deployment is proposed. From the Neutron API, cloud
administrator should be able to define these peers as well as the way to
interact with them.

Peer Configuration
++++++++++++++++++
A system that supports dynamic routing must be able to both advertise its own
routes to its peers as well as discover peers’ routes. It order to achieve this
the system must know the list of its peers and be able to trust the information
that it receives from them.


Route Advertisement
+++++++++++++++++++

When enabled, the system will automatically advertise all external networks to
configured dynamic routing peers. Routes added manually by administrator will
have an option to be advertised to external dynamic routing peers.


Route Discovery
+++++++++++++++

Routes advertised by remote dynamic routing peers will be added to the routing
tables of the neutron routers. These are the routes into the namespaces
qrouter-<route uuid> in the case of the reference L3 implementation. Since there
is already an RPC call in Neutron to advertise new routes (when user requests an
extra route), there will be no need to change anything in the reference L3
agent. The route information will show that it was installed dynamically and
will contain the ID of the dynamic routing peer that has advertised it.

The dynamically installed routes will also contain destination virtual port
information and relative weight. This additional configuration is useful when
the system has multiple uplinks to the same destination network. In such
scenario there may be 2 or more identical routes that may be differentiated by
the destination port and relative weight fields. The actual L3 agent or 3rd
party router implementation will be able to select the correct port to forward
packets to.


Dynamic Routing System
++++++++++++++++++++++

In default implementation a new system will be used to manage dynamic routing
information at the edge of OpenStack deployment. Dynamic peering will not be
performed by each Neutron router due to scaling concerns, as such approach may
create a high number of peering relationships.

This system will allow different implementations (develop your own BGP speaker
implementation, for instance) to fit into third party requirements.


IPv6 Considerations
+++++++++++++++++++

The implementation must be able to exchange IPv4 and IPv6 routes.


Solution Proposed
-----------------

This proposal, Dynamic Routing, is intended to abstract the common dynamic
routing protocols such as BGP and OSPF. Its first driver is based on an agent
(``dr_agent``) implementing BGP.

There is a comparison of different BGP speakers in the wiki's `BGP Comparison`_
table.

The agent will communicate to the uplink through the chosen protocol and to
neutron using RPC calls:


.. blockdiag::

  blockdiag dr_agent {
    uplinks <-> dr_agent <-> neutron
  }

That means the ``dr_agent`` will bind an interface to the management network and
another one to the *outer* network. This *outer* network may not be the same
network than the *external* data path network. The ports to bind shall be
informed in the ``dr_agent``.ini configuration file.

``dr_agent`` does not know anything about the networking topology in use, just
will advertise and receive routes as well as configure its remote peers
according to Neutron requests.

``dr_agent`` should be able to run in multiple instances, providing HA in the
functionality of the agent if you configure more than one agent using the same
peers.

.. nwdiag::

  nwdiag {
    network isp_network1 {
      dr_agent1;
      dr_agent2;
      peer1;
    }

    network external_network {
      dr_agent1;
      dr_agent2;
    }

    group {
      label = "Openstack deployment";
      color = lightgreen;
      dr_agent1;
      dr_agent2;
    }

    network isp_network2 {
      dr_agent1;
      dr_agent2;
      peer2;
    }
  }

Cloud administrator also can configure different agents using different peers,
to have the flexibility to expose different external networks to different paths
(useful if you have a multi-homed OpenStack deployment):

.. nwdiag::

  nwdiag {
    network isp_network1 {
      dr_agent1;
      peer1;
    }

    network external_network1 {
      dr_agent1;
    }

    group {
      label = "Openstack deployment";
      color = lightgreen;
      dr_agent1;
      dr_agent2;
    }

    network external_network2 {
      dr_agent2;
    }

    network isp_network2 {
      dr_agent2;
      peer2;
    }
  }

So, through the Neutron API, cloud administrator  should be able to define the
peer connections for each agent, as well as the policies to discover and
advertise routes from those peers, according to his needs. RoutingInstance
object will be the entity that will group all these configurations.

So the cloud administrator will be able to execute the following actions:

1. Create a peer connection.
2. Associate a peer connection to a ``dr_agent``.
3. Create a Routing Instance. This entity will establish the discovery/advertise
   options and associate external networks.
4. Associate a Routing Instance to networks.
5. Associate a Routing Instance to a ``dr_agent``.


Considerations
--------------

Have more than one peer providing routes and in the same time let the user add
manually routes could end up in routing conflicts at the neutron routers. It is
out of the scope of this blueprint provide a decision algorithm or offer a
policy interface to solve this conflicts.


Alternatives
------------

Dynamic routing may be implemented using protocols other than BGP. This document
aims to create the framework to make it easier to add more dynamic routing
protocols in the future.

Multi-homed clouds can be handled using classic networking infrastructure,
configuring manually the vendor router with BGP outside the OpenStack
deployment.


Data model impact
-----------------

This document proposes modifying data objects and schema in the following way.
For a quick glance of the Data Object Model, check out this etherpad_.

Data Object Changes
+++++++++++++++++++

Three new data model classes will be added: ``db.l3_db.RoutingPeer``,
``db.l3_db.RoutingInstance`` and ``db.l3_db.AdvertiseRoute``.
``db.models_v2.Route`` will be extended.

New ``db.l3_db.RoutingPeer`` class will contain the following attributes:

* ``id``: UUID
* ``peer``: String
* ``protocol``: String
* ``config``: String (json with remote_as, password, weight, and so on)

(in BGP case, local-as will be configured in ``dr_agent.ini`` config file.)

New ``db.l3_db.RoutingInstance`` class will contain the following attributes:

* ``id``: UUID
* ``networks``: List of network ``db.models_v2.Network`` resources
* ``agents``: List of ``db.agents_db.Agent`` resources
* ``advertised_routes``: List of ``db.l3_db.AdvertiseRoute`` resources
* ``ipv4``: String
* ``ipv6``: String
* ``discovery_mode``: String (available values will be: Network | None)
* ``advertise_mode``: String (available values will be: Network | None)

``advertise_mode`` and ``discovery_mode`` are defined as String and not Boolean to
allow more fine grained options in the future without modify the database

``ipv4`` and ``ipv6`` will be the values inserted as ``next_hop`` in the
advertised routes. So these should be the IP addresses of the gateway that
connects to the ISP.

In current Neutron implementation the route data object is defined in
``db.models_v2.Route`` class. The following attributes will be added to this
class:

* ``source``, String
* ``weight``, Integer
* ``type``, String
* ``dynamic``, Boolean
* ``origin_type``, String
* ``origin_id``, UUID

Another data object called ``db.l3_db.AdvertiseRoute`` will be created with the
following attributes:

* ``id``: UUID
* ``nexthop``, String
* ``destination``, String
* ``source``, String


Schema changes
++++++++++++++

A new resource type will be defined for Peer configuration. It will be called
``routingpeer`` and will contain the following attributes:

+---------+---------+-----+-----+---------+----------------+-------------------+
|Attribute|Type     |Req  |CRUD |Default  |Validation      |Notes              |
|         |         |     |     |Value    |Constraints     |                   |
+=========+=========+=====+=====+=========+================+===================+
|id       |uuid-str |n/a  |R    |generated|n/a             |Unique identifier  |
|         |         |     |     |         |                |for peer connection|
|         |         |     |     |         |                |configuration      |
+---------+---------+-----+-----+---------+----------------+-------------------+
|peer     |String   |Y    |CRU  |n/a      |n/a             |value to identify  |
|         |         |     |     |         |                | the peer          |
|         |         |     |     |         |                |                   |
|         |         |     |     |         |                |                   |
+---------+---------+-----+-----+---------+----------------+-------------------+
|protocol |String   |Y    |CRU  |n/a      |n/a             |Protocol to connect|
|         |         |     |     |         |                |to peer            |
|         |         |     |     |         |                |                   |
|         |         |     |     |         |                |                   |
+---------+---------+-----+-----+---------+----------------+-------------------+
|config   |string   |Y    |CRU  |n/a      |n/a             |json the needed in\|
|         |         |     |     |         |                |fo to connecto to  |
|         |         |     |     |         |                |the peer           |
|         |         |     |     |         |                |                   |
+---------+---------+-----+-----+---------+----------------+-------------------+

Between ``routingpeer`` and ``agents`` there is an n-to-n relationship
managed by the table ``peeragentbindings``

+---------+---------+-----+-----+---------+----------------+-------------------+
|Attribute|Type     |Req  |CRUD |Default  |Validation      |Notes              |
|         |         |     |     |Value    |Constraints     |                   |
+=========+=========+=====+=====+=========+================+===================+
|id       |uuid-str |n/a  |R    |generated|n/a             |Unique identifier  |
|         |         |     |     |         |                |for binding        |
+---------+---------+-----+-----+---------+----------------+-------------------+
|routingp\|uuid-str |Y    |CR   |generated|n/a             |Unique identifier  |
|eer_id   |         |     |     |         |                |for routing peer   |
|         |         |     |     |         |                |id                 |
+---------+---------+-----+-----+---------+----------------+-------------------+
|agent_id |uuid-str |Y    |CR   |generated|n/a             |Unique identifier  |
|         |         |     |     |         |                |for agent id       |
+---------+---------+-----+-----+---------+----------------+-------------------+

A new resource will be defined to describe how to forward and import traffic
from external Neutron networks to upstream provider routers. It will be called
``routinginstance`` and contain the following attributes:

+--------------+-------------+-----+-----+---------+-----------+---------------+
|Attribute     |Type         |Req  |CRUD |Default  |Validation |Notes          |
|              |             |     |     |Value    |Constraints|               |
+==============+=============+=====+=====+=========+===========+===============+
|id            |uuid-str     |n/a  |R    |generated|n/a        |Unique         |
|              |             |     |     |         |           |identifier for |
|              |             |     |     |         |           |external       |
|              |             |     |     |         |           |network gateway|
|              |             |     |     |         |           |configuration  |
+--------------+-------------+-----+-----+---------+-----------+---------------+
|ipv4          |str          |n/a  |CRUD |n/a      |it should \|IPv4 that will |
|              |             |     |     |         |be a valid |be used as 'ne\|
|              |             |     |     |         |ipv4 addre\|xthop' in IPv4 |
|              |             |     |     |         |ss         |networks.      |
+--------------+-------------+-----+-----+---------+-----------+---------------+
|ipv6          |str          |n/a  |CRUD |n/a      |it should \|IPv6 that will |
|              |             |     |     |         |be a valid |be used as 'ne\|
|              |             |     |     |         |ipv6 addre\|xthop' in IPv6 |
|              |             |     |     |         |ss         |networks.      |
+--------------+-------------+-----+-----+---------+-----------+---------------+
|advertise_mod\|string       |Y    |CRU  |network  |Valid      |whether to adv\|
|e             |             |     |     |         |mode       |ertise the ass\|
|              |             |     |     |         |           |ociated networ\|
|              |             |     |     |         |           |ks or not      |
+--------------+-------------+-----+-----+---------+-----------+---------------+
|discovery_mode|string       |Y    |RU   |network  |           |whether to pro\|
|              |             |     |     |         |           |pagate the dis\|
|              |             |     |     |         |           |covered routes |
|              |             |     |     |         |           |to the network\|
|              |             |     |     |         |           |'s router      |
+--------------+-------------+-----+-----+---------+-----------+---------------+

A ``routinginstance`` can manage several ``dr_agents``, but a ``dr_agent`` only
can be managed by one ``routinginstance``. The normal approach would be add a
``routinginstance_id`` in table ``agents``. But we don't want to touch this
entity because it is used by the rest of the agents. So we are going to create a
new ``routinginstanceagentbindings`` table with a constraint for ``agent_id``
column must be unique.

+---------+---------+-----+-----+---------+----------------+-------------------+
|Attribute|Type     |Req  |CRUD |Default  |Validation      |Notes              |
|         |         |     |     |Value    |Constraints     |                   |
+=========+=========+=====+=====+=========+================+===================+
|id       |uuid-str |n/a  |R    |generated|n/a             |Unique identifier  |
|         |         |     |     |         |                |for binding        |
+---------+---------+-----+-----+---------+----------------+-------------------+
|routingi\|uuid-str |Y    |CR   |generated|n/a             |Unique identifier  |
|nstance\\|         |     |     |         |                |for routing instan\|
|_id      |         |     |     |         |                |ce                 |
+---------+---------+-----+-----+---------+----------------+-------------------+
|agent_id |uuid-str |Y    |CR   |generated|must be unique  |Unique identifier  |
|         |         |     |     |         |                |for agent id       |
+---------+---------+-----+-----+---------+----------------+-------------------+

There is also a n-to-n relationship between the ``routinginstance`` and the
``networks`` table. ``routinginstancenetworkbindings`` is needed:

+---------+---------+-----+-----+---------+----------------+-------------------+
|Attribute|Type     |Req  |CRUD |Default  |Validation      |Notes              |
|         |         |     |     |Value    |Constraints     |                   |
+=========+=========+=====+=====+=========+================+===================+
|id       |uuid-str |n/a  |R    |generated|n/a             |Unique identifier  |
|         |         |     |     |         |                |for binding        |
+---------+---------+-----+-----+---------+----------------+-------------------+
|routingi\|uuid-str |Y    |CR   |generated|n/a             |Unique identifier  |
|nstance\\|         |     |     |         |                |for routing instan\|
|_id      |         |     |     |         |                |ce                 |
+---------+---------+-----+-----+---------+----------------+-------------------+
|network\\|uuid-str |Y    |CR   |generated|n/a             |Unique identifier  |
|_id      |         |     |     |         |                |for network id     |
+---------+---------+-----+-----+---------+----------------+-------------------+

A new resource type will be defined for storing route configuration. A
collection of these objects may be used to define a routing table in other
resources. This resource will be called ``advertiseroute`` and contain the following
attributes:

+------------+--------+-----+-----+---------+-----------+----------------------+
|Attribute   |Type    |Req  |CRUD |Default  |Validation |Notes                 |
|            |        |     |     |Value    |Constraints|                      |
+============+========+=====+=====+=========+===========+======================+
|id          |uuid-str|n/a  |R    |generated|n/a        |Unique Identifier for |
|            |        |     |     |         |           |route configuration   |
+------------+--------+-----+-----+---------+-----------+----------------------+
|routinginst\|uuid-str|Y    |CR   |generated|n/a        |Unique identifier     |
|ance_id     |        |     |     |         |           |for routing instan\   |
|            |        |     |     |         |           |ce                    |
+------------+--------+-----+-----+---------+-----------+----------------------+
|source      |CIDR    |N    |CRU  |0.0.0.0/0|n/a        |Value to compare with |
|            |        |     |     |         |           |the source IP address |
|            |        |     |     |         |           |of the flow being     |
|            |        |     |     |         |           |forwarded             |
+------------+--------+-----+-----+---------+-----------+----------------------+
|destination |CIDR    |Y    |CRU  |0.0.0.0/0|n/a        |Value to compare with |
|            |        |     |     |         |           |the destination IP    |
|            |        |     |     |         |           |address of the flow   |
|            |        |     |     |         |           |being forwarded       |
+------------+--------+-----+-----+---------+-----------+----------------------+
|nexthop     |IP      |Y    |CRUD |None     |Routable IP|IP address of the next|
|            |address |     |     |         |address    |hop                   |
+------------+--------+-----+-----+---------+-----------+----------------------+


REST API impact
---------------


Peer Connection Settings
++++++++++++++++++++++++

The following Neutron API changes will allow an administrator to configure peer
connections.


List Peer Connections, Show Peer Connection
*******************************************

+-----+------------------------------------------+-----------------------------+
|Verb |URI                                       |Description                  |
+=====+==========================================+=============================+
|GET  |/routingpeers                             |List the peer connections    |
|     |                                          |                             |
|     |                                          |                             |
|     |                                          |                             |
+-----+------------------------------------------+-----------------------------+
|GET  |/routingpeers/{routingpeer_id}            |Show the configuration for   |
|     |                                          |peer connection with the     |
|     |                                          |specified id                 |
+-----+------------------------------------------+-----------------------------+

Response Codes:

* 200: Normal
* 401: Unauthorized
* 403: Forbidden (for example, when non-administrator tries to access
  configuration)
* 404: Not Found (for example, when the peer connection with the specified ID
  doesn’t exist)

On success a response will contain one or more peer connection objects (JSON
format used in the example). On failure, the response will contain empty body.
Note that “password” field is not included in the output. The API may only be
used to create/update/delete password field, not to display it. Password will be
provided to the plugin code in the clear text. ::

 {
  [{
     “routingpeer”: {
        “id”: “88d2dbf0-35e5-11e3-aa6e-0800200c9a66”,
        “peer”: “10.32.0.2”,
        "protocol": "bgp",
        "configuration" : {
          “local_as”: “65000”,
          “remote_as”: “65001”,
          “weight”: “1”,
        }
     }
   },
   {
     “routingpeer”: {
       “id”: “40edaac2-881c-457b-9b4f-05bcd8510d28”,
       “peer”: “10.32.0.3”,
       "protocol": "bgp",
       "configuration": {
         “local_as”: “65000”,
         “remote_as”: “65002”,
         “weight”: “2”
       }
     }
   }]
 }


Create Peer Connection
**********************

+-----+-------------------------------------+------------------------------------+
|Verb |Uri                                  |Description                         |
+=====+=====================================+====================================+
|POST |/routingpeers                        |Create a new peer connection        |
|     |                                     |configuration                       |
+-----+-------------------------------------+------------------------------------+

Response Codes:

* 201: Normal
* 400: Bad Request (for example, invalid request format)
* 401: Unauthorized
* 403: Forbidden (for example, when non-administrator user tries to create BGP
  configuration)
* 404: Not Found (for example, when the router with the specified id does not
  exist)

This operation requires a request body and returns a response body. Both contain
BGP configuration inside object. A JSON-encoded example is provided.

Request: ::

 {
  “routingpeer”:
  {
    “peer”: “10.32.0.17”,
    "protocol": "bgp",
    "configuration": {
        “password”: “secret”,
        “local_as”: “65000”,
        “remote_as”: “65005”,
    }
  }
 }

Response: ::

 {
  “routingpeer”:
  {
    “id”: “c1bcd3d8-2f02-4d05-8283-ff87ae962223”,
    “peer”: “10.32.0.17”,
    "protocol": "bgp",
    "configuration": {
        “local_as”: “65000”,
        “remote_as”: “65005”,
        “weight”: “2147483647”,
    }
  }
 }


Update Peer Connection
**********************

+-----+------------------------------------------+-----------------------------+
|Verb |Uri                                       |Description                  |
+=====+==========================================+=============================+
|PUT  |/routingpeers/{id}                        |Update peer connection       |
|     |                                          |configuration                |
+-----+------------------------------------------+-----------------------------+

Response Codes:

* 200: Normal
* 400: Bad Request (for example, invalid request format)
* 401: Unauthorized
* 403: Forbidden (for example, when non-administrator tries to update BGP
  configuration)
* 404: Not Found (for example, when the BGP connection with the specified id
  does not exist)

This operation requires a request body and returns a response body. Both contain
RoutingPeer object. A JSON-encoded example is provided.

Request: ::

 {
  “routingpeer”:
  {
    “id”: “c1bcd3d8-2f02-4d05-8283-ff87ae962223”,
    "configuration": {
        “remote-as”: “65006”,
    }
  }
 }

Response: ::

 {
  “routingpeer”:
  {
    “id”: “c1bcd3d8-2f02-4d05-8283-ff87ae962223”,
    “peer”: “10.32.0.17”,
    "protocol" : "bgp",
    "configuration": {
        “local_as”: “65000”,
        “remote-as”: “65006”,
        “weight”: “2147483647”
    }
  }
 }


Delete Peer Connection
**********************

+-------+------------------------------------------+---------------------------+
|Verb   |Uri                                       |Description                |
+=======+==========================================+===========================+
|DELETE |/routingpeers/{id}                        |Delete peer connection     |
|       |                                          |configuration              |
+-------+------------------------------------------+---------------------------+

Response Codes:

* 204: Normal
* 401: Unauthorized
* 404: Not Found (for example, if the peer connection with the specified id does
  not exist)

This operation does not require request body and does not provide response body.


Routing Instance Settings
+++++++++++++++++++++++++

Routing instance is the configuration entity that makes relationships between
the networks to advertise, routes to discover and configured peers. It can be
seen as the entity that provides VRF_ in Neutron.

List Routing Instances, Show Routing Instance config
****************************************************

+-----+------------------------------------------+-----------------------------+
|Verb |URI                                       |Description                  |
+=====+==========================================+=============================+
|GET  |/routinginstances                         |List the routing instances   |
|     |                                          |                             |
|     |                                          |                             |
|     |                                          |                             |
+-----+------------------------------------------+-----------------------------+
|GET  |/routinginstances/{routinginstance_id}    |Show the configuration for   |
|     |                                          |the routing instance with the|
|     |                                          |specified id                 |
+-----+------------------------------------------+-----------------------------+

Response Codes:

* 200: Normal
* 401: Unauthorized
* 403: Forbidden (for example, when non-administrator tries to access
  configuration)
* 404: Not Found (for example, when the peer connection with the specified ID
  doesn’t exist)

Response: ::

  {[
    { "routinginstance":
        {
          “id”: “r1bcd3d8-2f02-4d05-8283-ff87ae962223”,
          "advertise_mode": "network",
          "discovery_mode": "none",
          "ipv4": "119.12.32.2",
          "ipv6": "",
          "agents": [
            “a1bcd3d8-2f02-4d05-8283-ff87ae962223”,
            “a2bcd3d8-2f02-4d05-8283-ff87ae962223”
          ],
          "networks": [
            “n1bcd3d8-2f02-4d05-8283-ff87ae962223”,
          ],
          "advertise_routes": [
          ]
        }
    },
    { "routinginstance":
      {
          “id”: “r2bcd3d8-2f02-4d05-8283-ff87ae962223”,
          "advertise_mode": "network",
          "discovery_mode": "network",
          "ipv4": "119.12.32.2",
          "ipv6": "",
          "agents": [
            “a3bcd3d8-2f02-4d05-8283-ff87ae962223”,
          ],
          "networks": [
            “n1bcd3d8-2f02-4d05-8283-ff87ae962223”,
            “n2bcd3d8-2f02-4d05-8283-ff87ae962223”,
          ],
          "advertise_routes": [
            {
              "destination": "33.33.33.0/22",
              "nexthop": "119.15.112.1",
            }
          ]

        }
      }
    }
  ]}

Now there is an example of the return entity, it is worth to clarify what this
data means:

* Routing Instance ``r1bcd3d8-2f02-4d05-8283-ff87ae962223`` will advertise the
  networks ``n1bcd3d8-2f02-4d05-8283-ff87ae962223`` through the peers connected
  to the agents ``a1bcd3d8-2f02-4d05-8283-ff87ae962223`` and
  ``a2bcd3d8-2f02-4d05-8283-ff87ae962223`` but won't update the routers of this
  network.

* Routing Instance ``r2bcd3d8-2f02-4d05-8283-ff87ae962223`` will advertise the
  networks ``n1bcd3d8-2f02-4d05-8283-ff87ae962223`` and
  ``n2bcd3d8-2f02-4d05-8283-ff87ae962223`` through the peers connected to the
  agent ``a3bcd3d8-2f02-4d05-8283-ff87ae962223`` and will update the routers
  associated to this networks with the discovered routes.


Create Routing Instance
***********************

+-----+-------------------------------------+------------------------------------+
|Verb |Uri                                  |Description                         |
+=====+=====================================+====================================+
|POST |/routinginstance                     |Create a new routing instance       |
+-----+-------------------------------------+------------------------------------+

Response Codes:

* 201: Normal
* 400: Bad Request (for example, invalid request format)
* 401: Unauthorized
* 403: Forbidden (for example, when non-administrator user tries to create a
  routing instance)
* 404: Not Found (for example, when the routing instance with the specified id
  does not exist)

This operation requires a request body and returns a response body. Both contain
RoutingInstance object. A JSON-encoded example is provided.

Request: ::

 {
  “routinginstance”:
  {
    "ipv4": "119.12.32.2",
    "discovery_mode": "network",
    "advertise_mode": "network"
  }
 }

Response: ::

 {
  “routinginstance":
  {
    “id”: “r1bcd3d8-2f02-4d05-8283-ff87ae962223”,
    "discovery_mode": "network",
    "advertise_mode": "network"
    "ipv4": "119.12.32.2",
    "ipv6": "",
    "agents": [
    ],
    "networks": [
    ],
    "advertise_routes": [
    ]
  }
 }


Associate a Routing Instance to a network
*****************************************

+-----+-------------------------------------+----------------------------------+
|Verb |Uri                                  |Description                       |
+=====+=====================================+==================================+
|PUT  |/routinginstance/{routinginstance_id\|Associate a Routing Instance to a |
|     |}/add_network                        |network                           |
+-----+-------------------------------------+----------------------------------+

Response Codes:

* 201: Normal
* 400: Bad Request (for example, invalid request format)
* 401: Unauthorized
* 403: Forbidden (for example, when non-administrator user tries to modify the
  routing instance)
* 404: Not Found (for example, when the network with the specified id does not
  exist)
* 409: Conflict (for example, the ID is valid but is not actually a network)

Request: ::

  {
    "network_id": “n1bcd3d8-2f02-4d05-8283-ff87ae962223”,
  }

Response: ::

  {
    “routinginstance":
    {
      “id”: “r1bcd3d8-2f02-4d05-8283-ff87ae962223”,
      "discovery_mode": "network",
      "ipv4": "119.12.32.2",
      "ipv6": "",
      "advertise_mode": "network",
      "agents": [
      ]
      "networks": [
        “n1bcd3d8-2f02-4d05-8283-ff87ae962223”,
      ],
      "advertise_routes": [
      ]
    }
  }


Disassociate a Routing Instance to a network
********************************************

+-----+-------------------------------------+----------------------------------+
|Verb |Uri                                  |Description                       |
+=====+=====================================+==================================+
|PUT  |/routinginstance/{routinginstance_id\|Disassociate a Routing Instance to|
|     |}/remove_network                     |a network                         |
+-----+-------------------------------------+----------------------------------+

Response Codes:

* 201: Normal
* 400: Bad Request (for example, invalid request format)
* 401: Unauthorized
* 403: Forbidden (for example, when non-administrator user tries to modify the
  routing instance)
* 404: Not Found (the routinginstance does not exist)

Request: ::

  {
    "network_id": “n1bcd3d8-2f02-4d05-8283-ff87ae962223”,
  }

Response: ::

  {
    “routinginstance":
    {
      “id”: “r1bcd3d8-2f02-4d05-8283-ff87ae962223”,
      "discovery_mode": "network",
      "advertise_mode": "network",
      "ipv4": "119.12.32.2",
      "ipv6": "",
      "agents": [
      ],
      "networks": [
      ],
      "advertise_routes": [
      ]
    }
  }


Associate a Routing Instance to an Agent
****************************************

+-----+-------------------------------------+----------------------------------+
|Verb |Uri                                  |Description                       |
+=====+=====================================+==================================+
|PUT  |/routinginstance/{routinginstance_id\|Associate a Routing Instance to an|
|     |}/add_agent                          |agent                             |
+-----+-------------------------------------+----------------------------------+

Response Codes:

* 201: Normal
* 400: Bad Request (for example, invalid request format)
* 401: Unauthorized
* 403: Forbidden (for example, when non-administrator user tries to modify the
  routing instance)
* 404: Not Found (for example, when the agent with the specified id does not
  exist)
* 409: Conflict (for example, the ID is valid but is not actually an agent)

Request: ::

  {
    "agent_id": “a1bcd3d8-2f02-4d05-8283-ff87ae962223”,
  }

Response: ::

  {
    “routinginstance":
    {
      “id”: “r1bcd3d8-2f02-4d05-8283-ff87ae962223”,
      "discovery_mode": "network",
      "advertise_mode": "network",
      "ipv4": "119.12.32.2",
      "ipv6": "",
      "agents": [
        "a1bcd3d8-2f02-4d05-8283-ff87ae962223"
      ],
      "networks": [
        “n1bcd3d8-2f02-4d05-8283-ff87ae962223”,
      ],
      "advertise_routes": [
      ]
    }
  }


Disassociate a Routing Instance to an Agent
*******************************************

+-----+-------------------------------------+----------------------------------+
|Verb |Uri                                  |Description                       |
+=====+=====================================+==================================+
|PUT  |/routinginstance/{routinginstance_id\|Dissasociate a Routing Instance to|
|     |}/remove_agent                       |an agent                          |
+-----+-------------------------------------+----------------------------------+

Response Codes:

* 201: Normal
* 400: Bad Request (for example, invalid request format)
* 401: Unauthorized
* 403: Forbidden (for example, when non-administrator user tries to modify the
  routing instance)
* 404: Not Found (for example, when the routing instance id does not exist)
* 409: Conflict (for example, the ID is valid but is not actually an agent)

Request: ::

  {
    "agent_id": “a1bcd3d8-2f02-4d05-8283-ff87ae962223”,
  }

Response: ::

  {
    “routinginstance":
    {
      “id”: “r1bcd3d8-2f02-4d05-8283-ff87ae962223”,
      "discovery_mode": "network",
      "advertise_mode": "network",
      "ipv4": "119.12.32.2",
      "ipv6": "",
      "agents": [
      ]
      "networks": [
        “n1bcd3d8-2f02-4d05-8283-ff87ae962223”,
      ],
      "advertise_routes": [
      ]
    }
  }


Update a Advertise Routes
*************************

This action gives you the chance to statically add routes to advertise. Please
note some advertise routes will be added dynamically by Neutron according to the
``advertise_mode`` field and the networks associated to the Routing Instance. So,
some deleted routes may appear again in the future.

+-----+-------------------------------------+----------------------------------+
|Verb |Uri                                  |Description                       |
+=====+=====================================+==================================+
|PUT  |/routinginstance/{routinginstance_id\|Update routes to advertise        |
|     |}/update_routes                      |                                  |
+-----+-------------------------------------+----------------------------------+

Response Codes:

* 201: Normal
* 400: Bad Request (for example, invalid request format)
* 401: Unauthorized
* 403: Forbidden (for example, when non-administrator user tries to modify the
  routing instance)
* 404: Not Found (for example, when the agent with the specified id does not
  exist)

Request: ::

  {
    "advertise_routes": [
      {
        "nexthop":"10.1.0.10",
        "destination":"40.0.1.0/24"
      }
    ]
  }

Response: ::

  {
    “routinginstance":
    {
      “id”: “r1bcd3d8-2f02-4d05-8283-ff87ae962223”,
      "discovery_mode": "network",
      "advertise_mode": "network",
      "ipv4": "119.12.32.2",
      "ipv6": "",
      "agents": [
        "a1bcd3d8-2f02-4d05-8283-ff87ae962223"
      ],
      "networks": [
        “n1bcd3d8-2f02-4d05-8283-ff87ae962223”,
      ],
      "advertise_routes": [
        {
          "nexthop":"10.1.0.10",
          "destination":"40.0.1.0/24"
        }
      ]
    }
  }


Update Routing Instance
***********************

+-----+-------------------------------------+------------------------------------+
|Verb |Uri                                  |Description                         |
+=====+=====================================+====================================+
|PUT  |/routinginstance/{routinginstance_id}|Update a routing instance           |
+-----+-------------------------------------+------------------------------------+

Response Codes:

* 201: Normal
* 400: Bad Request (for example, invalid request format)
* 401: Unauthorized
* 403: Forbidden (for example, when non-administrator user tries to modify the
  routing instance)
* 404: Not Found (for example, when the agent with the specified id does not
  exist)

Request: ::

  {
    "routinginstance": [
      {
        "advertise_mode":"None",
        "ipv6: "fe80::21f:3bff:fe02:8607/64"
      }
    ]
  }

Response: ::

  {
    “routinginstance":
    {
      “id”: “r1bcd3d8-2f02-4d05-8283-ff87ae962223”,
      "discovery_mode": "network",
      "advertise_mode": "None",
      "ipv4": "119.12.32.2",
      "ipv6: "fe80::21f:3bff:fe02:8607/64"
      "agents": [
        "a1bcd3d8-2f02-4d05-8283-ff87ae962223"
      ],
      "networks": [
        “n1bcd3d8-2f02-4d05-8283-ff87ae962223”,
      ],
      "advertise_routes": [
        {
          "nexthop":"10.1.0.10",
          "destination":"40.0.1.0/24"
        }
      ]
    }
  }


Delete Routing Instance
***********************

+-------+------------------------------------------+---------------------------+
|Verb   |Uri                                       |Description                |
+=======+==========================================+===========================+
|DELETE |/routinginstance/{id}                     |Delete a Routing Instance  |
|       |                                          |                           |
+-------+------------------------------------------+---------------------------+

Response Codes:

* 204: Normal
* 401: Unauthorized
* 404: Not Found (for example, if the Routing Instance does not exist)

This operation does not require request body and does not provide response body.

Security impact
---------------

This feature will allow an external system to manipulate routing information
within Neutron network. The external system should be trusted and may be
authenticated using a shared secret.

Dynamic routing may only be configured by the system administrator.


Notifications impact
--------------------

A notification should be provided when connectivity of control channel over
which routes are exchanged is interrupted


Other end user impact
---------------------

The following CLI commands will be added to manage peer connection
configuration:

* **routingpeer-list**: List configured peers.
* **routingpeer-show(id)**: Show detailed peer configuration for the
  specified peer connection.
* **routingpeer-create(gateway, peer, protocol, config)**: Create a new peer
  connection configuration.
* **routingpeer-update(id, config)**: Update existing peer connection
  configuration.
* **routingpeer-delete(id)**: Delete peer configuration for the specified
  peer connection.
* **agent-update(id)**: Modify the agent update call to allow associate
  ``dr_agent`` to ``routingpeer``.

The following CLI commands will be added to manage RoutingInstance specification for
connecting OpenStack to outside networks:

* **routinginstance-list**: List configured Routing Instances.
* **routinginstance-show(id)**: Show detailed Routing Instance configuration.
* **routinginstance-create(advertise_mode, discovery_mode)** Create new Routing
  Instance.
* **routinginstance-update(id, advertise-mode, discovery_mode)**: Update Routing
  Instance specification.
* **routinginstance-delete(id)**: Delete Routing Instance specification
* **routinginstance-routes-update(id, routes)**: Update the advertise routes.
* **routinginstance-agent-add(id, agent_id)**: Associate an agent to a Routing
  Instance
* **routinginstance-agent-remove(id, agent_id)**: Disassociate an agent to a
  Routing Instance
* **routinginstance-network-add(id, network_id)**: Associate a network to a Routing
  Instance
* **routinginstance-network-remove(id, network_id)**: Disassociate a network to a
  Routing Instance


Horizon Requirements
++++++++++++++++++++

A new screen will be added to configure gateway configuration for connecting
OpenStack to outside networks. This screen will allow routes and peer
configuration to be added to gateway configuration.

An external network will have an option to be linked to a routing instance.


Usage Example
+++++++++++++
Configure 2 uplinks for the routing instance serving an external network to
advertise its routes and update the discovered ones.

Sample configuration using Neutron CLI commands: ::

  neutron routingpeer-create --protocol bgp --gateway ext-gateway --peer
      10.10.0.15 \ --password secretsession --local-as 65010 --remote-as 65001

  neutron routingpeer-create --protocol bgp --gateway ext-gateway --peer
      10.10.0.16 \ --password secretsession --local-as 65010 --remote-as 65002

  neutron agent-update --add-peer created_peer1_uuid agent_id

  neutron agent-update --add-peer created_peer2_uuid agent_id

  neutron routinginstance-create --advertise_mode network --discovery-mode
    network

  neutron routinginstance-agent-add agent_id

  neutron routinginstance-network-add network_id



Performance Impact
------------------

This feature describes an out of band mechanism to negotiate routing
configuration. This feature should not have a performance impact on Neutron
network.


Other deployer impact
---------------------

This feature would have to explicitly enabled and configured before it will take
effect. There are no changes to configuration files.


Developer impact
----------------

This change does not affect current developments or any plugin development.

Neutron API exposed is agnostic of the exchange routing protocol used.  If
another developer want to provide other driver than BGP with exabgp, only the
``dr_agent`` part will be affected with new code.


Implementation
==============


Assignee(s)
-----------

This is a pre-liminary contributor list

Primary assignee:
  nextone92

Other contributors:
  devvesa


Work Items
----------

* Create the ``dr_agent``, exposing the API and implemented with the chosen
  BGP speaker. (`BGP Comparison`_)
* Model tables and API resources.
* Periodically scheduled process to communicate with ``dr_agent``.
* Testing.
* Devstack.
* Documentation.


Dependencies
============

Depending on the implementation, new system library or python library will need
to be installed.


Testing
=======

Dynamic routing testing may be performed in an isolated environment. An external
autonomous system may be simulated with an instance of BGP capable software
router (for example, quagga).

The following dynamic routing scenarios could be tested:

Verify that when BGP is enabled on the gateway and one peer is configured the
agent establishes BGP session with the peer, receives a list of routes, and
submits advertised routes to the peer.

Verify that when BGP is disabled on the gateway and one peer is configured the
dr_agent establishes no BGP sessions.

Verify that when BGP is enabled on the agent and 3 BGP peer connections are
configured, the agent establishes 3 BGP sessions, one to each of the
configured peers.

When 2 or more peers are configured, verify that BGP implementation is able to
detect when the BGP session is interrupted the routes received from that BGP
session are automatically removed from the routing table.


Documentation Impact
====================

New documentation for the whole functionality.


References
==========

* `Neutron Dynamic Routing Use Cases
  <https://wiki.openstack.org/wiki/Neutron/DynamicRoutingUseCases>`_
* `Pluggable External Net Blueprint
  <https://blueprints.launchpad.net/neutron/+spec/pluggable-ext-net>`_
* `Border Gateway Protocol
  <http://en.wikipedia.org/wiki/Border_Gateway_Protocol>`_
* `Quagga <http://www.nongnu.org/quagga/>`_
* `BGP/MPLS IP Virtual Private Networks (VPNs)
  <http://tools.ietf.org/html/rfc4364>`_
* `etherpad <https://etherpad.openstack.org/p/juno-dynamic-routing>`_
* `VRF <http://en.wikipedia.org/wiki/Virtual_Routing_and_Forwarding>`_
* `BGP Comparison
  <https://wiki.openstack.org/wiki/Neutron/BGPSpeakersComparison>`_
