..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================
Restructure OVS L2 agent
========================

https://blueprints.launchpad.net/neutron/+spec/restructure-l2-agent

:Author: Rossella Sblendido <rsblendido@suse.com>
:Author: Marios Andreou <marios@redhat.com>

This blueprint focuses on paying down the technical debt for the OVS L2 agent,
as already discussed during the Kilo design summit [#]_ .
The goal of this work is to improve the code quality of the OVS l2 agent, in particular
with respect to scalability and performance. These improvements will be evaluated by
performing stress tests using Rally and comparing the results before and after
this change. Test coverage for the OVS L2 agent will also be improved as part of this
blueprint.

.. [#] https://etherpad.openstack.org/p/kilo-neutron-agents-technical-debt

Problem Description
===================

The L2 agent presents several points that can be improved to boost performance and
scalability. This blueprint tackles the following areas: RPC, device processing and the
OVSDB monitor. Every point will be analized in detail in the next
section.
Orthogonal to this blueprint, there's an ongoing effort to use OVS Python lib instead of
the CLI [#]_ .

.. [#] https://blueprints.launchpad.net/neutron/+spec/vsctl-to-ovsdb


Proposed Change
===============

We propose changes for each area identified in the problem description.


RPC
---

* update_device_up and update_device_down are the calls used to notify the plugin that
  a device is up or down. At present, these calls accept only one device, but they can be improved
  to handle several devices.
  The following new RPC calls will be introduced: update_device_list_up and update_device_list_down.
  Instead of making an RPC call per device, the agent would make a single RPC call for all the
  devices. When possible, the neutron plugin will issue a single db update, instead of individual
  updates for each device

* When the agent gets a security_groups_provider_updated it refreshes the filters for all
  the ports, not just for the ports affected by the change. This is unnecessarily costly.
  A new parameter will be added to security_groups_provider_updated to specify the subnet_id,
  so the agent will be able to calculate a list of devices whose filter needs to be updated
  instead of refreshing the filters for all of them.

* modify the port_update message to indicate which changes affected the port. The L2 agent actually
  cares only about changes to the state of the port, the rest of the changes don't require any
  action by the agent.

Smart resync
~~~~~~~~~~~~

With the current implementation if there's an error in the communication with the plugin
during the agent loop, the sync flag is set to true and a complete resync will be
performed by the agent at the next iteration. This means that all the devices will be
processed again by the agent. To avoid that when an error occurs the device being processed
will be put in a list of devices in error. The resync will be performed only for those devices.
The agent's reaction to errors should be improved as well. The agent should analyze the error
and perform one of the following actions:

* Retry the failed operation

* Resync the device involved

* In case of fatal error (e.g. after number of failures of the same operation > threshold), put
  the device in error state

The L2 agent doesn't have a reliable way of ensuring the state reported on the server side is
consistent with the state applied on the backend. Providing a solution to this problem is
out of scope for this blueprint.


Avoid processing a device when there's no need
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* The agent keeps track of the ports for which it received a port update notification. During
  the agent loop updated ports are processed together with added ports. They both are handled in the
  same way but actually in most cases updated ports don't need any action performed by the agent.
  The proposed change is to add a check and process the updated port  only if the update
  is a state change. A state change should be defined. ML2, for instance sends port update
  messages for an awful lot of things. The ones which are important for the l2 agent are:

  - name or IP change
  - MAC address change (when the code will be merged)
  - admin state changes
  - security group changes
  - security group membership changes
  - port security changes (if the corresponding blueprint is accepted and the code is merged)

* When ports are updated, port filters are applied again. This is not needed if there's no change in
  the IP address. Add a check for that.

* When a security_groups_provider_updated is received then a global refresh of the firewall
  is performed.
  This can be avoided if we keep track of the subnet for which the provider rule was updated.
  The agent can refresh the filter of the devices that belong to that subnet.

OVS DB Monitor
~~~~~~~~~~~~~~

The OVS DB monitor knows which devices have been added, and which ones have been removed.
It can therefore be used to generate the events that the agent needs to process, at least the
ones initiated by changes on the host such as vif plug and vif unplug.
Nevertheless, we just use the OVS DB monitor to "signal" that an event occurred and then
scan the bridge again to gather information which was already retrieved by the OVS DB monitor.

Leveraging the OVS DB monitor in this way can also simplify the process of transforming the
agent event processing mechanism from a loop with polling to a queue-based mechanism.
Events can be either initiated on the host itself (e.g.: vif plugged) or from the neutron server
(e.g.: security group membership changed). In many cases these events can be processed independently.
Adding new events to queues will simplify the process of enabling multiple workers for consuming
these events and ensure events with a prerequisite event are executed in the appropriate order.

There is also a possibility of using different queues for handling events with different priorities
according to their criticality. This is however something that can be done in a subsequent iteration
(it won't be anymore debt repayment but 'enhancement').


There's an ongoing effort to modify OVS Python library to make events regarding port added or
deleted available to its client. Even though right now OVS Python lib essentially runs monitor
to update the local cache of the interfaces, it doesn't make those events (device added or removed)
available to the user of the library. Terry Wilson `otherwiseguy <https://launchpad.net/~otherwiseguy>`_ is
working on it.

The L2 agent could make use of this notification system when the change is merged upstream. This
is out of the scope for this blueprint though.

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

Modify port update to specify which change occured to the port

Other End User Impact
---------------------

None

Performance Impact
------------------

Performance should be improved. It's not possible to quantify it now but the following is expected:

* reduced wait time for event processing
* reduced risk of event starvation
* faster OVSDB communication
* reduced load over AMQP bus
* less device processing churn

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

None

Developer Impact
----------------

None

Community Impact
----------------

This change has been discussed during the Kilo design summit and supports the focus
for Kilo to pay down technical debt.


Alternatives
------------

This blueprint in the end is a list of small changes. Every small change can be
discussed and several slightly different variants can be proposed. But the only
general alternatives to this blueprint, are: to leave the agent as it is or to write a
completely new one.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  `rossella_s <https://launchpad.net/~rossella-o>`_

Other contributors:
  `marios <https://launchpad.net/~marios-b>`_
  `salv-orlando <https://launchpad.net/~salvatore-orlando>`_
  `mlavalle <https://launchpad.net/~minsel>`_

Work Items
----------

#. Functional testing of the agent
#. RPC improvements
#. Agent Loop - device processing

   - Avoid processing a device when there's no need

     + Add a check to process updated port
     + Avoid global refresh of the firewall

   - Use event notification from OVS Python library


Dependencies
============

None

Testing
=======


Tempest Tests
-------------

No new tests

Functional Tests
----------------

Functional tests for ip_lib and ovs_lib

Currently there's no functional test for the agent. The following cases will be tested:

* device up
* device down
* port update
* setup tunnel port
* clean up tunnel port
* ovs restart
* ping works between 2 ports on same subnet
* default port filters (check that traffic that is not allowed is blocked and vice versa
  traffic allowed passes)
* security group rule added
* security group rule removed

API Tests
---------

None


Documentation Impact
====================


User Documentation
------------------

None

Developer Documentation
-----------------------

#. New RPC calls will be added update_device_list_up and update_device_list_down
#. A new parameter will be added to security_groups_provider_updated.
#. Port update notification will be modified to specify the change that affected the port

References
==========

https://etherpad.openstack.org/p/kilo-neutron-agents-technical-debt
