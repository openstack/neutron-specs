..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================
Tap-as-a-Service: filtering rules for lport mirrors
===================================================

https://bugs.launchpad.net/neutron/+bug/2168008

A tap mirror copies everything the mirrored port sends and receives. This
spec adds filtering rules to tap mirrors of type ``lport`` so that users can
choose, at the source, which traffic is copied to the collector. The rules
are a new ``rules`` sub-resource of tap mirrors, use the vocabulary of
security group rules, and are implemented with the OVN ``Mirror_Rule`` table
(OVN 25.09 and later). The feature is ML2/OVN only.

The ``lport`` mirror type itself is covered by RFE 2168007, which was
approved without a spec at the drivers meeting of 2026-10-02 [1]_. The same
meeting asked for this spec because the rules add a new API and a new
database table, and raised three points that this spec answers: the
``tap_mirrors`` resource now has attributes that apply to some mirror types
only, the TaaS API has two models (tap services/flows and tap mirrors) whose
driver support differs, and the feature needs proper testing.


Problem Description
===================

With the ``lport`` mirror type a project can send a copy of the traffic of
one of its ports to a collector port (an IDS, a packet broker) inside the
overlay. Today such a mirror always copies everything. That is usually far
more than the collector needs:

* the copies double the traffic of the mirrored port on the overlay and on
  the collector port;
* several mirrored ports often feed the same collector, so the collector has
  to drop most of what it receives;
* security teams want to express the selection where it is cheapest, at the
  source, for example "only inbound TCP/443 and DNS" or "everything except
  the backup traffic to 10.0.9.0/24".

Use cases:

* A project member mirrors a web server port and only wants inbound
  TCP/443 at the collector.
* A project member mirrors a database port and wants everything except the
  replication traffic to a known subnet.
* An operator mirrors a tenant port into a collector in a service project
  (admin only, as for lport mirrors) and only wants traffic to and from the
  internet-facing prefixes.

OVN already provides the mechanism: since 25.09 a ``Mirror`` row of type
``lport`` can reference ``Mirror_Rule`` rows (priority, match, action
``mirror`` or ``skip``). Neutron has no API for it.

Background: lport tap mirrors (RFE 2168007)
-------------------------------------------

This spec builds on the ``tap-mirror-lport`` extension of RFE 2168007. For
the reader's convenience, what that extension does:

* ``mirror_type`` gets the value ``lport``; ``remote_port_id`` names the
  Neutron port that receives the copies; ``remote_ip`` and the tunnel ids
  in ``directions`` are not used for this type.
* The sink port must exist, must be bound to a host, must differ from the
  mirrored port and must belong to the same project unless the caller is an
  admin. Deleting either port deletes the mirror.
* A port can have at most one lport mirror per direction, because OVN
  installs one unconditional "mirror everything" flow per mirror and
  direction on the source port.
* The OVN driver creates one OVN ``Mirror`` row per mirrored direction:
  ``tm_in_<id>`` with filter ``to-lport`` for ``IN`` (traffic delivered to
  the port) and ``tm_out_<id>`` with filter ``from-lport`` for ``OUT``
  (traffic the port sends).

How OVN implements lport mirrors and rules
------------------------------------------

The semantics proposed below follow directly from ``ovn-northd`` [3]_, so
they are summarised here. For each lport ``Mirror`` attached to a logical
switch port, northd:

* creates a mirror port ``mp-<datapath>-<sink>`` on the switch of the
  mirrored port that forwards to the sink port, so the sink may be on
  another logical switch;
* installs a "pass" flow at priority 100 in ``ls_in_mirror`` (packets the
  port sends) and/or ``ls_out_mirror`` (packets delivered to the port),
  matching the port and executing ``mirror(<mp>); next;``;
* installs one flow per ``Mirror_Rule`` at priority ``100 + priority`` in
  the same stage(s), matching ``<port> && (<rule match>)``, with action
  ``mirror(<mp>); next;`` for ``mirror`` and ``next;`` for ``skip``;
