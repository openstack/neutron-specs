..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Agent child processes status
============================

https://blueprints.launchpad.net/neutron/+spec/agent-child-processes-status

Neutron agents spawn children which run unmonitored, if anything happens to
those children neutron won't take any action, failing to provide those
services reliably.

We propose monitoring those processes, and taking a configurable action,
making neutron more resilient to external failures.

Problem description
===================

When a ns-metadata-proxy dies inside an l3-agent [#liveness_bug]_,
subnets served by this ns-metadata-proxy will have no metadata until there
are any changes to the router, which will recheck the metadata agent
liveness.

Same thing happens with the dhcp-agent [#dhcp_agent_bug]_ and also
in lbaas and vpnaas agents.

This is a long known bug, which generally would be triggered
by bugs in dnsmasq, or the ns-metadata-proxy, and specially critical
on big clouds and HA environments.

Proposed change
===============

I propose to monitor the spawned children checking the
neutron.agent.linux.external_process.ProcessManager class .active method
periodically, spawning a pool of green threads which would avoid excessive
locking. The same approach that Bryan Haley started here [#check_metadata]_.

If a process that should be active is not, it will be logged, and we
could take any of the following admin configured actions, in the
configuration specified order.

* Respawn the process.
* Notify the process manager [#oslo_service_status]_.
* Exit the agent. (for use when an HA service manager is taking care of the
  agent and will respawn it, optionally in a different host).


Examples of configurations could be:

* Log (implicit) and respawn

::

  check_child_processes = True
  check_child_processes_actions = respawn
  check_child_processes_period  = 60

* Log (implicit) and notify

::

  check_child_processes = True
  check_child_processes_actions = notify
  check_child_processes_period  = 60

* Log (implicit), notify, respawn

::

  check_child_processes = True
  check_child_processes_actions = notify, respawn
  check_child_processes_period  = 60

* Log (implicit), notify, exit

::

  check_child_processes = True
  check_child_processes_actions = notify, exit
  check_child_processes_period  = 60

This feature will be disabled by default, and default
action will be 'respawn'.

Alternatives
------------

* Use popen to start services in the foreground and wait on SIGCHLD
  instead of polling. It wouldn't be possible to reattach after
  we exit or restart an agent because the parent will detach from
  the child and it's not possible to reattach when agent restarts
  (without using ptrace which sounds too hackish). This is a
  POSIX limitation.
  In our design, when an agent exits, all the underlying children
  stay alive, detached from the parent and continue to run
  to make sure there is no service disruption during upgrades.
  When the agent starts again, it will check in /var/neutron/{$resource}/
  for the pid of the child that serves each resource, and it's
  configuration, and make sure that it's running (or restart it
  otherwise). This is the point we can't re-attach, or wait [#waitpid]_
  for an specific non-child PID [#waitpid_non_child]_.

* Use a intermediate daemon to start long running processes and
  monitor them via SIGCHLD as a workaround for the problems in the first
  alternative. This is very similar to the soon-to-be available
  functionality in oslo rootwrap daemon, but rootwrap daemon won't
  be supporting long running processes yet, even though the problem
  with this alternative is the case when the intermediate process
  manager dies or gets killed. In that case we lose control
  over the spawn children (that we would be monitoring via SIGCHLD).

* Instead of periodically checking all children, spread the load
  in several batches over time. That would be a more complicated
  implementation, which probably could be addressed on a second
  round or as a last work item if the initial implementation doesn't
  perform as expected for a high amount of resources (routers, dhcp
  services, lbaas..).

* Initially, the notification part was planned to be implemented
  within neutron itself, but the design has been modularized in
  oslo with drivers for different types (systemd, init.d, upstart..).


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

Some extra periodic load would be added by checking the underlying
children. Locking of other green threads would be diminished by starting
a green thread pool for checking the children. A semaphore will be introduced
to avoid several check cycles from starting concurrently.

As there were concerns on polling /proc/$pid/cmdline, I implemented a
simplistic benchmark:

::

  i=10000000
  while  i>0:
    f = open ('/proc/8125/cmdline','r')
    f.readlines()
    i = i - 1


Please note that the cmdline file is addressed by kernel functions in memory
and does not make any I/O.

::

  root@ns316109:~# time python test.py
  real  0m59.836s
  user  0m23.681s
  sys 0m35.679s


That means, 170.000 reads/s using 1 core / 100% CPU on a 7400 bogomips machine.

If we had to check 1000 children processes we would need 1000/170000 = 0.0059
seconds plus the overhead of the intermediate method calls and the spawning
of greenthreads.

I believe ~ 6ms CPU usage to check 1000 children is rather acceptable, even
though the check interval is tunable, and it's disabled by default
to let the deployers balance the performance impact with the failure detection
latency.

Polling isn't ideal, but the alternatives aren't either, and
we need a solution for this problem, specially for HA environments.


Other deployer impact
---------------------

None

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

* https://launchpad.net/~mangelajo
* https://launchpad.net/~brian-haley

Adding brian-haley as I'm taking a few of his ideas, and reusing
partly his work on [#check_metadata]_.


Work Items
----------

* Implement in l3-agent, modularizing for reuse in other agent,
  implement functional testing.
* Implement in dhcp-agent, refactor to use external_process to
  avoid code duplication.
* Implement in lbaas-agent
* Implement in vpnaas-agent
* Implement in any other agents that spawn children.
* Implement the notify action once that's accepted and implemented
  in oslo.



Dependencies
============

The notify action depends on the implementation of [#oslo_service_status]_,
but all the other features/actions can be acomplished without that.

Testing
=======

Unit testing won't be enough and Tempest is not capable of running arbitrary
ssh commands on hosts, killing processes remotely to test this.

We propose the use of functional testing to validate the functionaly
proposed e.g.

* Create a parent process (e.g. l3-agent) responsible for launching/monitoring a
  child process (e.g neutron-ns-metadata-proxy)
* Kill the child process.
* Check that the configured actions are successfully performed (e.g. logging
  and respawn) within a reasonable interval.


Documentation Impact
====================

The new configuration options will have to be documented per agent.

This are the proposed defaults:

::

  check_child_processes = False
  check_child_processes_actions = respawn
  check_child_processes_period  = 60



References
==========

.. [#dhcp_agent_bug] Dhcp agent dying children bug:
   https://bugs.launchpad.net/neutron/+bug/1257524

.. [#liveness_bug] L3 agent dying children bug:
   https://bugs.launchpad.net/neutron/+bug/1257775

.. [#check_metadata] Brian Haley's implementation for l3 agent
   https://review.openstack.org/#/c/59997/

.. [#oslo_service_status]  Oslo service manager status notification spec
   http://docs-draft.openstack.org/48/97748/3/check/gate-oslo-specs-docs/ef96358/doc/build/html/specs/juno/service-status-interface.html]

.. [#oslo_sn_review] Oslo spec review
   https://review.openstack.org/#/c/97748/

.. [#old_agent_service_status] Old agent service status blueprint
   https://blueprints.launchpad.net/neutron/+spec/agent-service-status

.. [#waitpid] http://linux.die.net/man/2/waitpid

.. [#waitpid_non_child] http://stackoverflow.com/questions/1058047/wait-for-any-process-to-finish
