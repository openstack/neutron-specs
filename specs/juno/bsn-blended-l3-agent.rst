..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================================
Big Switch - optional offload of L3 features to L3 agent
========================================================


https://blueprints.launchpad.net/neutron/+spec/bsn-blended-l3-agent

Performing source nat and floating IP translations isn't possible in some
physical hardware. To handle these cases, the Big Switch plugin needs to inform
the L3 agent to perform these functions while leaving regular routing up to the
fabric for performance.


Problem description
===================

The physical switches that the BSN fabric uses for routing have limited table
sizes for rules. Under heavy load, there may be too many virtual machines with
floating IPs and SNAT for the switch to handle.


Proposed change
===============

To optionally offload the job of SNAT and floating IP translation, the backend
controller will issue a message back to the plugin on router creation that it
needs Neutron to handle source nat and/or floating IP translations. The plugin
will then issue an RPC message to have a router instantiated on an L3 agent.
Any time an interface is added to the Neutron router, an additional port will
be created for the L3 agent with a unique IP address.  All first-hop traffic
will be handled by the fabric, but any traffic leaving the tenant will be
routed through the L3 agent for floating IP translation or source NAT.

It is important to note that the creation of the L3 agent router is determined
by the backend controller. If there is a physical NAT box in the network, the
controller may use that instead, in which case no router would be instantiated
on an L3 agent.


Alternatives
------------

Always require special purpose NAT devices.

Data model impact
-----------------

N/A

REST API impact
---------------
N/A

Security impact
---------------
N/A

Notifications impact
--------------------
Potential L3 agent notifications when using the Big Switch plugin. The L3
agent will only receive notifications if the backend controller requires
the SNAT and/or the floating IP functions to be offloaded.

Other end user impact
---------------------
This will be mostly transparent to the end user other than a potential
extra IP address used by the additional router.

Performance Impact
------------------
This will improve dataplane scaling without requiring special-purpose hardware.

Other deployer impact
---------------------
L3 agents will need to be started when using the Big Switch plugin.

Developer impact
----------------
N/A


Implementation
==============

Assignee(s)
-----------
Primary assignee:
  kevinbenton

Work Items
----------
* Implement L3 RPC calls in Big Switch plugin
* Implement new response checking to router creations and floating IP creations
* Add unit tests


Dependencies
============
N/A


Testing
=======

The BSN CI will include a job exercising this offload capability.


Documentation Impact
====================
Mention L3 agent is now required when deploying the Big Switch Plugin.

References
==========
N/A
