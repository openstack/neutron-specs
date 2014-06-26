..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================
Service Flavor Framework
========================

https://blueprints.launchpad.net/neutron/+spec/neutron-flavor-framework

This specification details a framework to enable Operators to configure and
Users to select from different abstract representations of a service
implementation in Neutron. The representation decouples the logical
configuration from its instantiation allowing support of multiple possible
implementations to coexist. Decoupling also enables Operators to create user
options according to deployment needs. This proposal does not require any
significant driver integration to implement.


Problem Description
===================

The service provider framework allows services to be backed by multiple
drivers; however there are several limitations with the current approach. The
limitations include:

* Operators cannot modify the deployment without user knowledge.
* Operators cannot easily configure different service levels for same driver.
* Operators cannot transparently deploy a service in multi-vendor environment.
* Operators cannot easily reschedule service to another backend.
* Operators cannot influence the scheduling during backend selection.
* Users must be aware of the driver implementing the service.
* Vendors cannot differentiate.
* Extensions cannot be easily exposed to the user via REST API.


Proposed Change
===============

The proposed solution is to introduce two new logical entities exposed via the
REST API to decouple the logical configuration from the backend implementation.
The first entity is called Flavor and is an operator curated description of the
backend implementation. Flavor's attributes include id, name, extension list,
service, service profiles, selection algorithm and enabled flag. The extension
list is used to enable additional REST API functionality for logical
configurations associated with the flavor. An example would be enabling TLS
and/or L7 for load balancing. Users are allowed to query and view the public
attributes of the flavor. The selection algorithm determines the service
profile and the driver schedules the instance to the backend.

The second entity is Service Profile which has attributes of id, name,
entry point, driver initialization metadata and enabled flag. The Service
Profile enables an operator to associate the driver entrypoint with metadata
necessary to initialize the driver. A benefit of metadata is that the same
driver may back different flavors definitions. Multiple service profiles may be
associated with the same flavor to enable multi-vendor/multi-driver
environments.

This extension will introduce a new RPC method in the core server. The method
will handle scheduling the logical instance onto the backend. Scheduling will
involve binding the logical configuration to a backend with capacity. Once
scheduled, the plugin will directly dispatch calls to the driver. The
scheduling method implementation will be intentionally kept simple to enable
this work to be completed within the cycle. Future work may include a more
advanced scheduler by incorporating Gantt (a project to make the Nova scheduler
available to other OpenStack components) and/or developing a method for
backends to express load/capacity. Auto-rescheduling and instance health
monitoring are outside the scope of this change and are the responsibility of
the driver. Operators also retain control over cost and scheduling via their
selection of the service profiles.

While flavors and service profiles could be stored in configuration files, this
spec proposes to store the entities in the database. Storage in a relational
database ensures atomicity and consistency across neutron servers and workers
by avoiding configuration deployment races. Additionally, an operator may
add/update/delete flavors without system downtime. (Note: Flavor and service
profiles may not be changed or removed if in use).

User Workflow
  Users will query for a list of enabled flavors filtered by service. The
  user will then create the logical entity for the service passing the
  flavor id. The user workflow will be unchanged after this time.
  If a flavor is selected with multiple service profiles, the profile is
  selected via a weighted random algorithm (see below.)

Admin Workflow
  An admin knowing the hardware/software versions within the deployment will
  create flavors to offer. The operator will choose the extensions that a
  flavor will support and ensure the appropriate service profiles are
  associated.

Future work includes tenant specific flavors and advanced scheduling
(integration into Gantt project and/or filters).

This proposal enables vendor differentiation since flavor definitions may
include vendor API extensions that the operator may choose to enable.

Data Model Impact
-----------------

Two new logical models will be added to the database.

Flavor
  id: uuid
  name: string
  description: text
  service: LOADBALANCER, VPN, FIREWALL, L3_ROUTER, etc
  supported_extensions: comma separated value string
  selection_algorithm: Enum(random, available, least_used)
  service_profiles: [(uuid list, weight)] (JSON list)
  enabled: boolean

Service Profile
  id: uuid
  description: text
  entry_point: string
  metadata: string(json encoded dict)
  enabled: boolean

The existing provider attribute will be changed to service_profile_id and used
to associate the root of the service's logical model with the profile.


REST API Impact
---------------

The new logical models will be exposed via the API. The logical model
CRUD operations will only be exposed to administrators with the exception of
read operations on Flavor. All users will be able to query and get the details
on a single flavor with a limited set of attributes (id, name, description,
service and service extensions). Additionally, one administrative action will
be added to each service: reschedule(id). This action will enable an operator
to force the system to reschedule the backend for a logical service.

FLAVORS (/flavors):

