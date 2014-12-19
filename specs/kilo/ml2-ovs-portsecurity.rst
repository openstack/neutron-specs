..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================================================
Add Port security extension support for ML2 plugin and IptablesFirewallDriver
=============================================================================

https://blueprints.launchpad.net/neutron/+spec/ml2-ovs-portsecurity

This spec proposes to add support portsecurity extension to ML2 plugin and
IptablesFirewallDriver to match it.


Problem Description
===================

Neutron's security group always applies anti-spoof rules on the VMs.
This allows traffic to originate and terminate at the VM as expected,
but prevents traffic to pass through the VM. This is required in cases
where the VM routes traffic through it.
In order to run network services in VM instances (e.g. router service
in VM [router_plugin_cisco]_, [vyatta_l3_plugin]_ or firewall service in VM),
it is required by some services for VMs to be able to
receive/send all packets without any kind of firewall, security group,
anti spoofing on port. This is a basic requirement to run
network service within VMs. The necessity depends on the type of
services. Some services require it, some don't.

At this point Neutron has a port security extension to disable packet
filtering. ([port_security_extension]_, [port_security_extension_db]_)
But currently the portsecurity extension isn't supported by any open source
plugins/firewall driver.
This blueprint is to add the extension support to the ML2 plugin when
configured with OVS agents using the IptablesFirewallDriver.


Proposed Change
===============

How portsecurity extension works
--------------------------------

The original documentation for NSX plugin can be found at
[port_security_base_class]_ and [quantum_port_security]_.
The extension adds a new attribute, "port_security_enabled", to
network and port resources. The port_security_enabled of network is used as
the default attribute value at port creation.
When the attribute is set to True(by default), the behavior remains same to
the one without portsecurity extension, security group and anti spoofing will
act as before. When the attribute is set to False, security group and anti
spoofing are disabled on the port, and it is not allowed to set security group
or allowedaddresspair with such ports.  Since this feature is related to
security, only tenant owner is allowed to set/change the attribute.

Some clarifications

* The attribute of network affects only at port creation. The already created
  ports aren't affected when the value of network is changed.
* If the port is already associated with security group, it results in
  an error to try to change port_security_enabled to False.
* When port_security_enabled = False, it results in an error to set
  security group or allowedaddresspair

Steps
-----

Add port-security extension as ML2 extension driver.
And then add necessary feature in IptablesFirewallDriver.
Port security extension is already added in neutron. Implementation of this
extension in the ml2 plugin will allow enabling/disabling filters on the
neutron port as required when using the ml2 plugin.

The sketch of implementation of ovs agent enhancement:
OVS bridge and its flow rules are used to route packet, push/pop vlan tag,
and tunneling. So openflow rules doesn't need to be touched.
The security group/anti-spoofing are realized by iptables with linux
bridge whose name is qbrxxx. The actual firewall driver is
neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

* $e<device name> chain in filter table is used for security group filtering
  of egress packet
* $i<device name> chain in filter table is used for security group filtering
  of ingress packet
* $s<device name> chain in filter table is used for anti-spoofing.
  This is only partially works. anti ARP spoofing isn't implemented. For that,
  ebtables is necessary. [bug1274034]_
  NOTE: ARP spoofing will not implemented in the scope of this BP.


So those chains will be modified to ACCEPT.
Or the parent chain of those chains is $sg-chain which demuxes packets
to the above three chains. Another option is to change the rule in $sg-chain.


related bridges(take ovs plugging for example)::

   +----+
   | VM |
   +----+
     |
   +--------+   Linux bridge:
   | qbrxxx |   firewall are realized here as iptables chains/rules, which
   +--------+    will be modified in the implement of this spec
      |
   +--------+   ovs bridge
   | br-int |
   +--------+
      |
   +------------------+  ovs bridge
   | br-tun/br-eth<N> |
   +------------------+


Data Model Impact
-----------------

