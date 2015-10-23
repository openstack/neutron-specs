=============================
Firewall as a Service (FWaaS)
=============================

The FWaaS extension provides OpenStack users with the ability to deploy
firewalls to protect their networks. The FWaaS extension enables you to:

-  Apply firewall rules on traffic entering and leaving tenant networks.

-  Support for applying tcp, udp, icmp, or protocol agnostic rules.

-  Creation and sharing of firewall policies which hold an ordered
   collection of the firewall rules.

-  Audit firewall rules and policies.

This extension introduces these resources:

-  **firewall**: represents a logical firewall resource that a tenant
   can instantiate and manage. A firewall is associated with one
   firewall\_policy.

-  **firewall\_policy**: is an ordered collection of firewall\_rules. A
   firewall\_policy can be shared across tenants. Thus it can also be
   made part of an audit workflow wherein the firewall\_policy can be
   audited by the relevant entity that is authorized (and can be
   different from the tenants which create or use the firewall\_policy).

-  **firewall\_rule**: represents a collection of attributes like ports,
   ip addresses which define match criteria and action (allow, or deny)
   that needs to be taken on the matched data traffic.

Firewall rules
~~~~~~~~~~~~~~

Manage firewall rules.

**Table Firewall rule attributes**

Attribute

Type

Required

CRUD `:sup:`[a]` <#ftn.fwaas_rule_crud_note>`__

Default value

Validation constraints

Notes

id

uuid-str

N/A

R

generated

N/A

Unique identifier for the firewall rule object.

tenant\_id

uuid-str

Yes

CR

Derived from Authentication token

N/A

Owner of the firewall rule. Only admin users can specify a tenant
identifier other than their own.

name

String

No

CRU

None

N/A

Human readable name for the firewall rule (255 characters limit). Does
not have to be unique.

description

String

No

CRU

None

N/A

Human readable description for the firewall Rule (1024 characters
limit).

firewall\_policy\_id

uuid-str

No

R

None

N/A

This is a read-only attribute which gets populated with the uuid of the
firewall policy when this firewall rule is associated with a firewall
policy. A firewall rule can be associated with one firewall policy at a
time. The association can however be updated to a different firewall
policy. This attribute can be "null" if the rule is not associated with
any firewall policy.

shared

Bool

No

CRU

false

{true \false}

When set to True makes this firewall rule visible to tenants other than
its owner, and can be used in firewall policies not owned by its tenant.

protocol

String

No

CRU

None

{icmp \tcp \udp \null}

IP Protocol

ip\_version

Integer

No

CRU

4

{4 \6}

IP Protocol Version

source\_ip\_address

String (IP address or CIDR)

No

CRU

None

valid IP address (v4 or v6), or CIDR

Source IP address or CIDR

destination\_ip\_address

String (IP address or CIDR)

No

CRU

None

Valid IP address (v4 or v6), or CIDR

Destination IP address or CIDR

source\_port

Integer

No

CRU

None

Valid port number (integer or string), or port range in the format of a
':' separated range). In the case of port range, both ends of the range
are included.

Source port number or a range

destination\_port

Integer

No

CRU

None

Valid port number (integer or string), or port range in the format of a
':' separated range. In the case of port range, both ends of the range
are included.

Destination port number or a range

position

Integer

No

R

None

N/A

This is a read-only attribute that gets assigned to this rule when the
rule is associated with a firewall policy. It indicates the position of
this rule in that firewall policy. This position number starts at 1. The
position can be "null" if the firewall rule is not associated with any
policy.

action

String

No

CRU

deny

{allow \deny}

Action to be performed on the traffic matching the rule (allow, deny)

enabled

Bool

No

CRU

true

{true \false}

When set to False will disable this rule in the firewall policy.
Facilitates selectively turning off rules without having to disassociate
the rule from the firewall policy

-  **`:sup:`[a]` <#fwaas_rule_crud_note>`__\ C**. Use the attribute in
   create operations.

-  **R**. This attribute is returned in response to show and list
   operations.

