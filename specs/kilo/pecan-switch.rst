..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Replace home grown WSGI layer with Pecan
==========================================

https://blueprints.launchpad.net/neutron/+spec/pecan-switch

This document describes a plan to replace the current home-grown WSGI
framework, including REST controllers, with a solution entirely based
on the Pecan framework [1]_

The specification discussed in this document assumes that the REST controllers
will dispatch calls to the plugin in a different way, leveraging a new
interface which is thoroughly discussed in [2]_

Problem Description
===================

This specification addresses a number of issues arising from the fact that
Neutron so far has been relying on and evolving its own framework for
managing web service lifecycle and dispatch API operations to plugins.

Namely:

* API resource definition is performed using dictionaries, which contain
  information about object attribute types, default values and attribute
  validation. This has a number of limits, especially when it comes to
  performing validation and serialization of API resources, and it also
  encourages a behavior where everything is passed around as dictionaries.

* The current API extension management framework implies that extensions
  can pretty much do everything they want with the API - even redefining
  parts of it.

* There is home grown code based on fork() for managing multiple API workers.
  While this is generally not a problem, it still is a significant amount of
  code that needs to be maintained. Many REST frameworks like Pecan provide
  built-in support for spawning multiple API workers.

* The REST controllers have become heavyweight components since they also
  need to take care of tasks such as enforcing quotas and authorizing
  API requests.

* The actual response returned by the REST layer is currently built within
  the plugin, because there is no object-oriented interface with the plugin.
  Indeed, the REST controller passes the resource to the plugin as a dict,
  and expects a resource as a dict from the plugin. It assumes that the plugin
  builds a dictionary which respects the resource model (ensuring however
  that only valid attributes are returned to the API consumer).

* Most importantly, the current WSGI/REST framework is a relatively large
  size component in Neutron's codebase. Switching to a well-established
  framework will make the whole codebase a lot more maintainable.

Proposed Change
===============

In a nutshell: replace the existing framework with Pecan, remove the current
code, and ensure that operators are unaffected by the change.

This means that we expect the following for the Kilo release:

1) All API requests will be served by Pecan REST controllers
   This will have impact on in-tree attribute extensions. Such extensions
   indeed will be refactored as they currently define the resources they
   handle using the same dict-based style as attributes.py [3]_
   There will therefore be a new process for adding extensions to the Neutron
   API. This process will be documented as a part of this blueprint.

2) Service startup will not happen anymore through Python PasteDeploy, and
   will be managed by Pecan. Similarly multiple API workers will be handled
   through Pecan as well.

3) The Pecan REST controller will simply take care of serializing responses
   and deserializing requests into appropriate transfer objects describing
   API resources. However, the REST layer will no longer be responsible for
   authorization and quota enforcement. These operations will be handled by
   the new plugin layer discussed in the spec [2]_. For the sake of this
   document it is enough to say that these operations won't be performed by
   the plugin implementation.
   Request validation will occur in the REST API layer. The goal is to
   specify constraints using JSON schema. At the time of writing this spec it
   has not yet been analyzed whether it is possible to express all the
   validation constraints currently applied to the Neutron API using JSON
   schema. The final implementation, which is not necessarily the first
   iteration,  might either:

   * Use JSON schema only

   * Embed validation logic in API objects (see [2]_ more information)

   * Use a mix of JSON schema and custom validation logic, and possibly
     encapsulate everything within API objects.

4) On the other hand, authentication for a request must happen before the
   call is dispatched to the plugin layer. Pecan hooks [4]_ will be used to
   perform authentication at the appropriate time.

5) The Pecan framework however only takes care of managing the REST API
   server. For this reason as a part of this blueprint the REST and RPC over
   AMQP servers will be split. Potential impacts of this change on deployers
   are discussed in the relevant section. This split is not expected to have
   any other relevant impact on operators, developers, or users.
   The API server and RPC server will communicate via RPC where necessary.
   Where async notification tasks need to fire, any notification messages will
   be handled by post api call hooks. The notification system will be moving
   from being a side-effect of the API layer to part of the plugin layer.
   The API can fire off RPC notifications or possibly spawn tasks that execute
   on RPC/task flow workers.

Data Model Impact
-----------------

No data model change expected.

REST API Impact
---------------

Even if this patch has a deep impact on the Neutron management layer, the REST
API itself will not change at all, and will preserve its capabilities in terms
of resources, available operations, filtering, pagination and sorting.

Security Impact
---------------

Radical changes in the framework handling REST API requests always have a
potential security impact.

In this case, since we are moving away from a home grown framework to one
which is already widely adopted across OpenStack projects, the overall
security level should increase.

Notifications Impact
--------------------

The REST API layer is currently responsible for sending notifications such as
those needed by the Telemetry service. With these change the notifications
will not be handled in the REST API layer anymore, but moved within the plugin
interface as specified in [2]_

