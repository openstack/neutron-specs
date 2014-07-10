===================================================
Group Based Policy Driver for Nuage Networks
===================================================

Launchpad blueprint:
https://blueprints.launchpad.net/neutron/+spec/group-policy-nuage-driver

Group based policy driver for Nuage Networks

Problem description
===================

Nuage's Virtualized Services Platform(VSP) [1] supports
policy based orchestration which fits well with
newly defined group based policy framework in Neutron.
It will enrich the VSP solution by extending its usage through openstack.
And also allow openstack user to take advantage of Nuage's
fully baked policy driven, application centric service architecture.

Proposed change
===============

We propose the addition of a new GBP driver to support Nuage.
It will implement the PolicyDriver interface as defined in the
abstract base class services.group_policy_driver_api.PolicyDriver:

We will support CRUD operation on endpoint, endpoint-group, policy-classifier,
policy-action and policy-rule resources.

The proposed GBP driver will interface with the Nuage's VSD using ReST
channel similar to how its done in Nuage's monolithic plugin. Library will
be re-used to avoid code duplication.

Alternatives
------------

None

Data model impact
-----------------

None (existing GBP model should suffice)

REST API impact
---------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

This driver should allow for more efficient and scalable solution
for group based policy control of deployments using Nuage's VSP.

Other deployer impact
---------------------

None

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Ronak Shah (ronak-malav-shah)

Work Items
----------

1. Developing the Nuage GBP driver
2. Writing corresponding Unit and functional tests

Dependencies
============

Group Based Policy Plugin

Testing
=======

Unit tests will be provided.
Nuage CI may need to be enhanced to support this feature.

Documentation Impact
====================

Documentation needs to be updated to reflect the addition of a new
GBP driver and its configuration parameters.

References
==========

[1] http://www.nuagenetworks.net/products/

