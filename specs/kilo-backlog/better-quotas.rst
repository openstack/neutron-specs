..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Reliable quota enforcement
==========================================

Launchpad: https://blueprints.launchpad.net/neutron/+spec/better-quotas

Enforce network resource quotas in a reliable and efficient way, without
changing the quota API interface.

Problem Description
===================

Quotas enforcement is a relatively simple task. When a request for creating
one or more resources is received, the available resource quota is checked,
and if the request cannot be satisfied an error is returned to the user.
This works perfectly under the assumption that API requests are served
serially.

Unfortunately, this assumption is not true when multiple REST API workers are
employed. The same applies also if there is a single API worker, but multiple
API servers are operating behind a load balancer.

In these conditions, two or more concurrent requests can pass quota
enforcement even if there is not enough quota available to accept all of them.

For instance, assume there are 2 REST API workers and the current quota usage
is Nmax - 1 where Nmax is the resource quota.
Two requests are then sent and dispatched to the workers.
Each worker enforces the quota before the other worker completes processing
the API request.
Both requests are accepted, leading to a overall resource usage of Nmax + 1

Therefore, leveraging multiple workers and bulk creates, a tenant could in
theory create up to Nmax * Nworkers resources. While this is unlikely to
happen, there is still a good chance that tenants could manage to use more
resources then those they've been assigned by the cloud provider.
This situation is far from ideal both for the cloud provider and the user, as
the former might end up in a situation where data center resources are
oversubscribed, whereas the latter could be billed more than expected.

For RPC API workers instead, quotas are not enforced at all.
This implies that quotas are not enforced for resources created via RPC,
usually "internal" ports. While not enforcing quotas for service ports such
as DHCP ports might be a desired thing, this is probably not the correct
behavior. Indeed these ports will still contribute to the overall usage upon
enforcement for REST API requests.

Proposed Change
===============

Add a concept of 'resource reservation', which is already adopted in other
projects such as nova and cinder.
Whenever a quota enforcement check is successful, mark the requested amount of
resources as 'reserved' immediately.

The reservation shall be removed as soon as the corresponding API operation is
completed, successfully or not. This will be ideally done upon return from the
plugin call. In case of server crashes, an expiration time should be provided
to ensure stale reservations do not prevent allocation of resources upon server
restarts.

Multiple reservations can exist at anytime for the same tenant and the same
resource. The quota check should add all the existing reservations to the
current usage count.

Even with reservations, there still is a race condition which can occur if a
worker performs the quota check before the others specify their reservations.
This can be forced using lock-free algorithms which retry an operation if there
are not the conditions to perform it.
DB integrity constraints can be used to this aim, without recurring to locking
queries or other distributed synchronization constructs.

This can be achieved by introducing a table with the only purpose of keeping
track of reservations in progress.

The resource reservation process will therefore look as it follows:

1) ACQUIRE RESERVATION FACILITY
Write down in an appropriate DB table, say reservation_locks, that a given worker
is attempting to reserve a certain amount of a given resource for some tenant.

*---------------------------------------*------------*
| RESOURCE_TYPE | TENANT_ID | LOCKED_BY | EXPIRATION |  PK: RESOURCE_TYPE,
*---------------|-----------|-----------*------------*      TENANT_ID
| network       | xxx       | worker_0  | some time  |
*---------------------------------------*------------*

The selected primary key ensures that it won't be possible for two distinct
workers to attempt to concurrently reserve the same resource. The expiration
time stamp ensures stale locks are ignored in case of server failures.

2) COMMIT TRANSACTION.
This will trigger an integrity violation error if another worker attempts to
make another reservation for the same tenant and resource type.

In this case, step 1 will be tried again for a few times. There will be an
exponential backoff time between attempts.

3) DO RESERVATION

Record the reservation in an appropriate table

*--------------------------------------------*----------------*
| RESOURCE_TYPE | TENANT_ID | BOOKING_AMOUNT |   EXPIRATION   |
*---------------|-----------|----------------*----------------*
| network       | xxx       | 2              | at some point  |
*--------------------------------------------*----------------*

4) RELEASE RESERVATION FACILITY

5) COMMIT TRANSACTION

Example:

worker 0                           worker 1
----------                         ----------
acquire_resv_facility              acquire_resv_facility
COMMIT - SUCCESS                   COMMIT - FAIL (INTEGRITY VIOLATION)
do_reservation                     RETRY
release_resv_facility              COMMIT - FAIL (INTEGRITY VIOLATION)
COMMIT - SUCCESS                   RETRY
-                                  do_reservation
-                                  COMMIT - SUCCESS
-                                  release_resv_facility
-                                  COMMIT - SUCCESS

The table for acquiring the reservation facility acts as a centralized lock,
leveraging only primary keys, which are correctly enforced even in
active/active clusters such as MySql Galera.
The algorithm should however be regarded as non-blocking since workers will
always actively retry to perform the reservation; furthermore the backoff
mechanism should prevent starvation.

It is also worth noting that the proposed expiration timestamps will prevent
stale records from preventing acquiring the reservation facility or create
'ghost' resource reservation. The expiration timeout might be configurable,
but a default of at least two minutes is advisable considering that the
current neutron implementation still suffers of DB deadlocks triggered by
eventlet, which with the default MySql setting block threads for about 50
seconds.

