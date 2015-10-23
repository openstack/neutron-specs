..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Neutron PyDev debugger support
==========================================

https://blueprints.launchpad.net/neutron/+spec/pydev-debugger-support


Problem Description
===================

There is currently no proper way for a developer to run a remote PyDev debugger
and have the Neutron process connect back to it.

It is a missed opportunity for a Neutron seamless debugging experience when
using IDEs such as PyCharm or Eclipse PyDev.

Proposed Change
===============

Implementation similar to the one found in the Keystone or Glance project:

Adding the following new Neutron core client options:

- `pydev-debug-host` <host>: default is None
- `pydev-debug-port` <port>: default is None
- `standard-threads` (bool) do not monkey-patch threading system modules
- `no-standard-threads` (bool): default and inverse of `standard-threads`

`standard-threads` option can be enabled either using the Neutron core client
options above or by simply setting an environment variable called
`STANDARD_THREADS`.

The behavior is enabled if `pydev-debug-host` **and** `pydev-debug-port` are
both not None.

If the behavior is enabled, an `os.environ` variable will be set to indicate
that the thread must not get monkey patched:

The Pydev connection will be performed by the server right when starting after
reading the configuration.

Minor modifications are required to deal with the multiple calls to the eventlet
monkey patcher in multiple places throughout the Neutron code: a new module in
common needs to be defined performing the actual patch and checking if whether
or not standard threads are in use to discard thread patch calls.

Calls to eventlet monkey patcher have to be changed to leverage this new module
throughout the Neutron code.

If the Pydev connection fails to be established with the remote debugger an
exception is raised.

Examples:

1. neutron process connecting back to a PyDev remote debugger using standard
threads

   `--pydev-debug-host 1.2.3.4 --pydev-debug-port 12345`

   Using standard threads is the default when enabling Neutron to connect back
   to a Pydev remote debugger.

2. neutron process connection back to a PyDev remote debugger using patched
threads

   `--pydev-debug-host 1.2.3.4 --pydev-debug-port 12345 --no-standard-threads`

In a second step, we should probably consider centralizing this implementation
in oslo and have the various projects allowing PyDev remote debugger connection
use that new implementation from oslo. I would suggest to land this in Neutron
and then work with other projects to centralize though.


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

IPv6 Impact
-----------

None

Performance Impact
------------------

None


Other Deployer Impact
---------------------

None

Developer Impact
----------------

3rd party Neutron code should re-use the common `eventlet_patcher.py` lib to get
seamless PyDev debugger support as well.

Community Impact
----------------

None

Alternatives
------------

An alternative to Pydev remote debugger is using Python standard pdb libray programmatically adding and removing breakpoints in the code. Although working,
it prevents developers to get a seamless debugging experience with their
favorite IDEs.


Implementation
==============

Assignee(s)
-----------

Julien Anguenot <julien@anguenot.org>

Work Items
----------

See `Proposed Change` paragraph as well as the Gerrit code review:
  https://review.openstack.org/#/c/126546/


Dependencies
============

It will be complicated to define a proper PyDev debugger dependencies at the
moment because there is no proper client package to be specified.

IDEs seems to specify their own way to install the client. See references
section below.

Though, the IDE documentations are pretty clear and explicit about how to
install the PyDev debugger dependencies and setup.

If a common client package, working with all IDEs, was to become available at
some point we should definitely revisit this and think of including it in the
requirement.txt.


Testing
=======

Tempest Tests
-------------

Ensure non-regression in non-debug enabled environments.


Functional Tests
----------------

Ensure non-regression in non-debug enabled environments.


API Tests
---------

Ensure non-regression in non-debug enabled environments.


Documentation Impact
====================

User Documentation
------------------

Updating the documentation of the Neutron core client to reflect the new options
 listed above.


Developer Documentation
-----------------------

We could come up with some documentation covering the setup of a remote PyDev
debugger with PyCharm, Eclipse and others

References
==========

Actual implementation and Gerrit code review:
 https://review.openstack.org/#/c/126546/

PyCharm and Python Debug Server:
 https://www.jetbrains.com/pycharm/webhelp/remote-debugging.html

Eclipse PyDev and Remote Debugger:
 http://pydev.org/manual_adv_remote_debugger.html

OpenStack email thread regarding debugging and non-standard threads issues:
 http://lists.openstack.org/pipermail/openstack-dev/2012-August/000794.html
