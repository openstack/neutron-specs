..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================
ARP spoofing filtering via ebtables
===================================

This issue allows ARP cache poisoning attacks to be started from a guest
VM to other VMs in the same network. This is a security issue, especially
on shared networks.

This blueprint suggest that we utilize the ebtables tool to configure
Ethernet-frame-level filtering, which prevents these spoofing packets
to be sent by a VM.


Problem Description
===================

With an ARP cache poisoning attack a device in the local network can
impersonate someone else. Specifically, a device can claim ownership of an IP
address, which it does not actually have.

The attacker sends out a gratuituous ARP response, in which their network
interface (identified by MAC address) claims to be configured with a certain
IP address. However, this might be someone else's IP address. Other devices in
the network update their ARP cache with the spoofed entry and will put the
attacking device's MAC address as the destination address of frames that are
to be sent to the attacked IP address.


Proposed Change
===============

We propose to add an 'ebtables manager', which knows how to utilize the
ebtables utility. This utility can be used to create Ethernet frame level
filtering rules.

Whenever a port for a VM is created or deleted, we use the ebtables manager to
update ebtables rules, which prevent the sending of spoofed ARP packets
through this port.

Please note that the ebtables manager code is largely based on the work of
Édouard Thuleau, who had submitted a patch (see 'references' section below).

This patch, however, did not make it into the release and was abandoned. We
are now resurrecting much of this patch.

An updated version of this patch was submitted for Juno, but was rejected for
various reasons.

The existing patch replaced some of the iptables-based filtering with new
ebtables filters. Therefore, it changed quite a bit of the existing code in
iptables_firewall.py. This was not appreciated, since this code was seen as
brittle and in need of a re-write.

Therefore, we are now proposing that the changes to iptables code are kept to
a minimum. We merely need to add a few small hooks in the iptables code, so
that the ebtables manager code is invoked when a port is created or deleted.
The ARP spoofing related rules are the same as in the previous patch, however
none of the unrelated rules introduced by the previous patch are contained in
this patch.

If at some later time the iptables code is going to be re-written, any hooks
to the ebtables code can easily be ported over.

Data Model Impact
-----------------

None.

REST API Impact
---------------

None.

Security Impact
---------------

The ebtables manager code is run with the same privileges as the iptables code
(root-wrap).

It improves security of VMs on shared networks.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None.

Performance Impact
------------------

The new code will be called whenever a VM port is created or deleted. The
impact of calling the ebtables code is similar to the impact of the iptables
code. It will be called in addition to the iptables code and does not replace
it. The ebtables code is only touched when a port is created or deleted, no
other time.

IPv6 Impact
-----------
Not applicable.

Other Deployer Impact
---------------------

The 'ebtables' utility needs to be installed. This is normally available in
various standard packages. On Debian, for example, it's installed like this:

    # apt-get install ebtables

Developer Impact
----------------

None.

In fact, the proposed patch is designed to have as little impact as possible.
The existing code isn't changed and both the iptables and ebtables code can be
independently refactored without impacting each other.

Community Impact
----------------

?

Alternatives
------------

It seems that iptables isn't capable of performing such filtering and
therefore can't be used for this.

Another alternative is to do nothing: This is only an issue on shared
networks. That's a business decision, though, and it seems some people are
quite eager to have this patch.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
    Juergen Brendel (jbrendel@cisco.com)


Work Items
----------

* Implement the ebtables code.
* Implement the code that uses the ebtables manager to create the spoofing
  related rules.
* Integrate code with iptables: Add the small hooks into the iptables-firewall
  code.
* All sorts of tests.


Dependencies
============

* The ebtables utility needs to be installed.


Testing
=======

Tempest Tests
-------------

Not sure yet.

Functional Tests
----------------

Funtional tests to show that the iptables and ebtables code work together
properly.

API Tests
---------

None.


Documentation Impact
====================

The need for ebtables may be listed somewhere as a requirement.

User Documentation
------------------

None.

Developer Documentation
-----------------------

None.

References
==========


* Original patch by Édouard Thuleau: https://review.openstack.org/#/c/70067/

* Launchpad blueprint:
  https://blueprints.launchpad.net/neutron/+spec/arp-spoof-patch-ebtables

* Original bug report: https://bugs.launchpad.net/neutron/+bug/1274034
