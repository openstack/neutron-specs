..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

Newton Postmortem documentation
===============================

.. contents::

Release Metrics
---------------

+------------------------------------------------+
| Release Metrics                                |
+===============================+================+
| Blueprints targeted           |             25 |
+-------------------------------+----------------+
| Blueprints/RFE implemented    |       14 (56%) |
+-------------------------------+----------------+
| Bug reports submitted         |            789 |
+-------------------------------+----------------+
| Bug reports closed            |      428 (54%) |
+-------------------------------+----------------+
| Fixes released                |      315 (40%) |
+-------------------------------+----------------+
| Incomplete reports            |             51 |
+-------------------------------+----------------+
| RFEs submitted                |             49 |
+-------------------------------+----------------+

* Bug reports submitted: reports filed since Mar-16-2016 (Newton starts)
* Bug reports closed: reports marked released, committed, invalid or wontfix
* Fixes released: marked released/committed
* Incomplete reports: marked incomplete

.. note:: Metrics accurate at the time of writing.


Blueprints
----------

Firewall as a Service API 2.0
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: skandasw
* Link: https://blueprints.launchpad.net/neutron/+spec/fwaas-api-2.0
* FFE: Granted (testing and documentation lacking)

  * CLI support: Incomplete
  * Server/Agent support: Complete (see release notes for more details)
  * Testing coverage: Incomplete (unit testing only)
  * Documentation: Incomplete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: Incomplete
  * DevStack support: Complete
  * Horizon Support: Optional

* References

  * https://review.openstack.org/#/c/351582/
  * https://review.openstack.org/#/c/355576/
  * https://review.openstack.org/#/c/359320/
  * https://review.openstack.org/#/c/366916/

API for l2 agent extensions
~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: mangelajo
* Link: https://blueprints.launchpad.net/neutron/+spec/l2-api-extensions
* FFE: Denied (still at specification stage)

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

* References

  * Spec: https://review.openstack.org/#/c/320439/

Support multiple L3 backends
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete (pending documentation)
* Assignee: kevinbenton
* Link: https://blueprints.launchpad.net/neutron/+spec/multi-l3-backends
* FFE: Granted (small testing/devref gaps are being filled)

  * CLI support: Complete
  * Server/Agent support: Complete
  * Testing coverage: Complete (Init, API)
  * Documentation: In progress
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: Optional

* References

  * https://review.openstack.org/#/c/364001/
  * https://review.openstack.org/#/c/358866/

Split neutron into base library and servers/agents
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Ongoing
* Assignee: dougwig
* Link: https://blueprints.launchpad.net/neutron/+spec/neutron-lib
* FFE: N/A

  * CLI support: N/A
  * Server/Agent support: N/A
  * Testing coverage: N/A
  * Documentation: Complete
  * Advanced/Sub-project support: Ongoing
  * Other Projects support: N/A
  * OpenStack Infra support: Complete
  * DevStack support: Complete
  * Horizon Support: N/A

* References

  * http://docs.openstack.org/developer/neutron-lib/

Upgrade controllers with no API downtime
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: ihar-hrachyshka
* Link: https://blueprints.launchpad.net/neutron/+spec/online-upgrades
* FFE: Denied: no work happened, will be more active in Ocata; has a
  dependency on adopt-oslo-versioned-objects-for-db.

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

Use push style notifications for all server->agent information
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete (>50% complete - to land early in Ocata-1)
* Assignee: kevinbenton
* Link: https://blueprints.launchpad.net/neutron/+spec/push-notifications
* FFE: Denied (due to incomplete OVO refactoring).

  * CLI support: N/A
  * Server/Agent support: Incomplete
  * Testing coverage: Incomplete
  * Documentation: Incomplete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: N/A

Support Routed Networks in Neutron
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete (pending client and Nova support).
* Assignee: carl-baldwin
* Link: https://blueprints.launchpad.net/neutron/+spec/routed-networks
* FFE: Granted

  * CLI support: OSC bindings incomplete
  * Server/Agent support: Complete
  * Testing coverage: (unit, more in progress)
  * Documentation: In progress
  * Advanced/Sub-project support: N/A
  * Other Projects support: Nova scheduler support is incomplete.
  * OpenStack Infra support: N/A
  * DevStack support: Complete
  * Horizon Support: Optional

* References

  * https://review.openstack.org/#/c/302395/
  * https://review.openstack.org/#/c/302223/
  * https://review.openstack.org/#/c/347188/
  * https://review.openstack.org/#/c/353115/
  * https://review.openstack.org/#/c/356013/

Diagnostics of Neutron components
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: hmlnarik-s
* Link: https://blueprints.launchpad.net/neutron/+spec/troubleshooting
* FFE: Denied (still at specification stage)

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

