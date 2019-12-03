..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

Allow user to create default record on port creation from shared network
========================================================================

https://bugs.launchpad.net/neutron/+bug/1843218

As discussed in the bug thread above, we could add the feature to allow a user
to have default zone configured on a shared network.

Problem Description
-------------------

On Neutron, when the DNS ML2 plugin is enabled, each user can manage their own network, and
configure a dns_domain (i.e. example.com) for that network. The dns_domain (called zone) can
be hosted in one and only one tenant. To create a record in a zone, the record's tenant must be
the same as the zone's tenant. If a user creates a port in that network, it has to be the same
tenant as zone to get the record created.

Network (A project) <~~> Port (B project) <==> Zone (B project) <==> Record (B project)

So we need to have all these resources hosted on the same project (port, zone and record) to
get this feature working.

As ports are able to be created from several different projects in the case of a shared
network, this feature doesn't work for that case.


Proposed Change
---------------

As described in the RFE, the idea is to let the network's admin user configure keywords in
the default dns_domain on their network to allow zone to be different per user or per project.

Here are the accepted keywords :

- <project_id>
- <project_name>
- <user_name>
- <user_id>

For instance, configuring <user_name>.<project_id>.example.com. as dns_domain on any
network will allow users to have one default zone per user and per project, and then be able
to create records.

::

  $ source openrc_admin
  $ openstack network set --dns-domain "<user_name>.<project_id>.example.com." shared
  $ source openrc_demo
  $ openstack zone create --email dnsmaster@defaultzone.com UserName.HisProjectId.example.com.
  $ openstack port create --dns-name myport port

Further changes included:

* Documentation: api-ref.
* neutron-lib adaptation
* neutron dns_integration ML2 driver adaptation
* dns_integration unit tests
* dns_integration integration tests

Planned Impact
~~~~~~~~~~~~~~

- No impact expected on upgrades.
- No impact expected on configuration.
- No breaking changes

References
----------

* RFE bug report of this spec: https://bugs.launchpad.net/neutron/+bug/1843218
* Neutron Drivers Meeting discussion about the feature validation : http://eavesdrop.openstack.org/meetings/neutron_drivers/2019/neutron_drivers.2019-10-04-14.00.log.html#l-171
