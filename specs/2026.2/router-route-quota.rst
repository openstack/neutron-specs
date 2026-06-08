..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================
Router Route Quota
==================

https://bugs.launchpad.net/neutron/+bug/2026489

Replace the static ``[DEFAULT] max_routes`` configuration option with a
``router_route`` quota resource managed by the Neutron quota engine. The
limit value is configured per project (quota API / ``[QUOTAS]``), but
**enforcement is per parent resource (per router)**, not as a project-wide
aggregate across all routers.

This preserves the semantics of ``max_routes`` today: each router may have
up to *N* extra routes, where *N* comes from the project quota.


Problem Description
===================

Extra routes on Neutron routers are currently capped by the global
configuration option ``max_routes`` (default: 30). This has several
drawbacks:

* Operators cannot set different limits per project.
* The limit is not exposed through the quota API or the
  ``openstack quota`` commands.
* It does not follow the standard Neutron quota framework, which already
  manages networks, subnets, ports, routers, floating IPs, security
  groups, etc.

Deployers need a quota-managed limit that:

* can be set per project (and optionally overridden via the quota API);
* is enforced when routes are added or updated on a router;
* keeps the existing **per-router** behavior of ``max_routes``.


Quota Scope: Per Parent, Not Per Project
========================================

Most Neutron quotas (``network``, ``subnet``, ``router``, ``port``, ...)
count resources **across the whole project**. ``router_route`` is
different: the **limit is stored per project**, but **usage is checked per
router** (the parent object). A project with quota 30 and 10 routers can
have up to 30 routes on *each* router (300 total), not 30 routes shared
across all routers.

This pattern already exists in other OpenStack services:

.. list-table::
   :header-rows: 1
   :widths: 25 10 30

   * - Resource
     - Service
     - Enforcement scope
   * - ``server_group_members``
     - Nova
     - per server group
   * - ``per_volume_gigabytes``
     - Cinder
     - per volume
   * - ``key_pairs``
     - Nova
     - per user
   * - ``max_routes`` (today)
     - Neutron
     - per router (config only)

Like ``server_group_members`` or ``key_pairs`` in Nova, project-level
``openstack quota show --usage`` is not expected to show meaningful
``in_use`` for ``router_route``; operators inspect individual routers
instead.


Proposed Change
===============

Configuration and Quota Registration
-------------------------------------

* Add ``quota_router_route`` in ``[QUOTAS]`` (default: 30, matching the
  current ``max_routes`` default).
* Register ``router_route`` as a quota resource from the ``extraroute``
  extension via ``get_resources()``.
* Deprecate ``max_routes``; add an upgrade check warning when it is set
  to a non-default value.

Enforcement
-----------

* Remove the ``len(routes) > max_routes`` check from
  ``_validate_routes()``.
* In ``_update_extra_routes()``, before adding routes, compare the
  **resulting number of routes on that router** against the project's
  ``router_route`` limit.
* Reject with HTTP 409 (``OverQuota``) when the per-router limit would be
  exceeded. This replaces the current ``RoutesExhausted`` (HTTP 400) with
  standard quota-exceeded semantics.

Counting
--------

* Count routes for the **router being updated**, not a project-wide JOIN
  across all routers. The check is:
  ``existing_routes_on_this_router + newly_added_routes <= quota``.
* Quota usage tracking at project level (``quota show --usage``) reports
  ``in_use: 0`` for ``router_route`` (non-aggregatable resource),
  consistent with Nova ``server_group_members`` and ``key_pairs``.

API / Clients
-------------

* ``router_route`` appears in Neutron quota show/set/list API responses.
* Follow-up patches are needed in:

  - ``python-openstackclient`` (add ``router_route`` to ``NETWORK_QUOTAS``)
  - ``openstacksdk``


References
==========

* Launchpad RFE: https://bugs.launchpad.net/neutron/+bug/2026489
* POC: https://review.opendev.org/c/openstack/neutron/+/991024
* Nova ``server_group_members``: ``nova/limit/local.py``,
  ``nova/quota.py``
* Cinder ``per_volume_gigabytes``: ``cinder/volume/api.py``