portsecuritybindings and networksecuritybindings tables will be used by ml2
plugin.
The tables already exist and will be use without modification, no new tables
will be added.
They could be refered to in neutron/db/portsecurity_db.py


REST API Impact
---------------

Port security extension is cited here for convenience.
This blueprints doesn't add this API, enables it.

Default value of port_security_enable for a network is True, and same
attribute setting for port created from the network will inherit this value.
Then it means the same behaviour without this extension. The behavior For the
existing port (with or without security group) remain same as before.
When port_security_enabled = False, security group and anti-spoofing are
disabled on the port. It results in AddressPairAndPortSecurityRequired
exception to try to set to allowed address pairs attribute.

The code could be refered from neutron/extensions/portsecurity.py


Security Impact
---------------

This feature is dangerous to the unwary. They should only be available
to network owners to avoid compromise of other people's networks.


Notifications Impact
--------------------

port_security_enabled attribute of network and port will be added to
related notification.


Other End User Impact
---------------------

None, because the port security attribute defaults to True, and therefore
existing ports will be unaffected.


Performance Impact
------------------

OVS agent could be heavier because its port management task will be
enhanced.


IPv6 Impact
-----------

None


Other Deployer Impact
---------------------

None

Developer Impact
----------------

Since ML2 plugin will be changed to support port-security extension as
first class citizen. And we will use the current framework to notify l2 agent
when the attribute is changed.
Iptables firewall driver would be updated to add/update chains to make
packages passed through.


Community Impact
----------------

Portsecurity extension is desired by various parties, servicevm from various
vendors who promotes virtual appliance.


Alternatives
------------

Documentation can be done such that disable security (or to use other
extension like allowed address pairs) to use the service YYY.
However some features can't be disabled/enabled per port-wise and can't be
done automatically as Neutron ports are created/deleted.
It lacks flexibility. So simple documentation doesn't work.

Another alternative is to introduce keyed knobs to control security
group and anti spoofing each instead of single knob, port_security_enabled.
The value will be dict of

Example of value

.. code-block:: python

  {'port_security_group_enabled': True,
   'port_anti_spoofing_enabled': True},
  # dict value is adopted to allow more precise control for filtering in future
  # e.g. 'port_filter_level': 1
  {'all_filtering_enabled': False},
  # all_filtering_enabled is special key to enable/disable all
  # related filtering.


.. code-block:: python

  PORTSECURITY = 'port_security_enabled'
  # or port_security_disabled to list only disabled filters
  EXTENDED_ATTRIBUTES_2_0 = {
      'networks': {
          PORTSECURITY: {'allow_post': True, 'allow_put': True,
                         'validate': {}
                         'convert_to': attributes.convert_kvp_to_dict,
                         'enforce_policy': True,
                         'default': ATTR_NOT_SPECIFIED,
                         'is_visible': True},
      },
      'ports': {
          PORTSECURITY: {'allow_post': True, 'allow_put': True,
                         'convert_to': attributes.convert_kvp_to_dict,
                         'default': attributes.ATTR_NOT_SPECIFIED,
                         'enforce_policy': True,
                         'is_visible': True},
      }
  }


Another approach is to add a new attribute in port binding extension, like one
in "binding:profile".
But with port binding extension, only admin is allowed to set the attribute
and a way to specify default value for a network is necessary.


Implementation
==============

Implement the port security extension in ml2 plugin
This introduces the ability to enable/disable the port_security_state
in the ml2 plugin while creating/updating a port or a network.
The create network function will assign a default port_security_state
to all ports to be created in that network.


Assignee(s)
-----------

Primary assignee:
  * yalei-wang
  * Shweta P <shweta-ap05>
  * ijw-ubuntu (Ian Wells)

Other contributors:
  * yamahata (Isaku Yamahata)
  * rui-zang


Work Items
----------
(name) means task assignment.

* implement portsecurity extension as ML2 extension driver
  (Shweta and Isaku)
  ** convert dependent extension into extension driver if necessary
  (Shweta and Isaku)
