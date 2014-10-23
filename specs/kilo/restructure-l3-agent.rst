..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================
Restructure L3 Agent
====================

https://blueprints.launchpad.net/neutron/+spec/restructure-l3-agent

:Author: Carl Baldwin <carl.baldwin@hp.com>

The L3 agent is implemented mostly in a single python file.  At current count,
this file is just over 2,000 lines of code [#]_.  Most of the functionality is
provided by the L3NATAgent class which comprises about 75% of the file.  This
class handles everything from handling RPC messages down to sending gratuitous
arp for newly added addresses on the interfaces inside the routers’ namespaces.
This structure makes the agent very difficult to extend and modify.  This is a
bit of technical debt.  Paying it down will help enable the development of new
functionality.

.. [#] https://github.com/openstack/neutron/blob/c9bea66dfe/neutron/agent/l3_agent.py#L2047

Problem Description
===================

As mentioned in the introductory paragraph, the L3 agent has gotten out of
hand.  The structure of this code has made it difficult to extend and develop
new features.  Following is a list of responsibilities taken care of in the
l3_agent.py file and mostly in the L3NATAgent class within that file.

- Defines the L3PluginApi
- Manages link local addresses for the DVR fip namespace
- Defines the RouterInfo, basically a big struct of data about a router
- Handles router update messages from RPC in a queue
- Handles periodic synchronization of all routers
- L3NATAgent

  - Manages namespace lifecycle
  - Handles router addition removal
  - Runs metadata proxy in each router
  - Very large method called process_router

    - snat_rules, dnat_rules
    - floating_ip_address
    - external gateway
    - internal network interfaces
    - ipv6 support
    - cleanup of stale interfaces
    - static routes
    - HA router keepalive

  - DVR routers

    - rtr_2_fip
    - dvr floating ips
    - snat namespace handling
    - arp entries

  - routes_updated
  - _process_routers_loop
  - _sync_routers
  - gratuitous arp

- L3NATAgentWithStateReport

Code for DVR and HA routers is mingled throughout.  There is a lot of “if
router[‘distributed’]” this and “if ri.is_ha” that.

There is no clear strategy for resource life-cycle management of resources like
namespaces and devices.  Most of it has evolved over time as problems with the
initial implementation have been found.  Such problems are usually around them
not being deleted when they should be gone.


Proposed Change
===============

Overview
--------

This will not be a rewrite of the L3 agent from scratch.  Starting this project
as a Herculean effort designed to land as a single patch is a sure recipe for
failure.  Much of this work will be pure refactoring but not all of it.  Work
will be posted for review early and often.  Dependencies between patches will
be avoided when possible.  Each patch will stand on its own as a reasonably
reviewable improvement to the existing code base.

Proper separation of concerns is a goal however, this will be done in steps.
We'll start with the high level separation of the router abstraction from the
L3 agent.


L3 Agent
~~~~~~~~

The L3 agent will be responsible for listening for updates from RPC, queuing
the updates for processing by a worker.  It will continue to oversee the set of
routers which are managed by the agent.  If namespaces are enabled, this could
be a large set of routers.  If namespaces are not enabled, this will be a single
router.  (_process_routers, _sync_routers, routes_updated,
_router_added/removed)

The agent will still manage external networks available to routers on the
agent.

The L3PluginApi class will remain as it is in l3_agent.py.

The agent will retain the L3NATAgentWithStateReport capability.


Router
~~~~~~

A new router class will be introduced.  A lot of functionality currently
handled by the L3 agent – especially the functionality in the process_routers
method – will be encapsulated by this new class.  The current RouterInfo class
will move under this abstraction.  This class will obsolete and replace the
RouterInfo class.

The router class will be more than just a struct with data about the router.  It
will be a full-fledged class that is capable of handling the implementation of
the router.  It needs a clear and uncomplicated python API defined.

As of the Juno release, there are three kinds of routers available.  These are
distributed, highly available, and legacy routers.  A new router class
hierarchy will be added to encapsulate the details of each available type of
router.  The appropriate class will be loaded when the router instance is first
created.

Kilo or beyond will see the addition of a fourth type of router which combines
DVR and HA routers.  Adding this fourth type is out of the scope of this
blueprint.  However, adding this new type of router should be relatively easy
after this blueprint has been completed by creating a new class type which
combines the functions of the separate base classes.  This new classes should be
written in a way which efficiently makes use of the existing code in the two
base classes.  Any additional complexity in this module should only exist to work out
any coordination which needs to happen between the two classes.

The above uses inheritence to encapsulate the details of the various kinds of
routers with an abstract base router serving as the base class and the others
implemented as sub-classes.  The HA DVR type of router would then use multiple
inheritence.  Following is a dot representation of what I imagine the hierarchy
will be.  Notice that LegacyRouter is not a base class.  This reflects the fact
that "DistributedRouter is a LegacyRouter" is not a true statement.  Also,
there are two DVR classes.  This reflects the fact that non-network nodes have
the distributed part of a DVR and network nodes have the central part which
builds from the distributed part::

    digraph inheritence {
      "LegacyRouter" -> "Router"
      "DistributedRouter" -> "Router"
      "DistributedRouterCentral" -> "DistributedRouter"
      "HARouter" -> "Router"
      "HADistributedRouter" -> "HARouter"
      "HADistributedRouter" -> "DistributedRouterCentral"
    }

Given that HA and DVR are properties of individual routers and not properties
of the deployment, we will need to pay attention to the migration path from one
to another.  The code should fully expect that a router can change from one
type to another and have the capability to handle it by changing the class used
for a router.  I expect that the router should be functional with its new type
and that  any namespaces, devices, or other resources that are no longer
necessary after the router changes type will be cleaned up.  The clean up will
be handled by the resource lifecycle pattern described in the `Resource
Lifecycle`_ section.

The very long _process_router method needs to be refactored with this.  The
following responsibilities are handled here.  Eventually, these will be
abstracted behind other interfaces (like an iptables abstraction) but that work
may not be completely done as part of this effort.  At a high level, the
refactoring of this method will separate concerns like plugging interfaces to
networks from routing responsibilies.

- snat_rules, dnat_rules
- floating_ip_address
- external_gateway_added
- internal network added
- static routes

Services
~~~~~~~~

There are a few services implemented in the L3 agent in various ways.  This
blueprint will add a simple service driver model to support decoupling these
services from the L3 agent class and its inheritence hierarchy.  As stated
before, inheritence will not be used to integrate these services.  Each of the
services will be moved to a new service specific module

Essentially, the agent will be a basic container which loads services as
classes.  The routing service orchestrates the workflow for services by
dispatching router events to each of the known services sequentially.  For this
blueprint, the dispatching will likely be implemented as a simple method call
to a common service interface.  This can be expanded to support a more
pluggable model as a follow-on effort.

The services will have a reference to the router in order to access L3 function
such as adding/removing NAT rules and opening ports.

I don't intend to make any significant changes to the device driver models that
are implemented in the FW and VPN services in the scope of this blueprint.  I
don't expect this effort to have any effect on the configuration of services.
Backward compatibility will be actively preserved.  This may involve leaving
stubs in place for the VPNAgent and others to load a VPN enabled L3 agent.

Existing integration tests will be modified to work with the new structure.

The intent here will not be to make a model that is everything to everyone.
That is out of the scope of this blueprint.  The intent is to iteratively
develop an interface that will work for the following services which are
already integrated with the L3 agent.  The goal is to reduce coupling and pave
the way for a more sophisticated model which may be needed in the future.  They
will be tackled in the order listed and the interface will evolve to support
them all.

#. Metadata Proxy

  - The easiest one.  Low-hanging fruit.

#. FWaaS

  - Want to remove it as a super-class of L3NATAgent

#. VPNaaS

  - Want to remove it as a sub-class of L3NATAgent

The first step is to create a service abstract class, and then sub-classes
for the various services to use these as observers to the L3 agent.  The base
class would have no-op methods for each action that the L3 agent could notify
about, and the child classes would implement the ones they're interested in.
Each service will register as an observer.

Currently, the L3 agent (and VPN agent) load the device drivers for services.
What can be done in this first step, is, instead of doing the load, a service
object can be created. This object would do the loading and register with the
L3 agent for notifications.

The child services’ notification handlers will be populated by moving the code
in the various agent classes into the new service child classes, and adapt as
needed.

Anything more complicated than this should be considered out of the scope of
this blueprint.

Some guidelines for this work:

#. We don't need the service abstract class to be perfectly and completely
   defined in advance.  I intend to do this iteratively tackling the services
   in the order listed above.  This means that we don't review the changes to
   decouple the metadata proxy with the needs of the VPN agent in mind.
#. This initial decomposition should be done without changing any
   configuration or other deployment details.  This might mean that we leave,
   for example, a tiny stub of a VPNAgent class in place.
#. Initially, the services will get an L3 agent passed in on create, but in the
   as the blueprint progresses, a router instance can be passed to the service.

DVR Router Class
~~~~~~~~~~~~~~~~

Everything related to the floating IP namespace that was added for DVR should
be encapsulated in a driver for plugging a router in to an external network and
handle floating ip setup.  This includes the LinkLocalAllocator, dvr specific
floating ip processing, fip namespace management, connection of router to fip
(rtr_2_fip, fip_2_rtr), _create_dvr_gateway, and the management of proxy arp
entries.

HA Router Class
~~~~~~~~~~~~~~~

This encapsulation will hide the details related to starting keepalived and
creating and using interfaces needed for the HA network on which it
communicates.

Resource Lifecycle
~~~~~~~~~~~~~~~~~~

The major problem here is that resources are often left lying around beyond
their useful lifecycle.  Assumptions were made about the reliable availability
of the agent, guaranteed ordering and delivery of RPC messages, and other
unrealistic guarantees.  The new design will account for problems in these
areas.  No assumptions will be made.  This will result in a more robust
implementation.

The problem that we’ve had with this is that the agent fails to cleanup
resources when they should no longer exist.  To address this, I'm thinking of
something that supports the following pattern using namespaces as an example::

    if full_sync:
        with namespace_manager.prepare_to_clean_up_stale() as nsm:
            for router in all_active_routers:
                nsm.link_router_to_ns_somehow(router)

The __enter__ and __exit__ methods should work together to discover stale
namespaces and then clean them up.  I'm thinking maybe a namespace object
should hold a weak reference to the router that occupies it.  When the weak ref
goes stale then the namespace can be removed.  This pattern is not too
different from what exists in the code now since some earlier refactoring that
I did.  However, this effort will formalize the pattern and abstract it from
the rest of the code.  Code has been started to illustrate this pattern [#]_.

.. [#] https://review.openstack.org/#/c/130052/

The pattern can be applied to other resources such as interfaces inside of a
namespace.  We have had problems ensuring that those get removed when they are
no longer useful as well.  For devices and other resources in a router, the
active resources would all be marked each time a router is processed.  Stale
resources are then identified and removed.

There has been a problem with namespaces which are persistently difficult to
delete due to a problem in the version of iproute in use on the system [#]_ and
[#]_.  There really is nothing that can be done to remove these except to
reboot the machine.  However, the new implementation of resource lifecycle
management will hold a set of namespaces that it has tried to delete.  If the
deletion fails, it will skip this deletion in future clean up runs.  Ideally,
the operator will either keep namespace deletion disabled or upgrade the
iproute package on the system to avoid these problems.

.. [#] https://bugs.launchpad.net/ubuntu/+source/iproute/+bug/1238981
.. [#] https://bugzilla.redhat.com/show_bug.cgi?id=1062685

Configuration Handling
~~~~~~~~~~~~~~~~~~~~~~

The handling of the config options will be cleaned up a bit; there's so much
'if that' and 'if this' with config options too. Behavior needs to be properly
encapsulated so that we don't need to branch so much so often.  A few examples
examples are linked in the references [#]_ [#]_ [#]_ [#]_.

.. [#] https://github.com/openstack/neutron/blob/c9bea66dfe/neutron/agent/l3_agent.py#L584
.. [#] https://github.com/openstack/neutron/blob/c9bea66dfe/neutron/agent/l3_agent.py#L743
.. [#] https://github.com/openstack/neutron/blob/c9bea66dfe/neutron/agent/l3_agent.py#L1349
.. [#] https://github.com/openstack/neutron/blob/c9bea66dfe/neutron/agent/l3_agent.py#L1460


Data Model Impact
-----------------

None


REST API Impact
---------------

None


Security Impact
---------------

No impact is expected.  We need to be careful when reviewing code that these
changes do not introduce vulnerabilities in the agent.


Notifications Impact
--------------------

None


Other End User Impact
---------------------

None


Performance Impact
------------------

None


IPv6 Impact
-----------

We will take care to preserve all existing IPv6 functionality in Neutron.  No
changes or additions to the current IPv6 functionality are planned.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

Much of code in the l3_agent.py file will be moved out to other files.  This
refactoring will introduce better software engineering patterns to allow the
functionality to be extended, modified, and maintained more easily.

Developers who have become accustomed to the current implementation will
likely not recognize the end result.  However, they will be able to easily get
reaquainted with the new code.

To avoid problems with rebasing and potential regressions while the
heavy-lifting is being done, non-critical changes to the L3 agent should be
avoided while this work is in progress.  Mail will be sent to the openstack-dev
ML to begin a freeze on non-critical changes and another one to end it.  The
freeze will only be needed during the initial more disruptive restructuring.
As certain part stabilize, the freeze will be lifted.  For example, once the
VPN and FW services have been decoupled from the agent code -- which will be
the first step -- development on those services can continue.

Community Impact
----------------

This change is part of the approved Neutron priorities for Kilo.

It supports at least the following efforts which may also be planned for Kilo.

- Pluggable external networks blueprint (dynamic routing integration indirectly)
- Enabling HA routers and DVR to work together.
- Better integration of L3 services.
- Spinning out advanced services

Alternatives
------------

The alternative is to leave it like it is and to perform small bits of
refactoring only when it is necessary for a particular new feature.  This is
not ideal since there are already a number of things that this refactoring
needs to support.  It will slow down the development of that work if this is
delayed.

Writing a new agent and eventually deprecating the current one is another
alternative?  I've personally never had a very good experience with this
approach.  It seems to trade one set of known problems for another set of
unknown problems.  Regressions are all too common.  I prefer to restructure in
small reviewable pieces.  This does not guarantee no regressions but it can
uncover them earlier in the process and they are easier to pinpoint and fix.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  `carl-baldwin <https://launchpad.net/~carl-baldwin>`_

Other contributors:
  `amuller <https://launchpad.net/~amuller>`_
  `jschwarz <https://launchpad.net/~jschwarz>`_
  `pcm <https://launchpad.net/~pcm>`_
  `yamahata <https://launchpad.net/~yamahata>`_

.. TODO If you have expressed interest in helping and I have not added you,
   please state your interest in the review here.


Work Items
----------

I expect that some of the initial work items will need to be tackled in
sequential order because of the high degree of coupling in the code.  However,
as things are decomposed and the coupling is reduced, other work items can be
tackled in parallel.

For example, since the service agents are coupled with the L3 agent inheritence
hierarchy, they will need to be moved out before a proper router abstraction is
feasible.

#. Functional Testing for the Agent
#. Service Drivers

   - Start simple.  This won't be everything to everyone yet.  It is not meant
     to full-blown pluggable service drivers.

   #. Metadata Proxy
   #. FWaaS
   #. VPNaaS

#. Decomposition and modularization of DVR, HA, and legacy routers

   #. Create a proper abstraction of a router to replace RouterInfo

      - Can serve as an abstraction for other router implementations.  Again,
        we'll start simple to introduce the abstraction.

   #. Create the inheritence hierarchy.

      - This may be done in a few steps.  Initially, the inheritence hierarchy
        may be thin with most of the implementation still in the base class.
        Future steps will move responsibilities to the sub-classes and evolve
        the interface.


Dependencies
============

None


Testing
=======

In addition to the functional tests discussed below, effort will be made to use
existing unit tests as necessary to be sure that existing coverage is retained
and avoid regressions they were created to prevent.  The end result may look
like all of the old unit tests have been removed and new, better ones have been
written in their place.

*All* new and restructured code will be covered with proper unit test coverage.
It will be significantly easier to unit test with the new structure of the
code.  If it isn't then we're doing it wrong.

I don't plan to make an effort to add missing *unit* test coverage before the
code is restructured.


Tempest Tests
-------------

No new tempest tests are planned.


Functional Tests
----------------

Functional tests will be added from the L3 agent prior to any significant
restructuring of the agent code.  Assaf [#]_ will take the lead of this testing
effort with help from John Schwarz [#]_ and all of the other assignees listed
in this blueprint.  This includes the addition of functional tests for the new
DVR and HA [#]_ features.

.. [#] https://launchpad.net/~amuller
.. [#] https://launchpad.net/~jschwarz
.. [#] https://review.openstack.org/#/c/117994/


API Tests
---------

No new API tests are planned.


Documentation Impact
====================

None

User Documentation
------------------

None

Developer Documentation
-----------------------

New API interfaces in the code will be documented with doc strings


References
==========

https://etherpad.openstack.org/p/kilo-neutron-agents-technical-debt
https://review.openstack.org/#/c/105078/
