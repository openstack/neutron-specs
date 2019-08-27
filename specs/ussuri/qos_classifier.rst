..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
Integration of Neutron Classifier and QoS DSCP
==============================================

https://bugs.launchpad.net/neutron/+bug/1476527

Neutron Classifier(NC) is a service plugin which provides an API to define
traffic classes for consumption by other Neutron services.

Quality of service(QoS) is an OpenStack Neutron extension which provides
administrators with a service to specify and enforce SLAs based on
drop priority, bandwidth policing and bandwidth guarantees at a port level.

By introducing the consumption of NC's Classification Group resources
into QoS's Rule resources we can allow for a more targeted form of SLA
allowing admins to limit, prioritize or guarantee traffic based on
traffic classes rather than on a Neutron Port level. This spec
will focus on drop priority based on traffic class per port.

Problem Description
===================

Neutron's Quality of Service extension allows for SLAs to be enforced at a
Neutron port level, however it does not provide a resource to allow
administrators to define SLAs for the different traffic classes entering or
leaving the port.

This results in an all or nothing approach to traffic class SLAs, either
all the traffic leaving a port recieves the same DSCP mark or the traffic
retains its default DSCP mark assigned by the workload.

Example Use Cases:

* Assigning a higher drop priority for traffic leaving TCP port YY
  than other traffic leaving the Neutron port.

* Overriding an existing DSCP mark assigned by the instance destined for
  IPv6 address XX with UDP port YY specifically.

Proposed Change
===============

In order to consume the classification resources of the Neutron Classifier
in a way that does not involve the introduction of external dependencies
in Neutron, the Neutron Classifier service plugin will require it to be
migrated into the core Neutron repository.

Details of the resource models and API can be found here 'NC API spec'_

The modifications required to Neutron's QoS extension is minimal,
it involves the introduction of a two variables to the DSCP Marking rule.
The first variable takes an ID for a classification group (a collection of
individual classifications) which the QoS backends can consume to target
DSCP Marks to traffic classes as opposed to marking all traffic leaving
the port.

