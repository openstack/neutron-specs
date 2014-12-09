..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Use OVSDB instead of calling ovs-vsctl
======================================


Problem Description
===================
Neutron's ovs_lib uses the Open vSwitch CLI command *ovs-vsctl* to perform
basic vSwitch CRUD operations. This is almost an order of magnitude slower
than using either direct OVSDB protocol commands, or using the Open vSwitch
Python API.

Proposed Change
===============
First, an interface for interacting with with the OVSDB should be defined as
an abstract base class. Then, an implementation of this interface using
ovs-vsctl calls should be made and ovs_lib should make all ovsdb-related calls
through this new implementation.

Next, an implementation of the OVSDB interface should be implemented using
the Open vSwitch IDL Python library. A configuration option will be added
to allow selection of the OVSDB interface to use. It is necessary to allow
this choice as there are differences in how privileges will be handled between
the two OVSDB implementations. The ovs-vsctl implementation uses sudo/rootwrap
whereas the IDL library will require either giving the neutron user permissions
to the ovsdb unix socket, or using TCP/SSL sockets and controlling access via
firewall rules (though likely just using the loopback interface).

Any distributions or deployment targets that do not support the requirements
for the proposed OVSDB implementation will still be able to use the existing
implementation.

This will allow leaving the ovs_lib API largely unchanged. On a simple test
creating and deleting 100 ports on an existing bridge, the current ovs_lib
implementation was nearly 10x slower than using the OVS Python API. It should
be noted that ovs_lib was over 100x slower than sending raw OVSDB commands to
Open vSwitch. The performance disparity between the raw OVSDB and existing
OVS Python API is due to the poor performance of the OVS Python API's own
pure-Python JSON parser which tests show to be ~30x slower than Python's stdlib
JSON parser which, unfortunately, is not a good fit for OVS's use case as it
requires having the entire string ready to parse, whereas the OVS code is
structured to parse buffers as they are read. Potentially significant speedups
are possible in OVS's JSON parser by writing an extension that adds Python
bindings to the C version of their parser.

The OVS IDL implementation makes use of monitor commands for syncing a local
cache of the OVSDB. Unfortunately, it does not make these monitor events
available to users of the library. To replace ovsdb-monitor, it will be
necessary to either use a lower-level Open vSwitch API running an additional
monitor request to get these event notifications, to modify the IDL library at
runtime, or to try to get code merged to the upstream Open vSwitch library
that optionally exposes these events.

Data Model Impact
-----------------
n/a

REST API Impact
---------------
n/a

Security Impact
---------------
The existing implementation handles privileges via sudo/rootwrap. The proposed
change would be using a programatic API and would instead rely on the
appropriate permissions being set on underlying Open vSwitch unix domain socket
or through the use of Open vSwitch's SSL authentication.

Notifications Impact
--------------------
n/a

Other End User Impact
---------------------
n/a

Performance Impact
------------------
With no changes to the upstream OVS Python API, a 6-7x speed improvement is
possible. With Python bindings to the OVS C JSON parser, it should be possible
to approach the native OVSDB protocol performance which was 100x faster than
the current ovs_lib.

IPv6 Impact
-----------
n/a

Other Deployer Impact
---------------------
Repeating from the Security section, deployers would have to ensure that the
*neutron* user has r/w permissions to the Open vSwitch db.sock. A new config
option specifying the connection string for ovs-server would also be required.
Packagers will need to add a dependency for python-openvswitch (which seems to
be readily available across common distributions as it is part of standard
openvswitch packaging).

Developer Impact
----------------
For the initial implementation, there shouldn't be much change to ovs_lib's
API.

It should be noted that users of XenAPI currently use a modified version of
rootwrap to execute ovs-vsctl commands. To use the proposed OVSDB
implementation with the XenAPI, an equivalent wrapper may have to be
developed. This work is outside of the scope of this blueprint.

Community Impact
----------------
n/a

Alternatives
------------
It would be possible to write our own library around the OVSDB protocol. It
would most likely be faster, but it would most likely end up looking very much
like the OVS Python API by the time we were finished.


Implementation
==============

Assignee(s)
-----------
otherwiseguy

Work Items
----------
* Convert existing OVS unit tests that deal entirely with external Open vSwitch
  actions to functional tests.

* Convert existing ovs-vsctl calls to the equivalent OVS Python API calls

* Re-work OVSDB Monitoring to use the OVS Python API

* Write Python bindings for OVS's C-based JSON push parser to increase
  performance

Dependencies
============
Adds a requirement for the OVS python bindings

Testing
=======

Many of the unit tests for ovs_lib will have to be changed because they are
tied far too heavily to specific implementation details, mocking out calls
to ovs-vsctl, etc.

Tempest Tests
-------------
n/a

Functional Tests
----------------
Functional tests that actually test the CRUD operations against a
real Open vSwitch installation should be created. They should work against
both OVSDB interface implementations.

API Tests
---------
n/a

Documentation Impact
====================

User Documentation
------------------
Documentation of the new config option and security considerations will be
necessary.


Developer Documentation
-----------------------
The OVSDB abstract base class should be well-documented. In-tree developer
docs describing the ovs_lib implementation will be added.


References
==========
http://tools.ietf.org/html/rfc7047
https://github.com/openvswitch/ovs/tree/master/python/ovs
