..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
QoS DSCP marking support
=============================

RFE: https://bugs.launchpad.net/neutron/+bug/1468353

Problem Description
===================

The current `QoS API <https://review.openstack.org/#/c/88599/>`_ does not
provide functionality to mark outgoing network traffic with a DSCP value. This
proposal talks about enhancing the existing QoS API's by adding DSCP marking
support. The functionality will make use of the Open vSwitch support for adding
DSCP marks to the outbound network traffic.

Proposed Change
===============

We propose an update to the QoS API and OVS driver to support DSCP marks.
Valid DSCP mark values can be between 0 and 56, except 2-6, 42, 44, and 50-54.

This will serve as the reference implementation for how to use the QoS API to
manage DSCP marks using the OVS driver.

Data Model Impact
-----------------

The model follows the way that the QoS Bandwidth Limiting functionality applies
to QoS policies by adding a QosDscpMarkingRule table.

The QosDscpMarkingRule model would look like:

+--------------+-------+----------+--------+-----------------+------------+
|Attribute     |Type   |Access    |Default |Validation/      |Description |
|Name          |       |          |Value   |Conversion       |            |
+==============+=======+==========+========+=================+============+
|qos_policy_id |string |RO, all   |N/A     |uuid             | QoSPolicy  |
|              |(UUID) |          |        |                 | reference  |
+--------------+-------+----------+--------+-----------------+------------+
|dscp_mark     |integer|RW, tenant|N/A     |0 and 56, except |            |
|              |       |          |        |2-6, 42, 44, and | DSCP Mark  |
|              |       |          |        |50-54            |            |
+--------------+-------+----------+--------+-----------------+------------+

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
                                    VALID_DSCP_MARKS}}})
                      })
                }
        }


Sample REST calls::

        GET /v2.0/qos/policies

        Response:
        {
            "policy": {
                "name": "AF32",
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
                    "dscp_mark": 16
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
                "dscp_mark": 16
            }
        }

        PUT /v2.0/qos/policies/<policy-uuid>/dscp-marking-rules/<rule-uuid>
        {
            "dscp_marking_rule": {
                "dscp_mark": 8
            }
        }

        Response:
        {
            "dscp_marking_rule":{
                "id": "<id>",
                "policy_id": "<policy-uuid>",
                "dscp_mark": 8
            }
        }

Command Line Client Impact
--------------------------

* qos-dscp-marking-rule-create <policy-id> --dscp_mark <value>
* qos-dscp-marking-rule-show <mark-rule-id> <policy-id>
* qos-dscp-marking-rule-list <policy-id>
* qos-dscp-marking-rule-update <mark-rule-id> <policy-id> --dscp_mark <value>
* qos-dscp-marking-rule-delete <mark-rule-id> <policy-id>

Security Impact
---------------

None

Notifications Impact
--------------------

None

Performance Impact
------------------

None

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

Deployers may need to configure the specific QoS driver / ML2 agent extension.

Developer Impact
----------------

None

Community Impact
----------------

The ability to set DSCP marks on QoS policies on ports or networks using OVS.

Implementation
==============

Assignee(s)
-----------

* victor-r-howard
* nate-johnston
* james-reeves5546
* margaret-frances

Work Items
----------

* Versioned DB objects for the new rule type
* API changes to allow for DSCP API modifications
* Client changes to allow for DSCP values being set
* Openflow integration within OVS driver to add qos_dscp marking functionality

Dependencies
============


API-tests
---------

* Creating DSCP values
* Updating DSCP values
* Deleting DSCP values
* Listing DSCP values
* Showing a DSCP Value

Functional Tests
----------------

Functional tests will be used to verify system interactions:

* Setting DSCP values
* Updating DSCP values
* Deleting DSCP values
* Listing DSCP values
* Ensure traffic is using DSCP marks outbound

Fullstack Tests
---------------

* Setting a QoS policy for marking on a port from API, inspecting that the low-level system bits are set to do DSCP correctly
* Updating QoS policy, and checking the low bits (DSCP mark bits)
* Deleting QoS policy and verifying all the DSCP rules are deleted properly or not

Documentation Impact
====================

User Documentation
------------------

Existing `Networking Guide <https://github.com/openstack/openstack-manuals/blob/master/doc/networking-guide/source/adv-config-qos.rst>`_
will be updated for this feature.

Existing `CLI guide <https://github.com/openstack/openstack-manuals/blob/master/doc/cli-reference/source/neutron.rst>`_
will be updated for this feature.

Developer Documentation
-----------------------

Existing 'QoS devref document
<https://github.com/openstack/neutron/blob/master/doc/source/devref/quality_of_service.rst>`_
will be updated for this feature.


API Documentation
-----------------

We will update `the QoS API documentation
<https://review.openstack.org/#/c/226834/>`_ will be updated for this feature.


References
==========

.. [#qos_api_spec] https://review.openstack.org/#/c/88599/
.. [#qos_devref] https://github.com/openstack/neutron/blob/master/doc/source/devref/quality_of_service.rst
.. [#qos_api_doc] https://review.openstack.org/#/c/226834/
