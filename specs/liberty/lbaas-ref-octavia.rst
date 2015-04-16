..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
Lbaas, use Octavia as reference implementation
==============================================

https://blueprints.launchpad.net/neutron/+spec/lbaas-ref-octavia

The lbaas team intends to replace the reference implementation
with the service-vm like implementation provided by project
Octavia[1], which uses nova compute to provide scalability
and HA features.

Problem Description
===================

The existing reference implementation does not scale well,
does not handle HA, is fragile in the face of underlying host failures.

Proposed Change
===============

A neutron-lbaas Octavia driver (or shim) will be added to neutron-lbaas,
which will reference the controller/framework implemented by the Octavia
project[1].

The existing reference implementation, the namespace haproxy driver, will
remain in the neutron-lbaas tree for Liberty, for upgrades.

The default driver enabled in the neutron-lbaas.conf file will be Octavia,
a dependency on Octavia will be added to the project requirements file,
and packagers will have to install/deploy Octavia as part of an lbaas install.

The neutron-lbaas devstack plugin will start installing Octavia automatically.

Data Model Impact
-----------------

None.

REST API Impact
---------------

None.

Security Impact
---------------

None.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None.

Performance Impact
------------------

None.

IPv6 Impact
-----------

None.

Other Deployer Impact
---------------------

Upgrades will have to manually choose to switch to Octavia. The old driver
will continue to function.

New installs will need to pull in and start Octavia.

Developer Impact
----------------

None.

Community Impact
----------------

The existing reference implementation is not well liked. Hopefully, this
starts to change that perception.

Alternatives
------------

One alternative is to continue to maintain the existing reference/agent
driver, and keep Octavia present as an option. The extra maintenance cycles
of keeping both drivers prime-time is less desirable by the lbaas team.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Octavia team

Work Items
----------

- Octavia neutron-lbaas driver.
- Tweak requirements.
- Convert tests to stably run with Octavia.
- Modify devstack plugin

Dependencies
============

* A working Octavia implemention. The switch will happen no later than the
  middle of L-3, and only if Octavia is "complete enough" for production use,
  as determined by the services lieutenant and/or the PTL.

Testing
=======

Existing tests will exercise the Octavia backend. Additional jobs will be
added to continue testing the namespace/haproxy backend, until it is
removed (schedule TBD.)

Tempest Tests
-------------

Same as existing.

Functional Tests
----------------

Same as existing.

API Tests
---------

Same as existing.

Documentation Impact
====================

User Documentation
------------------

References to the driver/config settings will need to be updated, as will documentation
on how to check if the right agents are running/troubleshooting.

Developer Documentation
-----------------------

None.

References
==========

[1] https://github.com/stackforge/octavia

