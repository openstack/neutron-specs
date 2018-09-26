..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
Dynamic Advertising Routes for Public Ranges
============================================

`Blueprint Link
<https://blueprints.launchpad.net/neutron/+spec/bgp-dynamic-routing>`_

The goal of this blueprint is to add dynamic routing capability to OpenStack
deployment. This feature would allow Neutron to advertise its own routes to an
external router using the BGP protocol.

Problem Description
===================

This is a new feature and this section will describe use cases where dynamic
routing may be useful.


**Expose External Networks Dynamically**

Allow Neutron to dynamically announce to external uplink routers.

An OpenStack cloud would run a routing protocol (for example, BGP) against at
least one router in each uplink network provider. By announcing network
prefixes to those peers, the Neutron network would be reachable by the rest of
the internet via both paths. If the link to an uplink provider broke, the
failure information would propagate to routers further up the stream, keeping
the cloud reachable through the remaining healthy link.

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
      //label = "OpenStack Deployment";
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

This also would be a valid scenario for an internal cloud, where the uplink
router is actually one's provider router and the routes are advertised using
iBGP.


**Routed Model for Floating IPs on External Network**

This use case will allow an external network with a large public IP space to
possibly span more than one L2 network. This could allow for improved scale and
isolation between AZs while maintaining the appearance of a large external
network where floating IPs and networks can float freely.

This use case may also include announcing public networks behind Neutron routers
to an upstream router. For example, this will be useful for IPv6 networks when
using the `subnet-allocation` mechanism to allocate routable IPv6 prefixes to
tenant networks. It does not require learning routes from the upstream router.

`Address Scopes`_ blueprint adds a new L3 context to make public routable tenant
networks (among other advantages). Address Scopes may leverage this development
to advertise the new routable subnets to uplink routers.

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


The topology of this use case can be seen as a generalization of the previous
one, with a multi-homed OpenStack installation and leverage the fact that a
floating IP can be seen as a /32 network.

Proposed Change
===============


Overview
--------

A new system that dynamically advertises routes to other peers outside the
OpenStack deployment is proposed. From the Neutron API, cloud administrator
should be able to define these peers as well as the way to interact with them.


Routing Peer
++++++++++++

A system that supports dynamic routing must be able to advertise its own
routes to its peers. It order to achieve this goal the system must know the list
of its peers and be able to trust the information that it receives from them.


Route Advertisement
+++++++++++++++++++

Routes added manually by administrator will be advertised to external dynamic
routing peers.


Dynamic Routing System
++++++++++++++++++++++

In default implementation a new system will be used to manage dynamic routing
information at the edge of OpenStack deployment. Dynamic peering will not be
performed by each individual Neutron router due to scaling concerns. Such an
approach could create an inordinately high number of peering relationships.
Instead, the model proposed sets up a speaker to represent the Neutron
deployment as a whole to external routers.

This system will allow different implementations (develop your own BGP speaker
implementation, for instance) to fit into third party requirements.


IPv6 Impact
-----------

The implementation must be able to exchange IPv4 and IPv6 routes.  This
blueprint may be even more important with IPv6 to complete routing from outside
the cloud to tenant networks running IPv6.


Solution Proposed
-----------------

Reference implementation will employ a dynamic routing agent (``dr_agent``).
This agent will have direct connectivity with configured peers and it will be
responsible to update the dynamic routes of some defined routers. Another spec
will be filled to define the details of this implementation.

So, through the Neutron API, cloud administrator should be able to define the
peer connections for each agent according to his needs.

The cloud administrator will be able to execute the following new actions,
through the Neutron API:

1. Create a ``BGPSpeaker`` entity with configuration options of the BGP
   connection. First approach will only need the ``local_as`` attribute. Future
   implementations, such as the `policy support for dynamic routing`_, will need
   to add more attributes here. The implementation driver need to read this
   configuration options an establish BGP sessions to its peers.
2. Create a ``RoutingPeer`` entity.
3. Associate a ``RoutingPeer`` entity to a ``BGPSpeaker``. The
   ``BGPSpeaker`` implementation driver will be able to connect to this
   ``RoutingPeer``
