..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Enhancement to Neutron BGPaaS
=============================

https://bugs.launchpad.net/neutron/+bug/1921461

Enhancement to Neutron BGPaaS to directly support Neutron Routers and
bgp-peering from such routers over internal & external Neutron Networks

Problem Description
===================

There are very good foundation APIs in Neutron BGPaaS that brought in BGP
service functionality (see [1]_) into Neutron through Neutron Dynamic Routing.
(see [3]_ )

.. note::

    In this spec, Neutron-BGPaaS, BGPaaS and  neutron-dynamic-routing/bgp are
    used as synonymns. These refers to "Neutron Dynamic Routing using BGPaaS"
    function already available in Neutron. Similarly, 'ISP network' and
    'Neutron External Network' are used as synonymns.

In a Telco Service Provider environment, VNFs (Virtual Network Functions [4]_)
& CNFs (Containerized-Network-Functions) are run that provides simple-to-fairly
complex services to subscribers. These VNFs use many Non-Neutron IP-Addresses
which need to be exposed to subscribers. The subscribers access these services
through these Non-Neutron IP-Addresses, referred as 'Service-Address'. It can
belong to either IPv4 (or) IPv6 address-families.

Service-Address are wholly managed inside a VNF transparent to Neutron. But
these Service-Address are reachable on the dataplane by subscribers, through
VNICs (or Ports) holding Neutron-managed Addresses. In short, the Neutron
managed primary-addresses on VNFs VNICs act as NextHops to the Service-Address
of VNF.

In order to enable L3 reachability to these Service-Addresses from an
ISP-network (or) make the Service-addresses reachable from another
neutron-tenant-network entity, these Service-Addresses need to be brought into
a Neutron Router's Routing Table.

The number of VNFs are high and hence the number of service addresses to be
exposed are higher. "BGP between the VNFs and their hosting Neutron Routers"
provides a very elegant and scalable automated approach for a control-plane
driven route-learning of Service-Address by the Neutron Router from the VNFs.
This enables L3 reachability of such Service-Addresses from other
neutron-tenant-network-VNFs that is attached to the same Neutron Router.

These service addresses also need to be advertised to the ISP-PE-Router. This
will enable subscribers present on ISP-network be able to use the services
exported by applications in the VNFs.

The existing Neutron Dynamic BGPaaS falls a little short of supporting the
above requirements due to below reasons:

* Existing BGPaaS lacks the functionality (both API and implementations) to do
  BGP Peering over Neutron Internal Networks. The addition of service-addresses
  to Neutron Router's Routing Table requires BGP control plane to bring them in
  via BGP peering (see [2]_) towards the VNFs over Neutron Tenant Networks from
  Neutron Router. It also requires BGP speaker to learn routes from the BGP
  peers. Current implementation advertises routes to peers but doesn't learn
  from peers.
* Existing Neutron BGPaaS supports BGP Peering only over Non-Neutron Networks.
  Please refer the current implementation and documentation as in [8]_ .
  Advertisment of service-addresses to the ISP-PE-Router requires BGP Peering
  over Neutron External Networks. This will enable the ISP-PE-Router to learn
  the Service-Addresses over the BGP control plane. Thereby, enabling
  L3-reachability for subscribers on the ISP-side to be able to take advantage
  of services offered by VNFs (and CNFs).
* Exising BGPaaS doesn't provide API/implementation to associate a BGP speaker
  to a router. It currently supports a'loose-association'of network to BGP
  speaker. This 'loose-association' is concept where the BGP speaker derives
  the routes of the Neutron Router through APIs and not due to direct
  connectivity of BGP Speaker to the Neutron Router. Association of BGP speaker
  to router enables access to all the neutron-router-interfaces ie. both
  internal/external interfaces. This will enable a single bgpspeaker associated
  with the Neutron Router, be able to peer simultaneously towards VNFs on
  neutron-tenant-network as well as the ISP-PE-Router on
  neutron-external-network.
* It also doesn't support BGP peer group feature. BGP Peer Group feature
  enables automatic-bgp-peering towards VNFs by Neutron Router whenever the
  VNFs are scaled-out in a VNF cluster or whenever the VNFs are scaled in from
  a VNF cluster. The automatic-bgp-peering refers to the fact that the current
  BGP Speaker will be enhanced to automatically accept peering-requests from
  VNF if the VNFs are within a pre-configured ip-address-range (aka
  listen-range) that openstack tenant can configure for the BGP speaker.
