=======================================
Freescale FWaaS Plugin
=======================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/freescale-fwaas-plugin

Problem description
===================

CRD (Cloud Resource Discovery) Service is designed to support Freescale silicon
in data center environment. Like Neutron, it uses keystone authentication for
all ReSTful calls.

Neutron Firewall information (like rules, policies and firewall) is requried
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


Proposed change
===============

The basic work flow is as shown below.

::

 +-------------------------------+
 |                               |
 |       Neutron Service         |
 |                               |
 |    +--------------------------+
 |    |                          |
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


Data model impact
-----------------

No existing models are changed and new models are created.

REST API impact
---------------

None.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

Performance Impact
------------------

Negligible or None

Other deployer impact
---------------------

This change only affects deployments where neutron 'service_plugins' is
configured with 'fsl_firewall'.

In [DEFAULT] section of /etc/neutron/neutron.conf modify 'service_plugins'
attribute as,

::

 [DEFAULT]
 service_plugins = fsl_firewall

Update /etc/neutron/plugins/services/firewall/fsl_firewall.ini, as below.

::

 [fsl_fwaas]
 crd_auth_strategy = keystone
 crd_url = http://127.0.0.1:9797
 crd_auth_url = http://127.0.0.1:5000/v2.0/
 crd_tenant_name = service
 crd_password = <-service-password->
 crd_user_name = <-service-username->

CRD service must be running in the Controller.

Developer impact
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

- Freescale firewall service plugin (fsl_firewall_plugin.py)

Dependencies
============

None

Testing
=======

* Complete unit test coverage of the code is included.
* For tempest test coverage, third party testing is provided.
* The Freescale CI reports on all changes affecting this Plugin.
* The testing is run in a setup with an OpenStack deployment (devstack)
  connected to an active CRD server.

Documentation Impact
====================

Usage and Configuration details.


References
==========

None
