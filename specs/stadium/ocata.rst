..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================
Ocata - Stadium health
======================

For details about criteria please check Stadium documents.

Meeting the minimum bar
=======================

.. _Integration:

Neutron integration
-------------------

The most basic criteria for neutron integration is the adoption of neutron-lib,
with periodic checking. Going forward, the project team must demonstrate steps
towards 100% migration over neutron-lib imports. No red flags in architectural
and design decisions when building networking components must be present. API
definition and reference documentation will need to be made available in the
neutron-lib repo.

Acceptance test
~~~~~~~~~~~~~~~

Check that the Grafana dashboard displays correctly the periodic job, and
that artifacts in the periodic output shows at least one successful run.

* http://logs.openstack.org/periodic/
* http://grafana.openstack.org/

Assessment
++++++++++

  * networking-bagpipe: OK.
  * networking-odl: OK.
  * networking-bgpvpn: OK.
  * networking-midonet: OK.
  * neutron-dynamic-routing: OK.
  * neutron-fwaas: OK.
  * networking-ovn: OK.
  * networking-sfc: OK.

.. _Docs:

Documentation
-------------

Documentation (developer, deployer and end-user as required), release notes and
API reference (where applicable).

Acceptance test
~~~~~~~~~~~~~~~

Check for a working link to the documentation. Check for a working link to release
notes for the latest release (Newton). Check for a working link to API reference.

* http://docs.openstack.org/developer/openstack-projects.html
* https://releases.openstack.org/newton/index.html
* http://developer.openstack.org/api-ref/networking/

Assessment
++++++++++

  * networking-bagpipe: OK.
  * networking-odl: OK.
  * networking-bgpvpn: needs API documentation.
  * networking-midonet: needs API documentation.
  * neutron-dynamic-routing: needs API documentation.
  * neutron-fwaas: OK.
  * networking-ovn: OK.
  * networking-sfc: needs API documentation.

.. _CI:

Continuous Integration
----------------------

Grafana dashboard must be available; unit (both py2x and py3x) and tempest (API
and scenario) jobs must be gating and stable. This means that regex expressions to skip
basic networking tests will not be tolerated. For projects that have DB migrations,
sync tests must be available.

Acceptance test
~~~~~~~~~~~~~~~

Check for Grafana dashboard presence. Check for passing unit tests. Check for
gating tempest tests. Check for scenario tests. Check for DB migration tests.

* http://grafana.openstack.org/
* http://status.openstack.org/openstack-health/

Assessment
++++++++++

  * networking-bagpipe: OK.
  * networking-odl: OK.
  * networking-bgpvpn: OK.
  * networking-midonet: OK.
  * neutron-dynamic-routing: needs scenario tests. Needs DB/sync validation.
  * neutron-fwaas: OK.
  * networking-ovn: OK.
  * networking-sfc: OK.

.. _Release:

Release footprint
-----------------

Semver, upper-constraints, and bot proposal integrated are a must. Release tarballs and
release notes must be available on releases.openstack.org. There should be at least one
published deliverable for each of the supported releases.

Acceptance test
~~~~~~~~~~~~~~~

Look for tarball and release note link available on releases.openstack.org. Check for
the version. Check for presence of project on requirement repo. Check for use of
upper-constraints.

* https://releases.openstack.org/newton/index.html
* https://releases.openstack.org/independent.html
* https://github.com/openstack/requirements/blob/master/projects.txt
* http://codesearch.openstack.org/?q=UPPER_CONSTRAINTS_FILE&i=nope&files=&repos=

Assessment
++++++++++

  * networking-bagpipe: OK.
  * networking-odl: OK.
  * networking-bgpvpn: OK.
  * networking-midonet: OK.
  * neutron-dynamic-routing: OK.
  * neutron-fwaas: OK.
  * networking-ovn: OK.
  * networking-sfc: OK.

.. _Maintenance:

Stable backports
----------------

Stable branches aligned with neutron branches are a must.

Acceptance test
~~~~~~~~~~~~~~~

Check for supported stable branches (mitaka and newton). Check that tox_install.sh pins
to the right branch.

* http://git.openstack.org/cgit/openstack/<project-name>/?h=stable%2Fnewton
* http://codesearch.openstack.org/?q=tox_install.sh&i=nope&files=&repos=

Assessment
++++++++++

  * networking-bagpipe: OK.
  * networking-odl: OK.
  * networking-bgpvpn: OK.
  * networking-midonet: OK.
  * neutron-dynamic-routing: OK.
  * neutron-fwaas: OK.
  * networking-ovn: OK.
  * networking-sfc: OK.

.. _CLI:

Client library
--------------

For projects that need client extensions OSC bindings must be present.

Acceptance test
~~~~~~~~~~~~~~~

Check for presence of OSC bindings in python-neutronclient.

* https://github.com/openstack/python-neutronclient/tree/master/neutronclient/osc/v2
* https://github.com/openstack/python-neutronclient/blob/master/setup.cfg

Assessment
++++++++++

  * networking-bagpipe: N/A.
  * networking-odl: N/A.
  * networking-bgpvpn: needs porting to python-neutronclient.
  * networking-midonet: needs porting to python-neutronclient.
  * neutron-dynamic-routing: needs porting to python-neutronclient.
  * neutron-fwaas: OK.
  * networking-ovn: N/A.
  * networking-sfc: needs porting to python-neutronclient.

