..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Dropping rpc API compat layer
=============================

https://blueprints.launchpad.net/neutron/+spec/drop-rpc-compat

This proposal is to make more direct use of oslo.messaging APIs throughout
Neutron instead of a compatibility layer based on an old API.

Problem Description
===================

Neutron has migrated from the rpc code in oslo-incubator to the oslo.messaging
library.  The oslo.messaging library has a different (but better) API.  To ease
the transition to oslo.messaging, Neutron has used some compatibility code to
convert the older style API usage to oslo.messaging APIs.  This proposal is to
make more direct use of oslo.messaging APIs throughout Neutron.

Moving toward more direct usage of oslo.messaging APIs will improve consistency
with other projects, since all projects should be converging on the newer style
APIs.

Proposed Change
===============

The following classes are items considered as a part of the compatibility layer.
The direct use of each of these things will be removed and replaced with direct
use of equivalent and underlying oslo.messaging APIs.

  neutron.common.rpc.
    RpcProxy
    RpcCallback
    RPCException
    RemoteError
    MessagingTimeout

This looks like a small list, but it's more invasive than it looks.  It will be
broken up into several iterative patches that are more easily reviewed and
merged over time.

Here are some examples of the most invasive conversions (RpcProxy and
RpcCallback)::

    from oslo import messaging

    from neutron.common import rpc as n_rpc


Old style client interface using RpcProxy::

    class ClientAPI(n_rpc.RpcProxy):

        BASE_RPC_API_VERSION = '1.0'

        def __init__(self, topic):
            super(DhcpPluginApi, self).__init__(
                topic=topic, default_version=self.BASE_RPC_API_VERSION)

        def remote_method(self, context, arg1):
            return self.call(context,
                             self.make_msg('remote_method',
                                           arg1=arg1))

        def remote_method_2(self, context, arg1, arg2):
            return self.call(context,
                             self.make_msg('remote_method_2',
                                           arg1=arg1, arg2=arg2),
                             version='1.1')

New style client interface::

    class ClientAPI(object):

        def __init__(self, topic):
            target = messaging.Target(topic=topic, version='1.0')
            self.client = n_rpc.get_client(target)

        def remote_method(self, context, arg1):
            cctxt = self.client.prepare()
            return cctxt.call(context, 'remote_method', arg1=arg1)

        def remote_method_2(self, context, arg1, arg2):
            cctxt = self.client.prepare(version='1.1')
            return cctxt.call(context, 'remote_method_2', arg1=arg1, arg2=arg2)


Old style server interface::

    class ServerAPI(n_rpc.RpcCallback):

        RPC_API_VERSION = '1.1'

        def remote_method(self, context, arg1):
            return 'foo'

        def remote_method_2(self, context, arg1, arg2):
            return 'bar'


New style server interface::

    class ServerAPI(object):

        target = messaging.Target(version='1.1')

        def remote_method(self, context, arg1):
            return 'foo'

        def remote_method_2(self, context, arg1, arg2):
            return 'bar'


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

Negligible.   In theory, this is removing a compatibility layer so there should
be less code, but the overall performance impact of removing the layer is
negligible in the broader scope of things.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

There are several impacts to developers.  Developers used to the older rpc API
(and the equivalent compatibility layers) will have to learn what's different.
Arguably, it's conceptually quite similar so this shouldn't have a large impact.
The oslo.messaging library itself has documentation.  Looking at all of the
existing code once it has been converted will help quite a bit as well, since
you can just follow the existing pattern.

The other impact is patch conflicts.  Since this work involves some minor
refactoring across the tree, it may conflict with other code in progress.  This
will only occur in the case of features that modify rpc APIs.  The changes will
be done in a series of smaller changes, which will help with rebasing and fixing
conflicts against smaller sets of updates at a time.

Community Impact
----------------

None.

Alternatives
------------

The alternative is to leave the compatibility layer in place.  The downsides of
this are that Neutron's use of oslo.messaging will diverge further and further
from what the rest of OpenStack messaging usage looks like.  It may also make it
more difficult to take advantage of new features in oslo.messaging in the
future.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  russellb

Work Items
----------

The usage of each class listed in the proposed change section will be removed
one at a time.  The removal of each will be broken up into several changes, one
per interface (rpc API).


Dependencies
============

* None


Testing
=======

The unit tests will be updated as required.  New ones will be added in key
places where unit test coverage may be missing.

There are no functional changes here, so all of the existing unit and functional
tests will be relied upon to help ensure that no regressions are introduced.

Tempest Tests
-------------

None.

Functional Tests
----------------

None.

API Tests
---------

None.


Documentation Impact
====================

None.

User Documentation
------------------

None.

Developer Documentation
-----------------------

None.

References
==========

* Existing oslo.messaging documentation:
  http://docs.openstack.org/developer/oslo.messaging/
