..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================================
OVN Neutron Agent and hardware offloaded QoS extension
======================================================

Launchpad bug: https://bugs.launchpad.net/neutron/+bug/1998608

The aim of the RFE is to define the architecture of a new generic OVN agent
that will execute any needed functionality, being extensible as any other ML2
agent. The first functionality to be implemented will be the hardware
offloaded QoS extension.


Problem Description
===================

ML2/OVN is a mechanism driver that doesn't have backend agents. This mechanism
driver exposes two agent types: the OVN controller agent and the metadata
agent. The OVN controller agent is a representation of the status of the
ovn-controller service on the node; the metadata agent is the service that
provides the metadata to the virtual machines.

However the ML2/OVN mechanism driver does not have an agent to perform the
local Open vSwitch service configuration as in ML2/OVS, with extension drivers
(like QoS or Trunk). The ovn-controller reads the OVN Southbound database to
configure the local OVS database.

The lack of an agent running on the local node, owned by Neutron, prevents us
from implementing some functionalities currently not supported by OVN nor the
drivers. For example, and this will be the first feature that the OVN agent
will implement, hardware offloaded ports cannot apply the OVS QoS rules due to
limitations in the drivers, in particular the ones for the NVIDIA Mellanox
ConnectX-5 network cards [0]_.


Proposed Change
===============

This RFE proposes to create a generic agent running on the compute node. This
agent will be called "OVN Neutron Agent". The execution of this agent is
discretionary and will be needed only if the specific features implemented
on it are requested in a particular compute node. In other words, initially
this service will be needed if a compute node has hardware offloaded ports
and QoS is needed (new features could be implemented in the future).

Unlike other mechanism driver agents (like in ML2/OVS or ML2/SRIOV), this
agent does not implement an RPC client. The information required to work
will be retrieved from the local OVS database, the OVN Northbound database
and the OVN Southbound database. It will be discussed in a future RFE to
include RPC capabilities to have a direct connection to the Neutron
server and the Neutron database.

Extensible functionalities
--------------------------

The new agent functionalities will be defined via extensions. It will use
the ``neutron.agent.agent_extensions_manager.AgentExtensionsManager`` class
to load the configured OVN Neutron Agent extensions. It will use the same
configuration variable "agent.extensions" used by other ML2 agents. The OVN
Neutron Agent extensions will inherit from
``neutron_lib.agent.extensions.AgentExtension`` and will be loaded during the
agent startup by the agent extension manager.

As commented before, the OVN agent will have active OVN Northbound, OVN
Southbound and local OVS database connections, using the corresponding IDL
classes. These classes will monitor a certain set of tables and will
register a set of events. Each extension must report what tables and events
are needed during the startup process. The extensions will inherit from a
specific OVN Neutron Agent extension class (derived from the mentioned
``AgentExtension``) that will provide API methods to retrieve this
information.


OVN agent status report
-----------------------

This new OVN Neutron Agent needs to be tracked in the Neutron server.
There are currently two types of agents: the "OVN Controller agent"/"OVN
Controller Gateway agent" and the "OVN metadata agent". This RFE will
introduce the "OVN Neutron Agent" type. This is a generic name matching
the goal of this spec that is to create a generic OVN agent to implement
all needed functionalities that the OVN controller cannot provide
(metadata will be the next feature to be implemented there, migrated
from the current "OVN metadata agent").

The reporting mechanism will be the same as the one implemented in the
OVN metadata agent. When the Southbound table "SB_Global" is updated,
the agent will update a key in the "external_ids" dictionary. Please check
``neutron.agent.ovn.metadata.agent.SbGlobalUpdateEvent`` for an
implementation example. This variable will be read by the server that will
check the timestamp and determine the agent status.


QoS for hardware offloaded ports
--------------------------------

The first feature (extension) that will be implemented within this agent
is the QoS extension for hardware offloaded ports. Because of the current
drivers limitations, these ports cannot be properly configured with the
current QoS rules.

This RFE proposes to use the corresponding "ip-link" [1]_ commands (or
their "pyroute2" equivalents) to configure the port representor virtual
function rates (maximum and minimum). This is similar to what is done in
ML2/SRIOV for a virtual function (VF).

A port representor (or a devlink port [2]_) is an API to expose the device
information; it is not a physical port but a representative of a hardware
port. The devlink port is connected to the local OVS switch; using the
"devlink" tool (or the "pyroute2" equivalent), it is possible to retrieve
the port information, including the physical function (PF) PCI address.
E.g.:

.. code::

    $ devlink port show enp4s0f1_5 -jp
     "port": {
         "pci/0000:04:00.1/65542": {
             "type": "eth",
             "netdev": "enp4s0f1_5",
             "flavour": "pcivf",
             "pfnum": 1,
             "vfnum": 5,
             "splittable": false,
             "function": {
                 "hw_addr": "fa:16:3e:f8:7b:10"
             }
         }
     }


With the PF PCI address, it is possible to retrieve the PF name, that will
be stored in:

.. code::

    $ cat /sys/bus/pci/devices/$pf_pci_address/net


