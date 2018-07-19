..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================================
Firewall as a Service API 2.0 Address Groups Support
===============================================================

**Launchpad blueprint:**

| https://blueprints.launchpad.net/neutron/+spec/fwaas-2.0-address-groups

This bp introduces a enhancement to Firewall as a Service(FWaaS) API 2.0
for supporting address groups. This feature has been proposed in
fwaas-api-2.0 but still not implemented.

Problem Description
===================

In actual use of firewall groups, each IP or subnet requires a
corresponding firewall rule. When there are a large number of instances,
a large number of firewall rules are generated and it is difficult to
maintain and manage them.


Proposed Change
===============

Add address group functions to a firewall group. By aggregating multiple
address objects into address groups and using address groups instead of
the original cidr to generate firewall rules, the number of firewall rules
can be effectively reduced.


REST API Impact
---------------

Firewall Address Groups
~~~~~~~~~~~~~~~~~~~~~~~~

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
| addresses         | List    | Yes   | CRU  | Array of key-value pairs of address   |
|                   |         |       |      | and ip version. It supports both CIDR |
|                   |         |       |      | and IP range objects. Attributes of   |
|                   |         |       |      | CIDR and IP range objects:            |
|                   |         |       |      | "address": <CIDR or IP range>         |
|                   |         |       |      | "ip_version": 4 or 6(Integer value)   |
|                   |         |       |      | An example of addresses:              |
|                   |         |       |      | [{"address": "132.168.4.12/24",       |
|                   |         |       |      | "ip_version": 4}]                     |
+-------------------+---------+-------+------+---------------------------------------+

|

Firewall Rules
~~~~~~~~~~~~~~

Note that as with FWaaS 1.0, in FWaaS 2.0 firewall rules always use stateful connection
tracking.

+------------------------+------------+-----+------+---------------------------------------+
| Attribute              | Type       | Req | CRUD |  Description                          |
+========================+============+=====+======+=======================================+
| id                     | uuid-str   | N/A | R    | Unique identifier for the firewall    |
|                        |            |     |      | rule object.                          |
+------------------------+------------+-----+------+---------------------------------------+
| project_id             | uuid-str   | No  | CR   | Owner of the firewall rule. Only      |
|                        |            |     |      | admin users can specify a project     |
|                        |            |     |      | identifier other than their own.      |
+------------------------+------------+-----+------+---------------------------------------+
| name                   | String     | No  | CRU  | Human readable name for the firewall  |
|                        |            |     |      | rule (255 characters limit). Does     |
|                        |            |     |      | not have to be unique.                |
+------------------------+------------+-----+------+---------------------------------------+
| description            | String     | No  | CRU  | Human readable description for the    |
|                        |            |     |      | firewall Rule (255 characters limit). |
+------------------------+------------+-----+------+---------------------------------------+
| shared                 | Bool       | No  | CRU  | When set to True makes this firewall  |
|                        |            |     |      | rule visible to projects other than   |
|                        |            |     |      | its owner, and can be used in         |
|                        |            |     |      | firewall policies not owned by its    |
|                        |            |     |      | project.                              |
+------------------------+------------+-----+------+---------------------------------------+
| protocol               | String     | No  | CRU  | IP Protocol.                          |
+------------------------+------------+-----+------+---------------------------------------+
| source_port            | port-range | No  | CRU  | Source port number or a range (an     |
|                        |            |     |      | int in [1, 65535] or range in a:b).   |
+------------------------+------------+-----+------+---------------------------------------+
| destination_port       | port-range | No  | CRU  | Destination port number or a range (  |
|                        |            |     |      | an int in [1, 65535] or range in a:b).|
+------------------------+------------+-----+------+---------------------------------------+
| ip_version             | Integer    | No  | CRU  | IP Protocol Version.                  |
+------------------------+------------+-----+------+---------------------------------------+
| source_ip_address      | String     | No  | CRU  | Source IP address or CIDR.            |
+------------------------+------------+-----+------+---------------------------------------+
| destination_ip_address | String     | No  | CRU  | Destination IP address or CIDR.       |
+------------------------+------------+-----+------+---------------------------------------+
| source_address         | List       | No  | CRU  | This is a list of source address      |
| _group_ids             |            |     |      | groups. When they are specified, they |
|                        |            |     |      | are matched when the source IP address|
|                        |            |     |      | in the packet matches one of the IP   |
|                        |            |     |      | addresses in one of the address       |
|                        |            |     |      | groups.                               |
+------------------------+------------+-----+------+---------------------------------------+
| destination_address    | List       | No  | CRU  | This is a list of destination address |
| _group_ids             |            |     |      | groups. When they are specified, they |
|                        |            |     |      | are matched when the destination IP   |
|                        |            |     |      | address in the packet matches one of  |
|                        |            |     |      | the IP addresses in one of the address|
|                        |            |     |      | groups.                               |
+------------------------+------------+-----+------+---------------------------------------+
| action                 | String     | No  | CRU  | Action to be performed on the         |
|                        |            |     |      | traffic matching the rule (ALLOW,     |
|                        |            |     |      | DENY, REJECT). Default: DENY.         |
+------------------------+------------+-----+------+---------------------------------------+
| enabled                | Bool       | No  | CRU  | When set to False will disable this   |
|                        |            |     |      | rule in the firewall policy.          |
|                        |            |     |      | Facilitates selectively turning off   |
|                        |            |     |      | rules without having to disassociate  |
|                        |            |     |      | the rule from the firewall policy.    |
|                        |            |     |      | Default: True.                        |
+------------------------+------------+-----+------+---------------------------------------+

