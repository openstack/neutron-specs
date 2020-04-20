..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
Address Groups Support in Security Group Rules
==============================================

https://bugs.launchpad.net/neutron/+bug/1592028

This specification describes how to support address groups (groups of IP
address blocks) in Neutron security group rules. The concept of address groups
was introduced in FWaaS v2.0 but not implemented [1].

Problem Description
===================

Neutron security group rules currently support using an IP address block or a
security group as the remote end of the network access rule. In actual usage,
an OpenStack cloud may require connectivity between instances and external
services which are not provisioned by OpenStack. And each service may also
have multiple endpoint addresses which are not contiguous. To allow the
connectivity, one rule per external address needs to be created which can be
very cumbersome and difficult to maintain as the number of external addresses
may be substantial.


Proposed Change
===============

Add an API to aggregate IP address blocks into an address group object which
could be later referenced when creating a security group rule. Thus the number
of security group rules can be effectively reduced when there is the need to
allow connectivity to a number of external service addresses. When IP addresses
are updated in the address group, changes will also be reflected in the
associated security group rules.


REST API Impact
---------------

New address groups API
~~~~~~~~~~~~~~~~~~~~~~

+-------------------+---------+-------+------+---------------------------------------+
| Attribute         | Type    | Req   | CRUD | Description                           |
+===================+=========+=======+======+=======================================+
| id                | uuid-str| N/A   | R    | Unique identifier for the             |
|                   |         |       |      | address_group object.                 |
+-------------------+---------+-------+------+---------------------------------------+
| name              | String  | No    | CRU  | Human readable name for the address   |
|                   |         |       |      | group (255 characters limit). Does not|
|                   |         |       |      | have to be unique.                    |
+-------------------+---------+-------+------+---------------------------------------+
| description       | String  | No    | CRU  | Human readable description for the    |
|                   |         |       |      | address group (255 characters limit). |
+-------------------+---------+-------+------+---------------------------------------+
| project_id        | uuid-str| No    | CR   | Owner of the address group. Only      |
|                   |         |       |      | admin users can specify a project     |
|                   |         |       |      | identifier other than their own.      |
+-------------------+---------+-------+------+---------------------------------------+
| addresses         | List    | Yes   | CRU  | Array of address. It supports both    |
|                   |         |       |      | CIDR and IP range objects.            |
|                   |         |       |      | An example of addresses:              |
|                   |         |       |      | ["132.168.4.12/24",                   |
|                   |         |       |      | "132.168.5.12-132.168.5.24",          |
|                   |         |       |      | "2001::db8::f00/64"]                  |
+-------------------+---------+-------+------+---------------------------------------+

|

List address groups
^^^^^^^^^^^^^^^^^^^

Lists address groups.

    +----------------+------------------------------------------------+
    | Request Type   | ``GET``                                        |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/v2.0/address-groups``                       |
    +----------------+---------+--------------------------------------+
    |                | Success | 200                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401)                    |
    +----------------+---------+--------------------------------------+

|

**Example List address groups: JSON request**

.. code::

    GET /v2.0/address-groups
    User-Agent: python-neutronclient
    Accept: application/json

**Example List address groups: JSON response**


.. code::

    {
        "address_groups": [
            {
                "description": "",
                "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
                "name": "ADDR_GP_1",
                "project_id": "45977fa2dbd7482098dd68d0d8970117",
                "addresses": [
                   "132.168.4.12/24",
                   "132.168.5.12-132.168.5.24",
                   "2001::db8::f00/64"
                ]
            }
        ]
    }

Show address group details
^^^^^^^^^^^^^^^^^^^^^^^^^^

Shows address group details.

    +----------------+----------------------------------------------------+
    | Request Type   | ``GET``                                            |
    +----------------+----------------------------------------------------+
    | Endpoint       | ``/v2.0/address-groups/<address_group_id>``        |
    +----------------+---------+------------------------------------------+
    |                | Success | 200                                      |
    | Response Codes +---------+------------------------------------------+
    |                | Error   | Unauthorized(401), Not Found (404)       |
    +----------------+---------+------------------------------------------+

|

**Example Show address group: JSON request**

.. code::

    GET /v2.0/address-groups/8722e0e0-9cc9-4490-9660-8c9a5732fbb0
    User-Agent: python-neutronclient
    Accept: application/json


**Example Show address group: JSON response**

