==========================================
Neutron Plugin for Opencontrail
==========================================

https://blueprints.launchpad.net/neutron/+spec/juniper-plugin-with-extensions

This blueprint is for the Opencontrail-neutron-plugin to add support in Neutron
for Opencontrail based network virtualization. The list of core resources and
extensions supported are:

* Network
* Subnet
* Port
* Security Group
* L3 Extensions

We will also support Extraroute extension


Problem description
===================

Opencontrail is open source network virtualization solution. It uses standards
based BGP L3VPN closed user groups to implement virtual networks.
The link http://opencontrail.org/opencontrail-architecture-documentation/
explains the architecture of Opencontrail plugin

The Neutron plugin code would interact with Opencontrail Rest-API to provide
the network virtualization functionality.

Proposed change
===============

The Opencontrail Neutron plugin will implement the Neutron core and extension
APIs. It will receive the API request and pass it to the Opencontrail controller
for processing.
There are no changes to the Neutron common code. New files are addded to
implement the opencontrail-plugin.

Alternatives
------------
Current support is for traditional Neutron plugin interface.
We will consider using ML2 interface in future.

Data model impact
-----------------

None.

REST API impact
---------------

None.
There are no new API added to neutron. For above listed API all features
will be supported by the plugin.

Security impact
---------------
The communication channel to the backend is not secure.
We will support secure channel in the future.

Notifications impact
--------------------
None.

Other end user impact
---------------------

Users will now be able to configure Opencontrail as one of the available plugin
from upstream.

Performance Impact
------------------

None.

Other deployer impact
---------------------

Users will now be able to configure Opencontrail as one of the available plugin
from upstream. Following link explains setup of Opencontrail using devstack.
http://pedrormarques.wordpress.com/2013/11/14/using-devstack-plus-opencontrail/

Developer impact
----------------

None.
Other Developers wont be effected by this change. Tempest thirdparty environment
will be provided.

Implementation
==============

Opencontrail plugin does not use the Neutron persistence layer and so avoid
the problems when the database goes out of sync. Opencontrail plugin uses
additional objects and attributes for objects that are defined by Neutron
APIs, that data is stored in the Opencontrail controller which does the
persistency. By being stateless, the Neutron service doesn't have to deal
with its persistency layer getting out of sync with the implementation's
persistence layer.

A New Plugin 'opencontrail' has been created. A new file
'contrail_plugin.py' has been added to implement the core extension
RESTApi.

The configuration file contrailplugin.ini will be added to configure the
Opencontrail plugin. The file will contain the following parameters.

1 api_server_ip
    The Opencontrail controller IP address
2 api_server_port
    The Opencontrail controller IP port
3 multi-tenancy
    Multi-tenancy support
4 max_retries
    The maximum number of tries to connect to the Opencontrail controller
5 retry_interval
    Retry interval to connect to the Opencontrail controller

Example:
[API_SERVER]
api_server_ip = 10.10.10.10
api_server_port = 8082
multi_tenancy = False

The Class 'NeutronPluginContrailCoreV2' is added to implement the core RESTApi.
It will implement the following extensions and their RESTApis

Network
  - create_network
  - update_network
  - delete_network
  - get_network
  - get_networks
  - get_networks_count

Subnet
  - create_subnet
  - update_subnet
  - delete_subnet
  - get_subnet
  - get_subnets
  - get_subnets_count

Port
  - create_port
  - update_port
  - delete_port
  - get_port
  - get_ports
  - get_ports_count

Router
  - create_router
  - update_router
  - delete_router
  - get_router
  - get_routers
  - get_routers_count

Floatingip
  - create_floatingip
  - update_floatingip
  - delete_floatingip
  - get_floatingip
  - get_floatingips
  - get_floatingips_count

SecurityGroup
  - create_security_group
  - update_security_group
  - delete_security_group
  - get_security_group
  - get_security_groups
  - get_security_groups_count

Assignee(s)
-----------

Primary assignee:
  praneetb

Other contributors:
  hajay

Work Items
----------

1. Opencontrail plugin implementation
2. Opencontrail mocks for unit-tests

Dependencies
============

None.

Testing
=======

Existing and new (added in Juno) Neutron unit tests will be used.
Existing and new tempest testing for neutron will be used.
Tempest CI environment will be provided.

Documentation Impact
====================

Once the opencontrail neutron plugin is accepted, neutron documentation can
change to say opencontrail plugin as one of the supported plugin.

The link below explains setup of Opencontrail using devstack.
http://pedrormarques.wordpress.com/2013/11/14/using-devstack-plus-opencontrail/

References
==========

http://www.opencontrail.org
https://github.com/Juniper/contrail-controller
