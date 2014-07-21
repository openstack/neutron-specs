..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================================================
Advanced Network Services Specification for Service Chaining
=========================================================================

URL of the launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/neutron-service-chaining

One common use-case for using multiple services can be described as a
"service-chain" - that is, application of specific services in a specific
order for every packet on a specific datapath. This API provides an
abstraction to specify that behavior with clear definition of the expected
semantics.

By design, this specification does not mandate the technology used to provide
that service. It may be provided using the service insertion [3]_ and traffic
steering [4]_ functions (as specified in the blueprint above), or by
the existing advanced services in neutron, or by a multi-function appliance.

The goal of this specification is to provide the user with a tool that
captures their high level intent without getting coupled with incidental
details of their specific deployment. This then provides a path for migrating
such services across technology changes or for hybrid deployments.

Problem description
===================

Typical scenarios for combining services can usually be specified quite
succinctly (as traffic on this interface must first inspected by a firewall
and then processed by a loadbalancer). Unfortunately specifying that can many
times get mired in incidental complexity around service insertion and traffic
steering issues. If the user was able to describe that intent, we could
orchestrate the required service lifecycle events and steer the traffic as
required without exposing that complexity (and the resulting coupling)
outside the specific implementation details.

This has the added benefits that:

1. As we are only specifying the intent, it is back-end technology agnostic
2. The additional information allows us to provide technology upgrade without
   breaking service usage from a user perspective
3. It can support hybrid deployments, or migrations across vendors, even
   when the underlying technology used by those vendors is different.

But there is a reason that service insertion and traffic steering are
required. Specifying the most general case requires that level of detail.
We make the claim that a very large subset of multi-service users do not
need that general purpose specification and this simpler API should be
used in those cases.

Also, when specifying abstractions that are required to be implemented across
technologies, it is critical that the semantics that are expected by the API
(or implied) be clearly defined so that the usage can actually be portable
across those technologies.

Proposed change
===============

1. An API to specify a service-chain
2. An API to instantiate/delete an instance of a service-chain
3. Database updates (new resources) to support that API
4. Configuration of service-chain-providers in ini file

Alternatives
------------

As identified in the problem statement, service insertion and traffic steering
provide a more general solution. This is a simpler use-case that does not
require those technologies to be enabled (and be provided even when using the
existing advanced services in Neutron). When traffic steering is available,
it can be used as a provider for this API.

Data model impact
-----------------

The following new resources are proposed:

1. ServiceChainNode

+--------------+-------+---------+----------+-------------+---------------+
|Attribute     |Type   |Access   |Default   |Validation/  |Description    |
|Name          |       |         |Value     |Conversion   |               |
+==============+=======+=========+==========+=============+===============+
|id            |string |RO, all  |generated |N/A          |identity       |
|              |(UUID) |         |          |             |               |
+--------------+-------+---------+----------+-------------+---------------+
|name          |string |RW, all  |''        |string       |human-readable |
|              |       |         |          |             |name           |
+--------------+-------+---------+----------+-------------+---------------+
|type          |string |RW, all  |required  |foreign-key  |service-type   |
|(flavor?)     |       |         |          |             |               |
|              |       |         |          |             |               |
+--------------+-------+---------+----------+-------------+---------------+
|config        |string |RW, all  |''        |string       | service       |
|              |       |         |          |             | configuration |
|              |       |         |          |             | (as a HEAT    |
|              |       |         |          |             | template)     |
|              |       |         |          |             | [5]_          |
+--------------+-------+---------+----------+-------------+---------------+
|config_params |string |RW, all  |''        |string       | configuration |
|              |       |         |          |             | parameters    |
|              |       |         |          |             | (for a HEAT   |
|              |       |         |          |             | template)     |
|              |       |         |          |             |               |
+--------------+-------+---------+----------+-------------+---------------+

2. ServiceChainSpec

+------------+-------+---------+----------+-------------+-----------------+
|Attribute   |Type   |Access   |Default   |Validation/  |Description      |
|Name        |       |         |Value     |Conversion   |                 |
+============+=======+=========+==========+=============+=================+
|id          |string |RO, all  |generated |N/A          |identity         |
|            |(UUID) |         |          |             |                 |
+------------+-------+---------+----------+-------------+-----------------+
|name        |string |RW, all  |''        |string       |human-readable   |
|            |       |         |          |             |name             |
+------------+-------+---------+----------+-------------+-----------------+
|nodes       |string |RW, all  |required  |list of      |list of          |
|            |       |         |          |strings      |ServiceChainNode |
|            |       |         |          |(UUIDs)      |                 |
+------------+-------+---------+----------+-------------+-----------------+

3. ServiceChainInstance

