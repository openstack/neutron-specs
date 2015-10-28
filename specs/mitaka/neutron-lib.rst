..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Neutron-Lib Decomposition
==========================================

https://blueprints.launchpad.net/neutron/+spec/neutron-lib

This proposal is about stabilizing some of the common interfaces used
by Neutron's child repos, both the advanced services and decomposed plugins.

Problem Description
===================

Part of the goal of splitting functionality and vendor code out of neutron
was to increase focus on core neutron, getting it stable, and increasing the
velocity of making changes. Since the services repos and decomposed
plugins/drivers are both importing and using internal neutron interfaces,
unless we run existing tests against the universe of all of this code,
it becomes difficult to re-factor or make simple changes without breaking
these newly separated projects and plugins.

In addition, there is a certain amount of magic involved in the implicit
dependency that neutron is installed, because of the openstack/requirements
rules regarding global dependencies, and neutron not being on that list.
We end up having to assume neutron is present and are unable to check that
the version present is compatible.

The requirements driving this change are:

- A set of neutron methods which are considered "stable", and which will
  not be removed or have their method signatures changed without a long-term
  deprecation plan.
- This is not a mechanical split.  For each module, we must determine if the
  current interface is what we want to support, or if it needs to change. And
  for each module, it will likely need strict interface tests.
- At the end of the "split" process, there is no co-gate.
- A library in pypi.
- Addition of this library to the global requirements list.
- A versioned reference to the library in various projects requirements.txt.

Proposed Change
===============

This proposal is to split neutron into a neutron-lib of reusable library
code and neutron core (servers, agents, plugins, drivers.) Part of creating
neutron-lib would involve creating rigid interface tests for exposed
library methods.

This library will live in the new repo neutron-lib, and will be published
to pypi.

This would *not* be a mechanical split, but rather each piece of code
moved into the library would need to satisfy several criteria:

- Is this a reusable module that makes sense in a general library?
- Does it have interface tests, or do they need to be added?
- Is the current interface good for a library, or does it need to be
  re-factored?

Some trivial items, like the exceptions module, can move over directly
as part of this blueprint, however even trivial code that is not shared by
anyone should not move.  Most modules with any complexity at all will
require separate blueprints and careful consideration.

This blueprint can be considered complete when:

* A neutron-lib repo with testing framework and Jenkins exists.

* A stable version of neutron-lib has been released to pypi.

* Neutron, neutron-lbaas, neutron-fwaas, neutron-vpnaas depend on neutron-lib
  via requirements.txt, and NONE OF THEM IMPORT EACH OTHER.

* Api and functional tests are fully separate between the above four repos.
  In other words, there is NO co-gate, and interface breakages are enforced
  via tests in neutron-lib.

* Tests will no longer share code between repos. Unit test base classes will
  be repo specific, initially likely dup'ed, and diverge on their own.
  Any root functionality that the base classes require from neutron will
  be a candidate for inclusion in neutron-lib or re-factoring.

References
==========

* https://etherpad.openstack.org/p/neutron-lib-decomposition

* Neutron project repos
  - https://github.com/openstack/governance/blob/master/reference/projects.yaml#L81

* tempest-lib:
  - https://blueprints.launchpad.net/tempest/+spec/tempest-library
  - https://github.com/openstack/tempest-lib

* cinder:
  - https://review.openstack.org/#/c/153673/
  - https://github.com/openstack/os-brick

* ironic-lib:
  - https://review.openstack.org/#/c/157757/

* Depends-On:
  - http://lists.openstack.org/pipermail/openstack-dev/2015-February/056515.html
