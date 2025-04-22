..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================================
Add more information to network-ip-availabilites response
=========================================================

https://bugs.launchpad.net/neutron/+bug/2107316

This RFE intends to provide detailed information on the number of IPs in the
subnet from the response of GET ``/v2.0/network-ip-availabilites`` - the
number of available IPs in the subnet cidr and in the allocation pools, used
IPs in the subnet cidr and in the allocation pools.


Problem Description
===================

Now the response of network-ip-availabilites api contains:
(1) total_ips
(2) used_ips

(1) total_ips
- subnet cidr, when there are no allocation pools in the subnet
- the sum of IPs in each allocation pool, when there are allocation pools in
the subnet

(2) used_ips
- the number of used IPs in the subnet (does not consider allocation pools)

We cannot know how many available IPs are in the allocation pools from the
response. There are use cases that we need to know the number of available IPs
in the allocation pools, such as check ip availability before creating a vm in
the network.

Using IPv4 subnet, total_ips does not show the exact number of available IPs.
total_ips includes .0 and .255, which IPs can't be used as ports' ip.


Proposed Change
===============

To solve the problem described above, the proposal is to add a new API
extension ``network-ip-availability-details`` to extend
``network-ip-availability`` resource in Neutron with
``ip_availability_details`` attribute.

Server side changes
-------------------

A new API extension ``network-ip-availability-details`` will be added. The
extension adds a new attribute ``ip_availability_details``.

The attribute consists of:
(1) total_ips_in_subnet: the number of available IPs in the subnet cidr
(except .0, .255, in IPv4 subnet)
(2) total_ips_in_allocation_pool: the sum of IPs in each allocation pool, 0 in
case of no allocation pools
(3) used_ips_in_subnet: the sum of used IPs in the subnet (does not consider
allocation pools)
(4) used_ips_in_allocation_pool: the sum of used IPs in each allocation pool,
0 in case of no allocation pools

Response of GET ``/v2.0/network-ip-availabilities/{network_id}``::

    {
        "network_ip_availability": {
            "subnet_ip_availability": [
                {
                    "subnet_id": "44e70d00-80a2-4fb1-ab59-6190595ceb61",
                    "subnet_name": "private-subnet",
                    "ip_version": 4,
                    "cidr": "10.0.0.0/24",
                    "total_ips": 256,
                    "used_ips": 2,
                    "ip_availability_details": {
                      "total_ips_in_subnet": 254,
                      "total_ips_in_allocation_pool": 0,
                      "used_ips_in_subnet": 2,
                      "used_ips_in_allocation_pool": 0
                    }
                },
                {
                    "subnet_id": "a90623df-00e1-4902-a675-40674385d74c",
                    "subnet_name": "ipv6-private-subnet",
                    "ip_version": 6,
                    "cidr": "fdbf:ac66:9be8::/64",
                    "total_ips": 18446744073709551600,
                    "used_ips": 2,
                    "ip_availability_details": {
                      "total_ips_in_subnet": 18446744073709551616,
                      "total_ips_in_allocation_pool": 18446744073709551600,
                      "used_ips_in_subnet": 2,
                      "used_ips_in_allocation_pool": 1
                    }
                }
            ],
            "network_id": "6801d9c8-20e6-4b27-945d-62499f00002e",
            "project_id": "d56d3b8dd6894a508cf41b96b522328c",
            "tenant_id": "d56d3b8dd6894a508cf41b96b522328c",
            "network_name": "private",
            "total_ips": 18446744073709551856,
            "used_ips": 4,
            "ip_availability_details": {
              "total_ips_in_subnet": 18446744073709551870,
              "total_ips_in_allocation_pool": 18446744073709551600,
              "used_ips_in_subnet": 4,
              "used_ips_in_allocation_pool": 1
            }
        }
    }

