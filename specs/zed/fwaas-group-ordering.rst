..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
Firewall Group Ordering on Port Association
===========================================

https://bugs.launchpad.net/neutron/+bug/1979816

Currently, packets will sometimes be passed, and other times be blocked,
depending on the ordering of groups applied to a port. This is contrary
to the existing FWaaS spec, which states that a packet will be allowed so long
as any group on the port would allow the packet.

Problem Description
===================

When multiple firewall groups are applied to a port, the order in which the
groups are evaluated can change whenever one of the groups is modified. Therefore,
the combined firewall ruleset that results from multiple firewall groups is
rearranged unintentionally. This can result in certain traffic being allowed or
denied when the opposite behavior would be intended.

Proposed Change
===============

Similar to `firewall_policy_rule_associations_v2`, the
`firewall_group_port_associations_v2` table should have a required
`position` column to maintain the order in which `firewall groups` are
applied to ports.

In addition, modification of this ordering should be limited by user role.
For example, an openstack administrator may want a particular group to always be
applied first or last, regardless of which groups are added to a port by a tenant.
In iptables, this is typically referred to as `HEAD` and `TAIL` rules. All `HEAD`
groups should be applied first, in order. All `TAIL` groups should be applied last,
in order. All other groups would be applied in between, again, in order. Only
openstack users with the `admin` role should have access to the `HEAD` and `TAIL`
tiers by default.

Ex.

+--------------------------------------+--------------------------------------+----------+----------+
| firewall_group_id                    | port_id                              | position | tier     |
+--------------------------------------+--------------------------------------+----------+----------+
| da4be831-907b-43d9-86e0-b14a3bd391fc | efb7d60e-d3fc-4f97-91ed-ca71d930bb7c |        1 | HEAD     |
+--------------------------------------+--------------------------------------+----------+----------+
| 0814e179-d2be-464a-a9d4-e13c94451532 | efb7d60e-d3fc-4f97-91ed-ca71d930bb7c |        2 | HEAD     |
+--------------------------------------+--------------------------------------+----------+----------+
| 33ce9937-d9db-48b8-a65d-05fa3a75844a | efb7d60e-d3fc-4f97-91ed-ca71d930bb7c |        1 | null     |
+--------------------------------------+--------------------------------------+----------+----------+
| 6b3172af-9ae0-40e4-b455-c70de7c80c24 | efb7d60e-d3fc-4f97-91ed-ca71d930bb7c |        2 | null     |
+--------------------------------------+--------------------------------------+----------+----------+
| 70a7087e-c6ae-4cef-9b30-35e702746b68 | efb7d60e-d3fc-4f97-91ed-ca71d930bb7c |        1 | TAIL     |
+--------------------------------------+--------------------------------------+----------+----------+
| ff1e5eda-c285-4ec2-80f8-49f1a6d77347 | efb7d60e-d3fc-4f97-91ed-ca71d930bb7c |        2 | TAIL     |
+--------------------------------------+--------------------------------------+----------+----------+

`Position` should auto-increment if the `position` keyword is not specified. If the
`position` keyword is specified, and that number is available, that number is used.
If the number is already used, the existing groups are shifted downward from that point,
and the new group is applied in its place. For example, if positions 1-5 are in use, and
position 2 is added, the table would be updated as follows:

+----------+---------------+
| position | new position  |
+----------+---------------+
| 1        | 1             |
+----------+---------------+
| 2        | New Group (2) |
+----------+---------------+
| 2        | 3             |
+----------+---------------+
| 3        | 4             |
+----------+---------------+
| 4        | 5             |
+----------+---------------+
| 5        | 6             |
+----------+---------------+

REST API Impact
---------------

`PUT` and `POST` types for `/v2.0/fw/firewall_groups` will be updated to support the addition
of `position` and `tier.`

1. Response bodies should include the new fields.

   .. code-block:: python

    # Create (POST)
    {
        "firewall_rule": {
            "ports":[
                 "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
             ],
            "name": "FW_GROUP_1",
            "position": 2,
            "tier": "HEAD"
        }
    }

    # Update (PUT)
    {
        "firewall_rule": {
            "ports":[
                 "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
             ],
            "name": "FW_GROUP_1",
            "position": 3,
            "tier": "TAIL"
        }
    }

2. `GET` requests for both list and show methods should include the new values
in their responses.

   .. code-block:: python

    # List/Show (GET) Response
    {
        "firewall_groups": [
            {
                "description": "",
                "ingress_firewall_policy_id": null,
                "egress_firewall_policy_id": null,
                "id": "8722e0e0-9cc9-4490-9660-8c9a5732fbb0",
                "name": "FW_GROUP_1",
                "project_id": "45977fa2dbd7482098dd68d0d8970117",
                "ports":[
                     "8722e0e0-9cc9-4490-9660-8c9a5732fbb0"
                 ],
                "position": 3,
                "tier": "TAIL"
            }
        ]
    }

Data Model Impact
-----------------

`position` and `tier` are to be added to the `firewall_group_port_associations_v2` table.

Existing entries should be assigned consecutive `position` numbers starting at 1, and the
default `tier` value of `null.`

**Firewall Group Port associations**

+-----------+---------+-----+----------+---------------------------------------------------------------------------+
| Attribute | Type    | Req | CRUD     | Description                                                               |
+-----------+---------+-----+----------+---------------------------------------------------------------------------+
| position  | integer | Y   | CRU      | Position at which this firewall group is evaluated                        |
+-----------+---------+-----+----------+---------------------------------------------------------------------------+
| tier      | String  | Y   | CRU      | Tier at which this firewall group exists (HEAD, TAIL, null) Default: null |
+-----------+---------+-----+----------+---------------------------------------------------------------------------+

References
==========

https://etherpad.opendev.org/p/fwaas-api-evolution-spec
https://specs.openstack.org/openstack/neutron-specs/specs/newton/fwaas-api-2.0.html