* installs a flow at priority 65535 in ``ls_out_pre_acl`` that sends
  packets whose output port is the mirror port straight to
  ``ls_out_apply_port_sec``, so the copies bypass the ACL (security group),
  QoS and stateful stages on their way to the collector.

``ls_in_mirror`` is the third stage of the ingress pipeline, before the ACL
stages; ``ls_out_mirror`` comes right after the egress ACL stages. Hence
traffic the port sends is copied before its security groups are evaluated,
and traffic delivered to the port is copied after them (only what the port
actually receives). A ``Mirror_Rule`` has a ``match`` in the logical flow
expression language, an ``action`` (``mirror`` or ``skip``) and a
``priority`` in 0-32767; OVN identifies a rule by its priority and match.

Rules are not proposed for the ``gre`` and ``erspanv1`` mirror types. In
OVN those are plain OVS port mirrors configured by ``ovn-controller`` and do
not go through the logical pipeline, so there is no logical flow where a
match could be applied.


Proposed Change
===============

Overview
--------

A new API extension ``tap-mirror-rules`` (requires ``tap-mirror`` and
``tap-mirror-lport``) adds a ``rules`` sub-resource to tap mirrors::

  POST   /v2.0/taas/tap_mirrors/{tap_mirror_id}/rules
  GET    /v2.0/taas/tap_mirrors/{tap_mirror_id}/rules
  GET    /v2.0/taas/tap_mirrors/{tap_mirror_id}/rules/{rule_id}
  DELETE /v2.0/taas/tap_mirrors/{tap_mirror_id}/rules/{rule_id}

The sub-resource follows the model of QoS policy rules (neutron-lib
sub-resource definition, controller with a parent, per-rule policies).
Rules are immutable: to change a rule, delete it and create a new one. This
keeps the mapping to OVN rows one-to-one and avoids partial updates of a
rule that is already applied on the dataplane. The ``directions`` of a tap
mirror are immutable too, so the direction of a rule can not become invalid
after it was created.

Attributes by mirror type
-------------------------

At the RFE review it was pointed out that ``tap_mirrors`` now has attributes
that are only valid for some mirror types [1]_. The API makes this explicit
and rejects an attribute that does not apply, instead of ignoring it:

==========================  =======================  ======================
Attribute                   ``gre`` / ``erspanv1``   ``lport``
==========================  =======================  ======================
``port_id``                 required                 required
``remote_ip``               required                 rejected (``400``)
``directions`` tunnel ids   required, one per        must be ``null``
                            direction
``remote_port_id``          rejected (``400``)       required
``rules`` sub-resource      create rejected          supported
                            (``400``)
==========================  =======================  ======================

The lport implementation of RFE 2168007 rejects a ``remote_ip`` or tunnel
id given for an lport mirror, so that a user can not believe a value is used
when it is not.
The API reference and the user guide will carry the same table.

Rule attributes
---------------

==================================  ==========  =============================
Attribute                           Default     Notes
==================================  ==========  =============================
``priority``                        required    1-32767, higher is evaluated
                                                first
``action``                          ``mirror``  ``mirror`` or ``skip``
``direction``                       ``null``    ``IN``, ``OUT`` or ``null``
                                                (every direction the mirror
                                                copies)
``ethertype``                       ``null``    ``IPv4``, ``IPv6`` or
                                                ``null`` (any frame)
``protocol``                        ``null``    ``tcp``, ``udp``, ``sctp``,
                                                ``icmp``, ``ipv6-icmp``
``source_ip_prefix``                ``null``    CIDR
``destination_ip_prefix``           ``null``    CIDR
``source_port_range_min/max``       ``null``    1-65535, TCP/UDP/SCTP only
``destination_port_range_min/max``  ``null``    1-65535, TCP/UDP/SCTP only
==================================  ==========  =============================

