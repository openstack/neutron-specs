..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Networking-midonet Scorecard
============================

Neutron integration
-------------------

.. _N0:

* N0. Does the project use the Neutron REST API or relies on proprietary backends?

  No.

.. _N1:

* N1. Does the project integrate/use neutron-lib?

  Yes. Of the total ~200 neutron related imports, neutron-lib is imported
  roughly ~20% of the time. The project has a periodic job against
  master neutron-lib changes.

  * http://status.openstack.org/openstack-health/#/job/periodic-networking-midonet-py35-with-neutron-lib-master

.. _N2:

* N2. Do project members actively contribute to help neutron-lib achieve its
  goal?

  No, besides occasional review.

.. _N3:

* N3. Do project members collaborate with the core team to enable subprojects
  to loosely integrate with the Neutron core platform by helping with the definition
  of modular interfaces?

  Yes, on some areas like QoS.

.. _N4:

* N4. How does the project provide networking services? Does it use modular interfaces
  as provided by the core platform?

  It provides ML2 driver and a set of service plugins including L3, which communicate
  with midonet using via midonet REST API. Midonet and its agents provide networking
  services accordingly. Optionally it can be configured to work with neutron agents
  (like neutron dhcp/metadata agents) at the time of writing it also provides monolithic
  core plugins but they are planned to be replaced by the ML2 driver.

.. _N5:

* N5. If the project provides new API extensions, have API extensions been discussed
  and accepted by the Neutron drivers team? Please provide links to API specs, if
  required.

  It has several extensions: `agent-management <http://docs.openstack.org/developer/networking-midonet/specs/kilo/agent_membership.html>`_
  is planned to be deprecated and removed; `bgp-speaker-router-insertion <http://docs.openstack.org/developer/networking-midonet/specs/mitaka/bgp-speaker-router-insertion.html>`_
  is an extension to neutron-dynamic-routing. It is being tracked by RFE
  https://bugs.launchpad.net/neutron/+bug/1583184; `router-interface-fip <http://docs.openstack.org/developer/networking-midonet/specs/mitaka/router-interface-fip.html>`_.
  It is being tracked by RFE https://bugs.launchpad.net/neutron/+bug/1566191.
  `logging-resource <http://docs.openstack.org/developer/networking-midonet/specs/mitaka/logging-API-for-firewall-rules.html>`_
  is slightly matching spec proposal https://review.openstack.org/#/c/203509; finally,
  `gateway-device <http://docs.openstack.org/developer/networking-midonet/specs/kilo/device_management.html>`_
  which may match L2GW's l2-border-gateway api instead. None of these extensions have been
  reviewed and accepted by the Neutron drivers team.

  * http://docs.openstack.org/developer/networking-midonet/specs/index.html

Documentation
-------------

.. _D1:

* D1. Does the project have a doc tox target, functional and continuously
  working? Provide proof (links to logs.openstack.org).

  Yes.

  * http://docs.openstack.org/developer/networking-midonet/

.. _D2:

* D2. If the project provide API extensions, does the project have an
  api-ref tox target, functional and continously working? Provide proof
  (links to logs.openstack.org).

  No.

.. _D3:

* D3. Does the project have a releasenotes tox target, functional and
  continously working? Provide proof.

  Yes.

  * http://docs.openstack.org/releasenotes/networking-midonet/

.. _D4:

* D4. Describe the types of documentation available: developer, end user,
  administrator, deployer.

  Developer and admin documentation is available. End user documentation
  for client extensions does not seem to be available.

  * http://docs.openstack.org/developer/networking-midonet/


Continuous Integration
----------------------

.. _C1:

* C1. Does the project have a Grafana dashboard showing historical trends of
  all the jobs available? Provide proof (links to grafana.openstack.org).

  Yes.

  * http://grafana.openstack.org/dashboard/db/networking-midonet-failure-rate

.. _C2:

* C2. Does the project have CI for unit coverage? Provide proof (links to
  logs.openstack.org)

  Yes.

  * http://status.openstack.org/openstack-health/#/g/project/openstack~2Fnetworking-midonet

.. _C3:

* C3. Does the project have CI for functional coverage? If so, does it include
  DB migration and sync validation?

  No, but DB migration and validation is achieved via unit testing job.

.. _C4:

* C4. Does the project have CI for fullstack coverage?

  No.

.. _C5:

* C5. Does the project have CI for Tempest coverage? If so, specify nature
  (API and/or Scenario).

  Yes.

  * http://status.openstack.org/openstack-health/#/job/gate-tempest-dsvm-networking-midonet-v2

.. _C6:

* C6. Does the project require CI for Grenade coverage?

  No.

.. _C7:

* C7. Does the project provide multinode CI?

  No.

.. _C8:

* C8. Does the project support Python 3.x? Provide proof.

  Yes.

  * http://status.openstack.org/openstack-health/#/job/gate-networking-midonet-python35-db


Release footprint
-----------------

.. _R1:

* R1. Does the project adopt semver?

Yes.

.. _R2:

* R2. Does the project have release deliverables? Provide proof as available
  in the `release repo <http://git.openstack.org/cgit/openstack/releases/tree/>`_.

  Yes.

  * http://git.openstack.org/cgit/openstack/releases/tree/deliverables/_independent/networking-midonet.yaml

.. _R3:

* R3. Does the project use upper-constraints?

  Yes.

  * https://github.com/openstack/networking-midonet/blob/master/tox.ini#L10

.. _R4:

* Does the project integrate with OpenStack Proposal Bot for requirements updates?

  Yes.

  * https://github.com/openstack/requirements/commit/6f6bf9bfb70e22141041ce61f17c932c9c110d90


Stable backports
----------------

.. _S1:

* S1. Does the project have stable branches and/or tags? Provide history of
  backports.

  Yes.

  * https://review.openstack.org/#/q/project:openstack/networking-midonet+branch:stable/mitaka
  * https://review.openstack.org/#/q/project:openstack/networking-midonet+branch:stable/liberty

Client library
--------------

.. _L1:

* L1. If the project requires a client library, how does it implement CLI and
  API bindings?

  There are neutronclient extensions but no OSC mapping.


Scorecard
---------

+---------------+
| Scorecard     |
+===============+
| N0_ |    N    |
+---------------+
| N1_ |    Y    |
+---------------+
| N2_ |    N    |
+---------------+
| N3_ |    Y    |
+---------------+
| N4_ |    Y    |
+---------------+
| N5_ |    N    |
+---------------+
| D1_ |    Y    |
+---------------+
| D2_ |    N    |
+---------------+
| D3_ |    Y    |
+---------------+
| D4_ |    Y    |
+---------------+
| C1_ |    Y    |
+---------------+
| C2_ |    Y    |
+---------------+
| C3_ |    N    |
+---------------+
| C4_ |    Y    |
+---------------+
| C5_ |    Y    |
+---------------+
| C6_ |    N    |
+---------------+
| C7_ |    N    |
+---------------+
| C8_ |    Y    |
+---------------+
| R1_ |    Y    |
+---------------+
| R2_ |    Y    |
+---------------+
| R3_ |    Y    |
+---------------+
| R4_ |    Y    |
+---------------+
| S1_ |    Y    |
+-----+---------+
| L1_ |    N    |
+-----+---------+

Final remarks: overall networking-midonet is well managed. Its scope is a lot
wider than other subprojects as it covers almost the entirety of the networking
spectrum that Neutron provides. Some may consider networking-midonet a lot
closer to Dragonflow and Astara in terms of scope than networking-ovn or
neutron-dynamic-routing to name a few examples. Gaps in API documentation,
specs approval and client mappings will need to be addressed.