Data Model Impact
-----------------

This document proposes two new tables:

Table name: bookings
Attributes:
resource_type  string
tenant_id      string
booking_amount integer
expiration     datetime

Table name: booking_locks
resource_type string
tenant_id     string
locked_by     string
expiration    datetime

REST API Impact
---------------

No impact on the API interface.

Security Impact
---------------

The proposed change should not open any vulnerability which could lead to
DoS, leaking tenant information, or allowing attackers to plug into
other tenant's networks.

All the DB queries performed in the implementation of this specification
will be scoped by tenant, and built in a way to prevent SQL injection.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None.

Performance Impact
------------------

Some additional DB operations will be performed upon quota enforcement.

This change might also serialize some operations which were previously
processed in parallel by distinct workers.

While the overall performance impact is expected to be negligible, it will be
important to evaluate it upon code review.

There is also an interesting question pertaining resource usage. It is
indeed worth exploring whether it is better to count it every time or updating
their usage counter whenever resources are created or deleted.
To this aim, the cost of SELECT queries vs the cost of adding db hooks on
resource create/delete and performing the corresponding UPDATE query should be
carefully compared.
The gain/loss in terms of performance depends on relative frequency of GET
operations vs POST/DELETE operations. Update is definitely more expensive,
but SELECT are way more frequent. However, it should be possible to implement
this specification leaving resource usage calculation unchanged, and then
perhaps come back to it in the future.

IPv6 Impact
------------------

None.

Other Deployer Impact
---------------------

As service ports are not anymore counted in resource usage, deployers might
expect in theory an increase in port usage. This increase should however be
contained if one considers ports are typically used with instances, and the
enforcement criteria for instances are not being altered by this specification.

Developer Impact
----------------

The implementation for this specification will come with adequate developer
documentation.

Community Impact
----------------

None, I don't think so... But one can never know.

Alternatives
------------

One alternative worth considering is to us a single table for managing
bookings.
While the tuple (resource_type, tenant_id, resource_amount) cannot be
reliably used as a primary key for serializing bookings among workers,
the "locked_by" attribute can be added to this table and the following
tuple can constitute the primary key:

(resource_type, tenant_id, locked_by)

When committing the booking the locked_by value would be erased and this
will allow other workers to make their booking.

Nevertheless, this solution does not appear to bring benefits in terms
of performance and scalability, and also makes the code less readable.

Even if it would be possible to use locking queries (e.g.: SELECT ..
FOR UPDATE) statements, this solution has already proved not ideal, and will
also miserably fail with some DB backends if active/active replication is
employed.

Moreover, an alternative non-blocking algorithms could have been constructed
along the lines of the one proposed for Nova [#]_. That algorithm has the
advantage of performing on average 1 DB transaction instead of 2 for each
quota enforcement operation, so it is therefore more efficient.
On the other hand, the cost of a retry operation in case of conflict would
be slightly higher. While the latter details is not really important, this
alternative algorithm cannot be applied to Neutron as it does not have
resource usage counters; without them the implementation of this alternative
lock free algorithm would be rather difficult.

Another alternative would consist in a separate quota granting authority.
This is a possibility, and that is what the 'Blazar' [#]_  resource
reservation project advocates for, but might result in an overkill for many
OpenStack deployments. Moreover, while in theory it is possible to use
Blazar to this aim, the project has actually been conceived to book resources
which are meant to be time-shared across tenants.
On the other hand the 'Boson' [#]_ project might represent a more viable
alternative were quota management and enforcement are delegate to a 3rd
application. While this project is very interesting, its not yet in a
developments stage such that it can be considered for adoption by Neutron.

Finally, this problem can also be solved by introducing a distributed lock
among API workers. memcached or zookeeper could be used with relative
ease to implement this sort of distributed coordination.
Nevertheless, there is probably no need to resort to distributed
coordination if a lock-free algorithm can be devised just leveraging DB
integrity.

Implementation
==============

Assignee(s)
-----------

salv-orlando

Work Items
----------

1) Preliminary yak shaving - refactor existing quota module
2) Add resource booking logic and use them in quota enforcement
3) Remove service ports from quota enforcement

Dependencies
============

The big dependencies for this change could be the following:
1) removal of self-grown WSGI framework and subsequent switch to pecan
2) review of the plugin interface.

While the above work items pretty define new hooks for performing quota
enforcement, they won't change the logic of the quota enforcement module,
which can therefore be implemented orthogonally.

Testing
=======

Tempest Tests
-------------

None. This specification does not change the API interface nor any change
which might have an impact on the integrated gate.

Functional Tests
----------------

Appropriate functional tests will be added to validate correct quota
enforcement. As a proper verification will require triggering the race
condition, some sort of fault injection might be needed. The need and
feasibility of this will be evaluated separately.

API Tests
---------

No further API test is needed.

Documentation Impact
====================

User Documentation
------------------

Document that service ports won't count anymore in the overall
resource usage.

Developer Documentation
-----------------------

As there is currently no developer documentation for quotas, this is rather
easy: "do developer documentation for the quota enforcement module"

References
==========

.. [#] Nova lock-free quotas: https://review.openstack.org/#/c/135296
.. [#] Blazar project: https://wiki.openstack.org/wiki/Blazar
.. [#] Boson project: https://wiki.openstack.org/wiki/Boson
