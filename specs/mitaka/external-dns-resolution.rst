..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
External DNS Resolution
=======================

https://blueprints.launchpad.net/neutron/+spec/external-dns-resolution

:Author: Carl Baldwin <carl.baldwin@hp.com>
:Copyright: 2015 Hewlett-Packard Development Company, L.P.

This blueprint builds from where the Internal DNS Resolution blueprint leaves
off [#]_.  That one reconciles the Nova VM name with the hostname assigned by
Neutron DHCP and with a DNS name that can be used to look up the host using
Neutron's internal DNS.  The next step is to integrate DNSaaS so that instances
can be found using their instance names plus a DNS domain name from outside the
cloud.

.. [#] https://blueprints.launchpad.net/neutron/+spec/internal-dns-resolution

Problem Description
===================

As instances come and go and floating IPs are associated and disassociated,
changes may be required in an external DNS service to reflect them.  Currently,
these changes need to be made independently.  This is often a manual process
which is prone to error.  It can be scripted using an API to the external DNS
service but that adds complication for API users and their scripts.

Proposed Change
===============

Overview
--------

This change will link public IP addresses on Neutron external networks with an
external DNS service such as OpenStack Designate.  Tenants will expect that any
create, update, or delete of an IP address on a network will result in a
set of corresponding changes in DNS, covering both A/AAAA and PTR records.

A very important question to consider at this point is whether the A/AAAA
record(s) pointed to by the DNS name are associated with an instance or
with the public IP itself.  This question will be discussed in detail in
the `Data model impact`_ section.

The change will introduce an interface and a reference implementation of that
interface which will integrate with the OpenStack Designate service.  This
reference implementation will be as easy to configure and use as possible.

This driver will be administratively configured with credentials for the
external service in a way similar to how Nova is configured to access the
Neutron service for port creation when an instance gets created.

Split Horizon DNS
~~~~~~~~~~~~~~~~~

Consider an IPv4 situation where NAT is used and a DNS domain has been
associated with a network.  One might expect DNS to respond to queries on that
domain with the private IP address for VMs that happen to reside on the same
network.  Similarly, a reverse DNS request for the private IP address would
return the VMs DNS name.

The advantage of split horizon DNS is that any machine can use the same names
for other machines regardless of whether the machine is on the same network or
a different network.

This is the kind of thing that has been accomplished using BIND 9 dns server
views.  It is typical to disable recursive DNS on the external zone but not on
the internal zone.

This blueprint will accomplish this by leveraging the existing dnsmasq
implementation for internal resolution and integrate with an external dns
service for the external part.

Currently, the domain name is set by Neutron to openstacklocal.  To enable
split horizon DNS, dnsmasq for a network will be configured with the domain
name associated with that network instead of openstacklocal.  If no domain is
associated then it will default to openstacklocal.

There are some limitations to split horizon DNS.

#. If two networks are connected privately by a neutron router then split
   horizon DNS will not work between them.  It will only work for names of
   instances on the same network.
#. It will probably not work for names that come from the the floating IP, as
   explained in the `Data model impact`_

Data Model Impact
-----------------

::

    +–––––––––––––+    +–––––––––––––+    +–––––––––––––+    +–––––––––––––+
    | FloatingIP  |    | Port        |    | Network     |    | DNSDomain   |
    +–––––––––––––+    +–––––––––––––+    +–––––––––––––+    +–––––––––––––+
    | ...         |––> | ...         |––> |...          | -->|             |
    |             |    |             |    |             |    |             |
    | dns_domain  |    |             |    |             |    | dns_domain  |
    | dns_name    |    | dns_name    |    |             |    |             |
    +–––––––––––––+    +–––––––––––––+    +–––––––––––––+    +–––––––––––––+
         \                                                          |
          \                                                         V
           \                                           +––––––––––––––––––––––+
            \                                          | External DNS service |
             \                                         +––––––––––––––––––––––+
              +––––––––––––––––––––––––––––––––––––––> | dns_domain           |
                                                       | dns_name             |
                                                       | ...                  |
                                                       |                      |
                                                       +––––––––––––––––––––––+

The diagram above summarizes the changes that will be made to the data model
and how these changes will support the interactions with the external DNS
service. Such interactions are triggered when a public IP is created, updated
or deleted. Three cases will be supported:

#. Floating IP with null dns_domain and dns_name::

    In this case, DNS names are linked to instances.  More precisely, they are
    linked through the port attached to the instance.  When a floating IP
    association changes, the name and domain used for DNS will come from the
    backing port's dns_name and the DNSDomain associated to the port's
    network.

    The dns_name will match either the Nova display name or the Neutron
    generated name consistent with the first part of this blueprint.

#. Floating IP with no null dns_domain and dns_name::

    In this case, the dns_name and the dns_domain are linked to the floating
    IP. When the floating IP association changes there is no need to change
    anything in DNS.  The instance name has no relevance.

    This covers the use case where a floating IP should always have a given
    name and domain regardless of which instance it is associated with.

    The values for dns_name and dns_domain can be overridden for each floating
    IP.  If these values are set on the floating IP then the DNS name comes
    from the floating IP.

#. Port with no null dns_name on a public provider network with no null
   dns_domain::

    This is the case where the user is making the ports in a provider network
    accesible outside the Openstack cloud. Once more, DNS names are linked to
    instances through their ports attached to the provider network. An
    instance's name and domain used for DNS will come from its backing port
    dns_name and the DNSDomain associated to the port's network.

    The dns_name will match either the Nova display name or the Neutron
    generated name consistent with the first part of this blueprint.

.. NOTE:: Multiple networks can be linked to the same DNS domain.  This raises
   the potential concerns over duplicate names.  Duplicate entries for a single
   DNS name are valid part of DNS, and are considered acceptable by this
   blueprint.

.. NOTE:: Because how A and AAAA records map can depend on whether the
   publishing DNS is running split view, it may make sense to publish the IP
   under both names if they exist.

For PTR records, what will be published is a pointer to the instance dns name
under the private IP address and a pointer to the floating ip dns name under
the floating IP address.

DNSDomain
~~~~~~~~~

A DNSDomain object will be added to the data model, with a one to many
relationship with the network object. This will allow to associate several
networks with a DNS domain name. No relationship with a DNSDomain object means
that the network object is not associated with a DNS domain name. There is no
need to populate existing network objects on upgrade so the upgrade will not
require any special attention.

The dns_domain in a DNSDomain object will be validated as a legal dns domain at
the API level.  The plugin will also validate the domain with the external
service to ensure that it exists and is accessible to the tenant.

In some scenarios, a single DNS domain name may legally exist multiple times in
the external service.  A driver specific qualifier may be optionally appended
to the dns_domain name, and will be separated from the name by a single colon
(:) character.  For example, assuming the domain example.org exists in the
external service with a resource ID of 298c22d1-cebd-475b-b5c9-0f880ee2f23f,
the qualified domain name stored by DNSDomain could be
"example.org:298c22d1-cebd-475b-b5c9-0f880ee2f23f". The use of a qualifier must
be optional, and only required where a project has visibility of more than one
DNS domain with the same name in the external service.

Floating IP
~~~~~~~~~~~

Two fields -- dns_domain and dns_name -- will be added to the floating ip
object.  The dns_domain field will be validated in the same way it is on the
`DNSDomain`_ object.  The dns_name field will be validated as a valid PQDN. See
the internal-dns-resolution blueprint for details.

REST API Impact
---------------

The API will be extended to support CRUD operations on the new object
DNSDomain. The create and update operations will enable users to associate dns
domains with a list of networks.

The current API to manage floating ips will be extended to add optional
dns_domain and dns_name fields.

Security Impact
---------------

The Neutron configuration files will contain driver specific configuration for
accessing the external service, this will include sensitive information such
as usernames/passwords, and will be similar in nature to how Nova is
configured with Neutron credentials for port creation.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

This change will need a corresponding change to python-neutronclient to add
support for the API changes in `REST API impact`_.

Performance Impact
------------------

None

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

This change is something that has to be explicitly enabled by the deployer.  It
is optional, the defaults will work for existing or new deployments.

The deployer will be responsible for configuring the driver to interact with
the external DNS service.  This will be an optional feature and will not
require any action on the part of the deployer until enabling the new feature
is desired.

Developer Impact
----------------

None

Community Impact
----------------

None

Alternatives
------------

This functionality could be handled from Designate.  In general, this has the
disadvantage of excluding other DNS services.

This could be accomplished by polling Neutron for changes to public IP
addresses.  Doing this would require tuning the polling interval so that it is
often enough that users will not complain about the time it takes for changes
to take effect but not so often that it becomes a burden on the performance of
the system.

This could also be done by integrating Designate with Neutron RPC so that it
picks up on changes to public IP address.  This has a disadvantage that it ties
Designate to the implementation of Neutron.  It would be difficult for
designate to understand and account for the relationship between internal ports
and floating IP ports.  It would also be difficult to control the affinity of a
DNS name.

Implementation
==============

Work Items
----------

#. Extend database models.

   * Add dns_name and dns_domain attributes to FloatingIP model.
   * Add DNSDomain object.

#. Database upgrade script.
#. Extend API and validators.

   * Add dns_name and dns_domain to FloatingIP.
   * Implement CRUD operations for DNSDomain

#. Implement interface between Neutron and external DNS service
#. Implement driver for Designate.

Dependencies
============

https://blueprints.launchpad.net/nova/+spec/internal-dns-resolution

This change requires the use of the Designate API which is not currently used
by Neutron.

Testing
=======

Tempest Tests
-------------

Designate has a full DevStack plugin, which can be used as part of integration
testing within the gate.  There is an open question around if this plugin needs
to be included within the devstack tree, or if it can be consumed reliably from
the Designate repo.


Functional Tests
----------------

None

API Tests
---------

None

Documentation Impact
====================

`REST api impact`_ should be documented.

References
==========

- `Discussion on IRC that sparked this blueprint <http://eavesdrop.openstack.org/irclogs/%23openstack-neutron/%23openstack-neutron.2014-02-21.log>`_ (2014-02-21T00:19:20)
- `Openstack Designate Project <https://wiki.openstack.org/wiki/Designate>`_