4. Associate a ``Router`` to a ``BGPSpeaker``. That means the router will
   have dynamic routing capabilities that will let it advertise its routes
5. Associate a ``Network`` to a ``BGPSpeaker``. All the routers with
   external gateways attached to this network will use the same
   ``BGPSpeaker`` attributes.

At this point, Neutron will compare the address scope of the subnets to which
assigned routers (step 5 implies all the routers of the external network) have
their external gateways connected to the internal subnets of the tenants to see
if they belong to the same address scope.  If so, the BGP speaker
implementation will advertise these tenant subnets to its configured
``RoutingPeers``.  Floating IPs will be included implicitly since they are
allocated from the external network.


Considerations
--------------


Alternatives
------------

Multi-homed clouds can be handled using classic networking infrastructure,
configuring manually the vendor router with BGP outside the OpenStack
deployment.  This is limited.  It can't handle the routed floating ip model
proposed above.


Data Model Impact
-----------------

This document proposes modifying data objects and schema in the following way.
For a quick glance of the Data Object Model, check out this etherpad_.

Data Object Changes
+++++++++++++++++++

Three new data model classes will be added: ``BGPSpeaker``, ``RoutingPeer``
and ``AdvertiseRoute``.

We will need the binding entities:

 * ``RoutingPeerBGPSpeakerBinding`` to associate peers to ``BGPSpeaker``.
 * ``RouterBGPSpeakerBinding`` to associate routers ``BGPSpeaker``.
 * ``NetworkBGPSpeakerBinding`` to associate networks to ``BGPSpeaker``.

New ``BGPSpeaker`` class will contain the following attributes:

 * ``id``: UUID of the entity.
 * ``local_as``: Local AS value.

Now we only need these values. In the future, more advanced configuration
options of ``BGPSpeaker`` will be able to be added here.

New ``RoutingPeer`` class that represents a peer connection will contain the
following attributes:

 * ``id``: UUID of the entity
 * ``ip``: IP Address of the peer.
 * ``remote_as``: Remote Peer's AS value.
 * ``auth``: Authentication data of the connection that can be serialized as a
             dictionary. { ``type``: 'MD5', ``password``: '234a23d10234' } could be
             a simple example and first approach.

Another data object called ``AdvertiseRoute`` will be created extending the
``Route`` entity and associated to a router. Will have the following
attributes:

 * ``nexthop``:  IP address of the next hop.
 * ``destination``: CIDR prefix.
 * ``router_id``: UUID of the Routing Instance.

``RoutingPeerBGPSpeakerBinding`` is an n-to-n relationship between bgp peer
and bgp speaker and only will have:

 * ``peer_id``: UUID of the BGPPeer
 * ``bgpspeaker_id``: UUID of the Dynamic Routing Agent

``RouterBGPSpeakerBinding`` is an n-to-1 relationship with the attributes:

 * ``router_id``: UUID of the Router
 * ``bgpspeaker_id``: UUID of the Dynamic Routing Agent.

``NetworkBGPSpeakerBinding`` is an n-to-1 relationship with the attributes:

 * ``network_id``: UUID of the Network
 * ``bgpspeaker_id``: UUID of the Dynamic Routing Agent.

``RouterBGPSpeakerBinding`` it will be used when you want to propagate a single
router's AdvertiseRoutes. This option is suitable for the ``Expose External
Networks Dynamically`` use case, explained above. It can be seen as adding
dynamic routing capabilities to a Router

``NetworkBGPSpeakerBinding`` is the ``Routed Model for Floating IPs on External
Network`` use case: in this case you propagate tenant routes (IPv6) or Floating
IPs into an upstream router. These routes are more dynamic to assign, because it
is up to the tenant use to set them. You will want any router attached to a
network, to automatically be added to propagate its routes.


REST API Impact
---------------

API endpoints should be implemented according to the Solution Proposed section.

Security Impact
---------------

This feature will allow an external system to manipulate routing information
within Neutron network. The external system should be trusted and may be
authenticated using a shared secret.