|

Note: At most one of source_ip_address, source_address_group_ids and
source_firewall_group_id can be specified.  The rule is matched when the
source IP address in the packet matches any one of: source_ip_address,
one of the IP addresses in the address group, or an IP address of one
of the ports in the firewall group. If you want it to match any packet,
set the source or destination to 0.0.0.0/0 or ::/0. The same applies to
destination_ip_address, destination_address_group_ids, and destination
_firewall_group_id, with respect to the destination IP address in the
packet.


List address groups
^^^^^^^^^^^^^^^^^^^^^

Lists address groups.

    +----------------+------------------------------------------------+
    | Request Type   | ``GET``                                        |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/v2.0/fwaas/address_groups``                 |
    +----------------+---------+--------------------------------------+
    |                | Success | 200                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401)                    |
    +----------------+---------+--------------------------------------+

|

**Example List address groups: JSON request**

.. code::

    GET /v2.0/fwaas/address_groups.json
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
                   {"address": "132.168.4.12/24", "ip_version": 4},
                   {"address": "132.168.5.12-132.168.5.24", "ip_version": 4},
                   {"address": "2001::db8::f00/64", "ip_version": 6}
                ]
            }
        ]
    }

Show address group details
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Shows address group details.

    +----------------+----------------------------------------------------+
    | Request Type   | ``GET``                                            |
    +----------------+----------------------------------------------------+
    | Endpoint       | ``/v2.0/fwaas/address_groups/<address_group_id>``  |
    +----------------+---------+------------------------------------------+
    |                | Success | 200                                      |
    | Response Codes +---------+------------------------------------------+
    |                | Error   | Unauthorized(401), Not Found (404)       |
    +----------------+---------+------------------------------------------+

|

**Example Show address group: JSON request**

.. code::

    GET /v2.0/fwaas/address_groups/8722e0e0-9cc9-4490-9660-8c9a5732fbb0.json
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
               {"address": "132.168.4.12/24", "ip_version": 4},
               {"address": "132.168.5.12-132.168.5.24", "ip_version": 4},
               {"address": "2001::db8::f00/64", "ip_version": 6}
            ]
        }
    }



Create address group
^^^^^^^^^^^^^^^^^^^^^

Creates an address group.

    +----------------+------------------------------------------------+
    | Request Type   | ``POST``                                       |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/v2.0/fwaas/address_groups/``                |
    +----------------+---------+--------------------------------------+
    |                | Success | 201                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Bad Request(400)  |
    +----------------+---------+--------------------------------------+

|

**Example Create address group: JSON request**

.. code::

    POST /v2.0/fwaas/address_groups.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "address_group": {
            "name": "ADDR_GP_1",
            "addresses": [
               {"address": "132.168.4.12/24", "ip_version": 4},
               {"address": "132.168.5.12-132.168.5.24", "ip_version": 4},
               {"address": "2001::db8::f00/64", "ip_version": 6}
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
               {"address": "132.168.4.12/24", "ip_version": 4},
               {"address": "132.168.5.12-132.168.5.24", "ip_version": 4},
               {"address": "2001::db8::f00/64", "ip_version": 6}
            ]
        }
    }


