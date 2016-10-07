..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Networking-bagpipe Scorecard
============================

Neutron integration
-------------------

.. _N0:

* N0. Does the project use the Neutron REST API or relies on proprietary backends?

  No. The repo relies on https://github.com/Orange-OpenSource/bagpipe-bgp, which
  is released under the Apache license.

.. _N1:

* N1. Does the project integrate/use neutron-lib?

  Yes. So far the import ratio is ~13%. The neutron-lib periodic integration is
  not available, but in progress.

  * https://review.openstack.org/#/c/385337/
  * https://review.openstack.org/#/c/385445/

.. _N2:

* N2. Do project members actively contribute to help neutron-lib achieve its
  goal?

  No.

.. _N3:

* N3. Do project members collaborate with the core team to enable subprojects
  to loosely integrate with the Neutron core platform by helping with the definition
  of modular interfaces?

  The team proactively adopted L2 extensions where required, as shown by commits
  below.

.. _N4:

* N4. How does the project provide networking services? Does it use modular interfaces
  as provided by the core platform?

  Yes.

  * https://github.com/openstack/networking-bgpvpn/blob/753fa2410639dca3957f0bffbb2e632181f00f77/setup.cfg#L42
  * https://github.com/openstack/networking-bagpipe/blob/382a6a6930bb77764b40c65724b53ab1901132ba/setup.cfg#L58

.. _N5:

* N5. If the project provides new API extensions, have API extensions been discussed
  and accepted by the Neutron drivers team? Please provide links to API specs, if
  required.

  networking-bagpipe contains (a) a driver for networking-bgpvpn, and (b) an ML2 mech
  driver to deliver Neutron L2 that (1) is not necessary for the BGPVPN part and (2)
  does not require an other ML2 mech driver to deliver Neutron ML2.

Documentation
-------------

.. _D1:

* D1. Does the project have a doc tox target, functional and continuously
  working? Provide proof (links to logs.openstack.org).

  Yes.

  * http://docs.openstack.org/developer/networking-bagpipe/

.. _D2:

* D2. If the project provide API extensions, does the project have an
  api-ref tox target, functional and continously working? Provide proof
  (links to logs.openstack.org).

  No.

.. _D3:

* D3. Does the project have a releasenotes tox target, functional and
  continously working? Provide proof.

  No.

  * https://review.openstack.org/#/c/394423
  * https://review.openstack.org/#/c/394339

.. _D4:

* D4. Describe the types of documentation available: developer, end user,
  administrator, deployer.

  Just deployer documentation.


Continuous Integration
----------------------

.. _C1:

* C1. Does the project have a Grafana dashboard showing historical trends of
  all the jobs available? Provide proof (links to grafana.openstack.org).

  No.

  * https://review.openstack.org/#/c/385337/

.. _C2:

* C2. Does the project have CI for unit coverage? Provide proof (links to
  logs.openstack.org)

  Yes.

  * http://status.openstack.org/openstack-health/#/g/project/openstack~2Fnetworking-bagpipe

.. _C3:

* C3. Does the project have CI for functional coverage? If so, does it include
  DB migration and sync validation?

  No, and no DB migration and sync validation even though the mech driver has DB
  migrations (last one available for the Liberty release). The roadmap is to get
  rid of DB models, and instead use the VXLAN type driver and derive BGPVPN
  identifiers from that, in a way similar to IETF draft `draft-ietf-bess-evpn-overlay <https://tools.ietf.org/html/draft-ietf-bess-evpn-overlay-04#section-5.1.2.1>`_.

.. _C4:

* C4. Does the project have CI for fullstack coverage?

  No.

.. _C5:

* C5. Does the project have CI for Tempest coverage? If so, specify nature
  (API and/or Scenario).

  Non-voting and experimental.

  * http://logs.openstack.org/97/392097/1/experimental/gate-tempest-dsvm-networking-bagpipe-full/9b4cafb/
  * http://logs.openstack.org/97/392097/1/experimental/gate-tempest-dsvm-networking-bgpvpn-bagpipe/961fd47/

.. _C6:

* C6. How does a project validate upgrades on a continuous basis? Does
  the project require or support CI for Grenade coverage?

  No evidence of Grenade testing.

.. _C7:

* C7. Does the project provide multinode CI?

  No.

.. _C8:

* C8. Does the project support Python 3.x? Provide proof.

  Yes.

  * http://logs.openstack.org/97/392097/1/experimental/gate-networking-bagpipe-python35-db/b26d70c/testr_results.html.gz


Release footprint
-----------------

.. _R1:

* R1. Does the project adopt semver?

  Yes.

.. _R2:

* R2. Does the project have release deliverables? Provide proof as available
  in the `release repo <http://git.openstack.org/cgit/openstack/releases/tree/>`_.

  Yes.

  * https://github.com/openstack/releases/blob/master/deliverables/_independent/networking-bagpipe.yaml

.. _R3:

* R3. Does the project use upper-constraints?

  Yes.

  * https://github.com/openstack/networking-bagpipe/blob/master/tox.ini#L8

.. _R4:

* Does the project integrate with OpenStack Proposal Bot for requirements updates?

  Yes.

  * https://github.com/openstack/requirements/commit/3a8bcda570d1dc129518ca8c10752f837af07a39


Stable backports
----------------

.. _S1:

* S1. Does the project have stable branches and/or tags? Provide history of
  backports.

  Yes. Stable liberty, mitaka and newton look aligned with Neutron's.


Client library
--------------

.. _L1:

* L1. If the project requires a client library, how does it implement CLI and
  API bindings?

  No.


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
| D3_ |    N    |
+---------------+
| D4_ |    Y    |
+---------------+
| C1_ |    N    |
+---------------+
| C2_ |    Y    |
+---------------+
| C3_ |    N    |
+---------------+
| C4_ |    N    |
+---------------+
| C5_ |    N    |
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

Final remarks: neutron-lib integration, more documentation and better coverage
are the main gaps identified in this project.