+-------------------+-------+---------+---------+---------------------+---------------+
|Attribute          |Type   |Access   |Default  |Validation/          |Description    |
|Name               |       |         |Value    |Conversion           |               |
+===================+=======+=========+=========+=====================+===============+
|id                 |string |RO, all  |generated|N/A                  |identity       |
|                   |(UUID) |         |         |                     |               |
+-------------------+-------+---------+---------+---------------------+---------------+
|name               |string |RW, all  |''       |string               |human-readable |
|                   |       |         |         |                     |name           |
+-------------------+-------+---------+---------+---------------------+---------------+
|service-chain-spec |string |RW, all  |required |foreign-key          |service-chain  |
|                   |       |         |         |for ServiceChainSpec |spec for this  |
|                   |       |         |         |                     |instance       |
+-------------------+-------+---------+---------+---------------------+---------------+
|port               |string |RW, all  |required |foreign-key          | neutron       |
|                   |       |         |         |                     | port          |
|                   |       |         |         |                     |               |
+-------------------+-------+---------+---------+---------------------+---------------+

SEMANTICS:

The expected semantics would be equivalent of:

1. As if the services were bump-in-the-wire services
2. As if the services were applied on the identified neutron port
   NOTE: This is not the service-port or entry-point to the chain,
   this is just specifying that the service chain needs to be
   applied to all traffic that is traversing this specific port.
   The provider may implement it using any valid insertion strategy.
3. In the order of ServiceChainNodes in the ServiceChainSpec for
   inbound traffic, and in opposite order for outbound traffic
4. Not all providers will honor arbitrary ordering of services
   or arbitrary neutron port for application of the service.
   In that case, the provider will raise a "NotImplemented"
   exception.

NOTES:

1. In future the port attribute could be extended to support other
   integration points (like the service port or the external port),
   we will extend/review that use-case once those interfaces are
   available in neutron.

USAGE WORKFLOW:

1. Assume a topology with Neutron Networks N1 and N2 connected to
   a Neutron Router R1 which is itself connected to an external
   network E1. Assume that the neutron port on E1 that connects to
   R1 is P1.
2. Assume that the semantics that I want to provide are of having
   all traffic to/from external network to the router (and hence
   to N1 and N2) needs to be (a) first inspected by a firewall,
   and then (b) load balanced by a load balancer.
3. Then I would create a ServiceChainSpec with 2 ServiceChainNodes.
   The first node would be of type FW and the second one LB.
   The FW node would have config string as the HEAT template for
   FWaaS configuration and the LB would have the config string as
   the HEAT template for the LBaaS configuration. CLI for that
   would look like::

       neutron servicechain-node-create --type flavor_id --config_file fw_heat_template --config_params "destion1=IP1;destination2=IP2" fw_node
       neutron servicechain-node-create --type flavor_id --config_file lb_heat_template --config_params "router=router_uuid" lb_node
       neutron servicechain-spec-create --nodes "fw_node;lb_node" fwlb_spec

   This creates the ordered-list ["FW", "LB"] as the list of services in the
   chain.

4. Finally I would create a ServiceChainInstance from this ServiceChainSpec
   and associate it with the neutron port P1 (as the attribute port). CLI for
   that would look like::

       neutron servicechain-instance-create --servicechain_spec_id fwlb_spec --port_id P1 service-chain

   This creates a chain that applies services in the order:

   * FW->LB->R1 for ingress traffic, and
   * R1->LB->FW for egress traffic.


REST API impact
---------------

1. CRUD for ServiceChainNode
2. CRUD for ServiceChain
3. CRUD for ServiceChainInstance

Security impact
---------------

CRUD API is provided using existing API model, no new surface is exposed.

Service/Service configuration is provided by underlying services,
so no new surface is exposed.

Notifications impact
--------------------

1. All updates to service-chain-spec resources need to be relayed to the
configured service-chain-providers

2. Updates to ServiceChainNode or ServiceChainSpec need to generate
notification to backend to "fixup" the ServiceChainInstances as required.

3. It is assumed that the existing notifications exception handling
meets the needs for this API and no new constructs are specified.

Other end user impact
---------------------

1. The CLI/UI impact of this new API (not captured in this blueprint)

2. Additional configuration for service-chain-providers in ini file
   (configuration of service-chain-providers will be specific to
   service-chain-providers and is not in the scope of this BP).

Performance Impact
------------------

No significant performance impact is expected.

Other deployer impact
---------------------

No other deployment impacts are expected

Developer impact
----------------

Devstack will have to be updated for service-chain-providers.

Implementation
==============

Assignee(s)
-----------

The following people are working on several different aspects of the proposed
framework:

  Mandeep Dhami (mandeep-dhami)

  Subrahmanyam Ongole (osms69)

  Sumit Naiksatam (snaiksat)

  Prasad Vellanki (prasad-vellanki)

Work Items
----------

1. Build API
2. Update Datamodel
3. Build unit-tests
4. Update documentation

Dependencies
============

Dependencies between advanced services framework are captured in [2]_

Testing
=======

Unit Tests will be provided.

Documentation Impact
====================

Documentation will need to be updated for:

1. Services chain model and usage
2. Configuration of service-chain-providers

References
==========

.. [1] Advanced Services Meetings
   https://wiki.openstack.org/wiki/Meetings/AdvancedServices

.. [2] Common Framework Umbrella blueprint:
   https://review.openstack.org/#/c/92200/4

.. [3] Service Insertion blueprint:
   https://review.openstack.org/#/c/93128

.. [4] Traffic Steering blueprint:
   https://review.openstack.org/#/c/92477

.. [5] Openstack Heat
   https://wiki.openstack.org/wiki/Heat
