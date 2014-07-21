===================================================
Group Based Policy Driver for Cisco APIC Controller
===================================================

Launchpad blueprint:
https://blueprints.launchpad.net/neutron/+spec/group-policy-apic-driver

GBP plugin has defined a multi-driver based framework to support
various implementation technologies (like ML2 has done for L2 support).
This blueprint proposes a Group Based Policy (GBP) driver to enable
GBP plugin to be used with Cisco APIC controller.

Problem description
===================

Cisco APIC controller enables you to create an application centric fabric.
If you require a policy driven network control in an openstack deployment
using the ACI fabric, the reference driver for GBP can not leverage the
efficiency or scalability provided by the native fabric interfaces available
in the APIC controller.

Proposed change
===============

We propose the addition of a new GBP driver to support the APIC controller.
It will implement the PolicyDriver interface as defined in the abstract base
class services.group_policy_driver_api.PolicyDrive.

The proposed GBP driver will interface with the APIC controller and
allow efficient and scalable use of the ACI fabric for policy based control
from GBP plugin.

Alternatives
------------

There are no alternatives to use the native capability of the ACI fabric
for policy based control from openstack APIs.

Data model impact
-----------------

None (existing models for GBP should suffice to capture policy for ACI)

REST API impact
---------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

The configuration files will need to be updated for using this driver.
These parameters include the addresses, credentials, and any configuration
required for accessing or using the APIC controller. Where possible, it
will share the configuration with APIC ML2 driver.

Performance Impact
------------------

This driver should allow for more efficient and scalable solution
for group based policy control of deployments using an ACI fabric.

Other deployer impact
---------------------

As above, there is a configuration impact.

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Mandeep Dhami (mandeep-dhami)

Ivar Lazzaro (mmaleckk)


Work Items
----------

1. Developing the APIC GBP driver

Dependencies
============

Group Based Policy Plugin

Testing
=======

Unit tests will be provided.

Since access to an APIC controller is required for testing the
proposed changes, 3rd party testing is required and will be
provided by Cisco CI system.

Documentation Impact
====================

Documentation needs to be updated to reflect the addition of a new
GBP driver and its configuration parameters.

References
==========

None

