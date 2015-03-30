..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Migration from Nova-network to Neutron
======================================
https://blueprints.launchpad.net/neutron/+spec/migration-from-nova-net

This blueprint is intended to set expectations for the proposed way to
migrate OpenStack deployments from nova-network to neutron, it will describe
the end-user/deployer impact, starting/end points of it.


Problem Description
===================

A few users of nova-network have expressed interest in having a
migration tool to move to Neutron.  This will require additional
changes to both nova and neutron to support the migration.  This
document will describe the overall process and the features required
in both neutron and nova.

Proposed Change
===============

Impact/Limitations
------------------

It is expected that individual deployments will vary greatly in
details, specific technology choices, downtime and risk tolerances.
Consequently, the process described here is not a "one size fits all"
automated push-button tool but a series of steps that should be
obvious to automate and customise to meet local needs.

Without further customisation, the process described has the following
impact:

REST API Impact
---------------

* Neutron REST API is read-only until after the migration is
  complete.

* Nova REST API is available throughout the entire process, although
  there is a brief period where it is made read-only during a
  database migration.

Performance Impact
------------------

* During the migration, nova network API calls will go through an
  additional internal conversion to neutron calls.  This will have
  different (likely poorer) performance characteristics compared
  with either the pre-migration or post-migration APIs.

VM Instance Impact
------------------

* In order to support a wide range of deployment options, the "dumb"
  process described here requires a rolling restart of hypervisors.
  For specific before/after deployment options, a "smart" hit-less
  hypervisor migration may be possible.  The process described here
  makes it easy to take advantage of a smart tool, but the
  implementation of such a tool is out of scope for this document.

* The rate and timing of specific hypervisor restarts is under the
  control of the operator.  The migration may pause for an extended
  period of time (for example, while testing or investigating
  issues) with some hypervisors on nova-network and some on neutron,
  and nova API remains fully functional.

* Individual hypervisors may be rolled back to nova-network during
  this stage of the migration, although this will require an
  additional restart.

Supported Deployment Technologies
---------------------------------

* This process supports any nova-network and neutron deployment
  options which are wire-compatible.  For example, nova-network with
  VLANs can be migrated to neutron-ML2-OVS/LinuxBridge with VLANs (using the
  same VLAN ids), Flat and FlatDhcp deployments to neutron-ML2-OVS/LinuxBridge
  flat network.

* Note that multi-host nova-network is not wire-compatible with
  neutron DVR and this migration path is not supported.

* Advanced nova-network services like CloudPipe are not supported,
  although some of these may have equivalents available in neutron.

* Nova cells will not be supported in this revision of this process,
  although it is recognised that this is a widely used feature in
  existing larger nova-network deployments.  We welcome involvement
  from deployers to assist in developing a cells-aware variant, and
  expect this to be an important followup once the no-cells version
  has been demonstrated.

Other Deployer Impact
---------------------

* In order to support the widest range of deployer needs, the
  process described here is easy to automate but is not already
  automated.  Deployers should expect to perform multiple manual steps
  or write some simple scripts in order to perform this migration.

Data Model Impact
-----------------

(Detail for this section added in followup change)

IPv6 Impact
-----------

None

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Community Impact
----------------

The current process as designed is a minimally viable migration with
the goal of deprecating and then removing nova-network.

The nova and neutron teams agree that a "one-button" migration process
from nova-network to neutron is *not* the goal of this process and is
*not* a requirement for the deprecation and removal of nova-network
at a future date.

This spec includes a process and tools which are designed to solve a
simple use case migration. Users are encouraged to take these tools,
test them, provide feedback, and then expand on the feature set to
suit their own deployments. Deployers that refrain from participating
in this process with the intent of waiting for a path that suits their
use case better than what is offered are likely to be disappointed.

Developer Impact
----------------

This will involve some changes in both nova and neutron.

Security Impact
---------------

None

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Overall proposed migration process
----------------------------------

(Detail for this section added in followup change)

Alternatives
------------

(Detail for this section added in followup change)

Implementation
==============

(Detail for this section added in followup change)

Assignee(s)
-----------

Primary assignee:
obondarev

Other contributors:
gus

Work Items
----------

(Detail for this section added in followup change)


Dependencies
============

(Detail for this section added in followup change)

Testing
=======

Grenade upgrade test
--------------------

"Sideways" grenade test.
Configure "old" cloud config using nova-net and "new" config using
neutron.  Grenade sets up "old", runs some tests, shuts it down, runs
the "upgrade" steps, and fires up "new".  The Grenade check passes if
the tests pass against the "new" cluster. [#grenade_job]_

While "Sideways" job is a good start, we don't expect it to be the only test
for migration.

Other
-----

We will need to confirm that the mid-point situation where some nodes
are on nova-network and some are on neutron is able to forward
east-west traffic and handle nova API calls as expected.  This
requires a multi-node scenario so may not be able to use our regular
automated testing frameworks.

Tempest Tests
-------------

Existing tempest tests might be used during "sideways" grenade job (see above).
However the main thing is to ensure objects (networks, instances, etc.)
existing in the "old" cloud, migrate fine and continue to function properly in
the "new" cloud. This will likely require adding new special tests for the job.

Tempest Javelin tool can be used by the job to validate that resources survive
after the migration [#javelin]_

Functional Tests
----------------

(Detail for this section added in followup change)

API Tests
---------

The public nova-network and neutron APIs should not change with this spec.

Documentation Impact
====================

Documentation should be added to the operators guide [#operators_guide]_
describing the whole neutron migration process in detail.

User Documentation
------------------

None.

Developer Documentation
-----------------------

None.  This change should not alter the public nova-network nor
neutron REST APIs, except as required for neutron to expose sufficient
information to nova during the migration.

References
==========

.. [#grenade_job] "Sideways" grenade job proposal
   http://lists.openstack.org/pipermail/openstack-dev/2014-October/047593.html

.. [#operators_guide] Operators Guide
   https://github.com/openstack/operations-guide/blob/master/doc/openstack-ops/ch_ops_upgrades.xml

.. [#javelin] Javelin tool
   https://github.com/openstack/tempest/blob/master/tempest/cmd/javelin.py
