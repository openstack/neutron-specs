..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
LBaaS API and Object Model improvement
======================================

https://blueprints.launchpad.net/neutron/+spec/lbaas-api-and-objmodel-improvement

LBaaS needs improved API and object model to provide base line for
new API parts that provide advanced use cases such as L7, TLS, HA, etc.

This blueprint describes the changes that should be made to object model
so that further design and implementation of L7 switching, TLS and HA feature is
possible.  The new API that exposes the new entities will be done in an separate
extension.

This blueprint is not describing design of L7, TLS API parts but may briefly
assume some aspects of possible design of those features. Nor does it describe
the changes to the API that will fit with this object model.

Problem Description
===================

The "advanced" LB configuration which is supported by all LB vendors,
both hw and sw, may consist of multiple service endpoints on one ip address,
and multiple pools. Here, 'service endpoint' means IP address + port.

The problem with existing API/object model is that it only accounts for
single VIP and single pool per loadbalancer.

One of the biggest issue of the existing API and object model is that
Pool is playing 'root object' role and this solely prevents multiple pools
per single configuration.


Proposed Change
===============

There are a few things that needs to be done in order to make LBaaS API
suitable for addition of TLS, L7 and HA features.

1. This will entirely be a new extension and service plugin.  LBaaS V1 code
should not be affected, but there may be exceptions.

2. Currently Pool object is the root object(*), which is logically incorrect
and confusing. From API perspective that means that creating Pool object will
not be a starting point of the workflow.

3. Member object will add subnet_id as an optional attribute.

4. VIP object will not be used.  Its attributes will be added to the
LoadBalancer object and the Listener object.

5. LoadBalancer object becomes the root object(*) of the LBaaS object model. It
will hold the attributes that pertain to the vip and will be the parent of one
or many listeners.

6. Additional object Listener is introduced,
which is a placeholder for former VIP parameters such as protocol_port,
and protocol, see detailed description below

LoadBalancer and Listener relate as 1:M. Listener will initially only be a
child of a LoadBalancer, so its life-cycle is limited to a LoadBalancer.
Deleting a LoadBalancer will not be alllowed when it has children Listeners.

In the future, this relationship may become M:N if it is decided it is needed.
The M:N change will be an additive change so it will not break contract,
however, this blueprint will not attempt this.

7. To prevent entities in the database being out of sync with the backend due
to concurrent requests, no operations will be allowed if an entity is in a
transient state (PENDING_CREATE, PENDING_UPDATE, PENDING_DELETE).

8. Pool to Health Monitor relationship will change from M:N to 1:1 for now.

9. Attributes of health monitor and member will stay nearly identical except
for references to parents.

10. A new synchronous haproxy driver will be created to easily test out the
API and DB changes.  It will not be meant for production use.  Refactoring the
agent haproxy driver will be left to another blueprint.

11. Since there are two types of statuses we should worry about, LoadBalancer
and Listener will have both provisioning_status and operating_status fields,
while Pool and Member will have an operating_status field.

12. operating_status will be an enum of ('ONLINE', 'OFFLINE', 'DEGRADED',
'ERROR')

13. provisioning_status will be an enum of ('ACTIVE', 'PENDING_CREATE',
'PENDING_UPDATE', 'PENDING_DELETE', 'ERROR')

14. Every update of a LoadBalancer, including updates to and adding/removing
children entities, will put the LoadBalancer into a PENDING_UPDATE
provisioning_status.  No other updates to that LoadBalancer or its children
will be allowed until the operation has completed.

(*) Root object is an object that represents 'service instance'.
It could be deployed/undeployed, turned on/off, or capability requirements
may be applied to it. It also is a starting point of configuration workflow.


Alternatives
------------

* https://docs.google.com/a/mirantis.com/document/d/1mTfkkdnPAd4tWOMZAdwHEx7IuFZDULjG9bTmWyXe-zo/edit
  The documents describes differently structured API that uses 'single-call' approach.
* https://etherpad.openstack.org/p/neutron-lbaas-api-proposals
  Brief description of how other APIs for other object models look like
* http://lists.openstack.org/pipermail/openstack-dev/2014-November/051147.html
  Samuel Bercovici's idea of solving the status issue of shared objects

Data Model Impact
-----------------
LBaaS v1 tables will remain unchanged and the following tables will be added
for LBaaS v2 use:
* neutron.lbaas_loadbalancers
* neutron.lbaas_listeners
* neutron.lbaas_pools
* neutron.lbaas_members
* neutron.lbaas_healthmonitors
* neutron.lbaas_sessionpersistences
* neutron.lbaas_listener_statistics

1. lbaas_loadbalancers

