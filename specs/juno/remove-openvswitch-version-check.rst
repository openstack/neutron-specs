..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Remove openvswitch run-time version checks
==========================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/remove-openvswitch-version-check

The current method of checking openvswitch and kernel versions for specific
feature support is brittle, distro-specific and unsupportable.

Problem description
===================
When VXLAN tunnels are enabled, openvswitch-using agents call ovs_lib code to
check that the installed kernel and openvswitch versions are greater than a
minimum specified version. Due to distro-specific differences in 1) the fields
returned by modinfo and 2) the minimum required version due to the practice of
backporting, this run-time checking method fails.

In addition to VXLAN, other specific openvswitch features will need to be
tested for in the future. As the number of features and versions and distros
grows, checking version numbers becomes increasingly unsustainable and should
not be done at agent start, but instead at package installation or deployment.

Proposed change
===============
Trying to test features by version numbers across multiple distros is a fool's
errand. It is much better to leave dependency management to the distros and
packaging as they are the only ones who really know what version dependencies
exist in their environment. The run-time checks should be completely removed
from ovs_lib and the agents.

If an attempt is made to use a specific openvswitch feature fails, a helpful
error message suggesting that updating openvswitch *may* resolve the issue
should be logged.

To help deployment tools, a script will be created that attempts to use
specific openvswitch features and exits 0 for success and 1 for failure. This
way a deployment can be aborted, e.g. in devstack, if the feature is enabled
but not available in the current openvswitch/kernel versions without trying
to keep track of version numbers or incurring a messy run-time feature check
at agent start.

Alternatives
------------
Openvswitch does not have the ability to query whether it supports certain
features, so the only way to reliably test whether a feature exists at runtime
is to try to use the feature and see if it works. It would be possible, but
unacceptably messy, to attempt to use all required OVS features at agent start
and exit the agent on failure. Dependency management should be handled at
packaging and/or deployment and not checked every time an agent is started by
modifying system state.

Data model impact
-----------------
N/A

REST API impact
---------------
N/A

Security impact
---------------
N/A

Notifications impact
--------------------
N/A

Other end user impact
---------------------
N/A

Performance Impact
------------------
Negligible improvement in openvswitch-using agents due to removal of checks.

Other deployer impact
---------------------
Existing deployments will be unaffected as they will already have the
appropriate dependencies for their deployment.

Deployments using distro packaging will be unaffected as the dependencies will
already be handled by the packaging.

Source installs, e.g. devstack, should run the feature test script to verify
that an appropriate version of openvswitch has been installed on the system.

Developer impact
----------------
Agents using openvswitch will no longer need to check openvswitch versions
when using version-specific openvswitch features. Testing is simplified since
the version-check feature is being removed and no longer requires testing.

Devstack can also be modified to run the external feature test script before
deployment and fail early in the case of a configuration/dependency mismatch.

Implementation
==============

Assignee(s)
-----------
Primary assignee:
  otherwiseguy

Work Items
----------
* Remove runtime version checks and associated tests
* Create the feature test script

This will essentially be a refactoring of:

  https://review.openstack.org/#/c/88121/

into two pieces, the runtime check -> separate script and the existing code
removal in that patch.

Dependencies
============
N/A

Testing
=======
As this implementation primarily removes code that should not exist, it also
removes the need for testing the version checking code.

Documentation Impact
====================
N/A

References
==========
N/A