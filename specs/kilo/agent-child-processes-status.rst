..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Agent child processes status
============================

https://blueprints.launchpad.net/neutron/+spec/agent-child-processes-status

Neutron agents spawn external detached processes which run unmonitored, if
anything happens to those processes neutron won't take any action,
failing to provide those services reliably.

We propose monitoring those processes, and taking a configurable action,
making neutron more resilient to external failures.

Problem Description
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

Proposed Change
===============

I propose to monitor the spawned processes using the
neutron.agent.linux.external_process.ProcessMonitor class, which relies
on the ProcessManager to check liveness periodically.

If a process that should be active is not, it will be logged, and we
could take any of the following admin configured actions, in the
configuration specified order.

* Respawn the process: The failed external process will be respawned.
* Exit the agent: for use when an HA service manager is taking care of the
  agent and will respawn it, optionally in a different host. During exit
  action, all other external processes will be left running as for any
  other agent stop. So there is no downtime for the unaffected tenant network
  until the HA solution takes care of failing over the agent. In case of
  failover, responsibility for cleanup (processes and ports) lies on
  neutron-netns-cleanup and neutron-ovs-cleanup.

In future follow ups, we plan to implement a notify action to the process manager
when the corresponding piece lands in oslo [#oslo_service_status]_.

Examples of configurations could be:

* Disabled, external processes are not polled for liveness

::

  check_child_processes_period  = 0

* Log (implicit) and respawn

::

  check_child_processes_action = respawn
  check_child_processes_period = 60

* Log (implicit) and notify

::

  check_child_processes_action = notify
  check_child_processes_period = 60

* Log (implicit), and exit

::

  check_child_processes_action = exit
  check_child_processes_period = 60

This feature will be enabled by default (60 seconds), and default
action will be 'respawn'.

Data Model Impact
-----------------

None

REST API Impact
---------------

None

Security Impact
---------------

None

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

Some extra periodic load will be added by checking the underlying
children. Locking of other green threads will be diminished by starting
a green thread pool for checking the children. A semaphore is introduced
to avoid several check cycles from starting concurrently.

As there were concerns on polling /proc/$pid/cmdline, I implemented a
simplistic benchmark:

::

  i=10000000
  while  i>0:
    f = open ('/proc/8125/cmdline','r')
    f.readlines()
    i = i - 1


Please note that the cmdline file is addressed by kernel functions [#kernel_cmdline]_
in memory and does not rely on any I/O over a block device, that means there is no
cache speeding up the read of this file which would invalidate this benchmark.

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


IPv6 Impact
-----------

No effect on IPv6 expected here.


Other Deployer Impact
---------------------

People implementing their own external monitoring of the subprocesses, may
need to migrate into the new solution, taking advantage of the exit method,
or a later notify one when that's available.

Developer Impact
----------------

Developers which spawn external processes may start using ProcessMonitor
instead of using ProcessManager directly.

Community Impact
----------------

This change has been discussed several times on the mailing list, IRC,
and previously accepted for Juno, but didn't make it to the deadline
on time. It's something desired by the community, as it makes neutron
agents more resilient to external failures.

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

* Changing the restart mechanism of agents to an execve from inside
  the agent itself (via signal capture). The execve system call
  retains original PID and children PID relationship, thus we
  could wait on children pid. But this prevents stop/start capability
  of agents which could be handy during maintenance and development.
  If we decide to change this in the future, ProcessMonitor implementation
  could be easily modified to non-polling-wait on pids without changing
  any of it's API.

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

* ProcessMonitor, and functional testing: done
* Implement in dhcp-agent, refactoring the code duplication
  with neutron.agent.linux.external_process. [#dhcp_impl]_
* Implement in l3-agent [#l3_impl]_
* Implement in lbaas-agent
* Implement in vpnaas-agent

Notes: a notify action was planned, but it's depending on a new oslo feature,
this action can be added later via bug process once the oslo feature is accepted
and implemented.

Dependencies
============

The notify action depends on the implementation of [#oslo_service_status]_,
but all the other features/actions can be acomplished without that.

Testing
=======

Tempest Tests
-------------

Tempest tests are not capable of doing arbitrary execution of command
in the network nodes (killing processes for example). So we can't use
tempest to check this without implementing some sort of fault injection
in tempest.

Functional Tests
----------------

Functional testing is used to verify the ProcessMonitor class, in charge
of the core functionality of this spec.

API Tests
---------
None

Documentation Impact
====================

User Documentation
------------------

The new configuration options will have to be documented per agent.

This are the proposed defaults:

::

  check_child_processes_action = respawn
  check_child_processes_period  = 0

Developer Documentation
-----------------------
None

References
==========

.. [#dhcp_impl] DHCP agent implementation:
   https://review.openstack.org/#/c/115935/

.. [#l3_impl] L3 agent implementation:
   https://review.openstack.org/#/c/114931/

.. [#dhcp_agent_bug] Dhcp agent dying children bug:
   https://bugs.launchpad.net/neutron/+bug/1257524

.. [#liveness_bug] L3 agent dying children bug:
   https://bugs.launchpad.net/neutron/+bug/1257775

.. [#check_metadata] Brian Haley's implementation for l3 agent
   https://review.openstack.org/#/c/59997/

.. [#oslo_service_status]  Oslo service manager status notification spec
   http://docs-draft.openstack.org/48/97748/3/check/gate-oslo-specs-docs/ef96358/doc/build/html/specs/juno/service-status-interface.html]

.. [#waitpid] http://linux.die.net/man/2/waitpid

.. [#waitpid_non_child] http://stackoverflow.com/questions/1058047/wait-for-any-process-to-finish

.. [#kernel_cmdline] https://github.com/torvalds/linux/blob/master/fs/proc/cmdline.c#L8

