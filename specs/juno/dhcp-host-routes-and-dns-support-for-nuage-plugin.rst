
===========================================================================
Adding support for host-routes and dns_nameservers options for Nuage Plugin
===========================================================================

https://blueprints.launchpad.net/neutron/+spec/dhcp-host-routes-and-dns-support-for-nuage-plugin

Adding support for host-routes and dns_nameservers options via DHCP options
for the Nuage Plugin


Problem description
===================

The current the Nuage Plugin does not support adding host routes or
DNS nameservers via DHCP options for a Neutron subnet.


Proposed change
===============
Currently the Nuage Plugin does not support Neutron's adding host routes or DNS
nameservers via DHCP options on a subnet.

The Nuage's VSP supports this feature and the support needs to be added in the
plugin code.

The following DHCP options will be supported :
 - DNS nameserver
 - Host routes

The proposed change is to support the creation of DNS nameserver and/or Host
routes using the Neutron DNS and Host routes

For example:
  neutron subnet-create test 192.168.10.0/24\
    --dns_nameservers list=true 8.8.4.4 8.8.8.8

This action will create a subnet name test with a CIDR 192.168.10.0/24.
The nameservers 8.8.4.4 8.8.8.8 will added to this subnet, this translate to a
subnet in the Nuage's VSP subnet with the same nameservers

The CRUD operations will be supported by the Nuage's VSP plugin for both
DNS nameserver and Host routes.



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
None

Other deployer impact
---------------------
None

Developer impact
----------------
None

Implementation
==============

The modification are required on the plugin.py

* The new method __create_port_gateway will be created.
	This method will create a port of type network:dhcp for the current \
	subnet and tenant
* The __validate_create_subnet method will be modified to allow the host_routes key to be a valid options of the subnet dictionary
* The update_subnet method will be added to the plugin
* The  _create_nuage_subnet method will be merged into the create_subnet method. This will newly updated method will also create a DHCP port using the method _create_port_gateway described above.


Assignee(s)
-----------
Franck Yelles


Primary assignee:
  fyelles

Other contributors:

Work Items
----------
* Extension code in Nuage plugin
* Nuage CI coverage addition
* Nuage Unit tests additions, the following test units will be updated/added :
  * test_create_subnet_bad_hostroutes
  * test_update_subnet_adding_additional_host_routes_and_dns
  * test_create_subnet_with_one_host_route
  * test_create_subnet_with_two_host_routes
  * test_create_subnet_with_too_many_routes
  * test_update_subnet_route
  * test_update_subnet_route_to_None
  * test_update_subnet_route_with_too_many_entries
  * test_delete_subnet_with_route
  * test_delete_subnet_with_dns_and_route
  * test_validate_subnet_host_routes_exhausted
  * test_validate_subnet_dns_nameservers_exhausted


Dependencies
============
None

Testing
=======
Unit Test coverage for the DHCP options within Nuage unit test
Nuage CI will be modified to start supporting this extension tests


Documentation Impact
====================
None

References
==========
None