Update address group
^^^^^^^^^^^^^^^^^^^^^

Updates an address group.

    +----------------+----------------------------------------------------+
    | Request Type   | ``PUT``                                            |
    +----------------+----------------------------------------------------+
    | Endpoint       | ``/v2.0/fwaas/address_groups/<address_group_id>``  |
    +----------------+---------+------------------------------------------+
    |                | Success | 200                                      |
    | Response Codes +---------+------------------------------------------+
    |                | Error   | Unauthorized(401), Bad Request(400) \    |
    |                |         | Not Found(404)                           |
    +----------------+---------+------------------------------------------+

|

**Example Update address group: JSON request**

.. code::

    PUT /v2.0/fwaas/address_groups/8722e0e0-9cc9-4490-9660-8c9a5732fbb0.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "address_group": {
            "addresses": [
               {"address": "132.168.4.12/24", "ip_version": 4},
               {"address": "132.168.5.12-132.168.5.24", "ip_version": 4},
               {"address": "2001::db8::f00/64", "ip_version": 6}
            ]
        }
    }


**Example Update address group: JSON response**

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
               {"address": "132.168.4.12/24", "ip_version": 4},
               {"address": "132.168.5.12-132.168.5.24", "ip_version": 4},
               {"address": "2001::db8::f00/64", "ip_version": 6}
            ]
        }
    }


Delete address group
^^^^^^^^^^^^^^^^^^^^^

Deletes an address group.

This operation does not return a response body.

    +----------------+----------------------------------------------------+
    | Request Type   | ``DELETE``                                         |
    +----------------+----------------------------------------------------+
    | Endpoint       | ``/v2.0/fwaas/address_groups/<address_group_id>``  |
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

    DELETE /v2.0/fwaas/address_groups/8722e0e0-9cc9-4490-9660-8c9a5732fbb0.json
    User-Agent: python-neutronclient
    Accept: application/json

**Example Delete address group: JSON response**

.. code::

    HTTP/1.1 204 No Content
    Content-Length: 0


List firewall rules
^^^^^^^^^^^^^^^^^^^^

Lists firewall rules.

    +----------------+------------------------------------------------+
    | Request Type   | ``GET``                                        |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/v2.0/fwaas/firewall_rules``                 |
    +----------------+---------+--------------------------------------+
    |                | Success | 200                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401)                    |
    +----------------+---------+--------------------------------------+

|

**Example List firewall rules: JSON request**

.. code::

    GET /v2.0/fwaas/firewall_rules.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example List firewall rules: JSON response**


.. code::

    {
        "firewall_rules": [
            {
                "action": "ALLOW",
                "description": "",
                "enabled": true,
                "firewall_policy_id": "56632e51-d2aa-4b79-9fd4-45f51088c4ed",
                "id": "9faaf49f-dd89-4e39-a8c6-101839aa49bc",
                "name": "ALLOW_HTTP",
                "position": 1,
                "shared": false,
                "protocol": "tcp",
                "source_port": null,
                "destination_port": "80",
                "ip_version": 4,
                "source_ip_address": null,
                "destination_ip_address": null
                "source_address_group_ids": [],
                "destination_address_group_ids": ["8315762a-f0ae-4f6b-981a-a16a6c3103c2"],
                "project_id": "45977fa2dbd7482098dd68d0d8970117"
            }
        ]
    }

Show firewall rule details
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Shows firewall rule details.

    +----------------+----------------------------------------------------+
    | Request Type   | ``GET``                                            |
    +----------------+----------------------------------------------------+
    | Endpoint       | ``/v2.0/fwaas/firewall_rules/<firewall_rule_id>``  |
    +----------------+---------+------------------------------------------+
    |                | Success | 200                                      |
    | Response Codes +---------+------------------------------------------+
    |                | Error   | Unauthorized(401), Not Found (404)       |
    +----------------+---------+------------------------------------------+

|

**Example Show firewall rule: JSON request**

.. code::

    GET /v2.0/fwaas/firewall_rules/9faaf49f-dd89-4e39-a8c6-101839aa49bc.json
    User-Agent: python-neutronclient
    Accept: application/json


