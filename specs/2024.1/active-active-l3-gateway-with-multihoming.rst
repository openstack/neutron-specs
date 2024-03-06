..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================
Active-active L3 Gateway with Multihoming
=========================================

https://bugs.launchpad.net/neutron/+bug/2002687

Currently Neutron routers only support one external gateway port and ECMP
default routes can only be added manually as extra static routes. Likewise,
BFD is not configurable for ECMP routes. This specification provides an
extension to the existing Neutron API for configuring multiple external
gateways with automatic addition of ECMP default routes and BFD for those
routes. It also discusses the problem of scheduling multiple gateway ports per
router on different chassis.  OVN is chosen as a primary backend.

Problem Description
===================

Some network designs include multiple L3 gateways to:

* Share the load across different gateways: both in terms of different
  OVN chassis hosting different gateway ports and sharing the processing
  load and upstream gateways handling parts of the north-south flows;

* Provide independent network paths for the north-south direction (e.g. via
  different ISPs) for resiliency without relying on the same L2.

Having multi-homing implemented at the instance level imposes additional burden
on the end user of a cloud and support requirements for the guest OS, whereas
utilizing ECMP and BFD at the router side alleviates the need for instance-side
awareness of a more complex routing setup.

Adding more than one gateway port implies extending the existing data model
which was described in the `multiple external gateways spec`_. However, it left
adding additional gateway routes out of scope leaving this to future
improvements around dynamic routing. Also the focus of neutron-dynamic-routing
has so far been around advertising routes, not accepting new ones from the
external peers - so dynamic routing support like this is a very different
subject. However, manual addition of extra routes does not utilize the default
gateway IP information available from subnets in Neutron while this could be
addressed by implementing an extra conditional behavior when adding more than
one gateway port to a router.

ECMP routes can result in black-holing of traffic if the next-hop of a
route becomes unreachable. `BFD`_ is a standard protocol adopted by IETF
for next-hop failure detection which can be used for route eviction. OVN
supports BFD `as of v21.03.0`_ with a data model that allows enabling
BFD on a per next-hop basis by associating BFD session information with routes,
however, it is not modeled at the Neutron level even if a backend supports it.

Maintaining too many BFD sessions can have a performance impact as periodic
protocol messages are going to consume CPU cycles of a host. The implementation
will aim to minimize the amount of BFD sessions necessary per destination. One
way to do it is to reuse the BFD session information across routers' default
routes and extra routes if the peer endpoint and the rest of the session
configuration matches. However, care should be taken by operators when it comes
to the amount of routers they would like to use with BFD configured.

From the Neutron data model perspective, ECMP for routes is already a supported
concept since `ECMP support spec`_ got implemented in Wallaby (albeit the
spec focused on the L3-agent based implementation).

As for OVN and BFD, the OVN database state needs to be populated by Neutron
based on the data from the Neutron database, therefore, data model changes to
the Neutron DB is needed to represent the BFD session parameters.

Proposed Change
===============

DB Impact
---------

Core Model
^^^^^^^^^^

Currently there are two ways in which router to gateway port relationship is
expressed in Neutron:

* The ``gw_port_id`` foreign key in the ``routers`` table which is set to a
  UUID of the router port that has a type of ``network:router_gateway``;

* The ``routerports`` table which was `added for referential integrity`_
  purposes and stores ``router_id`` to ``port_id`` mappings along with a
  redundant ``port_type``.

In terms of the representation of multiple gateway ports in the Neutron DB the
proposal follows the `multiple external gateways spec`_:

* Keep the ``routers``, ``routerports`` and ``ports`` tables as they are now
  but start storing multiple ``network:router_gateway`` ports per router in the
  ``routerports`` table;

* For backwards-compatibility store a single gateway port id in
  ``routers.gw_port_id``.  The compatibility gateway port will be stored along
  with the other gateway ports in the ``routerports`` table.

* Extend the ``neutron.db.models.l3.Router`` class with a new attribute
  ``gw_ports`` that will map to all relevant ``network:router_gateway`` ports
  stored in the ``routerports`` table.

