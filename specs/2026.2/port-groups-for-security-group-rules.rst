..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================================================
Port Groups for Security Group Rules - Support Multiple Non-Contiguous Ports
=============================================================================

https://bugs.launchpad.net/neutron/+bug/2147189

Currently, Neutron security group rules only support a single port or a
continuous port range. This specification proposes to add support for
specifying multiple non-contiguous ports in a single security group rule
by introducing a new Port Group resource.


Problem Description
=====================

Current Limitations
-------------------

Neutron security group rules currently support only:

* A single port (e.g., port 80)
* A contiguous port range (e.g., ports 500-600)

To allow traffic on multiple non-contiguous ports (e.g., TCP ports 80, 443,
and 500-600), users must create separate security group rules for each port
or port range.

This limitation results in:

* Increased number of security group rules to manage
* More complex security group configurations
* Difficulty in managing large-scale deployments with thousands of rules

Use Cases
---------

1. **Web Servers**: Allow HTTP (80), HTTPS (443), and custom application
   ports (e.g., 8080, 8443) in a single rule
2. **Database Servers**: Allow multiple non-contiguous ports for different
   database services (e.g., MySQL 3306, PostgreSQL 5432, Redis 6379)
3. **Development Environments**: Allow multiple service ports for
   development and testing purposes
4. **Multi-Service Applications**: Applications that require multiple
   non-contiguous ports for different services

Industry Examples
-----------------

Major cloud providers already support this feature:

* **Google Cloud Platform (GCP)**: Supports comma-delimited port lists (e.g., "20-22, 80, 8080")

  Source: https://docs.cloud.google.com/firewall/docs/using-firewalls
* **Azure**: Supports comma-separated ports in augmented security rules (e.g., "80, 10000-10005")

  Source: https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview


Proposed Change
===============

This specification proposes to introduce a new Port Group resource, similar
to the existing Address Group resource. Port Groups will allow users to
define reusable collections of ports and port ranges, which can then be
referenced in security group rules.

Port Group Resource
-------------------

A new API extension ``port-group`` will be added to introduce the following
resources:

* **PortGroup**: A collection of ports and port ranges
* **PortAssociation**: Individual port or port range within a Port Group

Two additional API extensions will be added:

* ``rbac-port-group``: a shim extension registering the ``port_group`` object
  type in the existing Neutron RBAC framework
* ``security-groups-port-group``: adds the ``port_group_id`` field to the
  security group rules

Port Group Attributes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* ``id``: UUID
* ``name``: String (name of the port group)
* ``project_id``: String (project owner)
* ``shared``: Boolean (read-only, derived from the RBAC entries of the port
  group; it is not stored as a column of the port group table)
* ``ports``: List of port associations (synthetic field)

Sharing
~~~~~~~

Sharing a port group with other projects is handled by the existing Neutron
RBAC mechanism, in the same way as ``address_group`` does. A new RBAC object
type ``port_group`` will be added, with ``access_as_shared`` as the only valid
action.

This allows a port group to be shared with a single project or with every
project, instead of only being either private or fully public:

.. code::

    POST /v2.0/rbac-policies

    {
        "rbac_policy": {
            "action": "access_as_shared",
            "object_type": "port_group",
            "object_id": "port-group-uuid",
            "target_project": "project-uuid"
        }
    }

Using ``*`` as ``target_project`` shares the port group with all projects.
The ``shared`` attribute of a port group is reported as ``True`` when at least
one ``access_as_shared`` RBAC entry exists for it.

This covers the use case of an operator defining a well known set of ports
(for example, the TCP ports used by a Kubernetes control plane) once and
sharing it with the projects that need it.

Port Association Attributes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* ``port_range_min``: Integer (minimum port number, e.g., 80)
* ``port_range_max``: Integer (maximum port number, e.g., 90; same as port_range_min for single port)
* ``port_group_id``: UUID (reference to Port Group)

Security Group Rule Extension
------------------------------

The security group rule API will be extended to support referencing a Port
Group:

* ``port_group_id``: UUID (reference to Port Group)

This field cannot be used together with ``port_range_min`` and
``port_range_max``.

As done for ``remote_address_group_id`` with the
``security-groups-remote-address-group`` extension, this field is introduced
by its own API extension, ``security-groups-port-group``, so that it can be
advertised independently of the ``port-group`` extension itself.

REST API Impact
---------------

New API Endpoints
~~~~~~~~~~~~~~~~~~

* ``POST /v2.0/port-groups``: Create a port group
* ``GET /v2.0/port-groups``: List port groups
* ``GET /v2.0/port-groups/{id}``: Show port group details
* ``PUT /v2.0/port-groups/{id}``: Update port group
* ``DELETE /v2.0/port-groups/{id}``: Delete port group
* ``PUT /v2.0/port-groups/{id}/add_ports``: Atomically add ports to a port
  group. Body:

  .. code-block:: json

      {
          "ports": [
              {"port_range_min": 80, "port_range_max": 80},
              {"port_range_min": 443, "port_range_max": 443},
              {"port_range_min": 500, "port_range_max": 600}
          ]
      }

