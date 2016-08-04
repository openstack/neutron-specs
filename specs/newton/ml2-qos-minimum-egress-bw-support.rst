..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
QoS minimum egress bandwidth support
====================================

RFE: https://bugs.launchpad.net/neutron/+bug/1560963

Problem Description
===================

The current `QoS API`_ does not provide functionality to define an assured
minimum egress bandwidth per port. This proposal talks about enhancing the
existing QoS API's by adding assured minimum egress bandwidth.

This functionality is currently supported in:

* Open vSwitch: minimum bandwidth assurance is supported, although the traffic
  shaping applies only to egress traffic, from the switch point of view. This
  egress traffic (from the switch point of view) should not be mistaken with
  the egress traffic from the tenant, which is the goal of this new feature.

* Linux Bridge: using traffic control `HTB`_ implementation.

* SR-IOV: it will depends on the NIC driver and the `ip-link`_ version, which
  needs to have "min_tx_rate" option; this option was merged in version 3.16.0.

Proposed Change
===============
The proposed change is an update to the QoS API and the drivers mentioned
(Open vSwitch, Linux Bridge and SR-IOV) to support minimum egress bandwidth
assurance. The value of the bandwidth will be stored as an integer value, with
valid values greater than zero. The unit is kbps, the same unit used in
``QosBandwidthLimitRule``.

The scope of this feature is to enable the API access to the egress minimum
bandwidth value and to enable this feature in the previous commented drivers.
This feature doesn't provide a method to inform about the backend capacity or
to check the bandwidth availability. These limitations will be documented in
user and developer guides.

Data Model Impact
-----------------
The model follows the way that the QoS Bandwidth Limiting functionality applies
to QoS policies by adding a QosMinimumBandwidthRule table.

The ``QosMinimumBandwidthRule`` model would look like::

  +--------------+-------+----------+--------+-----------------+------------+
  |Attribute     |Type   |Access    |Default |Validation/      |Description |
  |Name          |       |          |Value   |Conversion       |            |
  +==============+=======+==========+========+=================+============+
  |id            |string |RO, all   |generat_|uuid             |identity    |
  |              |(UUID) |          |ed      |                 |            |
  +--------------+-------+----------+--------+-----------------+------------+
  |qos_policy_id |string |RO, all   |N/A     |uuid             |QoSPolicy   |
  |              |(UUID) |          |        |                 |reference   |
  +--------------+-------+----------+--------+-----------------+------------+
  |min_kbps      |integer|RW, tenant|N/A     |not negative,    |Minimum     |
  |              |       |          |        |<= max_kbps      |bandwidth   |
  +--------------+-------+----------+--------+-----------------+------------+
  |direction     |enum   |RW, tenant|'egress'|'egress'         |Traffic     |
  |              |       |          |        |                 |direction   |
  +--------------+-------+----------+--------+-----------------+------------+

If "max_kbps" is None, the validation check for "min_kbps" will not apply.

This QoS rule can be used in combination with QoS bandwidth limit rule, defined
in `Neutron QoS API Models and Extension`_.

Both "qos_policy_id" and "direction" will be linked as a unique constraint set,
because the combination of both parameters must be unique.

This QoS rule will have a "direction" parameter to define the direction of the
traffic. This new feature will implement only egress traffic shaping. "ingress"
traffic direction won't be allowed for the nonce.

REST API Impact
---------------
Proposed attribute::

        SUB_RESOURCE_ATTRIBUTE_MAP = {
           'minimum_bandwidth_rules':{
                'parent': {'collection_name': 'policies',
                           'member_name': 'policy'},
                'parameters': dict(QOS_RULE_COMMON_FIELDS,
                    **{'min_kbps': {
                        'allow_post': True, 'allow_put': True,
                        'convert_to': attr.convert_to_int,
                        'is_visible': True, 'default': None,
                        'validate': {'type:non_negative': None,
                                     'type:values': max_kbps}}
                      }),
                    **{'direction': {
                        'allow_post': True, 'allow_put': True,
                        'is_visible': True, 'default': 'egress',
                        'validate': {'type:values': common_constants.
                                     EGRESS_DIRECTION}}
                      }),
                }
        }


Sample REST calls::

        GET /v2.0/qos/policies

        Response:
        {
            "policy": {
                "name": "TypeOne",
                "description": "This policy sets a minimum bandwidth for type one users",
                "shared": "False"
             }
        }

        GET /v2.0/qos/policies/<policy-uuid>

        Response:
        {
            "policy": {
                "tenant_id": "<tenant-id>",
                "id": "<id>",
                "name": "TypeOne",
                "description": "This policy sets a minimum bandwidth for type one users",
                "shared": False,
                "rules": [{
                    "id": "<id>",
                    "policy_id": "<policy-uuid>",
                    "rule_type": neutron.services.qos.RULE_TYPE_MINIMUM_BANDWIDTH
                    "min_kbps": 10000,
                    "direction": "egress"
                }]
             }
        }

        POST /v2.0/qos/policies/<policy-uuid>/minimum_bandwidth_rules/
        {
            "minimum_bandwidth_rule": {
                "min_kbps": 20000,
                "direction": "egress"
            }
        }

        Response:
        {
            "minimum_bandwidth_rule":{
                "id": "<id>",
                "policy_id": "<policy-uuid>",
                "min_kbps": 20000,
                "direction": "egress"
            }
        }

        PUT /v2.0/qos/policies/<policy-uuid>/minimum_bandwidth_rules/<rule-uuid>
        {
            "minimum_bandwidth_rule": {
                "min_kbps": 10000,
                "direction": "egress"
            }
        }

        Response:
        {
            "minimum_bandwidth_rule":{
                "id": "<id>",
                "policy_id": "<policy-uuid>",
                "min_egress_kbps": 10000,
                "direction": "egress"
            }
        }

