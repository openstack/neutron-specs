..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
LBaaS reference implementation TLS support
==========================================

https://blueprints.launchpad.net/neutron/+spec/lbaas-ref-impl-tls-support

LBaaS reference HAProxy implementation needs improvement to support
TLS including SNI.

This blueprint describes the changes that should be made to the HAProxy
reference implementation to allow features provided by the blueprints:
https://blueprints.launchpad.net/neutron/+spec/lbaas-ssl-termination
https://blueprints.launchpad.net/neutron/+spec/lbaas-refactor-haproxy-namespace-driver-to-new-driver-interface
and its successors to be implemented.

Problem Description
===================

The reference driver and its utilities currently do not support 'advanced'
features hindering the forward development of advanced API features suggested in
the 'lbaas-ssl-termination' blueprint.

In order to support TLS offloading configurations the reference driver (HAProxy)
must be updated to ensure proper 'backend' behavior and capabilities.

Features not currently supported in HAProxy 1.4 (current stable):
 - TLS termination.
 - TLS Source IP session persistence
 - X-Forwarded-For headers for TLS connections.
 - TLS Source IP load balancing method
 - TLS re-encryption

This spec will not include scope for L7, source_ip session persistence, TLS
session ID session persistence, source_ip load balancing algorithm, TLS
re-encryption as well as x-forwarded-for or certificate based client
authentication.

Scope of this spec is to include TLS which includes SNI support.

Proposed Change
===============

The current reference driver named 'namespace_driver' utilizes HAProxy 1.4.
Update to use HAProxy 1.5(dependent on packaging)

In order to implement these features a few things need to be done:

1. Update HAProxy config. The configuration will be built using Jinja
as specified in spec: "https://blueprints.launchpad.net/neutron/+spec/
lbaas-refactor-haproxy-namespace-driver-to-new-driver-interface" and will
expand on it to include TLS features.

The configuration utility will configure new directories and files for
HAProxy and certificates in the structure below. This will ensure no name
collisions.

::

    $state_path/lbaas/$lb_uuid/
    $state_path/lbaas/$lb_uuid/$cert1_barbican_id.pem
    $state_path/lbaas/$lb_uuid/$cert2_barbican_id.pem
    $state_path/lbaas/$lb_uuid/$certN_barbican_id.pem
    $state_path/lbaas/$lb_uuid/haproxy.conf
    $state_path/lbaas/$lb_uuid/run/
    $state_path/lbaas/$lb_uuid/run/haproxy.pid
    $state_path/lbaas/$lb_uuid/run/haproxy_stats.sock

2. The pem file containing the private key will be written
with permissions such that its only readable by root to protect security
credentials.

Modification of neutron.agent.linux.util#replace_file to accept an optional
'file_mode' argument to specify permissions other then default '0644'. This
protects against race condition where attacker reads the private key
before the file permissions are set.

3. There are also tear down methods i.e. undeploy_instance that will need to be
updated for proper clean up. (kill pids)

Additional Thoughts:
Those using devstack will not be able to use this feature unless manually
installed or devstack itself is updated. This would need to be updated
on that side at some point.

Data Model Impact
-----------------

None

REST API Impact
---------------

None. This blueprint is intended to provide capabilities that can be supported
in future versions of the REST API.

Security Impact
---------------

Users private key will be written into a file readable by root on the local file
system of the network node.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

Devstack will need to be updated to install the new packages(HAProxy 1.5).

Performance Impact
------------------

Additional calls will have to be made to spawn additional instances.

TLS offloading increases overhead to the network node.

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

Deployer will need to ensure new dependencies are installed.

Developer Impact
----------------

Developers will need to ensure they are using the additional utilities based
on the lb configuration.

Developers will need to create a utility to retrieve Barbican secrets/data.

Community Impact
----------------

This change has been in review since Juno.  Much discussion has taken place
over IRC and the mailing list.

Alternatives
------------

Alternatively, if we would like to support different TLS offloading tools like
Stud we could support plugin or extensions that are loaded in front of HAProxy.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  phillip-toohill

Other contributors:
  dlundquist

Work Items
----------
Update haproxy 'haproxy.conf' and jinja templates to handle new configurations.
Update namespace_driver methods for new actions.
Testing.

Dependencies
============

 - Depends on blueprints:
   https://blueprints.launchpad.net/neutron/+spec/lbaas-api-and-objmodel-improvement
   https://blueprints.launchpad.net/neutron/+spec/lbaas-ssl-termination
   https://blueprints.launchpad.net/neutron/+spec/lbaas-refactor-haproxy-namespace-driver-to-new-driver-interface
   and its successors noted within.

Testing
=======

Tempest Tests
-------------

* Add TLS to existing LBaaS tempest tests

Functional Tests
----------------

* Test to verify SSL termination

API Tests
---------

None

Documentation Impact
====================

User Documentation
-------------------

Document behavior and capabilities of the refactored reference implementation.

Developer Documentation
------------------------

Document behavior and capabilities of the refactored reference implementation.

References
==========

http://www.haproxy.org/
https://blueprints.launchpad.net/neutron/+spec/lbaas-api-and-objmodel-improvement
https://blueprints.launchpad.net/neutron/+spec/lbaas-refactor-haproxy-namespace-driver-to-new-driver-interface
https://blueprints.launchpad.net/neutron/+spec/lbaas-ssl-termination