Response of GET ``/v2.0/network-ip-availabilities``::

    {
        "network_ip_availabilities": [
            {
                "network_id": "4cf895c9-c3d1-489e-b02e-59b5c8976809",
                "network_name": "public",
                "subnet_ip_availability": [
                    {
                        "subnet_id": "44e70d00-80a2-4fb1-ab59-6190595ceb61",
                        "subnet_name": "private-subnet",
                        "ip_version": 4,
                        "cidr": "10.0.0.0/24",
                        "total_ips": 256,
                        "used_ips": 2,
                        "ip_availability_details": {
                          "total_ips_in_subnet": 254,
                          "total_ips_in_allocation_pool": 0,
                          "used_ips_in_subnet": 2,
                          "used_ips_in_allocation_pool": 0
                        }
                    },
                    {
                        "subnet_id": "a90623df-00e1-4902-a675-40674385d74c",
                        "subnet_name": "ipv6-private-subnet",
                        "ip_version": 6,
                        "cidr": "fdbf:ac66:9be8::/64",
                        "total_ips": 18446744073709551600,
                        "used_ips": 2,
                        "ip_availability_details": {
                          "total_ips_in_subnet": 18446744073709551616,
                          "total_ips_in_allocation_pool": 18446744073709551600,
                          "used_ips_in_subnet": 2,
                          "used_ips_in_allocation_pool": 1
                        }
                    }
                ],
                "project_id": "1a02cc95f1734fcc9d3c753818f03002",
                "tenant_id": "1a02cc95f1734fcc9d3c753818f03002",
                "total_ips": 18446744073709551856,
                "used_ips": 4,
                "ip_availability_details": {
                  "total_ips_in_subnet": 18446744073709551870,
                  "total_ips_in_allocation_pool": 18446744073709551600,
                  "used_ips_in_subnet": 4,
                  "used_ips_in_allocation_pool": 1
                }
            },
            {
                "network_id": "6801d9c8-20e6-4b27-945d-62499f00002e",
                "network_name": "private",
                "subnet_ip_availability": [
                    {
                        "cidr": "10.0.0.0/24",
                        "ip_version": 4,
                        "subnet_id": "44e70d00-80a2-4fb1-ab59-6190595ceb61",
                        "subnet_name": "private-subnet",
                        "total_ips": 256,
                        "used_ips": 2,
                        "ip_availability_details": {
                          "total_ips_in_subnet": 254,
                          "total_ips_in_allocation_pool": 0,
                          "used_ips_in_subnet": 2,
                          "used_ips_in_allocation_pool": 0
                        }
                    },
                    {
                        "ip_version": 6,
                        "cidr": "fdbf:ac66:9be8::/64",
                        "subnet_id": "a90623df-00e1-4902-a675-40674385d74c",
                        "subnet_name": "ipv6-private-subnet",
                        "total_ips": 18446744073709551600,
                        "used_ips": 2,
                        "ip_availability_details": {
                          "total_ips_in_subnet": 18446744073709551614,
                          "total_ips_in_allocation_pool": 18446744073709551600,
                          "used_ips_in_subnet": 2,
                          "used_ips_in_allocation_pool": 2
                        }
                    }
                ],
                "project_id": "d56d3b8dd6894a508cf41b96b522328c",
                "total_ips": 18446744073709551856,
                "used_ips": 4,
                "ip_availability_details": {
                  "total_ips_in_subnet": 18446744073709551868,
                  "total_ips_in_allocation_pool": 18446744073709551600,
                  "used_ips_in_subnet": 4,
                  "used_ips_in_allocation_pool": 2
                }
            }
        ]
    }

DB Impact
---------

None

REST API Impact
---------------

New API extension: ``network-ip-availability-details`` introducing new
attribute ``ip_availability_details``.

Client Impact
-------------

Relevant changes in openstackclient and openstacksdk to add support for new
attribute ``ip_availability_details``.

Implementation
==============

Add new API extension ``network-ip-availability-details``. This extension
extends network ip availability with field ``ip_availability_details``
consists of attributes - total_ips_in_subnet, total_ips_in_allocation_pool,
used_ips_in_subnet, and used_ips_in_allocation_pool.

New class ``IpAvailabilityDetailsMixin`` will be added for the extension.
The class calculates value of the four attributes, and extends network ip
availability.

Assignee(s)
-----------

* jimin3-shin <jimin3.shin@samsung.com>

Testing
=======

* New test classes and methods will be added to test
  ``network-ip-availability-details`` extension.

* Tempest tests in neutron-tempest-plugin.

References
==========