The attribute names and values are those of security group rules, with two
differences that follow from the purpose of the rules: ``priority`` and
``action`` exist, and ``ethertype`` is optional. A security group rule
always needs an ethertype because it describes IP traffic; a mirror rule
without any match attribute means "every frame", which is exactly what the
catch-all ``skip`` rule of the "mirror only what I list" pattern needs, and
a rule such as ``protocol=tcp`` without ethertype matches TCP over IPv4 and
IPv6 alike. When an IP prefix is given, the ethertype is derived from its
address family; a prefix that disagrees with an explicit ``ethertype`` or
with the other prefix is rejected. ``icmp`` implies ``IPv4`` and
``ipv6-icmp`` implies ``IPv6``. Port ranges require ``tcp``, ``udp`` or
``sctp``; a range with ``min`` greater than ``max`` is rejected; one of
``min``/``max`` alone means ``min-65535`` or ``1-max``.

Translation to the OVN match
----------------------------

The server translates a rule into an OVN logical flow match expression;
the raw expression is deliberately not exposed in the API. It would tie the
API to one backend's expression language and would need its own validation.
A backend specific attribute can be added later if a real need appears.
Examples:

* no match attribute (catch-all)::

    1

* ``ethertype=IPv4``::

    ip4

* ``protocol=tcp`` (IPv4 and IPv6)::

    tcp

* ``protocol=tcp``, ``destination_ip_prefix=10.0.0.0/24``,
  ``destination_port_range_min=443``, ``destination_port_range_max=443``::

    ip4 && tcp && ip4.dst == 10.0.0.0/24 && tcp.dst == 443

* ``protocol=ipv6-icmp``::

    ip6 && icmp6

* ``protocol=udp``, ``source_port_range_min=1024``::

    udp && udp.src >= 1024 && udp.src <= 65535

The translation is a pure function (no OVN connection needed), so every
attribute combination, valid and invalid, is covered by unit tests.

Semantics
---------

* **Default allow.** OVN always installs the unconditional pass flow at
  priority 100 and places the rules above it at ``100 + priority``. So a
  mirror without rules, or a packet that matches no rule, is mirrored. This
  is what tap mirrors do today and gre/erspan mirrors have no rules at all.
  Users that want "mirror only what I list" add a catch-all ``skip`` rule
  (no match attribute) at priority 1. AWS VPC Traffic Mirroring and K2 Cloud
  chose the opposite default at their product layer; the drivers did not ask
  to change it at the RFE review (see Alternatives).
* **Priority 0 is rejected** because its flow would land on the same
  priority as the pass flow and the result would be undefined.
* **Direction.** A rule with a direction is written to the OVN ``Mirror``
  of that direction only; a rule without a direction is written to every
  ``Mirror`` of the tap mirror (one ``Mirror_Rule`` row per ``Mirror``). A
  rule for a direction the tap mirror does not copy is rejected.
* **Relation to security groups** (from the pipeline position of the mirror
  stages): ``OUT`` rules see the traffic the port sends before its security
  groups are evaluated; ``IN`` rules see only the traffic the port actually
  receives. The copies bypass the security groups of the collector port.
* **Duplicates and equal priorities.** A rule identical to an existing one
  of the same tap mirror (same priority, direction and match) is rejected
  with ``409``; this is also the identity OVN uses for a ``Mirror_Rule``.
  Two different rules with the same priority and overlapping matches are
  accepted, as OVN and OVN ACLs do, but which one wins is undefined; the
  documentation tells users to give overlapping rules distinct priorities.
  Deciding whether two arbitrary matches overlap is not cheap, so the only
  simple stricter option is to reject any second rule with the same
  priority and direction; it is listed under Alternatives for reviewers.
* **Mirror types.** Creating a rule on a ``gre`` or ``erspanv1`` tap mirror
  returns ``400`` with a message that names the mirror type. Deleting a tap
  mirror deletes its rules.

REST API Impact
---------------

