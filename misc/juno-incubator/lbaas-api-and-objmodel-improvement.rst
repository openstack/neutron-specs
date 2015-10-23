..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
LBaaS API and Object Model improvement
==========================================

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

Problem description
===================

The "advanced" LB configuration which is supported by all LB vendors,
both hw and sw, may consist of multiple service endpoints on one ip address,
and multiple pools. Here, 'service endpoint' means IP address + port.

The problem with existing API/object model is that it only accounts for
single VIP and single pool per loadbalancer.

One of the biggest issue of the existing API and object model is that
Pool is playing 'root object' role and this solely prevents multiple pools
per single configuration.


Proposed change
===============

There are a few things that needs to be done in order to make LBaaS API
suitable for addition of TLS, L7 and HA features.

1. Currently Pool object is the root object(*), which is logically incorrect
and confusing. Need to make Pool a pure-db object.
From API perspective that means that creating Pool object will not be a
starting point of the workflow.

So the following changes are needed for Pool resource:

* "provider" attribute of Pool will be marked as deprecated,
  "provider" is an attribute that belong to root object.
  Also there is a work that tries to get rid of this way of
  specifying service: https://review.openstack.org/#/c/90070/
* vip_id attribute removed.
* subnet_id attribute removed

2. Member object will add subnet_id as an optional attribute.

3. VIP object will be removed.  Its attributes will be added to the
LoadBalancer object and the Listener object.

4. LoadBalancer object becomes the root object(*) of the LBaaS object model. It
will hold the attributes that pertain to the vip and will have a links to its
children listeners.

5. Additional object 'Listener' is introduced,
which is a placeholder for former VIP parameters such as protocol_port,
protocol, protocol-specific paratemers such as TLS stuff, see detailed
description below

LoadBalancer and Listener relate as 1:M. Listener is a first-class object,
so its life-cycle is not limited to a LoadBalancer. Deleting
LoadBalancer will remove its relationship to its listeners.

In the future, this relationship may become M:N if it is decided it is needed.
This blueprint will not attempt this.

6. A translation layer is needed for backwards compatibility when the legacy
calls are made, such as creating a VIP.  When a user creates a VIP, it's
attributes must be mapped to both a LoadBalancer and Listener.  This means
creating a VIP will end up creaing a LoadBlaancer and Listener.  Health monitor
and member subnet_id change will also need to be translated.

7. A translation layer from plugin to driver will also be needed to translate
from old objects to new for drivers other than the namespace driver.

8. No operations will be allowed if an entity is in a transient state
(PENDING_CREATE, PENDING_UPDATE, PENDING_DELETE).

(*) Root object is an object that represents 'service instance'.
It could be deployed/undeployed, turned on/off, or capability requirements
may be applied to it. It also is a starting point of configuration workflow.


Alternatives
------------

* https://docs.google.com/a/mirantis.com/document/d/1mTfkkdnPAd4tWOMZAdwHEx7IuFZDULjG9bTmWyXe-zo/edit
  The documents describes differently structured API that uses 'single-call' approach.
* https://etherpad.openstack.org/p/neutron-lbaas-api-proposals
  Brief description of how other APIs for other object models look like

Data model impact
-----------------
Add DB tables:
* neutron.loadbalancers
* neutron.listeners

Existing DB tables changed:

neutron.vips
* remove this table

neutron.pools
* remove vip_id field
* remove subnet_id
* add health_monitor_id
* migrate vip_id to appropriate loadbalancer and listener

neutron.members
* add subnet_id
* copy parent pool subnet_ids to member subnet_ids

neutron.poolmonitorassociations
* remove this table

neutron.poolstatisticss
* rename to loadbalancerstatistics

neutron.poolagentbindings
* rename to loadbalanceragentbindings

Migration moving to new tables is required.
In particular, migration will add listeners and loadbalancers.  The old VIP
object's pool will be moved to the Listener and that Listener will be linked to
the LoadBalancer.
Pool subnet_ids will be copied to all of its members now that subnet_id is an
attribute of a Member.
Also, mapping between drivers and pools changed to mapping between drivers and
loadbalancers.
If a neutron port is encountered that has many fixed_ips then a load balancer
should be created for each fixed_ip with each being a deep copy of each other.
Existing pool ids will be copied to loadbalancer ids because backends use
the pool id to name objects.  Since the root object will be a load balancer now,
it is easiest to just use the same id instead of doing migrations on the
backend.


New DB models introduced:

1. LoadBalancer
Attributes:
* id - unique identifier
* name
* description
* vip_port_id - the neutron port tied to the vip (this can be used to get the
vip_subnet, and vip_address).  This will be stored in the database but not
exposed through the API.
* vip_subnet_id - the subnet a neutron port should be created on
* vip_address - the address of the subnet
* tenant_id
* status
* admin_state_up
* listeners - a list of back-references child listener ids
* provider - provider in which this load balancer should be provisioned