* References

  * https://review.openstack.org/#/c/308973/

VLAN aware VMs
~~~~~~~~~~~~~~

* Status: Complete (pending documentation)
* Assignee: rossella-o
* Link: https://blueprints.launchpad.net/neutron/+spec/vlan-aware-vms
* FFE: Granted (OVS and Linuxbridge agent-side patches need merging but
  are moving at fast pace, and the bulk has already merged in a while;
  small gaps to fill after that).

  * CLI support: Complete
  * Server/Agent support: Complete (pending LB+OVS agent patches)
  * Testing coverage: Complete (unit, functional, API)
  * Documentation: In progress
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: Complete
  * Horizon Support: Optional

* References

  * https://review.openstack.org/#/c/347466/
  * https://review.openstack.org/#/c/346377/
  * https://review.openstack.org/#/c/361776/

Adopt oslo.versionedobjects for database interactions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: ihar-hrachyshka
* Link: https://blueprints.launchpad.net/neutron/+spec/adopt-oslo-versioned-objects-for-db
* FFE: Granted (Slipping into Ocata)

  * CLI support: N/A
  * Server/Agent support: Incomplete
  * Testing coverage: Incomplete
  * Documentation: Incomplete
  * Advanced/Sub-project support: Incomplete
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: N/A

BGP Dynamic Routing spin out
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: vikschw
* Link: https://blueprints.launchpad.net/neutron/+spec/bgp-spinout
* FFE: Granted

  * CLI support: Complete (OSC bindings incomplete)
  * Server/Agent support: Complete
  * Testing coverage: Complete (unit, API, functional)
  * Documentation: Complete
  * Advanced/Sub-project support: Complete
  * Other Projects support: N/A
  * OpenStack Infra support: Complete
  * DevStack support: Complete
  * Horizon Support: Optional

* References

  * https://review.openstack.org/#/c/340763/

Allow for per-subnet dhcp options
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: sambetts
* Link: https://blueprints.launchpad.net/neutron/+spec/dhcp-options-per-subnet
* FFE: Denied

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

Use the new enginefacade from oslo_db
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete (>50% complete)
* Assignee: akamyshnikova
* Link: https://blueprints.launchpad.net/neutron/+spec/enginefacade-switch
* FFE: Granted (bulk of the code to enable adoption of new engine facade merged.
  There are more follow ups to go in Ocata).

  * CLI support: N/A
  * Server/Agent support: N/A
  * Testing coverage: Complete (unit, functional)
  * Documentation: Incomplete
  * Advanced/Sub-project support: Incomplete
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: N/A

Allow instance-ingress bandwidth limiting
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: slaweq
* Link: https://blueprints.launchpad.net/neutron/+spec/instance-ingress-bw-limit
* FFE: Denied (a few patches in conflict/stale).

  * CLI support: Incomplete
  * Server/Agent support: Incomplete
  * Testing coverage: Incomplete
  * Documentation: Incomplete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: N/A

* References

  * https://review.openstack.org/#/c/356690/
  * https://review.openstack.org/#/c/357055/
  * https://review.openstack.org/#/c/303626/
  * https://review.openstack.org/#/c/341186/

Moving to Keystone v3
~~~~~~~~~~~~~~~~~~~~~

* Status: Complete (pending documentation and codebase cleanup)
* Assignee: smigiel-dariusz
* Link: https://blueprints.launchpad.net/neutron/+spec/keystone-v3
* FFE: Granted (to provide API support).

  * CLI support: Complete
  * Server/Agent support: Complete
  * Testing coverage: Complete (Unit, functional, API)
  * Documentation: Incomplete (api-ref to be updated)
  * Advanced/Sub-project support: Complete (deprecation warnings emitted)
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: N/A

* References

  * https://review.openstack.org/#/c/357977/
  * https://review.openstack.org/#/c/372857/

Add agent extension framework for L3 agent
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: njohnston
* Link: https://blueprints.launchpad.net/neutron/+spec/l3-agent-extensions
* FFE: Granted (bulk of functionality is merged, increased coverage, or
  more documentation should be allowed to go in).

  * CLI support: N/A
  * Server/Agent support: Complete
  * Testing coverage: Complete (unit)
  * Documentation: Complete
  * Advanced/Sub-project support: Complete
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support:  Complete
  * Horizon Support: N/A

* References

  * http://docs.openstack.org/developer/neutron/devref/agent_extensions.html