BFD and ECMP Route Behavior Modeling
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Each external gateway info dictionary of a router should contain additional
information about the policy around handling of default routes derived from the
subnets associated with the gateway ports of a router. This is needed so that
Neutron as a CMS has the required state to specify to OVN whether ECMP routes
are wanted and whether BFD needs to be enabled for them instead of just
passing those options through at the time of addition of an external gateway
to a router. Therefore, this information needs to be stored along the router.

The following additional columns are proposed for the
``router_extra_attributes`` table:

``enable_default_route_ecmp`` is a router-level policy on whether ECMP default
routes should be used or not (if the L3 service plugin supports them).

``enable_default_route_bfd`` is a router-level policy on whether BFD should be
used for checking whether the next-hop of a default route is reachable (if
the L3 service plugin supports BFD).

`OVN modeling of BFD`_ will be used to implement the support for BFD sessions
for default routes.

OVN allows making a BFD session for a particular static route to use a
different destination IP for checking reachability rather than the next hop of
the static route itself (by storing the BFD peer IP address in the ``dst_ip``
field in the ``BFD`` table). This is a useful semantic for separating the
control plane and data plane and can be used for static routes of Neutron
routers too. However, when it comes to default routes inferred from gateway
port to subnet associations, Neutron's behavior should be to use the next hop
of the default route as a ``dst_ip``.

OVN models the BFD table as follows (see `ovn-nb docs`_ for more information):

* ``dst_ip`` - a BFD peer IP address;

* ``min_tx`` - an integer specifying the minimum interval (milliseconds, >=1)
  that OVN would use when transmitting the BFD control packet (minus jitter);

* ``min_rx`` - an integer specifying the minimum interval (milliseconds)
  between the received BFD control packets that OVN is capable of supporting
  (minus the jitter applied by the sender). Can be set to 0 to state that
  BFD control packet transmission from the BFD peer is not desired;

* ``detect_mult`` - an integer (>= 1) specifying the detection time multiplier.
  The negotiated  transmit  interval, multiplied  by  this  value, provides the
  Detection Time for the receiving system in Asynchronous mode.

* OVN includes the ``options`` field that includes a map of string-string pairs
  which is reserved for future use. A similar field or extra columns can be
  added later to the Neutron model to account for additional extensions.

The proposed approach for this spec is limited to optionally enabling BFD for
inferred default routes only leaving a more advanced BFD model with per extra
static route BFD sessions for the future iterations (e.g. during the
implementation of the `BFD support spec`_ spec).


Rest API Changes
----------------

Router API
^^^^^^^^^^

The API changes augment the changes from the `multiple external gateways spec`_
but also include additional changes behavioral changes, thus, a different name
for an API extension is used in this specification:
``external-gateway-multihoming`` and API changes are listed here in full.

New attributes are also added as extra router attributes and the API is
extended to handle those in the ``router`` API resource (a separate extension
is added per attribute):

* ``enable_default_route_ecmp`` is a router-level policy on whether ECMP
  default routes should be used or not (if the L3 service plugin supports
  them).

* ``enable_default_route_bfd`` is a router-level policy on whether BFD should
  be used for checking whether the next-hop of a default route is reachable (if
  the L3 service plugin supports BFD).

With the ``external-gateway-multihoming`` extension a new router API resource
attribute is added called ``external_gateways`` which is a list of
``external_gateway_info`` structures.

The first element of the ``external_gateways`` list is special for
compatibility purposes as it contains the same information as the
``external_gateway_info`` does. When ``enable_default_route_ecmp`` is set on
a router to ``false`` it also defines the default gateway placed into the
routers routing table (while the OVN driver currently does not support routing
for the multi-segment network case, the placement of a gateway port would
matter for inferring the default gateway based on the subnet used on a network
segment).

The order of the rest of the list is ignored.

