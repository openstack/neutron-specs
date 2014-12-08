..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
RPC Docs and Namespaces
=======================

https://blueprints.launchpad.net/neutron/+spec/rpc-docs-and-namespaces

This blueprint aims to enhance the existing usage of oslo.messaging to make the
version scheme more clear to developers.  Several people have expressed that the
current code is difficult to understand, so this spec aims to improve that.

Problem Description
===================

Neutron uses the oslo.messaging library to provide an internal communication
channel between Neutron services.  This communication is typically done via
AMQP, but those details are mostly hidden by the use of oslo.messaging and it
could be some other protocol in the future.

RPC APIs are defined in Neutron in two parts: client side and server side.

Client Side
-----------

Here is an example of an rpc client definition:

::

  from oslo import messaging

  from neutron.common import rpc as n_rpc


  class ClientAPI(object):
      """Client side RPC interface definition.

      API version history:
          1.0 - Initial version
          1.1 - Added my_remote_method_2
      """

      def __init__(self, topic):
          target = messaging.Target(topic=topic, version='1.0')
          self.client = n_rpc.get_client(target)

      def my_remote_method(self, context, arg1, arg2):
          cctxt = self.client.prepare()
          return cctxt.call(context, 'my_remote_method', arg1=arg1, arg2=arg2)

      def my_remote_method_2(self, context, arg1):
          cctxt = self.client.prepare(version='1.1')
          return cctxt.call(context, 'my_remote_method_2', arg1=arg1)


This class defines the client side interface for an rpc API.  The interface has
2 methods.  The first method existed in version 1.0 of the interface.  The
second method was added in version 1.1.  When the newer method is called, it
specifies that the remote side must implement at least version 1.1 to handle
this request.

Server Side
-----------

The server side of an rpc interface looks like this:

::

  from oslo import messaging


  class ServerAPI(object):

      target = messaging.Target(version='1.1')

      def my_remote_method(self, context, arg1, arg2):
          return 'foo'

      def my_remote_method_2(self, context, arg1):
          return 'bar'


This class implements the server side of the interface.  The messaging.Target()
defined says that this class currently implements version 1.1 of the interface.

Versioning
----------

Note that changes to rpc interfaces must always be done in a backwards
compatible way.  The server side should always be able to handle older clients
(within the same major version series, such as 1.X).

It is possible to bump the major version number and drop some code only needed
for backwards compatibility.  For more information about how to do that, see
https://wiki.openstack.org/wiki/RpcMajorVersionUpdates.

Each stream of version numbers is associated with a target.  On the client side,
you set up a target that identifies the destination of your method call.  Every
method exposed at that target is a single API from the oslo.messaging
perspective, regardless of the implementation details on the other side.

Unfortunately, in the Neutron code today, it's inconsistent and generally
difficult to tell what is considered a part of the same version stream or is
separate.

A demonstrative example: DHCP Agent
-----------------------------------

There is an RPC interface defined that allows the Neutron plugin to
remotely invoke methods in the DHCP agent.  The client side is defined in
neutron.api.rpc.agentnotifiers.dhcp_rpc_agent_api.DhcpAgentNotifyApi.  The
server side of this interface that runs in the DHCP agent is
neutron.agent.dhcp_agent.DhcpAgent.

The DHCP agent includes a client API, neutron.agent.dhcp_agent.DhcpPluginAPI.
The DHCP agent uses this class to call remote methods back in the Neutron
server.  The server side is defined in
neutron.api.rpc.handlers.dhcp_rpc.DhcpRpcCallback.  This is where versioning
starts to get complicated.

A version number is associated with an oslo.messaging.Target.  The target of a
message is specified using a topic (and optionally a specific host on that
topic).  Everything exposed via the same target must be versioned as a part of
the same version stream.  The DhcpRpcCallback API is exposed on the 'q-plugin'
topic.  Several other things are exposed on the same topic.  For example, when
using the ml2 plugin, the relevant code is:

::

        self.endpoints = [rpc.RpcCallbacks(self.notifier, self.type_manager),
                          securitygroups_rpc.SecurityGroupServerRpcCallback(),
                          dvr_rpc.DVRServerRpcCallback(),
                          dhcp_rpc.DhcpRpcCallback(),
                          agents_db.AgentExtRpcCallback(),
                          metadata_rpc.MetadataRpcCallback()]