* id - unique identifier
* tenant_id
* name
* description
* vip_port_id - the neutron port tied to the vip (this can be used to get the
  vip_subnet, and vip_address).  This will be stored in the database but not
  exposed through the API.
* vip_subnet_id - the subnet a neutron port should be created on
* vip_address - the address of the subnet
* provisioning_status
* operating_status
* admin_state_up
* listeners - a list of back-references child listener ids
* provider - provider in which this load balancer should be provisioned

2. lbaas_listeners

* id - identity
* tenant_id
* loadbalancer_id
* default_pool_id - ID of default pool. Must have compatible protocol with
  listener.
* protocol - Protocol to load balancer: HTTP, HTTPS, TCP, UDP
* protocol_port - port number to listen
* admin_state_up - admin state (True or False)
* provisioning_status
* operating_status

Listener model will later be amended with L7 and TLS-related attributes which
are out of scope of this blueprint.

3. lbaas_pools

* id - identity
* tenant_id
* name
* description
* protocol - Protocol to load balance
* lb_method - load balancing method
* healthmonitor_id - id of health monitor
* admin_state_up - admin state (True/False). That attribute defines
  administrative state of the pool on all of the backends where it is actually
  deployed.
* operating_status

4. lbaas_members

* id - identity
* tenant_id
* address - ip address
* pool_id - required parent pool
* subnet_id - optional subnet this member is on
* protocol_port
* weight
* operating_status
* admin_state_up

5. lbaas_healthmonitors

* id - id
* tenant_id
* type - (TCP, HTTP)
* delay
* timeout
* max_retries
* http_method
* url_path
* expected_codes
* admin_state_up

6. lbaas_sessionpersistences

* pool_id
* type - (HTTP_COOKIE, SOURCE_IP, APP_COOKIE)
* cookie_name

7. lbaas_listener_statistics

* listener_id
* bytes_in
* bytes_out
* active_connections
* total_connections


REST API Impact
---------------

A separate extension will be created exposing the following resources:

* /lbaas/loadbalancers
* /lbaas/listeners
* /lbaas/pools
* /lbaas/pools/{pool_id}/members
* /lbaas/healthmonitors

Resource Attributes:

/lbaas/loadbalancers


.. csv-table:: CSVTable
    :header: Attribute Name,Type,Access,Default Value,Validation Conversion,Description

    id,string (UUID),"RO, all",generated,N/A,identity
    tenant_id,string (UUID),"RW, all (No Update)",generated,string,tenant identity
    name,string,"RW, all",'',string,Human-readable
    description,string,"RW, all",'',string,Human-readable
    vip_subnet_id,string (UUID),"RW, all (No Update)",required,string,creates vip on this subnet
    vip_address,string (IP Address),"RW, all (No Update)",generated,IPv4 or IPv6,Frontend IP address
    admin_state_up,bool,"RW, all",True,bool,enabled
    provisioning_status,string,"RO, all",N/A,provisioning status
    operating_status,string,"RO, all",N/A,operational status


Deleting a load balancer will only succeed if it is not a parent of any
listeners.


/lbaas/listeners


.. csv-table:: CSVTable
    :header: Attribute Name,Type,Access,Default Value,Validation Conversion,Description

    id,string (UUID),"RO, all",generated,N/A,identity
    tenant_id,string (UUID),"RW, all",generated,string,tenant identity
    name,string,"RW, all",'',string,Human-readable
    description,string,"RW, all",'',string,Human-readable
    loadbalancer_id,string (UUID),"RW, all (No Update)",required,string,parent load balancer
    connection_limit,integer,"RW, all",-1,integer,max connections to the protocol port
    protocol,string,"RW, all (No Update)",required,"'TCP','HTTP','HTTPS'",listening protocol
    protocol_port,integer,"RW, all",required,0-65535,listening port
    admin_state_up,bool,"RW, all",True,bool,enabled
    provisioning_status,string,"RO, all",N/A,provisioning status
    operating_status,string,"RO, all",N/A,operational status

Note that loadbalancer_id is required for now.  This will not preclude later
implementations that may want to allow M:N loadbalancer to listeners as this
is only an API attribute and thus can be changed from required to optional.

Note that default_pool_id is not specified here as the pool will define
its parent listener.

Deleting a Listener will only succeed if it is not the parent of a pool.


/lbaas/pools


