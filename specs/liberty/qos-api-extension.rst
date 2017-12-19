..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Neutron QoS API Models and Extension
====================================

https://blueprints.launchpad.net/neutron/+spec/quantum-qos-api

The scope of this spec is to implement the bandwidth limiting API and layout
the QoS models for future API and models extension introducing more
types of QoS rules.

In the network jargon QoS (Quality of Service) is about limiting, prioritizing
or guaranteeing speed of traffic, in this case, on neutron ports.
For example we could mark traffic to be handled with higher priority
by switches/routers (IP/VLAN headers), or we may want to set minimum or maximum
bandwidth constraints on certain types of traffic (realtime traffic needs
to be processed with higher priority: Voice over IP, streaming, ..., to reduce
latency), or the administrator of a cloud may want to offer different service levels
based on the available network bandwidth.

As we propose them, QoS policies could be applied:

* Per network: All the ports plugged on the network where the QoS policy is
               applied get the policy applied to them.

* Per port: The specific port gets the policy applied, when the port had any
            network policy that one is overridden.

Problem Description
===================

The definition of network QoS can involve many parameters and it is
difficult to standardize support for it across installations and Cloud
Service Providers (CSPs). Currently, there are a number of Neutron
plugins that have their own quality of service API extension, but each
has their own parameters and structure. (Note: look for the extensions
and reference them)

Proposed Change
===============

The CSP could provide the option of choosing a QoS policy from a
pre-configured list of policies which that particular CSP supports.
A common way to express such
categorization is through the use of the name field in a
QoS policy, which can be used to create arbitrary levels of
service like “Platinum”, “Gold”, “Silver”,
“Bronze”, and “Best-effort” levels. Note, these categories are just
examples, the specific naming convention could be defined by the CSP,
and could have a different set of policies in this list.

Even from a tenant’s perspective, it might be convenient and easier to
select the desired QoS from a CSP supplied list, rather than have to
articulate complex details of the QoS policy. The actual definition of
the network QoS to which each of these levels maps to might vary
between installations and CSPs, and could include bandwidth
guarantees, priorities, etc., and will be left, by default to the CSP
admin to define as stated before.


In Telcos use cases, tenants could be interested (and trusted) to
configure policies themselves. That could be allowed by modifying the
default policies.json provided with neutron.

Data Model Impact
-----------------

The model consist of two main parts, policies, and rules. Policies are
composed of rules. Even if that can look overcomplicated as a start,
it will allow the feature to be easily extended without changes
to object relationships (QoSPolicy <-> Port / QoSPolicy <-> Network) in the
future.

For the case of bandwidth limiting, we have a third model QoSBandwidthLimitRule
that extends QoSRule base data model.

New rules could be introduced by adding new submodels to QoSRule.

QoSPolicy model has the following attributes:

+-----------+-------+-----------+---------+--------------+----------------+
|Attribute  |Type   |Access     |Default  |Validation/   |Description     |
|Name       |       |           |Value    |Conversion    |                |
+===========+=======+===========+=========+==============+================+
|id         |string |RO, all    |generated|uuid          |identity        |
|           |(UUID) |           |         |              |                |
+-----------+-------+-----------+---------+--------------+----------------+
|name       |string |RW, tenant |N/A      |none          |QoS name        |
+-----------+-------+-----------+---------+--------------+----------------+
|description|string |RW, tenant |N/A      |none          |QoS description |
+-----------+-------+-----------+---------+--------------+----------------+
|shared     |bool   |RW, tenant |False    |Boolean       | If accessible  |
|           |       |           |         |              | by other       |
|           |       |           |         |              | tenants        |
+-----------+-------+-----------+---------+--------------+----------------+
|tenant_id  |string |RO, tenant |N/A      |uuid          | Tenant ID of   |
|           |(UUID) |           |         |              | QoS object     |
|           |       |           |         |              | owner          |
+-----------+-------+-----------+---------+--------------+----------------+

the QoSRule object, which composes QoSPolicy, has the following attributes:

