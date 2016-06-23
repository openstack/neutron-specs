..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
L3 Agent Extension Framework
=============================

RFE: https://bugs.launchpad.net/neutron/+bug/1580239

Problem Description
===================

Neutron advanced services (\*aaS) projects sometimes need access to resources
internal to the L3 agent.  For example, FWaaS needs:

* The ability to map router_id to router info so we can program iptables to the
  correct namespace.
* The ability to load the Service Agent - so we have an RPC endpoint in the
  context of L3Agent.

Currently, the accepted methodology is inheritance: the L3NATAgent inherits
from the advanced service. Only those extensions whose classes are inherited
from L3NATAgent have access to agent resources like router and namespace data.
This has several drawbacks:

* The introduction of a new advanced service requires a patch to the agent code.
* Similarly, it is not possible to implement vendor-specific agent extensions
  without changing the agent code.
* Multiple agents cannot be run simultaneously.
* It isn't possible to arbitrarily enable an advanced services, as it is
  hard-coded.

Proposed Change
===============

To address these problems, we will decouple these services from the agent, so
that each \*aaS is a separate extension that registers with the extension
manager and provides the extension manager a list of RPC messages it wishes to
consume, and handler functions to process those RPC messages.  The extension
manager will register each RPC message type with neutron's callback registry as
a consumer and will use the existing RPC callback producer pattern to listen
for notifications about events of interest. When an RPC callback occurs, each
extension that registered to handle that type of RPC message will have its
handler function invoked.  Multiple \*aaS services will be able to plug in
simultaneously without interfering with each other.  Vendor-specific
extensions can be written as agent extension drivers.

This proposal will create the following new objects in the L3 agent:

* L3AgentExtension - This object will define a stable abstract interface for
  agent extensions.
* L3AgentExtensionManager - This object will be a
  stevedore.named.NamedExtensionManager, which is how extensions register and
  interface into the agent.  There will be only one L3AgentExtensionManager
  instantated at runtime; this is a limitation inherent to stevedore's
  NamedExtensionManager, but making the extension manager generalizable to
  handle multiple situations is good practice.

This mechanism will be similar to the extension system implemented for the L2
agent in http://git.openstack.org/cgit/openstack/neutron/tree/neutron/agent/l2/extensions/manager.py?id=4821196f94d333cb4c310601776f9b2319a6ddf0
.  To the maximum extent possible, generalizable code will be moved to a common
location so that both the L2 and L3 agent can re-use it.

Comparable Implementation
-------------------------

This implementation is patterned upon the implementation for the QoS agent
extension in the L2 agent and the accompanying L2 notification driver on the
neutron server. This section describes that interaction with pointers to code
(stable/mitaka branch).  Examples of agent specifics are provided in the
context of the neutron-openvswitch-agent.

The implementation in neutron-server flows thusly:

* The QoSPlugin class initializes an instance of the
  QosServiceNotificationDriverManager class and passes to it whenever a
  QoSPolicy object needs to be created, updated, or deleted.
  http://git.openstack.org/cgit/openstack/neutron/tree/neutron/services/qos/qos_plugin.py?id=4821196f94d333cb4c310601776f9b2319a6ddf0#n37

* The QosServiceNotificationDriverManager class loads all instances of
  configured notification drivers, and then calls appropriate methods in the
  RpcQosServiceNotificationDriver class to send update or delete events.
  http://git.openstack.org/cgit/openstack/neutron/tree/neutron/services/qos/notification_drivers/manager.py?id=4821196f94d333cb4c310601776f9b2319a6ddf0#n30

* When a QosPolicy update or delete event is received by the
  RpcQosServiceNotificationDriver, the driver then submits the QosPolicy object
  in question to the RPC callbacks registry
  (neutron.api.rpc.callbacks.producer.registry).
  http://git.openstack.org/cgit/openstack/neutron/tree/neutron/services/qos/notification_drivers/message_queue.py?id=4821196f94d333cb4c310601776f9b2319a6ddf0#n40

The neutron-openvswitch-agent implements the flows thusly:

* The AgentExtensionsManager class provides a method for agent extensions to be
  initialized.
  http://git.openstack.org/cgit/openstack/neutron/tree/neutron/agent/l2/extensions/manager.py?id=4821196f94d333cb4c310601776f9b2319a6ddf0#n37

* The AgentsExtensionManager class is instantiated inside the
  init_extension_manager function within the OVSNeutronAgent class, which is the
  primary class for the neutron-openvswitch-agent.
  http://git.openstack.org/cgit/openstack/neutron/tree/neutron/plugins/ml2/drivers/openvswitch/agent/ovs_neutron_agent.py?id=4821196f94d333cb4c310601776f9b2319a6ddf0#n397

* The QosAgentExtension subscribes--by submitting a callback method to the RPC
  registry--as an interested party to QoS policy events. This is the mechanism
  by which the agent receives notifications of events of interest.
  http://git.openstack.org/cgit/openstack/neutron/tree/neutron/agent/l2/extensions/qos.py?id=4821196f94d333cb4c310601776f9b2319a6ddf0#n203

