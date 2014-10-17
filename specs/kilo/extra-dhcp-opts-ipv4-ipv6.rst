..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Extra DHCP Options for IPv4 and IPv6
====================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/extra-dhcp-opts-ipv4-ipv6.rst

Problem Description
===================

A detailed description of the problem:

* Currently extra_dhcp_opts only works for DHCPv4 or DHCPv6 address on a
  port. Since a port can have both IPv4 and IPv6 address, we need to find
  a new way to specify extra dhcp options for DHCPv4 and DHCPv6, respectively.

* There is no validation of DHCP options, therefore any option name and value
  can be specified for a port.


Proposed Change
===============

* A new attribute "ip_version" will be used in Neutron port create/update
  extra_dhcp_opts API to specify the IP version of a given DHCP option.

* "ip_version" will be used to differentiate the option for DHCPv4 or DHCPv6
  when processing the DHCP options for a given port.

* How to deal with the case that ip_version is not specified:
  ip_version is default to 4 if ip_version is not specified for backward compatibility.

* Validation of DHCP option in DNSMASQ implementation:
  We can get a list of supported dhcp option names in DNSMASQ by:
  "dnsmasq --help dhcp" and "dnsmasq --help dhcp6".

  Unknown DHCP option names and wrong ip_version of a option (for example,
  IPv6 dns server with ip_version=4) should not be applied with clear error messages.

* Ignoring the options which cannot be applied
  If an option cannot be applied to a port due to the conflict between port's
  address and ip_version of the option, the option will be ignored until the
  port is updated with new addresses.

Data Model Impact
-----------------

* A new column "ip_version" will be added to Neutron extradhcpopts table with
  Integer type and nullable=True.

* Upgrade database migration requires adding this new column and downgrade
  database migration requires dropping this new column.

* The defult value of this column if not specified and the value after upgrade
  is null.

  ::

   +-----------+--------------+------+-----+---------+-------+
   | Field     | Type         | Null | Key | Default | Extra |
   +-----------+--------------+------+-----+---------+-------+
   | id        | varchar(36)  | NO   | PRI | NULL    |       |
   | port_id   | varchar(36)  | NO   | MUL | NULL    |       |
   | opt_name  | varchar(64)  | NO   |     | NULL    |       |
   | opt_value | varchar(255) | NO   |     | NULL    |       |
   | ip_version| int(11)      | YES  |     | NULL    |       |
   +-----------+--------------+------+-----+---------+-------+


REST API Impact
---------------

The modified extra_dhcp_opts attribute map is described as follows:

::
    POST /v2.0/ports
    Accept: application/json

    "port":{
       ...
       "extra_dhcp_opts": [
       {"opt_value": "pxelinux.0", "opt_name": "bootfile-name", "ip_version": 4},
       {"opt_value": "123.123.123.123", "opt_name": "tftp-server", "ip_version": 4},
       {"opt_value": "123.123.123.45", "opt_name": "server-ip-address", "ip_version": 4}
       ],
       ...}

The added attribute "ip_version" is not required. The allowed value of "ip_version" is 4 or 6.

Security Impact
---------------
None

Notifications Impact
--------------------
None

Other End User Impact
---------------------

Neutron client change: neutron port-create and neutron port-update needs following
changes:

::
   neutron port-create Network
     --extra-dhcp-opt opt_name=<dhcp_option_name>,opt_value=<value>,ip_version={4,6}

   neutron port-update Network
     --extra-dhcp-opt opt_name=<dhcp_option_name>,opt_value=<value>,ip_version={4,6}

Performance Impact
------------------
None

IPv6 Impact
-----------
This change is particularly designed for IPv6 tenant subnet ports.

Other Deployer Impact
---------------------
During data migration, ip_version will be set to 4 to keep backward
compability. The unsupported options will be ignored by DNSMASQ implementation.

Developer Impact
----------------

None

Community Impact
----------------

This proposed change has been discussed in Neutron IPv6 sub-team meeting
and community development mail list:

http://lists.openstack.org/pipermail/openstack-dev/2014-September/047222.html

Alternatives
------------
None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  xuhanp

Other contributors:
  TBD

Work Items
----------

* Add new API attribute and change current linux dhcp implementation by
  dnsmasq to treat IPv4 and IPv6 dhcp option separately.

* DB migration script to add "ip_version" column in Neutron extradhcpopts table.

* Add validation of extra dhcp options.

* Add "ip_version" attribute to Neutron client port create and port update CLI.


Dependencies
============

None


Testing
=======

Tempest Tests
-------------

Add tempest API test for Neutron port creation and update using the new
"ip_version" attribute of extra_dhcp_opts.

Functional Tests
----------------

Functional tests towards DHCPv4 and DHCPv6 option name, value and
the ip version.

API Tests
---------

API test to the added "ip_version" attribute of Port extra_dhcp_opts.


Documentation Impact
====================

Neutron API documation should be modified to add the new attribute.

User Documentation
------------------

The allowed DHCP option name and values should be documented in User
Documentation. Examples for frequently used options should be provided.

Developer Documentation
-----------------------

Developer API documentation should be updated with the new attribute.

References
==========

* Mail list discussion:
  http://lists.openstack.org/pipermail/openstack-dev/2014-September/047222.html

* DHCP options in DNSMASQ:
  http://www.thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html
