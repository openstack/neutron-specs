..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
Allow compute port mac_address to be updated
============================================

https://blueprints.launchpad.net/neutron/+spec/allow-mac-to-be-updated

Ironic servers can experience a nic failure which requires nic replacement.
The replacement nic can have a new mac address.  There is currently no way to
handle this situation without the potential of losing the IP address.

See also bug [2].


Problem Description
===================

Because the 'mac_address' attribute of port is not updatable, replacing a nic
on an ironic server today requires deleting the associated neutron port and
creating a new one with the correct mac address.  This can result in the server
losing its IP address to another port being created at the same time.

This spec does not seek to retain the same IPv6 SLAAC address when mac address
changes.  Such addresses are expected to change.


Proposed Change
===============

Change server code to allow the mac_address attribute of the port resource to
be updated.  Specifically, enable this feature for the ml2 plugin.  Any
work to be done to other plugins will be addressed in other specs.

In the base plugin, restrict mac_address update to compute ports only
(non-compute ports require more changes and are not covered by the use cases
handled here).  The use cases don't require an active port, so in the ml2
plugin, we require that the port not be 'plugged': vif_type must be
BINDING_FAILED or UNBOUND.  Note that for ironic servers, vif_type is always
BINDING_FAILED, so mac_address can always be changed for those ports.  This may
at some point be viewed as a bug however, so any fix to that bug will need to
account for this feature.  For 'normal' VMs, the life cycle of binding:vif_type
depends on the backend.  The l2pop mech driver is updated to change the fdb
table entries appropriately.

In the OVS L2 agent, send a gratuitous arp when a mac_address change is
detected.

The allowed_address_pairs extension allows for the mac_address of any pair to
default to port.mac_address.  This implies that there are cases where updating
the port's mac_address should also update the allowed_address_pair mac_address.


Data Model Impact
-----------------

None.


REST API Impact
---------------

Modify the 'allow_put' property of the 'mac_address' attribute in the 'ports'
entry of the RESOURCE_ATTRIBUTE_MAP from False to True.  The policy.json file
is modified to add rules for "update_port:mac_address" - the same rules are
used as for "create_port:mac_address" except network owner is not allowed.


Security Impact
---------------

To be conservative for now, this feature will be restricted by default to the
following.

* admin role

* advsvc role

The user can adjust policy.json as always, if desired.


Notifications Impact
--------------------

Introduces a new case where notifications are issued, but does not add a new
notification or modify or remove any existing notification.


Other End User Impact
---------------------

None.


Performance Impact
------------------

The performance impact when mac_address is changed during a port update are
characterized in the following list.

* The impact should be the same as that of changing the ip address.

* For dvr environments, adds the overhead of VM arp table updates for all
  routers on the network.


IPv6 Impact
-----------

No IPv6-specific issues are expected.  When a mac address changes, the SLAAC
address, if any, will change also, but since such addresses are dynamic, that
should be fine.  Domain name servers will need to start vending the new
address, and the current code should handle the AAAA record update to OpenStack
DHCP servers.  If the subnet's DHCP is not enabled, the propagation of the new
address to external DNS servers is out of scope.  The link local address also
changes, but that is also fine.


Other Deployer Impact
---------------------

Deployers can now replace nics in ironic servers.


Developer Impact
----------------

Non-ml2 plugin developers need to decide whether mac_address updates can be
allowed or are desirable in their environment.  The normal development process
should take place to deal with the fact that the neutron api and the base
plugin now allow mac address update.


Community Impact
----------------

Integrates well with ironic: in addition to nic replacement, some nics have
changeable mac addresses, so the ability to change the mac_address of a port
seems useful.

There hasn't been much discussion yet, aside from review comments against the
implementation.


Alternatives
------------

1. If fixed_ips were a separate resource (similar to floating_ips, and vaguely
referred to in [4]), then the tenant could retain ownership of ip address while
the port was deleted and re-created with the new mac address.  This would
require a large change to the data model, which may not be justified by the
requirements of this spec.  This also puts the burden of relating ip address
and mac address on the client, along with the various failure scenarios that
would have to be handled without the benefit of transactions.  Seems like a
bigger win to do the proposed relatively small change in the server and remove
some complexity from clients.

2. Another thought was to use DHCP client id instead of mac address in the dhcp
protocol.  Then the mac address becomes irrelevant to the ip address lookup,
and replacing the card would have no affect on the IP address assigned.
However, based on a quick discussion with the Ironic team [3], there is a hard
need to have the port mac_address match the hardware's mac address.

Implementation
==============


Assignee(s)
-----------

Primary assignee:
  ChuckC  (Chuck Carlino)


Work Items
----------

Work items related to neutron core.

1. Review [1]

2. Baremetal scenario testing to cover nic replacement

3. allowed_address_pairs support

4. API tests

Vendor plugin work items are out of scope.


Dependencies
============

None.


Testing
=======

Tempest Tests
-------------

Add tempest test cases to cover mad address update.

* VM mac address update verifying connectivity

* baremetal nic replacement


Functional Tests
----------------

Add tests updating a port's mac_address and validate that the following
are appropriately updated/notified.

* DHCP agents

* security group rules

* allowed_address_pairs


API Tests
---------

Since this feature involves an api change, we plan to add tests to validate the
update of port's mac_address.  If, at the time of implementation, neutron API
tests include negative test cases and tests of policy.json, then those will be
added for this feature as well.


Documentation Impact
====================

Need to document that it is now permitted to update the mac_address of
an unbound or failed VM port.

User Documentation
------------------

The following documents and sections may need updating.

* Cloud Administrator Guide

  - Core Networking API features

* Neutron/APIv2-specification (Port table)

Developer Documentation
-----------------------

None.

References
==========

[1] https://review.openstack.org/112129

[2] https://bugs.launchpad.net/neutron/+bug/1341268

[3] http://lists.openstack.org/pipermail/openstack-dev/2014-November/050329.html

[4] https://blueprints.launchpad.net/neutron/+spec/neutron-ipam
