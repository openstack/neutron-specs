..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============
Routed Networks
===============

https://blueprints.launchpad.net/neutron/+spec/routed-networks


Problem Description
===================

Desired Use Case
----------------

In many cases, physical data center networks are architected with a
differentiation between layers 2 and 3.  Said another way, there are distinct
L2 network segments which are only available to a subset of compute hosts.
Routing allows traffic to cross these boundaries.  These topologies are used to
manage network resource capacity (IP addresses, broadcast domain size, ARP
tables, etc.) to allow for large scale networks.

Operators from the `Large Deployers Team`__ (LDT) and others would like an
operator-provisioned Neutron construct that is exposed to users and maps onto
this kind of physical network.  They would like this construct to represent an
L3 domain, in general having a larger scope than each of the physical L2
segments that contribute to it.  Users must be able to specify that VMs are
booted anywhere in such a domain - as opposed to on a particular physical L2
segment.  Consequently the only guaranteed connectivity between VMs on this
construct/domain is at L3; as the data path between any two VMs may involve IP
routing hops.  Fortunately many workloads only require L3 connectivity.

__ large-deployers-team_

These operators would like tenants to continue to think about and choose
Neutron networks (e.g. named like "dev" and "prod", "red" and "blue", etc.),
though ensuring that only L3 semantics are preserved.  Then, it would be up to
Nova and Neutron to work together to schedule the instance to a host and an L2
segment that is both L2 reachable from that host and has routable IPs
available.

Specifically, operators would like to solve this problem `without the use of
overlay networks`__.  Overlays work well with many smaller to medium scale L2
networks (in terms of number of ports) and can be overlayed on top of an L3
domain.  However, putting a large scale L2 overlay on top of an L3 domain
defeats the purpose of using an L3 domain in the first place.  They don't want
to require users to use individual L2 domains.  Consequently, to the users'
point of view, this use case looks like booting to a single shared network with
direct external connectivity.  It does not involve self-service topologies with
individual tenant networks and virtual routers.  These may come in the future
some of which is described in `Potential Future Work`_.

__ no-overlay-networks_

Why Current Model is Insufficient
---------------------------------

In Neutron, a Network (either a provider or a logical tenant network) is
currently assumed to represent an L2 broadcast domain.  This may make it
well-suited to modeling L2 segments as individual provider networks or when the
l2 segment is indeed a logical representation physically realized through an
overlay (i.e tunnels).  In the former case, the problem is that the user will
see a potentially long list of individual Networks without any clear reason to
distinguish them.  This is bad practice from a user experience point of view.
And, really, these segments are an implementation detail that doesn't need to
be exposed to users.

If we decide to break the above assumption and use a Network to model an L3
domain then we run in to other problems.  This spec details the solution for
these problems.  The problems and proposed solutions are detailed in the
`Proposed Change`_ section.

Proposed Change
===============

Overview
--------

The proposed solution begins with Neutron's `existing extension mechanism`__
for defining `multi-segment networks`__.  In short, these kinds of networks are
admin provisioned networks made by a collection of segments of potentially
different layer 2 technologies (vxlan, vlan, flat, gre, ...).  It can just as
easily be a collection of homogeneous segments (i.e. a single technology like
all vlans).

__ multi-provider-extension_
__ multi-segment-networks-doc_

These multi-segment networks are not widely implemented by many plugins yet.
ML2 implements them but many others do not.  Other plugins will need to
implement model changes to support multiple segments as described in this
specification.  The model currently provides no way to determine (let alone
control) whether or not the segments are bridged together to provide L2
reachability between them.

Segments
~~~~~~~~

The Network provides no way to model the affinity of Subnets to particular L2
segments.  All subnets are associated with the Network (the collection of
segments).

This spec proposes making the Segment a first-class object in Neutron, one with
an id.  This will allow Subnets to associate with it.  The association from the
Subnet to the Network will remain but an optional *segment_id* field will be
added to the Subnet to associate it with a particular segment.

