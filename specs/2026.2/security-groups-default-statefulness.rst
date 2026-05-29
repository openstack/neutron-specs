..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================
Security Groups Default Statefulness per Project
=====================================================

https://bugs.launchpad.net/neutron/+bug/2146803

Currently, when a default security group is automatically created for a
project, it is always created with ``stateful=True`` unless the API caller
explicitly specifies otherwise. This makes it difficult for operators who
want to adopt stateless security groups by default. This spec proposes a new
Neutron API extension called ``security-groups-default-statefulness`` [2]_
that allows operators and administrators to control the default value of the
``stateful`` attribute on a per-project or system-wide basis.


Problem Description
===================

Stateful security groups in OVN environments can cause performance
degradation depending on the workload, due to the design of OVN's conntrack
[1]_. Operators running OVN-based deployments may want all new security
groups to be **stateless** by default.

The current behavior presents several problems:

* **No system-wide default control.** There is no configuration option or
  API to change the system-wide default from ``stateful=True`` to
  ``stateful=False``. Operators must patch every newly created security group
  after the fact, or ensure every API call explicitly sets ``stateful=False``.

* **No per-project default control.** Different projects may have different
  performance and security requirements. Some projects may benefit from
  stateful connection tracking, while others require the throughput
  improvement of stateless security groups. There is currently no way to
  configure this on a per-project basis.

* **Immutability after port association.** Once a security group is
  associated with ports, its statefulness **cannot** be changed. This
  makes it critical to get the value right at creation time. If a default
  security group is created with the wrong statefulness, the only remedy is
  to delete all ports, change the security group, and recreate them.

Use cases:

#. **Deployer / Operator:** An operator managing an OVN-based cloud wants all
   new security groups across the entire system to be stateless by default,
   without requiring any changes from end users.

#. **Deployer / Operator:** An operator wants to configure specific projects
   to use stateless security groups while keeping stateful as the system-wide
   default.

#. **End User:** A user creates a new security group (or triggers creation of
   the ``default`` security group for a new project) and expects the
   statefulness to match the policy configured by the operator for their
   project (or the system-wide default).


Proposed Change
===============

A new API extension called ``security-groups-default-statefulness`` will be
introduced. This extension adds a new top-level resource at
``/security-groups-default-statefulness``.

This resource allows administrators to manage the default value of the
``stateful`` attribute that is used when creating new security groups. The
default can be defined at two levels:

#. **System-wide default** (no ``project_id``): applies to all projects that
   do not have a project-specific override.
#. **Per-project default** (with ``project_id``): overrides the system-wide
   default for a specific project.

When a new security group is created and the ``stateful`` attribute is not
explicitly provided in the API request, Neutron will determine the default
value using the following precedence:

#. If a per-project default exists for the project, use that value.
#. Otherwise, if a system-wide default exists, use that value.
#. Otherwise, fall back to ``stateful=True`` (current behavior, backward
   compatible).


REST API Impact
---------------

The extension introduces a new top-level resource at
``/v2.0/security-groups-default-statefulness``, following the same convention
as ``/v2.0/default-security-group-rules``.

* ``POST /v2.0/security-groups-default-statefulness``

  Create a default statefulness setting. If a ``project_id`` is provided, the
  setting applies only to that project. If ``project_id`` is omitted, the
  setting applies system-wide.

  Request (system-wide)::

    {
      "security_groups_default_statefulness": {
        "stateful": false
      }
    }

  Response::

    {
      "security_groups_default_statefulness": {
        "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        "project_id": null,
        "stateful": false
      }
    }

  Request (per-project)::

    {
      "security_groups_default_statefulness": {
        "project_id": "fb0e16da8ce7435c8a444647738e2166",
        "stateful": false
      }
    }

  Response::

    {
      "security_groups_default_statefulness": {
        "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
        "project_id": "fb0e16da8ce7435c8a444647738e2166",
        "stateful": false
      }
    }

* ``GET /v2.0/security-groups-default-statefulness``

  List all default statefulness settings (system-wide and per-project).

  Response::

    {
      "security_groups_default_statefulness": [
        {
          "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
          "project_id": null,
          "stateful": false
        },
        {
          "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
          "project_id": "fb0e16da8ce7435c8a444647738e2166",
          "stateful": false
        }
      ]
    }

