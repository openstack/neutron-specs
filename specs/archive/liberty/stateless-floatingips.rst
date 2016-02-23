..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================
Stateless Floating IPs
======================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/stateless-floatingips

:Author: Carl Baldwin <carl.baldwin@hp.com>
:Co-Author: Cedric Brandily <zzelle@gmail.com>

In the current implementation, conntrack is not useful for floating ips.  There
is no reason for it except that there did not seem to be another way.  If we
could implement floating ips in a stateless way that didn't touch conntrack
then this could be both a performance improvement and a security improvement.

Problem Description
===================

In many contexts, NAT means translating from an arbitrary set of internal IPv4
addresses onto a single external and public IP address, and vice versa. It
implies port numbers have to be mapped as well as addresses, so the NAT engine
needs to track and maintain state for all connections to which the desired
address translation applies.

But floating IPs are just one to one mappings of private and floater IPs so
address translation can be done statelessly without port mapping.  However,
the current implementation uses SNAT and DNAT targets in the iptables nat
table. These targets depend on conntrack to identify packets traversing in
both directions which are part of the established connection.  Tracking these
connections takes processor and memory resources.

As the number of connections through a network node increases, the end-user may
experience connectivity issues unless conntrack is tuned up to handle them.
How to tune conntrack is out of the scope of this blueprint.  The point is that
it is needed with the current Neutron and really shouldn't be.  There would not
be any need to tune it if it were not used.


Proposed Change
===============