+-----------------+-------+-----------+---------+--------------+---------------+
|Attribute        |Type   |Access     |Default  |Validation/   |Description    |
|Name             |       |           |Value    |Conversion    |               |
+=================+=======+===========+=========+==============+===============+
|id               |string |RO, all    |generated|uuid          |identity       |
|                 |(UUID) |           |         |              |               |
+-----------------+-------+-----------+---------+--------------+---------------+
|qos_policy_id    |string |RO, all    |N/A      |uuid          | QoSPolicy     |
|                 |(UUID) |           |         |              | reference     |
+-----------------+-------+-----------+---------+--------------+---------------+
|type             |string |RW, tenant |N/A      |enum          | Type of QoS   |
+-----------------+-------+-----------+---------+--------------+---------------+
|direction        |string |RW, tenant |egress   |ingress/      | (reserved)    |
|                 |       |           |         |egress        | from vm       |
|                 |       |           |         |              | perspective   |
+-----------------+-------+-----------+---------+--------------+---------------+
|tenant_id        |string |RO, tenant |generated|uuid          | Tenant ID of  |
|                 |(UUID) |           |         |              | QoS object    |
|                 |       |           |         |              | owner         |
+-----------------+-------+-----------+---------+--------------+---------------+

The QoSBandwidthLimitRule model would look like:

+-----------------+-------+-----------+---------+--------------+---------------+
|Attribute        |Type   |Access     |Default  |Validation/   |Description    |
|Name             |       |           |Value    |Conversion    |               |
+=================+=======+===========+=========+==============+===============+
|qos_rule_id      |string |RO, all    |N/A      |uuid          | QoSRule we    |
|                 |(UUID) |           |         |              | extend        |
+-----------------+-------+-----------+---------+--------------+---------------+
|max_kbps         |integer|RW, tenant |N/A      |not NULL, >0  |               |
+-----------------+-------+-----------+---------+--------------+---------------+
|max_burst_kbps   |integer|RW, tenant |N/A      |NULL or >0    | burst over    |
|                 |       |           |         |              | max_kbps      |
+-----------------+-------+-----------+---------+--------------+---------------+

Future work beyond this spec
----------------------------
Traffic classification: traffic classifier which can be attached to QoS
rules, so the bandwidth limitations or specific marks are handled only on
certain types of traffic.

Example workflow with TC::

     neutron qos-tc-create sip-traffic \
              --protocol udp --port 5060  \
              --description "SIP signaling"

     neutron qos-rule-create <rule-type> \
                             <policy-name-or-id> \
                             [.. rule parameters ..] \
                             --traffic-classifier <tc-id>

We believe traffic classifiers should be something to be shared with other
pieces of software in neutron (tap as a service, service chaining, etc..).


More rule types:
* Traffic marking: dscp, ipv6 flow labels, vlan 802.1p
* Bandwidth guarantees: best effort, or strict (in conjunction with
nova-scheduling)

Access control to QoS policy could be performed by a generalized
RBAC support. Evaluate quota controlling the different types policies.


Other QoS work:
* Congestion notification support: setting the IP ECN bit over the
tenant network packets when guarantees can't be met, for example.

Explore integration with flavor framework.

Network aggregated bandwidth limit.

REST API Impact
---------------