A subnet without any segment id is still assumed to span all segments.  In
general, when provisioning a routed network, all of the subnets will be
associated with a segment.

When a provider Network is created, and the plugin supports the 'segments'
extension, a Segment object is automatically created to reflect how that
Network is implemented -- or a set of Segment objects, if the Network creation
used the multi-provider extension.

Subsequently, those Segment objects may be updated or destroyed by the cloud
admin, and further Segment objects may be added for the same Network, and the
underlying implementation of that Network should adjust accordingly.  In
practice not all possible 'adjustments' will be feasible, so we may need a way
of describing the Segment object changes that a given plugin can actually
support.

When a Network object is read, the 'provider:' and 'segments' properties
returned should reflect the Segment objects that are present at the time of the
read.

L2 Adjacency
~~~~~~~~~~~~

An attribute called *l2-adjacency* will be added to Neutron Networks.  This is
a boolean value where true means that you can expect L2 connectivity throughout
the Network and false means that there is no guarantee of L2 connectivity.
This attribute should be user visible to set that expectation.  The
implementation will not depend on this value, it is merely user facing.

This value will be read-only and derived from the current state of Segments
within the Network.  The value will be false if subnets on the network are
attached directly to segments.  It will be true otherwise.

Other plugins will be free to implement this according to their need.  For
example, Calico may just hard-code this to false and be done.  Plugins that
implement routed networks may make use of or mimick the reference
implementation's behavior.

DHCP Scheduling
~~~~~~~~~~~~~~~

DHCP scheduling is currently unaware of Subnet to Segment affinity.

The spec proposes changes to the DHCP scheduler when subnets are associated
with segments.  The scheduler will take care to ensure that all such segments
have a DHCP server.  If multiple DHCP servers are desired for redundancy, it
will schedule the desired number of servers to each segment.

For subnets that are not associated directly to a segment, scheduling will work
just like today, assuming that any segment in the network will suffice.  There
is a possibility with some plugins that such subnets exist when L2 adjacency is
False.  This spec doesn't solve this at this time.  Calico, one such plugin,
has solved this by running a DHCP on each compute host and some special
plumbing.

It is up to the operator to ensure that a DHCP agent is available with physical
connectivity on each of the segments.

Each server will be aware of the Subnets with affinity to its assigned segment.
The metadata proxy service will be available from each.

.. NOTE Access to the metadata service is normally provided by a Neutron
   logical router so that VMs access the metadata URL through their default
   route.  In this use case, the VMs' default routes are through a hardware
   router provided by the infrastructure, not a Neutron router.  Because of
   this, metadata needs to be provided by the DHCP namespaces.  This is already
   the fallback in Neutron for "isolated networks" (a fancy term for networks
   without a Neutron router as the gateway.)

Segment Host Mapping
~~~~~~~~~~~~~~~~~~~~

Until now, Neutron Networks have generally been assumed to reach all of the
compute hosts in the deployment.  With this proposal, that will continue to be
generally true.  However, for this use case, a given segment within the network
will only reach a subset of the hosts.  This is already possible with the ML2
plugin.  Further, there is already work being done to `utilize the mapping to
make DHCP aware of segments`__.  However, this needs to be formalized in a way
that any plugin or mechanism driver can provide it.

__ physnet-aware-dhcp_

Maintaining mappings between compute hosts and Segments will be the
responsibility of the Neutron plugin.  It could be handled using backend
specific configuration, or API primitives may be provided.

At a high level, we need to be able to do some form of the following mappings
efficiently.  These operations will be reflected in new optional plugin API
methods which can be implemented by any plugin.

- Segment -> hosts with L2 reachability to it.
- host -> Segment(s) it can reach over L2

For example, for agent-based ML2 deployments, the existing bridge mappings can
be leveraged.  In the current ML2 implementation, the necessary data is buried
in a JSON blob in a column making it opaque to DB queries.  This will have to
change.

