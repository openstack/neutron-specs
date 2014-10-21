..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
FWaaS Implementation for Cisco Virtual Router
=============================================

https://blueprints.launchpad.net/neutron/+spec/fwaas-cisco

Adds support for vendor FWaaS implementation based on the Cisco Virtual
Router.

Problem Description
===================
The Cisco Virtual Router implementation (CSR1kv) also supports the Firewall
Service in addition to Routing. The CSR1kv backend allows a Firewall to be
applied on any of it's interfaces for a specific direction of traffic. This
blueprint targets neutron support for this use case.

Proposed Change
===============
Support of the Plugin and Agent/Driver for the CSR1kv Firewall is being
proposed in this blueprint. There are no changes to any of the Resources from
the Reference implementation. The OpenStack resources are translated to the
backend implementation and the mapping to the backend resources is maintained.

Supporting the CSR1kv requires:
* Additional vendor attributes to specify firewall insertion points (neutron
port_id corresponding to router interface and associated direction). Although
this is being proposed as vendor extensions, the new framework being proposed
to replace extensions will be adopted or this will evolve to that based on the
timeline of its availability. The "extraroute" approach will be taken to add
the needed attributes of port and direction without any changes to the neutron
client.
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

Implementation is being targeted as a Service Plugin and will be refactored to
align with the Flavor Framework once there is clarity on that effort. Also as
a Vendor Service Plugin, the effort will be refactored or realigned as the
Services split discussions are finalized. Also, if the Service Insertion BP[3]
or similar proposals are resubmitted in Kilo, this effort will be aligned with
the community direction.

Data Model Impact
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

REST API Impact
---------------
No new REST API is introduced.

Security Impact
---------------
None.

Notifications Impact
--------------------
None to existing. New topic for messaging between the plugin and agent.

Other End User Impact
---------------------
None.

Performance Impact
------------------
None.

IPv6 Impact
-----------
Expected to work with IPv6.

Other Deployer Impact
---------------------
Deployer will have to enable the CSR1kv Routing Service Plugin, the Cisco
Config Agent in addition the CSR1kv Firewall Service Plugin being proposed
here. There is no impact to the community implementation when this is not
enabled. The Agent/backend driver is derived from the Service Plugin and
eventually from the flavor and this is messaged with the Config Agent avoiding
the need for a separate .ini file.

Developer Impact
----------------
None.

Community Impact
----------------

The spec was reviewed and approved in the Juno Cycle. This is proposed as a
Vendor Service Plugin and will be refactored to align with any Community
efforts in this area.

Alternatives
------------
The ideal approach is to base it on the flavor framework and service insertion
BP's. But given that these are TBD, this is being proposed as a Service Plugin
which will be refactored to align with the community model when available.

Implementation
==============

The below figure is a representation of the CSR1kv components and
interactions. The CSR1kv Routing Service Plugin [1] and the Cisco Config
Agent[2] have been upstreamed in Juno. The work being targeted here are the
two items suffixed with a '*' and their interfaces to the existing components.

::

 Neutron Server
 +---+---------------------+--------+             +-----------+
 |   +---------------------+        |             |Cisco Cfg  |
 |   | CSR1kv Routing      |        |             |  Agent    |
 |   | Service Plugin      |        |             |           |
 |   |                     |        |             |           |
 |   |                     |        |             |           |
 |   +---------------------+        |             +------+----+
 |           ^                      |             |CSR1kv|
 |           |                      |             |  FW  |
 |           |          +------------------------>|Agent*|<-----+
 |           v          |           |             +------+      |
 |   +------------------v--+        |                           v
 |   | CSR1kv Firewall     |        |                      +-----------+
 |   | Service Plugin*     |        |                      |REST Client|
 |   |                     |        |                      |   lib     |
 |   |                     |        |                      +-----------+
 |   +---------------------+        |                            |
 |                                  |                            v
 |                                  |                         +----------+
 |                                  |                         |          |
 |                                  |                         |  CSR1kv  |
 |                                  |                         |    VM    |
 +----------------------------------+                         +----------+


Assignee(s)
-----------
Primary assignee: skandasw
Other contributors: yanping

Work Items
----------
Service Plugin with vendor extension attributes for the Firewall Resource.
API & DB changes for the vendor specific extensions.
Cisco CSR1kv FWaaS service agent addition to the Cisco config Agent[2].
REST client lib Refactor to work with Cisco FWaaS and VPN implementations.

Dependencies
============

None. All CSR specific components needed are already upstreamed.

Testing
=======

Unit tests are being added to address the changes.

Tempest Tests
-------------

Tests will be added for Vendor implementations along with CI support.

Functional Tests
----------------

Tests will be added to validate the CSR FWaaS implementation in association
with the CSR Routing implementation.

API Tests
---------

Tests will be added for the Vendor extensions being proposed.


Documentation Impact
====================

User Documentation
------------------

Will require new documentation in Cisco sections.

Developer Documentation
-----------------------

Although API extensions are being proposed, these are vendor extensions and
will be documented accordingly. There are no other consumers of the API
changes.

References
==========
[1]https://blueprints.launchpad.net/neutron/+spec/cisco-routing-service-vm
[2]https://blueprints.launchpad.net/neutron/+spec/cisco-config-agent
[3]https://blueprints.launchpad.net/neutron/+spec/service-base-class-and-insertion