.. csv-table:: CSVTable
    :header: Attribute Name,Type,Access,Default Value,Validation Conversion,Description

    id,string (UUID),"RO, all",generated,N/A,identity
    tenant_id,string (UUID),"RW, all",generated,string,tenant identity
    name,string,"RW, all",'',string,Human-readable
    description,string,"RW, all",'',string,Human-readable
    listener_id,string (UUID),"RW, all (No Update)",required,string,parent listener
    protocol,string,"RW, all (No Update)",required,"'TCP','HTTP','HTTPS'",protocol to send to members
    lb_algorithm,string,"RW, all",required,"'ROUND_ROBIN','LEAST_CONNECTIONS',SOURCE_IP",load balancing algorithm
    session_persistence,dictionary,"RW, all",{},"type: 'SOURCE_IP','HTTP_COOKIE','APP_COOKIE', cookie_name: string",session persistence definition
    admin_state_up,bool,"RW, all",True,bool,enabled
    operating_status,string,"RO, all",generated,N/A,operational status
    members,list,"RO, all",generated,N/A,list of members belonging to this pool


Note that listener_id is required for now.  There will be validation that
the listener has only one pool as a child.

This should not preclude a later implementation of M:N listener to pools.

Deleting a pool will not succeed if it is the parent of a health monitor.  It
will however succeed if it is the parent of any children, and those children
will be deleted as well.


/lbaas/pools/{pool_id}/members


.. csv-table:: CSVTable
    :header: Attribute Name,Type,Access,Default Value,Validation Conversion,Description

    id,string (UUID),"RO, all",generated,N/A,identity
    tenant_id,string (UUID),"RW, all",generated,string,tenant identity
    address,string (IP),"RW, all (No Update)",required,IPv4 or IPv6,IP Address member is listening
    protocol_port,integer,"RW, all",required,0-65535,port member is listening
    weight,integer,"RW, all",1,integer,traffic distribution weight
    subnet_id,string (UUID),"RW, all (No Update)",None,string,subnet to access member port
    admin_state_up,bool,"RW, all",True,bool,enabled
    operating_status,string,"RO, all",generated,N/A,operational status



/lbaas/healthmonitors


.. csv-table:: CSVTable
    :header: Attribute Name,Type,Access,Default Value,Validation Conversion,Description

    id,string (UUID),"RO, all",generated,N/A,identity
    tenant_id,string (UUID),"RW, all",generated,string,tenant identity
    type,string,"RW, all (No Update)",required,"'HTTP','HTTPS','PING','TCP'",type of health check
    pool_id,string (UUID),"RW, all (No Update)",required,string,id of pool to monitor
    delay,integer,"RW, all",required,integer,seconds before health check
    timeout,integer,"RW, all",required,integer,seconds for a failed check
    max_retries,integer,"RW, all",required,integer,number of failed checks before member is considered OFFLINE
    http_method,string,"RW,all",'GET',"'GET','POST','PUT'",http verb to send http checks
    url_path,string,"RW, all",'/',string,url to send http checks
    expected_codes,string,"RW, all",'200',comma delimited HTTP response codes,expected HTTP response codes for a successful check
    admin_state_up,bool,"RW, all",TRUE,bool,enabled


Note that pool_id is a required attribute.  Similar to listener_id on the pool
object, this does not preclude later implementations of 1:M pool to health
monitor relationship.

(*) denotes a required attribute

Security Impact
---------------

Standard Neutron tenant object ownership rules will apply.


Notifications Impact
--------------------

None

New notifications:
- loadbalancer
- listener
- pool
- healthmonitor
- member


Other End User Impact
---------------------

Users may have access to a new lbaas resources.  The CLI commands will be
slightly different depending on which extension that is loaded.


Performance Impact
------------------

None

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

LBaaS V1 and LBaaS V2 can coexist in the codebase, but should not be run at
the same time.  This will have to be enforced in the code.

No migration path from LBaaS V1 to V2 will be done for this blueprint, however
another blueprint should do this as it will be a complicated effort.


Developer Impact
----------------

A new extension, service plugin, and driver.


Community Impact
----------------

This change has been in review since Juno.  Much discussion has taken place
over IRC and the mailing list.


Implementation
==============

Assignee(s)
-----------

Primary assignees:
  brandon-logan

Work Items
----------

* object model change
* new loadbalancer extension for new API
* unit tests

Dependencies
============

None


Testing
=======

Tempest Tests
-------------

https://review.openstack.org/#/c/106089/

Functional Tests
----------------

Functional Tests for the load balancer plugin.

API Tests
---------

All new resources added by the extension will have positive and negative
tests.


Documentation Impact
====================

User Documentation
------------------

Differences between LBaaS V1 and V2 should be documented.

Developer Documentation
-----------------------

There should be a new LBaaS V2 API document section documenting the new
resources added by the extension.


References
==========

* https://etherpad.openstack.org/p/juno-lbaas-design-session