Create a rule::

  POST /v2.0/taas/tap_mirrors/{tap_mirror_id}/rules
  {
      "rule": {
          "priority": 200,
          "action": "skip",
          "direction": "IN",
          "protocol": "tcp",
          "destination_port_range_min": 443,
          "destination_port_range_max": 443
      }
  }

Response ``201``::

  {
      "rule": {
          "id": "a2b1...",
          "project_id": "3f6c...",
          "priority": 200,
          "action": "skip",
          "direction": "IN",
          "ethertype": null,
          "protocol": "tcp",
          "source_ip_prefix": null,
          "destination_ip_prefix": null,
          "source_port_range_min": null,
          "source_port_range_max": null,
          "destination_port_range_min": 443,
          "destination_port_range_max": 443
      }
  }

``GET .../rules`` returns ``{"rules": [...]}`` and supports filtering and
sorting on ``priority``, ``action``, ``direction``, ``ethertype`` and
``protocol``. ``DELETE`` returns ``204``.

Error codes:

* ``400``: invalid attribute value; rule creation on a ``gre``/``erspanv1``
  mirror; direction not copied by the mirror; port range without an L4
  protocol or with ``min`` greater than ``max``; prefix family mismatch;
  ``priority`` outside 1-32767.
* ``403``: policy.
* ``404``: tap mirror or rule not found, or not visible to the caller.
* ``409``: identical rule exists.

For example, rule creation on a ``gre`` mirror returns ``400`` with the
message ``Tap Mirror <id> of type gre does not support rules``.

The ``tap-mirror-rules`` extension is advertised by the OVN service driver
only when the connected Northbound schema has the ``Mirror_Rule`` table.
On an older OVN the sub-resource is simply absent (``404``), ``gre`` and
``erspanv1`` mirrors work as before, ``lport`` mirrors without rules work
where the OVN version supports them, and clients can detect the feature
through extension discovery instead of a failing request.

If the schema cannot be read, the extension is not advertised either.
Rule changes are applied to OVN asynchronously, so a rule accepted by
the API but missing in OVN could cause unintended mirroring. For example,
a missing catch-all ``skip`` rule could allow more traffic to be copied
than the user intended.

Policies ``create_tap_mirror_rule`` and ``delete_tap_mirror_rule`` default
to ``ADMIN_OR_PROJECT_MEMBER`` and ``get_tap_mirror_rule`` to
``ADMIN_OR_PROJECT_READER``, the same as tap mirrors. A rule always belongs
to the project of its tap mirror; for a cross-project mirror created by an
admin the rules therefore belong to the admin's project and are managed
there.

Data Model Impact
-----------------

A new table ``tap_mirror_rules`` (expand migration only):

* ``id`` (primary key), ``project_id``;
* ``tap_mirror_id``, foreign key to ``tap_mirrors.id`` with
  ``ON DELETE CASCADE``;
* ``priority``, ``action``, ``direction``, ``ethertype``, ``protocol``,
  ``source_ip_prefix``, ``destination_ip_prefix`` and the four port range
  columns.

No existing table changes; no contract migration. The duplicate check is
done in the plugin inside the same transaction that inserts the row.

OVN driver
----------

* ``create_tap_mirror_rule_postcommit`` translates the rule and calls
  ``mirror_rule_add`` for the OVN ``Mirror`` row(s) of the selected
  direction(s); ``delete`` calls ``mirror_rule_del``. The commands and the
  ``Mirror_Rule`` table registration come from ovsdbapp change 950742
  [2]_.
* The driver base class gets ``create/delete_tap_mirror_rule_pre/postcommit``
  hooks that default to no-op; drivers that do not implement them do not
  advertise the extension.
* TaaS has no OVN DB maintenance (sync) task for tap mirrors today, so the
  rules have none either. A maintenance task that reconciles mirrors and
  rules with the Northbound DB is a sensible follow-up but is out of scope
  here.

Clients
-------

