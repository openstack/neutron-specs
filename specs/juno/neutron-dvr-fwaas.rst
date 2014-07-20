..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================================
FWaaS changes to support Distributed Virtual Router(DVR)
========================================================

https://blueprints.launchpad.net/neutron/+spec/neutron-dvr-fwaas

Problem description
===================
As we move to the DVR model, current L3 based services such as FWaaS that rely
on seeing both sides of the traffic on the same namespace router are broken.
The reference implementation of FWaaS is built on iptables and is stateful in
that connections are tracked. DVR by intent distributes the routing
functionality across instances on compute nodes and all traffic is no longer
routed through a centralized router to achieve goals of scalability.

Proposed change
===============
To maintain FWaaS functionality in the context of a DVR deployment the first
step (which is being targeted in this proposal)  is to ensure that we can
preserve current perimeter Firewall functionality (North - South) and ensure
that the manner in which rules are applied does not break normal DVR
East - West traffic flow.

DVR[1] classifies North - South traffic into two categories:

Default SNAT use case - traffic flowing out to the external network will go
through the Network/Service Node, and the SNAT Namespace hosted there, to get
out. Response traffic will again go thru the SNAT Namespace for translation
and is then directly switched to the appropriate VM in a Compute Node.

VMs with Floating IPs(FIP) use case - traffic flowing out will go through the
Internal Router for routing and translation and then through the FIP Namespace
on the Compute Node to the external network through its br-ex. And on the
incoming traffic, again the flow is through the br-ex on the Compute Node,
then the FIP Namespace and through the Internal Router which also handles
translation to the VM.

DVR runs L3Agent on the Network Node as well as on the Compute Nodes. Since
the FWaaS Agent runs in the context of the L3Agent, the FWaaS Plugin to Agent
messaging is already available. In the context of both of the above use cases,
FWaaS Agent on the Network/Service Node should install the Firewall Rules in
the SNAT Namespace, and the FWaaS Agent in the Compute Nodes should install
the Firewall Rules in the IR Namespace to be applied on traffic flowing
through the FIP Namespace (to ensure that there is no effect on East - West
flows). In this manner, the FWaaS requirement of seeing traffic on both
directions is satisfied for both of these use cases. FWaaS Agent can use the
agent_mode configuration option to determine whether it is on a
Network/Service node or Compute node to make the determination on how to
install the rules.

The SNAT interfaces are prefixed with 'sq-' in the SNAT namespace. The
interface of interest in the IR namespace has the prefix 'rfp-'. The API's
being added as part of DVR support in the L3Agent[2] will be used by the FWaaS
Agent to make this determination for inserting the Firewall Rules
appropriately.

Alternatives
------------
None. This is required as DVR - FWaaS integration will be broken.

Data model impact
-----------------
No impact, the changes are on the Agent/iptables Driver.

REST API impact
---------------
No new REST API is introduced.

Security impact
---------------
The effort is to minimize the Security impact as DVR functionality breaks
FWaaS. Coverage for North - South traffic but no support for East - West
traffic which will be addressed as a separate proposal post Juno.

Notifications impact
--------------------
None to existing.

Other end user impact
---------------------
No Support for FWaaS on East - West traffic.

Performance Impact
------------------
None.

Other deployer impact
---------------------
Deployers should be aware that this only targets North - South functionality.

Developer impact
----------------
None.

Implementation
==============

Assignee(s)
-----------
Primary assignee: skandasw / badveli_vishnuus / (TBD)
Other contributors: TBD

Work Items
----------
* FWaaS Agent changes to recognize agent_mode to identify appropriate
  Namespace and interface.
* Corresponding FWaaS driver changes.

Dependencies
============
Patches awaiting merge on [1].

Testing
=======
Unit tests, Tempest API tests.

Documentation Impact
====================
DVR and FWaaS documentation changes to reflect requirements.

References
==========
[1] https://blueprints.launchpad.net/neutron/+spec/neutron-ovs-dvr
[2] https://review.openstack.org/#/c/89413/