Proposed attribute::

        QOS_RULE_COMMON_FIELDS = {
                'id': {'allow_post': False, 'allow_put': False,
                       'validate': {'type:uuid': None},
                       'is_visible': True,
                       'primary_key': True},
                'tenant_id': {'allow_post': True, 'allow_put': False,
                              'required_by_policy': True,
                              'is_visible': True},
        }

        RESOURCE_ATTRIBUTE_MAP = {
            'policies': {
                'id': {'allow_post': False, 'allow_put': False,
                       'validate': {'type:uuid': None},
                       'is_visible': True,
                       'primary_key': True},
                'name': {'allow_post': True, 'allow_put': True,
                                'is_visible': True, 'default': '',
                                'validate': {'type:string': None}},
                'description': {'allow_post': True, 'allow_put': True,
                                'is_visible': True, 'default': '',
                                'validate': {'type:string': None}},
                'shared': {'allow_post': True, 'allow_put': True,
                           'is_visible': True, 'default': False,
                           'convert_to': attr.convert_to_boolean},
                'tenant_id': {'allow_post': True, 'allow_put': False,
                              'required_by_policy': True,
                              'is_visible': True},
                'rules': {'allow_post': False, 'allow_put': False,
                          'is_visible': True},
            },
            'rule_types': {
                 'type': {'allow_post': False, 'allow_put': False,
                          'is_visible': True},
            }
        }
        SUB_RESOURCE_ATTRIBUTE_MAP = {
           'bandwidth_limit_rules':{
                'parent': {'collection_name': 'policies',
                           'member_name': 'policy'},
                'parameters': {
                        dict(QOS_RULE_COMMON_FIELDS,
                             **{
                               'max_kbps': {'allow_post': True, 'allow_put': True,
                                            'is_visible': True, 'default': None,
                                            'validate': {'type:integer', None}},
                               'max_burst_kbps': {
                                            'allow_post': True, 'allow_put': True,
                                            'is_visible': True, 'default': 0,
                                            'validate': {'type:integer', None}},
                             })
                }
        }
        QOS = "qos_policy_id"

        EXTENDED_ATTRIBUTES_2_0 = {
            'ports': {QOS: {'allow_post': True,
                            'allow_put': True,
                            'is_visible': True,
                            'default': None,
                            'validate': {'type:uuid_or_none': None}}},
            'networks': {QOS: {'allow_post': True,
                               'allow_put': True,
                               'is_visible': True,
                               'default': None,
                               'validate': {'type:uuid_or_none': None}}},
        }

Sample request/responses:

Create Policy Request::

        POST /v2.0/qos/policies/
        {
            "policy": {
                "name": "10Mbit",
                "description": "This policy limits the ports to 10Mbit max.",
                "shared": "False"
             }
        }

        Response:
        {
           "policy": {
               "name": "10Mbit",
               "description": "This policy limits the ports to 10Mbit max.",
               "id": "46ebaec0-0570-43ac-82f6-60d2b03168c4",
               "tenant_id": "8d4c70a21fed4aeba121a1a429ba0d04",
               "shared": "False"
           }
        }


List available rule types::

        GET /v2.0/qos/rule-types

        Response:
        {
           "rule_types": [{"type": "bandwidth_limit"}]
        }


Create Rule Request::

        POST /v2.0/qos/policies/46ebaec0-0570-43ac-82f6-60d2b03168c4/bandwidth_limit_rules/
        {
            "bandwidth_limit_rule": {
            "max_kbps": "10000",
            }
        }


        Response:
        {
            "bandwidth_limit_rule":{
                "id": "5f126d84-551a-4dcf-bb01-0e9c0df0c793",
                "policy_id": "46ebaec0-0570-43ac-82f6-60d2b03168c4",
                "max_kbps": "10000",
                "max_burst_kbps": "0",
            }
        }


Show specific policy::

        GET /v2.0/qos/policies/46ebaec0-0570-43ac-82f6-60d2b03168c4
        Accept: application/json

        Response:
        {
            "policy": {
                "tenant_id": "8d4c70a21fed4aeba121a1a429ba0d04",
                "id": "46ebaec0-0570-43ac-82f6-60d2b03168c4",
                "name": "10Mbit",
                "description": "This policy limits the ports to 10Mbit max.",
                "shared": False,
                "bandwidth_limit_rules": [{
                    "id": "5f126d84-551a-4dcf-bb01-0e9c0df0c793",
                    "policy_id": "46ebaec0-0570-43ac-82f6-60d2b03168c4",
                    "max_kbps": "10000",
                    "max_burst_kbps": "0",
                }]
             }
        }


List Request::

        GET /v2.0/qos/policies

        Response:
        {
           “policies”:
               [
                    {
                       "tenant_id": "8d4c70a21fed4aeba121a1a429ba0d04",
                       "id": "46ebaec0-0570-43ac-82f6-60d2b03168c4",
                       "name": "10Mbit",
                       "description": "This policy limits the ports to 10Mbit max.",
                       "shared": False,
                       "bandwidth_limit_rules": [{
                          "id": "5f126d84-551a-4dcf-bb01-0e9c0df0c793",
                          "policy_id": "46ebaec0-0570-43ac-82f6-60d2b03168c4",
                          "max_kbps": "10000",
                          "max_burst_kbps": "0",
                       }]
                    },
                    {
                        ...
                    }
               ]
        }



Security Impact
---------------

By default QoS policies and rules will be managed by the cloud administrator,
that makes the tenant unable to create specific qos rules, or attaching
specific ports to policies.

In some use cases, like telcos, the administrator may trust the tenants, and
therefore let them create and attach their own policies to ports. Those use
cases would be supported by modification of policy.json and specific
documentation will be released with the extension.

This limitation could be partly overcome by the use of RBAC.


Notifications Impact
--------------------

None

Other End User Impact
---------------------

Additional methods will be added to python-neutronclient to create,
list, update, and delete QoS policies and rules.
Dedicated CLI command will be added to create and update each QoS rule type.

policy manipulation::

    neutron qos-policy-list
    neutron qos-policy-create  <policy-name> [--description policy-description]
                                [--shared True]
    neutron qos-policy-update  <policy-name-or-id> [--description ....]
                                [--name ...]

    neutron qos-policy-show    <policy-name-or-id>
    neutron qos-policy-delete  <policy-name-or-id>


policy rules manipulation::

    neutron qos-bandwidth-limit-rule-create <policy-name-or-id> \
                           --max_kbps x [--max_burst_kbps y]
    neutron qos-bandwidth-limit-rule-update <rule-id> <policy-name-or-id> \
                           --max_kbps x [--max_burst_kbps y]
    neutron qos-bandwidth-limit-rule-list   <policy-name-or-id>
    neutron qos-bandwidth-limit-rule-delete <rule-id> <policy-name-or-id>
    neutron qos-bandwidth-limit-rule-show   <rule-id> <policy-name-or-id>

    +-------------------+---------------------------------+
    | Field             | Value                           |
    +-------------------+---------------------------------+
    | id                | <rule-id>                       |
    | type              | bandwidth_limit                 |
    | description       | 10 Mbps limit                   |
    | max_kbps          | 10000                           |
    | max_burst_kbps    | 0                               |
    +-------------------+---------------------------------+

    neutron qos-available-rule-types

    +-----------------------+
    | QoS policy rule types |
    +-----------------------+
    | bandwidth_limit       |
    +-----------------------+


attach port/net to policy::

    neutron port-create NET-NAME-OR-ID --qos-policy <policy-name-or-id> ...
    neutron net-create NAME --qos-policy <policy-name-or-id> ....

    neutron port-update <port-id> --qos-policy <policy-name-or-id>
    neutron net-update <net-name-or-id> --qos-policy <policy-name-or-id>

NOTE: based on the initial implementation, regular tenants may be able to use
policies marked as ``Shared`` without any ``policy.json`` modification. Later
in time the RBAC implementation may allow more granular control.


detach port/net from policy::

    neutron port-update <port-id> --no-qos-policy


Performance Impact
------------------

* In some QoS drivers, additional messaging calls will be created that the
  L2 agents in the cluster will use to query QoS information when
  creating networks and ports. Although those message flows should be optimized
  to avoid scale issues, and we should look into common methods and messages
  to propagate this kind of information related to ports and network resources.



IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

* An Additional configuration section will be added to the Neutron
  plugin configuration, to configure a driver that implements the QoS
  API.

Developer Impact
----------------

None


Alternatives
------------

 - Doing QoS / traffic classification inside instances. This is limited to the
   most basic ones, since instances wouldn't be able to mark external
   segmentation packets to prioritize traffic at L2/L3 level.
   Also the tenants could not be trusted to do the right thing.

 - Nova flavors support for QoS [2]_ allows bandwidth limiting settings via
   the libvirt interface on the VM tap. This is enough for basic BW limiting
   on the VMs, but other QoS rules are not supported, and this also lacks
   support for service port QoS. User may need to stick to one approach or
   the other. This needs to be documented.


Community Impact
----------------

Community, specially telcos and operators has been looking for a way to
introduce QoS capabilities into neutron managed SDNs. For some use
cases prioritization and low jitter is fundamental to some types of
applications for example voice over IP, or video streaming.

Implementation
==============

Assignee(s)
-----------

* mangelajo
* gsagie
* scollins
* irenab
* vikram
* Moshe Levi
* Mathieu Rohon
* gampel

Work Items
----------

* REST API + API tests
* Database models & database migrations
* python-neutronclient support
* command line client implementation, bash completion included.
* openstack-sdk implementation

On other specs:

* RPC methods
* Driver model
* Driver implementation
* Reference OVS implementation [1]_

Dependencies
============

None

Testing
=======

Tempest Tests
-------------
None, since this is covered by the in-tree API tests.

Functional Tests
----------------

None, in the lower level specs functional testing will be used to
verify the low level reference implementation, and make sure effective
bandwidth limiting is performed.

API Tests
---------
The new api interface will be tested via API tests, to ensure all
the operations work as expected.

We should include tests to make sure incompatible rules are tested, with
the very basic bandwidth limiting we only need to look at not having
two different bandwidth limiting rules in one policy.

Documentation Impact
====================

User Documentation
------------------
* Additional documentation will be needed for deployers/operators, including
  alternatives to the default policy.json file provided in neutron.
* Additional documentation will be required for the REST API additions.
* Additional documentation will be required for the User Guide.

Developer Documentation
-----------------------

* Design documentation.
* Documentation about how to add new rule types.

References
==========
.. [1] ML2/OVS spec: https://review.openstack.org/#/c/182349/
.. [2] https://wiki.openstack.org/wiki/InstanceResourceQuota#Bandwidth_limits

Related Information
-------------------

-  https://review.openstack.org/#/c/132661/
-  https://lwn.net/Articles/640101/
-  http://specs.openstack.org/openstack/neutron-specs/specs/liberty/neutron-flavor-framework-templates.html