QoS support with dscp marking
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: victor-r-howard
* Link: https://blueprints.launchpad.net/neutron/+spec/ml2-ovs-qos-with-dscp

  * CLI support: Complete (from python-neutronclient 4.2.0)
  * Server/Agent support: Complete
  * Testing coverage: Complete (unit, API, functional, fullstack)
  * Documentation: Complete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: Complete
  * DevStack support: Complete
  * Horizon Support: Optional

* References

  * http://docs.openstack.org/draft/networking-guide/config-qos.html
  * http://docs.openstack.org/cli-reference/neutron.html
  * http://docs.openstack.org/security-guide/networking/services.html

Neutron in-tree API reference
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Ongoing
* Assignee: amotoki
* Link: https://blueprints.launchpad.net/neutron/+spec/neutron-in-tree-api-ref

  * CLI support: N/A
  * Server/Agent support: N/A
  * Testing coverage: N/A
  * Documentation: In progress
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: Complete
  * DevStack support: N/A
  * Horizon Support: N/A

* References

  * http://developer.openstack.org/api-ref/networking/

QoS minimum egrees bandwidth
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete (pending documentation)
* Assignee: rodolfo-alonso-hernandez
* Link: https://blueprints.launchpad.net/neutron/+spec/qos-min-egress-bw
* FFE: Granted (close to being complete, OVS and Linuxbridge missing
  the implementation).

  * CLI support: Complete
  * Server/Agent support: Complete
  * Testing coverage: Complete (no fullstack testing)
  * Documentation: Incomplete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: Optional

* Reference

  * https://review.openstack.org/#/c/344145/
  * https://review.openstack.org/#/c/347302/
  * https://review.openstack.org/#/c/351833/

Improved validation mechanism for QoS rules with port types
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: slaweq
* Link: https://blueprints.launchpad.net/neutron/+spec/qos-rules-validation
* FFE: Denied (requires more work that will slip into Ocata)

  * CLI support: N/A
  * Server/Agent support: Incomplete
  * Testing coverage: Incomplete
  * Documentation: Incomplete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: N/A

* Rerefences

  * https://review.openstack.org/#/c/319694/
  * https://review.openstack.org/#/c/351858/

Security-group Logging
~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: y-furukawa-2
* Link: https://blueprints.launchpad.net/neutron/+spec/security-group-logging
* FFE: Denied (still at specification stage)

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

* References

  * https://review.openstack.org/#/c/203509/

Differentiate between service and floating subnets
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete (pending client support)
* Assignee: john-davidge
* Link: https://blueprints.launchpad.net/neutron/+spec/service-subnets
* FFE: Granted (nearly complete, pending CLI)

  * CLI support: Incomplete
  * Server/Agent support: Complete
  * Testing coverage: Complete (unit)
  * Documentation: Complete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: Optional

* References

  * http://docs.openstack.org/draft/networking-guide/config-service-subnets.html
  * https://review.openstack.org/#/c/360526/

Enable adoption of an existing subnet into a subnetpool
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: ryan-tidwell
* Link: https://blueprints.launchpad.net/neutron/+spec/subnet-onboard
* FFE: Denied (server side patch needs some love).

  * CLI support: Incomplete
  * Server/Agent support: In progress
  * Testing coverage: In progress
  * Documentation: Incomplete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: Optional

* References

  * https://review.openstack.org/#/c/348080/

Allow vm to boot without l3 address(subnet)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete (>50% complete)
* Assignee: carl-baldwin
* Link: https://blueprints.launchpad.net/neutron/+spec/vm-without-l3-address
* FFE: Granted (patch actively under review)

  * CLI support: Incomplete (allow creation of ports with no fixed IPs)
  * Server/Agent support: Complete
  * Testing coverage: Incomplete (test how security groups behave with an unaddressed ports)
  * Documentation: In progress
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: Optional

* References

  * https://review.openstack.org/#/c/361455

add-neutron-extension-resource-timestamp
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: zhaobo
* Link: https://blueprints.launchpad.net/neutron/+spec/add-neutron-extension-resource-timestamp
* FFE: Granted (cleanup/refactoring patches from kevinbenton)

  * CLI support: N/A
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: N/A
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: Optional

RFEs
----

Openstack services should support SIGHUP signal
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: eezhova
* Link: https://bugs.launchpad.net/neutron/+bug/1276694
* FFE: Denied

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

[RFE] Neutron support for OSprofiler
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: dbelova
* Link: https://bugs.launchpad.net/neutron/+bug/1335640

  * CLI support: N/A
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Complete (release notes)
  * Advanced/Sub-project support: Complete
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: N/A

[RFE] [LBaaS] ssh connection timeout
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: reedip-banerjee
* Link: https://bugs.launchpad.net/neutron/+bug/1457556
* FFE: Denied (not progress, not worked on)

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