* BGPaaS capability will have varying implementations from different Neutron
  backends.  This makes it important to have some feedback ingrained in the
  Neutron BGPaaS API. It will allow various Neutron backends (including
  reference BGPaaS backend) to drive realization status inside Openstack Infra
  for the BGP resources exposed by Neutron BGPaaS. To provide that, the
  proposal in the spec is to add a new resource type 'Association' that can
  work with current BGPSpeaker object. More specifically, the plan is to
  support router_association, peer_association as resources under a BGPSpeaker
  object. A ``router_association`` resource will be used to create a binding
  between a BGPSpeaker and a Neutron Router, that will internally realizes BGP
  Control Plane over that Neutron Router. Similarly, a ``peer_association``
  resource will be used to create a binding between a BGPSpeaker and a BGPPeer
  that will internally realize BGP Peering between the BGPSpeaker and the
  BGPPeer resource.

Proposed Change
================

To enable L3 reachability to the Service-Addresses from ISP-network or from
another-neutron-tenant-network entity, the below usecases are introduced :

#. ``Enable direct BGP-Peering (see [2]_) from a Neutron Router towards the
   VNFs (or CNFs) on its connected networks.`` This provides L3 reachability to
   the service-addresses hosted on such VNFs, as such addresses will be
   installed in the Neutron Router's routing-table by the BGPaaS control
   plane. The proposal is to provide facility to directly associate a BGP
   Speaker to a neutron-router, that will in turn enable the router to reach
   BGP-Control plane learned routes from the VNF.. A single router can be
   associated to a BGP speaker and the speaker will be running inside the L3
   router namespace.
#. ``Enable BGP-Peering over Neutron External Networks from a Neutron Router to
   the ISP-PE-Router.`` This allows the ISP-PE-Router to learn the
   Service-Addresses over the BGP control plane from Neutron Router. Thereby,
   enabling L3-reachability for subscribers on the ISP-side to leverage
   services offered by VNFs (and CNFs).
#. ``Support BGP peer group feature`` This will allow to transparently scale
   bgp peering to a  number of VNFs, whenever VNF clusters are scaled-out or
   scaled-in. It supports BGP peering to a group of remote neighbors that are
   configured using a range of IP addresses. This can also be used in k8s over
   openstack deployment where CNF scale-out/scale-in is a pure k8s operation.
   It would be undesirable to call neutron APIs in this case.

New resources (Network Association, Router Association, Peer Association) will
be added as part of this spec. Association resources provides the below
benefits

* Enables provisioning of certain BGP control plane parameters specific to
  BGP-Speaker-to-BGP-Peer binding and also specific to BGP-Speaker-to-Router
  binding. For eg: some attributes (status, advertise-extra-routes etc) are
  useful when linked to associations rather than directly to parent resources
  (speaker, peer). A BGP Peer can be associated to different BGP Speakers, so
  having a Status field in bgp-peer won’t be beneficial. It is useful only
  when associated to Peer Association resource.
* Enables the Neutron BGPaaS APIs to provide feedback on the realization
  status of BGP Control Plane towards Openstack Tenant. For eg: In this spec,
  a new Status field is planned to be introduced. This will allow a
  Neutron-BGPaaS backend driver to provide realization-status for different
  BGP resources inside Openstack Infra.

::

   +-----------------------------------------------+
   |                                               |
   |                ISP- PE Router                 |---------|
   |                                               |         |
   +-----------------------------------------------+        EBGP
                           |                                 |
                           |                                 |
   +-----------------------------------------------+         |
   |                                               |---------|
   |                Neutron Router                 |
   |                                               |---------|
   +-----------------------------------------------+         |
                         |                                   |
                         |                                 EBGP
   +-----------------------------------------------+         |
   |                 VNF Cluster                   |         |
   |  +-----------------+   +-----------------+    |---------|
   |  |       VM1       |   |       VM2       |    |
   |  +-----------------+   +-----------------+    |
   +-----------------------------------------------+