* ``PUT /v2.0/port-groups/{id}/remove_ports``: Atomically remove ports from a
  port group. Body:

  .. code-block:: json

      {
          "ports": [
              {"port_range_min": 80, "port_range_max": 80}
          ]
      }

The ``add_ports`` and ``remove_ports`` actions use ``PUT``, following the
``add_addresses`` and ``remove_addresses`` actions of the address group API.

Both actions modify a port group that may already be referenced by security
group rules. Applying such a change to the backends is described in
`Propagation of Port Group Changes`_.

A port group can only be removed when no security group rule references it,
as done for the address groups. Otherwise ``PortGroupInUse`` is returned.

Security Group Rule Extension
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Security group rule creation will support the new ``port_group_id``
field:

.. code::

    POST /v2.0/security-group-rules

    {
        "security_group_rule": {
            "direction": "ingress",
            "ethertype": "IPv4",
            "protocol": "tcp",
            "port_group_id": "port-group-uuid",
            "security_group_id": "sg-uuid"
        }
    }

DB Impact
---------

A new database model will be added to store the port groups and their associated port ranges.

**security_groups_port_groups table**:

+------------------+---------+------+-------+---------------------------------------------------+
| Attribute        | Type    | Req  | CRUD  | Description                                       |
+==================+=========+======+=======+===================================================+
| id               | uuid-str| No   | R     | ID of the port group.                             |
+------------------+---------+------+-------+---------------------------------------------------+
| name             | String  | No   | CRU   | Name of the port group.                           |
+------------------+---------+------+-------+---------------------------------------------------+
| project_id       | String  | No   | CR    | Project ID that owns the port group.              |
+------------------+---------+------+-------+---------------------------------------------------+
| standard_attr_id | Integer | Yes  | R     | ID of the associated standard attribute record.   |
+------------------+---------+------+-------+---------------------------------------------------+

**security_groups_port_ranges table**:

+------------------+---------+------+-------+------------------------------------------------+
| Attribute        | Type    | Req  | CRUD  | Description                                    |
+==================+=========+======+=======+================================================+
| port_range_min   | Integer | Yes  | CR    | Minimum port number in the range.              |
+------------------+---------+------+-------+------------------------------------------------+
| port_range_max   | Integer | Yes  | CR    | Maximum port number in the range.              |
+------------------+---------+------+-------+------------------------------------------------+
| port_group_id    | uuid-str| Yes  | CR    | ID of the port group this range belongs to.    |
+------------------+---------+------+-------+------------------------------------------------+

**security_groups_port_group_rbacs table**:

This table stores the RBAC entries of a port group, following the same model
as ``addressgrouprbacs``. The ``shared`` attribute of the API is derived from
its content, so no ``shared`` column is added to the port group table.

+----------------+---------+------+-------+--------------------------------------------------+
| Attribute      | Type    | Req  | CRUD  | Description                                      |
+================+=========+======+=======+==================================================+
| id             | uuid-str| No   | R     | ID of the RBAC entry.                            |
+----------------+---------+------+-------+--------------------------------------------------+
| project_id     | String  | No   | CR    | Project ID that owns the RBAC entry.             |
+----------------+---------+------+-------+--------------------------------------------------+
| target_project | String  | Yes  | CRU   | Project the port group is shared with, or ``*``. |
+----------------+---------+------+-------+--------------------------------------------------+
| action         | String  | Yes  | CR    | Only ``access_as_shared`` is valid.              |
+----------------+---------+------+-------+--------------------------------------------------+
| object_id      | uuid-str| Yes  | CR    | ID of the port group, FK with ``ON DELETE        |
|                |         |      |       | CASCADE``.                                       |
+----------------+---------+------+-------+--------------------------------------------------+

A unique constraint on (``target_project``, ``object_id``, ``action``) is
added, as done for the other RBAC tables.

**securitygrouprules table**:

A new column will be added to the existing securitygrouprules table:

+--------------+---------+------+-------+------------------------------------------------+
| Attribute    | Type    | Req  | CRUD  | Description                                    |
+==============+=========+======+=======+================================================+
| port_group_id| uuid-str| No   | CRU   | ID of the port group referenced by this rule.  |
+--------------+---------+------+-------+------------------------------------------------+

Mechanism Driver Implementation
---------------------------------

ML2/OVN Mechanism Driver
~~~~~~~~~~~~~~~~~~~~~~~~~

For ML2/OVN mechanism driver, Port Groups will be translated to OVN ACL match
expressions using set syntax:

* Input: ports = [80, 443, 500-600]
* Output: ``(tcp.dst == {80, 443} || (tcp.dst >= 500 && tcp.dst <= 600))``

This leverages the existing OVN ACL syntax capabilities.

The generated expression is parenthesised because it is appended with ``&&``
to the rest of the ACL match, which already carries the direction, the
ethertype, the remote address and the protocol. Without the parentheses the
``||`` would split the whole match, and the resulting ACL would allow traffic
that does not belong to the rule.