The other, a priority value, will be a positive integer ranging from 0-1000
in ascending priority. This will allow admins to decide which DSCP marking
rules should be applied in a case where 2 DSPC rule classifications overlap.
eg::

    openstack network qos policy create example_policy

    openstack network qos rule create example_policy --type dscp-marking \
        --dscp-mark 22

    openstack network classification create ethernet --ethertype 0x0800 \
        eth_ipv4
    openstack network classification create ipv4 --protocol 6 \
        ipv4_tcp
    openstack network classification create tcp --dst-port-max 80 \
        tcp_80
    openstack network classification group create class_1 \
        --classification tcp_80 ipv4_tcp eth_ipv4

    openstack network qos rule create example_policy --type dscp-marking \
        --dscp-mark 28 --classification-group-id class_1 \
        --classification-priority 100

    openstack network classification create ipv4 --dst-addr 10.0.0.5 \
        ipv4_dst
    openstack network classification group create class_2 \
        --classification ipv4_dst eth_ipv4

    openstack network qos rule create example_policy --type dscp-marking \
        --dscp-mark 24 --classification-group-id class_2 \
        --classification-priority 600

    openstack network classification create ethernet --ethertype 0x86DD \
        eth_ipv6
    openstack network classification create ipv6 --next-header 17 \
        --dst-addr 0:0:0:0:0:ffff:a00:5 ipv6_dst
    openstack network classification create udp --dst-port-max 21 \
        udp_21
    openstack network classification group create class_3 \
        --classification eth_ipv6 ipv6_dst udp_21

    openstack network qos rule create example_policy --type dscp-marking \
        --dscp-mark 8 --classification-group-id class_3 \
        --classification-priority 158

    openstack network classification create ethernet --ethertype 0x0806 \
        eth_arp
    openstack network classification group create class_4 \
        --classification eth_arp

    openstack network qos rule create example_policy --type dscp-marking \
        --dscp-mark 40 --classification-group-id class_4 \
        --classification-priority 999

    openstack network qos rule list example_policy
    +--------------------------------------+--------------------------------------+--------------+----------+-----------------+----------+-----------+-----------+--------------------------------------+----------+
    | ID                                   | QoS Policy ID                        | Type         | Max Kbps | Max Burst Kbits | Min Kbps | DSCP mark | Direction | Classification Group                 | Priority |
    +--------------------------------------+--------------------------------------+--------------+----------+-----------------+----------+-----------+-----------+--------------------------------------+----------+
    | 153e4957-5c50-47f5-b934-c30eb3ada180 | 812c3e2e-cc86-4842-b41b-6b69419a123b | dscp_marking |          |                 |          |        40 |           | ae0de79d-c26a-4090-8ec2-0d1ae62797f2 |      999 |
    | 184d7156-ab93-444e-96eb-f91cf8c9160e | 812c3e2e-cc86-4842-b41b-6b69419a123b | dscp_marking |          |                 |          |         8 |           | b0fd0ca7-8596-4053-b69b-630016acfd28 |      158 |
    | 385970f8-9a6c-4fe2-a328-95630138c9ca | 812c3e2e-cc86-4842-b41b-6b69419a123b | dscp_marking |          |                 |          |        28 |           | 6adb4199-e42f-4fa9-8ab6-2bf64e2ea68d |      100 |
    | 7fa581ac-8b82-4d64-9032-52b1ce9f3509 | 812c3e2e-cc86-4842-b41b-6b69419a123b | dscp_marking |          |                 |          |        22 |           |                                      |        0 |
    | d1c5268b-302c-498b-bfd7-484e19fd28af | 812c3e2e-cc86-4842-b41b-6b69419a123b | dscp_marking |          |                 |          |        24 |           | cd0a1ea2-0cdf-42b0-a4fc-7f581d4a4be2 |      600 |
    +--------------------------------------+--------------------------------------+--------------+----------+-----------------+----------+-----------+-----------+--------------------------------------+----------+

Without a clear priority between the above classes there are instances where
classes class_1, class_3, class_4 and class_5 will also match class_2.
Depending on the backend this can result in unpredictible behavior without
a clear prioritization mechanism between traffic classes.

To accomodate these changes, modifications will need to be made in the
backend to allow drivers to retrieve the classification definitions from
the service plugin. This will be achieved through the extension of the
agent extensions API. By extending this API all extensions will have
access to classifications without having to introduce any additional
resources.

Data Model Impact
-----------------

In order to provide consistent functionality on upgrade the
classification_group_id defaults to None, meaning a DSCP marking rule
will behave the same way it did before the introduction of this feature.

The proposed change would result in the model looking like this:

+------------------------+-------+----------+--------+-----------------+-----------------+
|Attribute               |Type   |Access    |Default |Validation/      |Description      |
|Name                    |       |          |Value   |Conversion       |                 |
+========================+=======+==========+========+=================+=================+
|id                      |string |RO, all   |N/A     |uuid             |QoSRule          |
|                        |(UUID) |          |        |                 |identifier       |
+------------------------+-------+----------+--------+-----------------+-----------------+
|qos_policy_id           |string |RO, all   |N/A     |uuid             |QoSPolicy        |
|                        |(UUID) |          |        |                 |reference UUID   |
+------------------------+-------+----------+--------+-----------------+-----------------+
|dscp_mark               |integer|RW, tenant|N/A     |0 and 56, except |                 |
|                        |(enum) |          |        |2-6, 42, 44, and |DSCP Mark        |
|                        |       |          |        |50-54            |                 |
+------------------------+-------+----------+--------+-----------------+-----------------+
|classification_group_id |string |RW, tenant|None    |uuid             |Classification   |
|                        |(UUID) |          |        |                 |Group            |
|                        |       |          |        |                 |reference UUID   |
+------------------------+-------+----------+--------+-----------------+-----------------+
|classification_priority |integer|RW, tenant|0       |A positive       |Positive integer |
|                        |       |          |        |integer          |to denote prioity|
|                        |       |          |        |                 |of this rule     |
+------------------------+-------+----------+--------+-----------------+-----------------+