To advertise the multiple service addresses hosted by VNFs to ISP-PE routers,
2 BGP sessions are created. One BGP session created directly towards the VNFs
from a Neutron Router hosting the neutron-tenant-networks of the VNF. And the
next BGP-Peering towards the ISP-PE-Routers from Neutron Router directly over
the Neutron External Networks.

Proposal is to enhance existing BGPaas (see [6]_), allow neutron router to be
associated to a BGP Speaker and allow BGP Speaker to peer with both the
internal-Networks and External-Networks present on that Neutron Router. This
will be implemented using enhancements to the neutron-service and
neutron-dynamic-routing.

Implementation approaches
-------------------------

Option 1
~~~~~~~~

A BGP speaker is spawned inside router namespace, when a neutron-router is
associated to BGP Speaker. BGP speaker will be running inside the L3 router
namespace which enables access to all the neutron-router-interfaces ie. both
internal/external interfaces. BGP functionality provided by OS-Ken will be
reused to excite BGP speaker functionality to run only within the neutron
router namespace.

``BGP Service Plugin`` will be enhanced to accept the request to associate a
BGP speaker to the router. ``BGP L3 agent extension`` will implement the L3
agent side of BGP enhancements. It will be responsible for realizing
bgpspeaker inside the router-namespace and now bgpspeaker can peer over
networks on the router from inside the router-namespace. BGP speaker can
listen on 0.0.0.0:179 which enables it to listen on all the
neutron-router-interfaces and can peer with the neighbour based on the
configuration.

::

 +---------------+              +-----------------------------------+
 |DHCP namespace |              |Router namespace                   |
 |     qdhcp     |              |     qrouter                       |
 |   +------+    |              |  +---+   +---+   +-----------+    |
 |   | tap  |    |              |  | qr|   | qg|   |BGP Speaker|    |
 |   +------+    |              |  +---+   +---+   +-----------+    |
 |       |       |              |    |       |                      |
 +---------------+              +-----------------------------------+
         |                           |       |
 +--------------------------------------------------------------+
 |       |                           |       |                  |
 |     +--------+               +-------+  +-------+            |
 |     |Port tap|               |Port qr|  |Port qg|            |
 |     +--------+               +-------+  +-------+            |
 |                      br-int                                  |
 +--------------------------------------------------------------+


Option 2
~~~~~~~~

In Option 1, since BGP speaker is spawned inside the router namespace, the
speaker is tightly coupled with that router namespace. For a different BGP
Speaker on a different router, wherein that router in turn is realized on the
same network-node hosting the original router, a new instantiation of the BGP
Speaker is required. This could potentially bring in scalability and
performance problems with Option 1.

In Option 2, the proposal is to create and use VRF for BGP-Peering. VRF device
(see [9]_) can be used instead of namespace for BGP peering. VRF is a layer 3
master network device with its own associated routing table. It also provides
added benefit of ``VRF any`` socket which allows a single process instance to
efficiently provide service across all VRFs. So for N router associated BGP
speakers, N vrfs will be created but a single BGP speaker will serve all of
them.

::

 +---------------+    +-----------------+    +-----------------+
 |DHCP namespace |    |Router namespace |    |  VRF device     |     +-------+
 |        qdhcp  |    |       qrouter   |    |                 |     |  BGP  |
 |   +------+    |    |  +---+   +---+  |    |  +---+   +---+  |N---1|speaker|
 |   | tap  |    |    |  | qr|   | qg|  |    |  | vr|   | vg|  |     +-------+
 |   +------+    |    |  +---+   +---+  |    |  +---+   +---+  |
 |       |       |    |    |       |    |    |    |       |    |
 +-------------- +    +-----------------+    +-----------------+
 |                 |       |       |              |       |
 +-----------------------------------------------------------------+
 |       |                 |       |              |       |        |
 |     +--------+      +-------+  +-------+  +-------+  +-------+  |
 |     |Port tap|      |Port qr|  |Port qg|  |Port vr|  |Port vg|  |
 |     +--------+      +-------+  +-------+  +-------+  +-------+  |
 |                                                                 |
 |                      br-int                                     |
 +-----------------------------------------------------------------+