Other Mechanism Drivers (ML2/OVS)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For mechanism drivers that do not support multiple ports in a single rule, the Port
Group will be expanded into multiple backend rules:

* Input: Port Group with ports [80, 443, 500-600]
* Output: Three separate backend rules:

  * Rule 1: port_range_min=80, port_range_max=80
  * Rule 2: port_range_min=443, port_range_max=443
  * Rule 3: port_range_min=500, port_range_max=600

This ensures backward compatibility with existing mechanism drivers while maintaining
a single security group rule at the Neutron API level.

The expansion is done on the server side, when the security group rules are
sent to the agents. The agents and the firewall drivers are not modified and
never see the ``port_group_id`` field.

Propagation of Port Group Changes
----------------------------------

A port group is a shared and mutable resource, so ``add_ports`` and
``remove_ports`` change the meaning of every security group rule that
references it. Those changes are applied to the backends without requiring
any action on the security group rules themselves:

* The port group CRUD operations notify the ``port_group`` resource through
  the callback registry, as the address groups already do.
* For agent based mechanism drivers, the security groups referencing the port
  group are resolved and the ports using them are refreshed, so the expanded
  rules are recomputed.
* For ML2/OVN, the ACLs of the security groups referencing the port group are
  regenerated.

Empty Port Groups
------------------

A port group can hold no port at all, either because it was created empty or
because every port was removed from it. A security group rule referencing an
empty port group matches no traffic, on every mechanism driver.

This is the behaviour of the address groups today. An empty address group is
translated to an empty OVN address set, and a rule matching against it never
matches anything; for the agent based drivers the rule is expanded into no
rule at all.

Port groups need an explicit implementation to reach the same result, because
the port ranges are inlined in the ACL match instead of being referenced by
name as an address set is. For ML2/OVN, no ACL is created for a security group
rule whose port group is empty, which is exactly what the expansion already
produces for the agent based drivers.

Implementation
===============

Assignee(s)
-----------

* Kyuyeong Lee <kyu0.lee@samsung.com>

Work Items
----------

* API Definition

  * Define PortGroup and PortAssociation objects
  * Define API extension ``port-group``
  * Define API extension ``security-groups-port-group``, adding the
    ``port_group_id`` field to the security group rules
  * Define shim API extension ``rbac-port-group``

* Database Implementation

  * Create security_groups_port_groups table
  * Create security_groups_port_ranges table
  * Create security_groups_port_group_rbacs table
  * Add port_group_id column to securitygrouprules table

* Service Implementation

  * Implement PortGroup CRUD operations
  * Implement add_ports/remove_ports operations
  * Implement security group rule validation for port_group_id
  * Notify the ``port_group`` resource on every change, so that the mechanism
    drivers can refresh the affected security groups

* RBAC Implementation

  * Add the ``port_group`` RBAC object type with the ``access_as_shared``
    action
  * Make the PortGroup object RBAC aware and expose the derived ``shared``
    attribute
  * Add the ``shared_port_groups`` policy rule and allow the members of the
    target projects to get a shared port group
  * Prevent the removal of an RBAC entry while a security group rule of the
    target project still references the port group

* Mechanism Driver Implementation

  * ML2/OVN: Implement Port Group to OVN ACL translation, and skip the ACL of
    a security group rule whose port group is empty
  * Other mechanism drivers: Implement Port Group expansion logic

Testing
========


* Unit tests for PortGroup and PortAssociation objects

* Functional tests for PortGroup CRUD operations

* Functional tests for security group rules with port_group_id

* Unit and functional tests for the ``port_group`` RBAC object type, covering
  sharing with a single project, sharing with all projects and the removal of
  an RBAC entry still in use

* Functional tests covering the propagation of ``add_ports`` and
  ``remove_ports`` to the backends

* Functional tests checking that a security group rule referencing an empty
  port group matches no traffic, on every mechanism driver

* A single set of Tempest tests, executed by the jobs of the different
  mechanism drivers, so that ML2/OVN and ML2/OVS are validated against the
  same expected behaviour

* Tempest tests for using a shared port group from another project

Documentation
==============

* User Documentation: How to create and use Port Groups

* Admin Documentation: How to share a Port Group with other projects, to be
  added to the RBAC configuration guide

* API Documentation: Port Group API reference

References
===========

* RFE: https://bugs.launchpad.net/neutron/+bug/2147189

* GCP Firewall Documentation: https://docs.cloud.google.com/firewall/docs/using-firewalls

* Azure Network Security Groups: https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview

* OVN ACL Syntax: https://github.com/ovn-org/ovn/blob/main/tests/ovn.at#L365

* Neutron RBAC configuration: https://docs.openstack.org/neutron/latest/admin/config-rbac.html

* Address group RBAC support, used as the reference implementation for the
  sharing of port groups:
  https://opendev.org/openstack/neutron/commit/8094b524f693180fe9f435ef0d1f23b6a44259ae