The iptables raw table was introduced to give a way to skip conntrack for
certain packets.  From the iptables man page [#]_ ::

   This table is used mainly for configuring exemptions from connection
   tracking in combination with the NOTRACK target. It registers at the
   netfilter hooks with higher priority and is thus called before ip_conntrack,
   or any other IP tables.

.. [#] http://ipset.netfilter.org/iptables.man.html

The NOTRACK target will allow bypassing conntrack for packets associated with
the floating IP.  However, it does not accomplish the actual rewriting of
addresses.  The SNAT, DNAT, and MASQUERADE are not useful because they are
meant to work with conntrack and stateful NAT.  We need something to perform
stateless address translation.

tc (traffic control) has the ability to perform stateless nat [#]_ by rewriting
destination ip in ingress packets::

  # to do once
  tc qdisc add dev qg-... ingress handle ffff:
  # to do for each floating-ip,fixed-ip tuple
  tc filter add dev qg-... parent ffff: protocol ip prio 10 u32\
     match ip dst 212.201.100.135/32\
        action nat ingress 212.201.100.135/32 199.181.132.250

and source ip in egress packets::

  # to do once
  tc qdisc add dev qg-... root handle 10: htb
  # to do for each floating-ip,fixed-ip tuple
  tc filter add dev qg-... parent 10: protocol ip prio 10 u32\
     match ip src 199.181.132.25/32\
        action nat egress 199.181.132.250/32 212.201.100.135

The above is a generic example, the floating IPs will be applied with tc
ingress/egress filters instead of iptables SNAT and DNAT rules being used
today.

.. [#] http://linux.programdevelop.com/2898840/

There is one aspect of stateless NAT that is more difficult than stateful NAT.
It is that a rule must be configured for both directions of packet flow.  This
is because there is no such thing as a connection with stateless rules and
therefore you can't think of the connection as inherently having a direction.

It implies some limitations: stateless NAT always rewrites egress packets;
an external machine sending packets to the fixed-ip of a natted vm/port will
receive packets with the floating-ip as source ip.  So external machines must
use floating-ips of natted vms/ports.

There is a shared SNAT feature of neutron routers for ports which do not have
an associated floating ip.  This shared SNAT will still use conntrack (i.e. it
will still use the iptables nat table with SNAT and DNAT targets).  It is
necessary to demultiplex return traffic back to the various ports using it.

A new option will be defined to enable/disable (disable by default) stateless
NAT in order to support deployments requiring vms/ports to be reachable using
their fixed-ips from external networks (typically private/enterprise clouds).

Data Model Impact
-----------------

None

REST API Impact
---------------

None

Security Impact
---------------

There was a security note released recently related to a bug [#]_ in Neutron
where established connections through floating ips would continue to work after
the floating ip was disassociated from a private IP address.  This bug as fixed
in Juno by explicitly calling conntrack to delete the connections.  This adds
complexity to the L3 agent which can be removed when stateless floating IPs
have been fully implemented.

.. [#] https://bugs.launchpad.net/neutron/+bug/1334926
.. [#] https://review.openstack.org/#/c/124375/

Additionally, one can create a DoS attack using tools like nmap to trigger
conntrack to consume a lot of resources.  The only indication you're likely to
see is "table full".  Once that happens it can cause legitimate connections to
not get through as they were unable to create a cache entry.  Without conntrack
involved, this will no longer be a security concern.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

As stated above, there will be no need to tune conntrack to handle a high
volume of connections by either increasing the size of the connection table or
decreasing the timeout to clean out stale entries.  The entries are not needed.
Conntrack will not be invoked to inspect packets through the floating IP
addresses.

This change will result in more packets being matched in the iptables raw
tables.  Since this table's rules are checked before conntrack, every packet
will be checked against the new rules in the PREROUTING and POSTROUTING chains
of the raw and rawpost tables respectively.  I expect that the performance
savings from bypassing conntrack to outweigh this small hit.

IPv6 Impact
-----------

Since IPv6 does not have a floating IP implementation in Neutron, there is no
effect.

Other Deployer Impact
---------------------

None as tc is part of iproute.


Developer Impact
----------------

None

Community Impact
----------------

None

Alternatives
------------

The kernel once had stateless nat built in to the routing rules feature [#]_.
This was removed (or deprecated) long ago and so it is not viable::

   --> ip rule add nat 205.254.211.17 from 192.168.100.17
       Warning: route NAT is deprecated

   --> ip route add nat 205.254.211.17 via 192.168.100.17
       RTNETLINK answers: Invalid argument

.. [#] http://linux-ip.net/html/nat-stateless.html

The Xtables-addons project [#]_ had an implementation for performing stateless
NAT in the iptables raw table::

   -t raw -A PREROUTING -i lan0 -d 212.201.100.135 -j RAWDNAT --to-destination 199.181.132.250
   -t rawpost -A POSTROUTING -o lan0 -s 199.181.132.250 -j RAWSNAT --to-source 212.201.100.135

But RAWSNAT/RAWDNAT were removed in recent xtable-addons (for kernel >= 3.13)
because the feature was unmaintained.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  `cbrandily <https://launchpad.net/~cbrandily>`_

Other assignees:
  `brian-haley <https://launchpad.net/~brian-haley>`_
  `carl-baldwin <https://launchpad.net/~carl-baldwin>`_

Work Items
----------

#. Implement tc low-level driver for nat management [#]_
#. Define nat driver interface
#. Implement stateless nat driver
#. Implement stateful nat driver (transform current implementation)
#. Implement nat driver loader (new config option)
#. Disable conntrack in stateless nat driver
#. Move conntrack cleanup in stateful nat driver
#. Cleanup stateless nat qdisc when using stateful nat driver

.. [#] https://review.openstack.org/177245


Dependencies
============

Depends on tc (iproute package).


Testing
=======

Full unit test coverage will be added or maintained for the parts of code that
need to be touched to implement this new feature.

Existing higher-level will be sufficient since this blueprint does not change
the nature of floating ips from an external observer's point of view.  If we
need to support this new feature as an optional one with the default off then
we may need to assess whether it needs to be enabled in CI tests.  At the
least, a patch to enable the feature will be used to run tests.


Tempest Tests
-------------

Existing

Functional Tests
----------------

Functional tests will be added for tc, stateless and stateful drivers.

API Tests
---------

None


Documentation Impact
====================

None

User Documentation
------------------

Document new option to enable stateless nat.

Changing the config and restarting the l3-agent are a priori enough to migrate
from stateful nat to stateless nat: the l3-agent should cleanup automatically
iptables rules and stateful nat conntrack entries should time out as
connections will be handled in raw table.  Changing the config and restarting
the l3-agent should be enough to migrate from stateless nat to stateful nat:
the stateful nat will cleanup stateless nat rules on startup.  So migration
documentation is a priori not needed.



Developer Documentation
-----------------------

None

References
==========

See inline references throughout the document.
