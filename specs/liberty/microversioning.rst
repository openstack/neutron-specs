..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================================
Neutron API evolution: extension consolidation and micro-versioning
===================================================================

https://blueprints.launchpad.net/neutron/+spec/consolidate-extensions

This specification has the rather ambitious goal of introducing a new process
and paradigm for evolving the Neutron API.
Starting from similar approaches adopted in Nova [#]_ and Ironic [#]_, this
document discusses, with a fair amount of detail, how a similar strategy
can be implemented for Neutron, and what are the expected benefits.

Generally speaking, the aim of the work described in this document is to
achieve an API evolution method which will benefit operators and users, as well
as developers.

In order to achieve this goal, two main activities can be identified:
* Redefining the concept of "core" and "extensions" APIs, progressively merging
the latter into a single "unified" Neutron API
* Defining and implementing a strategy for evolving the API through versioning


The expected benefits are:
* Stopping extensions non senses. For instance, several features which are
nowadays required for a fully functional interaction with nova are considered
extensions;
* Providing a consolidate view of the API. Currently extensions, and in
particular "attribute extensions", are causing the API to look different from
deployment to deployment;
* Simplifying DB management code. Condensing extensions into core will allow us
for squashing together a few DB mixins, such as those for L3 DB management;
* Introducing a simpler and more effective strategy for evolving the API;
* Enabling developers to break backward compatibility without breaking client;

The content of this document is based on the experiences of OpenStack services
which already adopted similar API evolution strategies ([1]_, [2]_), and adapt
principles and techniques adopted in these services to Neutron-specific
nuances such as, for instance, its plugin and driver based architecture, as
well as its deep "vendorization".


Problem Description
===================

Currently Neutron defines over 25 extensions in the neutron.extensions
package, plus a number of vendor-specific extensions which are mostly
off-tree.

Extensions have so far served two purposes: flexibility and evolution.

Extensions such as load balancing, or IPSEC VPN, implement additional
services which might or might not be part of a Neutron service offering.
However, most of the currently defined extensions represent an evolution of
Neutron's small core API - which has been roughly unchanged since the Folsom
release.

Even if the number of available extensions is not overwhelming, this already
causes some problems to API consumers. Since the structure of requests and
responses can change from deployment to deployment, achieving portable
applications which make use of the Neutron API is rather difficult. By
"portable" here we mean applications which can be used against different
OpenStack API providers.

Moreover, API evolution through extension is an approach that tends to
become confusing and hard to maintain in the long term.
As Neutron has reached the point where extensions are being introduced to
extend other extensions, this probably means that the extension mechanism
is already being abused as it is not suitable for managing API evolution.

The previous paragraphs hopefully provided enough evidence to convince the
audience of the need to move away from extensions as a mechanism for
evolving the Neutron API. Nevertheless, this should not come at the expense
of the flexibility of the extension mechanism, in particular for what
pertains optional Neutron features (such as port security) or, to some extent,
vendor-specific extensions.


Proposed Change
===============

This section discusses mostly changes in Neutron's API evolution strategy.
Actually, since Neutron so far has no evolution strategy it is probably fairer
to state that this section presents a strategy for evolving the Neutron API.

It is intentionally light from a technical perspective. Some details concerning
how these process changes will be reflected in changes into Neutron's API
layer are reported in the "implementation" section.

Part I - API micro-versioning
------------------------------

In this sub-section we assume that the various extensions have already been
consolidated in a "unified" API - made by a relatively small "core" and a
number of optional features. Moreover, the change presented here follows
the guidelines adopted for implementing microversioning in Nova and Ironic
([1]_, [2]_). At the time of writing this document, there isn't yet an API-wg
guideline available for versioning.

The API version will be a monotonic counter in the X.Y form where X and Y
are set according to the following convention:
* X will only be changed if a significant backwards incompatible API change is
made which affects the API as whole. This could be, for instance, a change in
the API paradigm which completely transforms the way in which users interact
with the Neutron API. For instance, a 3.0 Neutron API will be introduced
should the API become completely asynchronous and task-oriented. This number
should be incremented very rarely.
* Y should be incremented for any change made to the API. Therefore the API
version should be incremented anytime the structure of the resources managed
by the API changes, or every time changes are made to request/response headers
and/or response codes.
Due to the plugin-oriented nature of the Neutron API, it is quite difficult
to require changes in the neutron API for semantic changes (eg: changes that
alter the behaviour of an operation).

The server will list supported versions in the same way as discussed for the
Nova specification; client interactions will also occur in the same way as
it is done for Nova - which is convenient as it makes the introduction of
microversioning transparent to older clients.

The advantage of 'copying' what other OpenStack projects are doing, at least
for the user facing aspects of this change is consistency. Even if the current
microversioning approach has issues, it is surely not fatally flawed. It is
therefore advisable to provide OpenStack users with a consistent experience
across projects, and then improve together versioning with the assistance of
the API working group.

The proposed versioning schema does not distinguish between backwards
compatible and backwards incompatible changes. Even if one of the reasons for
introducing micro-versioning is to be able to break backward compatibility
without breaking clients, it is important to strive for preserving it.
In this specification we do not discuss what steps must be taken should
backward compatibility be deliberately broken. It might be argued that a
X.Y.Z versioning number might be more suitable, with the Y counter being used
for backward incompatible changes. However, it is the opinion of the
proposer that discussions about version numbering scheme should be regarded
as cross-project, and therefore are outside the scope of this document.

Evolution and experimentation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

One aspect that so far has hampered Neutron API development is that the API
"must be perfect" at the first iteration. Once released, backward compatibility
has to be maintained. Doing any change, even when there's something wrong
in the way the user interacts with the API, become very difficult. Needless
to say, this causes a lot of friction and frustration both in users and
developers. For this reason the concept of "current", "stable", and "latest"
APIs developed for Nova and Ironic make a lot of sense. Similar concepts will
be used in Neutron.

The goal here is to enable innovation in the API, without having to put with
multi-year conversations like the QoS discussion [#]_ or endless negotiations
which resemble more the Congress of Vienna [#]_ rather than a design discussion
like the ones around the Load Balancing v1 API.

There are two categories of "innovation" in the API which might require
experimentation:
1. API changes which add functionalities to the Neutron APIs. This might
include, for instance, dynamic routing or task management.
2."Next-gen" APIs. For instance developers might attempt a complete
redefinition of the API and would like to provide early adopters which
a sneak preview.

99% of API innovation is likely to fall in the first category. These API will
be first proposed as "experimental" and completely unversioned. Deployers will
be able to switch them on or off with a configuration setting. A server which
supports experimental API will return a version header like the following:

::

  X-OpenStack-Neutron-API-Version: <version_number>, experimental

And a response like the following will be returned when querying the version
URI:

::

  GET /

  {
       "versions": [
          {
              "id": "v2.0",
              "links": [
                    {
                      "href": "http://localhost:9696/v2.0/",
                      "rel": "self"
                  }
              ],
              "status": "CURRENT",
              "latest_version": "2.114",
              "min_version": "2.0",
              "default_version: "2.0",
              "experimental_apis": "enabled"
          }
     ]
  }

Experimental APIs can change in any way without any requirement on versioning,
backward compatibility. Obviously, should they be removed, no deprecation
process should be followed. At some point it should be possible to propose
experimental APIs for "promotion". This should go through the usual review
process. Once promoted into the "consolidated" API an experimental API will
get assigned a version number. From that point on, backward compatibility will
be enforced.
This will allow for early feedback from user and operators, which will be
finally empowered in cooperating in the API definition process.
On the other hand it will enable developers to shape the API in the right way
without having to obsess about backward compatibility.

The second case, "next-gen" APIs, can be enabled by enabling something like
the following:

::

  GET /

  {
       "versions": [
          {
              "id": "v2.0",
              "status": "CURRENT",
              "latest_version": "2.114",
              "min_version": "2.0",
              "default_version: "2.0"
          },
          {
              "id": "v3.0",
              "status": "PROPOSED",
              "latest_version": "3.1",
              "min_version": "3.044",
              "default_version: "3.1"
          }

     ]
  }

While interesting, this is however simply outside the scope of the current
work. At some point the Neutron API layer will be able to offer something
like this. For the sake of the present release cycle, there is simply no
need to address "future" APIs.


API extensions for advanced services
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Advanced services which have been spun off into separate repositories
during the Kilo release cycle are exempted from micro-versioning.
The corresponding extensions will not be versioned.
Once such advanced services will become standalone, they will hopefully
adopt a similar versioning scheme for their APIs.

Plugin and vendor-specific API extensions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

There are also plenty of in-tree and out-of-tree vendor extensions in neutron.
Vendor extension here means an extension which is either available only to a
certain plugin or driver, or an extension specific to a particular deployment.
The extension loading mechanism would still enable this kind of extension.
However, Neutron will provide no versioning guarantees. It will simply provide
some hooks for adding headers in the HTTP response. For instance:

::

  X-OpenStack-Neutron-API-Ext: Name=SomethingExtended; Version=99.99;

It is up to the extension implementor to perform versioning for them, if they
wish so. It is important to note that these extensions are NOT part of the
Neutron API. Official clients, such as the CLI, should not provide support
for them. Additionally, developers wishing to write portable APIs should not
rely on them. And finally there is no provision for considering for "promotion"
into the "unified" API.

Part II - API Extension veto
------------------------------

While micro-versioning plan is discussed and implemented it is quite likely that
new extensions will be added to the Neutron code base. While these events are
generally inconvenient, it will be however unfair and unpractical to impose a
veto on new extensions as this would be tantamount to freezing the API.

For this reason new extensions will be allowed and tolerated until the
micro-versioning approach outlined in the previous sub-section is implemented.

New extensions added before microversioning is in place will be integrated into
the "consolidated" API discussed in the next sub-section.

Part III - API consolidation
------------------------------

Once microversioning is in place most of Neutron's extensions APIs will become
part of a "consolidated" Neutron API. From a technical perspective, this simply
means that the relevant resources will not be loaded anymore by the extension
manager.

This section discusses impact on plugins and ML2 drivers. Impact for end users
and deployers is discussed in the appropriate section.
The aim of this change is not to force plugin and drivers to implement things
that nowadays are considered optional. Therefore, plugin and driver maintainers
will not need to ensure they implement all the extensions which will be moved
into the consolidated API.

Extensions generally represent features which might or might not be enabled in
a given deployment. It might be rightly argued that a lot of Neutron API
extension are actually essential features, but this is not very relevant to
the purpose of this discussion.
Therefore Plugins and drivers still won't be required to implement these
optional features.

From a plugin maintainer perspective this will imply that:
* Rather then declaring supported extensions, a plugin will declare which of the
neutron features it implements. This will include things like DVR, or DHCP
options. If a plugin does not implement one of the Neutron API features,
API requests pertaining that feature will return with a 4xx error response
code [#]_.
* The mixin approach, albeit another candidate for overhauling, will not be
touched as part of this blueprint. However, some mixins might be "coalesced"
as a part of this process. This should have no impact on plugin and drivers,
beyond probably changing the name of the mixin the plugin implements.
There might however be cases in which this consolidation might break a
plugin, for instance when the plugin redefines methods defined in such
mixin classes. Extreme care should be paid to the results of 3rd party CI
tests. The general approach with mixins should be that any refactoring
should be performed only if absolutely necessary. Unnecessary yak shaving
in badly painted bike sheds should be avoided.
* Plugins and drivers can still implement vendor-specific extensions, which,
as specified in the previous section, are not part in any form of the neutron API.

So far the change described in this section merely replaces the term
"extension" with "feature". The only advantage is that there will be a fixed
list of features rather than an open ended list of extensions.
Regardless, this is going to do little towards portability of API across
deployments. For this reason a list of "mandatory" features is proposed.
This list if detailed in the "Implementation" section. This list is simply the
minimum set of features required for complete integration of Neutron with the
other OpenStack services.

Data Model Impact
-----------------

We do not expect any change in the Neutron data model.

REST API Impact
---------------

This change will introduce a new URI for retrieving enabled features.
In other words this will be the URI to retrieve the list of optional
feature which were previously available under the 'extensions' URI.

A feature item will have the following structure:

+------------+-------+---------+---------+------------+-------------------+
|Attribute   |Type   |Access   |Default  |Validation/ |Description        |
|Name        |       |         |Value    |Conversion  |                   |
+============+=======+=========+=========+============+===================+
|name        |string |RO, all  |N/A      |string      |short name         |
|            |       |         |         |            |                   |
+------------+-------+---------+---------+------------+-------------------+
|description |string |RO, all  |''       |N/A         |a relatively       |
|            |       |         |         |            |short description  |
|            |       |         |         |            |of what this       |
|            |       |         |         |            |feature does       |
+------------+-------+---------+---------+------------+-------------------+
|enabled     |boolean|RO, all  |N/A      |true/false  |True if the feature|
|            |       |         |         |            |is enable in the   |
|            |       |         |         |            |current deployment |
+------------+-------+---------+---------+------------+-------------------+
|experimental|boolean|RO, all  |N/A      |true/false  |True if the feature|
|            |       |         |         |            |is to be considered|
|            |       |         |         |            |experimental.      |
+------------+-------+---------+---------+------------+-------------------+


The /extensions URI will still be available, and it will list those
extensions which are orthogonal to the neutron core API (eg: load balancing)

It is also important to note that this change will also change the structure
of all the resources for which an 'attribute extension' is being moved into
the core API. So far is the deployment did not support a given extension, the
attribute was not present at all. This caused a resource to have a different
'shape' according to which extensions were enabled.
With these change, resources will always have the same shape, but attributes
for optional features, such as 'distributed' in the 'router' resource, will
be set to a 'null' or 'N/A' value in responses if the feature is disabled;
on the other hand if they're specified in requests, and the corresponding
feature is disabled a 501 error code will be returned. This approach is
preferable over blindly ignoring attributes for disabled features.

As a part of this change Neutron the response version information endpoint:

GET /

will change for comply with the microversioning strategies adopted by Nova and
Ironic. There is a change that such response will be changed in a backward
incompatible way. The proposer regards this as acceptable - considering, for
instance, the fact that the version information API for Neutron is not
documented.

Security Impact
---------------

The proposed change does not change the way in which APIs are consumed nor
it changes the way in which APIs are dispatched to the plugins.

We therefore do not expect this change to have any security impact.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

It is important to note that applications relying on querying the 'extensions'
URI for feature discovery, including Nova,  might be affected by this change.
In this case a transitory solution for the Kilo release cycle might be
devised, or critical consumers (such as Nova) might be appropriately patched.

Old client applications, unaware of API versioning, will keep working unchanged.
The server will automatically serve these clients using the 'default' version
of the API.

For a thorough discussion of the various combinations of old/new clients with
old/new servers, please refer to the Ironic specification [2]_, as the same
considerations apply to Neutron versions as well.


IPv6 Impact
-----------

None.

Performance Impact
------------------

None.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

As soon as the code for supporting micro versioning is tested and verified,
no new extensions will be allowed in the neutron source tree.
The approval of this work item alone might be regarded as a declaration of
intent towards having an official Neutron API which is versioned and has no
extensions.
From a process perspective, the developer community should expect the
Neutron PTL, or somebody chosen for this specific task to announce on the
mailing list the beginning of the "versioning era", and then the end of the
"extension era". The latter announcement should be made once the feasibility
of microversioning has been proved by creating revisions for one or more
extensions. This will impact all developments with an impact on the API.
If those development propose new extensions, they will have to be converted
into an API revision.
This won't affect however advanced services and plugin-specific extensions
developed outside of the neutron repository.

No immediate impact is expected on plugin maintainers. Moving forward, plugin
will need to become version-aware, especially when backward compatibility is
broken. For these cases tools will be provided for building version-aware
plugin operations.

Finally, it is not possible at this stage to estimate the precise impact this
work will have on the ML2 extension manager. Considering the goal it achieves,
it might make sense to keep it as a mechanism for loading driver-specific
extensions. Regardless, the team working on micro-versioning will work with the
ML2 team to ensure a smooth transition for every ML2 driver. It is worth noting
that driver-specific extensions will be treated as plugin-specific and will not
be assigned in any case a Nuetron API revision.

Community Impact
----------------

This proposal will obviously concern all those community members who wish to
keep evolving the Neutron API by adding extensions.

However, this proposal does not advocate for the suppression of vendor
specific APIs; they can still be allowed, even if they should not be in any
case considered part of the Neutron API - and to this extent should not live
in the neutron source code tree.

Alternatives
------------

This section discusses alternatives under the assumption that there is
consensus about the fact that extensions are not anymore a viable way of
evolving the API.

A simple alternative approach is to do nothing, and simply document which
extensions should be considered part of the core API. However, as there
is no way to actually enforce a different API core, this will just increase
the chances that actual deployments of the Neutron service provide an API
which diverges from the documentation.


Implementation
==============

This section presents only a high level implementation plan. Full details on
the implementation will be made available with the code which makes micro
versioning possible. Once micro-versioning is enabled, migration of extensions
into features should be trivial.

It is indeed impossible to present a detailed plan here. The Pecan switch
activity [#]_ indeed will bring changes to the WSGI framework which are likely
to have an impact on the API itself and therefore on the way microversioning
is performed. Neverthless, in this document we assume microversioning is going
to be implemented on the current WSGI framework.

Specifying a new API revision
------------------------------

TODO

The RESOURCE_ATTRIBUTE_MAP in a micro-versioned world
------------------------------------------------------

TODO


Version-aware plugins
----------------------

It is well known that plugins provide management layer implementation for the
REST controllers. Plugins must be therefore aware of versions so that they can
treat appropriately request data and generate response data.

As an example, let's assume a change into the structure of the 'networks'
resource requires a change into create_network starting with revision 2.111.
This might be implemented as follows:

::

  @api_version(method='create_network', min_version='2.0', max_version='2.110')
  def _create_network_old(context, network_data):
      # code goes here

  @api_version(method='create_network', min_version='2.111')
  def _create_network_new(context, network_data):
      # code goes here


At the time of writing this document it has not yet been clearly defined how
this mechanism could be made available for ML2 drivers. The final
implementation will however provide a solution for this.

Migration of Neutron extensions into features
---------------------------------------------

The list below discusses which extensions will be made part of the core
Neutron API:

* agent
  Admin extension for managing agents
  feature: agent_management
* allowedaddresspairs
  Relax anti spoof rules on ports
  feature: address_pairs
* dhcpagentscheduler
  schedule networks on dhcp agents
  feature: agent_scheduler
* DVR
  Dsitributed virtual router
  feature: l3_DVR
* external_net
  router:external attribute
  feature: l3 (MANDATORY)
* extra_dhcp_opt
  custom dhcp options on ports
  features: extra_dhcp_options
* extraroute
  router static routes
  feature: l3_static_routes
* firewall
  edge firewal
  advanced service - keep as extension
* flavor
  specific to meta plugin
  make 3rd party specific, it should not even be here
* l3
  routers and floating ips
  feature: l3 (MANDATORY)
* l3_ext_gw_mode
  control ability of doing SNAT at the gateway
  feature: l3_snat_control
* l3_ext_ha_mode
  create HA router
  feature: l3_ha
* l3agentscheduler
  schedule routers on l3 agents
  feature: agent_scheduler
* lbaas_agentscheduler
  schedule vips on lb agents
  advance service - keep as extension
* loadbalancer
  load balancing
  advanced service - keep as extension
* metering
  label based bandwidth metering
  feature: metering
* multiprovidernet
  like provider, but with multiple bindings
  feature: provider_networks
* portbindings
  admin extension for exchanging info and capabilities about ports
  feature: port_bindings (MANDATORY)
* portsecurity
  enable/disable antispoof capabilities on a port or network
  feature: port_security
* providernet
  provider networks
  feature: provider_networks
* quotasv2
  quota management
  feature: Quotas (MANDATORY)
* routedserviceinsertion
  attach services to routers
  consider removal or make it plugin specific
* routerservicetype
  specifies which type of router should be implemented
  consider removal or make it plugin specific
* security groups
  control access to ports
  feature: security_groups (MANDATORY)
* service type
  select driver for adv services
  advanced service: keep as extension
* vpnaas
  IPSEC VPN
  advanced service: keep as extension
* subnet_allocation
  Subnet pools and allocation of subnet from pools
  feature: ipam (MANDATORY)
* OTHER KILO EXTENSIONS PLEASE!


Assignee(s)
-----------

Primary assignee:
  salvatore-orlando

Other contributors:
  <your name here>

Work Items
----------

* Introduce 'features' URI
* Make chosen extensions part of the new unified API
* Revisit DB mixin and coalesce where possible (eg: external gw modes)
* Introduce micro-versioning

Dependencies
============

Ideally this change should depend on the Pecan switch. Implementing this
blueprint with the current WSGI framework will just mean that we'll have to
do the work again once the switch to Pecan occurs.


Testing
=======

This specification does not change the testing surface, if not for the
addition of the 'features' URI for which appropriate unit and functional
tests will be provided.

Functional Tests
-----------------

Todo.

API Tests
----------

Todo.


Tempest Tests
--------------

No tempest test should be implemented for the purpose of this work.

Documentation Impact
====================

User Documentation
------------------

Significant changes in the API reference documentation. Finally most of the
extension appendix will go away. The need for these changes should actually
seen as something good.

Developer Documentation
-----------------------

We would need to document the introduction of the supported_features variable.
Trouble is that our developer documentation is so poor that at the moment there
is not even a place where this can be stated.

References
==========

.. [#] Nova API micro versioning specification
 http://specs.openstack.org/openstack/nova-specs/specs/kilo/implemented/api-microversions.html
.. [#] Ironic API micro versioning specification
 http://specs.openstack.org/openstack/ironic-specs/specs/kilo/api-microversions.html
.. [#] First mailing list discussion on QoS API:
.. [#] Congress of Vienna: http://en.wikipedia.org/wiki/Congress_of_Vienna
.. [#] It is the opinion of this document's author that a decision about the
 actual status code might be taken even after approval of the work proposed
 in this document. However, it is worth noting that a "501 Not Implemented"
 status code is not correct, because i) that status code was conceived with
 a different goal in mind, and ii) 5xx status codes are used to describe
 failures due to server-side issues; in this case however nothing went wrong
 on the server side. More information on the subject can be found at the
 ever-referenced RFC 2616: http://www.w3.org/Protocols/rfc2616/rfc2616.txt
.. [#] Pecan switch for Neutron specification
