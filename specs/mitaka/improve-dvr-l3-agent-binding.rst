..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Improve DVR L3 Agent Binding
============================

https://blueprints.launchpad.net/neutron/+spec/improve-dvr-l3-agent-binding

Neutron code does not properly handle scheduling for DVR and often confuses why
a router has been scheduled to an L3 agent, especially in cases where an agent
could host both the centralized and a distributed piece of the router.

Problem Description
===================

There are two ways that a router gets bound to an L3 agent with DVR.  The first
is for the centralized component of the router and is similar to how legacy
routers are bound.  The second is for the distributed component and should be
very different.  Essentially, a distributed router should be hosted by any L3
agent where the same host has a DVR serviceable port.

Binding the centralized part of the router is very much like binding a legacy
router.  It should be bound to a network node.  When we want to finally enable
HA and DVR together, then it will be bound to multiple.  It makes sense to give
the operator control over the binding through the api L3 extensions.  Yet, it
is this type of binding for which a new table, CentralizedSnatL3AgentBinding,
was created.

The other kind of binding shares the same table as legacy router bindings yet
it works very differently.  This binding could be computed from other
information in the database.  Basically, you find all of the DVR serviceable
ports on a network where a router is connected.  The set of agents on the hosts
where these ports are bound is the set of agents to which the router should be
bound.  It doesn't make sense to give direct control over this type of binding
through the API.  Yet this is the binding that shares the bindings table with
legacy routers.

Can you see why we're constantly confused about DVR scheduling?

Proposed Change
===============

The proposed change is to move the binding of the centralized portion of a
router to share the table with legacy routers and hopefully eliminate the
explicit binding of the distributed parts of the router.

There is a chance of introducing a performance penalty by eliminating the
explicit binding of the distributed parts of the router in the database.  In
that case, a new table may be added for this type of binding.  However, this
will be a last resort.  Every attempt will be made to optimize queries to
enable working without an extra table.  So far, I don't think the queries will
be too complex.

It will be required to change the RPC calls between the L3 agent and the
Neutron server.  The problem is that some L3 agents can host either a
centralized part of the router, a distributed part, or both.  The current RPC
mechanisms make it difficult to distinguish which case it is.  There should be
separate RPC calls to query each type independently.

The DvrRouter class in the L3 agent will be split in to two classes:  one to
represent the centralized component and another for a distributed component of
the router.  These are two distinct components of the router that share almost
nothing in common and should be handled independently.  Yet, the code is
mingled together and has a track record of confusing which type is relevant.
The L3 agent will keep track of them independently.

Much of the special case code for scheduling the centralized component of a
router can be removed when these bindings are returned to the
RouterL3AgentBinding table.  The only exception will be in determining eligible
agents; an l3 agent in 'dvr_snat' mode can host either a legecy or a dvr
central component but one in 'legacy' mode can only host a legacy router.

Data Model Impact
-----------------

RouterL3AgentBinding rows for DVR routers will be removed.  The
CentralizedSnatL3AgentBinding table will be removed.  Before it is removed,
these rows will be migrated to the RouterL3AgentBinding table.

There are a few scenarios which should be considered:

#.  Add DVR serviceable port to a host.

    This will be a notification to a specific L3 agent based on the host to
    which the port is bound.  A query will be performed to list the router
    internal ports on the same network as the port.  The L3 agent should now be
    hosting a distributed component for each.

#.  Remove DVR serviceable port to a host.

    This is essentially the reverse of adding a port.  After deleting the port,
    the list of routers the L3 agent should handle may be reduced in which case
    the L3 agent should clean up.

#.  L3 Agent requests a list of distributed router components it should be
    hosting with details.  It should be able to request a single router in
    response to a notification or request a full list.  A query will be
    performed to list the networks associated with all of the DVR serviceable
    ports on the host.  Then, for each network, the connected routers should be
    collected in a set.  The final set should be returned to the agent with
    details.

TODO(Carl) Are there more cases?

REST API Impact
---------------

It will not be possible to modify or view the bindings of the distributed parts
of router because they will come and go as ports on the network come and go.
This should have been the case from the beginning.  It doesn't change the API
definition but it may affect what is seen through the API.  There was never any
valid use case for manipulating the these bindings through the API.

Security Impact
---------------

None

Notifications Impact
--------------------

Notifications will be adjusted where there is any ambiguity between centralized
and distributed parts of routers.  It must be clear to the L3 agent which type
is being notified.

Other End User Impact
---------------------

None

Performance Impact
------------------

This change will modify the database model and the queries that find router to
L3 agent bindings.  In the case of the distributed parts of a DVR router the
bindings must be computed by examining the ports on a network.  This is
discussed in more detail in the `Data Model Impact`_ section.  Initially, it
looks like there will not be a large performance impact.

IPv6 Impact
-----------

Distributed virtual routing does not yet benefit IPv6.  The two are expected to
coexist in that a distributed router should be able to handle IPv6 correctly
even if not in a distributed manner.  This blueprint will not break this
capability.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

None

Community Impact
----------------

This simplifies DVR a bit making it easier to understand how routers are bound
to L3 agents because it will use a more sensible approach.

Alternatives
------------

I don't see an alternative way to make DVR binding easier to understand.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  `obondarev <https://launchpad.net/~obondarev>`_

Other contributors:
  `carl-baldwin <https://launchpad.net/~carl-baldwin>`_

Work Items
----------

#. Break the DvrRouter class in to two classes.
#. Modify RPC messages to distinguish between centralized and distributed
   components.
#. Modify dvr scheduling code

TODO Flesh this out a bit

Dependencies
============

None

Testing
=======

Tempest Tests
-------------

No new tempest tests will be added.

Functional Tests
----------------

Functional tests will be developed for the L3 agent to test the handling of
combinations of centralized and distributed router components on an L3 agent.

API Tests
---------

No new api tests will be added.


Documentation Impact
====================

None

User Documentation
------------------

None

Developer Documentation
-----------------------

None

References
==========