openstacksdk gets a ``TapMirrorRule`` resource and the proxy methods
``create/delete/get/tap_mirror_rules``; python-openstackclient the
commands::

  openstack tap mirror rule create <mirror> --priority 200 --action skip \
      --direction IN --protocol tcp --dst-port 443
  openstack tap mirror rule create <mirror> --priority 1 --action skip
  openstack tap mirror rule list <mirror>
  openstack tap mirror rule show <mirror> <rule>
  openstack tap mirror rule delete <mirror> <rule>

with ``--ethertype``, ``--src-ip``, ``--dst-ip``, ``--src-port`` and
``--dst-port`` (``port`` or ``min:max``) for the match.

Security Impact
---------------

* A rule can only narrow what a mirror copies. It can not make a mirror
  copy traffic of a port the caller could not mirror already; who may
  create a mirror is unchanged (same project, or admin).
* Rules are created by the owner of the tap mirror. For an admin-created
  cross-project mirror the project of the mirrored port can not see or
  change the rules, as it can not see the mirror.
* The OVN match expression is generated by the server from validated
  attributes; users never supply an expression, so nothing user-controlled
  reaches the OVN expression parser.
* The copies bypass the security groups, QoS and conntrack of the collector
  port (OVN sends them from ``ls_out_pre_acl`` to port security directly).
  This is the behaviour of lport mirrors with or without rules and is not
  changed here; it is documented in the user guide together with the
  pipeline positions above, so that security teams know that ``OUT`` copies
  are pre-security-group and ``IN`` copies are post-security-group.

Performance Impact
------------------

* Each rule becomes one logical flow per mirrored direction on the logical
  switch of the mirrored port, matched only for packets of that port. A
  rule change makes northd rebuild the flows of that port, comparable to
  an ACL change.
* Rules reduce dataplane load: fewer copies on the overlay and at the
  collector.
* No explicit limit on the number of rules per mirror is proposed. The
  priority range does not bound the rule count, since different matches
  may share a priority. K2 Cloud caps it at 10 per mirror at the product
  layer. A Neutron quota resource ``tap_mirror_rule`` can be added later if
  operators ask for it.

Upgrade Impact
--------------

New table only (expand). The extension is loaded only with the OVN driver
and only when the Northbound schema has ``Mirror_Rule`` (OVN 25.09+), so
an OVN upgrade followed by a neutron-server restart enables it. During a
rolling upgrade of neutron-server, old servers do not know the extension
and return ``404`` for rule calls; existing mirrors are not affected. The
minimum ovsdbapp version in requirements is raised to the release that
contains [2]_.

Not supported in the initial release
------------------------------------

* Updating a rule in place.
* Rules on ``gre`` and ``erspanv1`` mirrors.
* Quotas on rules.
* The OVS driver. It does not implement ``lport`` mirrors, and these rules
  are not implemented for it.

Alternatives
------------

API shape
~~~~~~~~~

During the RFE review it was noted that the ``tap_mirrors`` resource now
carries attributes that only apply to some mirror types (``remote_ip`` and
tunnel ids for ``gre``/``erspanv1``, ``remote_port_id`` and rules for
``lport``) [1]_. Options:

#. **Rules as a sub-resource of tap_mirrors (proposed).** Smallest
   change, reuses the resource the OVN driver implements, one OVN ``Mirror``
   per tap mirror direction. The type specific attributes are validated per
   type (table above) and the documentation lists which ones apply.
#. **A separate resource for lport mirrors** (for example
   ``/taas/port_mirrors`` with ``port_id``, ``remote_port_id``,
   ``directions`` and ``rules``). Cleaner attribute set, but a second
   resource for the same OVN object, a second set of policies and clients,
   and the lport mirror API approved in RFE 2168007 would have to move.
#. **Rules as a separate top-level tap_mirror_filters resource** shared
   by several mirrors (the AWS model). This would require defining how
   groups attach to mirrors, how changes propagate, and how group rules
   interact with per-mirror rules. Reusable groups could be considered in
   a follow-up RFE.

