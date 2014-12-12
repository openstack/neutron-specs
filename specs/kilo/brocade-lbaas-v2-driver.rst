===================================================
Brocade LBaaS Plugin Driver (v2 Data Model Support)
===================================================


Problem Description
===================

URL of the launchpad blueprint:
https://blueprints.launchpad.net/neutron/+spec/neutron-brocade-lbaas-driver

Brocade LBaaS Plugin and Driver for the Brocade ADX Load Balancer Devices
for the LBaaS in Neutron.


Proposed Change
===============

The driver will implement the interfaces according to the driver interfaces
mentioned in the spec https://review.openstack.org/100690
for the blueprint
https://blueprints.launchpad.net/neutron/+spec/lbaas-objmodel-driver-changes

The following managers will be implemented:

* LoadBalancerManager
* ListenerManager
* PoolManager
* MemberManager
* HealthMonitorManager

Implementation is composed of Plugin Driver and Brocade Device Driver.

Brocade LBaaS Plugin Driver extends/implements the driver interfaces
as mentioned above and forwards the request to the device driver.

The device driver communicates with the Brocade ADX Load Balancer
Device (Physical and Virtual) via SOAP/XML APIs.
The device driver use SUDs python module for the SOAP/XML API calls.

Supported Features

    Protocols:  HTTPS, HTTP, TCP
    LB Algorithms: ROUND_ROBIN, LEAST_CONNECTIONS
    Session persistence: SOURCE_IP
    Health Monitoring: TCP, HTTP, HTTPS
    Stats retrieval
    CRUD on loadbalancer, listener, pool, healthmonitor, member
    Additional lbaas features supported as part of framework in kilo release
    (TERMINATED_HTTPS, L7 etc)


Product Versions supported

    ADX 12.5 and above
    Virtual ADX 3.0, 3.1 and above


Exceptions

Brocade LBaaS Device Driver will raise one of the following exceptions
    ConfigError . Raised when a configuration exception occurs
                  on the load balancer device
    UnsupportedFeature . Raised when a particular feature is not yet
                         supported by the Device Driver
    UnsupportedOption . Raised when an unsupported value is
                        specified for an attribute.

Data Model Impact
-----------------

None

REST API Impact
---------------

None

Security Impact
---------------

None

Notifications Impact
--------------------

None


Other End User Impact
---------------------

None

Performance Impact
------------------

None

IPv6 Impact
-----------

Will support ipv6 at the same level as neutron lbaas.

Other Deployer Impact
---------------------

service_provider entry in the neutron-lbaas.conf file needs to be
updated to reflect Brocade as one of the service_provider for
the plugin to be effective.

Brocade Device Driver must be installed prior to using this driver


Developer Impact
----------------

None

Community Impact
----------------

None

Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

https://launchpad.net/~pattabi


Work Items
----------

* Brocade plugin driver code
* Unit tests
* Voting CI


Dependencies
============

* https://blueprints.launchpad.net/neutron/+spec/lbaas-api-and-objmodel-improvement


Testing
=======

- Unit Tests
- Brocade QA
- Existing LBaaS tests provide complete coverage, if driver is installed
  and configured (as our CI will do)

Tempest Tests
-------------

Brocade ADX CI will run existing LB tempest tests with Brocade ADX/vADX.

Functional Tests
----------------

Brocade ADX CI will run existing LB functional tests with Brocade ADX/vADX.

API Tests
---------

Brocade ADX CI will run existing LB API tests with Brocade ADX/vADX.

Documentation Impact
====================

None

User Documentation
------------------

None

Developer Documentation
-----------------------

None

References
==========

None