With the way the code is written today, the API exposed via the 'q-plugin' topic
is the *union* of the APIs defined by the following classes:

::

    neutron.plugins.ml2.rpc.RpcCallbacks
        neutron.plugins.ml2.drivers.type_tunnel.TunnelRpcCallbackMixin
    neutron.api.rpc.handlers.securitygroups_rpc.SecurityGroupServerRpcCallback
    neutron.api.rpc.handlers.dvr_rpc.DVRServerRpcCallback
    neutron.api.rpc.handlers.dhcp_rpc.DhcpRpcCallback
    neutron.db.agents_db.AgentExtRpcCallback
    neutron.api.rpc.handlers.metadata_rpc.MetadataRpcCallback

That means that an API change to *ANY* of these APIs means an increase in a
version number stream that spans all of these interfaces.  However, the
implementation seems to suggest that they are versioned independently, but it's
not consistent.

Proposed Change
===============

This spec proposes two changes.  The goal of the changes is to help make it more
clear what the scope of the version numbers are.  It should make handling of
these version numbers a bit easier to understand and hopefully less error prone
as a result.

1) Documentation

For every client and server rpc interface definition in Neutron, add
documentation that makes it very clear where the corresponding other side of the
interface lives in the code.  Also, the documentation will list any other
classes that are included in the same version stream.

2) Use namespaces

When defining an oslo.messaging.Target, you can add a namespace.  This allows
you to expose multiple interfaces on the same topic but still version them
independently.

If we revisit the example of the 'q-plugin' topic and the ml2 driver, we have
several interfaces exposed on the same topic in the default namespace.
Namespaces will be used to separate these interfaces so that each interface can
be versioned independently.

For example, one of the interfaces exposed is
neutron.api.rpc.handlers.dhcp_rpc.DhcpRpcCallback.  The beginning of that
implementation looks like this:

::

    class DhcpRpcCallback(object):
        """DHCP agent RPC callback in plugin implementations."""

        target = messaging.Target(version='1.1')

Instead, the target will now be defined with a namespace:

::

    class DhcpRpcCallback(object):
        """DHCP agent RPC callback in plugin implementations."""

        target = messaging.Target(namespace='dhcp', version='1.1')

Similarly, the client side must be updated as well.
neutron.agent.dhcp_agent.DhcpPluginAPI.__init__ includes the following code:

::

        target = messaging.Target(topic=topic, version='1.0')
        self.client = n_rpc.get_client(target)

The change here is quite similar to what's done on the server side:

::

        target = messaging.Target(namespace='dhcp', topic=topic, version='1.0')
        self.client = n_rpc.get_client(target)

Now, when the client makes a request it is being much more explicit.  The server
side will only look for the requested method on the endpoint marked with the
'dhcp' namespce, instead of all included endpoints.

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

None

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

Technically, this change is not backwards compatible.  The reality is that only
matters if you expect any sort of live upgrade to work.  Since live upgrades
aren't actually expected to work, this spec proposes just accepting this
theoretical incompatibility since it doesn't matter in practice, yet.

Developer Impact
----------------

The developer impact should be minimal.  Hopefully the impact is just that when
developers come across the need to change one of these interfaces, it's a bit
more clear what other parts of the code are impacted.  Developers will also have
to be aware of the new namespace usage for any new interfaces added in the
future.

Community Impact
----------------

The only things proposed here are documentation and use of a standard
oslo.messaging APIs.

Alternatives
------------

Instead of using namespaces, you could expose each interface on its own topic.
Namespaces are a bit more convenient, as a service tends to have one topic set
up using common code.  That common code would have to be reworked to support an
arbitrary list of endpoints mapped to topics.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  russellb

Work Items
----------

* Convert code base to use namespaces as appropriate.
* Add doc strings to all rpc client and server interfaces.

Dependencies
============

* drop-rpc-compat blueprint:
  https://blueprints.launchpad.net/neutron/+spec/drop-rpc-compat

Testing
=======

Existing testing will validate that these changes have not caused regressions.

Tempest Tests
-------------

None

Functional Tests
----------------

None

API Tests
---------

None

Documentation Impact
====================

None

User Documentation
------------------

None

Developer Documentation
-----------------------

Additional developer documentation is a key deliverable of this blueprint.

References
==========

* oslo.messaging documentation:
  http://docs.openstack.org/developer/oslo.messaging/
