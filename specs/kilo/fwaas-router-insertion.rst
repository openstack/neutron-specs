..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
FWaaS Insertion Model on Routers
================================

https://blueprints.launchpad.net/neutron/+spec/fwaas-router-insertion

The blueprint proposes  a change to the FWaaS insertion model to associate a
Firewall with a specified set of Routers (can be a single Router) as opposed
to the current model of association with all Routers in the tenant.

Problem Description
===================

The FWaaS model defines 3 resources: Firewall rules, Firewall policies and
Firewalls. The Firewall rules has attributes to specify a filter rule. The
Firewall policy is a container for a collection of rules. The Firewall itself
is associated with a Firewall policy and thereby indirectly to a set of rules.
Currently no insertion point can be defined for a Firewall, and there is no
intent to abstract the insertion from the service itself to allow for various
types of insertion points(L3, Bump in the wire etc).

The FWaaS reference implementation inserts the Firewall on all 'qr-' interfaces on
all routers in the tenant. The plan was to allow the insertion point to be driven by
the Service Insertion proposals[1][2]. Given that the Service Insertion proposals
have not yet been adopted for various reasons, the objective of this effort is to
restrict the scope to FWaaS and insertion on a specified set of Routers.

In the current model and implementation we can only support a single Firewall on a
tenant as the Firewall is present on all Routers. This does not provide the ability
to selectively firewall traffic for a subnet or a set of subnets in the tenant topology.
This requires all Rules for a tenant topology to be collected together in the policy
associated with the Firewall. This is both inefficient and prone to errors.

Routers are not tracked on the Firewall DB so it is not always straightforward to reflect
the Firewall state based on the state transtions on one of potentially many Routers
associated with the Firewall. We can handle the case when a new Router is added but it
is more involved to handle Router deletions.

Proposed Change
===============

It is being proposed to associate a Firewall with a set of Routers which is specified
in the API. Resource validation can be performed and appropriate action can be taken
by the Plugin. The insertion point as the Router(s) will be tracked in the Firewall DB.

With some trepidation the additional attribute can be proposed as an extension
or this can be added as an attribute to the Firewall extension. With the rework
happening in this area, this effort will align with the community direction here.

A minor variation with this same fundamental intent of the Firewall being associated
with a set of Routers is more favored based on discussions with members of the FWaaS
subteam and associated individuals with interactions with Customers and Deployers.
Rather than just associating a set of Routers and implicitly on all their interfaces,
an option to selectively apply to a specified set of the Router interfaces across the
Routers is also proposed as a phase 2 of the implementation. This can support backends
that might need the Firewall association with a list of ports. The default behavior
is to  apply on all interfaces of the Router(s).

With this variation on the theme, the  list of interfaces in addition to the Router will
be accomodated as an attribute to the Firewall extension or proposed as an extension.

With this change we can support multiple Firewalls within a tenant with better
discrimination based on the Router insertion point rather than have all rules for the
entire topology applied every where. The constraint of a port being a member of a single
Firewall will be enforced for consistent behavior. So with the default of applying on all
interfaces of a Router - the constraint will imply that a Router can only accomodate a
single Firewall. With the port list, to apply on a subset of Router interfaces, we can
support multiple Firewalls on a Router as well.

Data Model Impact
-----------------

The proposal is to associate the Firewall with the list of Routers provided in the
Create or Update workflows. Appropriate Foreign Key constraints will be applied.

+-------------------+-------+-----------+------+------------------------+
| Attribute name    |  Type | Required  | CRUD | Description            |
+-------------------+-------+-----------+------+------------------------+
| fwid              | uuid  | Y         |  R   |Firewall id             |
+-------------------+-------+-----------+------+------------------------+
| router_ids        | list  | Y         | CRU  |Associated Routers      |
+-------------------+-------+-----------+------+------------------------+

The port-id list attribute will be added to the above model as Phase 2 of
the implementation.

Database Migration:
The current implementation installs the Firewall on all Routers in the tenant. The
Routers are not tracked in the Firewall Database. On upgrade, as part of the migration
it is proposed that each Router in the tenant will be added as a Router associated with
the Firewall as in the new Data Model. Downgrade will not be supported.

REST API Impact
---------------

This would be the addition to the Firewall resource. As mentioned earlier, an approach
consistent with the extensions rework will be adopted. Support for Routers is added in
Phase 1.

.. code-block:: python

  RESOURCE_ATTRIBUTE_MAP = {
    'firewalls': {
        'router_ids': {'allow_post': True, 'allow_put': True,
                      'validate': {'type:uuid_list': None},
                      'is_visible': True},
    }
  }

And the list of ports as targeted for Phase 2.

.. code-block:: python

  RESOURCE_ATTRIBUTE_MAP = {
    'firewalls': {
        'router_ports': {'allow_post': True, 'allow_put': True,
                           'validate': {'type:uuid_list': None},
                           'convert_to': attr.convert_none_to_empty_list,
                           'default': None, 'is_visible': True},
    }
  }


