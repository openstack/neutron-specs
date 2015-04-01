..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Add availability zones for agents
=================================

https://blueprints.launchpad.net/neutron/+spec/add-availability-zone

Implement availability zones for the DHCP and L3 agents.  Just like
Nova and Cinder, this allows users to specify where the network
services run, giving better fault isolation.


Problem Description
===================

Nova and Cinder have availability zones today.  Cloud administrators can assign
availability zones to physical nodes. Each of the nodes generally is equipped
different power sockets, network switches, cooling devices and others.  By
properly choosing from the provided availability zones, users can minimize
their chance of service failures.

But, as Neutron doesn't have availability zones, there's no way to put network
services under distinct availability zones as a VM instance or a VM volume.
What happens with Neutron is completely by chance today. A user has risk of
higher probability of network failure because the user cannot allocate network
resources to availability zones for high availability.  Also, network traffic
can go through long paths between availability zones.  DVR and L3 HA can
mitigate these issues somewhat, but they don't entirely solve the problem since
DVR still need central SNAT router, which need to be HA capable, and L3 HA
is not aware of underlying hardware configuration to be HA as a system
(i.e. not only assigning routers to "other node" but to the "appropriate node
(or group of nodes)").

Note: This spec focuses on high availability of network resources. This spec
does NOT address the scalability issue of process communications related to
cell discussion nor underlying network topology related to network segment
discussion.

Proposed Change
===============

This change introduces the concept of an "Availability Zone" into Neutron. In
particular, an availability zone is an optional attribute for Network and
Router resources.  These attributes in no way affect the behavior of Neutron in
terms of allowed logical network connectivity.  These attributes are simply
used as hints to the backend about the location of other resources (compute and
storage) that will be using these network resources.  The Neutron backend may
be able to use this to optimize its dynamic placement of resources to improve
performance and/or ensure resources are placed in the same defined failure
domain.

Create a new extension called availability_zone.

* The extension adds a new API that lists availability zones.

The rest of this information applies to the implementation of availability
zones for the built-in reference backend.

The extension adds the availability_zone attribute to Agent DB models. It also
adds availability_zones and availability_zone_hints arrtibute for Network and
Router DB models.  The corresponding API resources will see the availability_zone
attribute, too.

The new config options availability_zone and default_availability_zones are
added. Availability zone of each agent is set by the availability_zone config
parameter in each configuration file. If availability_zone parameter is not
given in agent config, the agent is assigned to the default availability zone
named "nova". The name "nova" is referred to availability zone of Nova and
Cinder. When a user executes resource create API without availability zone
attribute, neutron set default_availability_zones value to the resource. The
default_availability_zones value can be blank. If that’s the case, the
scheduler selects any agent from any availability zone without any preference
of specific availability zone. This helps to avoid the unbalance of resource
assignment.

API and config are arranged to the following.

* Using config, deployer specifies which availability_zone an agent
  belongs to, and they can also define default availability zones for
  user resources.
* Using GET API of availability zone, users can get all the
  availability zones which neutron manages. API of availability zone
  is "GET" only.
* Using GET API of network resources, users can get which availability
  zones their network resources is assigned.
* Using POST API of network resources, users can create a network resource with
  availability zone hints as candidate for availability zone which the
  resource belongs to.

This spec enables each resource to belong to multiple availability zones. A
user is able to specify the list of multiple AZs as a parameter when a resource
is created. The list of multiple AZs defines the candidates of availability
zone where the resource may be deployed. If the parameter at the creation is
not given and the default_availability_zones config is not specified, the
resource can be deployed at any availability zone. In other words, the list of
multiple AZs for a resource restricts the scope of the deployment. Therefore,
we can get redundancy by scheduling a network or a router to two agents in two
distinct availability zones. Scheduler is also improved so that routers and
networks are properly allocated with availability zone.

Limitations: With the reference L3 implementation without HA, we apparently
cannot assign a router to multiple L3 agents and as a result we cannot achieve
pure high availability from availability zone. A user just has an expectation
of failure domain by setting availability zone to non-HA router. With L3-HA
enabled router in the reference L3 implementation, all L3 agents across
availability zones still need to have the connectivity to an external network
uniformly to achieve high availability deployment.

Future work: It is definitely expected that all other services in neutron such
as lbaas, fwaas, vpnaas and so on are able to handle the availability zone as
its attributes. As these haven’t supported HA capability in the reference
implementation yet, I suggest to implement them separately in another spec by
step-by-step approach, hopefully almost concurrently with this spec.

Data Model Impact
-----------------

As noted above, the spec adds availability_zone attribute to DB. A migration
script will be provided.  When operators update config, neutron checks
different availability zone between resources and agents, then outputs some
logs.

Attribute will be added:

Availability_zone attribute to RouterExtraAttributes

.. csv-table::
    :header: Attribute,Type,Description

    availability_zone_hints, String, availability zone candidate for the router
    availability_zones, String, availability zone for the router

Availability_zone attribute to NETWORKS as extend

.. csv-table::
    :header: Attribute,Type,Description

    availability_zone_hints, String, availability zone candidate for the network
    availability_zones, String, availability zone for the network

Availability_zone attribute to Agent

.. csv-table::
    :header: Attribute,Type,Description

    availability_zone, String, availability zone for the agent


REST API Impact
---------------

* /agents

'availability_zone' key is added to 'configurations' attribute(dict). Note that
'configurations' attribute is read only.

* /networks and /routers

The following attribute is added.

.. csv-table:: New attribute
    :header: Attribute Name,Type,Access,Default Value,Validation Conversion,Description

    availability_zone_hints,list of string,"RW(POST only), all",[],list of string,list of human-readable name
    availability_zones,list of string,"RO, all",[],list of string,list of human-readable name

* /availability_zones

The extension introduces a new availability_zone API resource. Only GET is available.

.. csv-table::
    :header: Attribute Name,Type,Access,Default Value,Validation Conversion,Description

    availability_zones,list of dict,"RO, all",N/A,N/A,see example below

An example of a JSON response:

::

  {
      "availability_zones": [
          {
              "name": "nova",
              "state": "available"
          }]
  }


Security Impact
---------------

None.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

python-neutronclient and horizon will support new availability_zone value.

Performance Impact
------------------

None.

IPv6 Impact
-----------

None.  This proposal is protocol agnostic.

Other Deployer Impact
---------------------

To make use of this feature, deployers need to set availability_zone in the
each configuration file(e.g. l3_agent.ini and dhcp_agent.ini), specifying each
network node's availability zone.

The spec expects deployer to set an availability zone to an agent by config file
since availability zone is related to a place of power socket and fixed
equipment. However it doesn't block new feature connected with availability zone
from providing API, which enables deployer to specify availability zone without
the config. It includes feature managing physical resources like
Host_aggregation, Cell and others.

Upgrade Impact
---------------------

Agent side: Before the upgrade, all agents are considered to be in the default
availability zone named “nova.” Once an operator configures availability zone
config parameter ‘availability_zone‘ in its agent config file and the agent
is restarted, the agent belongs to the availability zone set in the config
file. If an operator sets “nova” to the parameter, it means same as the
default availability zone.

Resource side: Before the upgrade, all resources are considered to be at any
availability zone.  Even though an operator changes the availability zone of
agents, it doesn’t break the matching to existing resources on the agent.

Developer Impact
----------------

None.

Community Impact
----------------

None.

Alternatives
------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Hirofumi Ichihara <ichihara-hirofumi>

Secondary assignee:
  Iwamoto Toshihiro <iwamoto>

Work Items
----------

* Add availability_zone to the DB models
* Make agents report their availability_zone settings
* Add the availability_zone extension
* (Validate REST API availability_zone parameters)
* Add AvailabilityZoneFilter based on existing neutron scheduler implementations
* Modify the L3(non-DVR and dvr_snat router) and DHCP agent schedulers to be AZ aware
* Modify the L3(HA router) agent schedulers to be AZ aware
* Add availability zone to python-neutronclient(Volunteers needed)
* Add availability zone to horizon(assignee: amotoki)

Dependencies
============

None.

Testing
=======

Tempest Tests
-------------

None.

Functional Tests
----------------

Add tests, which ensure resources are allocated for proper availability
zone. Two new tests will be added for the following resources:

* Network availability zone
* Router availability zone

API Tests
---------

Tests for the new attribute and the new API resource will be added.

Documentation Impact
====================

User Documentation
------------------

The new config options will be documented. Availability zone use cases and the
usage will be documented in the devref.

Developer Documentation
-----------------------

None.

References
==========

* Nova availability zone
* Cinder availability zone
* An implementation of this blueprint
  https://review.openstack.org/#/c/183369/