-  **U**. You can update the value of this attribute.

-  **D**. You can delete the value of this attribute.



List firewall rules
^^^^^^^^^^^^^^^^^^^

**GET**

/fw/firewall\_rules

Lists firewall rules.

Normal Response Code: 200

Error Response Codes: Unauthorized (401).

This operation does not require a request body.

This operation returns a response body.

**Example 4.44. List firewall rules: JSON request**

.. code::

    GET /v2.0/fw/firewall_rules.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example List firewall rules: JSON response**

.. code::

    {
        "firewall_rules": [
            {
                "action": "allow",
                "description": "",
                "destination_ip_address": null,
                "destination_port": "80",
                "enabled": true,
                "firewall_policy_id": "c69933c1-b472-44f9-8226-30dc4ffd454c",
                "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
                "ip_version": 4,
                "name": "ALLOW_HTTP",
                "position": 1,
                "protocol": "tcp",
                "shared": false,
                "source_ip_address": null,
                "source_port": null,
                "tenant_id": "45977fa2dbd7482098dd68d0d8970117"
            }
        ]
    }



Show firewall rule details
^^^^^^^^^^^^^^^^^^^^^^^^^^

**GET**

/fw/firewall\_rules/*``firewall_rule-id``*

Shows firewall rule details.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Forbidden (403), Not Found
(404)

This operation does not require a request body.

This operation returns a response body.

**Example 4.46. Show firewall rule: JSON request**

.. code::

    GET /v2.0/fw/firewall_rules/9faaf49f-dd89-4e39-a8c6-101839aa49bc.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example Show firewall rule: JSON response**

.. code::

    {
        "firewall_rule": {
            "action": "allow",
            "description": "",
            "destination_ip_address": null,
            "destination_port": "80",
            "enabled": true,
            "firewall_policy_id": null,
            "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
            "ip_version": 4,
            "name": "ALLOW_HTTP",
            "position": null,
            "protocol": "tcp",
            "shared": false,
            "source_ip_address": null,
            "source_port": null,
            "tenant_id": "45977fa2dbd7482098dd68d0d8970117"
        }
    }



Create firewall rule
^^^^^^^^^^^^^^^^^^^^

**POST**

/fw/firewall\_rules

Creates a firewall rule.

Normal Response Code: 201

Error Response Codes: Unauthorized (401), Bad Request (400)

This operation requires a request body.

This operation returns a response body.

**Example 4.48. Create firewall rule: JSON request**

.. code::

    POST /v2.0/fw/firewall_rules.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "firewall_rule": {
            "action": "allow",
            "destination_port": "80",
            "enabled": true,
            "name": "ALLOW_HTTP",
            "protocol": "tcp"
        }
    }



**Example Create firewall rule: JSON response**

.. code::

    HTTP/1.1 201 Created
    Content-Type: application/json; charset=UTF-8

.. code::

    {
        "firewall_rule": {
            "action": "allow",
            "description": "",
            "destination_ip_address": null,
            "destination_port": "80",
            "enabled": true,
            "firewall_policy_id": null,
            "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
            "ip_version": 4,
            "name": "ALLOW_HTTP",
            "position": null,
            "protocol": "tcp",
            "shared": false,
            "source_ip_address": null,
            "source_port": null,
            "tenant_id": "45977fa2dbd7482098dd68d0d8970117"
        }
    }



Update firewall rule
^^^^^^^^^^^^^^^^^^^^

**PUT**

/fw/firewall\_rules/*``firewall_rule-id``*

Updates a firewall rule.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Bad Request (400), Not Found
(404)

**Example Update firewall rule: JSON request**

.. code::

    PUT /v2.0/fw/firewall_rules/41bfef97-af4e-4f6b-a5d3-4678859d2485.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "firewall_rule": {
            "shared": "true"
        }
    }



**Example Update firewall rule: JSON response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
        "firewall_rule": {
            "action": "allow",
            "description": "",
            "destination_ip_address": null,
            "destination_port": "80",
            "enabled": true,
            "firewall_policy_id": "c69933c1-b472-44f9-8226-30dc4ffd454c",
            "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
            "ip_version": 4,
            "name": "ALLOW_HTTP",
            "position": 1,
            "protocol": "tcp",
            "shared": true,
            "source_ip_address": null,
            "source_port": null,
            "tenant_id": "45977fa2dbd7482098dd68d0d8970117"
        }
    }



Delete firewall rule
^^^^^^^^^^^^^^^^^^^^

**DELETE**

/fw/firewall\_rules/*``firewall_rule-id``*

Deletes a firewall rule.

Normal Response Code: 204

Error Response Codes: Unauthorized (401), Not Found (404), Conflict
(409). The Conflict error response is returned when an operation is
performed while the firewall is in a PENDING state.

This operation does not require a request body.

This operation does not return a response body.

**Example Delete firewall rule: JSON request**

.. code::

    DELETE /v2.0/fw/firewall_rules/1be5e5f7-c45e-49ba-85da-156575b60d50.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example 4.53. Delete firewall rule: JSON response**

.. code::

    HTTP/1.1 204 No Content
    Content-Length: 0



Firewall policies
~~~~~~~~~~~~~~~~~

Manage firewall policies.

**Table 4.6. Firewall policy attributes**

Attribute

Type

Required

CRUD `:sup:`[a]` <#ftn.fwaas_policy_crud_note>`__

Default value

Validation constraints

Notes

id

uuid-str

N/A

R

generated

N/A

Unique identifier for the firewall policy object.

tenant\_id

uuid-str

Yes

CR

Derived from Authentication token

N/A

Owner of the firewall policy. Only admin users can specify a tenant
identifier other than their own.

name

String

No

CRU

None

N/A

Human readable name for the firewall policy (255 characters limit). Does
not have to be unique.

description

String

No

CRU

None

N/A

Human readable description for the firewall policy (1024 characters
limit)

shared

Bool

No

CRU

false

{true \false}

When set to True makes this firewall policy visible to tenants other
than its owner.

firewall\_rules

List

No

CRU

Empty list

JSON list of firewall rule uuids

This is an ordered list of firewall rule uuids. The firewall applies the
rules in the order in which they appear in this list.

audited

Bool

No

CRU

false

{true \false}

When set to True by the policy owner indicates that the firewall policy
has been audited. This attribute is meant to aid in the firewall policy
audit workflows. Each time the firewall policy or the associated
firewall rules are changed, this attribute will be set to False and will
have to be explicitly set to True through an update operation.

-  **`:sup:`[a]` <#fwaas_policy_crud_note>`__\ C**. Use the attribute in
   create operations.

-  **R**. This attribute is returned in response to show and list
   operations.

-  **U**. You can update the value of this attribute.

-  **D**. You can delete the value of this attribute.



List firewall policies
^^^^^^^^^^^^^^^^^^^^^^

**GET**

/fw/firewall\_policies

Lists firewall policies.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Forbidden (403)

This operation does not require a request body.

This operation returns a response body.

**Example List firewall policies: JSON request**

.. code::

    GET /v2.0/fw/firewall_policies.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example List firewall policies: JSON response**

.. code::

    {
        "firewall_policies": [
            {
                "audited": false,
                "description": "",
                "firewall_rules": [
                    "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
                ],
                "id": "c69933c1-b472-44f9-8226-30dc4ffd454c",
                "name": "test-policy",
                "shared": false,
                "tenant_id": "45977fa2dbd7482098dd68d0d8970117"
            }
        ]
    }



Show firewall policy details
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**GET**

/fw/firewall\_policies/*``firewall_policy-id``*

Shows firewall policy details.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Not Found (404)

This operation does not require a request body.

This operation returns a response body.

**Example Show firewall policy: JSON request**

.. code::

    GET /v2.0/fw/firewall_policies/9faaf49f-dd89-4e39-a8c6-101839aa49bc.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example Show firewall policy: JSON response**

.. code::

    {
        "firewall_policy": {
            "audited": false,
            "description": "",
            "firewall_rules": [
                "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
            ],
            "id": "c69933c1-b472-44f9-8226-30dc4ffd454c",
            "name": "test-policy",
            "shared": false,
            "tenant_id": "45977fa2dbd7482098dd68d0d8970117"
        }
    }



Create firewall policy
^^^^^^^^^^^^^^^^^^^^^^

**POST**

/fw/firewall\_policies

Creates a firewall policy.

Normal Response Code: 201

Error Response Codes: Unauthorized (401).

This operation requires a request body.

This operation returns a response body.

**Example 4.58. Create firewall policy: JSON request**

.. code::

    POST /v2.0/fw/firewall_policies.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "firewall_policy": {
            "firewall_rules": [
                "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
            ],
            "name": "test-policy"
        }
    }



**Example Create firewall policy: JSON response**

.. code::

    HTTP/1.1 201 Created
    Content-Type: application/json; charset=UTF-8

.. code::

    {
        "firewall_policy": {
            "audited": false,
            "description": "",
            "firewall_rules": [
                "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
            ],
            "id": "c69933c1-b472-44f9-8226-30dc4ffd454c",
            "name": "test-policy",
            "shared": false,
            "tenant_id": "45977fa2dbd7482098dd68d0d8970117"
        }
    }



Update firewall policy
^^^^^^^^^^^^^^^^^^^^^^

**PUT**

/fw/firewall\_policies/*``firewall_policy-id``*

Updates a firewall policy.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Not Found (404)

**Example Update firewall policy: JSON request**

.. code::

    PUT /v2.0/fw/firewall_policies/41bfef97-af4e-4f6b-a5d3-4678859d2485.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "firewall_policy": {
            "firewall_rules": [
                "a08ef905-0ff6-4784-8374-175fffe7dade",
                "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
            ]
        }
    }



**Example Update firewall policy: JSON response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
        "firewall_policy": {
            "audited": false,
            "description": "",
            "firewall_rules": [
                "a08ef905-0ff6-4784-8374-175fffe7dade",
                "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
            ],
            "id": "c69933c1-b472-44f9-8226-30dc4ffd454c",
            "name": "test-policy",
            "shared": false,
            "tenant_id": "45977fa2dbd7482098dd68d0d8970117"
        }
    }



Delete firewall policy
^^^^^^^^^^^^^^^^^^^^^^

**DELETE**

/fw/firewall\_policies/*``firewall_policy-id``*

Deletes a firewall policy.

Normal Response Code: 204

Error Response Codes: Unauthorized (401), Not Found (404), Conflict (409
). Conflict error code is returned the firewall policy is in use.

This operation does not require a request body.

This operation does not return a response body.

**Example Delete firewall policy: JSON request**

.. code::

    DELETE /v2.0/fw/firewall_policies/1be5e5f7-c45e-49ba-85da-156575b60d50.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example Delete firewall policy: JSON response**

.. code::

    HTTP/1.1 204 No Content
    Content-Length: 0



Insert firewall rule in firewall policy
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**PUT**

/fw/firewall\_policies/*``firewall_policy-id``*/insert\_rule