The proposal is option 1. Reusing the tap service and tap flow resources
instead of tap mirrors was suggested on RFE 2168007 and discussed at the
drivers meeting; keeping tap mirrors was found reasonable because the OVN
driver only implements tap mirrors and an OVN ``Mirror`` maps one-to-one to
a tap mirror.

Default deny
~~~~~~~~~~~~

Make a mirror with rules copy only the matching traffic. This is what AWS
and K2 Cloud do. It is implementable by having the driver add a priority 1
``skip`` rule whenever the first rule is created, but it makes the result
depend on hidden rows and changes the meaning of a mirror when its first
rule is added. Users who want it add the catch-all rule themselves; the
user guide shows it as the first example.

Unique priority per direction
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Reject any second rule with the same priority and direction on a mirror,
whatever its match. It removes the undefined case of overlapping matches at
equal priority at the cost of forbidding harmless combinations (two
non-overlapping ``skip`` rules at the same priority). The proposal keeps
OVN's behaviour; reviewers who prefer the stricter rule can ask for it, the
change is a single check in the plugin.

TaaS API cleanup
~~~~~~~~~~~~~~~~

The drivers meeting raised the need to clarify the two TaaS API
models [1]_. Tap services and tap flows are implemented by the agent-based
``TaasRpcDriver``, with the OVS and SR-IOV NIC agent drivers, but not by the
OVN service driver. Tap mirrors of type ``gre`` and ``erspanv1`` are
supported by both the OVS and the OVN drivers [4]_. The ``lport`` type
introduced by RFE 2168007 is currently implemented only by the OVN driver.

When the TaaS plugin is enabled with the OVN driver, its tap service and
tap flow extensions are still advertised although those operations are not
implemented. A cleanup is desirable but is a change of its own, so this
spec only states how it relates to the rules:

* This series documents the current mapping explicitly: an OVS/OVN parity
  table of resources and attributes in the TaaS documentation (also a
  condition of RFE 2168007).
* With lport mirrors available, the OVN driver could implement tap services
  and tap flows natively (one OVN ``Mirror`` of type ``lport`` per tap flow,
  with the tap service port as sink and the flow direction as filter). That
  would give both backends the same API for in-cloud collectors. Whether
  to do this is left to a separate RFE; this spec does not propose to change
  or deprecate the existing tap mirror API.
* The rule attributes defined here are backend neutral (security group
  vocabulary, server-side translation). If tap flows later gain filtering,
  the same sub-resource definition can be attached to them, so the choice
  made in this spec does not pre-empt the cleanup.
* Advertising only the extensions the loaded driver implements, so that
  API discovery tells the truth, is a small independent fix that can be
  done in the same cleanup; the tap service and tap flow extensions with
  the OVN driver are one example.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Chanyeol Yoon (ycy1766)

Work Items
----------

* neutron-lib: ``tap-mirror-rules`` API definition and exceptions; make the
  ``tap-mirror-lport`` definition reject ``remote_ip``/tunnel ids for lport.
* ovsdbapp: ``Mirror_Rule`` table and ``mirror_rule_add/del`` commands
  [2]_, then a release and a requirements bump.
* tap-as-a-service: DB model and migration, plugin validation and match
  translation, OVN driver, driver hooks, conditional extension
  advertisement, policies, documentation, release note, CI job.
* openstacksdk and python-openstackclient support.
* neutron-tempest-plugin: API and scenario tests.

A working implementation exists and was tested end to end on OVN 26.03
with Neutron 2025.1, including a skip rule and a higher priority mirror
rule overriding it; the series will be pushed to Gerrit in the order
ovsdbapp, neutron-lib, tap-as-a-service, openstacksdk,
python-openstackclient.


Dependencies
============

* RFE 2168007 (lport tap mirrors) and its neutron-lib and tap-as-a-service
  changes; the rules change is stacked on them.
* ovsdbapp change 950742 [2]_, merged and released in ovsdbapp 3.0.0.
* OVN 25.09 or later at runtime (Northbound schema with ``Mirror_Rule``).
* openstacksdk and python-openstackclient releases for the client parts.