To be precise, the *configurations* column in the `*agents* table`__ holds a
json blob.  The blob looks something like this (with extraneous data trimmed
out)::

    {...
     "bridge_mappings": {...}}

.. __: https://github.com/openstack/neutron/blob/b1999202b8/neutron/db/agents_db.py#L100

Each bridge mapping is (physnet, bridge).  The bridge can be ignored at this
point, but the physnet can be paired with the `host`__ for a host / physnet
mapping.  We can join this with the `network segment / physnet mapping`__ to
end up with a segment / host mapping.

.. __: https://github.com/openstack/neutron/blob/b1999202b8/neutron/db/agents_db.py#L87
.. __: https://github.com/openstack/neutron/blob/b1999202b8/neutron/plugins/ml2/models.py#L40

This spec introduces a new table, segment_host_mapping, to make this final
mapping explicit.  This table will be populated and updated as agents continue
to report reachability to physnets.  In ML2, this will only work for flat and
vlan type segments.

For agent-less ML2 implementations, another mechanism may need to be provided
to populate this table.  Non-ML2 plugins will also need to provide this
possibly leveraging the work already done.  Currently, there is only interest
in creating routed networks with vlan or flat networks.  We may need to
consider what happens if someone does try using it with other segment types.

Nova Scheduling
~~~~~~~~~~~~~~~

The boot workflow perceived by the user won't change regardless of whether
he/she chooses to boot from port or from network.  However, the user should be
aware that specifying an IP address at port creation time will constrain it to
the segment with the matching subnet thus limiting the set of compute hosts on
which they could land.  In most use cases, having a particular IP address won't
be the first thing that users want constraining their VM placement.  This
should be clear in documentation that this trade-off exists.

All subnets in a typical routed Network will be associated with a Segment.  So,
if no address is specified, Neutron will delay IP address allocation for this
port.  In other words, Neutron will not consider any Subnets with associations
to Segments for automatic IP allocation until the port is bound to a host.  It
will be okay to have a port without an IP address until then.

This work is under much more detailed discussion in a `Nova backlog spec`__.
This includes discussion of scheduling for live migration and how the scheduler
will consider IP subnets as resource pools.

__ nova-backlog-spec_

Binding
~~~~~~~

When scheduling is done, the port will be bound to the chosen host.  At this
point, Neutron can pick a suitable Subnet for the Port's address and do IP
allocation.

Potential Future Work
~~~~~~~~~~~~~~~~~~~~~

This section describes some potential follow-on work that is out of the scope
of this specification.

- Connecting Neutron routers to routed networks as their external gateway.
  This gets really fun with DVR routers.  It could also enable dropping the FIP
  namespace since the scale of each L2 network will be small (large scale L2
  networks was one motivation for having the FIP namespace)

- The Segment api could have a DHCP enabled flag to allow disabling DHCP for
  particular segments.

- DHCP relay could be implemented as an alternative to running a DHCP agent in
  each segment.

Data Model Impact
-----------------

#. Multi-provider network extension is foundation for modeling segmented L3
   networks
#. Segments are first class objects with Subnets associated to them

   - Migration will be provided for ML2 static segments (maybe no migration is
     required, `ML2 segments`__ already have all the data)

.. __: https://github.com/openstack/neutron/blob/b1999202b8/neutron/plugins/ml2/models.py#L26

#. Explicit denotation of l2 adjacency of segments within the same network

   - Read-only attribute derived from current state of segments
   - True for single segment networks, false otherwise

#. Implementation changes required to make DHCP work in non-adjacent
   multi-segment networks

   - May not require further modeling changes.

#. Formal modeling of host-to-segment mappings

   - New table with migration from agents/configurations json blob

#. Deferral of IP allocation for ports of multi-segment networks

   - May not require any new model changes


REST API Impact
---------------

Segment
~~~~~~~

Normal CRUD

- For admins, ability to list the hosts reachable on a Segment
- List Subnets associated with the Segment