Command Line Client Impact
--------------------------

* qos-minimum-bandwidth-rule-create <policy-id> --min-kbps <value> --direction <value>
* qos-minimum-bandwidth-rule-show <rule-id> <policy-id>
* qos-minimum-bandwidth-rule-list <policy-id>
* qos-minimum-bandwidth-rule-update <rule-id> <policy-id> --min-kbps <value>
  --direction <value>
* qos-minimum-bandwidth-rule-delete <rule-id> <policy-id>

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

None

Implementation
==============

Assignee(s)
-----------

* Rodolfo Alonso Hernandez
* Miguel Angel Ajo
* Hirofumi Ichihara
* Moshe Levi

Work Items
----------

* Versioned DB objects for the new rule type

* API changes to allow for minimum egress bandwidth modifications

* Client changes to allow minimum egress bandwidth values being set

* QoS Openflow integration within the L2 agent extension for the Open vSwitch
  driver to enable the minimum egress bandwidth support:

  * Added/modified switch flows to mark the packet processing queue.

  * Added/modified QoS rules and processing queues in the integration bridge,
    physical bridges and tunnel bridge.

* QoS Traffic Control integration within the L2 agent extension for the Linux
  Bridge driver to enable this feature.

* QoS ip-link integration within the L2 agent extension for the SR-IOV driver
  to enable this feature. Add a sanity check to detect the version of ip-link
  tool; minimum version required is 3.16.0.

Open vSwitch driver
^^^^^^^^^^^^^^^^^^^
The current Open vSwitch QoS implementation only shapes egress traffic.

Linux Bridge driver
^^^^^^^^^^^^^^^^^^^
The current Linux Bridge QoS implementation using Traffic Control (TC) only
shapes egress traffic (from the interface point of view). To shape ingress
traffic from the interface point of view (egress from the instance), what is
needed in this feature, the following steps are required:

* Create a `IFB`_.

* Send all ingress traffic to this IFB, mirroring it, using a TC rule. This
  traffic will be sent to this IFB as egress traffic.

* Add a qdisc and a class in this IFB, with the QoS filter.

Also the algorithm used, `TBF`_, must be substituted by `HTB`_, a classful
algorithm that allows to set a "rate" parameter, defined as the "maximum rate
this class and all its children are guaranteed".

Future work
-----------

* Implement a method to report backend capacity to Neutron. Several methods to
  enable this feature can be carried out:

  * Use an agent to periodically read the capacity of the backends, using
    config files, monitoring systems, etc.
    The agent state reports could be used for that purpose. The information
    could come from system inspection also enabling an option to override the
    details in the agent config file.
    The bandwidth mapping should be per physical network, and there's a
    peculiarity we must consider: connection to physical networks can happen
    by several unbound interfaces (like an SR-IOV card having several PFs
    (with it's separate cables) to the switch. That would effectively give
    different connection resources to the same net which are exhausted
    separately.
    This is the best option.

  * Insert the backend capacity manually, using a static file during the
    deployment process.

  * Enable an API to manually insert the backend capacity. The API can be used
    for testing and development purposes, but this option should be avoided.

* Inform Nova Scheduler about the total backend capacity and the QoS minimum
  egress bandwidth rules, to improve the Scheduler decision. This feature is
  being addressed by `[RFE] Strict minimum bandwidth support (egress)`_.

Dependencies
============

None

Testing
=======

API-tests
---------

* Creating minimum egress bandwidth values
* Updating minimum egress bandwidth values
* Deleting minimum egress bandwidth values
* Listing minimum egress bandwidth values
* Showing a minimum egress bandwidth value

Functional Tests
----------------

Functional tests will be used to verify system interactions:

* Setting minimun egress bandwidth values
* Updating minimun egress bandwidth values
* Deleting minimun egress bandwidth values
* Listing minimun egress bandwidth values

Fullstack Tests
---------------

* Setting a QoS policy for minimum egress bandwidth on the port of the first
  instance from the API, making a query to the backend and check the
  correctness of the stored values.
* Updating QoS policy and checking again the values.
* Deleting QoS policy and verifying all the values related to the rule are
  deleted.

These are no benchmark tests.

Documentation Impact
====================

User Documentation
------------------

Existing `Networking Guide`_ will be updated for this feature.

Existing `CLI guide`_ will be updated for this feature.

Developer Documentation
-----------------------

Existing `QoS devref document`_ will be updated for this feature.


API Documentation
-----------------

Existing `QoS API documentation`_ will be updated for this feature.


References
==========
.. target-notes::

.. _`QoS API`: https://review.openstack.org/#/c/88599/
.. _`HTB`: http://linux.die.net/man/8/tc-htb
.. _`ip-link`: http://manpages.ubuntu.com/manpages/xenial/en/man8/ip-link.8.html
.. _`Neutron QoS API Models and Extension`: http://specs.openstack.org/openstack/neutron-specs/specs/liberty/qos-api-extension.html
.. _`IFB`: http://www.linuxfoundation.org/collaborate/workgroups/networking/ifb
.. _`TBF`: http://linux.die.net/man/8/tc-tbf
.. _`[RFE] Strict minimum bandwidth support (egress)`: https://bugs.launchpad.net/neutron/+bug/1578989
.. _`Networking Guide`: https://github.com/openstack/openstack-manuals/blob/master/doc/networking-guide/source/adv-config-qos.rst
.. _`CLI guide`: https://github.com/openstack/openstack-manuals/blob/master/doc/cli-reference/source/neutron.rst
.. _`QoS devref document`: https://github.com/openstack/neutron/blob/master/doc/source/devref/quality_of_service.rst
.. _`QoS API documentation`: https://review.openstack.org/#/c/226834/