[RFE] Add API to set ipv6 gateway
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: scollins
* Link: https://bugs.launchpad.net/neutron/+bug/1460720
* FFE: Denied (not progress, not worked on)

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

[RFE] Create a full load balancing configuration with one API call
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: trevor-vardeman
* Link: https://bugs.launchpad.net/neutron/+bug/1463202
* FFE: Granted (to fill testing/documentation gaps).

  * CLI support: Incomplete
  * Server/Agent support: Complete
  * Testing coverage: Complete (unit)
  * Documentation: Incomplete
  * Advanced/Sub-project support: Complete
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: Optional

[RFE] Add the ability to create lb vip and member with network_id
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: dougwig
* Link: https://bugs.launchpad.net/neutron/+bug/1465758
* FFE: Granted

  * CLI support: Incomplete
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Incomplete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: Optional

* References

  * https://review.openstack.org/#/c/363302/

RFE: Pure Python driven Linux network configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: gus
* Link: https://bugs.launchpad.net/neutron/+bug/1492714
* FFE: Denied (not progress, not worked on).

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

[RFE] DHCP agent should provide ipv6 RAs for isolated networks with ipv6 subnets
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: None
* Link: https://bugs.launchpad.net/neutron/+bug/1498987
* FFE: Denied (not progress, not worked on).

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

[RFE] IPAM migration from non-pluggable to pluggable
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete (pending documentation)
* Assignee: carl-baldwin
* Link: https://bugs.launchpad.net/neutron/+bug/1516156

  * CLI support: N/A
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Incomplete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: N/A
  * DevStack support: N/A
  * Horizon Support: N/A

* References

  * https://bugs.launchpad.net/neutron/+bug/1588984

[RFE] Transition neutron CLI from python-neutronclient to python-openstackclient
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Ongoing
* Assignee: rtheis
* Link: https://bugs.launchpad.net/neutron/+bug/1521291
* FFE: N/A (OSC in Mitaka has a significant increase in support for core Neutron
  resources. In addition, there is now an OSC plugin for some advanced neutron
  features and services. The overall status is available in the `docs <http://docs.openstack.org/developer/python-neutronclient/devref/transition_to_osc.html>`_
  with a detailed status available on `etherpad <https://etherpad.openstack.org/p/osc-neutron-support>`_.

  * CLI support: Incomplete
  * Server/Agent support: N/A
  * Testing coverage: Incomplete
  * Documentation: Incomplete
  * Advanced/Sub-project support: Incomplete
  * Other Projects support: Incomplete
  * OpenStack Infra support: Complete
  * DevStack support: Incomplete
  * Horizon Support: N/A

[RFE] Cascading delete for LBaaS Objects
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: brandon-logan
* Link: https://bugs.launchpad.net/neutron/+bug/1521783
* FFE: Denied (not currently worked on).

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

[RFE] Add support for external vxlan encapsulation to neutron router
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: ruansx
* Link: https://bugs.launchpad.net/neutron/+bug/1525059
* FFE: Denied

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:

ML2 OpenvSwitch Agent GRE/VXLAN tunnel code does not support IPv6 addresses as tunnel endpoints
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Complete
* Assignee: brian-haley
* Link: https://bugs.launchpad.net/neutron/+bug/1525895

  * CLI support: N/A
  * Server/Agent support: Complete
  * Testing coverage: Complete
  * Documentation: Complete
  * Advanced/Sub-project support: N/A
  * Other Projects support: N/A
  * OpenStack Infra support: Complete (dsvm-neutron-serviceipv6 experimental job)
  * DevStack support: Incomplete
  * Horizon Support: N/A

* References

  * http://docs.openstack.org/draft/networking-guide/config-ipv6.html
  * https://bugs.launchpad.net/devstack/+bug/1619476

[RFE] Add F5 plugin driver to neutron-lbaas
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: Incomplete
* Assignee: None
* Link: https://bugs.launchpad.net/neutron/+bug/1539717
* FFE: Denied (no active progress)

  * CLI support:
  * Server/Agent support:
  * Testing coverage:
  * Documentation:
  * Advanced/Sub-project support:
  * Other Projects support:
  * OpenStack Infra support:
  * DevStack support:
  * Horizon Support:


Miscellaneous
~~~~~~~~~~~~~

* Switch to OVSDB and OpenFlow native interfaces.

  * Status: Complete
  * Assignee: Terry Wilson, IWAMOTO Toshihiro
  * References:

    * https://review.openstack.org/#/c/299655/ (ovsdb)
    * https://review.openstack.org/#/c/319770/ (openflow)

  * Agent support: Complete
  * Testing coverage: Complete (Functional tests show parity,
    Tempest API and scenario tests use the new default)
  * Documentation: config options being documented.
