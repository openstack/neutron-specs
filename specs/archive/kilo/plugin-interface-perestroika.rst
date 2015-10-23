..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Introduce a new plugin interface
==========================================

https://blueprints.launchpad.net/neutron/+spec/plugin-interface-perestroika

This specification proposes a new plugin interface which will provide a
stronger interface between the management layer and the plugin themselves,
as well as paving the way for a proper separation of concerns between the
REST layer, Neutron management layer, and the plugin layer.

Problem Description
===================

It is well known that there are a few shortcomings with both the current
plugin interface and the inheritance pattern which ties all the plugins
to the database "base" class [#]_

While issues arising from the inheritance pattern and its (ab)use are outside
of the scope of this document, in the rest of this section we list the issues
that we are planning to solve with this piece of work.

* The interface is currently operation oriented. While this worked fairly well
  in the Folsom world, when plugins where required to implement only a small
  set of operations, it is not practical anymore.

* While python's abc enforces checks on the shape of implementation of the
  plugin interface, no sort of contract exists when it comes to data passed
  to such interface (and returned by it). This happens because data are
  passed as free-form dicts, with the plugin implementations expecting these
  dicts to have a particular structure.

* The current plugin interface delegates all the business logic to the plugin,
  with the exception of authorization and quota enforcement. This leads plugin
  to over rely on the business logic contained in the "db base plugin", whereas
  some of that logic, especially the one for managing "composite" operations
  such as attaching a subnet to a router, should be moved to an appropriate
  management layer thus ensuring plugin operations are as simple as possible.

* As the plugin is currently 100% responsible for determining how an API
  operation is performed, it is very difficult to add capabilities such as
  task-based workflows, or server failure recovery without having them as
  plugin specific.

Summarizing the current plugin interface has become all but unmaintainable.

Proposed Change
===============

The aim of this specification is to introduce a new, and hopefully better,
interface between the REST layer and the plugin layer, which addresses the
issues listed in the section above.

This specification relies on the following assumptions:

* There will be an object framework for describing API resources. This
  objects framework will encapsulate resource validation logic, but should
  not, at least at this stage, include capabilities such as call remoting
  or persistence. Objects backed by a persistence layer cannot be
  implemented at the moment because of the need of supporting existing
  plugin which handle the persistence layer. This can be revisited in
  subsequent iterations.

* Plugins are able to accept such objects, and return them.

The plugin interface being introduced with this blueprint will be called
the "v3" plugin interface, to distinguish it from the "v2" interface which
Neutron has been running since the Folsom release (2012.1). It is important
to note that the versioning here is purely informal and has absolutely
no relationship with REST or RPC API versioning. It is within the scope of
this blueprint to ensure that the v3 plugin interface works perfectly with
the Neutron v2 REST API.

Together with the plugin interface, a new management layer will be added in
Neutron's architecture. The role of this management layer is to perform
common business logic tasks which are currently performed either within
the REST layer or within the plugin, mostly in the base database plugin
class. This management layer will be initially very thin. It is likely to
grow over time to include, for instance, operations such as authorization
and quota enforcement.

The new interface will use object composition for the sake of readability
and maintanability. As an example the REST layer calls the current (v2) plugin
interface by building a method name given resource and operation as seen
in [#]_

On the other hand, with the new interface being proposed here, calls to the
plugin interface will look like the following:

::

  net = Network(...)
  net.validate()
  plugin.network_manager.create(net)

Just like the v2 interface, the v3 interface will be constituted by the sum
of the interface for "core" operations and the various interfaces for API
extensions. For this reason, every API extension which introduces new
resources and therefore extends the surface of the plugin interface will have
to be redefined in order to comply with the new interface.
To this aim, every extension should implement the resource managers for the
resources it declares.
Several extensions also "extend" existing resource by adding attributes.
These extensions are particularly difficult. While a final solution has not
yet been devised at the time of writing this document, it is likely that they
will be folded into the API objects model, even if they will still be treated
as extensions from a REST layer perspective.
However, we expect the work for adapting extensions to not be overwhelming,
even if there are some large extensions such as load balancing or VPN which
might prove challenging.

It is also worth noting that the proposal [#]_ advocates for collapsing
several API extensions into the "core" or "unified" API. While this proposal
would simplify implementation of this "v3" plugin interface, its approval
is unlikely, so this documents assumes that every extension available in
the neutron code tree will be converted.

To summarize the following changes will be included within the scope of this
blueprint:

* Introduce a new namespaced interface to the plugin layer, called the "v3"
  plugin interface (where "v3" has no reference whatsoever to REST API
  versioning)

* Both the neutron core interface [#]_ and extension interfaces as [#]_ will
  be redesigned to reflect this new interface.

  As a consequence of this change, the "v2" interface will cease to exist.
  This won't be a problem for existing plugins, because of the introduction
  of the "adaptor" discussed below.

* Implement a thin management layer beyond the plugin interface which
  will take care of performing common operations which are now delegated
  to plugins. This management layer will also be used for performing
  operations such as authorization and quota enforcement as discussed in [#]_

* Define "Resource Managers" which encapsulate CRUD operations on API
  resources such as get, list, count, create, update, and delete.
  At a very high level, a resource manager will look like the following:

::

  class NetworkManager(BaseResourceManager):

      def create(network):
          """Creates a persistent network object.

          :param network: a Network instance with the network to create
          :returns: a Network instance with the created network.
          The returned object includes the network's unique identifier
          plus any auto-generated attribute.
          """

      def update(network):
          """Makes changes a network object persistent.

          :param network: the network instance to update
          :returns: None
          """

      def delete(network_id):
          """Permanently deletes a network.

          This method accepts a network identifier rather than a network
          object to avoid unnecessarily loading data from the database and/or
          third party backend for deleting an object with a known identifier.
          :param network_id: Unique identifier of the network to delete
          :returns: None

      def get(network_id):
          """Retrieve a network object.

          :param network_id: unique identifier of the network to retrieve
          :returns: a Network object for the retrieved network
          """

      def list(**filters):
          """Return a list of networks.

          If filters are specified only networks matching those are returned.
          Please note that filters also included pagination markers
          Otherwise this method returns all networks.
          """

      def count(**filters):
          """Return the number of networks matching filters.

          This method is similar to list, but rather than returning a list of
          objects, returns a scalar describing the number of networks matching
          specified filters.
          """


* Define an adaptor (or shim) to be able to run "v2" plugins from the "v3"
  interface. This will allow for seamless backward compatibility when
  switching to the new plugin interface.
  This adaptor will be indeed implemented through a set of "special"
  resource managers which will be merely wrappers for "v2" plugin operations.

The implementation of a reference plugin (possibly based on a modular driver
approach) for the v3 interface is outside of the scope of the blueprint.
However, such a plugin is highly desirable, and the contributors to this
blueprint should strive to implement it even if only for PoC purposes.

Nevertheless, since implementing such a plugin might in turn translate into
a large refactoring activity for the ML2 plugin, it is probably advisable to
postpone this activity to the next release cycle, focusing on defining the
"v3" interface and the adaptor only in this release cycle.

::


                                                        | REST
                                                        | API                                          
                                                        | Endpoint                                     
                                                        |                                              
                                                        |                                              
                  +-------------------------------------+-------------------------------------+        
                  |                                                                           |        
  REST            |                            Pecan controllers                              |        
  Layer           |                                                                           |        
                  +----+------------+------------+------------+------------+------------+-----+        
        "V3" plugin    |            |            |            |            |            |              
        interface      |            |            |            |            |            |              
        +--------------------------------------------------------------------------------------------+ 
                       |            |            |            |            |            |              
                       |            |            |            |            |            |              
                       |            |            |            |            |            |              
              +----------------------------------------------------------------------------------+     
              |V3/V2 sh|m           |            |            |            |            |        |     
              |   +----+----+  +----+----+  +----+----+  +----+----+  +----+----+  +----+----+   |     
              |   |         |  |         |  |         |  |         |  | Floating|  | Sec     |   |     
  Management  |   | Network |  | Port    |  | Subnet  |  | Router  |  | IP      |  | Group   |   |     
  Layer       |   | Manager |  | Manager |  | Manager |  | Manager |  | Manager |  | Manager |   |     
              |   |         |  |         |  |         |  |         |  |         |  |         |   |     
              |   +----+----+  +----+----+  +----+----+  +----+----+  +----+----+  +----+----+   |     
              |        |            |            |            |            |            |        |     
              +----------------------------------------------------------------------------------+     
                       |            |            |            |            |            |              
                       |            |            |            |            |            |              
        "V2" plugin    |            |            |            |            |            |              
        ---------------+------------+------------+------------+------------+------------+------------+ 
  Plugin                                                                                               
  Layer                                                                                                

Data Model Impact
-----------------

No change in Neutron's data model is expected, at least in terms of database
schemas.

The objects introduced here represent a rather important change when it comes
to how data is transferred across interfaces. This change will definitely be
beneficial from a maintainability perspective.
On the other hand, current plugins clearly do not support it.

As depicted in the diagram from the previous section, existing
plugins will attach to the v3 interface through a shim layer. The
role of this shim layer is mostly to convert API objects into a dict
representation which is understood by v2 plugins and vice versa.

While this may seem illogical (for instance on responses there is a
dict -> object -> dict conversion from the plugin to the REST layer), it is
probably the best solution to evolve the plugin interface without impacting
all the existing plugins.

REST API Impact
---------------

None

Security Impact
---------------

We do not expect that the code changes that will be performed as part of this
blueprint will bring in any security flaw.

Moreover, it is not possible to perform an analysis of the security
implication of this kind of change without analyzing the code. Moreover, doing
so would require time and the help of security experts. Both kind of resources
are not available at the moment. For this reason an analysis of the security
impact of this change will be considered "best effort".

Notifications Impact
--------------------

The management layer inserted behind the plugin interface will take care of doing
notifications currently performed by the REST layer.

Other End User Impact
---------------------

None.

Performance Impact
------------------

This blueprint is not about performance and scalability, so no improvement on
this front is expected.

The introduction of the shim layer adds some latency in the call path; we expect
this additional latency to be negligible.
Accurate testing is however needed to ensure no performance or scalability
regressions are introduced.

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

None

Developer Impact
----------------

Developers should be encouraged to start developing new plugins against the v3
API.
Also, the proposed changes will also impact some API extensions as they will
need to use the v3 interface rather than defining a simple abstract class as
they currently do.

Appropriate developer documentation should be added to this aim.

Community Impact
----------------

The blueprint intends to minimize the impact on the community, and in
particular on its members contributing to and maintaining plugins
implementing the v2 API.
These plugins will continue to be supported. We are not yet able to make
a call on a potential deprecation date, but it is unlikely that something
like that will happen before 12-18 months.

Alternatives
------------

It should be possible to run a "dual stack" version of neutron server.
This would allow v2 plugins to have a choice of running in the "old"
environment consisting of the home grown WSGI and dict based interfaces,
or in the "new" environment built around Pecan and the v3 interface.

However, this requires the neutron team to keep supporting the "old"
way of running plugins for the foreseeable future. Such an effort is
probably not worth it, if v2 plugins can efficiently run under the v3
interface through the v3/v2 shim.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Mark McClain (markmcclain)

Other contributors:
  Salvatore Orlando (salv-orlando) [in the role of the code monkey]
  Bob Melander (bob-melander) [in the role of fw/vpn code monkey]

Work Items
----------

* Implementation of the API objects framework. It is worth noting that this
  will be leveraged and extended by the work for switching the WSGI framework
  to Pecan.
* Definition of new namespace oriented interface.
* Implementation of "manager" classes sitting between REST and plugin layer.
* Implementation of the v3/v2 adaptor.
* Verify performance and scalability impact.
* Documentation.

Dependencies
============

None

Testing
=======

This change is likely to require some change in the unit test framework, just
like the blueprint for switching the WSGI layer to Pecan. However, considering
how unit tests are currently executed in Neutron, the adoption of the v3/v2
shim should ease the transition from a unit testing perspective.

It is worth reminding that we are assuming this blueprint is a prerequisite
for the Pecan switch one. In this document we are therefore assuming that the
unit tests can still leverage the home grown WSGI framework.


Tempest Tests
-------------

Current coverage is enough considering the scope of this change

Functional Tests
----------------

No new functional test needed

API Tests
---------

No additional API tests are needed

Documentation Impact
====================

The changes in the blueprint require large updates in developer documentation

User Documentation
------------------

This change is transparent to both final users and deployers.

Developer Documentation
-----------------------

Additional developer documentation is needed for

* implementing v3 plugins
* definition extensions which expand the v3 plugin interface

References
==========

.. [#] DB plugin base class: http://git.openstack.org/cgit/openstack/neutron/tree/neutron/db/db_base_plugin_v2.py
.. [#] http://git.openstack.org/cgit/openstack/neutron/tree/neutron/api/v2/base.py#n244
.. [#] https://review.openstack.org/#/c/136760
.. [#] Core plugin interface: http://git.openstack.org/cgit/openstack/neutron/tree/neutron/neutron_plugin_base_v2.py
.. [#] L3 extension interface: http://git.openstack.org/cgit/openstack/neutron/tree/neutron/extensions/l3.py
.. [#] Pecan switch spec: https://review.openstack.org/#/c/140454/