Inserts a firewall rule in a firewall policy relative to the position of
other rules.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Bad Request (400), Not Found
(404). Bad Request error is returned in the case the rule information is
missing.

**Example Insert firewall rule in firewall policy: JSON request**

.. code::

    PUT /v2.0/fw/firewall_policies/41bfef97-af4e-4f6b-a5d3-4678859d2485/insert_rule.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "firewall_rule_id": "7bc34b8c-8d3b-4ada-a9c8-1f4c11c65692",
        "insert_after": "a08ef905-0ff6-4784-8374-175fffe7dade",
        "insert_before": ""
    }



**Example Insert firewall rule in firewall policy: Response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
        "audited": false,
        "description": "",
        "firewall_list": [],
        "firewall_rules": [
            "a08ef905-0ff6-4784-8374-175fffe7dade",
            "7bc34b8c-8d3b-4ada-a9c8-1f4c11c65692",
            "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
        ],
        "id": "c69933c1-b472-44f9-8226-30dc4ffd454c",
        "name": "test-policy",
        "shared": false,
        "tenant_id": "45977fa2dbd7482098dd68d0d8970117"
    }



insert\_before and insert\_after parameters refer to firewall rule uuids
already associated with the firewall policy. firewall\_rule\_id refers
to uuid of the rule being inserted. insert\_before takes precedence over
insert\_after and if neither is specified, firewall\_rule\_is inserted
at the first position.