Current L3Plugin and L3Agent will continue to provide Neutron-Router
functionality. VRF will be realized by BGP-Dr-Agent and a new VRF is created
when a neutron-router is associated to BGP Speaker. VRF will be used only for
BGP peering and the learnt routes are installed inside router namespace.

Extra IPs has to be allocated from internal and external subnet pools for the
vr, vg interfaces respectively. These IPs will be used for BGP peering and
the allocation of new IPs can be considered as a short coming with this
option.

Option 3
~~~~~~~~

VRF device can provide most of the L3 functionalities provided by router
namespace. In this option, the idea is to completely replace the namespace
with VRF device. A new VRF will be created when a neutron-router is created.

Both L3-Routing as well as BGP can be realized through VRFs. Linux-VRF will
provide neutron router functionality which includes dataplane L3-forwarding
for east-west, north-south and NATing.

This provides the same advantages as in option 2 and eliminates its
disadvantages. But the main concern with this approach is that it will make a
huge deviation from the existing L3 implementation which will require a lot
of effort and time. This also requires changes to upgrade mechanisms.

.. note::

    The current plan is to implement Option 1 which is simpler and doesn't
    deviate much from the existing implementation. Incase the implementation
    option 3 is going to be used, it will be only a gradual replacement (from
    namespace to vrf) with proper upgrade path.

HA-Capable Neutron Router with BGPaaS
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
When HA-Capable neutron router is associated to a BGPSpeaker, two BGPSpeaker
instances will be run, one on active router namespace and other on the standby
router namespace. Whenever the failover happens from Active to Standby
Namespace, the BGPSpeaker on Standby will be able to peer with BGP-Peers and
become an Active-BGP-Speaker managing the Router.

The implementation planned in this spec supports only Centralized Neutron
Routers (both HA and non-HA) and not Distributed Routers.

REST API & Neutron client command Impact
----------------------------------------

Use-case a)

Associate neutron router to BGP Speaker API
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

 POST /v2.0/bgp-speakers/<bgp-speaker-id>/router_associations

 {
  "router_association":
      {
       "router_id": "c930d7f6-ceb7-40a0-8b81-a425dd994ccf",
       "advertise_extra_routes": True
      }
 }

* ``router_id`` represents the UUID of the neutron router to which the BGP
  speaker has to be associated.
* ``advertise_extra_routes`` decides whether neutron extra routes on the
  neutron router will be redistributed to bgp-peers by the bgpspeaker. Default
  is ``True`` and so all neutron extra routes will be redistributed to every
  bgp-peer that is bound to the BGP Speaker. Turning OFF
  advertise_extra_routes will disable advertisement of neutron router’s
  extra-routes by the bgpspeaker.

.. note::

    The enhancement will support a router to be associated to only a single
    BGP speaker. The association of router to speaker and network to speaker
    will be a mutually exclusive operation.

New neutron Client Command::

 openstack bgp speaker router association create
               [--advertise-extra-routes]
               [--no-advertise-extra-routes]
               <bgpspeaker>
               <router>

Disassociate neutron router from BGP Speaker API
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

 DELETE  /v2.0/bgp-speakers/<bgp-speaker-id>/router_associations/<router
     -association-uuid>

New neutron Client Command::

 openstack bgp speaker router association delete
               <bgpspeaker>
               <router>

.. note::

    Deleting a BGP SPeaker will not be permitted, if the speaker already has
    multiple associations on itself like peer-associations and
    router-associations. Similarly, deletion of a router that is asoociated
    to a BGP speaker is also not allowed.

Update BGP Speaker Router Association
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

 PUT /v2.0/bgp-speakers/<bgp-speaker-id>/router_associations/<router
     -association-uuid>

 {
  "router_association":
      {
       "router_id": "c930d7f6-ceb7-40a0-8b81-a425dd994ccf",
       "advertise_extra_routes": True
      }
 }

* ``advertise_extra_routes`` field can be set to True or False when updating
  the router association.

New neutron Client Command::

 openstack bgp speaker router association update
               [--advertise-extra-routes]
               [--no-advertise-extra-routes]
               <bgpspeaker>
               <router>

