..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Report HA Router Master
=======================

https://blueprints.launchpad.net/neutron/+spec/report-ha-router-master

Highly available routers is a new functionality that was merged in the
l3-high-availability blueprint. HA routers are scheduled on multiple L3 agents
however the cloud operator has no way of knowing where the active instance
is.

Problem Description
===================

A cloud operator can know which L3 agents are providing a router, but not
where the active instance is. Legacy routers may be manually moved from
one agent to another. With HA routers, the equivalent is moving the active
instance, but that is not currently possible.
The first step is to know where the active instance, which will be addressed
in this blueprint, however setting the location of the active instance is
out of scope and will be addressed in the future.

The operator might want to perform node maintenance which is assisted by
manually moving routers from the node. Likewise the operator might want
to see the state of routers after a failover (Did the active instance
actually failover?).

Proposed Change
===============

l3-agent-list-hosting-router <router_id>

Currently shows all L3 agents hosting the router. It will now also show the HA
state (Active, standby or fault) of said router on every agent.

::

 +-----------+------+----------------+-------+----------+
 | id        | host | admin_state_up | alive | ha_state |
 +-----------+------+----------------+-------+----------+
 | 534c4b37- | net1 | True           | :-)   | active   |
 | da2730c6- | net2 | True           | :-)   | standby  |
 | 7abcd991- | net3 | True           | xxx   | fault    |
 +-----------+------+----------------+-------+----------+

Keepalived doesn't support a way to query the current VRRP state.
The only way to know then is to use notifier scripts.
These scripts are executed when a state transition occurs,
and receive the new state (Master, backup, fault).

Every time we reconfigure keepalived (When the router is created or updated)
we tell it to execute a Python script
(That is maintained as part of the repository).

The script will:

1) Write the new state to a file in $state_path/ha_confs/router_id/state
2) Notify the agent that a transition has occurred via a Unix domain socket.
   The reason that step 1 will happen in the script and not in the agent after
   it receives the notification is that we want to write down the state
   transition whenever it happens so that it isn't lost if the agent is down.
   keepalived does not expose a way to query for the current state, so that
   if a state transition occurred but we failed to write it down, that
   information is forever lost.

The L3 agent will start and stop the metadata proxy when it receives a
notification. This is to save on memory usage by enabling the proxy
only on the active instance. This can be important at scale as every
proxy takes 20+ MBs.

The L3 agent will batch these state change notifications over a period of T
seconds. When T seconds have passed and no new notifications have arrived it
will send a RPC message to the server with a map of router ID to VRRP state
on that specific agent. How it works is that once an event is received
by the agent, it batches all future events over a period of T seconds. When
the timer goes off, it sends all of the state changes in a single message
to the controller. Additionally, every time the agent starts it gets a list
of routers scheduled on the agent. The agent will now loop through said
routers, collect their HA states from disk and update the server.
This is to catch any state changes that occurred if and when an agent was down.
If a router changes states multiple times during the batching period, the
agent will only send the most up to date state.

The RPC message send will be retried in case the management
network is temporarily down, or the agent is disconnected from it.

The server will then persist this information following the RPC message:
The tables are already set up for this. Each router has an entry in the HA
bindings table per agent it is scheduled to, and the record contains the
VRRP state on that specific agent. The controller will also persist
the last time a state change was received, so that in a split brain situation
the admin would be able to understand which is the 'real' master by observing
the time stamps.

Optionally*, the server will look for dead agents (That have not sent
heartbeats in a while) and will mark their HA routers as down. This will
aid the main use case of a hypervisor dying (Of course not being able
to report of any state changes), and another hypervisor hosting all of the
routers. In this case the API will return 'active' for all routers on both
machines until the server notices that the first agent died and marks
its routers as down.

* This is an optional enhancement that could be added after the enhancement
  lands if we find it correct.

Data Model Impact
-----------------

The HA state of every router to agent binding is persisted in the
L3HARouterAgentPortBinding table. It is currently unused. A DB migration
will be necessary in order to add time stamps as well as the 'fault'
state, as currently only the 'active' and 'standby' can be persisted.

REST API Impact
---------------

l3-agent-list-hosting-router will now return an extra column that can be
'active', 'standby' or 'fault' for HA routers, or None for other types of
routers.

Security Impact
---------------

keepalived runs as root, as does the transition script that it invokes.
The transition script talks to the agent via a Unix domain socket.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

python-neutronclient will support the new ha_state column. It will show
'active', 'standby' or 'fault' when a proper response is received. '-' will be
displayed if None is received by an old server or for non-HA routers.

Performance Impact
------------------

Assuming two L3 agents and 1,000 routers hosted on each, a failover from node
1 to node 2 should induce only a single RPC call from node 2 to the server,
and a single DB transaction.

IPv6 Impact
-----------

None.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

None.

Community Impact
----------------

None.

Alternatives
------------

Instead of neutron-keepalived-state-change notifying the agent via a Unix
domain socket, the agent could poll for the state of all HA routers every
T seconds. It would then diff the new states against a cached copy
and notify the server of any changes. One could argue that this is simpler
to implement and maintain, but is less performant.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Assaf Muller <amuller>

Work Items
----------

* Current keepalived notifier bash scripts are generated in-line. These will
  now be a Python script maintained as part of the repository. The script
  will be available as neutron-keepalived-state-change and will be invoked
  by keepalived.
* At first the script will replicate the existing behavior of the bash
  scripts: Write the new state to disk and start up or shut down the metadata
  proxy.
* The script must also notify the agent of the state change via a Unix domain
  socket. Starting and stopping the metadata proxy will be moved ot the agent.
* The RPC message that updates HA routers states will be implemented (It
  currently actually already exists but cannot be used without changing
  its format).
* The agent will batch up state change notifications in to a single RPC
  message. The Nova notifier mechanism batches notifications and the code
  will be reused.
* The API must expose the new ha_state column.
* The L3 agent must report HA states after it starts.
* Add the fault state and state change timestamp via a DB migration patch.
* Optional: The controller will look for dead agents and move their HA routers
  to the fault state.

Dependencies
============

None.

Testing
=======

Tempest Tests
-------------

L3 HA cannot be tested in Tempest without multi-node support. L3 HA is the
first candidate to be tested when in-tree integration tests are introduced
via the integration-tests blueprint.

Functional Tests
----------------

The L3 agent already has functional testing in place. Two new tests will
be added:

1) When a state change occurs, that the notification arrives at the agent.
2) When multiple state changes occur, that the RPC call is sent to the server
   with the expected parameters.

API Tests
---------

The RPC and DB methods will be tested with unit tests.

Documentation Impact
====================

The changes to the API and CLI require documentation.

User Documentation
------------------

The CLI client documentation must be updated.

Developer Documentation
-----------------------

The Neutron API change must be documented.

References
==========

* Topic branch:
  https://review.openstack.org/#/q/branch:master+topic:bug/1365453,n,z
* https://blueprints.launchpad.net/neutron/+spec/l3-high-availability
* https://bugs.launchpad.net/neutron/+bug/1365453