.. code::

    {
       "address_group": {
            "description": "",
            "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
            "name": "ADDR_GP_1",
            "project_id": "45977fa2dbd7482098dd68d0d8970117",
            "addresses": [
               "132.168.4.12/24",
               "132.168.5.12-132.168.5.24",
               "2001::db8::f00/64"
            ]
        }
    }



Create address group
^^^^^^^^^^^^^^^^^^^^

Creates an address group.

    +----------------+------------------------------------------------+
    | Request Type   | ``POST``                                       |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/v2.0/address-groups/``                      |
    +----------------+---------+--------------------------------------+
    |                | Success | 201                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Bad Request(400)  |
    +----------------+---------+--------------------------------------+

|

**Example Create address group: JSON request**

.. code::

    POST /v2.0/address-groups
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "address_group": {
            "name": "ADDR_GP_1",
            "addresses": [
               "132.168.4.12/24",
               "132.168.5.12-132.168.5.24",
               "2001::db8::f00/64"
            ]
        }
    }

**Example Create address group: JSON response**

.. code::

    HTTP/1.1 201 Created
    Content-Type: application/json; charset=UTF-8

.. code::

    {
       "address_group": {
            "description": "",
            "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
            "name": "ADDR_GP_1",
            "project_id": "45977fa2dbd7482098dd68d0d8970117",
            "addresses": [
               "132.168.4.12/24",
               "132.168.5.12-132.168.5.24",
               "2001::db8::f00/64"
            ]
        }
    }


Update address group
^^^^^^^^^^^^^^^^^^^^

Updates an address group. To update addresses, use the add addresses and
remove addresses operations.

    +----------------+----------------------------------------------------+
    | Request Type   | ``PUT``                                            |
    +----------------+----------------------------------------------------+
    | Endpoint       | ``/v2.0/address-groups/<address_group_id>``        |
    +----------------+---------+------------------------------------------+
    |                | Success | 200                                      |
    | Response Codes +---------+------------------------------------------+
    |                | Error   | Unauthorized(401), Bad Request(400) \    |
    |                |         | Not Found(404)                           |
    +----------------+---------+------------------------------------------+

|

**Example Update address group: JSON request**

.. code::

    PUT /v2.0/address-groups/8722e0e0-9cc9-4490-9660-8c9a5732fbb0
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "address_group": {
            "description": "new description",
            "name": "new name"
        }
    }


**Example Update address group: JSON response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
       "address_group": {
            "description": "new description",
            "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
            "name": "new name",
            "project_id": "45977fa2dbd7482098dd68d0d8970117",
            "addresses": [
               "132.168.4.12/24",
               "132.168.5.12-132.168.5.24",
               "2001::db8::f00/64"
            ]
        }
    }


Add addresses to address group
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Add addresses to an existing address group.

    +----------------+------------------------------------------------------------------+
    | Request Type   | ``PUT``                                                          |
    +----------------+------------------------------------------------------------------+
    | Endpoint       | ``/v2.0/address-groups/<address_group_id>/add_addresses``        |
    +----------------+---------+--------------------------------------------------------+
    |                | Success | 200                                                    |
    | Response Codes +---------+--------------------------------------------------------+
    |                | Error   | Unauthorized(401), Bad Request(400), Not Found(404)    |
    +----------------+---------+--------------------------------------------------------+

|

**Example Update addresses: JSON request**

.. code::

    PUT /v2.0/address-groups/8722e0e0-9cc9-4490-9660-8c9a5732fbb0/add_addresses
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "addresses": [
           "10.0.0.1/32",
           "2001:3889:120:fe42::/64"
        ]
    }


**Example Update addresses: JSON response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
       "address_group": {
            "description": "",
            "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
            "name": "ADDR_GP_1",
            "project_id": "45977fa2dbd7482098dd68d0d8970117",
            "addresses": [
               "132.168.4.12/24",
               "132.168.5.12-132.168.5.24",
               "10.0.0.1/32",
               "2001::db8::f00/64",
               "2001:3889:120:fe42::/64"
            ]
        }
    }

Remove addresses from address group
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Remove addresses from an existing address group.

    +----------------+------------------------------------------------------------------+
    | Request Type   | ``PUT``                                                          |
    +----------------+------------------------------------------------------------------+
    | Endpoint       | ``/v2.0/address-groups/<address_group_id>/remove_addresses``     |
    +----------------+---------+--------------------------------------------------------+
    |                | Success | 200                                                    |
    | Response Codes +---------+--------------------------------------------------------+
    |                | Error   | Unauthorized(401), Bad Request(400), Not Found(404)    |
    +----------------+---------+--------------------------------------------------------+