**Example Show firewall rule: JSON response**

.. code::

    {
        "firewall_rule": {
            "action": "ALLOW",
            "description": "",
            "enabled": true,
            "firewall_policy_id": "56632e51-d2aa-4b79-9fd4-45f51088c4ed",
            "id": "9faaf49f-dd89-4e39-a8c6-101839aa49bc",
            "name": "ALLOW_HTTP",
            "position": 1,
            "shared": false,
            "protocol": "tcp",
            "source_port": null,
            "destination_port": "80",
            "ip_version": 4,
            "source_ip_address": null,
            "destination_ip_address": null,
            "source_address_group_ids": [],
            "destination_address_group_ids": ["8315762a-f0ae-4f6b-981a-a16a6c3103c2"],
            "project_id": "45977fa2dbd7482098dd68d0d8970117"
        }
    }



Create firewall rule
^^^^^^^^^^^^^^^^^^^^^

Creates a firewall rule.

    +----------------+------------------------------------------------+
    | Request Type   | ``POST``                                       |
    +----------------+------------------------------------------------+
    | Endpoint       | ``/v2.0/fwaas/firewall_rules/``                |
    +----------------+---------+--------------------------------------+
    |                | Success | 201                                  |
    | Response Codes +---------+--------------------------------------+
    |                | Error   | Unauthorized(401), Bad Request(400)  |
    +----------------+---------+--------------------------------------+

|

**Example Create firewall rule: JSON request**

.. code::

    POST /v2.0/fwaas/firewall_rules.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "firewall_rule": {
            "action": "ALLOW",
            "enabled": true,
            "name": "ALLOW_HTTP",
            "protocol": "tcp",
            "source_port": null,
            "destination_port": "80",
            "source_ip_address": null,
            "destination_ip_address": null,
            "source_address_group_ids": [],
            "destination_address_group_ids": ["8315762a-f0ae-4f6b-981a-a16a6c3103c2"]
        }
    }

**Example Create firewall rule: JSON response**

.. code::

    HTTP/1.1 201 Created
    Content-Type: application/json; charset=UTF-8

.. code::

    {
        "firewall_rule": {
            "action": "ALLOW",
            "description": "",
            "enabled": true,
            "firewall_policy_id": null,
            "id": "9faaf49f-dd89-4e39-a8c6-101839aa49bc",
            "name": "ALLOW_HTTP",
            "position": 1,
            "shared": false,
            "protocol": "tcp",
            "source_port": null,
            "destination_port": "80",
            "ip_version": 4,
            "source_ip_address": null,
            "destination_ip_address": null,
            "source_address_group_ids": [],
            "destination_address_group_ids": ["8315762a-f0ae-4f6b-981a-a16a6c3103c2"],
            "project_id": "45977fa2dbd7482098dd68d0d8970117"
        }
    }


Update firewall rule
^^^^^^^^^^^^^^^^^^^^^

Updates a firewall rule.

    +----------------+----------------------------------------------------+
    | Request Type   | ``PUT``                                            |
    +----------------+----------------------------------------------------+
    | Endpoint       | ``/v2.0/fwaas/firewall_rules/<firewall_rule_id>``  |
    +----------------+---------+------------------------------------------+
    |                | Success | 200                                      |
    | Response Codes +---------+------------------------------------------+
    |                | Error   | Unauthorized(401), Bad Request(400) \    |
    |                |         | Not Found(404)                           |
    +----------------+---------+------------------------------------------+

|

**Example Update firewall rule: JSON request**

.. code::

    PUT /v2.0/fwaas/firewall_rules/9faaf49f-dd89-4e39-a8c6-101839aa49bc.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "firewall_rule": {
            "shared": "true"
        }
    }

**Example Update firewall rule: JSON response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::


    {
        "firewall_rule": {
            "action": "ALLOW",
            "description": "",
            "enabled": true,
            "firewall_policy_id": null,
            "id": "9faaf49f-dd89-4e39-a8c6-101839aa49bc",
            "name": "ALLOW_HTTP",
            "position": 1,
            "shared": true,
            "protocol": "tcp",
            "source_port": null,
            "destination_port": "80",
            "ip_version": 4,
            "source_ip_address": null,
            "destination_ip_address": null,
            "source_address_group_ids": [],
            "destination_address_group_ids": ["8315762a-f0ae-4f6b-981a-a16a6c3103c2"],
            "project_id": "45977fa2dbd7482098dd68d0d8970117"
        }
    }


