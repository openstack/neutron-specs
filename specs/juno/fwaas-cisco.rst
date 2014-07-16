..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
FWaaS Implementation for Cisco Virtual Router
=============================================

https://blueprints.launchpad.net/neutron/+spec/fwaas-cisco

Problem description
===================
The Cisco Virtual Router implementation (CSR1kv) also supports the Firewall
Service in addition to Routing. The CSR1kv backend allows a Firewall to be
applied on any of it's interfaces for a specific direction of traffic. This
blueprint targets neutron support for this use case.

Proposed change
===============
Support of the Plugin and Agent/Driver for the CSR1kv Firewall is being
proposed in this blueprint. There are no changes to any of the Resources from
the Reference implementation. The OpenStack resources are translated to the
backend implmentation and the mapping to the backend resources is maintained.

Implementation targeted as a Service Plugin and will be refactored to align
with the Flavor Framework post Juno. Also given that the Service Insertion
BP[3] is in discussion, the initial implementation will be done using Vendor
extension attributes to capture the insertion points(Router interface &
direction) of the service in as simple a form as possible. This will be
refactored to align with the community post Juno.

Supporting the CSR1kv requires:
* Additional vendor attributes to specify firewall insertion points (neutron
port corresponding to router interface and associated direction). Supported
as vendor extension attributes as a simple model that will be refactored to
adopt the Service Insertion BP when available. The "extraroute" approach
will be taken to add the needed attributes of port and direction without
any changes to the client.
* Introduce new table to track insertion points of a firewall resource in the
vendor plugin.
* Interaction with the CSR1kv Routing Service Plugin[1] which is limited to
querying for the hosting VM and some validation for the attached interface.
* Add validators for the attribute extensions to conform to vendor
implementation constraints.
* Agent support for Firewall built on Cisco Config Agent[2] as a service agent
to handle messaging with the plugin along with the messaging interfaces
(firewall dict, plugin API and agent API) mostly along the lines of the
reference implementation.
* Agent to backend communication using existing vendor REST communication
library.

Alternatives
------------
The ideal approach is to base it on the flavor framework and service insertion
BP's. But given that these are WIP, this is being proposed as a Service Plugin
which will be refactored to align with the community model when available.

Data model impact
-----------------
There are no changes planned to existing Firewall resources (FirewallRule,
FirewallPolicy and Firewalls). The insertion point attributes are tracked
by introducing a new table CiscoFirewallAssociation:

* firewall_id  - uuid of logical firewall resource
* port_id      - uuid of neutron port corresponding to router interface
* direction    - direction of traffic on the portid to apply firewall on
                 can be:
                  - ingress
                  - egress
                  - bidirectional

REST API impact
---------------
No new REST API is introduced.

Security impact
---------------
None.

Notifications impact
--------------------
None to existing. New topic for messaging between the plugin and agent.

Other end user impact
---------------------
None.

Performance Impact
------------------
None.

Other deployer impact
---------------------
Deployer will have to enable the CSR1kv Routing Service Plugin, the Cisco
Config Agent in addition the CSR1kv Firewall Service Plugin being proposed
here. There is no impact to the community implementation when this is not
enabled. The Agent/backend driver is derived from the Service Plugin and
eventually from the flavor and this is messaged with the Config Agent avoiding
the need for a separate .ini file.

Developer impact
----------------
None.

Implementation
==============

The above figure is a representation of the CSR1kv components and
interactions. The CSR1kv Routing Service Plugin [1] and the Cisco Config
Agent[2] are being addressed in separate BP's. The work being targeted
here are the two items suffixed with a '*' and their interfaces to the
existing components.

::

 Neutron Server
 +---+---------------------+--------+             +-----------+
 |   +---------------------+        |             |Cisco Cfg  |
 |   | CSR1kv Routing      |        |             |  Agent    |
 |   | Service Plugin      |        |             |           |
 |   |                     |        |             |           |
 |   |                     |        |             |           |
 |   +---------------------+        |             +----+------+
 |           ^                      |             |CSR1|kv Firewall
 |           |                      |             |Agen|t*
 |           |          +------------------------>|    |<---+
 |           v          |           |             +----+    | Cisco
 |   +------------------v--+        |                       | Device specific
 |   | CSR1kv Firewall     |        |                       v REST i/f
 |   | Service Plugin*     |        |                     +------+
 |   |                     |        |                     |CSR1kv|
 |   |                     |        |                     | VM   |
 |   +---------------------+        |                     |      |
 |                                  |                     |      |
 |                                  |                     +------+
 |                                  |
 |                                  |
 +----------------------------------+


Assignee(s)
-----------
Primary assignee: skandasw
Other contributors: yanping

Work Items
----------
Service Plugin with vendor extension attributes for the Firewall Resource.
API & DB changes for the vendor specific extensions.
Cisco CSR1kv FWaaS service agent addition to the Cisco config Agent[2].

Dependencies
============
https://blueprints.launchpad.net/neutron/+spec/cisco-routing-service-vm
https://blueprints.launchpad.net/neutron/+spec/cisco-config-agent

Testing
=======
Unit tests, Tempest API tests and support for Vendor CI framework will be
addressed. Scenario tests will be attempted based on the tests available
for the reference implementation.

Documentation Impact
====================
Will require new documentation in Cisco sections.

References
==========
[1]https://blueprints.launchpad.net/neutron/+spec/cisco-routing-service-vm
[2]https://blueprints.launchpad.net/neutron/+spec/cisco-config-agent
[3]https://blueprints.launchpad.net/neutron/+spec/service-base-class-and-insertion