Remove firewall rule from firewall policy
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**PUT**

/fw/firewall\_policies/*``firewall_policy-id``*/remove\_rule

Removes a firewall rule from a firewall policy.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Bad Request (400), Not Found
(404). Bad Request error is returned if the rule information is missing
or when a firewall rule is tried to be removed from a firewall policy to
which it is not associated.

**Example Remove firewall rule from firewall policy: JSON
request**

.. code::

    PUT /v2.0/fw/firewall_policies/41bfef97-af4e-4f6b-a5d3-4678859d2485/remove_rule.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "firewall_rule_id": "7bc34b8c-8d3b-4ada-a9c8-1f4c11c65692"
    }



**Example Remove firewall rule from firewall policy: JSON
response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
        "audited": false,
        "description": "",
        "firewall_list": [],
        "firewall_rules": [
            "a08ef905-0ff6-4784-8374-175fffe7dade",
            "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
        ],
        "id": "c69933c1-b472-44f9-8226-30dc4ffd454c",
        "name": "test-policy",
        "shared": false,
        "tenant_id": "45977fa2dbd7482098dd68d0d8970117"
    }



Firewalls
~~~~~~~~~

Manage firewalls.

**Table Firewall attributes**

Attribute

Type

Required

CRUD\ `:sup:`[a]` <#ftn.fwaas_firewall_crud_note>`__