|

Delete firewall rule
^^^^^^^^^^^^^^^^^^^^^

Deletes a firewall rule.

This operation does not return a response body.

    +----------------+----------------------------------------------------+
    | Request Type   | ``DELETE``                                         |
    +----------------+----------------------------------------------------+
    | Endpoint       | ``/v2.0/fwaas/firewall_rules/<firewall_rule_id>``  |
    +----------------+---------+------------------------------------------+
    |                | Success | 204                                      |
    | Response Codes +---------+------------------------------------------+
    |                | Error   | Unauthorized(401), Not Found(404)        |
    |                |         | Conflict(409) The Conflict error response|
    |                |         | is returned when an operation is         |
    |                |         | performed while firewall rule is in use. |
    +----------------+---------+------------------------------------------+

|

**Example Delete firewall rule: JSON request**

.. code::

    DELETE /v2.0/fwaas/firewall_rules/9faaf49f-dd89-4e39-a8c6-101839aa49bc.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example Delete firewall rule: JSON response**

.. code::

    HTTP/1.1 204 No Content
    Content-Length: 0



Data Model Impact
------------------

The following are the backend database tables for the REST API proposed above.

|
| **Firewall Address Groups**


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
| **Firewall Address Group Address associations**

+-------------------+---------+-------+------+----------------------------------------+
| Attribute         | Type    | Req   | CRUD | Description                            |
+===================+=========+=======+======+========================================+
| id                | uuid-str| N/A   | R    | Unique identifier for the              |
|                   |         |       |      | address_group object.                  |
+-------------------+---------+-------+------+----------------------------------------+
| firewall_address  | uuid-str| No    | CRU  | UUID of firewall address group.        |
| _group_id         |         |       |      |                                        |
+-------------------+---------+-------+------+----------------------------------------+
| address           | String  | No    | CRU  | Address that has to be associated to   |
|                   |         |       |      | the firewall address group.            |
+-------------------+---------+-------+------+----------------------------------------+
| ip_version        | Integer | No    | CRU  | IP Protocol Version of the address.    |
+-------------------+---------+-------+------+----------------------------------------+



|
| **Firewall Rules**


+------------------------+------------+-----+------+---------------------------------------+
| Attribute              | Type       | Req | CRUD |  Description                          |
+========================+============+=====+======+=======================================+
| id                     | uuid-str   | N/A | R    | Unique identifier for the firewall    |
|                        |            |     |      | rule object.                          |
+------------------------+------------+-----+------+---------------------------------------+
| project_id             | uuid-str   | Yes | CR   | Owner of the firewall rule. Only      |
|                        |            |     |      | admin users can specify a project     |
|                        |            |     |      | identifier other than their own.      |
+------------------------+------------+-----+------+---------------------------------------+
| name                   | String     | No  | CRU  | Human readable name for the firewall  |
|                        |            |     |      | rule (255 characters limit). Does     |
|                        |            |     |      | not have to be unique.                |
+------------------------+------------+-----+------+---------------------------------------+
| description            | String     | No  | CRU  | Human readable description for the    |
|                        |            |     |      | firewall Rule (255 characters limit). |
+------------------------+------------+-----+------+---------------------------------------+
| shared                 | Bool       | No  | CRU  | When set to True makes this firewall  |
|                        |            |     |      | rule visible to projects other than   |
|                        |            |     |      | its owner, and can be used in         |
|                        |            |     |      | firewall policies not owned by its    |
|                        |            |     |      | project.                              |
+------------------------+------------+-----+------+---------------------------------------+
| protocol               | String     | No  | CRU  | IP Protocol.                          |
+------------------------+------------+-----+------+---------------------------------------+
| source_port            | port-range | No  | CRU  | Source port number or a range (an     |
|                        |            |     |      | int in [1, 65535] or range in a:b).   |
+------------------------+------------+-----+------+---------------------------------------+
| destination_port       | port-range | No  | CRU  | Destination port number or a range (  |
|                        |            |     |      | an int in [1, 65535] or range in a:b).|
+------------------------+------------+-----+------+---------------------------------------+
| ip_version             | Integer    | No  | CRU  | IP Protocol Version.                  |
+------------------------+------------+-----+------+---------------------------------------+
| source_ip_address      | String     | No  | CRU  | Source IP address or CIDR.            |
+------------------------+------------+-----+------+---------------------------------------+
| destination_ip_address | String     | No  | CRU  | Destination IP address or CIDR.       |
+------------------------+------------+-----+------+---------------------------------------+
| source_address         | List       | No  | CRU  | When a source_address_group is        |
| _group_ids             |            |     |      | specified, it is matched when the     |
|                        |            |     |      | source IP address in the packet       |
|                        |            |     |      | matches one of the IP addresses in    |
|                        |            |     |      | the address group.                    |
+------------------------+------------+-----+------+---------------------------------------+
| destination_address    | List       | No  | CRU  | When a destination_address_group is   |
| _group_ids             |            |     |      | specified, it is matched when the     |
|                        |            |     |      | destination IP address in the packet  |
|                        |            |     |      | matches one of the IP addresses in the|
|                        |            |     |      | address group.                        |
+------------------------+------------+-----+------+---------------------------------------+
| action                 | String     | No  | CRU  | Action to be performed on the         |
|                        |            |     |      | traffic matching the rule (ALLOW,     |
|                        |            |     |      | DENY, REJECT). Default: DENY.         |
+------------------------+------------+-----+------+---------------------------------------+
| enabled                | Bool       | No  | CRU  | When set to False will disable this   |
|                        |            |     |      | rule in the firewall policy.          |
|                        |            |     |      | Facilitates selectively turning off   |
|                        |            |     |      | rules without having to disassociate  |
|                        |            |     |      | the rule from the firewall policy.    |
|                        |            |     |      | Default: True.                        |
+------------------------+------------+-----+------+---------------------------------------+