Another modification is the removal of the of the unique contraint on the
qos_policy_id. This has been replaced by a unique constraint on the
qos_policy_id and classification_group_id as a tuple.
This is to allow multiple DSCP rules with different classifications within
the same policy, this includes when the classification_group_id is None.
As a result, a QosPolicy can only have a single DSCPMarkingRule with
classification_group_id set to None.

Neutron DB table: QosDSCPMarkingRule

* id: primary key
* qos_policy_id: foreign key to QosPolicy.id
* dscp_mark: Integer within a defined set of values.
* classification_group_id: foreign key to ClassificationGroup.id
* classification_priority: positive integer between 0 and 1000

Before delving into further detail in regards to the data model, API and how
classifications can be used, a few points need to be clarified:

- 1 Classification is of a single type, e.g. either Ethernet, IP, HTTP,
  or another supported at the time of a specific CCF release. The definition,
  i.e. fields to match on, depends on the type specified.

- To clarify, Classification Types define the set of possible fields and values
  for a Classification (essentially, an instance of that Classification Type).
  Classification Types are defined in code, where Classifications are created
  via the REST API as instances of those types.

- Not all supported fields need to be defined - only the ones
  required by the Consuming Service - which it should validate on consumption.

- There are also Classification Groups, which allow Classifications or other
  Classification Groups to be grouped together using boolean operators. CGs
  are the resources that will end up being consumed by Consuming Services.

- From the Consuming Service's point of view, Classifications can only be read,
  not created or deleted. They need to have been previously
  created using the User-facing Classifications API.

The initial model of the CCF will include the following Classification Types:
Ethernet, IPv4, IPv6, TCP and UDP, which when combined are sufficient
to provide any 5-tuple classification.


The following table presents the attributes of a Classification Group
(asterisk on RW means that the attribute is non-updatable):

 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | Attribute            | Type    | Access | Default   | Validation/ | Description                 |
 | Name                 |         | CRUD   | Value     | Conversion  |                             |
 +======================+=========+========+===========+=============+=============================+
 | id                   | string  | RO,    | generated | uuid        | Identity                    |
 |                      | (UUID)  | all    |           |             |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | project_id           | string  | RO,    | from auth | uuid        | Project ID                  |
 |                      | (UUID)  | project| token     |             |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | name                 | string  | RW,    | None      | string      | Name of Classification Group|
 |                      |         | project|           |             |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | description          | string  | RW,    | None      | string      | Human-readable description  |
 |                      |         | project|           |             |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | shared               | bool    | RW,    | False     | boolean     | Shared with other projects  |
 |                      |         | project|           |             |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | operator             | string  | RW*,   | "and"     | ["and",     | Boolean connective: AND/OR  |
 |                      | (values)| project|           |  "or"]      |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | classification_groups| list    | RW*,   | []        |             | List of Classification      |
 |                      |         | project|           |             | Groups included             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | classifications      | list    | RW*    | []        |             | List of Classifications     |
 |                      |         | project|           |             | included                    |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+

Consuming Services will consume Classification Groups, and not atomic
Classifications, any Classification needs to be grouped in a Classification
Group to be consumed individually. As such, the "operator" field is to be
ignored for Classification Groups that only contain 1 Classification inside.

The following table presents the attributes of Classifications
of any of the types stated in this spec
(asterisk on RW means that the attribute is non-updatable):

 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | Attribute            | Type    | Access | Default   | Validation/ | Description                 |
 | Name                 |         |        | Value     | Conversion  |                             |
 +======================+=========+========+===========+=============+=============================+
 | id                   | string  | RO,    | generated | uuid        | Identity                    |
 |                      | (UUID)  | all    |           |             |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | project_id           | string  | RO,    | from auth | uuid        | Project ID                  |
 |                      | (UUID)  | project| token     |             |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | name                 | string  | RW,    | None      | string      | Name of Classification      |
 |                      |         | project|           |             |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | description          | string  | RW,    | None      | string      | Human-readable description  |
 |                      |         | project|           |             |                             |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | type                 | string  | RW*,   |           | from enum   | The type of the             |
 |                      |         | project|           | of types    | Classification              |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+
 | definition           | type-specific attributes will go here,                                   |
 |                      | given their volume I won't detail them unless requested.                 |
 +----------------------+---------+--------+-----------+-------------+-----------------------------+