List Router associations for a given BGP speaker
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

 GET /v2.0/bgp-speakers/<bgp-speaker-id>/router_associations

 {
  "router_associations": [
      {
       "router_id": "c930d7f6-ceb7-40a0-8b81-a425dd994ccf",
       "advertise_extra_routes": True
       "status": 'ACTIVE'
      },
      {
       "router_id": "a330d7f6-ceb7-40a0-8b81-a425dd994bbe",
       "advertise_extra_routes": True
       "status": 'ACTIVE'
      }
  ]
 }

* ``status`` attribute helps to know whether Neutron-BGPaaS backend software
  has realized the router association successfully on the openstack
  infrastructure. Status field can be either ``DOWN`` or ``ACTIVE``.


Show details for a BGP speaker Router Association
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

 GET /v2.0/bgp-speakers/<bgp-speaker-id>/router_associations/<router
     -association-id>

 {
  "router_association":
      {
       "router_id": "c930d7f6-ceb7-40a0-8b81-a425dd994ccf",
       "advertise_extra_routes": True
       "status": 'ACTIVE'
      }
 }

Create BGP speaker Peer Association
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

 POST /v2.0/bgp-speakers/<bgp-speaker-id>/peer_associations

 {
  "peer_association":
      {
       "peer_id": "b930d7f6-ceb7-40a0-8b81-a425dd994ccf",
      }
 }

* ``peer_id`` represents the UUID of the BGP peer to which the BGP speaker has
  to be associated.

New neutron Client Command::

 openstack bgp speaker peer association create
               <bgpspeaker>
               <peer>

Delete BGP speaker Peer Association
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

 DELETE  /v2.0/bgp-speakers/<bgp-speaker-id>/peer_associations/<peer
     -association-uuid>

.. note::

    Deleting a BGP speaker which has peer-associated will not be allowed.

New neutron Client Command::

 openstack bgp speaker peer association delete
               <bgpspeaker>
               <peer>

List Peer associations for a given BGP speaker
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

 GET /v2.0/bgp-speakers/<bgp-speaker-id>/peer_associations

 {
  "peer_associations": [
      {
       "peer_id": "b930d7f6-ceb7-40a0-8b81-a425dd994ccf",
       "status": 'ACTIVE'
      },
      {
       "peer_id": "a640d7f6-ceb7-40a0-8b81-a425dd994dde",
       "status": 'ACTIVE'
      }
  ]
 }

* ``status`` attribute helps to know whether Neutron-BGPaaS backend software
  has realized the peer association successfully on the openstack
  infrastructure. Status field can be either ``DOWN`` or ``ACTIVE``.

Show details for a BGP speaker Peer Association
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

 GET /v2.0/bgp-speakers/<bgp-speaker-id>/peer_associations/<peer
     -association-id>

 {
  "peer_association":
      {
       "peer_id": "b930d7f6-ceb7-40a0-8b81-a425dd994ccf",
       "status": 'ACTIVE'
      }
 }

Show routes managed by BGP Speaker API
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

 GET /v2.0/bgp-speakers/<bgp-speaker-id>/get_routes

 {
      "routes": [
          {
               "cidr": "192.168.10.0/24",
               "nexthop": ["10.0.0.1"],
               "route-type": "local"
          },
          {
               "cidr": "192.168.11.0/24",
               "nexthop": ["10.0.0.5", "10.0.0.6"],
               "route-type": "peer"
          }
        ]
 }

* ``nexthop`` is a list which contains ip addresses that is going to be used
  to reach a certain destination cidr.
* ``cidr`` represents the CIDR prefix.
* ``route-type`` can be local or peer based on whether the routes are local or
  learnt from the peer respectively.

The routes can be obtained from the backend. For example, in case of os_ken
backend, rib_get method (see [10]_) can be used to obtain the routes.

New Neutron Client Command::

 openstack bgp speaker list routes <bgpspeaker>

Use-case b)

Create BGP Peer Group API
~~~~~~~~~~~~~~~~~~~~~~~~~
::

 POST /v2.0/bgp-peer-groups/
 {
    "bgp_peer_groups":{
       "name":"bgppeergroup1",
       "project_id":"",
       "remote_asn":"4566",
       "next_hop_self" : True,
       "update_source_ip": "10.20.1.5"
       "auth_type": "md5",
       "password": "<passwd>"
     }
 }

