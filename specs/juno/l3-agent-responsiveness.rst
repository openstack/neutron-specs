..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
L3 Agent Responsiveness
=======================

https://blueprints.launchpad.net/neutron/+spec/l3-agent-responsiveness

:Author: Carl Baldwin <carl.baldwin@hp.com>
:Copyright: 2014 Hewlett-Packard Development Company, L.P.

On agent restart, the L3 agent loops through all routers to be sure they're in
sync with the database.  This task can take over an hour on a heavily loaded
system because of rootwrap, sudo and other inefficiencies.  This task locks out
RPC processing until it is done.  From a user's perspective, it appears that
the system is completely unresponsive to floating ip and port changes.


Problem description
===================

On agent restart, the L3 agent immediately kicks off a periodic task called
*_sync_routers_task*.  This task grabs a semaphore which locks out the
*_rpc_loop* until it is done.  This makes the L3 agent unresponsive to new work
coming in via RPC.  Floating IPs need to wait to become active or inactive,
router gateways don't get plugged or unplugged, and subnet ports cannot be
manipulated.  This gives a poor impression to a user who has just made an API
call to get something done.


Proposed change
===============

Overview
--------

This blueprint proposes unifying *_sync_routers_task* and *_rpc_loop* in to a
single processing loop.  This single loop will give priority to RPC messages.
In other words, an RPC message about a given router will bump that router ahead
in the queue before all of the routers that are in the queue from the
*_sync_routers_task* code path.

The justification for prioritizing in this way is that *_sync_routers_task*
requests maintenance updates to routers.  It is meant to catch the somewhat
unlikely case that a change was made to a router while the agent was down.  RPC
messages generally represent changes to the system that are being requested
through the API in the moment.  When you consider this, it is clear that RPC
messages should be given precedence to improve the user experience.

To be fair, the *_sync_routers_task* is also helpful if the system reboots
after a crash.  In this case, these updates are more than just maintenance.
However, in this case, each router on the system is already down.  It is still
prudent to respond to user requests with priority.

Each update will carry a timestamp so that they can be prioritized by time if
there are many updates at once with the same priority.

Parallelism
~~~~~~~~~~~

The current L3 implementation allows processing many routers in parallel.  In
fact, there is virtually no bound on the number of routers that can be
processed in parallel except for the limit of 1000 grean threads.  In reality
though, it is not practical to process more than 4-8 routers in parallel
because there are enough contention points that prevent proper cooperation
between the threads that most of them get starved anyway.

This blueprint implementation will create a _process_routers_loop to process
all updates.  This loop will use a green thread pool of fixed size.  The loop
will continuously spawn worker threads to ensure that the maximum number of
workers are either processing a router or waiting on the queue for the next
update to come in.

The size of the worker pool can easily be made configurable in a follow on to
this blueprint if there is enough demand.  However, based on testing done at
scale, the size will initially be set to 8.

ExclusiveRouterProcessor
~~~~~~~~~~~~~~~~~~~~~~~~

A new worker will spawn and immediately call _process_router_update.  This
method will immediately look to the queue for the next router update.  It will
block (friendly green thread style block) until one is available.

At this point, there are some timing and coordination issues to consider.  The
ExclusiveRouterProcessor class was designed to take care of these.

First, since there are multiple workers and many routers, we don't want to have
multiple workers touching the same router at once.  To avoid this, the queue
implementation will return an instance of ExclusiveRouterProcessor that
guarantees that worker has exclusive access to the router, even if other update
messages come in while it is being processed.  This worker will be considered
the master for this router until updates are finished.

Second, there is the possibility that a new update for the router will come in
and bubble to the front of the priority queue while the router is being
processed with outdated information.  To handle this case, the worker that
picks up this new update will try to get exclusive access to the router by
creating an instance of ExclusiveRouterProcessor.  This instance will realize
that it is not the master processor and will simply append the update to the
list of updates that the master instance will process.

When the master instance is done processing the router, it will check its queue
of updates to see if the router needs to be processed again.  If another update
is found, it will process the router again, fetching new information from the
DB.  It will loop until there are no more updates.  This covers the case where
a user is actively making updates to a router over a period of time.  The
router simply needs to be processed several times in a row to respond to these
updates until the user is finished.

It is important to note here that the new update must have bubbled all the way
to the front of the priority queue and a worker needs to grab it off of the
queue before the router processor will loop on the router and process it
multiple times in a row.  Without this important distinction, the algorthim
would be subject to a denial of service attack where 8 routers could completely
starve all of the other routers on the system.

The complexity in this class is mostly around making a guarantee that there
is only one master processor for any given router.  The rest is around the
`Timing`_ issues discussed next.

Timing
~~~~~~

Each update carries a timestamp that is initialized to the time when the update
was received.

The ExclusiveRouterProcessor class carries one timestamp per router that is
updated to just before a database query fetched the latest data about the
router.  This timestamp is *not* recorded until the router has been processed
using those data.  It will be recorded by calling the fetched_and_processed
method.  This is very important because the timestamp records the age of the
data that was last used to complete an update to the router on the system.
This handles the time delta between when the data were fetched and when the
router is finally updated.

In the case of *_sync_routers_task*, the same timestamp is used for the update
and for the age of the data since the system will immediately run a query to
get all of the router data after the updates are created.  However, the new
implementation will still update all routers.  They will just be updated with
a lower priority than the RPC generated updates.

An update will be processed for a router iff the update timestamp is newer than
the most recent router_data_timestamp.

Alternatives
------------

Speeding up the L3 agent has been a work item for some time.  Progress has been
made in this area during the Icehouse time frame.  For example, sudo was found
to have an inefficiency that added 100 milliseconds or more to each invocation.
This affected the L3 agent's ability to plumb routers in a timely manner.

In the Juno timeframe, we will get a new daemon mode for rootwrap which will
speed up the agent a great deal.

The bottom line is that speeding up the agent will not be enough.  On an agent
hosting hundreds of routers, there will still be a significant delay caused by
the *_sync_routers_task* which will affect the end user experience.

Data model impact
-----------------

None

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

Improved responsiveness to L3 changes made through the API following an agent
restart.

Other deployer impact
---------------------

This change will allow deployers of large scale cloud deployments using L3
agent to breathe easier.  They will be able to deploy updates to the code base,
restart the L3 agents and not worry about the effect it has on the system's
overall responsiveness.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  `carl-baldwin <https://launchpad.net/~carl-baldwin>`_

Other contributors:
  None

Work Items
----------

https://review.openstack.org/#/c/78819

Dependencies
============

None

Testing
=======

No new gate tests will be required as this does not change functionality.  The
implementation will be fully unit tested including new tests to cover the
functionality of the priority queue and router processor.


Documentation Impact
====================

None


References
==========

https://review.openstack.org/#/c/78819
