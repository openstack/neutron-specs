..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================================
'security_group_rules_for_devices' RPC call refactoring
=======================================================

https://blueprints.launchpad.net/neutron/+spec/security-group-rules-for-devices-rpc-call-refactor

Security group rules synchronization from neutron-server to L2 agents scales
poorly for high density clouds, leading to a blocked neutron-server in some
situations.


Problem description
===================
The security_group_rules_for_devices RPC call from L2 agents to
neutron-server doesn't scale well because all the security group rule entries
are expanded with each specific IP address (see [#call_results]_) in a security
group, when that group is referenced in rules like:

allow all from 'default' group for IPv4 and IPv6

This leads to:

* huge AMQP messages (>20-600 MB)

* very long processing time at neutron-server side when we have lots of
  instances under the same tenant/security group (>60 seconds)

* neutron-server lockups, when RPC call times out at agent side, and the same
  security_group_rules_for_devices call is issued back to neutron.

For a more detailed insight:

* security_group_rules_for_devices is an RPC call from the L2 agents to
  neutron-server, see [#call_results]_.

This call's arguments are a list of device_ids, device_ids are connected
to ports. Neutron builds a list [#call_results]_ of security group rules and
returns the list of security group rules per device_id

The message size between neutron-server and L2 agent will grow
according to the following formula:

MessageSize ~= base + L * Instances_in_host * (Instances_in_security_group-1)

Where L = len(str(security_group_rule)) ~=440 bytes

This problem is more likely to happen in big clouds, or denser
ones, but it's an issue, as nova can scale to 10x instances
without incident: see [#n2nov_scaling]_.


Proposed change
===============

Refactor the security_group_rules_for_devices into a new call
'security_group_rules_for_devices_compact', which instead of returning
[#call_results]_ , returns a more compact result
that can be expanded by the l2 agent.

The new 'security_groups_rules_for_devices_compact' RPC call returns:

.. code-block:: python

    {'security_groups': {'sg-id1': {'rules': [ {},{},{},{}]}
                        },
     'security_group_member_ips':  { 'sg-id1' : {'ipv4': ['192.168.11.2/32'],
                                                 'ipv6': [] },
                                     'sg-id2-referenced-from-id1' : {...},
                                   },

     'devices': {'dev-id1': { ... ,
                             'fixed_ips': ['192.168.11.4'],
                             'security_groups': ['sg-id1', 'sg-id2', ...] }}
    }


Security group rules would be passed in a non-expanded way, which implies
that any reference to src or dst security groups in rules would be stored
as security group ids.


A response like this (old result):

.. code-block:: python

    {'dev-id1':
     {...,
      'security_group_rules': [{'direction': u'egress',
                                'ethertype': u'IPv6',
                                'security_group_id':
                                 u'1809f907-4b0c-4445-a366-ff28eaab9c2e'},
                              {'direction': u'egress',
                               'ethertype': u'IPv4',
                               'security_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e'},
                              {'direction': u'ingress',
                               'ethertype': u'IPv4',
                               'protocol': u'icmp',
                               'security_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e'},
                              {'direction': u'ingress',
                               'ethertype': u'IPv4',
                               'remote_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e',
                               'security_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e',
                               'source_ip_prefix': '192.168.11.2/32'},
                              {'direction': u'ingress',
                               'ethertype': u'IPv4',
                               'remote_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e',
                               'security_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e',
                               'source_ip_prefix': '192.168.11.3/32'},
                              {'direction': u'ingress',
                               'ethertype': u'IPv4',
                               'remote_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e',
                               'security_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e',
                               'source_ip_prefix': '192.168.11.4/32'},
                              {'direction': u'ingress',
                               'ethertype': u'IPv4',
                               'remote_group_id':
                                       u'23138476-4fde-454e-33ad-abc123456782',
                               'security_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e',
                               'source_ip_prefix': '192.168.33.4/32'}
                                ]
     },
    'dev-id2': {
     ...,
     'security_group_rules': [{'direction': u'egress',
                               'ethertype': u'IPv6',
                               'security_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e'},
                              {'direction': u'egress',
                               'ethertype': u'IPv4',
                               'security_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e'},
                              {'direction': u'ingress',
                               'ethertype': u'IPv4',
                               'protocol': u'icmp',
                               'security_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e'},
                              {'direction': u'ingress',
                               'ethertype': u'IPv4',
                               'remote_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e',
                               'security_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e',
                               'source_ip_prefix': '192.168.11.2/32'},
                              {'direction': u'ingress',
                               'ethertype': u'IPv4',
                               'remote_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e',
                               'security_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e',
                               'source_ip_prefix': '192.168.11.3/32'},
                              {'direction': u'ingress',
                               'ethertype': u'IPv4',
                               'remote_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e',
                               'security_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e',
                               'source_ip_prefix': '192.168.11.4/32'},
                              {'direction': u'ingress',
                               'ethertype': u'IPv4',
                               'remote_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e',
                               'security_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e',
                               'source_ip_prefix': '192.168.11.5/32'},
                              {'direction': u'ingress',
                               'ethertype': u'IPv4',
                               'remote_group_id':
                                       u'23138476-4fde-454e-33ad-abc123456782',
                               'security_group_id':
                                       u'1809f907-4b0c-4445-a366-ff28eaab9c2e',
                               'source_ip_prefix': '192.168.33.4/32'}
                                ]
     }
    }


Would be like this in the new version:

.. code-block:: python

    {'security_groups': {u'1809f907-4b0c-4445-a366-ff28eaab9c2e':
                          {'rules': [
                             {'direction': u'egress', 'ethertype': u'IPv6'},
                             {'direction': u'egress', 'ethertype': u'IPv4'},
                             {'direction': u'ingress',
                              'ethertype': u'IPv4',
                              'protocol': u'icmp',},
                             {'direction': u'ingress',
                              'ethertype': u'IPv4',
                              'remote_group_id':
                                   u'1809f907-4b0c-4445-a366-ff28eaab9c2e'},
                             {'direction': u'ingress',
                              'ethertype': u'IPv4',
                              'remote_group_id':
                                   u'23138476-4fde-454e-33ad-abc123456782'}
                            ]
                           }
                        },
     'security_group_member_ips':  { u'1809f907-4b0c-4445-a366-ff28eaab9c2e' :
                                     {u'ipv4': ['192.168.11.2/32',
                                               '192.168.11.3/32',
                                               '192.169.11.4/32',
                                               '192.168.11.5/32'],
                                      u'ipv6': []
                                     },
                                     u'23138476-4fde-454e-33ad-abc123456782' :
                                     {u'ipv4': ['192.168.33.2/32'],
                                      u'ipv6': []
                                     }
                                   },

     'devices': {'dev-id1': { ... ,
                             'fixed_ips': ['192.168.11.4'],
                             'security_groups':
                              ['1809f907-4b0c-4445-a366-ff28eaab9c2e'] },
                 'dev-id2': { ... ,
                             'fixed_ips': ['192.168.11.4'],
                             'security_groups':
                              ['1809f907-4b0c-4445-a366-ff28eaab9c2e'] },

                 }


All security groups referenced from devices will be included in the
response.

All security group members ip addresses from all remote_group_id referenced
groups will be included in the response.

The old call could be marked as deprecated during this J cycle,
and removed during K cycle.

Making the refactor into a new call would have the following advantages:

#. Compatibility during neutron-server upgrade with older agents
#. Ability to split patches (server/agents) in more steps, as we will have
   the ability to address the new call, while keeping the agents calling
   the old one, and then refactor the agents in further steps.

The resulting message size would be:

MessageSize ~= base +
               D * Instances_in_host +
               L * Referenced_security_groups +
               I * Instances_in_referenced_security_groups

Where L = len(str(compact_security_group_rule)) ~= 220 bytes
      D = len(str(device_description_including_ips_and_sg_ids))
      I = len(str(ip_address + ',')) ~= 17 bytes

In the new message format no data is replicated, thus now two variables
become multiplication factors.


Next steps:

There is a proposal from Édouard Thuleau to use an RPC topic per security
group [#sec_fanout]_ which would be addressed in a second iteration after
this one.


Alternatives
------------

* Instead of including all the security groups in one rpc call, this could
  be split to a second call, 'security_groups_and_referenced_members', which
  would receive a list of security groups ids, and would return a list of
  security groups and a list of security group ip addresses.
  A full sync would require 2 calls to neutron server.

* We could have a 'security_groups' and a 'security_groups_members', which
  would provide the security groups, without member IP addresses and,
  a list of IPv4 and IPv6 addresses members for each security group.
  A full sync would require 3 calls to neutron server. But this approach
  would allow separate communication of new members in security groups,
  or new rules in security groups, further reducing the information transmited
  in those cases. As for the first alternative, reducing the traffic volume
  by increasing the number of calls seems like a bad tradeoff due to the
  overhead/latency generated by each call.

* We could just compact CIDR ranges in rules generation, that wouldn't
  require modifications to the agents, but that would increase the
  rpc request processing time.



Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

The performance impact should be very positive in the next situations:

* Security group changes: neutron-server load, and AMQP message sizes.
  At this moment oslo messaging serializes structures to JSON because of
  AMQP version limitations (string sizes for dictionaries is
  one of those). Reducing the message size will reduce the dictionary
  building time and also the serialization time.

* Creation time for new ports

Even higher performance impact in packet processing would be achieved
by the optimization at iptables level which is proposed in the ipset spec
[#ipset_spec]_.

If tranmission times become our bottleneck instead of processing times
we may consider compression if that's available at the AMQP level.

Other deployer impact
---------------------

None

Developer impact
----------------

* I'm unsure if there are proprietary l2-agents talking to the RPC.
  We would allow some time for those to be upgraded by introducing
  it as a new RPC call instead of modifying the existing one.

* All the agents that use this RPC call would need to be updated
  to the new call before removing the old one.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  https://launchpad.net/~mangelajo

Other contributors:
  http://launchpad.net/~shihanzhang


Work Items
----------

* Refactor the rpc call into a new one in neutron-server and
  and the matching rpc call at the agent mixin.

* Add functional testing to validate the approach.

* Upgrade the agents to use the new call, one by one.

* Analyze db access and file a new spec if improvements can
  be made in this area.

Dependencies
============

None

Testing
=======

Functional testing will validate the approach and make sure
the result that was possible with the old method can be
consistently replicated (only faster) with the new method.
Testing both rpc calls with functional testing will avoid
regressions in both of the code paths, while testing this
in Tempest only allows testing the default rpc call.

Documentation Impact
====================

None

References
==========

.. [#call_results] http://www.fpaste.org/104401/14008522/

.. [#n2nov_scaling] http://javacruft.wordpress.com/2014/06/18/168k-instances/

.. [#sec_fanout] http://lists.openstack.org/pipermail/openstack-dev/2014-June/038374.html

.. [#ipset_spec] https://review.openstack.org/#/c/100761/
