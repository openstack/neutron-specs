..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================================================
Configurable Security Group rules added by default to the new Security Groups
=============================================================================

https://bugs.launchpad.net/neutron/+bug/1983053

Problem Description
===================

For every project, Neutron always creates security group called ``default``. It
is created always when first time security groups are going to be used or
requested to be listed via API. For each of such ``default`` security group
Neutron automatically creates 4 hardcoded rules. Those rules allow:

#. all ``IPv4`` egress traffic from the port,
#. all ``IPv6`` egress traffic from the port,
#. all ``IPv4`` ingress traffic to the port incoming from other ports which are
   using the same security group,
#. all ``IPv6`` ingress traffic to the port incoming from other ports which are
   using the same security group.

For any other security group created by user, Neutron automatically creates 2
rules which allows:

#. all ``IPv4`` egress traffic from the port,
#. all ``IPv6`` egress traffic from the port,

There is at least couple of issues with such approach:

* it is a known fact that the security group rules with remote_group_id (rule 3.
  and 4. above) don't scale well e.g. with neutron-openvswitch-agent [1]_,
* some operators would like to define different rules to be created by
  default for each new project.

Of course, those rules created by default by Neutron can be easily removed by
the security group owner, but having possibility to define different set of
rules added automatically to each security group could make it easier for users
and administrators of the cloud.

Proposed Change
===============

To solve the problem described above, proposal is to introduce new API to create
``security group rules template`` used to create new security group rules for
each new security group.
To keep backward compatibility with the existing hardcoded rules, by default
Neutron will have configured the same 4 rules as are hardcoded today (see above
for defails). Cloud admin will be able to delete those rules and create new ones
which will be then used by Neutron for every new security group.
In case when list of the ``default_security_group_rule`` is empty, new security
groups will be created without any rules created by default.

REST API Impact
---------------

New API resource called ``default_security_group_rule`` will be introduced.
Details of the API are below:

* ``GET /v2.0/default-security-group-rules``

  List default security group rules used to create rules for every new
  Security Group

  Response::

    {
      "default_security_group_rules": [
      {
          "direction": "egress",
          "ethertype": "IPv6",
          "id": "3c0e45ff-adaf-4124-b083-bf390e5482ff",
          "port_range_max": null,
          "port_range_min": null,
          "protocol": null,
          "remote_group_id": null,
          "remote_ip_prefix": null,
          "used_in_default_security_group": True
          "revision_number": 1,
          "created_at": "2022-09-15T19:16:56Z",
          "updated_at": "2022-09-15T19:16:56Z",
          "description": ""
      },
      {
          "direction": "egress",
          "ethertype": "IPv4",
          "id": "93aa42e5-80db-4581-9391-3a608bd0e448",
          "port_range_max": null,
          "port_range_min": null,
          "protocol": null,
          "remote_group_id": null,
          "remote_ip_prefix": null,
          "used_in_default_security_group": True
          "revision_number": 1,
          "created_at": "2022-09-15T19:16:56Z",
          "updated_at": "2022-09-15T19:16:56Z",
          "description": ""
      },
      {
          "direction": "ingress",
          "ethertype": "IPv6",
          "id": "3c0e45ff-adaf-4124-b083-bf390e5482ff",
          "port_range_max": null,
          "port_range_min": null,
          "protocol": null,
          "remote_group_id": "PARENT",
          "remote_ip_prefix": null,
          "used_in_default_security_group": True
          "revision_number": 1,
          "created_at": "2022-09-15T19:16:56Z",
          "updated_at": "2022-09-15T19:16:56Z",
          "description": ""
      },
      {
          "direction": "ingress",
          "ethertype": "IPv4",
          "id": "93aa42e5-80db-4581-9391-3a608bd0e448",
          "port_range_max": null,
          "port_range_min": null,
          "protocol": null,
          "remote_group_id": "PARENT",
          "remote_ip_prefix": null,
          "used_in_default_security_group": True
          "revision_number": 1,
          "created_at": "2022-09-15T19:16:56Z",
          "updated_at": "2022-09-15T19:16:56Z",
          "description": ""
      },
      {
          "direction": "ingress",
          "ethertype": "IPv6",
          "id": "3c0e45ff-adaf-4124-b083-bf390e5482ff",
          "port_range_max": 22,
          "port_range_min": 22,
          "protocol": null,
          "remote_group_id": null,
          "remote_ip_prefix": null,
          "used_in_default_security_group": False
          "revision_number": 1,
          "created_at": "2022-09-15T19:16:56Z",
          "updated_at": "2022-09-15T19:16:56Z",
          "description": "Allow SSH connections over IPv6"
      },
      {
          "direction": "ingress",
          "ethertype": "IPv4",
          "id": "3c0e45ff-adaf-4124-b083-bf390e5482ff",
          "port_range_max": 22,
          "port_range_min": 22,
          "protocol": null,
          "remote_group_id": null,
          "remote_ip_prefix": null,
          "used_in_default_security_group": False
          "revision_number": 1,
          "created_at": "2022-09-15T19:16:56Z",
          "updated_at": "2022-09-15T19:16:56Z",
          "description": "Allow SSH connections over IPv4"
      }]
    }

