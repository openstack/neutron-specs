..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================
LBaaS Refactor HAProxy namespace driver
=======================================

https://blueprints.launchpad.net/neutron/+spec/lbaas-refactor-haproxy-namespace-driver-to-new-driver-interface

With the new LBaaS object model and driver interface we no longer have a
working reference implementation.

Problem description
===================

Existing HAProxy namespace driver does not implement new LBaaS driver API or
support multiple listeners load balancer as supported by new LBaaS object
model.

Proposed change
===============

Refactor LBaaS HAProxy namespace driver to use new object model driver
interface.

Use Jinja2 to render HAProxy configuration template, rather than the custom
configuration generation.

HAProxy configurations may include multiple listeners per load balancer in a
single HAProxy process.

Separate files in $state_path/lbaas/$lb_uuid/ placing files written by HAProxy
under $state_path/lbaas/$lb_uuid/run to prepare for storing TLS private keys
under $state_path/lbaas/$lb_uuid/. Renaming configuration file, PID file and
statistics socket stored in $state_path/lbaas/$lb_uuid/ to avoid name conflicts
with Stunnel. Further TLS related changes are outside the scope of this spec.
The new directory structure will look like this:

::

  $state_path/lbaas/$lb_uuid/
  $state_path/lbaas/$lb_uuid/haproxy.conf
  $state_path/lbaas/$lb_uuid/run/
  $state_path/lbaas/$lb_uuid/run/haproxy.pid
  $state_path/lbaas/$lb_uuid/run/haproxy_stats.sock

The driver will need to detect if the sock and/or PID files are not present in
the new locations, and upgrade the running load balancer namespace to this new
file system. This will result in a brief interruption of load balancer service
while HAProxy is shutdown and configuration updated.

Further TLS and Layer 7 content filtering/manipulation are outside the scope of
this spec, but other specs may depend on this.

Alternatives
------------

Implement a different LBaaS reference driver.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

Restricting write access to $state_path/lbaas/$lb_uuid/ to root, and placing
HAProxy modified files in $state_path/lbaas/$lb_uuid/run/.

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

Preformance should remain similar to existing HAProxy namespace driver.

Other deployer impact
---------------------

The deployer will need to stop the lbaas-agent before while restarting
neutron-server upgrade the database schema and RPC protocol.

The deployer should also note the interruption in load balancer services and
each namespace's file system is updated to the new configuration.

Adding Jinja2 requirement for HAProxy namespace driver functionality, this is
already used within Neutron by the VPNaaS driver.

Developer impact
----------------

This will significantly affect the HAProxy namespace driver in order to
accommodate the new driver interface and multiple listeners.

Addition of Jinja2 requirement for LBaaS support may require DevStack updates.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  https://launchpad.net/~dlundquist

Other contributors:
  https://launchpad.net/~phillip-toohill

Work Items
----------

* Develop Jinja2 template for HAProxy configuration.
* Update HAProxy namespace driver.


Dependencies
============

* LBaaS API and object model improvements
* LBaaS object model driver changes

Testing
=======

Implementation of this spec will allow end to end testing of new LBaaS object
model.

Documentation Impact
====================

None

References
==========

* specs/juno/lbaas-api-and-objmodel-improvement.rst
* specs/juno/lbaas-objmodel-driver-changes.rst
