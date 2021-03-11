..
       This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
QoS Rule Type Packet per Second
===============================

RFE: https://bugs.launchpad.net/neutron/+bug/1912460

Neutron supports bandwidth rate limit for ports and L3 IPs. But packet
rate limit (packet per second) is not available, although it is a common
measurement.

So, this spec describes adding a new QoS rule type packet per second (pps).

Problem Description
===================

Packet per second is a very general network performance metric.
Like bandwidth, it is usually used to evaluate the packet forwarding
performance of a device.

For cloud providers, to limit the packet per second (pps)
of VM NIC is popular and sometimes essential. Transit large set of
packets for VM in physical compute hosts will consume the
CPU and physical NIC I/O performance. For small packets, even if the
bandwidth is low, the pps can still be higher. Without the limitation,
it can be an attack point inside the cloud while some VMs are becoming
hacked.

For L2 drivers like ovs and ovn, it may get extremly high usage of
CPU when user send small packet (typically 64B small) from
the VM to the others, even if the device has lower QoS bandwidth limitation.
Then your host services and other users' VMs will be under a higher
failure point.

For network quality assurance, the resource consumption of the system
is determined according to the user's VM specifications. With the pps
limitation, the VM will not consume more CPU with smaller bandwidth.

Proposed Change
===============

Adding new API extension to QoS service plugin to allow CURD actions for
packet rate limit (packet per second) rule.

.. note:: We will not elaborate the real limitation in L2/L3 backend,
          this spec only shows how we add a new QoS rule type.

Server side changes
-------------------

A new API extension of Neutron will be added with new resource
``PacketRateLimitRule``:

::

    qos_apidef.SUB_RESOURCE_ATTRIBUTE_MAP = {
        'packet_rate_limit_rules': {
            'parent': qos_apidef._PARENT,
            'parameters': {
                qos_apidef._QOS_RULE_COMMON_FIELDS,
                'max_kpps': {
                    'allow_post': True, 'allow_put': True,
                    'convert_to': converters.convert_to_int,
                    'is_visible': True,
                    'is_filter': True,
                    'is_sort_key': True,
                    'validate': {
                        'type:range': [0, db_const.DB_INTEGER_MAX_VALUE]}
                },
                'max_burst_kpps': {
                        'allow_post': True, 'allow_put': True,
                        'is_visible': True, 'default': 0,
                        'is_filter': True,
                        'is_sort_key': True,
                        'convert_to': converters.convert_to_int,
                        'validate': {
                            'type:range': [0, db_const.DB_INTEGER_MAX_VALUE]}
                },
                'direction': {
                    'allow_post': True,
                    'allow_put': True,
                    'is_visible': True,
                    'is_filter': True,
                    'is_sort_key': True,
                    'default': constants.EGRESS_DIRECTION,
                    'validate': {
                        'type:values': constants.VALID_DIRECTIONS}
                }
            }
        }
    }

.. note:: The unit for the rate and burst is kilo (1000) packets per second,
          so the value range will be 1 kpps to 2147 gpps.

Potential agent side enhancements
---------------------------------

Each of the following will be an independent and huge proposal, we will
not describe the detail here.

* apply pps rule to L3 IPs in agent side by iptables rule [1]_.
* apply pps rule to VM port by ovs meter [2]_ [3]_ [4]_.
* apply pps rule to router ports (gateway port, router interface) by iptables rule.
* apply pps rule to L3 IPs and ports to OVN related devices by ovs meter.

User use cases
--------------

After the L2/L3 backends have the real abilities to limit the pps,
users can use the pps rule in the following scenarios:

* "pps" rule for floating IP, using the L3 agent
* "pps" rule for floating IP, using OVN L3 agent
* "pps" rule for gateway IP, using the L3 agent
* "pps" rule for gateway IP, using OVN L3 agent
* "pps" rule for ML2 OVS VM ports
* "pps" rule for ML2 OVN VM ports
* "pps" rule for router gateway or interface ports

Data Model Impact
-----------------

Add table ``qos_packet_rate_limit_rules``:

::

        op.create_table(
            'qos_packet_rate_limit_rules',
            sa.Column('id', sa.String(36), nullable=False,
                      index=True),
            sa.Column('qos_policy_id', sa.String(36),
                      nullable=False, index=True),
            sa.Column('max_kpps', sa.Integer()),
            sa.Column('max_burst_kpps', sa.Integer()),
            sa.Column('direction', sa.Enum(constants.EGRESS_DIRECTION,
                                           constants.INGRESS_DIRECTION,
                                           name="directions"),
                      nullable=False,
                      server_default=constants.EGRESS_DIRECTION),
            sa.PrimaryKeyConstraint('id'),
            sa.ForeignKeyConstraint(['qos_policy_id'], ['qos_policies.id'],
                                    ondelete='CASCADE')
        )

