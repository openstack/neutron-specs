..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

Mitaka Postmortem documentation
===============================

.. contents::

Release Metrics
---------------

+------------------------------------------------+
| Release Metrics                                |
+===============================+================+
| Blueprints submitted          |             33 |
+-------------------------------+----------------+
| Blueprints implemented        |       21 (63%) |
+-------------------------------+----------------+
| Bug reports submitted         |           1104 |
+-------------------------------+----------------+
| Bug reports closed            |      678 (61%) |
+-------------------------------+----------------+
| Fixes released                |            538 |
+-------------------------------+----------------+
| Incomplete reports            |             59 |
+-------------------------------+----------------+
| RFEs submitted                |             33 |
+-------------------------------+----------------+
| RFEs approved                 |       27 (81%) |
+-------------------------------+----------------+

* Bug reports submitted: reports filed since Sept-23-2015 (Mitaka starts)
* Bug reports closed: reports marked released, committed, invalid or wontfix
* Fixes released: marked released/committed
* Incomplete reports: marked incomplete

.. note:: Metrics accurate at the time of writing.


Blueprints
----------

External DNS Resolution
~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: minsel
* Link: https://blueprints.launchpad.net/neutron/+spec/external-dns-resolution

  * CLI support: Complete
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Complete
  * Advanced/Sub-project support: N/A
  * Other Projects support: Complete (Nova/Designate)
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: Optional

Get Me a Network
~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: gessau
* Link: https://blueprints.launchpad.net/neutron/+spec/get-me-a-network

  * CLI support: Complete
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Complete
  * Advanced/Sub-project support: N/A
  * Other Projects support: Incomplete (`283206 <https://review.openstack.org/#/c/283206/>`_)
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: Complete
  * Horizon Support: Optional

API for l2 agent extensions
~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: ihar-hrachyshka
* Link: https://blueprints.launchpad.net/neutron/+spec/l2-api-extensions

  * CLI support: N/A
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Complete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: N/A

Modular L2 Agent
~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: andreas-scheuring
* Link: https://blueprints.launchpad.net/neutron/+spec/modular-l2-agent

  * CLI support: N/A
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: N/A
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: N/A

Support multiple L3 backends
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred
* Assignee: kevinbenton
* Link: https://blueprints.launchpad.net/neutron/+spec/multi-l3-backends
* FFE Status: Denied (Some experimental code may still make the release,
  but nothing production worthy)

  * CLI support: Incomplete
  * Server/Agent support: Incomplete
  * Testing coverage: Incomplete
  * Documentation: Incomplete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: Incomplete
  * DevStack/Grenade support: Incomplete
  * Horizon Support: N/A

Provide better user-facing mechanism to choose service capabilities
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete (Undocumented)
* Assignee: dougwig
* Link: https://blueprints.launchpad.net/neutron/+spec/neutron-flavor-framework

  * CLI support: Complete
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Incomplete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: Complete
  * Horizon Support: Optional

