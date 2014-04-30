..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
Add firewall driver for McAfee NGFW firewall
============================================

https://blueprints.launchpad.net/neutron/+spec/mcafee-ngfw-fwaas-driver

Implements FWaaS driver for McAfee ngfw firewall


Problem Description
===================

McAfee NGFW(next generation firewall) support/integration is missing.
It provides cloud admin/users more choice.


Proposed Change
===============

Add a new firewall driver for NGFW.
(neutron firewall driver. a subclass of FwaasDriverBase)

Introduce a new driver for the existing reference FWaaS agent.
The FWaaS agent will load this new driver for NGFW firewall if configured.
There will be no changes to the existing reference FWaaS agent.

The new driver gets user requests from the FWaaS agent and sends
these requests to the SMC server (NGFW management server) via REST API.
SMC server is a sort of a controller of firewall devices.

The new driver needs the SMC server IP address and the API key (NGFW specific)
to talk to the SMC API. This information should be specified in the agent
configuration file. The new driver will read these information from the
configuration file and make connections to the SMC server REST API interface
using this information.

The agent driver would inherit from the base class (FwaasDriverBase)
and overrides methods for NGFW-specific feature.

The agent will run on network node as it is today.(But the agent doesn't
have to run on network node. It will be addressed by future phase.
See the section of future work.)

References to L3 router plugin in the diagram are added to help with
better understanding and is not in scope of this proposal
[ngfw-l3-router]_

diagram for first implementation::


    +---------------------------------+
    |Neutron server                   |
    |                                 |
    |  +---------+                    |    +------+
    |  |l3 plugin+--------------------+----+ Nova |
    |  +---+-----+                    |    +--+---+
    |      |        +---------------+ |       |
    |      |        |firewall plugin| |       |
    |      |        +-----+---------+ |       |
    +------|--------------|-----------+       |
           |              |                   |
           |              |openstack rpc      |
           |              |topic:L3_AGENT     |
           |              |                   |
           |        +-----|---------+         |
           |        |     | l3 agent|         |
           |        | +---+--+      |         |
           |        | |fwaas |      |         |
           |        | |driver|      |         |
           |        | +-+----+      |         |
           |        +---|-----------+         |
           |            |                     |
           |            | provider network    |
           |            | cloud admin manages |
     +-----+------------++                    |
     |SMC VM(product)    |                    |
     |(management server)|<-------------------+ spin up/down
     +----+--------------+                    |
          |                                   |
          | tenant network:                   |
          |  cloud admin manages              |
     +----+----------------+                  |
     |SG-engine VM(product)|                  |
     |(actual service)     |<-----------------+ spin up/down
     +--+--+---------------+                    add/remove interfaces
        |  |
        |  |...
        |  |
     tenant networks for cloud user


future work

Firewall driver for config agent will be implemented during the third phase once
l3 routervm plugin and config agent are merged and which is enhanced to
support firewall service as well. [config-agent]_ [modular-l3-router-plugin]_
(Allowing multiple type of routers will be addressed by
[modular-l3-router-plugin]_. It's different topic.)

References to L3 router plugin in the diagram are added to help with
better understanding and is not in scope of this proposal
For details, please refer to [ngfw-l3-router]_.

The main difference of config agent is that fwaas agent is tied to
network node and host fwaas instance which is instantiated on the physical
node, on the other hand the config agent is not tied to network node or
servicevm which serves fwaas instance and it can run anywhere as long as
it can receive RPC and communicate with fwaas management service.
This direction aligns with other similar activities.
[fwaas-csr1kv]_, [fwaas-tcs]_

diagram for implementation with l3-routervm plugin and config agent::


    +---------------------------------+
    |Neutron server                   |
    |                                 |
    |  +---------+                    |      +------+
    |  |l3 plugin+--------------------+------+ Nova |
    |  +---+-----+                    |      +--+---+
    |      |        +---------------+ |         |
    |      |        |firewall plugin| |         |
    |      |        +-----+---------+ |         |
    +------|--------------|-----------+         |
           |              |                     |
           |              |openstack rpc        |
           |              |                     |
    +---------------------|--------+            |
    |      |              |        |            |
    |  +------+       +---+--+     |            |
    |  |router|       |fwaas |     |            |
    |  |driver|       |driver|     |            |
    |  +---+--+       +-+----+     |            |
    |      |            |          |            |
    |      |  config    |          |            |
    |      |  agent     |          |            |
    +------|------------|----------+            |
           |            |                       |
           |            | provider network:     |
           |            | cloud admin manageses |
     +-----+------------++                      |
     |SMC VM             |                      |
     |(management server)|<---------------------+ spin up/down
     +----+--------------+                      |
          |                                     |
          | tenant network:                     |
          |  cloud admin manages                |
     +----+-----------+                         |
     |SG-engine VM    |                         |
     |(actual service)|<------------------------+ spin up/down
     +--+--+----------+                           add/remove interfaces
        |  |
        |  |...
        |  |
     tenant networks for cloud user



Data Model Impact
-----------------

None


REST API Impact
---------------

None


Security Impact
---------------

None
Although this NGFW driver provides cloud users another choice for security,
this section is for the potential impact of the system. Not for user impact.


Notifications Impact
--------------------

None


Other End User Impact
---------------------

User will have another choice of firewall provider.


Performance Impact
------------------

None


IPv6 Impact
-----------

None


Other Deployer Impact
---------------------

New service provider for the driver will be introduced.The deployer
who wants use NGFW needs to configure to use the l3 router plugin and
firewall driver.


Developer Impact
----------------

None


Community Impact
----------------

The NGFW fwaas driver provides cloud user more choice of Neutron FWaaS.
Thus it promotes Neutron FWaaS.


Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  rui-zang
  yalei-wang
  yamahata

Other contributors:
  None

Work Items
----------

* FWaaS driver
* tests
* third party CI

Once l3 routervm plugin and config agent are merged
* Refactor firewall driver into a driver for config agent


Dependencies
============

* NGFW l3 router plugin [ngfw-l3-router]_


Testing
=======

Third party testing would be added.


Tempest Tests
-------------

Third party testing will be added to Intel CI.


Functional Tests
----------------

Scenario tests will be added to validate the NGFW driver implementation.


API Tests
---------

None


Documentation Impact
====================

Admin guide will be updated.


User Documentation
------------------

The another choice of FWaaS backend will be added.


Developer Documentation
-----------------------

None


References
==========

.. [ngfw-l3-router]
   * https://blueprints.launchpad.net/neutron/+spec/mcafee-ngfw-l3-router
   * https://review.openstack.org/#/c/134198/

.. [config-agent]
   * http://git.openstack.org/cgit/openstack/neutron-specs/tree/specs/juno/cisco-config-agent.rst

.. [fwaas-csr1kv]
   * https://blueprints.launchpad.net/neutron/+spec/fwaas-cisco
   * https://review.openstack.org/#/c/129836/
     spec
   * https://review.openstack.org/#/c/115308/
     patch

.. [fwaas-tcs]
   * https://blueprints.launchpad.net/neutron/+spec/tcs-fwaas-netconf-host-plugin
   * https://review.openstack.org/#/c/98104/

.. [modular-l3-router-plugin]
   * https://blueprints.launchpad.net/neutron/+spec/l3-plugin-for-routervm
   * https://review.openstack.org/#/c/105078/