2. Listener.
Listener object receives some protocol-specific properties of
the former VIP.

Attributes:
* id - identity
* tenant_id
* loadbalancer_id
* default_pool_id - ID of default pool. Must have compatible protocol with
listener.
* protocol - Protocol to load balancer: HTTP, HTTPS, TCP, UDP
* protocol_port - port number to listen
* admin_state_up - admin state (True or False)
* status - operational status (ACTIVE, PENDING_CREATE, PENDING_DELETE, etc)

Listener model will later be amended with L7 and TLS-related attributes which
are out of scope of this blueprint.

Changed DB Models:

1. VIP
Removed.

2. Pool.

vip_id will be removed
subnet_id will be removed

Attributes:

* id - identity
* name
* description
* protocol - Protocol to load balance
* lb_method - load balancing method
* health_monitor_id - id of health monitor
* admin_state_up - admin state (True/False). That attribute defines
  administrative state of the pool on all of the backends where it is actually
  deployed.
* members - a list of back-references to child pool member ids.
* monitor - relationship to health monitors (list of monitor ids), this should
  be deprecated in favor of a M:1.  Should this be left to another BP since this
  is already implemented?

3. Member
subnet_id will be added

asciiflow::

                        +-----------------+
                        | Listener        |
  +----------------+    |                 |
  | LoadBalancer   |    | id              |     +-------------------+
  |                |    | tenant_id       |     | Pool              |
  | id             |<---| loadbalancer_id |     |                   |
  | name           |    | default_pool_id |---->| id                |<--\
  | description    |    | protocol        |     | name              |   |
  | vip_port_id    |    | protocol_port   |     | description       |   |
  | vip_subnet_id  |    | admin_state_up  |     | lb_method         |   |
  | vip_address    |    | status          |  /--| health_monitor_id |   |
  | tenant_id      |    +-----------------+  |  | admin_state_up    |   |
  | status         |                         |  +-------------------+   |
  | admin_state_up |                         |                          |
  +----------------+                         |                          |
                                             |                          |
                                             |                          |
   /-----------------------------------------/                          |
   |                                                                    |
   |   +----------------+   /-------------------------------------------/
   |   | HealthMonitor  |   |
   |   |                |   |  +----------------+
   \-->| id             |   |  | Member         |
       | type           |   |  |                |
       | delay          |   |  | id             |
       | timeout        |   \--| pool_id        |
       | max_retries    |      | address        |
       | http_method    |      | protocol_port  |
       | url_path       |      | subnet_id      |
       | expected_codes |      | weight         |
       | admin_state_up |      | admin_state_up |
       +----------------+      +----------------+


REST API impact
---------------

A separate extension will be created that will expose the /loadbalancers and
/listeners resources.  Both the old extension and new extension will be active
at the same time.

Security impact
---------------

Standard Neutron tenant object ownership rules will apply.


Notifications impact
--------------------

None

Notifications will be impacted because the payload will change and other
notifications will go away. New notifications:
- loadbalancer
- listener
- pool
Removed notifications: VIP


Other end user impact
---------------------

Compatibility will be preserved between REST calls to v1 and v2. A user should
not intermix v1 and v2 calls. Additionally the following M:N relationships will
not be supported v1 calls are used.


Performance Impact
------------------

None


Other deployer impact
---------------------

Deployer should be able to migrate to new code without additional actions
other than running a DB migration.  Deployers will have
to run a migration check to verify that their existing deployment can be
migrated.  If it cannot it will fail and they should not proceed until they
fix the issues.


Developer impact
----------------

New object model will not be backwards compatible with the old one.  However,
the REST API will still behave as previously expected.  If a user tries to
create many health monitors on a pool then it will fail in the old API.


Implementation
==============

Assignee(s)
-----------

Primary assignees:
  brandon-logan
  enikanorov

Work Items
----------

* object model change - single patch
* translation layer from existing API to new object model
* translation layer from new object model to drivers
* new loadbalancer extension for new API
* migration check for Many to Many health monitors
* data migration vips to load balancers and listeners
* database table additions and drops

Dependencies
============

None


Testing
=======

The change expected to be fully backward-compatible.
Existing tests should be able to pass.
New tests for /loadbalancers and /listeners will need to be created.
- Tempest API tests
- Tempest scenario tests
- Neutron functional tests


Documentation Impact
====================

Loadbalancers and Listeners resources will be added and need documentation as
another version of the API.


References
==========

* https://etherpad.openstack.org/p/juno-lbaas-design-session
