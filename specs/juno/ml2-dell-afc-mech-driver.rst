=================================
Dell Neutron ML2 Mechanism Driver
=================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/dell-mechanism-driver

This blueprint is to add mechanism driver to OpenStack
Neutron ML2 plugin. The mechanism driver will work with Dell AFC SDN controller.


Problem description
===================

Dell Access Fabric Controller(AFC) provides multi-tenancy network connectivity
for servers and VMs connected to the active fabric. It manages a group of Dell
Switches (currently supporting Dell 4810, 4820, MXL, S6000) that use the Openflow
protocol to communicate with the AFC controller. The Dell ML2 mechanism driver
proposed will serve as the channel to communicate orchestration from Openstack
to AFC controller. The communication is conducted via AFC Rest API. The mechanism
driver will provide information needed by AFC to configure the Dell switches,
such as vlan and port. It will also provide the ability to synchronize with the
AFC controller when there is a disconnection between Openstack and AFC controller.

Proposed change
===============

Dell mechanism driver will implement the following functions from the mechanism
driver framework:

- create_network_postcommit
- create_subnet_postcommit
- update_network_postcommit
- delete_network_postcommit
- create_port_postcommit
- delete_port_postcommit

Other than the standard information passed down by ml2 plugin, Dell mechanism
driver will provide AFC controller with information needed to form the abstract
fabric view. These information would include details about the hypervisors, VMs
and tenants.

In addition to these functions, the Dell mechanism driver also performs a
"periodic task" that will check the AFC controller start up time. If the start up
time is different from the record on Dell mechanism driver, it will synchonize the
information with the AFC controller. The synchonization function will read from nova
and neutron of the same information described above and post to the AFC controller.

Alternatives
------------

An alternative solution would be to develop a monolithic plugin. The
biggest advantage of using the ML2 mechanism driver approach is that
it reduces the duplication of work done by openvswitch.

Data model impact
-----------------

None.

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

None.

Other deployer impact
---------------------

The deployer must configure the following information in separate config file
named ml2_conf_dell.ini

::

  [ml2_dell]
  controller_ip // the AFC controller ip
  controller_port // the AFC controller rest api port
  provider_id = // the openstack provider id for AFC
  username = // the AFC rest api user name
  password = // the AFC rest api password
  adminuser = // the openstack admin user name
  adminpwd = // the openstack admin password
  authurl = // the keystone authentication url
  projectname = // the openstack project name

Additionally, the deployer must configure the ML2 plugin to include
the openvswitch mechanism driver before the Dell AFC mechanism driver:

::

  [ml2]
  mechanism_drivers = openvswitch,dell

Note the openvswitch agent is not modified

Developer impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Andy Shi

Work Items
----------

- The mechanism driver basic functions
- The additional information(not included in the context of mechanism driver),
  such as hypervisors, VMs and tenants
- The synchronization function


Dependencies
============

None.


Testing
=======

Complete unit test coverage of the code is included.
For tempest test coverage, third party testing is provided.
The Test is run using devstack in Dell Access Fabric setup

Documentation Impact
====================

Configuration details.


References
==========