Using the ML2/SRIOV class ``PciDeviceIPWrapper``, it is possible to set the
maximum and minimum rates of a VF, knowing the VF index and the PF name.


OVN QoS information location
----------------------------

The OVN QoS information is stored in two different places (due to how QoS
is currently implemented in ML2/OVN and core OVN):

* The maximum bandwidth ("rate") and maximum burst ("burst") limits are stored
  in the "Qos:bandwidth" dictionary:

.. code::

    $ ovn-nbctl list Qos
     _uuid               : 376303c4-5290-4c4e-a489-d35894315199
     action              : {}
     bandwidth           : {rate=1000, burst=800}
     direction           : to-lport
     external_ids        : {"neutron:port_id"="89a81cc0-7b3e-473d-8c01-2539cf2a9a6a"}
     match               : "outport == \"89a81cc0-7b3e-473d-8c01-2539cf2a9a6a\""
     priority            : 2002


* The minimum bandwidth rate is stored in the "Logical_Switch_Port:options"
  dictionary:

.. code::

    $ ovn-nbctl list Logical_Switch_Port
     _uuid               : 502b3852-4a46-4fcb-9b49-03063cfc0b34
     addresses           : ["fa:16:3e:c4:b8:29 10.0.0.8"]
     dhcpv4_options      : a7e4cfb1-ef22-490a-9ffe-fea04de3fe1c
     dhcpv6_options      : []
     dynamic_addresses   : []
     enabled             : true
     external_ids        : {...(skipped)}
     ha_chassis_group    : []
     name                : "e6808371-c9ac-4015-94a3-7f16ac3fbb2d"
     options             : {mcast_flood_reports="true", qos_min_rate="1000",
                            requested-chassis=u20ovn}
     parent_name         : []
     port_security       : ["fa:16:3e:c4:b8:29 10.0.0.8"]
     tag                 : []
     tag_request         : []
     type                : ""
     up                  : true



OVN QoS events
--------------

The hardware offloaded QoS extension will configure the QoS setting on a
port reacting to the following events (please check the working POC [3]_
for more information):

* The local OVS interface creation: with this event, the OVN monitor will
  store what ports are bound to the local instance. It will store the
  Neutron port ID (stored in the "Interface.external_ids:iface-id" string)
  and the OVS port name.
* The local OVS interface deletion: this event will trigger the QoS reset
  and the interface local cache deletion.
* The OVN Southbound "Port_Binding" creation event: this event is received
  after a port is created in a local OVS. If this event is received, that
  will trigger the QoS update of the local port. It's worth mentioning that
  this event happens always after the local OVS interface creation; that
  means the OVN monitor has already registered that this port is bound
  locally.
* The OVN Northbound "Logical_Switch_Port" register change: if minimum
  bandwidth of a locally bound LSP changes, this event triggers the QoS
  update.
* The OVN Northbound "QoS" register change: similar to the previous one
  but affecting the maximum bandwidth limit.



REST API Impact
---------------

This RFE does not introduce any API change.


Data Model Impact
-----------------

This RFE does not introduce any model change.


Security Impact
---------------

None.


Performance Impact
------------------

Each monitor will have a connection to the local OVS database and the remotes
OVN Northbound and Southbound databases. The connection to the remote OVN
databases can have a severe impact on the load of the OVN database node (that
are usually the OpenStack controllers). This initial implementation will
subscribe to the following tables:

* Northbound: QoS, Logical_Switch_Port and Logical_Switch
* Southbound: Chassis, Encap, Port_Binding, Datapath_Binding and SB_Global


This performance impact should be reduced by:

* Reducing the number of agents on the node. The next step to be implemented
  (out of scope in this RFE) is to move the OVN metadata agent functionality
  to this new OVN Neutron agent. That will reduce the Southbound database
  connections to one single agent.
* Find a way to store the minimum bandwidth information outside the Logical
  Switch Port. By not subscribing to this table, the Northbound connection
  will reduce the load notably. The Logical Switch Port is one of the most
  populated and active ones; not locally caching nor receiving its updates
  will reduce the impact on the OVN database.


Other Impact
------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignees:
  Rodolfo Alonso Hernandez <ralonsoh@redhat.com> (IRC: ralonsoh)

Work Items
----------

* OVN monitor implementation and the hardware offloaded QoS extension.
* Tests and CI related changes.


Testing
=======

* Unit/functional tests.

.. NOTE::

   The hardware offloaded QoS extension requires specific hardware to
   test this feature. Currently is not possible to implement any
   tempest test on the CI.


Documentation Impact
====================

User Documentation
------------------

* Information about how to configure and spawn the OVN monitor.
* Information about the hardware offloaded QoS extension.


References
==========

.. [0] https://bugzilla.redhat.com/show_bug.cgi?id=2002406
.. [1] https://man7.org/linux/man-pages/man8/ip-link.8.html
.. [2] https://www.kernel.org/doc/html/latest/networking/devlink/devlink-port.html
.. [3] https://review.opendev.org/c/openstack/neutron/+/866480
