..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
SSL-VPN Support for Neutron VPNaaS
==========================================

https://blueprints.launchpad.net/neutron/+spec/neutron-ssl-vpn


Problem description
===================

OpenStack Neutron today provides VPN as a Service feature for IPsec based VPNs.
It currently only supports Site-to-Site VPNs.
This blueprint will define and implement the SSL VPN for
Remote Clients or Remote Warriors.

Proposed change
===============

In this blueprint, we are going to add support for SSL VPN. We are going to add SSL-VPN
related data models, and agent.
For agent implementation, we will use IPsec implementation as a basement.

Alternatives
------------

User can run SSL-VPN process in his VM, however he can't support bridge model because
of ip-spoofing functionalities.


Data model impact
-----------------

We are going to add SSLVPNConnection resources.
This is a relationship diagram with existing objects.


.. blockdiag::

  blockdiag model {
    SSLVPNConnection -> VPNService -> Router;
    SSLVPNConnection -> Secret;
  }


The connection model of SSL-VPN implementation is based on
what we done in IPsec.
We are going to use Barbican as a secure credential store, so
SSLVPNConnection have a link to the secret resource in Barbican.

Note, Delete operation of dependent resource should be prohibited.

We will add SSLVPNConnection database table and a proper migration script.

REST API impact
---------------

This is attributes for SSLVPNConnection.

.. csv-table:: SSLVPNConnection
   :header: Attribute Name,Type,Access,Default Value,Validation/Constraint,Description,

   id,uuid-str,R,generated,N/A,UUID for VPNService Object,
   tenant_id,uuid-str,"RW, owner",None,valid tenant_id,UUID of the tenant for the vpn service,
   name,string,"RW, owner",None,N/A,name of the VPN Service,
   status,string,R,N/A,N/A,Indicates whether ipsec vpnservice is currently operational. Possible values include:,
   ,,,,,ACTIVE DOWN BUILD ERROR,
   admin_state_up,bool,"RW, owner,admin",TRUE,true/false,"Administrative state of vpnservice. If false (down), port does not forward packets",
   client_address_pool_cidr,cidr,R,N/A,Valid cidr,Client address pool subnet which will be used by sslvpn client,
   credential_id,uuid-str,R,N/A,valid vpn credential id,UUID for Barbican Secret,
   vpnservice_id,uuid-str,R,N/A,valid vpn service id,UUID for VPNService,
   port_no,int,RW,N/A,non-negative,port number which sslvpn listen on,




Security impact
---------------

In the blueprint, we should deal with private keys for SSL-VPN. If this key get stolen,
all encrypted traffic could be stolen, so the management of these key is really important.
The access to the keys should also be monitored.
For auditing, we are going to use Barbican. We are going to store all credential in the Barbican API, and
vpn-agent will get private key on demand using the Barbican API.
We will use ephemeral disc such as RAM disc to store the secret key in the agent, and we will set
proper file permission on it.


Notifications impact
--------------------

CRUD event for SSLVPNConnection/Barbican Credential will be also notified for vpn-agent.


Other end user impact
---------------------

We will add a support for SSL-VPN.

This is a list of commands we are adding to the client.

::

  vpn-credential-create          Create an VPNCredential.
  vpn-credential-delete          Delete a given VPNCredential.
  vpn-credential-list            List VPNCredentials that belong to a given tenant.
  vpn-credential-show            Show information of a given VPNCredential.
  vpn-credential-update          Update a given VPNCredential.


ssl-vpn-connection-create::

  usage: neutron ssl-vpn-connection-create [-h] [-f {shell,table}] [-c COLUMN]
                                           [--variable VARIABLE]
                                           [--prefix PREFIX]
                                           [--request-format {json,xml}]
                                           [--tenant-id TENANT_ID]
                                           [--admin-state-down] [--name NAME]
                                           [--client_address_pool_cidr CLIENT_ADDRESS_POOL_CIDR]
                                           VPNSERVICE VPNCREDENTIAL


ssl-vpn-connection-list::

  usage: neutron ssl-vpn-connection-list


ssl-vpn-connection-show::

    usage: neutron ssl-vpn-connection-show [-h] [-f {shell,table}] [-c COLUMN]
                                       [--variable VARIABLE] [--prefix PREFIX]
                                       [--request-format {json,xml}] [-D]
                                       [-F FIELD]
                                       SSL_VPN_CONNECTION


ssl-vpn-connection-update::

    usage: neutron ssl-vpn-connection-update [-h] [--request-format {json,xml}]
                                             SSL_VPN_CONNECTION


ssl-vpn-connection-delete::

    usage: neutron ssl-vpn-connection-delete [-h] [--request-format {json,xml}]
                                         SSL_VPN_CONNECTION

Performance Impact
------------------

This extension add two performance impact. The first impact is database relationship for
SSL-VPN. We introduce table join, this may impact the performance.
The second impact is ssl-vpn resource related notification. vpn-agent need to know
crud event of SSL-VPN related resources.


Other deployer impact
---------------------

- We need openvpn setup
- A deployer needs to use VPNService implementation who supports ssl-vpn

Developer impact
----------------

- A VPN Service plugin may support ssl vpn extension

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Nachi Ueno

Other contributors:
  Rajesh Mohan
  Swaminathan Vasudevan

Work Items
----------

- SSL-VPN extension (Service Side)
- SSL-VPN agent (on l3-agent)
- Client Support
- Horizon

Dependencies
============

* We will use OpenVPN as a ssl-vpn Client
* This implementation depends on namespace based L3-agent


Testing
=======

* Proper UT for CRUD for resources with dependent resources
* create ssl vpn connection in two router, connect each other
* try to use IPsec & SSL-VPN same time

Documentation Impact
====================

New resource model and how to use SSL-VPN extension should be
documented.

References
==========

- Barbican https://wiki.openstack.org/wiki/Barbican
- OpenVPN http://openvpn.net/