Default value

Validation constraints

Notes

id

uuid-str

N/A

R

generated

N/A

Unique identifier for the firewall object.

tenant\_id

uuid-str

Yes

CR

Derived from Authentication token

N/A

Owner of the firewall. Only admin users can specify a tenant identifier
other than their own.

name

String

No

CRU

None

N/A

Human readable name for the firewall (255 characters limit). Does not
have to be unique.

description

String

No

CRU

None

N/A

Human readable description for the firewall (1024 characters limit)

admin\_state\_up

Bool

N/A

CRU

true

{true \false }

Administrative state of the firewall. If false (down), firewall does not
forward packets and will drop all traffic to/from VMs behind the
firewall.

status

String

N/A

R

N/A

N/A

Indicates whether firewall resource is currently operational. Possible
values include: ACTIVE, DOWN, BUILD, ERROR, PENDING\_CREATE,
PENDING\_UPDATE, or PENDING\_DELETE.

shared

Bool

No

CRU

false

{true \false}

When set to True makes this firewall rule visible to tenants other than
its owner, and can be used in firewall policies not owned by its tenant.

firewall\_policy\_id

uuid-str

No

CRU

None

valid firewall policy uuid

The firewall policy uuid that this firewall is associated with. This
firewall will implement the rules contained in the firewall policy
represented by this uuid.

-  **`:sup:`[a]` <#fwaas_firewall_crud_note>`__\ C**. Use the attribute
   in create operations.

-  **R**. This attribute is returned in response to show and list
   operations.

-  **U**. You can update the value of this attribute.

-  **D**. You can delete the value of this attribute.



List firewalls
^^^^^^^^^^^^^^

**GET**

/fw/firewalls

Lists firewalls.

Normal Response Code: 200

Error Response Codes: Unauthorized (401)

This operation does not require a request body.

This operation returns a response body.

**Example 4.68. List firewalls: JSON request**

.. code::

    GET /v2.0/fw/firewalls.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example List firewalls: JSON response**