Add DB model ``QosPacketRateLimitRule``:

::

    class QosPacketRateLimitRule(model_base.HasId, model_base.BASEV2):
        __tablename__ = 'qos_packet_rate_limit_rules'
        qos_policy_id = sa.Column(sa.String(36),
                                  sa.ForeignKey('qos_policies.id',
                                                ondelete='CASCADE'),
                                  nullable=False)
        max_kpps = sa.Column(sa.Integer)
        max_burst_kpps = sa.Column(sa.Integer)
        revises_on_change = ('qos_policy',)
        qos_policy = sa.orm.relationship(QosPolicy, load_on_pending=True)
        direction = sa.Column(sa.Enum(constants.EGRESS_DIRECTION,
                                      constants.INGRESS_DIRECTION,
                                      name="directions"),
                              default=constants.EGRESS_DIRECTION,
                              server_default=constants.EGRESS_DIRECTION,
                              nullable=False)
        __table_args__ = (
            sa.UniqueConstraint(
                qos_policy_id, direction,
                name="qos_packet_rate_limit_rules0qos_policy_id0direction"),
            model_base.BASEV2.__table_args__
        )

With OVO object:

::

    @base.NeutronObjectRegistry.register
    class QosPacketRateLimitRule(QosRule):

        db_model = qos_db_model.QosPacketRateLimitRule

        fields = {
            'max_kpps': obj_fields.IntegerField(nullable=True),
            'max_burst_kpps': obj_fields.IntegerField(nullable=True),
            'direction': common_types.FlowDirectionEnumField(
                default=constants.EGRESS_DIRECTION)
        }

        duplicates_compare_fields = ['direction']

        rule_type = constants.RULE_TYPE_PACKET_RATE_LIMIT

REST API Impact
---------------

GET: List packet rate limit rules for QoS policy

* /v2.0/qos/policies/{policy_id}/packet_rate_limit_rules

Response:

::

  {
    "packet_rate_limit_rules": [
        {
            "id": "5f126d84-551a-4dcf-bb01-0e9c0df0c793",
            "max_kpps": 10000,
            "max_burst_kpps": 0,
            "direction": "egress"
        }
    ]
  }

POST: Create packet rate limit rule

* /v2.0/qos/policies/{policy_id}/packet_rate_limit_rules

Request:

::

  {
    "packet_rate_limit_rule": {
        "max_kpps": "10000"
    }
  }

Response:

::

  {
    "packet_rate_limit_rule": {
        "id": "5f126d84-551a-4dcf-bb01-0e9c0df0c793",
        "max_kpps": 10000,
        "max_burst_kpps": 0,
        "direction": "egress"
    }
  }

GET: Show packet rate limit rule details

* /v2.0/qos/policies/{policy_id}/packet_rate_limit_rules/{rule_id}

Response:

::

  {
    "packet_rate_limit_rule": {
        "id": "5f126d84-551a-4dcf-bb01-0e9c0df0c793",
        "max_kpps": 10000,
        "max_burst_kpps": 0,
        "direction": "egress"
    }
  }

PUT:  Update packet rate limit rule

* /v2.0/qos/policies/{policy_id}/packet_rate_limit_rules/{rule_id}


Request:

::

  {
    "packet_rate_limit_rule": {
        "max_kpps": 10000
    }
  }

Response:

::


  {
    "packet_rate_limit_rule": {
        "id": "5f126d84-551a-4dcf-bb01-0e9c0df0c794",
        "max_kpps": "10000"
    }
  }

DELETE: Delete packet rate limit rule

* /v2.0/qos/policies/{policy_id}/packet_rate_limit_rules/{rule_id}

And, neutron will allow attaching new ``PacketRateLimitRule`` to QoS policy.

The Neutron basic workflow
--------------------------

1. User creates QoS policy
2. Creates packet rate limit rules with multiple directions to this QoS policy
3. Attaching this QoS policy to a port
4. (No available) related L2 driver apply PPS limitation driver rule to the port
5. Attaching this QoS policy to a L3 IP (floating IP or gateway IP).
6. (No available) related L3 driver apply PPS limitation driver rule to the IP

Implementation
==============

Assignee(s)
-----------

* LIU Yulong <i@liuyulong.me>


Work Items
----------

* Adding API extension and DB models for neutron server.
* Testing.
* Documentation.

Dependencies
============

None

Testing
=======

Unit test cases to verify the DB rules are created/updated/deleted.

References
==========

.. [1] https://linux.die.net/man/8/iptables
.. [2] http://workshop.netfilter.org/2017/wiki/images/d/db/Nfws-ovs-metering.pdf
.. [3] http://www.openvswitch.org//support/dist-docs/ovs-ofctl.8.txt
.. [4] https://github.com/openvswitch/ovs/blob/master/NEWS#L312
