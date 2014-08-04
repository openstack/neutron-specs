..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================
Configuration Agent for Cisco Service VMs
=========================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/cisco-config-agent

This blueprint covers the Cisco configuration agent that is responsible for
configuring Cisco service VMs to implement network services. This agent is
meant to be generic and intended to support multiple services. But initially it
will cover only routing and NAT services in CSR1kv Service VM. The design of
this agent is inspired by the current L3 agent which configures neutron routers
in Linux namespaces.

This blueprint depends on the blueprint:
https://blueprints.launchpad.net/neutron/+spec/cisco-routing-service-vm
which implements the plugin side.


Problem description
===================

There are use cases for Neutron services implemented using Service VMs.
CSR1kv is one such service VM that can implement a variety of network
services like routing, NAT, firewalls, VPNs, etc.

The configuration agent proposed here will connect to and configure a CSR1kv VM
for implementing Neutron routers. Neutron routers are configured inside the
CSR1kv VM as VRFs. The agent will also configure NAT and floatingIP
functionality for external gateways.


Use Cases:

 1. Implement and configure Neutron routers via service VMs.



Proposed change
===============

This change will introduce a Cisco configuration agent (here after called the
config agent) similar to the current L3 agent, but for service VMs.

The diagram below provides a high level overview of how the config agent
fits in with other components and its north and sound bound interfaces.

::

  +––––––––––––––––––––+
  |cisco routing       |
  |service plugin      |
  +–––––––––+––––––––––+
            |
            |
  +–––––––––+––––––––––+
  |cisco configuration |
  |agent               |
  +–––––––––+––––––––––+
            |
  +–––––––––+––––––––––+
  |service VM          |
  |(CSR1kv)            |
  +––––––––––––––––––––+

Note that unlike the l3 agent, the config agent can manage and configure
multiple devices. There can be one or more config agents in the system.
The plugin will divide the total number of devices (to manage) among these
agents. But at any given time, a service VM is configured by one and only one
config agent. This assignment is done by the plugin.

The config agent can be run in any node, even including the controller/network
node as long as it can reach the devices.

Failure handling
----------------

Config agents implement the agent state reporting framework and send status
reports periodically. This mechanism is also used to detect a failed agent.
In case of a failed agent, the service VM and the services configured in it is
transferred to a different config agent.

Assumptions
-----------

The service VMs are owned and managed by the cloud admin. They are not visible
to the tenants. They are connected to a management network which is mapped to a
provider network in neutron. This management network can be seperate from the
openstack management network. Also the VM which provides the network service is
considered as a trusted VM which the admin can safely connect to his management
network.

Design
------

The config agent has a modular design with a number of components.
They are:

  * Plugin RPC: Handle the communication towards the plugin side.
  * Device driver: Configuration logic for individual device(s). Also
    encapsulates the configuration protocol. e.g: Netconf or REST.
  * Service helper: Encapsulates the logic for any bookkeeping operation or
    intermediate conversions, before calling the driver.
  * Health monitoring: Monitor the state of the device(s) being configured and
    report unreachable device(s) back to plugin.


Components:

::

 +––––––––––––––––––––––––––––––––––––––––––––––––––––––––––––+
 |                                                            |
 |                     +————————+                             |
 |                    +—––––––+ |                             |
 |                    |Plugin | |                             |
 |                    |RPC    +—+                             |
 |                    +–––––––+                               |
 |    +––––––––––+                     +––––––––––––––––+     |
 |    |Health    |                     |(routing)       |     |
 |    |Monitoring|    Config agent     |service helper  |-+   |
 |    +––––––––––+                     +-–––––––––––––––+ |   |
 |                                      +_________________+   |
 |                                                            |
 |                   +–––––––+                                |
 |                   |Device |                                |
 |                   |Driver |—+                              |
 |                   +——–––––+ |                              |
 |                    +––––––——+                              |
 |                                                            |
 +––––––––––––––––––––––––––––––––––––––––––––––––––––––––––––+

When additional services are added, they can add their own Plugin RPC, service
helper and device driver. Also, a service can have more than one device driver
too. The device driver to be used to configure for a particular device is
specified by the plugin as part of the resource dict in the RPC message.


Alternatives
------------

An alternative solution would be to configure the devices (service VMs)
directly from the service plugin. Though possible, this increases the load on
the plugin which is also responsible for the API and DB components too.


Data model impact
-----------------

The config agent does not use persistence. Hence no new models are added.
All existing models are unchanged.

REST API impact
---------------

None.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

Performance Impact
------------------

Having a seperate config agent for configuration tasks and health monitoring
reduces load on the neutron server, when compared to doing this from the plugin
directly. Secondly, since the config agent need not be colocated with the device
it configures and can manage multiple devices, agent state reporting traffic to
neutron server is reduced, compared to the l3 agent.

Since the config agent orchestrates how configurations for multiple services are
applied in a device, the control plane load on the device is reduced and
the risk of mis-configuration states is alleviated.

The config agent makes use of device drivers, which themselves uses Netconf,
REST APIs etc., to configure devices. These mechanisms need to connect and
authenticate with the device and establish a session before configurations can
be made. Session establishment adds an additional but necessary overhead.


Other deployer impact
---------------------

The service VMs have a management port which is connected to a management
network. The config agent connects to the devices over this management network
and hence the host where it is running should have connectivity to this network.


Developer impact
----------------

None. The routing service helper exposes the same RPC methods as the reference
l3 agent maintaining backward compatibility.

Implementation
==============

Assignee(s)
-----------

Hareesh Puthalath <hareesh-puthalath>

Work Items
----------

The work is split up into two parts:

  1. The config agent itself.
  2. Routing service driver for CSR1kv.


Dependencies
============

There are no new library requirements.
It does depend on a modified service plugin referred in the blueprint:
https://blueprints.launchpad.net/neutron/+spec/cisco-routing-service-vm


Testing
=======

Complete unit test coverage of the code is included.

For tempest test coverage, third party testing will be provided. The Cisco
CI reports on all changes affecting the config agent.


Documentation Impact
====================

None directly.
Though documentation needs to indicate how to use the config agent, if desired.


References
==========

http://www.cisco.com/c/en/us/products/routers/cloud-services-router-1000v-
series/index.html