.. code::

    {
        "firewalls": [
            {
                "admin_state_up": true,
                "description": "",
                "firewall_policy_id": "c69933c1-b472-44f9-8226-30dc4ffd454c",
                "id": "3b0ef8f4-82c7-44d4-a4fb-6177f9a21977",
                "name": "",
                "status": "ACTIVE",
                "tenant_id": "45977fa2dbd7482098dd68d0d8970117"
            }
        ]
    }



Show firewall details
^^^^^^^^^^^^^^^^^^^^^

**GET**

/fw/firewalls/*``firewall-id``*

Shows firewall details.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Forbidden (403), Not Found
(404)

This operation does not require a request body.

This operation returns a response body.

**Example 4.70. Show firewall: JSON request**

.. code::

    GET /v2.0/fw/firewalls/9faaf49f-dd89-4e39-a8c6-101839aa49bc.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example Show firewall: JSON response**

.. code::

    {
        "firewall": {
            "admin_state_up": true,
            "description": "",
            "firewall_policy_id": "c69933c1-b472-44f9-8226-30dc4ffd454c",
            "id": "3b0ef8f4-82c7-44d4-a4fb-6177f9a21977",
            "name": "",
            "status": "ACTIVE",
            "tenant_id": "45977fa2dbd7482098dd68d0d8970117"
        }
    }



Create firewall
^^^^^^^^^^^^^^^

**POST**

/fw/firewalls

Creates a firewall.

Normal Response Code: 201

Error Response Codes: Unauthorized (401), Bad Request (400)

This operation requires a request body.

This operation returns a response body.

**Example Create firewall: JSON request**

.. code::

    POST /v2.0/fw/firewalls.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "firewall": {
            "admin_state_up": true,
            "firewall_policy_id": "c69933c1-b472-44f9-8226-30dc4ffd454c"
        }
    }



**Example Create firewall: JSON response**

.. code::

    HTTP/1.1 201 Created
    Content-Type: application/json; charset=UTF-8

.. code::

    {
        "firewall": {
            "admin_state_up": true,
            "description": "",
            "firewall_policy_id": "c69933c1-b472-44f9-8226-30dc4ffd454c",
            "id": "3b0ef8f4-82c7-44d4-a4fb-6177f9a21977",
            "name": "",
            "status": "PENDING_CREATE",
            "tenant_id": "45977fa2dbd7482098dd68d0d8970117"
        }
    }



Update firewall
^^^^^^^^^^^^^^^

**PUT**

/fw/firewalls/*``firewall-id``*

Updates a firewall, provided status is not ``PENDING_*``.

Normal Response Code: 200

Error Response Codes: Unauthorized (401), Bad Request (400), Not Found
(404)

**Example 4.74. Update firewall: JSON request**

.. code::

    PUT /v2.0/fw/firewalls/41bfef97-af4e-4f6b-a5d3-4678859d2485.json
    User-Agent: python-neutronclient
    Accept: application/json

.. code::

    {
        "firewall": {
            "admin_state_up": "false"
        }
    }



**Example Update firewall: JSON response**

.. code::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

.. code::

    {
        "firewall": {
            "admin_state_up": false,
            "description": "",
            "firewall_policy_id": "c69933c1-b472-44f9-8226-30dc4ffd454c",
            "id": "3b0ef8f4-82c7-44d4-a4fb-6177f9a21977",
            "name": "",
            "status": "PENDING_UPDATE",
            "tenant_id": "45977fa2dbd7482098dd68d0d8970117"
        }
    }



Delete firewall
^^^^^^^^^^^^^^^

**DELETE**

/fw/firewalls/*``firewall-id``*

Deletes a firewall.

Normal Response Code: 204

Error Response Codes: Unauthorized (401), Not Found (404)

This operation does not require a request body.

This operation does not return a response body.

**Example Delete firewall: JSON request**

.. code::

    DELETE /v2.0/fw/firewalls/1be5e5f7-c45e-49ba-85da-156575b60d50.json
    User-Agent: python-neutronclient
    Accept: application/json



**Example Delete firewall: JSON response**

.. code::

    HTTP/1.1 204 No Content
    Content-Length: 0



