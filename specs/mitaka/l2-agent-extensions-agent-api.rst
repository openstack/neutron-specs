..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================================
Allow Open vSwitch agent extensions to own their flows
======================================================

https://bugs.launchpad.net/neutron/+bug/1517903

In Liberty, `L2 agent extensions`_ and `L2 agent extension manager`_ were
introduced. Those extensions are triggered on specific events (particularly, on
port updates and deletes), but they have no way of interacting with the agent's
underlying resources, e.g. bridges, flows, etc. Some features currently in
development (SFC, QoS DSCP, others) require some level of coordination between
an extension and the agent running it.


Problem Description
===================

One major use case where extensions need to coordinate with the agent that is
running them arises from the Open vSwitch agent graceful restart feature added
in Liberty cycle. For this feature, a per-session agent id is used to
distinguish flows that belong to the current agent session from those that
belong to the previous one. The problem starts when an extension wants to
inject its own flows into the switch. Since those extensions don't have a way
to determine what the current agent id is, they cannot mark their new flows
with it, which results in the agent dropping all their flows on restart under
the assumption that they are stale in the sense of "belonging" to the previous
agent session.

Allowing each extension to reserve a unique cookie value known to the agent,
and to use that value for its own flows, will be the first reference
implementation agent-specific API that will be exposed through the new proposed
mechanism. This will be achieved by exposing integration and tunnel bridges
thru a light weight wrapper around Open vSwitch bridges that allocates and
manages cookie values for extensions and hardens them from breaking flows that
belong to other extensions.

Later, the agent-specific API will be extended once the extensions needs are
better understood. One of the features planned for the future is a higher level
flow management API that would abstract out the processing pipeline, allowing
multiple flow-aware extensions to be managed in a more controlled manner
(enforcing processing ordering, better safe guarding other features from
accidental breakages by misbehaving extensions, etc.) Once that higher level
API is implemented, we may consider deprecating the API proposed by this spec.

Note that additional APIs are out of scope for the proposal.


Proposed Change
===============

The proposed change will expand the extensions manager and the L2 agent
extensions APIs with an optional agent-specific API object. The extension
manager will in turn propagate the new API object to each of the extensions.
An extension that needs to interact with the agent running it will be able to
determine the agent type (e.g., Open vSwitch, Linuxbridge, SR-IOV, etc.) that
is running it by inspecting the driver_type argument that is already passed
into its initialize() method.

Since extensions can be maintained outside the Neutron tree, we should consider
not breaking them with later agent API changes. Hence the agent API will be
considered part of public Neutron API and will evolve with due attention to
backwards compatibility concerns.

Since out-of-tree agents could already rely on extensions mechanism, the new
API object argument will be optional in the extensions manager and in
extensions themselves.

Only Open vSwitch will be updated to  pass the agent API object of
corresponding type into L2 agent extensions. Other agents will be extended with
the mechanism as needs arise.

As part of the proposal, the Open vSwitch agent API object will include two new
methods that will return wrapped and hardened bridge objects with cookie values
allocated for calling extensions. Extensions will be able to use those wrapped
bridge objects to set their own flows, while the agent will rely on the
collection of those allocated values when cleaning up stale flows from the
previous agent session::

  +-----------+
  | Agent API +--------------------------------------------------+
  +-----+-----+                                                  |
        |                                   +-----------+        |
        |1                               +--+ Extension +--+     |
        |                                |  +-----------+  |     |
  +---+-+-+---+  2  +--------------+  3  |                 |  4  |
  |   Agent   +-----+ Ext. manager +-----+--+   ....    +--+-----+
  +-----------+     +--------------+     |                 |
                                         |  +-----------+  |
                                         +--+ Extension +--+
                                            +-----------+

Interactions with the agent API object are in the following order::

#1 the agent initializes the agent API object (bridges, other internal state)
#2 the agent passes the agent API object into the extension manager
#3 the manager passes the agent API object into each extension
#4 an extension calls the new agent API object method to receive bridge wrappers with cookies allocated.

Call #4 also registers allocated cookies with the agent bridge objects.

Note: the proposal does not cover major flow table rework suggested in the
corresponding `openstack-dev@ mailing thread`_. Instead, a simpler approach is
chosen for Mitaka, so that we unblock features interested in setting custom
flows, and work on table management part of the mailing list proposal on our
pace.

Implementation
==============

Assignee(s)
-----------

* Ihar Hrachyshka (ihrachys)
* David Shaughnessy (davidsha)

References
==========

.. target-notes::

* _`L2 agent extensions`: https://git.openstack.org/cgit/openstack/neutron/tree/neutron/agent/l2/agent_extension.py?id=b01e3084d135492850384f7732763ad40a1bbc0b
* _`L2 agent extension manager`: https://git.openstack.org/cgit/openstack/neutron/tree/neutron/agent/l2/extensions/manager.py?id=b01e3084d135492850384f7732763ad40a1bbc0b
* _`openstack-dev@ mailing thread` on agent-specific API for L2 agent extensions: http://lists.openstack.org/pipermail/openstack-dev/2015-December/081264.html
