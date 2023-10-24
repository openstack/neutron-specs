..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================================================
L3/BGP - Make BGP Speaker peer sessions resilient to infrastructure outage
==========================================================================

https://bugs.launchpad.net/neutron/+bug/2006145


This RFE intends to implement resilient support for BGP peer sessions
established by the BGP speaker. Nowadays, the Neutron Dynamic Routing service
architecture depends on some type of message between the DRAgent service and
the Neutron server. However, these communications are strongly dependent on the
messaging service availability (such as RabbitMQ), and any transient/permanent
failures in OpenStack infrastructure nodes may affect prefix advertising via
BGP.


Problem Description
===================

When we are using dynamic routing with BGP to advertise network address
prefixes in a cloud infrastructure, all the network traffic depends on the
correct operation by the BGP peer elements. The DRAgent has a crucial role in
that scenario because this service is responsible for advirtise all L3 prefixes
in a North/South communication, being a single point of vulnerability for the
cloud operation.

The current DRAgent reference design [1]_ using two separated periodic tasks as
described below:

State Report periodic task (DrAgentWithStateReport class)::

 +------------------------------------------------+
 |DrAgentWithStateReport class                    |
 |                                                |
 |     +------------------------------------+     |
 |     |heartbeat = _report_state           |     |
 |     |interval=CONF.AGENT.report_interval |     |
 |     |  call _report_state                |     |
 |     +------------------+-----------------+     |
 |                        |                       |
 |     +------------------+-----------------+     |
 |     |_report_state polling task          |     |
 |     |                                    |     |
 |     | RPC processing                     |     |
 |     |   if agent_status == revived       |     |
 |     |     call schedule_full_resync      |     |
 |     +------------------+-----------------+     |
 |                        |                       |
 |     +------------------+-----------------+     |
 |     |schedule_full_resync                |     |
 |     +------------------------------------+     |
 |                                                |
 +------------------------------------------------+

BGP DRAgent periodic task (BgpDrAgent class)::

 +---------------------------------------+
 |BgpDrAgent class                       |
 |                                       |
 |   +-------------------------------+   |
 |   |periodic_resync polling task   |   |
 |   |interval=CONF.periodic_interval|   |
 |   |  call _periodic_resync_helper |   |
 |   +---------------+---------------+   |
 |                   |                   |
 |   +---------------+---------------+   |
 |   |_periodic_resync_helper        |   |
 |   | if full_sync or resync/reason |   |
 |   |   call sync_state             |   |
 |   +---------------+---------------+   |
 |                   |                   |
 |   +---------------+---------------+   |
 |   |sync_state                     |   |
 |   |                               |   |
 |   | if bgp_speaker_id == None     |   |
 |   |   remove speaker from DRAgent |   |
 |   |                               |   |
 |   | Exception (MessagingTimeout)  |   |
 |   |   call schedule_full_resync   |   |
 |   +--------------+----------------+   |
 |                  |                    |
 |   +--------------+----------------+   |
 |   |schedule_full_resync           |   |
 |   +-------------------------------+   |
 |                                       |
 +---------------------------------------+

These two periodic tasks have independently configured `interval` ranges, the
`periodic_interval` for BGP DRAgent task, and the `AGENT.report_interval`
for the heartbeat report state. However, a full resync request can come from
both periodic tasks, and the sync_state method will be processed. The case of
the RPC heartbeat response with agent_status `revived` activates this entire
flow described above until the speaker is removed from the DRAgent. For the
case where an `exception` occurs in the sync_state method, a full resync is
scheduled (according to the flow described above) until the speaker is removed
from the DRAgent.

Regardless of the origin of the full resync request, the caching mechanism
apparently is not working, as the agent was previously configured and after
receiving a `revived` status via RPC it simply removes the speaker from the
DRAgent, and reschedules a future resync to later be re-included (depending on
the configured periodic intervals). When the speaker is removed from DRAgent
the BGP peer sessions are closed and will only be reestablished if the speaker
is re-added.

This problem can be manifested in two different ways:

* When the messaging service/RabbitMQ is offline and does not respond for a
  long interval `AMQP server is unreachable` (note the timeouts configured for
  Neutron agent_down); and then RMQ/RPC becomes active again.

* When the messaging service/RabbitMQ queues are under a lot of pressure and/or
  experiencing `exception` timeouts, such as: `Timed out waiting for a reply`;
  and then RMQ/RPC becomes active again.


Proposed Change
===============

To solve the problem described above, the proposal is to introduce a new
speaker cache logic for the DRAgent can keep the speaker settings and the BGP
peer sessions in case of RPC Exceptions, and/or reestablishment of
communication via RPC. In addition, to not remove the BGP speaker configuration
from DRAgent due to transient failures in RPC communication, it is required to
change the `get_bgp_speakers` empty returns handling logic to first schedule a
full sync, and then allow the BGP speaker to be removed in the next periodic
sync.

To enable the new DRAgent speaker cache mechanism, a new config option should
be enabled via [BGP] section in bgp_dragent.ini file.

.. code::

  * ``speaker_cache_timeout = 300``

This configuration option enable the DRAgent to keep the speaker settings and
related BGP peer sessions until the configured timeout (in seconds). The
purpose of this timeout cache is prevent errors in the resync checking logic
so that no speaker removed until the timeout condition is satisfied.

The default value for `speaker_cache_timeout` must be zero and thus maintain
the current DRAgent behavior. Any non-zero value should affect DRAgent behavior
as described below:

* State Report periodic task: with the `revived` RPC status, the DRAgent must
  be start the cache timeout timer, and set the `cache_out_of_sync` flag as
  True. If the DRAgent performs the sync_state method before the
  `cache_out_of_sync` becomes to False, then the full sync process must be
  ignored, and wait for the next periodic check.

* DRAgent periodic task: if the sync_state method throws any `exception` during
  operation, the DRAgent must be start the cache timeout timer, and set the
  `cache_out_of_sync` flag as True. Similar to the case described above, if the
  DRAgent performs the sync_state method before the `cache_out_of_sync` becomes
  to False, then the full sync process must be ignored, and wait for the next
  periodic check.

* Cache timeout task: if another `revived` or `Exception` event occurs during
  the timer count, the cache task must reset the timeout interval count and
  start again. Otherwise, the cache task must terminate at the expiry of the
  configured timeout, and set the `cache_out_of_sync` flag to False.

* Default workflow: If the `speaker_cache_timeout` is empty or set to zero; or
  if the configured cache timeout timer has expired - `cache_out_of_sync` flag
  is False; any periodic task should be run a full resync.

DB Impact
---------

None

Rest API Changes
----------------

None


Implementation
==============

Assignee(s)
-----------

* Primary assignees:
  Roberto Bartzen Acosta <rbartzen@gmail.com>

Work Items
----------

* Implement a new cache logic in DRAgent speaker resync.

* Implement relevant unit and functional tests using the existing facilities
  in Neutron Dynamic Routing.

* Write documentation.


Documentation Impact
====================

User Documentation
------------------

* Information about the DRAgent speaker caching support.


Testing
=======

* Unit/functional tests.


References
==========

.. [1] https://opendev.org/openstack/neutron-dynamic-routing/src/branch/master/neutron_dynamic_routing/services/bgp/agent/bgp_dragent.py