Dynamic routing may only be configured by the system administrator.


Notifications Impact
--------------------

A notification should be provided when connectivity of control channel over
which routes are exchanged is interrupted


Other End User Impact
---------------------

The following CLI commands will be added to manage dynamic routing specification for
connecting OpenStack to outside networks:

* **bgp-speaker-list**: List configured bgpspeakers.
* **bgp-speaker-show**: Show detailed bgpspeaker configuration.
* **bgp-speaker-create** Create new bgpspeaker connection.
* **bgp-speaker-update**: Update bgpspeaker specification.
* **bgp-speaker-delete**: Delete bgpspeaker specification.
* **bgp-speaker-peer-add**: Associate a peer to a bgpspeaker.
* **bgp-speaker-peer-list**: List Peers on bgpspeaker
* **bgp-speaker-peer-remove**: Remove peers on bgpspeaker.
* **bgp-speaker-network-add**: Associate a network to a bgpspeaker.
* **bgp-speaker-network-list**: List networks on bgpspeaker
* **bgp-speaker-network-remove**: Remove networks on bgpspeaker.
* **bgp-speaker-router-add**: Associate a peer to a bgpspeaker.
* **bgp-speaker-router-list**: List routers on bgpspeaker
* **bgp-speaker-router-remove**: Remove router on bgpspeaker.

* **bgp-peer-list**: List configured peers.
* **bgp-peer-show**: Show detailed peer configuration.
* **bgp-peer-create** Create new peer connection.
* **bgp-peer-update**: Update peer specification.
* **bgp-peer-delete**: Delete peer specification.

* **router-advertiseroutes-list**: List advertise routes.

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

  neutron bgp-speaker-create --local-as 12345

  neutron bgp-peer-create --ip 123.23.43.4 --remote-as=1234

  neutron bgp-speaker-peer-add peer1 (previous step created peer. should be an
  uuid, but modified for the sake of understanding)

  neutron bgp-speaker-network-add bgpspeaker1 network1 (same here, with uuids)


Performance Impact
------------------

This feature describes an out of band mechanism to negotiate routing
configuration. This feature should not have a performance impact on Neutron
network.


Other Deployer Impact
---------------------

This feature would have to explicitly enabled and configured before it will take
effect. There are no changes to configuration files.


Developer Impact
----------------

This change does not affect current developments or any plugin development.

Neutron API exposed is agnostic of the exchange routing protocol used.  If
another developer want to provide other driver than BGP with exabgp, only the
``dr_agent`` part will be affected with new code.


Community Impact
----------------

This change does not impact community.


Alternatives
------------

Taking into account the use case (BGP connectivity), we think that the agent
approach is the only one that fits into Neutron. Although maybe the
functionality would be solved using another entities and workflow.


Implementation
==============


Assignee(s)
-----------

This is a pre-liminary contributor list

Primary assignee:
  tidwellr
  vikram

Other contributors:
  devvesa
  YAMAMOTO


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

Tempest Tests
-------------

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


Functional Tests
----------------

Full top-down Neutron API internal logic must be developed by mocking the agent.


API Tests
---------

All API exposed endpoints by Neutron extensions must be tested.


Documentation Impact
====================

New documentation for the whole functionality.

User Documentation
------------------

User documentation explaining the functionality must be provided.

Developer Documentation
-----------------------

Developer documentation about how to develop a new driver for the ``dr_agent``
must be provided.


References
==========

Previous work:

* `Introducing entities
  <https://review.openstack.org/#/c/115554/>`_
* `Add the dr_agent
  <https://review.openstack.org/#/c/111311/>`_

Links and helpers:

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
* `Ryu's BGP speaker
  <http://ryu.readthedocs.org/en/latest/library_bgp_speaker.html>`_
* `policy support for dynamic routing
  <https://blueprints.launchpad.net/neutron/+spec/neutron-route-policy-support-for-dynamic-routing-protocol>`_
* `Address Scopes
  <https://bugs.launchpad.net/neutron/+bug/1453921>`_
