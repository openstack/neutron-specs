..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Internal DNS Resolution
=======================

:Author: Carl Baldwin <carl.baldwin@hp.com>

https://blueprints.launchpad.net/neutron/+spec/internal-dns-resolution

Users of an OpenStack cloud would like to look up their instances by name in an
intuitive way using the Domain Name System (DNS).  They boot an instance using
Nova and they give that instance an "Instance Name" as it is called in the
Horizon interface.  That name is used as the hostname from the perspective of
the operating system running in the instance.  It is reasonable to expect some
integration of this name with DNS.

Problem Description
===================

Neutron already enables DNS lookup for instances using an internal dnsmasq
instance.  It generates a generic hostname based on the private IP address
assigned to the system.  For example, if the instance is booted with
*10.224.36.4* then the hostname generated is *host-10-224-36-4.openstacklocal.*

The first problem with this implementation is that the name used in DNS does
not match what the instance knows as its host name.  This creates problems with
sudo and other software that expect to be able to look up its own hostname
using DNS [#]_.

.. [#] https://bugs.launchpad.net/nova/+bug/1175211

The second problem with this implementation is that the end user does not know
this name without some special knowledge.  The generated name is not presented
anywhere in the API and therefore cannot be presented in any UI either.

Proposed Change
===============

Overview
--------

This blueprint will reconcile the DNS name between Nova and Neutron.  Neutron
DHCP offers will use Nova's instance *hostname* value as its hostname instead
of the Neutron generated hostname.  Neutron DNS will reply to queries for the
new hostname.  See the corresponding Nova blueprint for more details about the
hostname that will be passed [#]_.

.. [#] https://blueprints.launchpad.net/nova/+spec/internal-dns-resolution

The Neutron Port API will be extended to add a *dns_name* field to the port.
This field will be used by Nova to pass an intance's *hostname* during port
creation or update. The valid values for this field are:

- A partially qualified domain name (PQDN).

- A fully qualified domain name (FQDN).

To handle existing installations, Neutron will fall back completely to the
current behavior in the event that a *dns_name* is not supplied with the port.

Also, as another level of backwards compatibility, the DNS names formerly
created by Neutron will continue to be available for forward DNS lookups in
dnsmasq.

The Neutron API will add the *dns_name* field when showing port detail so that
users can see it.

The following table shows how DNS will respond to various queries.  The table
uses *example.com.* rather than *openstacklocal.* to reflect that this will
only be enabled when a domain name is associated.

========= ============================= ======== =============================
Rec. Type Query                         dns_name Response
========= ============================= ======== =============================
A         name.example.com.             name     10.224.36.4
A         host-10-224-36-4.example.com. (n/a)    10.224.36.4
AAAA      name.example.com.             name     fd5d:19f:52e9::2
PTR       4.36.224.10.in-addr.arpa      (none)   host-10-224-36-4.example.com.
PTR       4.36.224.10.in-addr.arpa      name     name.example.com.
PTR       fd5d:19f:52e9::2.ip6.arpa     name     name.example.com.
========= ============================= ======== =============================

When Neutron cannot accept a name from Nova because it does not validate,
Neutron will fail the port creation or update. See the  `Data model impact`_
section for more details about this association.

A second read-only attribute, *dns_fqdn*, will be added to Neutron ports. The
API will add this *dns_fqdn* attribute when showing the port details. This wiil
enable users to see the fully qualifed name generated by Neutron based on the
*dns_name* passed during port creation or update. The following table shows how
FQDN's will be generated from *dns_name* with *example.com.* as the configured
domain name:

======================= =====================================
dns_name                dns_fqdn
======================= =====================================
vm01                    vm01.example.com.
vm01.test1              vm01.test1.example.com.
vm01.test1.example.com. vm01.test1.example.com.
vm01.test1.other.com.   API will fail port creation or update
Not specified by user   Null
======================= =====================================

Based on above table, the rules for *dns_fqdn* generation can be summarized as
follows:

- If *dns_name* is a FQDN, validate that the higher level labels match the
  configured domain name.

- If *dns_name* is a PQDN, append the configured domain name to form a FQDN.

- If *dns_name* is not specified by the user, *dns_fqdn* will be null.

Is Not
~~~~~~

This change does not propose a model that would allow driver implementations to
implement internal DNS using something other than dnsmasq.  That could be done
in a follow-on blueprint to this one.  Some things to consider for the future
are:

#. The port that DNS listens on currently is the same as DHCP.  This is how
   dnsmasq does it but it may not be natural with other resolvers.  There are
   problems inherent with sharing this port.  For example, there is little to
   no need for the DHCP ports to have predictable IP addresses.  However, it is
   essential for DNS to have a predictable IP address.  Problems have resulted
   from this [#]_.  It may be worth considering reserving IP addresses on the
   local network for use by the DNS resolver.
#. Coupling the DNS resolver with the gateway router and using its IP address
   as the DNS IP address will not work on an isolated network where there is no
   gateway.
#. For internal DNS, the resolver should provide both authoritative responses
   for the local domain and either forward other queries to an upstream
   resolver or provide recursive resolution.

.. [#] https://bugs.launchpad.net/neutron/+bug/1288923

Data Model Impact
-----------------

This blueprint will add a dns_name field to the Port data model. Values will
be validated as legal partially qualified domain names (PQDN) or fully
qualified domain names (FQDN). Within these names, DNS labels will be validated
according to RFC 1035, which describes the valid format as:

    They must start with a letter, end with a letter or digit, and have as
    interior characters only letters, digits, and hyphen.  There are also some
    restrictions on the length.  Labels must be 63 characters or less. [#]_

.. [#] http://tools.ietf.org/html/rfc1035

.. TODO These rules have been expanded to allow unicode letters.  See RFCs
   5890, 5891, 5892, and 5893.  I will implement the stricter rules first and
   later follow on to allow relaxed rules to keep the scope of this blueprint
   under control.  The follow on should be fairly low hanging fruit.

A complication is that DNS labels are treated as case insensitive [#]_.
Neutron will convert upper case characters to lower case before recording the
name.

.. [#] http://tools.ietf.org/html/rfc4343

Validation will only be performed on the name when the user has enabled DNS for
the port's network by associating a domain with the network.

A second read-only attribute, *dns_fqdn*, will be added to the Port data model.
The API will add this *dns_fqdn* attribute when showing the port details and
will store the FQDN generated by Neutron, as described in the 'Overview_'
section.

An appropriate database migration script will be provided along with the
implementation to add these fields to the ports table on upgrade.

The default value for these fields will be null in order to maintain
compatibility with existing installations.  A null value corresponds to the
field being unspecified in the API.


REST API Impact
---------------

The addition of the dns_name and dns_fqdn fields to the port object in the data
model will need correspending changes to the Ports API . These changes will be
done in a way that maintains compatibility with the existing API.

Users will be able to specify the dns_name field for POST and PUT operations.
Leaving it unspecified in the API call will result in a null value in the
model.  Any non-null value passed in this field will be validated at the API
level as a PQDN or FQDN.  The format is described in the `Data model impact`_
section.

dns_fqdn is a read-only attribute and will be included by the API when
returning a port details as a result of POST, PUT or GET operations. This
attribute will have a null value when the user has not specified a value to
dns_name. The 'Overview_' section describes how Neutron generates values for
this attributes.

No policy changes are needed.


Security Impact
---------------

This change will enable collection of user data in the dns_name field.  This
data will be validated as a legal PQDN or FQDN.  This user data will then be
used to enable DNS lookup using the name internally to Neutron.  I don't expect
this to present any new security risks due to the design of this feature.  As
always, care should be taken in the code review to ensure that no risks creep
in inadvertently.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

The python-neutronclient may need modification to handle the new fields.

Performance Impact
------------------

Database operations for creating and updating a port may be affected marginally
due to the addition of a new constraint on the dns_name field. No new database
locks are needed so the effect is expected to be minimal.

IPv6 Impact
-----------

None


Other Deployer Impact
---------------------

This change was carefully designed to allow new Nova and Neutron code to be
deployed independently. The new feature will be available when both upgrades
are complete.

If Neutron is upgraded before Nova, there is no problem because the
dns_name field is not required and behavior defaults to old behavior.

If Nova is upgraded before Neutron then Nova will see errors from the Neutron
API when it tries passing the dns_name field.  Nova will recognize this error
and retry the operation without the dns_name.

DNS names will only be passed for new instances after this feature is
enabled. Existing instances will use the old Neutron generated generic
names.

No new configuration options are introduced with this change.

Developer Impact
----------------

None

Community Impact
----------------

A number of folks in the community have complained about the lack of agreement
between the compute instance hostname and the DNS name in Neutron.  In addition
to the bug mentioned several times in this blueprint, there have been IRC
discussions [#]_ and face-to-face discussions at summit about this problem.  It
is not the biggest problem in Neutron but has been around a while and annoys a
number of people.

.. [#] http://eavesdrop.openstack.org/irclogs/%23openstack-neutron/%23openstack-neutron.2014-02-21.log (2014-02-21T00:19:20)

Alternatives
------------

An alternative is to use the existing port *name* field in the API to allow
Nova to pass the name to Neutron instead of adding a new field.  The reasons
for not doing this are:

#. That field may be used by Neutron API consumers for other purposes.
#. The field was not intended to carry a PQDN or FQDN and therefore does not
   validate values as such.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  `miguel-lavalle <https://launchpad.net/~minsel>`_

Other contributors:
  `zack-feldstein <https://launchpad.net/~zack-feldstein>`_

Work Items
----------

#. Database upgrade script.
#. API extension with validator.
#. Change API to return dns_name or default generated name and dns_fqdn on GET.

  - Moves default hostname generation to API server.

#. Change to RPC to include dns names in the port section of network data
   requested by DHCP agent.
#. Change to DHCP agent to write dns names to hosts file when updating dnsmasq.

Dependencies
============

In order for this to work end to end, we need coordinated change in Nova.

https://blueprints.launchpad.net/nova/+spec/internal-dns-resolution

Testing
=======

Tempest Tests
-------------

Tempest tests should be added or modified for the following use cases

- No name is specified in port create. DNS lookup should work using old method.
- A port create that duplicates a name on a network should fail.
- An instance created using the nova API can be looked up using the instance
  name.

I believe that the validation of input can be tested using unit tests.

Functional Tests
----------------

None

API Tests
---------

None

Documentation Impact
====================

User Documentation
------------------

None

Developer Documentation
-----------------------

`REST api impact`_ should be documented.

References
==========

https://bugs.launchpad.net/nova/+bug/1175211