Split neutron into base library and servers/agents
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Ongoing effort
* Assignee: dougwig
* Link: https://blueprints.launchpad.net/neutron/+spec/neutron-lib
* FFE Status: Denied (Goal for Mitaka was for lbaas to be fully severed,
  with fw/vpn to follow. We are not there. Work will be ongoing throughout
  Newton as well. The limited goal in Mitaka was completely severing lbaas.
  At present, the library exists and is plumbed throughout the infra. The
  first rev is being used by neutron and neutron-lbaas. Patches exist for
  bumping both to the second version of the lib. More patches exist to
  delete a lot of cruft from lbaas that will mean less to migrate, and
  plans are in place to stop the dependency on test code. The remaining
  items that were aimed at Mitaka but will miss are base db model/migration
  foo, and data model foo, both of which are ongoing, but neither of which
  needs to land in the critical end of mitaka timeframe (they can iterate
  in gerrit for now.) As soon as the Mitaka branch is baked, we can:

  * turn on deprecation warnings
  * nuke lbaas v1 and v2 agent
  * start mass import renames in neutron
  * merge the above db/model items

  The goal of completely severing lbaas is realistically about 3-4
  weeks away from Feature Freeze. The separation of the remaining subprojects
  are next after that. Attempting to get patches in gerrit for the lbaas
  separation goal before the summit.

  * CLI support: N/A
  * Server/Agent support: See FFE notes
  * Testing coverage: See FFE notes
  * Documentation: Incomplete
  * Advanced/Sub-project support: See FFE notes
  * Other Projects support: See FFE notes
  * OpenStack Infra support: Complete
  * DevStack/Grenade support: Complete
  * Horizon Support: N/A

Use push style notifications for all server->agent information
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred
* Assignee: kevinbenton
* Link: https://blueprints.launchpad.net/neutron/+spec/push-notifications
* FFE Status: Denied (minor enhancemetns may be allowed as RC bugs, e.g.
  `280595 <https://review.openstack.org/#/c/280595/>`_).

  * CLI support: N/A
  * Server/Agent support: Incomplete
  * Testing coverage: Incomplete
  * Documentation: N/A
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: N/A

Restructure L2 agent
~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: rossella-o
* Link: https://blueprints.launchpad.net/neutron/+spec/restructure-l2-agent

  * CLI support: N/A
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: N/A
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: N/A

Clean up resources when a tenant is deleted
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: john-davidge
* Link: https://blueprints.launchpad.net/neutron/+spec/tenant-delete
* FFE Status: Granted

  * CLI support: Complete
  * Server/Agent support: N/A
  * Testing coverage: Complete
  * Documentation: Complete
  * Advanced/Sub-project support: Optional
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: Optional

Add availability zones for agents
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: ichihara-hirofumi
* Link: https://blueprints.launchpad.net/neutron/+spec/add-availability-zone

  * CLI support: Complete
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Complete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: Optional

Add tags to core resources
~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: ichihara-hirofumi
* Link: https://blueprints.launchpad.net/neutron/+spec/add-tags-to-core-resources
* FFE Status: Granted (docs pending)

  * CLI support: Complete
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Complete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: Optional

Add timestamp to neutron resources
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: zhaobo6
* Link: https://blueprints.launchpad.net/neutron/+spec/add-timestamp-attr
* FFE Status: Granted

  * CLI support: N/A
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: N/A
  * Advanced/Sub-project support: Optional
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: N/A

Introduce Address Scopes
~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: carl-baldwin
* Link: https://blueprints.launchpad.net/neutron/+spec/address-scopes
* FFE Status: Granted (docs pending)

  * CLI support: Complete
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Incomplete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: Optional

* References

  * https://review.openstack.org/#/c/286294/

Automatically generate etc/neutron.conf file
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: martin-hickey
* Link: https://blueprints.launchpad.net/neutron/+spec/autogen-neutron-conf-file

  * CLI support: N/A
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Complete
  * Advanced/Sub-project support: Complete
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: Complete
  * Horizon Support: N/A

* References:

  * http://docs.openstack.org/draft/config-reference/networking/samples/
  * http://docs.openstack.org/developer/neutron/devref/contribute.html#configuration-files

Router Extension for Dynamic Routing Using BGP
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: ryan-tidwell
* Link: https://blueprints.launchpad.net/neutron/+spec/bgp-dynamic-routing
* FFE Status: Granted

  * CLI support: Complete
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Complete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: Complete
  * Horizon Support: Optional

* References:

  * https://review.openstack.org/#/c/288856/
  * https://review.openstack.org/#/q/topic:bp/bgp-dynamic-routing
  * https://review.openstack.org/#/c/268726/ (to be spun out)

Allow for per-subnet dhcp options
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred
* Assignee: sambetts
* Link: https://blueprints.launchpad.net/neutron/+spec/dhcp-options-per-subnet
* FFE Status: Denied (Unable to determine status)

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack/Grenade support:
  * Horizon Support:

Add Guru Meditation Report Functionality to Neutron
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: ihar-hrachyshka
* Link: https://blueprints.launchpad.net/neutron/+spec/guru-meditation-report
* FFE Status: Granted (to provide guru support to vpn/lb/fwass)

  * CLI support: N/A
  * Server/Agent support: N/A
  * Testing coverage: Complete
  * Documentation: Incomplete
  * Advanced/Sub-project support: Incomplete
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: Incomplete (`279035 <https://review.openstack.org/#/c/279035/>`_ needs review)
  * Horizon Support: N/A

* References:

  * https://review.openstack.org/#/c/287795/
  * https://review.openstack.org/#/c/287801/

Improve DVR router sheduling mechanism for better performance/scalability
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: obondarev
* Link: https://blueprints.launchpad.net/neutron/+spec/improve-dvr-l3-agent-binding

  * CLI support: N/A
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: N/A
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: N/A

Moving to Keystone v3
~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred
* Assignee: smigiel-dariusz
* Link: https://blueprints.launchpad.net/neutron/+spec/keystone-v3
* FFE Status: Denied (Neutron Server supports v3, but schema and API
  migration is still undergoing).

  * CLI support: Incomplete
  * Server/Agent support: Incomplete
  * Testing coverage: Incomplete
  * Documentation: Incomplete (`281357 <https://review.openstack.org/#/c/281357/>`_ needs work).
  * Advanced/Sub-project support: Incomplete
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: Optional
  * Horizon Support: N/A

LBaaS Layer 7 rules
~~~~~~~~~~~~~~~~~~~

* Status: Incomplete (Undocumented)
* Assignee: avishayb
* Link: https://blueprints.launchpad.net/neutron/+spec/lbaas-l7-rules
* FFE Status: Granted (docs pending)

  * CLI support: Complete
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Incomplete
  * Advanced/Sub-project support: Complete
  * Other Projects support:  Complete (Octavia)
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: Incomplete

ML2/LinuxBridge QoS support with bandwidth limiting
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: slaweq
* Link: https://blueprints.launchpad.net/neutron/+spec/ml2-lb-ratelimit-support

  * CLI support: N/A
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Complete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: Optional
  * DevStack/Grenade support: Complete
  * Horizon Support: N/A

Adds a network-ip-usage api to fetch network and subnet IP usage counts
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: mdorman-m
* Link: https://blueprints.launchpad.net/neutron/+spec/network-ip-usage-api
* FFE Status: Granted

  * CLI support: Complete
  * Server/Agent support: Complete
  * Testing coverage: Incomplete
  * Documentation: Complete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: Optional

Role-based Access Control for QoS policies
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: hdaniel
* Link: https://blueprints.launchpad.net/neutron/+spec/rbac-qos
* FFE Status: Granted

  * CLI support: Complete
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Complete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: Optional

enable vhost-user support with ovs agent
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: sean-k-mooney
* Link: https://blueprints.launchpad.net/neutron/+spec/vhost-ovs

  * CLI support: N/A
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Complete (user guide desirable)
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: N/A

* References

  * http://docs.openstack.org/developer/neutron/devref/ovs_vhostuser.html

Allow vm to boot without l3 address(subnet)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred
* Assignee: yalei-wang
* Link: https://blueprints.launchpad.net/neutron/+spec/vm-without-l3-address
* FFE Status: Denied (`vm-without-l3-address <https://review.openstack.org/#/q/topic:bp/vm-without-l3-address>`_
  shows a post-nuclear landscape).

  * CLI support: Incomplete
  * Server/Agent support: Incomplete
  * Testing coverage: Incomplete
  * Documentation: Incomplete
  * Advanced/Sub-project support: N/A
  * Other Projects support: Incomplete (Nova)
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: Optional

* References:

  * https://review.openstack.org/#/c/239276/
  * https://review.openstack.org/#/q/topic:bp/vm-without-l3-address

allow multiple subnets to connect to vpn
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: pcm
* Link: https://blueprints.launchpad.net/neutron/+spec/vpn-multiple-subnet

  * CLI support: Complete
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Complete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: Optional

RFE
---

Openstack services should support SIGHUP signal
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: eezhova
* Link: https://bugs.launchpad.net/neutron/+bug/1276694

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack/Grenade support:
  * Horizon Support:

[RFE] Neutron support for OSprofiler
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred
* Assignee: salvatore-orlando
* Link: https://bugs.launchpad.net/neutron/+bug/1335640

  * CLI support: N/A
  * Server/Agent support: Incomplete
  * Testing coverage: Incomplete
  * Documentation: Incomplete
  * Advanced/Sub-project support: Incomplete
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: N/A

* References

  * https://review.openstack.org/#/c/273951/


[RFE] Unable to create a router thats both HA and distributed
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: carl-baldwin
* Link: https://bugs.launchpad.net/neutron/+bug/1365473
* FFE Status: Granted (needs docs)

  * CLI support: Complete
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Complete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: Incomplete
  * DevStack/Grenade support: N/A
  * Horizon Support: Optional

* References

  * https://review.openstack.org/#/c/296711/
  * https://review.openstack.org/#/c/296836/

[RFE] [LBaaS] ssh connection timeout
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred
* Assignee: reedip-banerjee
* Link: https://bugs.launchpad.net/neutron/+bug/1457556
* FFE Status: Denied (no tangible progress)

  * CLI support: Incomplete
  * Server/Agent support: Incomplete
  * Testing coverage: Incomplete
  * Documentation: Incomplete
  * Advanced/Sub-project support: N/A
  * Other Projects support: Incomplete
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: Optional

[rfe] openvswitch based firewall driver
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete (partially documented)
* Assignee: libosvar
* Link: https://bugs.launchpad.net/neutron/+bug/1461000
* FFE Status: Granted

  * CLI support: N/A
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Incomplete (user guide docs desirable)
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: Optional
  * Horizon Support: N/A

* References:

  * http://docs.openstack.org/developer/neutron/devref/openvswitch_firewall.html
  * https://review.openstack.org/#/c/284259
  * https://review.openstack.org/#/c/283137

[RFE] Create a full load balancing configuration with one API call
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred
* Assignee: trevor-vardeman
* Link: https://bugs.launchpad.net/neutron/+bug/1463202
* FFE Status: Denied (too many missing pieces).

  * CLI support: Incomplete
  * Server/Agent support: Incomplete
  * Testing coverage: Incomplete
  * Documentation: Incomplete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: Incomplete

RFE: Security Rules should support VRRP protocol
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: sreesiv
* Link: https://bugs.launchpad.net/neutron/+bug/1475717

  * CLI support: Optional
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: N/A
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: Optional

[RFE] Allow annotations on Neutron resources
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: kevinbenton
* Link: https://bugs.launchpad.net/neutron/+bug/1483480
* FFE Status: Granted

  * CLI support: N/A
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: N/A
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: Optional

[RFE] DHCP agent should provide ipv6 RAs for isolated networks with ipv6 subnets
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred
* Assignee: ihar-hrachyshka
* Link: https://bugs.launchpad.net/neutron/+bug/1498987

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack/Grenade support:
  * Horizon Support:

RFE Replace the existing default subnetpool configuration options with an admin-only API
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: john-davidge
* Link: https://bugs.launchpad.net/neutron/+bug/1501328
* FFE Status: Granted (docs pending)

  * CLI support: Complete
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Complete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: Optional

* References

  * https://review.openstack.org/#/c/286293/

[RFE] Improve SG performance as VMs/containers scale on compute node
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: kevinbenton
* Link: https://bugs.launchpad.net/neutron/+bug/1502297

  * CLI support: N/A
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: N/A
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: N/A

RFE There is no facility to name LBaaS v2 Members and Health Monitors
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete (undocumented)
* Assignee: reedip-banerjee
* Link: https://bugs.launchpad.net/neutron/+bug/1515506

  * CLI support: Complete
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Incomplete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: Optional

[RFE] IPAM migration from non-pluggable to pluggable
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred
* Assignee: pasha117
* Link: https://bugs.launchpad.net/neutron/+bug/1516156

  * CLI support: N/A (server side entry point)
  * Server/Agent support: N/A
  * Testing coverage: Incomplete
  * Documentation: Incomplete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: Incomplete
  * DevStack/Grenade support: Incomplete
  * Horizon Support: Incomplete

* References:

 * https://review.openstack.org/#/c/277767

[RFE] Transition neutron CLI from python-neutronclient to python-openstackclient
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred
* Assignee: rtheis
* Link: https://bugs.launchpad.net/neutron/+bug/1521291
* FFE Status: Denied (even though `Devref patch <https://review.openstack.org/#/c/282555/>`_
  will likely merge, the actual work will need to move to Newton. OSC in Mitaka will have a
  significant increase in support for core neutron resources Detailed status available at
  https://etherpad.openstack.org/p/osc-neutron-support.

  * CLI support: Incomplete
  * Server/Agent support: N/A
  * Testing coverage: Incomplete
  * Documentation: Incomplete
  * Advanced/Sub-project support: Incomplete
  * Other Projects support: Incomplete
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: Incomplete
  * Horizon Support: N/A

RfE: Cascading delete for LBaaS Objects
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred
* Assignee: german-eichberger
* Link: https://bugs.launchpad.net/neutron/+bug/1521783
* FFE Status: Denied

  * CLI support: Incomplete
  * Server/Agent support: Incomplete
  * Testing coverage: Incomplete
  * Documentation: Incomplete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: Complete

* References

  * https://review.openstack.org/#/c/287593/
  * https://review.openstack.org/#/c/288187/

[RFE] use oslo-versioned-objects to help with dealing with upgrades
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred
* Assignee: justin-hammond
* Link: https://bugs.launchpad.net/neutron/+bug/1522102
* FFE Status: Denied (Wont happen in Mitaka. Should be moved to Newton.
  Was not expected to land in Mitaka, the bug is just a placeholder for a
  large effort).

  * CLI support: N/A
  * Server/Agent support: Incomplete
  * Testing coverage: Incomplete
  * Documentation: Incomplete
  * Advanced/Sub-project support: Incomplete
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: N/A

RFE Add support for external vxlan encapsulation to neutron router
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred
* Assignee: ruansx
* Link: https://bugs.launchpad.net/neutron/+bug/1525059
* FFE Status: Denied (too many parts lacking exhaustive review)

  * CLI support: Incomplete
  * Server/Agent support: Incomplete
  * Testing coverage: Incomplete
  * Documentation: Incomplete
  * Advanced/Sub-project supprt: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: Optional
  * Horizon Support: Optional

* References:

  * https://review.openstack.org/#/q/topic:bug/1525059

[RFE] Security groups resources are not extendable
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: roeyc
* Link: https://bugs.launchpad.net/neutron/+bug/1529109

  * CLI support: N/A
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: N/A
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack/Grenade support: N/A
  * Horizon Support: N/A

RFE Add F5 plugin driver to neutron-lbaas
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Deferred
* Assignee: j-longstaff
* Link: https://bugs.launchpad.net/neutron/+bug/1539717
* FFE Status: Denied (no met requirements for inclusion)

  * CLI support: N/A
  * Server/Agent support: Incomplete
  * Testing coverage: Incomplete
  * Documentation: N/A
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: Incomplete
  * DevStack/Grenade support: N/A
  * Horizon Support: N/A
