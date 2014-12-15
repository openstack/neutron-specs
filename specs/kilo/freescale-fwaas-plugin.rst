..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
  License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================
Freescale FWaaS Plugin
=======================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/freescale-fwaas-plugin

Note: This Blueprint is proposed for Neutron-FWaaS repository.

CRD (Cloud Resource Discovery) Service is designed to support Freescale silicon
in data center environment. Like Neutron, it uses keystone authentication for
all ReSTful calls.

Problem Description
===================
Neutron-FWaaS Firewall information (like rules, policies and firewall) is requried
by CRD service to manage Firewall deployment in virtual network appliances and
openflow controller apps.

In order to send this information to CRD from neutron, a new FWaaS plugin is
required.

Freescale FWaaS Plugin proxies ReSTful calls (formatted for CRD Service) from
Neutron to CRD Service.

It supports the Cloud Resource Discovery (CRD) service by updating the Firewall
related data (rules, policies and firewall) into the CRD database.

CRD service manages creation of firewall on network nodes, virtual network
appliances and openflow controller network applications.


Proposed Change
===============
The basic work flow is as shown below.

::

 +-------------------------------+
 |                               |
 |       Neutron Service         |
 |                               |
 |    +--------------------------+
 |    |  Neutron-FWaaS with      |
 |    |  Freescale Firewall      |
 |    |  Service Plugin          |
 |    |                          |
 +----+-----------+--------------+
                  |
                  |  ReST API
                  |
                  |
        +---------v-----------+
        |                     |
        |   CRD Service       |
        |                     |
        +---------------------+

Freescale Firewall Service plugin sends the Firewall related data to
CRD server.

The plug-in implements the CRUD operation on the following entities:
  * Firewall Rules
  * Firewall Policies
  * Firewall

The plug-in uses the exisitng firewall database to store the firewall
data.

The creation of firewall in network node or Virtual Network appliance or
Openflow controller app is decided by CRD service.

Sequence flow of events for create_firewall is as follows:

::

  create_firewall
  {
    neutron       ->  fsl_fw_plugin
    fsl_fw_plugin ->  crd_service
    fsl_fw_plugin <-- crd_service
    neutron       <-- fsl_fw_plugin
  }


Alternatives
------------

None


Data Model Impact
-----------------

Existing models are used.
No new models are created.

REST API Impact
---------------

None.

IPv6 Impact
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

Negligible or None

Other Deployer Impact
---------------------

This change only affects deployments where neutron 'service_plugins' is
configured with 'fsl_firewall', 'core_plugin' is 'ml2' and 'ml2' plugin is
configured with 'fslsdn' mechanism driver.

'fslsdn' mechanism driver configuration is used by this service plugin to
contact CRD service.

Update [DEFAULT] section of /etc/neutron/neutron.conf as,

::

 [DEFAULT]
 service_plugins = fsl_firewall
 core_plugin = ml2

The following configuration changes are made to enable
Freescale SDN mechanism driver.

In [ml2] section of /etc/neutron/plugins/ml2/ml2_conf.ini,
modify 'mechanism_drivers' attributes as,

::

 mechanism_drivers = fslsdn

Update /etc/neutron/plugins/ml2/ml2_conf_fslsdn.ini, as below.

::

 [ml2_fslsdn]
 crd_auth_strategy = keystone
 crd_url = http://127.0.0.1:9797
 crd_auth_url = http://127.0.0.1:5000/v2.0/
 crd_tenant_name = service
 crd_password = <-service-password->
 crd_user_name = <-service-username->


Developer Impact
----------------

None.

Community Impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  trinath-somanchi

Other contributors:
  None

Work Items
----------

- Freescale FWaaS Service Plugin

Dependencies
============

None

Testing
=======

Tempest Tests
-------------
* Complete unit test coverage of the code is included.
* For tempest test coverage, third party testing is provided.
* The Freescale CI reports on all changes affecting this Plugin.
* The testing is run in a setup with an OpenStack deployment (devstack)
  connected to an active CRD server.

Functional Tests
----------------
Complete unit test coverage of the code is included.

API Tests
---------
None.

Documentation Impact
====================

User Documentation
------------------
Since the usage of this plugin requires 'ml2' as core_plugin and 'fslsdn' as
mechanism_driver, the above detailed deployer impact will be documented.

Developer Documentation
-----------------------
None needed beyond documentation changes listed above.


References
==========
None