* ``remote_asn`` represents the remote AS number of a BGP peer group
* ``next_hop_self`` decides whether to modify the nexthop attribute to its own
  during BGP advertisement.
* ``update_source_ip`` forces BGP to use the IP address specified while
  talking to a BGP neighbor.
* ``auth_type`` determines the authentication algorithm. Supported algorithms
  are  none and md5, none by default.
* ``password`` represents the authentication password for the specified
  authentication type.

New neutron Client Commands::

 openstack bgp peer group create
               [--next-hop-self]
               [--no-next-hop-self]
               [--auth-type <auth-type>]
               [--password <password>]
               [--update_source-ip <ip-address>]
               --remote-asn <asn-number>
               <bgp-peer-group-name>

Delete BGP Peer Group API
~~~~~~~~~~~~~~~~~~~~~~~~~
::

 DELETE /v2.0/bgp-peer-groups/<bgp-peer-group-id>

Deletion of peer-group will succeed only if there are no BGP-Peers referring
to this peer-group.

New Neutron Client Commands::

 openstack bgp peer group delete <bgp-peer-group-id>

Create a bgp-peer using a pre-created peer-group API
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

 POST /v2.0/bgp-peers/
 {
    "bgp_peer":{
       "peer_group_id":"a930d7f6-ceb7-40a0-8b81-a425dd994ccf",
       "name":"bgppeer1",
       "listen_range": "10.2.1.0/24"
       "listen_limit": 10
       "auth_type": "md5",
       "password": "<passwd>"
       }
 }

* ``peer_group_id`` represents the UUID of the BGP peer group
* ``listen_range`` defines a prefix range to be associated with the peer group.
* ``listen_limit`` defines the maximum number of BGP peers that can be created
  automatically.

These are new fields to the existing BGP peer API and are optional parameters.
A BGP-Peer-Group can be re-used by multiple BGP Peers.