Classification Groups and Classifications of every type will be stored as the
following tables and relationships (with table name prefix ``ccf_``)::

                           +---------------------+
                           |classification_groups|
                           +---------------------+
                           |id                   |*
                           |cg_id                +--------+
                           |name                 |        |
                           |description          |        |
                           |project_id           |        |
                           |shared               +--------+
                           |operator             |1
                           +---------------------+
                                       |1
                                       |
                                       |*
                       +------------------------------+
                       |classification_groups_mapping |
                       +------------------------------+
                       |cg_id                         |
                       |classification_id             |
                       +------------------------------+
                                       |1
 +--------------------+                |                +--------------------+
 |ipv4_classifications|                |                |ipv6_classifications|
 +--------------------+                |                +--------------------+
 |classification_id   |                |                |classification_id   |
 |ihl                 |1               |               1|traffic_class       |
 |diffserv            +--------+       |       +--------+traffic_class_mask  |
 |diffserv_mask       |        |       |       |        |length              |
 |length              |        |       |       |        |next_header         |
 |flags               |        |       |       |        |hops                |
 |flags_mask          |        |       |       |        |src_addr            |
 |ttl                 |        |1      |1     1|        |dst_addr            |
 |protocol            |     +---------------------+     +--------------------+
 |src_addr            |     |classifications      |
 |dst_addr            |     +---------------------+
 |options             |     |id                   |
 |options_mask        |     |name                 |
 +--------------------+     |description          |
                            |project_id           |
                            |shared               |     +-------------------+
                            |type                 |     |tcp_classifications|
                            +---------------------+     +-------------------+
                              1|      1|       |1       |classification_id  |
 +-------------------+         |       |       |        |src_port           |
 |udp_classifications|         |       |       |        |dst_port           |
 +-------------------+         |       |       |        |flags              |
 |classification_id  |1        |       |       |       1|flags_mask         |
 |src_port           +---------+       |       +--------+window             |
 |dst_port           |                1|                |data_offset        |
 |length             |     +------------------------+   |option_kind        |
 |window_size        |     |ethernet_classifications|   +-------------------+
 +-------------------+     +------------------------+
                           |classification_id       |
                           |preamble                |
                           |src_addr                |
                           |dst_addr                |
                           |ethertype               |
                           +------------------------+


Masking fields allow the user to specify which individual bits of the
respective main field should be looked up during classification.

Classification Types are used to select the appropriate model of the
Classification and consequently what table it will be stored in.

Classification Groups get stored in a single table and can point to other
Classification Groups, to allow mixing boolean operators.

``operator`` in Classification Group: specifies the boolean operator used
to connect all the child Classifications and Classification Groups of that
group. This can be either AND or OR.

OR: Logical OR, the classifications contained in these groups are
inter-changable within the traffic class ie.
ipv4 dst_addr y.y.y.y or ipv6 dst_addr Z::ZZZZ.

AND: logical AND, the classifications or classification groups are used
to create a traffic class, the composition of this classification can
be made up of Classifications or OR operand ClassificationGroups.


