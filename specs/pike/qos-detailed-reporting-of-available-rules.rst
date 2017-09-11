..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
QoS detailed reporting of available QoS rules
=============================================

https://bugs.launchpad.net/neutron/+bug/1686035

Problem Description
===================

With [1] different QoS rules can be supported by different backend drivers.
Whether the driver can handle the given rule type, and the parameters passed
with the rule, are validated only when the user wants to apply the rule to a
port/network.
With [2] merged, Neutron returns a list of all rule types supported by
at least one of the enabled backends.
This is not enough as users are unable to discover what values are supported by
the back ends for each rule type


Proposed Change
===============

We propose to add new API. This call would be used to get details of every
available rule type.

REST API
--------
New REST API call is::

    GET /v2.0/qos/rule-types/{rule_type}

Request example::

    GET /v2.0/qos/rule-types/bandwidth_limit

The response is a dict with details for the supported rule type for each of the
enabled backends.

As parameter values can be returned:

* list of supported values if driver uses "type:values" validator from
  neutron-lib to validate this parameter
* dict with 'start' and 'end' value if driver uses "type:range" validator from
  neutron-lib to validate this parameter. Both 'start' and 'end' values are
  inclusive in the range.

QoS rule types uses only "type:values" and "type:range" validators currently and
only those validators are supported by this spec.

This call should be available only for users with admin rights to not expose
details about cloud infra to regular users.

Response example::

    {
        "rule_type": {
            "type": "bandwidth_limit",
            "drivers": [
                {
                    "name": "ovs",
                    "supported_parameters": [
                        {
                            "parameter_name": "max_kbps",
                            "parameter_type": "range",
                            "parameter_values": {
                                "start": 0,
                                "end": 1000
                            }
                        },
                        {
                            "parameter_name": "max_burst_kbps",
                            "parameter_type": "range",
                            "parameter_values": {
                                "start": 0,
                                "end": 1000
                            }
                        },
                        {
                            "parameter_name": "direction",
                            "parameter_type": "choices",
                            "parameter_values": ["ingress", "egress"]
                        }
                    ]
                },
                {
                    "name": "linuxbridge",
                    "supported_parameters": [
                        {
                            "parameter_name": "max_kbps",
                            "parameter_type": "range",
                            "parameter_values": {
                                "start": 0,
                                "end": 1000
                            }
                        },
                        {
                            "parameter_name": "max_burst_kbps",
                            "parameter_type": "range",
                            "parameter_values": {
                                "start": 0,
                                "end": 1000
                            }
                        },
                        {
                            "parameter_name": "direction",
                            "parameter_type": "choices",
                            "parameter_values": ["egress"]
                        }
                    ]
                }
            ]
        }
    }


References
==========
[1] :doc:`qos-improved-validation-mechanism-rules`
[2] https://review.openstack.org/#/c/461257