Summary
=======

+-------------------------------------------------------------------+---------------+---------------+---------------+---------------+---------------+---------------+
| Project                                                           | Integration_  | Docs_         | CI_           | Release_      | Maintenance_  | CLI_          |
+===================================================================+===============+===============+===============+===============+===============+===============+
| `networking-bagpipe <./ocata/networking-bagpipe.html>`_           | Met           | Met           | Met           | Met           | Met           | N/A           |
+-------------------------------------------------------------------+---------------+---------------+---------------+---------------+---------------+---------------+
| `networking-odl <./ocata/networking-odl.html>`_                   | Met           | Met           | Met           | Met           | Met           | N/A           |
+-------------------------------------------------------------------+---------------+---------------+---------------+---------------+---------------+---------------+
| `networking-bgpvpn <./ocata/networking-bgpvpn.html>`_             | Met           | Needs work    | Met           | Met           | Met           | Needs work    |
+-------------------------------------------------------------------+---------------+---------------+---------------+---------------+---------------+---------------+
| `networking-midonet <./ocata/networking-midonet.html>`_           | Met           | Needs work    | Met           | Met           | Met           | Needs work    |
+-------------------------------------------------------------------+---------------+---------------+---------------+---------------+---------------+---------------+
| `neutron-dynamic-routing <./ocata/neutron-dynamic-routing.html>`_ | Met           | Needs work    | Needs work    | Met           | Met           | Needs work    |
+-------------------------------------------------------------------+---------------+---------------+---------------+---------------+---------------+---------------+
| `neutron-fwaas <./ocata/neutron-fwaas.html>`_                     | Met           | Met           | Met           | Met           | Met           | Met           |
+-------------------------------------------------------------------+---------------+---------------+---------------+---------------+---------------+---------------+
| `networking-ovn <./ocata/networking-ovn.html>`_                   | Met           | Met           | Met           | Met           | Met           | N/A           |
+-------------------------------------------------------------------+---------------+---------------+---------------+---------------+---------------+---------------+
| `networking-sfc <./ocata/networking-sfc.html>`_                   | Met           | Needs work    | Met           | Met           | Met           | Needs work    |
+-------------------------------------------------------------------+---------------+---------------+---------------+---------------+---------------+---------------+
| `neutron-vpnaas <./ocata/neutron-vpnaas.html>`_ (*)               | Met           | Needs work    | Needs work    | Met           | Needs work    | Needs work    |
+-------------------------------------------------------------------+---------------+---------------+---------------+---------------+---------------+---------------+
| `networking-l2gw <./ocata/networking-l2gw.html>`_ (*)             | Needs work    | Needs work    | Needs work    | Needs work    | Needs work    | Needs work    |
+-------------------------------------------------------------------+---------------+---------------+---------------+---------------+---------------+---------------+
| `networking-calico <./ocata/networking-calico.html>`_ (*)         | Needs work    | Needs work    | Needs work    | Needs work    | Needs work    | N/A           |
+-------------------------------------------------------------------+---------------+---------------+---------------+---------------+---------------+---------------+
| `networking-onos <./ocata/networking-onos.html>`_ (*)             | Needs work    | Needs work    | Needs work    | Needs work    | Needs work    | N/A           |
+-------------------------------------------------------------------+---------------+---------------+---------------+---------------+---------------+---------------+

(*) To re-apply for inclusion in Pike or future releases.

How Reconcile API and client bindings
=====================================

One key step of the stadium evolution effort is to consolidate API
definitions/documentation and client bindings into neutron-lib and
`python-neutronclient <http://git.openstack.org/cgit/openstack/python-neutronclient/tree/doc/source/devref/transition_to_osc.rst#n152>`_
respectively. If all of the criteria are met and the outstanding
ones are Docs and CLI, then it is clear that in order to complete
the effort end-to-end and keep the project in the neutron governance,
a project team is left to contribute the API definition, its API
reference documentation and the client bindings to neutron-lib and
python-neutronclient respectively.

In order to address this need, please follow these steps:

* Propose a patch to neutron-lib that includes API definition and
  API documentation. Use topic `'stadium-implosion' <https://review.openstack.org/#/q/topic:stadium-implosion+status:open>`_.
  You can break down API definition and API reference into separate
  patches, but this will lengthen the merge process. Please use the
  same topic for both patches, in case you break this down.

* Propose a patch to python-neutronclient that includes OSC commands,
  CLI documentation as well as API bindings. Use topic 'stadium-implosion'
  as hinted above.  Make the client patch depend on the neutron-lib patch
  and *not* vice versa. That is because, should the API patch crash and
  burn for whatever reason, we do not need to revert the client patch
  prior to a client release.

Once all of the outstanding 'stadium-implosion' patches have merged
a new release of neutron-lib and python-neutronclient is going to
be cut. At this point, when the Bot Proposal change merges, the API
definitions can be used in the subproject, and that seals the completion
of the stadium effort for Ocata.

NOTE: Merging of these patches are pending confirmation that all
the outstanding work has been addressed, but review will proceed
nonetheless.
