..
..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Brocade Neutron L3 Plugin for Vyatta vRouter
=============================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/l3-plugin-brocade-vyatta-vrouter

This blueprint is for implementing an L3 service plugin for Brocade Vyatta
vRouter appliance.

Brocade Neutron L3 Plugin for Vyatta vRouter supports CRUD operations on
vRouter, add/remove interfaces from vRouter and floating IPs for VMs.
It performs vRouter VM lifecyle management by calling Nova APIs during the
Create and Delete Router calls. Once the vRouter VM is up, L3 plugin connects
to the REST API end-point exposed by the vRouter VM using REST API to perform
the appropriate configurations.L3 plugin supports add/remove router interfaces
by attaching/detaching the neutron ports to vRouter VM using Nova API.

Basic workflow is as shown below:

::

	+---------------------------+
	|                           |
	|                           |
	|     Neutron Server        |
	|                           |
	|                           |
	| +-----------------------+ |        +---------------+
	| | L3 Plugin for Brocade | |        |               |
	| | Vyatta vRouter        +---------->   Nova API    |
	| |                       | |        |               |
	+-+----------+------------+-+        +-------+-------+
	             |                               |
	             |                               |
	             |REST API                       |
	             |                               |
	             |                               |
	     +-------V----------+                    |
	     |                  |                    |
	     |  Brocade Vyatta  |                    |
	     |  vRouter VM      <--------------------+
	     |                  |
	     +------------------+

Problem description
===================

Cloud service providers want to use Brocade Vyatta vRouter as a tenant virtual
router in their OpenStack cloud. In order to perform the vRouter VM lifecycle
management and required configurations, a new Neutron L3 plugin for Brocade
Vyatta vRouter is required.

Proposed change
===============

Brocade Vyatta vRouter L3 plugin implements the below operations:

- Create/Update/Delete Routers
- Configure/Clear External Gateway
- Create/Delete Router-interfaces
- Create/Delete Floating-IPs

During the tenant router creation, L3 plugin will invoke nova-api by using the
admin tenant credentials mentioned in the plugin configuration file (More
details specified in deployer impact section). Nova-api is invoked to provision
Vyatta vRouter VM on-demand in admin tenant (Service VM tenant) by using the
tenant-id, image-id, management network name and flavor-id specified in plugin
configuration file. Vyatta vRouter VM's UUID is used while creating the Neutron
router so that router's UUID is the same as VM's UUID. During vRouter VM
creation, we will poll the status of the VM synchornously.Only when it becomes
'Active', we create the neutron router and the router creation process is
declared as successful.Once the vRouter VM is up, L3 plugin will use REST API
to configure the router name and administration state. If L3 plugin encounters
error from nova-api during vRouter VM creation or while using REST API to
communicate with the vRouter VM, router creation will fail and appropriate
error message is returned to the user.

When external gateway is configured, L3 plugin will create a neutron port in
external network and attach the port to vRouter VM using nova-api. Vyatta
vRouter image will recognize the hot-plugged interface. Once the port is
attached, L3 plugin will use REST API to configure the interface ip-address
on the ethernet interface. It will also create SNAT rules for all the private
subnets configured in router interfaces using REST API. SNAT rules and the
external gateway port will be deleted when external gateway configuration is
removed. If L3 plugin encounters error from nova-api during port attachment
or while using REST API to communicate with the vRouter VM, external gateway
configuration will fail and appropriate error message is returned to the user.

While adding a router interface, L3 plugin will create a neutron port in
tenant network and attach the port to vRouter VM using nova-api. Vyatta vRouter
image will recognize the hot-plugged interface. Once the port is attached,
L3 plugin will use REST API to configure the subnet on the ethernet interface.
It will also create SNAT rule for the router interface subnet using REST API
if external gateway is configured in the router. If L3 plugin encounters error
from nova-api during port attachment or while using REST API to communicate
with the vRouter VM, router interface addition will fail and appropriate error
message is returned to the user.

While deleting a router interface, L3 plugin will remove the ethernet interface
ip-address configuration, SNAT rule (if configured because of external gateway)
using REST API, detach and delete the neutron port in the tenant network.
If L3 plugin encounters error from nova-api during port detachment or while
using REST API to communicate with the vRouter VM, router interface deletion
will fail and appropriate error message is returned to the user.

When floating IPs are configured, L3 plugin will create SNAT and DNAT rules for
the translation between floating IPs and private network IPs using REST API.
SNAT and DNAT rules will be deleted when the floating IPs are disassociated.
If L3 plugin encounters error while using REST API to communicate with the
vRouter VM, floating IP configurations will fail and appropriate error message
is returned to the user.

While deleting the router, l3 plugin will first validate for non-existence of
router interfaces and external gateway configuration. It will delete the
router VM using Nova API and then delete the Neutron router.

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

While creating the Neutron router, end user has to wait for the vRouter VM
to be up (as it is spawned on-demand). This can take around 20 seconds.

Performance Impact
------------------

None

Other deployer impact
---------------------

1. Edit Neutron configuration file /etc/neutron/neutron.conf to specify
   Vyatta vRouter L3 plugin:

   service_plugins =
     neutron.plugins.brocade.vyatta.vrouter_neutron_plugin.VyattaVRouterPlugin

2. Import the Brocade Vyatta vRouter image using the below glance command:

   glance image-create --name "Vyatta vRouter" --is-public true
   --disk-format qcow2 --file ./vyatta_l3_plugin/image/vyatta_vrouter.qcow2
   --container-format bare

3. Note the provider management network name. This needs to be specified in
the plugin configuration.

4. Configure the L3 plugin configuration file
   /etc/neutron/plugins/brocade/vyatta/vrouter.ini with the below parameters:

	# Tenant admin name
	tenant_admin_name = admin

	# Tenant admin password
	tenant_admin_password = devstack

	# Admin or service VM Tenant-id
	tenant_id = <UUID of the admin or service VM tenant>

	# Keystone URL. Example: http://<Controller node>:5000/v2.0/
	keystone_url = http://10.18.160.5:5000/v2.0

	# Vyatta vRouter Image id. Image should be imported using Glance
	image_id = <UUID>

	# vRouter VM Flavor-id (Small)
	flavor = 2

	# vRouter Management network name
	management_network = management

Once configured, L3 plugin will be invoked for the CRUD operations on
tenant router, add/remove router interfaces and floating ip support.

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  natarajk

Other contributors:
  None

Work Items
----------

Brocade Vyatta vRouter L3 plugin source code files:

vrouter_neutron_plugin.py - Implements L3 API and calls the vRouter driver.
vrouter_driver.py - Uses Nova API for vRouter VM provisioning and
vRouter REST API for configuration.

Code is available for review:
https://review.openstack.org/#/c/102336/

Dependencies
============

None

Testing
=======

- Complete Unit testing coverage of the code will be included.
- For tempest test coverage, 3rd party testing will be provided (Brocade CI).
- Brocade CI will report on all changes affecting this plugin.
- Testing is done using devstack and Vyatta vRouter.

Documentation Impact
====================

Will require new documentation in Brocade sections.


References
==========

None
