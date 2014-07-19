..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Use OVSDB instead of calling ovs-vsctl
======================================


Problem description
===================
Neutron's ovs_lib uses the Open vSwitch CLI command *ovs-vsctl* to perform
basic vSwitch CRUD operations. This is almost an order of magnitude slower
than using either direct OVSDB protocol commands, or using the Open vSwitch
Python API.

In addition, using the CLI commands make it easy to misunderstand the
asynchronous nature of some Open vSwitch operations. For example:
ovs_lib.OVSBridge.add_port() runs *ovs-vsctl add-port* then returns
the value returned by OVSBridge.get_port_ofport, which calls *ovs-vsctl
get Interface ${ifname} ofport*. Open vSwitch does not assign the ofport
inside a transaction, leading to a race condition where ofport may or may not
be set, even though the port creation was successful.

Using the proper APIs would make problems like the above easier to realize. In
addition, relying on the parsing of CLI output which is intended to be human
readable is both non-performant and brittle.

Proposed change
===============
Leaving the actual ovs_lib API largely unchanged, we should replace calls to
ovs-vsctl with the upstream Open vSwitch Python API calls. On a simple test
creating and deleting 100 ports on an existing bridge, the current ovs_lib
implementation was 6-7x slower than using the OVS Python API. It should be
noted that ovs_lib was over 100x slower than sending raw OVSDB commands to
Open vSwitch. The performance disparity between the raw OVSDB and existing
OVS Python API is due to the poor performance of the OVS Python API's own
pure-Python JSON parser which tests show to be ~30x slower than Python's stdlib
JSON parser which, unfortuanately, is not a good fit for OVS's use case as it
is a pull-parser. Significant potential speedups are possible in OVS's JSON
parser by writing an extension that adds Python bindings to the C version of
their parser.

The OVS Python API's IDL implementation may need to be subclassed to provide
a hook for being able to take the update notifications it receives so that
we can use those to automatically update the OVS Plugin.

Alternatives
------------
It would be possible to write our own library around the OVSDB protocol. It
would most likely be faster, but it would most likely end up looking very much
like the OVS Python API by the time we were finished.

Data model impact
-----------------
n/a

REST API impact
---------------
n/a

Security impact
---------------
The existing implementation handles privileges via sudo/rootwrap. The proposed
change would be using a programatic API and would instead rely on the
appropriate permissions being set on underlying Open vSwitch unix domain socket
or through the use of Open vSwitch's SSL authentication.

Notifications impact
--------------------
n/a

Other end user impact
---------------------
n/a

Performance Impact
------------------
With no changes to the upstream OVS Python API, a 6-7x speed improvement is
possible. With Python bindings to the OVS C JSON parser, it should be possible
to approach the native OVSDB protocol performance which was 100x faster than
the current ovs_lib.

Other deployer impact
---------------------
Repeating from the Security section, deployers would have to ensure that the
*neutron* user has r/w permissions to the Open vSwitch db.sock. A new config
option specifying the connection string for ovs-server would also be required.
Packagers will need to add a dependency for python-openvswitch (which seems to
be readily available across common distributions as it is part of standard
openvswitch packaging).

Developer impact
----------------
For the initial implementation, there shouldn't be any change to ovs_lib's
public-facing API, though many of the unit tests will have to be changed
because they are tied far too heavily to specific implementation details,
mocking out calls to ovs-vsctl, etc.

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
Existing tempest tests should be useful, but most of the ovs_lib unit tests
will need to be removed as they don't test anything but the implementation
details. Functional tests that actually test the CRUD operations against a
real Open vSwitch installation should be created.

Documentation Impact
====================
Documentation of the new config option and security considerations will be
necessary.

References
==========
http://tools.ietf.org/html/draft-pfaff-ovsdb-proto-04
https://github.com/openvswitch/ovs/tree/master/python/ovs
