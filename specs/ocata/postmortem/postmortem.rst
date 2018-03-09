..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

Ocata Postmortem documentation
==============================

.. contents::

Release Metrics
---------------

+------------------------------------------------+
| Release Metrics                                |
+===============================+================+
| Blueprints targeted           |             15 |
+-------------------------------+----------------+
| Blueprints/RFE implemented    |              4 |
+-------------------------------+----------------+
| Bug reports submitted         |            556 |
+-------------------------------+----------------+
| Bug reports closed            |            334 |
+-------------------------------+----------------+
| Fixes released                |            225 |
+-------------------------------+----------------+
| Incomplete reports            |             29 |
+-------------------------------+----------------+
| RFEs submitted                |             22 |
+-------------------------------+----------------+

* Bug reports submitted: reports filed since Sep-16-2016 (Ocata starts)
* Bug reports closed: reports marked released, committed, invalid or wontfix
* Fixes released: marked released/committed
* Incomplete reports: marked incomplete

.. note:: Metrics accurate at the time of writing.

Blueprints
----------

Adopt oslo.versionedobjects for database interactions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: ihar-hrachyshka
* Link: https://blueprints.launchpad.net/neutron/+spec/adopt-oslo-versioned-objects-for-db

  * CLI support: N/A
  * Server/Agent support: Partially implemented
  * Testing coverage: unit, API, functional coverage
  * Documentation: developer documentation (https://review.openstack.org/#/c/336518/)
  * Advanced/Sub-project support: out of scope
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: N/A

Split neutron into base library and servers/agents
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete (the process to develop and consume neutron-lib).
* Assignee: boden
* Link: https://blueprints.launchpad.net/neutron/+spec/neutron-lib

  * CLI support: N/A
  * Server/Agent support: Complete
  * Testing coverage: N/A
  * Documentation: Complete
  * Advanced/Sub-project support: Complete
  * Other Projects support:
  * OpenStack Infra support: Complete
  * DevStack support: N/A
  * Horizon Support: N/A

Upgrade controllers with no API downtime
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred
* Assignee: ihar-hrachyshka
* Link: https://blueprints.launchpad.net/neutron/+spec/online-upgrades

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

Use push style notifications for all server->agent information
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: kevinbenton
* Link: https://blueprints.launchpad.net/neutron/+spec/push-notifications
* FFE Status: Granted

  * CLI support: N/A
  * Server/Agent support: Server support partially merged. L2 agent
    support pending reviews. In particular, server generates
    notifications for all L2 components (port, networks, etc) and
    security groups. The only remaining server component waiting to merge
    is a way to query for multiple OVO objects.
  * Testing coverage: code has UTs and is executed as part of tempest runs.
  * Documentation: N/A
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: N/A

Support Routed Networks in Neutron
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: minsel
* Link: https://blueprints.launchpad.net/neutron/+spec/routed-networks
* FFE status: Granted

  * CLI support: Complete
  * Server/Agent support: Complete
  * Testing coverage: Coverage, except scenario coverage.
  * Documentation: Complete
  * Advanced/Sub-project support: N/A
  * Other Projects support: We got all the functionality we needed from
    Nova to implement the updating of routed networks segments IPv4
    addresses inventories in the placement API. The Nova scheduler
    will not be able to use that information yet to place instances.
    Nova will implement that during Pike.
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: N/A

Support agentless driver in neutron-dynamic-routing
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred
* Assignee: yuyangbj
* Link: https://blueprints.launchpad.net/neutron/+spec/agentless-driver

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

Use the new enginefacade from oslo_db
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Ongoing
* Assignee: akamyshnikova
* Link: https://blueprints.launchpad.net/neutron/+spec/enginefacade-switch

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

Firewall as a Service API 2.0
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred
* Assignee: nate-johnston
* Link: https://blueprints.launchpad.net/neutron/+spec/fwaas-api-2.0

  * CLI support: Complete
  * Server/Agent support: missing two neutron server side patches
  * Testing coverage: Complete
  * Documentation: Complete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: Complete
  * DevStack support: Complete
  * Horizon Support: Incomplete

Moving to Keystone v3
~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: smigiel-dariusz
* Link: https://blueprints.launchpad.net/neutron/+spec/keystone-v3

  * CLI support: Complete
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Complete
  * Advanced/Sub-project support: Complete
  * Other Projects support: N/A
  * OpenStack Infra support: Complete
  * DevStack support: Complete
  * Horizon Support: N/A

API for l2 agent extensions (+ovs flow management)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred (spec under discussion)
* Assignee: david-shaughnessy
* Link: https://blueprints.launchpad.net/neutron/+spec/l2-api-extensions

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

nova portbinding information for live migration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred (spec merged).
* Assignee: anindita-das
* Link: https://blueprints.launchpad.net/neutron/+spec/live-migration-portbinding

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

Neutron in-tree API reference
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Ongoing
* Assignee: amotoki
* Link: https://blueprints.launchpad.net/neutron/+spec/neutron-in-tree-api-ref

  * CLI support: N/A
  * Server/Agent support: N/A
  * Testing coverage: N/A
  * Documentation: Complete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: Complete
  * DevStack support:
  * Horizon Support:

Port data plane status
~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred (spec approved).
* Assignee: cgoncalves
* Link: https://blueprints.launchpad.net/neutron/+spec/port-data-plane-status

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

Security-group Logging
~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred (spec in progress).
* Assignee: y-furukawa-2
* Link: https://blueprints.launchpad.net/neutron/+spec/security-group-logging

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

Diagnostics of Neutron components
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred
* Assignee: boden
* Link: https://blueprints.launchpad.net/neutron/+spec/troubleshooting

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:


RFEs
----

RFE: Pure Python driven Linux network configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: omer-anson
* Link: https://bugs.launchpad.net/neutron/+bug/1492714

  * CLI support: N/A
  * Server/Agent support: Only partially complete
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support: Incomplete
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: N/A

[RFE] Transition neutron CLI from python-neutronclient to python-openstackclient
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: amotoki
* Link: https://bugs.launchpad.net/neutron/+bug/1521291

  * CLI support: Incomplete
  * Server/Agent support: N/A
  * Testing coverage: N/A
  * Documentation: Incomplete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: Complete
  * Horizon Support: Incomplete

[RFE] No notification on floating ip status change
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: boden
* Link: https://bugs.launchpad.net/neutron/+bug/1593793

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

relocate model definitions
~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: manjeet-s-bhatia
* Link: https://bugs.launchpad.net/neutron/+bug/1597913

  * CLI support: N/A
  * Server/Agent support: N/A
  * Testing coverage: N/A
  * Documentation: https://github.com/openstack/neutron/blob/master/doc/source/devref/db_models.rst
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: N/A

Support for DSCP marking in Linuxbridge L2 agent
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status:
* Assignee: slaweq
* Link: https://bugs.launchpad.net/neutron/+bug/1644369

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:
