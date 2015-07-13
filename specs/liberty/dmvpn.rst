..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================
Neutron Dynamic Multipoint VPN
==============================

https://blueprints.launchpad.net/neutron/+spec/dynamic-multipoint-vpn

Problem Description
===================

Site to Site IPsec VPN is currently available in Neutron. It is primarily
used to connect a remote IPsec network to a Neutron tenant network. However in
many deployments there is a need to connect a tenant network to more than one
remote site and between each remote sites directly. To realize this using
existing Neutron VPNaaS constructs you need multiple site-to-site IPsec VPN
connection between a tenant network and every other sites in a fully meshed
model. This is cumbersome to configure and maintain. Imagine a new site coming
online where every other VPN site need to configured to connect to this new site.

Dynamic Multipoint VPN(DMVPN) based VPN is used in the deployments to alleviate
this. DMVPN introduces a hub and spoke model where the hub is configured once
and remote VPN sites are the spokes that connects to the hub using IPsec VPN.
DMVPN also facilitates automatic spoke-to-spoke connection that comes up
dynamically. Adding a new remote site has minimal impact in the configuration
of the other sites participating in the multipoint VPN. DMVPN uses
Next Hop Routing Protocol (NHRP) [1] to implement the hub and spoke
reachability. See [4] for an introduction of DMVPN technology.


Proposed Change
===============

The blueprint proposes to extends existing Neutron VPNaaS APIs to create DMVPN
hub and spoke. It leverages existing constructs to create IPsec attributes like
IKE policy, IPsec policy and uses them to create dynamic multipoint VPN instead
of the site to site IPsec connections.

This blueprint also proposes to introduce a reference implementation of DMVPN using
OpenNHRP daemon [2], Linux kernel based multipoint GRE (mGRE) and leverage
already existing StrongSwan for the underlying hub to spoke and spoke to spoke
connections.  Note, DMVPN itself is agnostic to the software package used for
IPsec connection and it could work with either OpenSwan or StrongSwan. However
given StrongSwan is the current actively maintained IPsec package and it is
supported in Neutron VPNaaS this spec explicitly calls out to use StrongSwan
option for DMVPN.  This will reduce amount of work required to support this for
both package given OpenSwan is slated for eventual deprecation.

 ::

                             +---------------------+
                             |                     |
                             |                     |
                             |                     |
                             |     Neutron Router  |
                             |                     |
                             |                     |
                             |                     |
                             |      DMVPN Hub      |
                             |                     |
                             |                     |
                             +-------+------+------+
                                     ^      ^
                                     |      |
                                     |      |
                                     |      |
                                     |      |
    +----------------------+         |      |          +---------------------+
    |                      |         |      |          |                     |
    |                      |         |      |          |                     |
    |                      |         |      |          |                     |
    |                      <---------+      +---------->                     |
    |  Neutron Router      |       ipsec tunnels       |  Neutron Router     |
    |                      +--------------------------->                     |
    |   DMVPN Spoke+1      |                           |   DMVPN Spoke+2     |
    |                      |                           |                     |
    |                      |                           |                     |
    |                      |                           |                     |
    +----------------------+                           +---------------------+

Alternatives
------------

As stated in the problem statement multiple point to point IPsec connections can be
used to build a similar topology. But it is operationally unfeasible as the number
of remote VPN sites grows.

Data Model Impact
-----------------

DMVPN Data Model will build on top of existing IKEPolicy, IPsecPolicy and VPNService
data-model. It will introduce a new DMVPNConnection resource.  DMVPNConnection will
have reference to IKEPolicy, IPsecPolicy and VPNService.

New database table for DMVPNConnection will be added with proper migration scripts.

REST API Impact
---------------

Here is the description of DMVPNConnection

