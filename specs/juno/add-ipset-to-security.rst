..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Add ipset to security group
==========================================

https://blueprints.launchpad.net/neutron/+spec/add-ipset-to-security

Now neutron uses iptables to achieve security group functions, but iptables
chain is linear storage and filtering, we can use ipset to improve security
group's performance.


Problem description
===================

Now neutron uses iptables to achieve security group functions, but there are
following problems.

Problem 1:
When a security group has many rules between other security group, this would
affect security group's performance.

Problem 2:
When a port is updated, the security group chain related to that port will
be destroyed and rebuilt, this would affect L2 agent's performance.


Proposed change
===============

The proposal is to improve security group performance based on existing
security group agent codes.

In L2 agent, it makes use of ipset to optimize the iptables rule chain. When a
port is created, L2 agent will add a additional ipset chain to it's iptables
chain, if the security group that this port belongs to has rules between other
security group, the member of that security group will add to ipset chain.

When a port is created in default security group, its corresponding iptables
rules in L2 agent like these(old result):
-A neutron-openvswi-i92605eaf-b -m state --state INVALID -j DROP
-A neutron-openvswi-i92605eaf-b -m state --state RELATED,ESTABLISHED -j RETURN
-A neutron-openvswi-i92605eaf-b -s 192.168.83.20/32 -j RETURN
-A neutron-openvswi-i92605eaf-b -s 192.168.83.19/32 -j RETURN
-A neutron-openvswi-i92605eaf-b -s 192.168.83.16/32 -j RETURN
-A neutron-openvswi-i92605eaf-b -s 192.168.83.17/32 -j RETURN
-A neutron-openvswi-i92605eaf-b -s 192.168.83.18/32 -j RETURN
-A neutron-openvswi-i92605eaf-b -s 192.168.83.15/32 -j RETURN

The new iptables rules by using 'iptables+ipset' like these:
-A neutron-openvswi-i92605eaf-b -m state --state INVALID -j DROP
-A neutron-openvswi-i92605eaf-b -m state --state RELATED,ESTABLISHED -j RETURN
-A neutron-openvswi-i92605eaf-b -m set --match-set ${ipset_name} src -j RETURN

The ipset chain like this:
Name: ${ipset_name}
Type: hash:ip
Header: family inet hashsize 1024 maxelem 65536
Size in memory: 16632
References: 1
Members:
192.168.83.16
192.168.83.17
192.168.83.15
192.168.83.18
192.168.83.19
192.168.83.20


Alternatives
------------

None.

Data model impact
-----------------

None.

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

1) iptables-save/iptables-load time will be greatly reduced, as we just need to
modify one ipset per security group, and not all the bunch of duplicated rules
per port.

2) With iptables, kernel has to linearly go evaluating after all iptables rules
in chain order, while for an ipset in iptables, a lots of rules will become
into a "hash" with much faster evaluation.


Other deployer impact
---------------------

Considered a deployer doesn't provide ipset in their system, using
'iptables+ipset' to enhance security group will be optional.
A configuration flag will be added to L2 agent configuration file:

[securitygroup]
enable_ipset_enhancement = False

Developer impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  https://launchpad.net/~shihanzhang

Other contributors:
  https://launchpad.net/~mangelajo


Work Items
----------

* Improve 'security_group_rules_for_devices'
* Add ipset chain in '_add_rule_by_security_group'
* Unit tests


Dependencies
============

None.


Testing
=======

Unit tests should be enough, as this is to optimize already existing
functionality which is already covered by tempest


Documentation Impact
====================

None.


References
==========

* http://ipset.netfilter.org