|

**Example Remove addresses: JSON request**

.. code::

    PUT /v2.0/address-groups/8722e0e0-9cc9-4490-9660-8c9a5732fbb0/remove_addresses
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "addresses": [
           "132.168.4.12/24",
           "2001::db8::f00/64"
        ]
    }


**Example Remove addresses: JSON response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
       "address_group": {
            "description": "",
            "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
            "name": "ADDR_GP_1",
            "project_id": "45977fa2dbd7482098dd68d0d8970117",
            "addresses": [
               "132.168.5.12-132.168.5.24"
            ]
        }
    }

Delete address group
^^^^^^^^^^^^^^^^^^^^

Deletes an address group.

This operation does not return a response body.

    +----------------+----------------------------------------------------+
    | Request Type   | ``DELETE``                                         |
    +----------------+----------------------------------------------------+
    | Endpoint       | ``/v2.0/address-groups/<address_group_id>``        |
    +----------------+---------+------------------------------------------+
    |                | Success | 204                                      |
    | Response Codes +---------+------------------------------------------+
    |                | Error   | Unauthorized(401), Not Found(404)        |
    |                |         | Conflict(409) The Conflict error response|
    |                |         | is returned when an operation is         |
    |                |         | performed while address group is in use. |
    +----------------+---------+------------------------------------------+

|

**Example Delete address group: JSON request**

.. code::

    DELETE /v2.0/address-groups/8722e0e0-9cc9-4490-9660-8c9a5732fbb0
    User-Agent: python-neutronclient
    Accept: application/json

**Example Delete address group: JSON response**

.. code::

    HTTP/1.1 204 No Content
    Content-Length: 0


Changes to the existing security group rules API
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

List security group rules
^^^^^^^^^^^^^^^^^^^^^^^^^

Lists security group rules.

Remote address group id field will be added to response.

.. code::

    GET /v2.0/security-group-rules
    User-Agent: python-neutronclient
    Accept: application/json


**Example List security group rules: JSON response**


.. code::

    {
        "security_group_rules": [
            {
                "direction": "ingress",
                "ethertype": "IPv4",
                "id": "f7d45c89-008e-4bab-88ad-d6811724c51c",
                "port_range_max": null,
                "port_range_min": null,
                "protocol": null,
                "remote_group_id": null,
                "remote_ip_prefix": null,
                "remote_address_group_id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
                "security_group_id": "85cc3048-abc3-43cc-89b3-377341426ac5",
                "project_id": "e4f50856753b4dc6afee5fa6b9b6c550",
                "revision_number": 1,
                "created_at": "2018-03-19T19:16:56Z",
                "updated_at": "2018-03-19T19:16:56Z",
                "tenant_id": "e4f50856753b4dc6afee5fa6b9b6c550",
                "description": ""
            }
        ]
    }

Show security group rule details
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Shows security group rule details.

Remote address group id field will be added to response.

**Example Show security group rule: JSON request**

.. code::

    GET /v2.0/security-group-rules/f7d45c89-008e-4bab-88ad-d6811724c51c
    User-Agent: python-neutronclient
    Accept: application/json


**Example Show security group rule: JSON response**

.. code::

    {
        "security_group_rule": {
            "direction": "ingress",
            "ethertype": "IPv4",
            "id": "f7d45c89-008e-4bab-88ad-d6811724c51c",
            "port_range_max": null,
            "port_range_min": null,
            "protocol": null,
            "remote_group_id": null,
            "remote_ip_prefix": null,
            "remote_address_group_id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
            "security_group_id": "85cc3048-abc3-43cc-89b3-377341426ac5",
            "project_id": "e4f50856753b4dc6afee5fa6b9b6c550",
            "revision_number": 1,
            "created_at": "2018-03-19T19:16:56Z",
            "updated_at": "2018-03-19T19:16:56Z",
            "tenant_id": "e4f50856753b4dc6afee5fa6b9b6c550",
            "description": ""
        }
    }



Create security group rule
^^^^^^^^^^^^^^^^^^^^^^^^^^

Creates a security group rule with a remote address group.

Note: At most one of remote-group, remote-ip and remote-address-group can be
specified. The rule is matched when the remote IP address in the packet matches
any one of: remote_ip_address, one of the IP addresses in the address group,
or an IP address of one of the ports in the remote security group.

**Example Create security group rule: JSON request**