+--------------------+-------+---------+---------+------------+--------------+
|Attribute           |Type   |Access   |Default  |Validation/ |Description   |
|Name                |       |         |Value    |Conversion  |              |
+====================+=======+=========+=========+============+==============+
|id                  |string |RO, all  |generated|N/A         |identity      |
|                    |(UUID) |RW, admin|         |            |              |
+--------------------+-------+---------+---------+------------+--------------+
|name                |string |RO, all  |''       |string      |human-readable|
|                    |       |RW, admin|         |            |name          |
+--------------------+-------+---------+---------+------------+--------------+
|description         |string |RO, all  |''       |string      |human-readable|
|                    |       |RW, admin|         |            |description   |
+--------------------+-------+---------+---------+------------+--------------+
|service             |string |RO, all  |''       |string      |token mapping |
|                    |       |RW, admin|         |            |flavor to svc |
+--------------------+-------+---------+---------+------------+--------------+
|service_profiles    |list   |RO, all  |[]       |json list   |driver mapping|
|                    |       |RW, admin|         |            |with weight   |
+--------------------+-------+---------+---------+------------+--------------+
|supported_extensions|string |RO, all  |''       |string      |available api |
|                    |       |RW, admin|         |            |extensions    |
+--------------------+-------+---------+---------+------------+--------------+
|selection_algorithm |enum   |RO, all  |random   |see model   |how to select |
|                    |       |RW, admin|         |            |profile       |
+--------------------+-------+---------+---------+------------+--------------+
|enabled             |bool   |RO, all  |true     |bool        |toggle        |
|                    |       |RW, admin|         |            |              |
+--------------------+-------+---------+---------+------------+--------------+

SERVICE_PROFILES (/service_profiles):

+-----------------+-------+---------+---------+------------+--------------+
|Attribute        |Type   |Access   |Default  |Validation/ |Description   |
|Name             |       |         |Value    |Conversion  |              |
+=================+=======+=========+=========+============+==============+
|id               |string |RO, all  |generated|N/A         |identity      |
|                 |(UUID) |RW, admin|         |            |              |
+-----------------+-------+---------+---------+------------+--------------+
|description      |string |RO, all  |''       |string      |human-readable|
|                 |       |RW, admin|         |            |description   |
+-----------------+-------+---------+---------+------------+--------------+
|entry_point      |string |RO, all  |''       |string      |python module |
|                 |       |RW, admin|         |            |path to driver|
+-----------------+-------+---------+---------+------------+--------------+
|metadata         |string |RO, all  |''       |json string |meta data     |
|                 |       |RW, admin|         |            |              |
+-----------------+-------+---------+---------+------------+--------------+
|enabled          |bool   |RO, all  |true     |bool        |toggle        |
|                 |       |RW, admin|         |            |              |
+-----------------+-------+---------+---------+------------+--------------+

Security Impact
---------------

The policy.json will be updated to allow all users to query the flavor
listing and requst details about a specific flavor entry. All other REST
points for create/update/delete operations will admin only. Additionally, the
CRUD operations for Service Profiles will be restricted to administrators.


Notifications Impact
--------------------

The content of current notifications will change minimally to add a new flavor
attribute. This attribute will be used for applications such as billing and can
be captured by Ceilometer. The provider attribute will be retained for
operators that wish to track the user's flavor and backend.

Other End User Impact
---------------------

N/A

Performance Impact
------------------

There will be a minimal overhead incurred when the logical representation is
scheduled onto the actual backend. Once the backend is selected, direct
communications will occur via driver calls.

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

The deployer will need to craft flavor configurations that they wish to expose
to their users. During migration the existing provider configurations will be
converted into basic flavor types. Once migrated, the deployer will have the
opportunity to modify the flavor definitions.

Developer Impact
----------------

The expected developer impact should be minimal as the framework only impacts
the initial scheduling of the logical service onto a backend. The driver
implementations should remain unchanged except for the addition of the capacity
call.

Community Impact
----------------

This proposal allows operators to offer services beyond those directly implemented,
and to do so in a way that does not increase community maintenance or burden.

Alternatives
------------

Keep doing nothing.

Implementation
==============

Assignee(s)
-----------
Doug Wiegley (original spec and code by markmcclain and enikanorov)

Work Items
----------

* Implement the new models
* Implement the REST API Extension (including tests)
* Implement the scheduling RPC call.
* Add support for flavors to LBaaS, VPNaaS, and FWaaS APIs.
* Implementation migration script for existing deployments.
* Ensure notifications for Ceilometer are available.
* Add client API support

Dependencies
============

No dependencies on other work. It should be noted that many other items depend
on flavors.

Testing
=======

Tempest Tests
-------------

Tempest testing including new API and scenario tests to validate new entities.

Functional Tests
----------------

Functional testing is being excluded from this change because this API
does not directly alter the data path. (Elements that alter the datapath are
covered by other functional test).

API Tests
---------

The new API will be tested.

Documentation Impact
====================

User Documentation
------------------

User documentation will need be included to describe to users how to use
flavors when building their logical topology. Operator documentation will
need to be created to detail how to manage Flavors and Service Profiles.

Developer Documentation
-----------------------

Additionally, documentation of the new REST endpoints will need to be included
in the Networking API description.

References
==========
* Juno spec gerrit review - https://review.openstack.org/#/c/102723/