REST API Impact
---------------
Proposed attribute::

        SUB_RESOURCE_ATTRIBUTE_MAP = {
           'dscp_marking_rules':{
                'parent': {'collection_name': 'policies',
                           'member_name': 'policy'},
                'parameters': dict(QOS_RULE_COMMON_FIELDS,
                    **{'dscp_mark': {
                        'allow_post': True, 'allow_put': True,
                        'convert_to': attr.convert_to_int,
                        'is_visible': True, 'default': None,
                        'validate': {'type:values': common_constants.
                                    VALID_DSCP_MARKS}
                      },
                       'classification_group_id': {
                        'allow_post': True,
                        'allow_put': True,
                        'is_visible': True,
                        'default': None,
                        'validate': {'type:uuid_or_none': None}
                      },
                       'classification_priority': {
                        'allow_post': True,
                        'allow_put': True,
                        'convert_to': attr.convert_to_int,
                        'is_visible': True,
                        'default': 0,
                        'validate': {'type:values': range(1000)}
                }
        }

Sample REST calls::

        GET /v2.0/qos/policies

        Response:
        {
            "policy": {
                "name": "AF32",
                "project_id": "<project-id>",
                "id": "<id>",
                "description": "This policy marks DSCP outgoing AF32 traffic for DTV Control",
                "shared": "False"
             }
        }

        GET /v2.0/qos/policies/<policy-uuid>

        Response:
        {
            "policy": {
                "tenant_id": "<tenant-id>",
                "id": "<id>",
                "name": "AF32",
                "description": "This policy marks DSCP outgoing AF32 traffic for DTV Control",
                "shared": False,
                "dscp_marking_rules": [{
                    "id": "<id>",
                    "policy_id": "<policy-uuid>",
                    "dscp_mark": 16,
                    "classification_group_id": "<classification_group_id>",
                    "classification_priority": 800
                }]
             }
        }

        POST /v2.0/qos/policies/<policy-uuid>/dscp-marking-rules/
        {
            "dscp_marking_rule": {
                "dscp_mark": 16
            }
        }

        Response:
        {
            "dscp_marking_rule":{
                "id": "<id>",
                "policy_id": "<policy-uuid>",
                "dscp_mark": 16,
                "classification_group_id": None,
                "classification_priority": 0
            }
        }

        GET /v2.0/classification_groups/<classification_group_id>
        {
            "classification_group":{
                "id": "<classification_group_id>",
                "project_id": "<project_id>",
                "name": "ipv4-group"
                "description": "",
                "classification": [
                    {
                        "id":"<classification_id>",
                        "name":"ipv4-dscp"
                        "project_id":"<project_id>",
                        "c_type": "ipv4",
                        "description": "",
                        "shared":false,
                    }
                ],
                "classification_group": [],
                "operator": "AND",
                "shared": false,
            }
        }

        PUT /v2.0/qos/policies/<policy-uuid>/dscp-marking-rules/<rule-uuid>
        {
            "dscp_marking_rule": {
                "dscp_mark": 8,
                "classification_group_id": "<classification_group_id>"
            }
        }

        Response:
        {
            "dscp_marking_rule":{
                "id": "<id>",
                "policy_id": "<policy-uuid>",
                "dscp_mark": 8,
                "classification_group_id": "<classification_group_id>",
                "classification_priority": 0

            }
        }

Documentation Impact
====================

User Documentation
------------------

Existing `Networking Guide`_ will be updated for this feature.

Existing `CLI guide`_ will be updated for this feature.

Developer Documentation
-----------------------

Existing `QoS devref document`_ will be updated for this feature.
A New Document will be drafted for the Neutron Classifier API also.


API Documentation
-----------------

Existing `QoS API documentation`_ will be updated for this feature.
A New Document will be drafted for the Neutron Classifier API also.

References
==========

.. _'NC API spec': https://specs.openstack.org/openstack/neutron-specs/specs/pike/common-classification-framework.html
.. _`Networking Guide`: https://github.com/openstack/openstack-manuals/blob/master/doc/networking-guide/source/adv-config-qos.rs
.. _`CLI guide`: https://github.com/openstack/openstack-manuals/blob/master/doc/cli-reference/source/neutron.rst
.. _`QoS devref document`: https://github.com/openstack/neutron/blob/master/doc/source/devref/quality_of_service.rst
.. _`QoS API documentation`: https://docs.openstack.org/api-ref/network/v2/#quality-of-service