Network
~~~~~~~

- l2-adjacency attribute described in `L2 Adjacency`_.

Subnet
~~~~~~

- segment_id attribute described in `Segments`_

Port
~~~~

- segment_id (An unbound port with an IP may still have affinity to a Segment)

Security Impact
---------------

This change isn't expected to pose any new security concerns.

Notifications Impact
--------------------

I think we can handle these as they come up.

Other End User Impact
---------------------

The client will be enhanced to support changes in the API.  Horizon will need
to at least expose the l2-adjacency attribute on the network.

Performance Impact
------------------

I don't expect this to effect performance significantly.  It enables an
L3-centric physical network topology that is proven to perform better at scale.

IPv6 Impact
-----------

This change will consider IPv4 and IPv6 equally.

Other Deployer Impact
---------------------

Deployers will not be required to do anything when upgrade code until they want
to make use of the new features provided by this spec.  In that case, they will
need to begin with their network architecture.

This spec is mostly targeted at greenfield deployments where operators have
ultimate flexibility to define the physical network topology in an L3-centric
manner.

Having said that, existing deployments can make use of this new feature after
upgrading their code by deploying a new Neutron network.  They might create new
VLANs, turn on routing features in their top-of-rack switches, add routers, and
whatever else needs to be done to create an L3 network for the new network.  In
some cases, it can be done just by configuring existing physical devices.

Either way, I consider this a greenfield deployment from the perspective of the
Network.

There will be operators who have a large flat network, or a collection of
Neutron networks already routed together who want to convert to using a single
multi-segmented Neutron Network with *l2-adjacency* set to false.  I would like
to discuss a migration plan with those operators.  For now, I don't have a
complete picture for this.

Developer Impact
----------------

This change will have an effect on developers.  This blueprint contains API and
model changes that plugin providers will need to look at implementing.
Specifically, they will want to look at implementing `L2 Adjacency`_,
`Segments`_, `Segment Host Mapping`_.

Community Impact
----------------

Yes.  This change is already a highly visible change that has been discussed on
the ML, in Neutron meetings (especially L3), and at various design summits.
Most notably, in a large deployment team session in Vancouver and in a design
session in Tokyo.

Alternatives
------------

The community had a lengthy discussion about an alternative.  It avoided adding
a new object to the model by using instances of a Network for an L3 domain.  A
Neutron router would've been created to associate a "front" L3-only network to
any number of "backing" networks representing the L2 segments.  This
alternative acknowledged that there was L2 and L3 mingled together in the same
object.

Provider networks allow much of what is described in this specification.  In
fact, the kind of Network described here is a lot like a provider network.
This spec needs to enhance the provider network idea to support the kind of L3
architecture described.

Another alternative discussed at length was to add a new IpNetwork construct to
represent L3 networks.  This is how I would do it if I were to model Neutron
all over again but there isn't a smooth path to get there from where we are
now.  The current proposal is a compromise which appears to have a much better
migration path.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

* `Carl Baldwin <https://launchpad.net/~carl-baldwin>`_

Other contributors:

* `Miguel Lavalle <https://launchpad.net/~minsel>`_
* `Neil Jerram <https://launchpad.net/~neil-jerram>`_
* `Vikram <https://launchpad.net/~vikschw>`_
* `Swami <https://launchpad.net/~swaminathan-vasudevan>`_
* `Ryan Tidwell <https://launchpad.net/~ryan-tidwell>`_
* `Reedip Banerjee <https://launchpad.net/~reedip-banerjee>`_
* `Cedric Brandily (ZZelle) <https://launchpad.net/~cbrandily>`_
* `Kevin Benton <https://launchpad.net/~kevinbenton>`_
* `Hirofumi Ichihara <https://launchpad.net/~ichihara-hirofumi>`_
* `Brandon Logan <https://launchpad.net/~brandon-logan>`_
* `Richard Theis <https://launchpad.net/~rtheis>`_ (Openstack client)

