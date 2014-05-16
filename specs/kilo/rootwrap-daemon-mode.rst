..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================
rootwrap daemon mode
====================

https://blueprints.launchpad.net/neutron/+spec/rootwrap-daemon-mode

Neutron is one of projects that heavily depends on executing actions on network
nodes that require root priviledges on Linux system. Currently this is achieved
with oslo.rootwrap that has to be run with sudo. Both sudo and rootwrap produce
significant performance overhead. This blueprint covers mitigating the sudo
and rootwrap introduced part of the overhead by using the new mode of
rootwrap operation called 'daemon mode'.

Problem Description
===================

As Miguel Angel Ajo stated in [#ml]_:

..

  On a database with 1 public network, 192 private networks, 192 routers, and
  192 nano VMs, with OVS plugin:

  Network node setup time (rootwrap): 24 minutes

  Network node setup time (sudo):     10 minutes

As you can see, rootwrap presents major overhead comparing to plain sudo usage
and most of this overhead is due to long Python interpreter startup time
required for every request.
Details of the overhead are covered in [#rw_bp]_.

Proposed Change
===============

This blueprint proposes adopting oslo.rootwrap daemon model which allows
to run rootwrap as a daemon. The daemon works just as a usual rootwrap but
will accept commands to be run over authenticated UNIX domain socket instead of
command line, running continuously in background.
An overview of rootwrap daemon is at [#rw_eth]_.

Two new configuration options should be added:

* ``rootwrap_mode`` = daemon | process will make agents use daemon
  or the usual rootwrap process; daemon will be enabled by default.
  This option could be deprecated in the future if we consider the daemon
  mode performs as we expect.

* ``rootwrap_config`` will provide path to rootwrap config file used to run
  daemon, by default it will point to the neutron provider rootwrap files.

Since ``root_helper`` is passed around in Neutron (as opposed to use of global
config in other projects) we have to make root_helper as some object
encapsulating all necessary info to run commands with root priviledges.
This is done in the first patch [#cr_rh]_.

The patch introduces two root helper classes:

* ``ProcessHelper`` (base) simply runs command as is, with no wrappers;
* ``WrappedProcessHelper`` runs commands with a wrapper just as ``root_helper``
  option used to do.

These classes have two interface methods:

* ``create_process`` starts a process and returns Popen instance associated
  with it;
* ``execute`` starts a process (using ``create_process`` by default), feeds it
  some input and captures return code along with stdout and stderr output.

Note that ``create_process`` and ``execute`` methods in the
``neutron.agent.linux.utils`` module are still there but they just preprocess
arguments and pass them to appropriate methods of ``root_helper``.

The second step [#cr_rw]_ would be to create those configuration options and
add another root helper class ``RootwrapDaemonHelper``. This class would employ
one rootwrap client for all instances with same rootwrap_config. Its
``execute`` method would just call ``execute`` method of rootwrap client that
would pass request to the daemon.

Note that we can't currently start long-running processes using daemon so
``create_process`` will use usual rootwrap to start the process. Also the gain
for long running processes is negligible.

Data Model Impact
-----------------

None

REST API Impact
---------------

None

Security Impact
---------------

This change will start oslo rootwrap daemons running as root, and available
to neutron via a well authenticated (OTP) unix domain socket.

Therefore ``neutron-rootwrap-daemon`` should be added to the ``sudoers`` file,
with the same settings used for neutron-rootwrap.

Security is handled by oslo-rootwrap which is quite well proven.

All security issues with using client+daemon instead of plain rootwrap are
covered in [#rw_bp]_.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

This change introduces performance boost for agents that rely on calling a huge
amount of commands with rootwrap. Current state of rootwrap daemon shows over
10x speedup comparing to usual ``sudo rootwrap`` call. Total speedup for agents
will be less impressive but should be very noticeable. The rootwrap daemon
implementation includes a benchmark which generated the speedup results.

Other Deployer Impact
---------------------

This change introduces two new config variables:

* ``rootwrap_mode`` using ``daemon`` as default;
* ``rootwrap_config`` must point to ``rootwrap.conf`` deployed for Neutron
  (default ``/etc/neutron/rootwrap.conf``).

Note that by default ``use_rootwrap_daemon`` will be turned off so to get the
speedup one will have to turn it on. With it turned on ``root_helper`` config
option is ignored and ``neutron-rootwrap-daemon`` is used to run most commands
that require root priviledges.

This change also introduces new binary ``neutron-rootwrap-daemon`` that should
be deployed beside ``neutron-rootwrap`` and added to ``sudoers``.

Developer Impact
----------------

None

Community Impact
----------------

The community is broadly interested in faster agent operation.

Alternatives
------------

* Use sudo without rootwrap, at the cost of lower security, or
  by implementing specific sudo plugins to allow equivalent
  equivalent rootwrap filtering. That would require maintaining
  C sudo plugins specific for openstack.

* Rewrite rootwrap into C, but that requires code auditing, and
  also deviates from the standard python of the openstack community.

* Automatically translate rootwrap into C++, and obtain a compiled
  version, this option has been explored, and seems to be possible
  via shedskin with small modifications to rootwrap to avoid dynamic
  typing. By the way, shedskin is experimental, and some python
  modules are not implemented. Also the C++ produced code is not easily
  human readable, and may need a security assesment. This solution
  doesn't eliminate the sudo performance impact, which is not negligible.

Other Solutions
----------------

Other solutions are mostly related to rootwrap optimizations and are
covered in [#rw_bp]_ (some of them are covered in [#eth]_).

This is not the only option of what can be done on the Neutron side besides
switching to rootwrap daemon (see [#eth]_ for details):

* Avoid unnecessary locking in router processing in L3 agent
* Avoid unnecessary calls
* Consolidate system calls: Carl Baldwin explored this option, and proved to
  be very difficult and invasive to the code base.

All of these can be done with rootwrap daemon as well since every call to the
daemon still impose some overhead.

IPv6 Impact
-----------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  * twilson (Terry Wilson, otherwiseguy @ freenode)

Original contributor:
  * yorik-sar (Yuriy Taraday, YorikSar @ freenode)


Work Items
----------

* Abstract out root_helper calls to classes - done [#cr_rh]_;
* Implement rootwrap daemon support - done, needs rebase [#cr_rw]_;

Dependencies
============

None

Testing
=======

Tempest Tests
-------------

This change doesn't change APIs so it doesn't require additional integration
tests. If tempest is happy with ``use_rootwrap_daemon`` turned on, the feature
works.

Functional Tests
----------------

Existing functional tests will exercise the rootwrap client side when performing
agents functional testing. Rootwrap also has it's own functional testing for
the rootwrap client/daemon pieces [#rw_func]_.


API Tests
---------

No API tests are required.

Documentation Impact
====================

User Documentation
------------------

As we are including two new configuration settings, those need to be
properly documented.

Developer Documentation
-----------------------

Developer documentation may be updated to describe how to use the
new interface to execute system commands, if any change were made to the
interface.


References
==========

.. [#rw_bp] oslo.rootwrap blueprint:
   https://blueprints.launchpad.net/oslo.rootwrap/+spec/rootwrap-daemon-mode

.. [#ml] Original mailing list thread:
   http://lists.openstack.org/pipermail/openstack-dev/2014-March/029017.html

.. [#rw_eth] Daemon design overview:
   https://etherpad.openstack.org/p/rootwrap-agent

.. [#cr_rh] Change request "Abstract out root_helper calls to classes":
   https://review.openstack.org/82787

.. [#cr_rw] Change request "Implement rootwrap daemon support":
   https://review.openstack.org/84667

.. [#eth] Original problem statement summarized here:
   https://etherpad.openstack.org/p/neutron-agent-exec-performance

.. [#rw_func] Rootwrap daemon functional testing
   https://github.com/openstack/oslo.rootwrap/blob/master/tests/test_functional.py
