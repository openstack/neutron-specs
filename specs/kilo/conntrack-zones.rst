..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Conntrack zones support
==========================================


Problem Description
===================
Network isolation could be broken since security groups created for one
network can affect connectivity between ports of other network if they have
same IP addresses as ports on the initial network.

Forcing connection close via conntrack by IP will break existing connections
on networks which aren't related to security group being enforced.

See [1] for the reference.

Proposed Change
===============
The goal is to add support for conntrack zones which allow to handle multiple
connections with equal identities in conntrack and NAT.

A zone is simply a numerical identifier associated with a network
device that is incorporated into the various hashes and used to
distinguish entries in addition to the connection tuples. Additionally
it is used to separate conntrack defragmentation queues. An iptables
target for the raw table could be used alternatively to the network
device for assigning conntrack entries to zones. See [3] for more information.

In the case of security groups, each conntrack zone should correspond to a
neutron network. Adding some sort of network identifier makes connection tuple
unique.

In fact, that is already done in ovs agent, where there is a local vlan mapping.
Exactly the same strategy could be applied to conntrack zones.
Local vlan ids could be used as a conntrack zone id.

Changes are required in Firewall driver. It should keep current network-to-zone
mapping and apply port firewall rules with this additional parameter.
Upon ovs agent start/restart this mapping could be populated from local vlan
mapping. Changing zone identifies is ok because iptables rules are updated
after ovs agent restart.

This could also be utilized by other agents such as OFAgent.

Data Model Impact
-----------------
None

REST API Impact
---------------
None

Security Impact
---------------
This feature should actually improve security by fixing tenant network isolation.

Notifications Impact
--------------------
None

Other End User Impact
---------------------
None

Performance Impact
------------------
None or insignificant.

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
None

Alternatives
------------
None

Implementation
==============

Assignee(s)
-----------
yangxurong

Work Items
----------
1. The implementation for ovs agent. See [2]
2. Functional tests.
3. Implementations for other affected agents.

Dependencies
============
None

Testing
=======

Tempest Tests
-------------
None

Functional Tests
----------------
Functional test is required.

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
None


References
==========
[1] https://bugs.launchpad.net/neutron/+bug/1359523
[2] https://review.openstack.org/#/c/118274/
[3] http://lwn.net/Articles/370152/


