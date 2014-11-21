=======================
Kilo Project Priorities
=======================

List of work items the neutron development team is prioritizing in Kilo.

+-----------------------------------+--------------------------+
| Priority                          | Owner                    |
+===================================+==========================+
| REST/RPC/Plugin API Refactor      | Mark McClain             |
+-----------------------------------+--------------------------+
| nova-network to neutron Migration | Oleg Bondarev            |
+-----------------------------------+--------------------------+
| Plugin Decomposition              | Armando Migliaccio       |
+-----------------------------------+--------------------------+
| Testing                           | Maru Newby               |
+-----------------------------------+--------------------------+
| Advanced Services Split           | Kyle Mestery             |
+-----------------------------------+--------------------------+
| L2 Agent Refactor                 | Armando Migliaccio       |
+-----------------------------------+--------------------------+
| L3 Agent Refactor                 | Carl Baldwin             |
+-----------------------------------+--------------------------+
| DHCP Agent Refactor               | Salvatore Orlando        |
+-----------------------------------+--------------------------+
| Pluggable IPAM                    | Salvatore Orlando        |
+-----------------------------------+--------------------------+
| VM Trunk Ports                    | Maru Newby               |
+-----------------------------------+--------------------------+
| Agent Child Processes Status      | Miguel Angel Ajo         |
+-----------------------------------+--------------------------+
| Rootwrap Daemon Mode              | Miguel Angel Ajo         |
+-----------------------------------+--------------------------+

REST / RPC / Plugin API Refactor
--------------------------------
This work covers all the refactoring of the core layers of Neutron. This
includes the following:

* RPC refactor
* REST API refactor
* Plugin interface refactor
* Switch to pecan from homegrown WSGI

`REST / RPC / Plugin API etherpad <https://etherpad.openstack.org/p/neutron-rest-api>`_

nova-network to neutron Migration
---------------------------------
This tracks the work on the Neutron side involved with migrating existing
deployments from nova-network to Neutron.

`nova-network to neutron Migration <https://etherpad.openstack.org/p/kilo-nova-nova-network-to-neutron>`_

Plugin Decomposition
--------------------
This work outlines the plan for addressing the pain points that every
Neutron contributor feels on a daily basis when working with the project.
This is specifically focused on the thinning of plugins and drivers in the
tree, which benefits the project as well as the plugins and drivers
themselves.

`Plugin Decomposition etherpad <https://etherpad.openstack.org/p/aE7ydRU35m>`_

Testing
-------
This work covers all of the testing going on for Kilo. This includes:

* Full-stack testing
* DHCP agent functional testing
* Metadata agent functional testing
* OVS agent functional testing

L2 Agent Refactor
-----------------
This tracks all the updates and technical debt repayment around the L2 agent.
This includes:

* Higher level abstractions for ovs_lib
* Functional testing for the agent
* RPC improvements
* OVSDB monitoring improvements
* Modular Layer2 Agent

`L2 Agent Refactor etherpad <https://etherpad.openstack.org/p/kilo-neutron-agents-technical-debt>`_

L3 Agent Refactor
-----------------
This work covers refactoring of the L3 agent, including work needed for
the advanced services split in the case of services utilizing the L3 agent.

`Technical Debt in Agents etherpad <https://etherpad.openstack.org/p/kilo-neutron-agents-technical-debt>`_

DHCP Agent Refactor
-------------------

This work focuses in refactoring and improving the performance and reliability
of the DHCP agent. This includes:

* Restart improvements
* Load based scheduling
* Dead agent rescheduling
* Functional testing

`Technical Debt in Agents etherpad <https://etherpad.openstack.org/p/kilo-neutron-agents-technical-debt>`_

Advanced Services Split
-----------------------
This work covers the splitting out of advanced services in Neutron. This
includes LBaaS, VPNaaS, and FWaaS.

`Advanced Services Split <https://etherpad.openstack.org/p/neutron-services>`_

Pluggable IPAM
--------------
This work introduces the pluggable IPAM subsystem in Neutron that allows
flexible control over the lifecycle of the network resources.

`Pluggable IPAM etherpad <https://etherpad.openstack.org/p/neutron-ipam>`_

VM Trunk Ports
--------------
There is heavy interest from the NFV team around this item. This will track
the various BPs around this work with the goal of implementing this API
support in Kilo.

Agent Child Processes Status
----------------------------
Neutron agents spawn external detached processes which run unmonitored, if
anything happens to those processes neutron won't take any action,
failing to provide those services reliably.

We propose monitoring those processes, and taking a configurable action,
making neutron more resilient to external failures.

Rootwrap Daemon Mode
--------------------
Neutron is one of projects that heavily depends on executing actions on network
nodes that require root priviledges on Linux system. Currently this is achieved
with oslo.rootwrap that has to be run with sudo. Both sudo and rootwrap produce
significant performance overhead. This blueprint covers mitigating the sudo
and rootwrap introduced part of the overhead by using the new mode of
rootwrap operation called 'daemon mode'.
