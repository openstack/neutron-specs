..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Fix the dns-assignment on neutron port
======================================

https://bugs.launchpad.net/neutron/+bug/1873091

The dns-domain part of the dns-assignment for the neutron port is always taken
from the dns-domain defined in the neutron configuration

Problem Description
===================
When the user enable the dns or dns_domain_ports extension there are multiple ways to
define Designate dns_domain (Designate zone) in Neutron side as follows:

* Neutron configuration file which contain the default value for dns_domain
* Neutron network definition which define the dns_domain for network and enable the dns
  extension for that network so every port created under the network will have a dns
  record in that dns_domain and it override the default value in configuration file
* Neutron port definition when the dns_domain_ports extension is enabled which is used
  to override the Neutron network dns_domain and create the dns record under another
  dns_domain

The dns-domain can be set on network level using `openstack network set --dns-domain`
and on port level using dns_domain_ports extension `openstack port create --dns-domain`
and both of those values (network level, port level dns-domians) are ignored when forming
the port dns-assignment which always take the dns-domain in neutron config

Proposed Change
===============

The port level dns-domain should take precedence over network level dns-domain which take
precedence over the neutron config dns-domain which will be the default if the above two levels
dont have dns-domain assigned

Testing
=======

Unit Test
---------
Unit test should be added to make sure that the right dns-domain is used in Neutron port dns assignment

Documentation Impact
====================

User Documentation
------------------
The documemtation is showing the dns assignment with the dns domain defined in the Neutron configurations
or defined in the Neutron network (which is usually the same) but not the dns_domain comming from Neutron
port creare (use case 3) which should be updated

References
==========

https://docs.openstack.org/neutron/latest/admin/config-dns-int-ext-serv.html#config-dns-int-ext-serv