Duplicates in the list (that is multiple external gateways with the same
``network_id``) are allowed: in that case multiple gateway ports will be
attached to the same network (this can be used to have the active-active setup
when external gateways are available on the same network). However, attaching
multiple gateway ports to different networks with overlapping subnet ranges
will cause routing issues, so that kind of overlap is not allowed.

Updating ``external_gateway_info`` also updates the first element of
``external_gateways`` and it leaves the rest of ``external_gateways``
unchanged.  Setting ``external_gateway_info`` to an empty value removes a
single (compatibility) gateway from the set of gateway ports of a router and
chooses an existing extra gateway as a replacement for the compatibility
gateway instead.

The ``external_gateways`` attribute cannot be set in
``POST /v2.0/routers`` or ``PUT /v2.0/routers/{router_id}`` requests,
instead it can be managed via sub-methods:

* ``PUT /v2.0/routers/{router_id}/add_external_gateways``

  Accepts a list of ``external_gateway_info`` structures. Adding gateways to
  the same network is allowed provided that fixed IPs (if specified) are not
  used yet. If one or more gateways are present for a router already then
  the first item in the list for addition will become an extra gateway. If none
  are present, the first item will be treated as a compatibility gateway.

* ``PUT /v2.0/routers/{router_id}/update_external_gateways``

  Accepts a list of ``external_gateway_info`` structures.  The external
  gateways to be updated are identified by the ``network_id`` field and
  ``external_fixed_ips`` found in the PUT request. Updating ``enable_snat`` is
  only possible at the per-router basis on the first item specified. Updating
  ``external_fixed_ips`` is possible without recreating a port.

* ``PUT /v2.0/routers/{router_id}/remove_external_gateways``

  Accepts a list of potentially partial ``external_gateway_info``
  structures.  A combination of ``network_id`` and ``external_fixed_ips``
  fields from ``external_gateway_info`` structure is used to identify
  a particular gateway to be removed. Other keys can be present but
  their values are ignored.

The add/update/remove PUT sub-methods respond with the whole router
object just as ``POST/PUT/GET /v2.0/routers``.

Extra routes API: ECMP
^^^^^^^^^^^^^^^^^^^^^^

As the `ECMP support spec`_ notes, there are no API changes to make to support
ECMP routes per se: multiple routes to the same destination and different
next-hops can already be specified when adding extra routes. However, that spec
focused on the agent-based implementation - part of the work to implement this
spec is to check whether the same is true for the OVN-based L3 implementation.

Extra routes API: BFD
^^^^^^^^^^^^^^^^^^^^^

In the absence of a full BFD API, users will have an option to specify a policy
on the routers (``enable_default_route_bfd``).


OVN driver changes
------------------

In general, we will update the existing OVN driver to handle the presence of
multiple gateway ports wherever gateway ports are currently handled in the
existing code.  A few areas of interest are highlighted below.

Router level External IDs
^^^^^^^^^^^^^^^^^^^^^^^^^

There are a couple of router level external IDs in the existing implementation
which do not work with multiple gateway ports:

* ovn_const.OVN_GW_PORT_EXT_ID_KEY

* ovn_const.OVN_GW_NETWORK_EXT_ID_KEY

These will be deprecated and replaced by methods that look up the required
information at runtime.

L3 Scheduler Changes
^^^^^^^^^^^^^^^^^^^^

One of the main use cases for routers with multiple gateway ports is
resiliency.  Whenever there are multiple gateway ports present for a single
router, we want to ensure diverse placement of these ports across chassis to
minimize impact of chassis failure.

This will be implemented by updating the `leastloaded` scheduler to apply
soft anti-affinity when scheduling gateway ports for routers with multiple
gateway ports.

No changes will be made to the `chance` scheduler.

Out of Scope
============

* BFD session data model and API;

* BFD authentication as it is not implemented in the OVN BFD implementation
  while it is present in the protocol RFC itself. Therefore, the data model
  should be extensible to support this in the future;

* Enabling BFD for extra routes. For now the spec will only address the
  inferred routes leaving this for future iterations;