Changed Neutron Client Commands ((see [7]_)::

 openstack bgp peer create
               [--listen-range <listen-network-range>]
               [--listen-limit <number>]
               [--auth-type <auth-type>]
               [--password <password>]
               [--peer-group <peer-group-id>]
               <bgp-peer>

 openstack bgp speaker add peer mybgpspeaker bgpppeer1

``peer-group`` attribute is introduced in the already existing ``bgp peer``
neutron client command. It specifies the name/UUID of the BGP peer group that
has to be used by BGP Peer.

Create BGP peers with update-source and next-hop-self parameters API
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
As part of the enhancement, new attributes are introduced for the BGP peer
model. These are next_hop_self and update_source_ip. ``next_hop_self`` decides
whether to modify the nexthop attribute to its own during BGP advertisement.
And ``update_source_ip`` forces BGP to use the IP address specified while
talking to a BGP neighbor.

::

 POST /v2.0/bgp-peers/
 {
    "bgp_peer":{
       "auth_type":"none",
       "remote_as":"1001",
       "name":"bgppeer1",
       "peer_ip":"10.0.0.3",
       "next_hop_self":True,
       "update_source_ip": "10.2.0.15"
    }
 }

Changed Neutron Client Commands::

 openstack bgp peer create
               [--peer-ip <peer-ip>]
               [--next-hop-self True]
               [--auth-type <auth-type>]
               [--update-source-ip <ip>]
               --remote-as <as>
               <bgppeer>

``update-source-ip`` and ``next-hop-self`` are introduced in the existing
``bgp peer`` neutron client command.

Backend for reference implementation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

os-ken can be used to handle BGP (see [5]_ ).
os-ken is able to start bgp speaker using BGPSpeaker class and
it is possible to receive event notifications (``BGP_BEST_PATH_CHANGED``)
which can be then used to populate routes inside the namespace.


Data Model Impact
-----------------

``bgp_peers`` table will be updated to incorporate the new fields.

+---------------------+----------+------------------------------------------+
| Field               | Type     | Description                              |
+=====================+==========+==========================================+
| update_source_ip    | String   | Forces BGP to use the IP address         |
|                     |          | specified while talking to a BGP neighbor|
| next_hop_self       | Boolean  | Setting to True enables all published    |
|                     |          | routes to carry BGPSpeaker IP address    |
|                     |          | over the BGP Control Plane towards this  |
|                     |          | BGP Peer.                                |
+---------------------+----------+------------------------------------------+

New table ``bgp_speaker_router_bindings`` will be created to manage the
speaker to router association.

+------------------------+----------+---------------------------------------+
| Field                  | Type     | Description                           |
+========================+==========+=======================================+
| id                     | uuid-str | UUID of the BGP speaker router        |
|                        |          | association                           |
| bgp_speaker_id         | uuid-str | UUID of the BGP speaker               |
| router_id              | uuid-str | UUID of the router to which the BGP   |
|                        |          | speaker has to be associated          |
| advertise_extra_routes | Boolean  | Decides whether neutron extra routes  |
|                        |          | on the neutron router will be         |
|                        |          | advertised to bgp-peers               |
| status                 | String   | Shows whether Neutron-BGPaaS backend  |
|                        |          | software has realized the router      |
|                        |          | association successfully on the       |
|                        |          | openstack infrastructure. Status field|
|                        |          | can be either DOWN or ACTIVE.         |
+------------------------+----------+---------------------------------------+

Similarly, ``id`` and ``status`` field will be introduced in the existing
``bgp_speaker_peer_bindings`` tables.

New table ``bgp_peer_group``  will be created to manage bgp peer group.

+---------------------+----------+------------------------------------------+
| Field               | Type     | Description                              |
+=====================+==========+==========================================+
| id                  | uuid-str | UUID of the BGP peer group               |
| name                | String   | Human readable name of the BGP peer group|
| project_id          | String   | Owner of the BGP peer group              |
| remote_asn          | Integer  | Remote AS number of a BGP peer group     |
| update_source_ip    | String   | Forces BGP to use the IP address         |
|                     |          | specified while talking to a BGP neighbor|
| next_hop_self       | Boolean  | decides whether to modify the nexthop    |
|                     |          | attribute to its own during BGP          |
|                     |          | advertisement                            |
+---------------------+----------+------------------------------------------+

Security Impact
---------------
There can be security impacts with the introuduction of peer groups which
support automatic peering requests from peers within a pre-configured
listen-range. This is of major concern if we support this option for external
networks.

And this problem can be resolved using authentication which will be supported
in peer-groups.

Performance Impact
------------------
There can be performance impact based on the selection of the implementation
approaches.

Accurate testing is however needed to understand the overhead.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

 Manu B <manubk2020@gmail.com> (IRC: manubk)

Work Items
----------

* REST API update.
* BGP plugin, L3 agent extension to handle BGP:

  * Associate router to bgp speaker
  * Disassociate router from bgp speaker.
  * HA support for BGP speaker
  * Support new peer group APIs
  * New attributes for BGP peer APIs

* CLI update.
* Documentation.
* Tests and CI related changes.

Testing
=======

* Unit Test
* Functional test
* API test
  Perhaps this is something that can be easily tested end-to-end with fullstack
  tests. Need more investigation.

Documentation Impact
====================

User Documentation
------------------

New API and changes to legacy APIs like neutron-router and neutron-bgpaas must
be documented.

References
==========

.. [1]  https://tools.ietf.org/html/rfc4271
.. [2]  https://tools.ietf.org/html/rfc4271#section-8
.. [3]  https://docs.openstack.org/neutron-dynamic-routing/latest
.. [4]  https://www.sdxcentral.com/networking/nfv/definitions/virtual-network-function
.. [5]  https://opendev.org/openstack/os-ken/src/branch/master/os_ken/services/protocols/bgp/bgpspeaker.py#L225
.. [6]  https://docs.openstack.org/neutron/latest/admin/config-bgp-dynamic-routing.html
.. [7]  https://docs.openstack.org/neutron-dynamic-routing/latest/cli/index.html
.. [8]  https://docs.openstack.org/neutron-dynamic-routing/latest/contributor/testing.html
.. [9]  https://www.kernel.org/doc/Documentation/networking/vrf.txt
.. [10] https://docs.openstack.org/os-ken/latest/library_bgp_speaker_ref.html#os_ken.services.protocols.bgp.bgpspeaker.BGPSpeaker.rib_get
