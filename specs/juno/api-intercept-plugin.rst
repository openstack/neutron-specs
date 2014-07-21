..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================
API Intercept Plugin
====================

https://blueprints.launchpad.net/neutron/+spec/api-intercept-plugin

There currently isn't a good place to put complex dynamic policy enforcement in
Neutron without modifying the core and service plugins to be policy aware. This
spec is to add an option to configure neutron with an API intercept plugin that
(if configured) will have all of the API requests (internal and external)
routed through it instead of being sent directly to the core and service
plugins.


Problem description
===================

There currently isn't a central place to enforce dynamic policy decisions on
API requests that come into Neutron. The policy.json file is good for making
decisions about API requests based on privileges the requesting user has on
the target objects. However, it lacks the ability to base policy on complex
conditions such as relationships between the object being modified and existing
objects.

For example, there currently is no central place to enforce a policy that states
that a firewall rule or security group rule permitting DB traffic cannot be
installed unless the source address field is restricted to a specific set of
CIDRs. This would require modifications to the core plugin as well as the FWaaS
service plugin. Even then, once the policy changes, there is nothing to trigger
a cleanup or creation of resources without making the core plugin policy aware.

Additionally, modifications to place policy in front of the external API have
no effect on calls between the core and service plugins, which may be necessary
to audit actions for certain objects (e.g. creating ports) regardless of where
they are started from.

Proposed change
===============

As an experimental approach to allow a policy plugin to restrict API
operations, this specification proposes a pluggable API interceptor. The
configured intercept plugin will receive any requests to the core and service
plugins via the neutron manager. It will then be able to enforce policies on
those requests before delegating them to the core and service plugins. It may
drop certain calls or make additional calls to other components of Neutron as
required.

This approach allows a separation of concerns by making the core and service
plugins oblivous to any complex policy enforcement and the API intercept plugin
oblivous to the implementation specifics of provisioning ports, networks, etc.

If an API intercept plugin is not configured, requests will simply be handled
in the current manner. If a plugin is configured, any requests to retrieve the
service or core plugins will return pointers to the intercept plugin which can
determine what to do with any calls that it receives.


Request Flow without Intercept Plugin::


 +----------+   Core Reqs         +----------+
 |          +---------------------+  Core    |
 |  API     |                     |  Plugin  |
 |          +-----------------+   +----+-----+
 +----------+    Svc Reqs     |        | Inter-plugin Reqs
                              |   +----+-----+
                              |   | Service  |
                              +---+ Plugin(s)|
                                  +----------+


Request Flow with Intercept Plugin::


                              +----------+
                              |  Core    |
                              |  Plugin  |
                              +----+-----+
                                   |
                                   |
 +----------+   Core Reqs    +-----+------+
 |          +----------------+  API       |
 |  API     |                |  Intercept |
 |          +----------------+  Plugin    |
 +----------+    Svc Reqs    +-----+------+
                                   |
                                   |
                              +----+-----+
                              | Service  |
                              | Plugin(s)|
                              +----------+


The target implementation of this intercept plugin will be a group policy
plugin. However, for this spec a simple logging and dummy plugin will be added
to provide an example and something to use for unit testing.

Overall, this should require a relatively minor change to the neutron manager
to reconfigure the core/service plugin references to point to the API intercept
plugin and pass the original references to the API intercept plugin.

The intercept plugin should accept all function calls and attribute accesses by
passing them through to the core or service plugin they were originally
intended for unless the intention is specifically to block them or modify them
in some way.


Alternatives
------------

Make the core and service plugins policy aware by including all of the policy
check code directly in the plugins. This approach would make the core and
service plugins complex and tightly coupled since they would be responsible for
enforcing a global policy between each other.


Data model impact
-----------------

This should have no visible impact to the data model since this just allows
requests to be intercepted in a plugin. Individual intercept plugins may change
the data model, but that is beyond the scope of this specification.


REST API impact
---------------

This plugin framework will not modify the schema of the REST API. A specific
plugin may modify the REST API depending on the policy it is enforcing, but
that is beyond the scope of this specification.


Security impact
---------------

Since these intercept plugins will always be sitting behind the normal API
policy.json enforcement, there should be no loss of security. They may add
additional checks, but they will not circumvent the current checks.


Notifications impact
--------------------

N/A


Other end user impact
---------------------

If the intercept plugin is being used to enforce strict policies, the user may
receive more error messages when trying to perform operations not permitted by
the intercept plugin.

Performance Impact
------------------

If no API intercept plugin is configured, there will be no performance impact.
If one is configured, the impact will largely depend on what the plugin checks
for every request.

Other deployer impact
---------------------

The deployers will have to specify an API intercept plugin in the same way they
specify the core and service plugins if they want to use one.

Developer impact
----------------

N/A

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Kevin Benton (kevinbenton)

Other contributors:
  Mohammad Banikazemi (banix)

Work Items
----------

* Modify neutron manager to make API intercept plugin get all API calls
* Add dummy and logging plugin for unit tests and example

Dependencies
============

N/A

Testing
=======

Unit tests will cover all of the change to allow API intercept plugins.
Individual plugins (e.g. group policy) may be tested with tempest tests.


Documentation Impact
====================

Policy plugins can be configured by the end user using the new API
intercept option.


References
==========

 * Neutron Group Policy - https://wiki.openstack.org/wiki/Neutron/GroupPolicy