.. csv-table:: DMVPNConnection
    :header: Attribute Name,Type,CRUD,Default Value,Validation/Constraint,Description

    id,uuid,R,Generated,UUID_PATTERN,id of DMVPNHubConnection resource
    tenant_id,uuid,R,,UUID_PATTERN,tenant_id of owner
    name,string,CRU,"",String,name of this connection
    description,string,CRU,,None,Description of dmvpn-hub-connection
    node_type,string,CR,"",String,type of dmvpn node. Possible values are HUB and SPOKE
    tunnel_key,integer, CRU, None, None, unique key for the DMVPN (maps to mGRE key). This should be same for all nodes in a multipoint vpn
    hub_address,list[dict{}],CRU,"",List with one dict entry, List of Hub NBMA & Tunnel addresses (v4 or v6). Required only when dmvpn node-type is SPOKE. Format [{hub_nbma_address=x.x.x.x, hub_tunnel_address=x.x.x.x}, ...]
    local_tunnel_address, string, CRU, None, None, local DMVPN tunnel address (v4 or v6) with prefix size.
    tunnel_auth_psk, string, CRU, None, None, Tunnel pre-shared-key (currently maps to IPSec psk). Should be same for all nodes in a multipoint vpn
    nhrp_auth, string, CRU, None, None, NHRP authentication key. Should be same for all nodes in a multipoint vpn
    ikepolicy_id,uuid,CR,N/A,uuid of ikepolicy,   uuid id of ikepolicy
    ipsecpolicy_id,uuid,CR,N/A, uuid of ipsecpolicy, uuid id of ipsecpolicy
    vpnservice_id,uuid,CR,N/A, uuid of vpnservice,  service id of vpnservice
    admin_state_up,bool,CRU,TRUE,"true / false",  Administrative state of dmvpn connection.
    status,string,R,N/A,N/A, Indicates whether dmvpn connection is currently operational. Possible values include: PENDING_CREATE ACTIVE DOWN  ERROR

.. csv-table:: DMVPNConnectionRoute
    :header: Attribute Name,Type,CRUD,Default Value,Validation/Constraint,Description

    dmvpn-id,uuid,CR,Generated,UUID_PATTERN, DMVPNHubConnection resource ID
    static_route_map, list[string], CRU, N/A, None, List of static route to each DMVPN node. Format  [(net1, nexthop1), (net2, nexthop2)]. Note, IP version should be same for all routes (all v4 or all v6)


Restrictions:

* hub_address - is currently proposed as a list of 2-tuple
  (hub_nbma_address, hub_tunnel_address). This is introduced to eventually support
  Dual Hub DMVPN deployments [6]. However for the initial release only one
  entry is allowed in this list. Support for Dual Hub VPN will come at a
  later enhancement of Neutron DMVPN.

* local_tunnel_address - should be unique within one DMVPN multipoint network.
  Also it cannot overlap with any subnets in the neutron router to which
  the DMVPN commection is attached to.

* Both site-to-site IPSec and DMVPN connection cannot be associated at the same
  time to a Neutron Router.

* DMVPN will not be supported if Neutron Router is configured in DVR mode.

Workflow
--------
The workflow involves creating IPsec tunnel related attributes first and then,
instead of create ipsec-site-connection, a dmvpn-connection will be created to
kick start a multipoint DMVPN network.


Security Impact
---------------
None

Notifications Impact
--------------------

The reference VPN service plugin will send a event notification for any CRUD on DMVPNConnection resource.

Other End User Impact
---------------------

We are also going to add support for this in python-neutronclient.
Here is a list of commands we will have add.