* Solving the distributed SNAT problem.
  One direction is to use conntrack state synchronization between the gateway
  ports. Other ideas involve making smarter control plane choices about where
  this conntrack state should exist instead of distributing it
  everywhere - this can be done by ensuring that processing of flows is done
  locally to the instance but there are downsides to that as well which needs
  to be considered more carefully

* Dealing with asymmetric routing:

  * Conntrack can be utilized to avoid responses generated by instances to go
    via the route different from the one the request came in on in presence of
    ECMP routes. OVN has support for making the reply traffic take the
    symmetric path.
    This can be configured by utilizing the `options column`_ in the logical
    router static routes table in OVN which allows configuring
    `ECMP symmetric reply`_ by setting ``ecmp_symmetric_reply`` option to
    ``true`` (it is modeled at the route level in OVN as well).

  * Routes in Neutron could have an ``ecmp_symmetric_reply`` option to specify
    a policy on whether to enable `ECMP symmetric reply`_ depending on whether
    the L3 service plugin supports it or not.

  * However, the `commit introducing the feature`_ in OVN notes a limitation on
    its use: it can only be used on gateway routers, not distributed routers
    that have a gateway port due to the dependency of the ingress pipeline
    logic of the logical router on the hypervisor-local CT state.

* Accepting ECMP routes via dynamic routing protocols. The current aim is to
  utilize the default gateway information available in Neutron subnets to
  configure default gateway ECMP routes or to use the extra routes extension.
  This specification is a building block for the future support of dynamic
  routing.

* Modeling of route metrics. While there are cases where one default route
  could be preferred over the other for the same destination, neither Neutron
  nor OVN model this concept today;

* Implementation of BFD for the non-OVN L3 implementation based on Linux
  namespaces.

Implementation
==============

Assignee(s)
-----------

* Frode Nordahl <frode.nordahl@canonical.com> (~fnordahl)
* Dmitrii Shcherbakov <dmitrii.shcherbakov@canonical.com> (~dmitriis)

Work Items
----------

* Add the new REST API by making neutron-lib and Neutron changes to the API,
  core Neutron and OVN integration code;

* Change the DB schema to add new attributes and create relevant DB migrations;

* Implement support for external-gateway-multihoming extension in the OVN
  driver.

* Update the OVN L3 `leastloaded` scheduler to apply soft anti-affinity when
  scheduling gateway ports for routers with multiple gateway ports.

* Update the CLI in order to utilize the newly added rest API;

* Update the relevant documentation;

* Implement relevant unit and functional tests using the existing facilities
  in Neutron.

.. _multiple external gateways spec: https://specs.openstack.org/openstack/neutron-specs/specs/xena/multiple-external-gateways.html
.. _BFD: https://www.rfc-editor.org/rfc/rfc5880
.. _as of v21.03.0: https://github.com/ovn-org/ovn/commit/6e0a69ad4bcdf9e4cace5c73ef48ab06065e8519
.. _ECMP support spec: https://specs.openstack.org/openstack/neutron-specs/specs/wallaby/l3-router-support-ecmp.html
.. _added for referential integrity: https://opendev.org/openstack/neutron/commit/93012915a3445a8ac8a0b30b702df30febbbb728
.. _OVN modeling of BFD: https://github.com/ovn-org/ovn/blob/v22.12.0/ovn-nb.ovsschema#L612-L636
.. _logical router routes with BFD records: https://github.com/ovn-org/ovn/blob/v22.12.0/ovn-nb.ovsschema#L449-L452
.. _ovn-nb docs: https://www.ovn.org/support/dist-docs/ovn-nb.5.txt
.. _options column: https://github.com/ovn-org/ovn/blob/v22.12.0/ovn-nb.ovsschema#L453-L455
.. _ECMP symmetric reply: https://github.com/ovn-org/ovn/blob/v22.12.0/ovn-nb.xml#L3312-L3319
.. _commit introducing the feature: https://github.com/ovn-org/ovn/commit/4fdca656857d4a5caeec35ae813888cb9e403e5e
.. _BFD support spec: https://specs.openstack.org/openstack/neutron-specs/specs/xena/bfd_support.html