|
| **Firewall Rules Source Address Group associations**

+-------------------+---------+-------+------+----------------------------------------+
| Attribute         | Type    | Req   | CRUD | Description                            |
+===================+=========+=======+======+========================================+
| id                | uuid-str| N/A   | R    | Unique identifier for the              |
|                   |         |       |      | address_group object.                  |
+-------------------+---------+-------+------+----------------------------------------+
| firewall_rule_id  | uuid-str| No    | CRU  | UUID of firewall rule.                 |
+-------------------+---------+-------+------+----------------------------------------+
| address_group_id  | String  | No    | CRU  | UUID of source address group.          |
+-------------------+---------+-------+------+----------------------------------------+

|
| **Firewall Rules Destination Address Group associations**

+-------------------+---------+-------+------+----------------------------------------+
| Attribute         | Type    | Req   | CRUD | Description                            |
+===================+=========+=======+======+========================================+
| id                | uuid-str| N/A   | R    | Unique identifier for the              |
|                   |         |       |      | address_group object.                  |
+-------------------+---------+-------+------+----------------------------------------+
| firewall_rule_id  | uuid-str| No    | CRU  | UUID of firewall rule.                 |
+-------------------+---------+-------+------+----------------------------------------+
| address_group_id  | String  | No    | CRU  | UUID of destination address group.     |
+-------------------+---------+-------+------+----------------------------------------+


Security Impact
---------------

None.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None.

Performance Impact
------------------

None.

IPv6 Impact
-----------

None.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

None.

Community Impact
----------------

None.

Alternatives
------------

None.

Implementation
==============

Assignee(s)
-----------

* Wang Tao

Work Items
----------

* REST API
* DB Schema
* FWaaS plugin update
* CLI update
* L3 agent iptables driver
* L2 agent ovs driver
* FWaaS dashboard

Dependencies
============


Testing
=======

Tempest Tests
--------------

* DB mixin and schema tests
* FWaaS Plugin with mocked driver end-to-end tests
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
-------------------

* Neutron CLI and FWaaS API documentation have to be modified.

Developer Documentation
-----------------------

* neutron-fwaas repo will have a devref and documentation will be written.

References
===========

[1] https://specs.openstack.org/openstack/neutron-specs/specs/newton/fwaas-api-2.0.html

[2] https://developer.openstack.org/api-ref/network/v2/#fwaas-v2-0-current-fwaas-firewall-groups-firewall-policies-firewall-rules