* OVS agent and iptables firewall driver modification
  (Yalei)

* tests
  (everyone)

Dependencies
============

None


Testing
=======

tempest will be enhanced to check if security group isn't applied.
i.e. API tests for port-security extension and scenario tests for functional
tests.
* creation/deletion of ports with or without port_security_enabled=True/False
* try to send/receive packets that is filtered by port filtering to other ports
* check if the packets can be received/sent with other port

Tempest Tests
-------------

Related scenario test will be added.


Functional Tests
----------------

Necessary test will be added.


API Tests
---------

port_security_extension unit test has been added in repo.


Documentation Impact
====================

User Documentation
------------------

API and Admin guide will be updated so that it includes
* configuration to enable portsecurity extension for ML2 OVS driver
* new attributes and new CLI interfaces


Developer Documentation
-----------------------

None


References
==========

.. [port_security_extension] port security extension
   http://git.openstack.org/cgit/openstack/neutron/tree/neutron/extensions/portsecurity.py
   https://docs.google.com/document/d/18trYtq3wb0eJK2CapktN415FRIVasr7UkTpWn9mLq5M/edit

.. [port_security_extension_db] port security extension db part
   http://git.openstack.org/cgit/openstack/neutron/tree/neutron/db/portsecurity_db.py

.. [ml2_extension_driver] Support for extensions in ML2 Mechanism Drivers
   * spec review: https://review.openstack.org/#/c/89208/
   * etherpad: https://etherpad.openstack.org/p/ML2_MD_extensions

.. [modular_l2_agent] Modular L2 agent
   spec review: https://review.openstack.org/#/c/99187/

.. [router_plugin_cisco]
   Describes design of router service plugin for Cisco devices
   https://review.openstack.org/#/c/91071/

.. [vyatta_l3_plugin] Design Spec For Brocade Vyatta L3 Plugin
   https://review.openstack.org/#/c/101052/

.. [dvr] Neturon Distributed Virtual Router for OVS
   https://blueprints.launchpad.net/neutron/+spec/neutron-ovs-dvr

.. [ovs_firewall_driver]
   Open vSwitch-based Security Groups: Open vSwitch Implementation of
   FirewallDriver
   https://blueprints.launchpad.net/neutron/+spec/ovs-firewall-driver
   https://review.openstack.org/#/c/89712/

.. [port_security_base_class] Port Security API base class
   https://blueprints.launchpad.net/neutron/+spec/port-security-api-base-class

.. [quantum_port_security] Quantum Port Security
   https://docs.google.com/document/d/18trYtq3wb0eJK2CapktN415FRIVasr7UkTpWn9mLq5M/edit?pli=1

.. [ml2_extensions] Support for extensions in ML2 Mechanism Drivers
   https://blueprints.launchpad.net/neutron/+spec/neutron-ml2-mechanismdriver-extensions

.. [port_security_ml2] Add Port Security Implementation in ML2 Plugin
   duplicated proposal. consolidated to this one.
   * blueprint https://blueprints.launchpad.net/neutron/+spec/port-security-ml2
   * spec review https://review.openstack.org/#/c/106222/

.. [nfv_unaddressed_interfaces]
   NFV unaddressed interfaces
   * blueprint https://blueprints.launchpad.net/neutron/+spec/nfv-unaddressed-interfaces
   * spec review https://review.openstack.org/#/c/97715/

.. [port_security_ml2_patch] Add portsecurity extension support
   patch for ovs firewall driver
   https://review.openstack.org/#/c/126552/

.. [related_bugs]
   related bugs: creating network without subnet and port without subnet
   * https://bugs.launchpad.net/bugs/1039665
   * https://bugs.launchpad.net/bugs/1175464

.. [bug1274034]
   Neutron firewall anti-spoofing does not prevent ARP poisoning
   https://bugs.launchpad.net/neutron/+bug/1274034
