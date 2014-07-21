..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================================
Group Based Policy Driver for One Convergence NVSD Controller
=============================================================

https://blueprints.launchpad.net/neutron/+spec/gbp-oc-nvsd-driver

This blueprint proposes a Group Based Policy (GBP) plugin driver to realize
GBP policy with One Convergence NVSD controller.

Problem description
===================

One Convergence NVSD controller implements an overlay fabric to provide
virtual networks and enable the deployment of network services in the
virtual networks. GBP plugin framework provides the capability to use
different drivers to render the policy definition using a specific technology.
One Convergence GBP driver is required to implement the GBP APIs [1] using the
connectivity, policy flow and service insertion primitives provided by NVSD
controller.

Proposed change
===============

We propose the addition of a new GBP driver to implement the GBP APIs [1] and
render the policy using the NVSD controller. This driver will proxy the APIs
via REST interface to the NVSD controller. The GBP driver for NVSD controller
will implement the PolicyDriver interface as defined in the abstract base class
services.grouppolicy.group_policy_driver_api.PolicyDriver.

Alternatives
------------

None

Data model impact
-----------------

None (existing GBP model is used)

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

The driver will reuse the configuration for NVSD Neutron plugin [2] to access
the NVSD controller.

Performance Impact
------------------

This driver should allow for a more scalable solution of GBP deployments
using the NVSD controller.

Other deployer impact
---------------------

None

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Hemanth Ravi (hemanthravi)

Subrahmanyam Ongole (songole)


Work Items
----------

1. Developing the NVSD GBP driver

Dependencies
============

Group Based Policy Plugin

Testing
=======

Unit tests will be provided.

The 3rd party One Convergence CI setup will be enhanced to cover the
testing of NVSD GBP driver using the NVSD controller.

Documentation Impact
====================

Documentation needs to be updated to reflect the addition of a new
GBP driver and it's configuration parameters.

References
==========

.. [1] Group-based Policy Abstractions: https://review.openstack.org/#/c/89469/

.. [2] NVSD Neutron Plugin: https://review.openstack.org/#/c/69246/