* The QosAgentExtension class handles incoming QoSPolicy objects.
  http://git.openstack.org/cgit/openstack/neutron/tree/neutron/agent/l2/extensions/qos.py?id=4821196f94d333cb4c310601776f9b2319a6ddf0#n188

* Inside the neutron-openvswitch-agent the QosOVSAgentDriver class implements
  QoS actions against the resources available to the agent.
  http://git.openstack.org/cgit/openstack/neutron/tree/neutron/plugins/ml2/drivers/openvswitch/agent/extension_drivers/qos_driver.py?id=4821196f94d333cb4c310601776f9b2319a6ddf0#n26

* The QosOVSAgentDriver and all other agent extension drivers are children of
  the QosAgentDriver metaclass, which defines the stable abstract interface for
  the QoS Agent Driver.
  http://git.openstack.org/cgit/openstack/neutron/tree/neutron/agent/l2/extensions/qos.py?id=4821196f94d333cb4c310601776f9b2319a6ddf0#n37

* To notify the the driver of incoming events, QosAgentExtension loads up the
  QosAgentDriver
  http://git.openstack.org/cgit/openstack/neutron/tree/neutron/agent/l2/extensions/qos.py?id=4821196f94d333cb4c310601776f9b2319a6ddf0#n196
  and then calls it explicitly
  http://git.openstack.org/cgit/openstack/neutron/tree/neutron/agent/l2/extensions/qos.py?id=4821196f94d333cb4c310601776f9b2319a6ddf0#n255
  http://git.openstack.org/cgit/openstack/neutron/tree/neutron/agent/l2/extensions/qos.py?id=4821196f94d333cb4c310601776f9b2319a6ddf0#n275
  http://git.openstack.org/cgit/openstack/neutron/tree/neutron/agent/l2/extensions/qos.py?id=4821196f94d333cb4c310601776f9b2319a6ddf0#n282

The idea to simply use and adapt the existing L2 implementation to handle L3
communications also was considered and rejected.  The QoS notification
mechanism needs to remain specific to port and network updates in order not to
crush the message queue.

Data Model Impact
-----------------

None

Command Line Client Impact
--------------------------

None

Security Impact
---------------

None

Notifications Impact
--------------------

The notifications proposed in this spec will override certain existing
notifications but should not dramatically increase the number of notifications.

Performance Impact
------------------

None

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

Server- and agent-side configuration changes must be made.  For instance, for
FWaaS:

* On the server side, in neutron.conf, 'firewall' needs to be added to the
  service_plugins list in neutron.conf, as before. Also in neutron.conf, the
  needed notification_drivers in the [fwaas] section must be specified
  (message_queue is the default).

* On the L3 agent side, 'firewall' must be added to 'extension_drivers' in
  l3_agent.ini.

* L3 agent variants (i.e. neutron-vpn-agent) could be an alias of the base l3
  agent but with a different set of default extension drivers.

Developer Impact
----------------

This change introduces a standardized interface for developing advanced
services on top of the L3 agent, and thus eases adoption of new L3 advanced
services and facilitates developer experimentation.

Community Impact
----------------

This change expands the stability and standardization of extension hooks into
neutron, making the platform even more friendly to new technologies and
vendors.

Implementation
==============

Assignee(s)
-----------

* skandasw
* german-eichberger
* nate-johnston
* emspiege
* margaret-frances
* y-furukawa-2
* victor-r-howard

Work Items
----------

* Generalize the extension management framework so that as much as possible
  is common between the L2 and L3 agent.
* Implement new extension framework in L3 agent leveraging the code made common
  in the previous step.
* Implement notification driver in Neutron server.
* Add unit and functional tests for L3 agent extension framework.
* Add unit and functional tests for notification driver in Neutron server.
* Implement proper devstack configuration to enable testing of extensions.
* Implement whatever gate changes are required to adopt the new extension
  mechanisms.
* Implement an extension that uses this extension management capability
  (FWaaS).
* Change the QoS agent extension in the L2 agent (ML2/OVS and ML2/LB) to also
  use the common extension management code.

Dependencies
============

API-tests
---------

None.

Functional Tests
----------------

Functional tests would need to verify that the L3AgentExtensionManager is
loading the L3AgentExtension, as well as testing the communications of the
notification driver.

Fullstack Tests
---------------

Fullstack tests would need to exist to enable loading of a dummy extension and
then ensuring that the dummy extension receives RPC calls properly when issued
by Neutron server.

Documentation Impact
====================

User Documentation
------------------

Existing user documentation describing services that make use of the L3 agent
facility, for example the `_Admin Guide's "Introduction to Networking" section
<http://docs.openstack.org/admin-guide/networking_introduction.html>_` would
need to be updated.

Developer Documentation
-----------------------

New developer documentation describing the L3 agent extension mechanism would
need to be created.

API Documentation
-----------------

Documentation of the method to discover what extensions are active would need
to be created.

References
==========

* Prior art: https://review.openstack.org/#/c/91532/11/specs/juno/l3-agent-consolidation.rst