* ``GET /v2.0/security-groups-default-statefulness/{id}``

  Show a specific default statefulness setting.

  Response::

    {
      "security_groups_default_statefulness": {
        "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        "project_id": null,
        "stateful": false
      }
    }

* ``PUT /v2.0/security-groups-default-statefulness/{id}``

  Update an existing default statefulness setting.

  Request::

    {
      "security_groups_default_statefulness": {
        "stateful": true
      }
    }

  Response::

    {
      "security_groups_default_statefulness": {
        "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        "project_id": null,
        "stateful": true
      }
    }

* ``DELETE /v2.0/security-groups-default-statefulness/{id}``

  Delete a default statefulness setting. After deletion, the affected scope
  (project or system-wide) reverts to the next applicable default.


DB Impact
---------

A new database table ``securitygroupdefaultstatefulness`` will be created:

+-------------------+-----------+------+------+------------------------------------------+
| Attribute         | Type      | Req  | CRUD | Description                              |
+===================+===========+======+======+==========================================+
| id                | uuid-str  | No   | R    | Unique identifier for this setting.      |
+-------------------+-----------+------+------+------------------------------------------+
| project_id        | String    | No   | CR   | The project to which this setting        |
|                   |           |      |      | applies. ``NULL`` means system-wide.     |
+-------------------+-----------+------+------+------------------------------------------+
| stateful          | Boolean   | Yes  | CRU  | The default value for the ``stateful``   |
|                   |           |      |      | attribute when creating new security     |
|                   |           |      |      | groups.                                  |
+-------------------+-----------+------+------+------------------------------------------+
| standard_attr_id  | Integer   | Yes  | R    | Id of the associated standard attribute  |
|                   |           |      |      | record.                                  |
+-------------------+-----------+------+------+------------------------------------------+

A unique constraint will be added on ``project_id`` to ensure that only one
setting exists per project and at most one system-wide setting (where
``project_id`` is ``NULL``).

Security Impact
---------------

* The new API will be restricted through Oslo policy rules. By default,
  **admin users** can manage both system-wide and per-project default
  statefulness settings. **Project managers** can manage per-project settings
  for their own project. Regular users will only be able to read the effective
  default for their own project.

* This change does **not** affect the enforcement of security group rules
  themselves; it only changes the default value used when the ``stateful``
  attribute is omitted during security group creation.

Performance Impact
------------------

Minimal. When creating a new security group, an additional DB lookup will
be performed to determine the applicable default statefulness value. This
lookup involves at most two queries (one for the project-specific setting,
one for the system-wide setting) and both are indexed.

In OVN-based deployments, adopting stateless security groups by default can
**improve** performance by avoiding conntrack overhead [1]_.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Rodolfo Alonso Hernandez <ralonsoh@redhat.com>

Work Items
----------

* neutron-lib: API definition for the ``security-groups-statefulness``
  extension (new resource attributes and sub-resource).

* Neutron: API extension implementation under
  ``neutron/extensions/security_groups_statefulness.py``.

* Neutron: DB model and migration for the new
  ``securitygroupdefaultstatefulness`` table.

* Neutron: Update security group creation logic to consult the default
  statefulness table when the ``stateful`` attribute is not explicitly
  provided.

* Neutron: Oslo policy rules for the new API endpoints.

* OpenStack CLI (python-openstackclient / python-neutronclient): add
  commands for managing default statefulness settings.

* openstacksdk: add support for the new sub-resource.

* Documentation updates (API reference, admin guide).

* Tests and CI related changes.


Testing
=======

* Unit tests for the new DB model, API extension, and creation logic.
* Functional tests for the API endpoints.
* API / Tempest tests to verify the end-to-end behavior:

  - Setting a system-wide default and verifying new security groups respect
    it.
  - Setting a per-project default and verifying it overrides the system-wide
    default.
  - Verifying that an explicit ``stateful`` attribute in the API request
    always takes precedence over any default.
  - Verifying backward compatibility when no default is configured.


Documentation Impact
====================

User Documentation
------------------

* The new API endpoints must be documented in the Neutron API reference.
* An admin guide section should explain how to configure default statefulness
  system-wide and per-project.
* Release notes should describe the new extension and its impact.


References
==========

.. [1] https://bugs.launchpad.net/neutron/+bug/1996593
.. [2] https://bugs.launchpad.net/neutron/+bug/2146803