Please add your name here and attend the `Routed Networks Meeting
<https://wiki.openstack.org/wiki/Meetings/Neutron-Routed-Networks>`_ if you'd
like to contribute.  If your name is here and you don't have any time to
contribute, let me know.  This section is really just my sticky note of
contributors who can play a role in this effort.

Work Items
----------

#. Admin-only Segments extension

   - For ML2, use the multi-provider extension data model.
   - API tests
   - Client support
   - Add segment_id to Subnet

     - Client support

   - Add segment_id to Port

     - This represents the segment_id where IPs are allocated from and isn't
       quite the same as the Segment associated through port binding.

     - This should be consistent with any segment_id associated through an
       `IPAllocation`__.

.. __: https://github.com/openstack/neutron/blob/88b821c76e/neutron/db/models_v2.py#L92

#. Defer IP allocation for ports of multi-segment networks

   - Without a segment, a Port should only get an IP address if there are
     subnets with no segment association.
   - Nova validates that the port has an IP.  This must be removed (preferred)
     or deferred until after host binding.

#. Host / Segment Mapping

   - Update mapping when bridge mappings are received from the agent
   - Expose through API

     - The list of hosts available to a Segment will be available through the
       new segment API.

#. Maintain host aggregates and create scheduler resource pools in Nova.

   - These should be kept in lock-step with the Host / Segment mappings

#. In Nova boot / live migrate, tie the segment_id of an existing Port to a
   resource pool.

   - In the scheduler table, there is an "external id" in the resource pool
     which needs to be matched with the segment_id in the existing Port.  I'm
     not sure how Nova will notice the existing segment_id on a Port and match
     it.  See the `Nova spec`__ for more details.

.. __: https://review.openstack.org/#/c/263898/4/specs/backlog/approved/neutron-routed-networks.rst@148

#. DHCP scheduler and agent

   - Schedule segments with associated subnets to agents with reachability
   - Ideally, an agent should only be configured to serve subnets on the
     segment its attached to (in addition to subnets without association to any
     segment)

#. Add L2 adjacency to Network

   - Read-only attribute derived from current state of segments
   - True for single segment networks, false otherwise


Dependencies
============

The `Nova spec`__.  This will require coordination with Nova, especially with
the scheduler team.

__ nova-backlog-spec_

Testing
=======

Tempest Tests
-------------

There will be changes to tempest tests.

Functional Tests
----------------

Unknown

API Tests
---------

- Segment resource (CRUD)
  - its relationship to Subnets and Networks
- Network l2-adjacency flag (read-only through API)
- Host / Segment mapping (read-only through API)


Documentation Impact
====================

Yes

User Documentation
------------------

Document the l2-adjacency flag on the Network

Developer Documentation
-----------------------

Yes

References
==========

.. _rfe: https://bugs.launchpad.net/neutron/+bug/1458890
.. _my-blog-post: http://blog.episodicgenius.com/post/neutron-needs-l3-model/
.. _networking-calico: http://docs.openstack.org/developer/networking-calico/
.. _no-overlay-networks: https://bugs.launchpad.net/neutron/+bug/1458890
.. _multi-provider-extension: https://github.com/openstack/neutron/blob/master/neutron/extensions/multiprovidernet.py
.. _multi-segment-networks: https://wiki.openstack.org/wiki/Neutron/ML2#Multi-Segment_Networks
.. _multi-segment-networks-doc: https://bugs.launchpad.net/openstack-api-site/+bug/1242019
.. _large-deployers-team: http://lists.openstack.org/pipermail/openstack-operators/2015-May/007080.html
.. _routed-networks-brainstorm: https://etherpad.openstack.org/p/routed-networks-brainstorm
.. _vm-with-unaddressed-port: https://review.openstack.org/#/c/239276
.. _physnet-aware-dhcp: https://review.openstack.org/#/c/205631
.. _nova-backlog-spec: https://review.openstack.org/#/c/263898/