Security Impact
---------------

None.

Notifications Impact
--------------------

The FWaaS Agent will not need to consume all router_add events as in the current model.
No changes to any messaging is planned.

Other End User Impact
---------------------

In the current model, FWaaS does not specify any insertion point. The API in the
proposed model will accomodate a set of Routers as an additional attribute . As
in the current model, the Firewall state will start in PENDING_CREATE (or CREATED in
DVR mode) and will go to ACTIVE with successful completion of messaging with the FWaaS
Agent and the iptables backend. The CLI representations below are some possible
suggestions.

neutron firewall-create FW-POLICY --router <r1> --router <r2>

neutron firewall-create FW-POLICY --routers "<r1> <r2> ... <rm>"

And this implicitly installs the Firewall on all internal interfaces of each
router specified.

If this new attribute is not specified on firewall-create, the firewall logical
resource will be created and be in PENDING_CREATE (or CREATED in DVR mode) state. This
enables decoupling the Firewall Resource creation from Router creation. This can support
a scenario where Firewalls can be pre-provisioned and can later be bound to specific
Routers which can get created at a later point on a tenant network.

This is similar to the current model when there are no routers present in the tenant.
With this proposal, the Firewall state will change to ACTIVE on a firewall-update with
the insertion point details. The CLI representation below is one possible suggestion.

neutron firewall-update FIREWALL --router <r1> --router <r2>

In Phase 2, with the additional attribute of port list, the Create would be something
of the form:

neutron firewall-create \
        FW-POLICY --router <r1> --router <r2>  --port-id <p1> --port-id <p2> ...

neutron firewall-create \
        FW-POLICY --routers "<r1> <r2> ... <rm>" --port-ids "<p1> <p2> ... <pn>"

The port list is again optional in the update workflow. If it is not provided, the
Firewall is applied to all the internal interfaces of the Routers. And when the list of
ports is provided, it is applied on those internal interfaces mapped to the
corresponding Routers. This is identical to the create workflow.

The update workflow will also support updating the routers and/or the list of ports on a
ACTIVE Firewall. All updates are effected with proper validation of ports, routers and
port ownership with the Routers. This can be represented as below.

neutron firewall-update FIREWALL \
        --router <r1> --router <r2> --port-id <p1> --port-id <p2> ...

Performance Impact
------------------

None.

IPv6 Impact
-----------

No impacts.

Other Deployer Impact
---------------------

With support for migration, upgrades will be handled with a move to the new Data
model.

Developer Impact
----------------
Any Plugins that rely on the Firewall being installed on all Routers in the tenant and
are based directly off the reference implementation will need to be changed. This model
has been brought up in the FWaaS IRC and by reaching out to the FWaaS developers. No
significant impact has been noted. This spec also serves to address any concerns.

Community Impact
----------------
This change has been discussed in the past and attempts were made to address this
through the Service Insertion blueprints. The need to address this issue has been
discussed within the FWaaS subteam. And this has been brought up at the recent summit
by Mark and discussed with other cores. Some of the details on the approach has been
discussed amongst the FWaaS subteam at the summit and continued over at the FWaaS IRC.

Alternatives
------------
* Look again at one of the earlier Service Insertion proposals.[1][2]

Implementation
==============

The first phase will target applying on the specified routers. The addition of the
option to specify a list of ports will be phased after this.

Assignee(s)
-----------
skandasw

Work Items
----------

Phase 1
* Refactor Plugin for new routers attribute.
* DB changes to associate the routers for the firewall.
* Migration changes on upgrade.
* Agent interaction changes to the messaging dict to include the routers.
* Refactor FW Agent, to deal with specified routers and remove handling for new routers.

Phase 2 (Low Priority)
* Refactoring for additional port list attribute as above.
* Validation logic on ports on Routers and multiple Firewalls on Router.
* Agent changes to messaging for interface attributes.
* Refactor iptables driver to support specific interface in filter rules.

Dependencies
============
No direct dependency but with the services spinout, L3 Agent refactor this can cause
some pain. But it is seen that some of this can be complementary.

Testing
=======

Tempest Tests
-------------
Existing tempest test will be modified to add additional attributes.

Functional Tests
----------------
Basic functionality is not being changed so the changes to functional tests
will just involve the addition of specifying new insertion point attributes.

API Tests
---------
Changes to add the new attributes.

Documentation Impact
====================

User Documentation
------------------
User Documentation will be updated to include the new attributes and minor change to
the workflow wherein now the insertion points will need to be specified.

Developer Documentation
-----------------------
API changes will be documented.

References
==========
[1]https://blueprints.launchpad.net/neutron/+spec/neutron-services-insertion-chaining-steering
[2]https://blueprints.launchpad.net/neutron/+spec/service-base-class-and-insertion