Testing
=======

* Unit tests: API definition, plugin validation (every rejected case of the
  REST API section), translation to OVN match (every attribute combination
  and the invalid ones), OVN driver calls per direction, DB migration and
  cascade delete, and a Northbound schema without ``Mirror_Rule`` (the
  extension is not advertised, lport mirrors still work).
* Functional tests in ovsdbapp for ``mirror_rule_add/del`` against a real
  OVN Northbound schema (skipped when the schema has no ``Mirror_Rule``),
  part of [2]_.
* TaaS has no functional test suite today. A small functional module for
  the OVN driver, based on the OVN functional base class of neutron and
  running against a real ``ovn-northd``, is part of the series; it checks
  the ``Mirror`` and ``Mirror_Rule`` rows written for each direction and
  their removal with the rule and with the mirror.
* The ``neutron-tempest-plugin-tap-as-a-service-ovn`` job currently builds
  OVN ``branch-24.03``, which has no lport mirror. A job variant building
  OVN 25.09 or later is added; the existing job keeps covering gre/erspan
  on the older OVN. The new job runs:

  * API tests: create, list, show and delete rules; rejected cases (rule on
    a gre mirror, priority 0, a direction the mirror does not copy, port
    range without an L4 protocol, prefix family mismatch, identical rule);
    rules deleted with their tap mirror; the attribute checks of the mirror
    type table.
  * Scenario tests with a collector VM running ``tcpdump``: a ``skip`` rule
    removes a flow from the collector, a higher priority ``mirror`` rule
    brings it back, an ``IN`` only rule leaves the outbound traffic
    untouched, and a catch-all ``skip`` plus one ``mirror`` rule delivers
    only the listed traffic.

  The rule tests are skipped when the ``tap-mirror-rules`` extension is not
  advertised, so they are safe on the older job too.


Documentation Impact
====================

* TaaS user guide: rules, default allow and the catch-all pattern, rules
  only on lport mirrors, OVN version requirement, and where lport mirrors
  copy traffic: ``OUT`` copies before and ``IN`` copies after the security
  groups of the mirrored port, and the copies bypass the security groups,
  QoS and conntrack of the collector port.
* TaaS API reference (new sub-resource, mirror type attribute table) and a
  release note.
* OVS/OVN feature parity table for TaaS resources and attributes, requested
  at the RFE review.


References
==========

.. [1] Neutron drivers meeting, 2026-10-02:
   https://meetings.opendev.org/meetings/neutron_drivers/2026/neutron_drivers.2026-10-02-13.06.log.html
.. [2] ovsdbapp: nb: add support for mirror-rules in mirrors:
   https://review.opendev.org/c/openstack/ovsdbapp/+/950742
.. [3] ovn-northd mirror logical flows (``build_mirror_lflows``,
   ``OVN_LPORT_MIRROR_OFFSET``):
   https://github.com/ovn-org/ovn/blob/main/northd/northd.c
.. [4] TaaS documentation, tap mirrors under the hood:
   https://opendev.org/openstack/tap-as-a-service/src/branch/master/doc/source/tap_mirrors_under_the_hood.rst

* RFE for lport tap mirrors: https://bugs.launchpad.net/neutron/+bug/2168007
* OVN Mirror_Rule: https://github.com/ovn-org/ovn/commit/3dd8f36b85b3e648bc506839aa62e81b21c41e30
* ovn-nb(5), ``Mirror`` and ``Mirror_Rule`` tables:
  https://www.ovn.org/support/dist-docs/ovn-nb.5.html
* Neutron QoS rules API (model for the sub-resource):
  https://docs.openstack.org/api-ref/network/v2/index.html#qos-rules
* AWS VPC Traffic Mirroring filters:
  https://docs.aws.amazon.com/vpc/latest/mirroring/traffic-mirroring-filter.html
* K2 Cloud Traffic Mirroring:
  https://docs.k2.cloud/en/services/interconnect/traffic_mirroring.html