* ``POST /v2.0/default-security-group-rules``

  Create default security group rule used to create rules for every new
  Security Group

  Request::

    {
      "default_security_group_rule": {
        "direction": "ingress",
        "port_range_min": "80",
        "ethertype": "IPv4",
        "port_range_max": "80",
        "protocol": "tcp",
      }
    }

  Response::

    {
      "default_security_group_rule": {
        "direction": "ingress",
        "ethertype": "IPv4",
        "id": "2bc0accf-312e-429a-956e-e4407625eb62",
        "port_range_max": 80,
        "port_range_min": 80,
        "protocol": "tcp",
        "remote_group_id": null,
        "remote_ip_prefix": null,
        "used_in_default_security_group": False
        "revision_number": 1,
        "created_at": "2022-09-15T19:16:56Z",
        "updated_at": "2022-09-15T19:16:56Z",
        "description": ""
      }
    }

* ``GET /v2.0/default-security-group-rules/{rule_id}``

  Show default security group rule used to create rules for every new
  Security Group

  Response::

    {
      "security_group_rule": {
        "direction": "egress",
        "ethertype": "IPv6",
        "id": "3c0e45ff-adaf-4124-b083-bf390e5482ff",
        "port_range_max": null,
        "port_range_min": null,
        "protocol": null,
        "remote_group_id": null,
        "remote_ip_prefix": null,
        "used_in_default_security_group": False
        "revision_number": 1,
        "created_at": "2022-09-15T19:16:56Z",
        "updated_at": "2022-09-15T19:16:56Z",
      }
    }

* ``DELETE /v2.0/default-security-group-rules/{rule_id}``

  Delete default security group rule used to create rules for every new
  Security Group

DB Impact
---------

Default security group rule DB table:

+--------------------+---------+------+------+---------------------------------------+
| Attribute          | Type    | Req  | CRUD | Description                           |
+====================+=========+======+======+=======================================+
| id                 | uuid-str| No   | R    | Id of default security group rule.    |
+--------------------+---------+------+------+---------------------------------------+
| direction          | String  | Yes  | CR   | Direction in which the security group |
|                    |         |      |      | rule is applied.                      |
+--------------------+---------+------+------+---------------------------------------+
| ethertype          | String  | No   | CR   | Must be IPv4 or IPv6.                 |
+--------------------+---------+------+------+---------------------------------------+
| remote_group_id    | String  | No   | CR   | The remote group UUID to associate    |
|                    |         |      |      | with this security group rule.        |
|                    |         |      |      | Special value ``PARENT`` can be also  |
|                    |         |      |      | used and it means to always use       |
|                    |         |      |      | id of the security group in which     |
|                    |         |      |      | will be created with such rule.       |
+--------------------+---------+------+------+---------------------------------------+
| protocol           | String  | No   | CR   | The IP protocol can be represented by |
|                    |         |      |      | a string, an integer, or null.        |
|                    |         |      |      | Valid strings or integers are the     |
|                    |         |      |      | same as for the                       |
|                    |         |      |      | ``security group rule``.              |
+--------------------+---------+------+------+---------------------------------------+
| port_range_min     | String  | No   | CR   | The minimum port number in the        |
|                    |         |      |      | range that is matched by the security |
|                    |         |      |      | group rule.                           |
+--------------------+---------+------+------+---------------------------------------+
| port_range_max     | Integer | No   | CR   | The maximum port number in the        |
|                    |         |      |      | range that is matched by the security |
|                    |         |      |      | group rule.                           |
+--------------------+---------+------+------+---------------------------------------+
| remote_ip_prefix   | Integer | No   | CR   | The remote IP prefix that is matched  |
|                    |         |      |      | by this security group rule.          |
+--------------------+---------+------+------+---------------------------------------+
| standard_attr_id   | Ingeger | Yes  | R    | Id of the associated standard         |
|                    |         |      |      | attribute record.                     |
+--------------------+---------+------+------+---------------------------------------+
| used_in_default_sg | Boolean | No   | CR   | If it is set to ``True`` such rule    |
|                    |         |      |      | will be used in a template for the    |
|                    |         |      |      | ``default`` security group which is   |
|                    |         |      |      | created automatically for every       |
|                    |         |      |      | project. Default value is ``False``   |
+--------------------+---------+------+------+---------------------------------------+

Security Impact
---------------

New API will be by default available only for the admin users.


Performance Impact
------------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Slawek Kaplonski <skaplons@redhat.com> (IRC: slaweq)

Work Items
----------

* REST API update.

* DB schema update.

* Security Group DB code update.

* CLI update.

* Documentation.

* Tests and CI related changes.

Testing
=======

* Unit Test
* API test


Documentation Impact
====================

User Documentation
------------------

New API must be documented in the Neutron API reference document.


References
==========

.. [1] https://etherpad.opendev.org/p/openstack-networking-train-ptg#L348