.. code::

    POST /v2.0/security-group-rules
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "security_group_rule": {
            "direction": "ingress",
            "port_range_min": "80",
            "ethertype": "IPv4",
            "port_range_max": "80",
            "protocol": "tcp",
            "remote_address_group_id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
            "security_group_id": "a7734e61-b545-452d-a3cd-0189cbd9747a"
        }
    }

**Example Create security group rule: JSON response**

.. code::

    HTTP/1.1 201 Created
    Content-Type: application/json; charset=UTF-8

.. code::

    {
        "security_group_rule": {
            "direction": "ingress",
            "ethertype": "IPv4",
            "id": "2bc0accf-312e-429a-956e-e4407625eb62",
            "port_range_max": 80,
            "port_range_min": 80,
            "protocol": "tcp",
            "remote_ip_prefix": null,
            "remote_group_id": null,
            "remote_address_group_id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
            "security_group_id": "a7734e61-b545-452d-a3cd-0189cbd9747a",
            "project_id": "e4f50856753b4dc6afee5fa6b9b6c550",
            "revision_number": 1,
            "tenant_id": "e4f50856753b4dc6afee5fa6b9b6c550",
            "created_at": "2018-03-19T19:16:56Z",
            "updated_at": "2018-03-19T19:16:56Z",
            "description": ""
        }
    }


Data Model Impact
-----------------

The following are the backend database tables for the REST API proposed above.

|
| **Address Groups**

+-------------------+---------+-------+------+----------------------------------------+
| Attribute         | Type    | Req   | CRUD | Description                            |
+===================+=========+=======+======+========================================+
| id                | uuid-str| N/A   | R    | Unique identifier for the              |
|                   |         |       |      | address_group object.                  |
+-------------------+---------+-------+------+----------------------------------------+
| name              | String  | No    | CRU  | Human readable name for the address    |
|                   |         |       |      | group (255 characters limit). Does not |
|                   |         |       |      | have to be unique.                     |
+-------------------+---------+-------+------+----------------------------------------+
| description       | String  | No    | CRU  | Human readable description for the     |
|                   |         |       |      | address group (255 characters limit).  |
+-------------------+---------+-------+------+----------------------------------------+
| project_id        | uuid-str| Yes   | CR   | Owner of the address group. Only       |
|                   |         |       |      | admin users can specify a project      |
|                   |         |       |      | identifier other than their own.       |
+-------------------+---------+-------+------+----------------------------------------+


|
| **Address Group Address Associations**

+-------------------+---------+-------+------+----------------------------------------+
| Attribute         | Type    | Req   | CRUD | Description                            |
+===================+=========+=======+======+========================================+
| address_group_id  | uuid-str| No    | CRU  | UUID of address group.                 |
+-------------------+---------+-------+------+----------------------------------------+
| address           | String  | No    | CRU  | Address that has to be associated to   |
|                   |         |       |      | the address group.                     |
+-------------------+---------+-------+------+----------------------------------------+


|
| **Security Group Rules**

Attribute remote_address_group_id will be added to security group rules table

+------------------------+------------+-----+------+---------------------------------------+
| Attribute              | Type       | Req | CRUD |  Description                          |
+========================+============+=====+======+=======================================+
| remote_address         | String     | No  | CRU  | When a remote_address_group is        |
| _group_id              |            |     |      | specified, it is matched when the     |
|                        |            |     |      | remote IP address in the packet       |
|                        |            |     |      | matches one of the IP addresses in    |
|                        |            |     |      | the address group. This is exclusive  |
|                        |            |     |      | with remote_ip_prefix and             |
|                        |            |     |      | remote_group_id.                      |
+------------------------+------------+-----+------+---------------------------------------+


Implementation
==============

Assignee(s)
-----------

* Hang Yang

Work Items
----------

* REST API update
* DB schema update
* CLI update
* Open vSwitch and iptables firewall drivers update
* RBAC support for address groups
* Documentation update

Testing
=======

Tempest Tests
-------------

* DB mixin and schema tests
* Tempest tests
* CLI tests

Functional Tests
----------------

* New tests need to be written

API Tests
---------

* REST API and attributes validation tests

Documentation Impact
====================

User Documentation
------------------

* Neutron CLI and API documentation have to be modified.

Developer Documentation
-----------------------

* Neutron API devref and documentation need to be updated.

References
==========

[1] https://specs.openstack.org/openstack/neutron-specs/specs/rocky/fwaas-2.0-address-groups-support.html

[2] https://docs.openstack.org/api-ref/network/v2/#security-group-rules-security-group-rules