::

    # Tenant
    neutron dmvpn-connection-create [--tenant-id TENANT_ID]
                                    --name NAME
                                    [--description DESCRIPTION]
                                    [--admin-state-down]
                                    [--tenant-id TENANT_ID ]
                                    --node-type [HUB | SPOKE]
                                    --hub-address nbma=IP-ADDRESS,tunnel=IP-ADDRESS
                                    --local-tunnel-address IP-ADDRESS/PREFIX-SZ
                                    --tunnel-key DMVPN-KEY
                                    [--psk PSK]
                                    --vpnservice VPNSERVICE
                                    --ikepolicy-id IKEPOLICY
                                    --ipsecpolicy-id IPSECPOLICY
    neutron dmvpn-connection-update <dmvpn id or name>
                                    [--name NAME]
                                    [--description DESCRIPTION]
                                    [--admin_state_up True|False]
    neutron dmvpn-connection-show <dmvpn id or name>
    neutron dmvpn-connection-list
    neutron dmvpn-connection-delete <dmvpn id or name>


    neutron dmvpn-connection-route-add <dmvpn id or name>
                                    --type static --network CIDR --nexthop IP-ADDRESS
    neutron dmvpn-connection-route-remove <dmvpn id or name>
                                    --type static --network CIDR --nexthop IP-ADDRESS
    neutron dmvpn-connection-route-list

Performance Impact
------------------

There is no performance impact to neutron framework by introducing DMVPN.
There is zero impact to neutron when DMVPN is not configured. When
configured, similar to site-to-site ipsec connection, neutron-vpn-agent would
periodically collect status of DMVPN links to report back to the neutron server

Other Deployer Impact
---------------------

Current reference implementation plan is to use OpenNHRP. An Ubuntu PPA
package for OpenNHRP is available in [3]. Need to work with the packagers
to include this component in their distros.

Developer Impact
----------------

* Reference implementation will use OpenNHRP

Community Impact
----------------

N/A

IPv6 Impact
-----------

DMVPN is expected to work with IPv6. Following scenarios will be tested to
validate DMVPN using IPv6 works fine:

- using IPv6 address for Hub and Spoke endpoints
- using IPv6 peer CIDRs

Implementation
==============

Assignee(s)
-----------

Primary assignee:
    Sridhar Ramaswamy <srics-r>

Other contributors:
    Yanping Qu <yanping>
    Vikram Choudhary <vikschw>

Work Items
----------

- DMVPNConnection API Extension
- VPNaaS service driver and device driver for DMVPN
- Neutron vpn agent enhancements to implement DMVPN
- python-neutronclient enhancement for new DMVPN API
- Horizon UI changes for DMVPN configuration
- Documentation - User and Developer
- OpenNHRP package in the distros


Dependencies
============

* OpenNHRP for the reference implementation


Testing
=======

API Tests
---------

Following unit tests will be added,

 * Unit tests for CRUD operations for the new DMVPN connection
 * Unit test for reference service-driver and device-driver for DMVPN

Tempest Tests
-------------

Scenario tests currently is a work-in-progress for site-to-site ipsec vpn.
Once that is introduced a similar test will be added for a simple
dmvpn-hub <--> dmvpn-spoke connection test. A multi-point topology test
can be added at a later time with more than on dmvpn-spokes.

Functional Tests
----------------
Functional tests will be added to configure DMVPN connection and verify
whether appropriate NHRP configurations are correctly created.

Documentation Impact
====================

User Documentation
------------------

Following user facing documented will be updated,

 * Neutron API document to document the new DMVPN API
 * OpenStack deployment guide to how to configure hub and spokes correctly to
   successfully deploy DMVPN in OpenStack deployments

Developer Documentation
-----------------------

The developer documentation will be added to describe the integration of
OpenNHRP, mGRE and IPsec components used to implement DMVPN


References
==========

.. [1] RFC 2332 NBMA Next Hop Resolution Protocol: http://tools.ietf.org/html/rfc2332
.. [2] OpenNHRP project: http://sourceforge.net/projects/opennhrp/
.. [3] OpenNHRP PPA: https://launchpad.net/~thegner/+archive/ubuntu/opennhrp
.. [4] DMVPN Introduction: http://www.cisco.com/c/dam/en/us/products/collateral/security/dynamic-multipoint-vpn-dmvpn/prod_presentation0900aecd80313c9d.pdf
.. [5] Neutron DMVPN wiki: https://wiki.openstack.org/wiki/Neutron/VPNaaS/DMVPN
.. [6] Dual Hub DMVPN deployment: https://supportforums.cisco.com/document/32231/design-single-dmvpn-dual-hubs-redundant-path-over-internet