Other End User Impact
---------------------

End users will not even notice the difference between a server running the home
grown framework and one which switched to Pecan.

Performance Impact
------------------

No significant impact expected.
We have however no measurement available to justify this claim.

For this reason performance measurements should be done as part of this
blueprint implementation to ensure that switching to Pecan does not
negatively impact application performance.

For the purpose of this work, Rally will be used to provide before/after
benchmarks.
If other tools such as OsProfiler are deemed useful, they will be used
as well in the evaluation.

IPv6 Impact
-----------

IPv6-related APIs and IPAM capabilities will be unchanged.

Other Deployer Impact
---------------------

We expect the deployer impact to be minimal.
The main difference introduced by this change, from a deployer perspective
is the fact that the HTTP server will be split from the AMQP server.

For green-field deployments this will not be a problem at all.
It will also provide deployers with the desirable option of deploying the
HTTP and AMQP servers on different nodes.

For existing deployments, updates should be smooth and transparent.
The only difference would be that after an upgrade there would not be
a single neutron server service, but two - one for the REST API, and one
for RPC over AMQP.

Developer Impact
----------------

New extensions will need to be developed in a different way.
This will be thoroughly documented in developer documentation.

Community Impact
----------------

Moving away from the home-grown framework will allow the community to focus
exclusively on Neutron's business logic. Moreover, members of the Neutron
community will also be encouraged to contribute back to Pecan.

Alternatives
------------

Other solutions such as Falcon [5]_ and WSME + Pecan [6]_ have been
considered. However the adoption of Pecan appears the one that better suits
Neutron.

A mailing list discussion [7]_ on REST API frameworks has been used to provide
some guidance. For WSME, even if it is an interesting solution to increase code
maintanability, and ease the development process, we struggled during some
early experiments to make it work with the current extension model. Even if
it might be argued that the problem in this case is the extension model, we are
unable to recommend it as a part of this blueprint.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Mark McClain (markmcclain)
  Kevin Benton (kevinbenton)

Other contributors:
  Sean Collins (sccal68) [developer docs]
  Salvatore Orlando (salv-orlando) [reserve dev]

Work Items
----------

1) Define framework for Pecan controllers for core and extended resources.
2) Re-implement controllers for base and extended resources, paying particular
   attention to dealing properly with 'attribute' extensions. The deliverable
   of this work item will be a new "base controller" which will leverage the
   v3 plugin interface proposed in [2]_.
3) Plug authorization and quota enforcement in the "plugin management"
   layer.
4) Split out RPC over AMQP server
5) Redefine unit tests to work with new framework
6) Validate new solution with integration testing, perform performance and
   scalability analysis.

Dependencies
============

* New plugin interface specification [2]_

Testing
=======

Once the changes are in place and integrated with the new plugin interface
discussed in [2]_, gate tests should run as usual. We do not expect this
change to have any impact that might trigger race conditions leading to
intermittent gate failures.

On the other hand, this change will have a significant impact on unit
testing. Most unit tests exercise the REST API server and with this change
these unit tests will be inevitably broken.
Under this proposal we therefore expect significant changes in the "base
classes" for unit test, such as [8]_.


Besides, new modules introduced as a part of this blueprint should be
thoroughly unit tested, with a target level of coverage between 90% and 100%.
Test coverage should be verified with tox -ecover.

Tempest Tests
-------------

No new tests are anticipated.

Functional Tests
----------------

Even if API functional testing will eventually be a relevant part of Neutron's
functional testing suite, this is outside the scope of this spec.

API Tests
---------

Please see previous section.

Documentation Impact
====================

As the specification discussed in this document changes the way in which the
Neutron server is deployed because of the split between the HTTP and RPC over
AMQP server, this will need to be appropriately documented in the admin guide.

User Documentation
------------------

No change.

Developer Documentation
-----------------------

The new process for developing Neutron extensions should be thoroughly
documented.

Also the developer documentation for the api layer [9]_ needs to be updated
according to the changes being made as part of this blueprint.

References
==========
.. [1] Pecan documentation: http://pecan.readthedocs.org
.. [2] v3 plugin interface: https://review.openstack.org/#/c/140527/
.. [3] https://github.com/openstack/neutron/blob/master/neutron/api/v2/attributes.py
.. [4] Pecan hooks: http://pecan.readthedocs.org/en/latest/hooks.html
.. [5] Falcon WSGI framework: http://falconframework.org/
.. [6] WSME: http://wsme.readthedocs.org/
.. [7] http://lists.openstack.org/pipermail/openstack-dev/2014-March/030385.html
.. [8] http://git.openstack.org/cgit/openstack/neutron/tree/neutron/tests/unit/test_db_plugin.py
.. [9] http://docs.openstack.org/developer/neutron/devref/api_layer.html
