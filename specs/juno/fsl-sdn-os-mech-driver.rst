=====================================================
Freescale SDN Mechanism Driver for Neutron ML2 plugin
=====================================================

https://blueprints.launchpad.net/neutron/+spec/fsl-sdn-os-mech-driver

FSL-SDN MD : Freescale SDN Mechanism Driver

CRD        : Cloud Resource Discovery Service - like neutron,
             uses keystone authentication for all ReSTful calls.

Freescale SDN (FSL-SDN) Mechanism Driver proxies ReSTful calls (formatted for
CRD Service) from ML2 plugin of Neutron to CRD Service.

It supports the Cloud Resource Discovery (CRD) service by updating
the Network, Subnet and Port Create/Update/Delete data into the CRD database.

CRD service manages network nodes, virtual network appliances and openflow
controller network applications.

The basic work flow is as shown below.

::

 +---------------------------------+
 |                                 |
 |       Neutron Server            |
 |      (with ML2 plugin)          |
 |                                 |
 | +-------------------------------+
 | |        Freescale SDN          |
 | |       Mechanism Driver        |
 +-+--------+----------------------+
            |
            |  ReST API
            |
 +----------+-------------+
 |      CRD server        |
 +------------------------+


Problem description
===================

Openstack neutron based networks and ports information is required by
CRD service to managed virtual network appliances and openflow controller
apps.

In order to send this information from neutron service, a new ML2
mechanism driver is required to post the _postcommit data to the CRD
service.

Proposed change
===============

Freescale Mechanism driver handles the following postcommit operations.

Network create/update/delete
Subnet  create/update/delete
Port    create/delete

Supported network types by FSL OF Controller include vlan and vxlan.

Freescale SDN mechanism driver handles VM port binding within in the
mechanism driver (like ODL MD).

'bind_port' functions verifies the supported network types (vlan,vxlan)
and calls context.set_binding with binding details.

Freescale Openflow Controller manages the flows required on OVS.

Sequence flow of events for create_network is as follows:

::

 create_network
 {
   neutron    ->  ML2_plugin
   ML2_plugin ->  FSL-SDN-MD
   FSL-SDN-MD ->  crd_service
   FSL-SDN-MD <-- crd_service
   ML2_plugin <-- FSL-SDN-MD
   neutron    <-- ML2_plugin
  }

Port binding task is handled within the mechanism driver, So OVS agent driver
is not required when this mechanism driver is enabled.


Alternatives
------------

None

Data model impact
-----------------

None


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

None


Performance Impact
------------------

Negligible/None

Though data is sent from mechansim driver to CRD service, the performance
impact is negligible(less than 1%) or None.

Other deployer impact
---------------------

This change doesn't take immediate effect.

To work with the change, neutron need to be configured with ml2,
and fslsdn as mechanism driver with CRD service running.

The following configuration changes are to be made to enable
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

- Mechanism Driver (mechanism_fslsdn.py)

Dependencies
============

None

Testing
=======

- Complete Unit testing coverage of the code is included.
- For tempest test coverage, 3rd party testing is provided (Freesacle CI).
- Freescale CI reports on all changes affecting this driver.
- Testing is done using devstack and CRD service.
- CRD service logs are also posted into the CI log repository.


Documentation Impact
====================

Configuration Reference guide will be updated from the code.


References
==========

None
