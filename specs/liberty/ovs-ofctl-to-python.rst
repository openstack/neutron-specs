..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================================
OVS agent: Use python binding instead of ovs-ofctl command
==========================================================

https://blueprints.launchpad.net/neutron/+spec/ovs-ofctl-to-python


Problem Description
===================

OVS agent uses ovs-ofctl command to program flow.
it works but has some drawbacks:

* Involves high overhead; same as ovs-vsctl described in
  other blueprint [#vsctl_bp]_.

* Difficult or impossible to handle OpenFlow async messages;
  async messsages are useful for monitoring switch state changes
  like port additions and removals [#ofagent_port_monitor_bp]_.

Proposed Change
===============

Instead of invoking ovs-ofctl each times, use OpenFlow python library
("Ryu ofproto library" [#ryu_ofproto]_) provided by Ryu SDN Framework
("Ryu" [#ryu]_) to program OVS.
This approach was taken for ofagent [#ofagent]_ and proved to work.

Plan:

    1. Turn the existing code which uses ovs-ofctl into a driver
       [#ovs_agent_ovs_ofctl_driver]_

    2. Introduce Ryu-based driver [#ovs_agent_ryu_driver]_ as a
       non-default option.

    3. Maintain both of drivers for a while.  (one release cycle?)
       It would need Experimental/non-voting jenkins jobs to cover
       the non-default driver.
       Looking the result of the jobs, once the Ryu-based driver
       gets reasonably stable, consider to switch the default.

    4. Deprecate ovs-ofctl driver later

Implementation details:

* OpenFlow version

    * OVS agent currently uses OpenFlow 1.0.
      The new Ryu-based driver will use OpenFlow 1.3 for reasons mentioned
      below.  ovs-ofctl based driver will keep using OpenFlow 1.0 with
      the exactly same flows as it currently uses.

    * OpenFlow 1.3 provides some features possibly useful for OVS agent.
      For example, ofagent uses metadata to implement "local vlan" isolation.
      OVS agent can do the same.

    * Ryu ofproto library API for OpenFlow 1.0 is not sophisticated compared
      to later versions like OpenFlow 1.3.  It works well, but only provides
      older unsophisticated API [#ryu_old_api]_.

    * OpenFlow 1.3 works well with the recent versions of Open vSwitch.
      Open vSwitch v2.0.0 introduced OpenFlow 1.3 support and
      Open vSwitch v2.3.0 enabled it by default.

    * OVS-agent itself configures the switch to use the appropriate OpenFlow
      version.  There's no need for extra operator investigations.

* Open vSwitch version in distributions:

    * Ubuntu 14.04 LTS has Open vSwitch v2.0.1 [#ovs_ubuntu_package]_.
      ofagent, which uses OpenFlow 1.3, uses that version for both of
      development and CI.  It works well [#ofagent_ci]_.

    * RHEL-OSP uses Open vSwitch 2.1.2 for Juno release.

    * Fedora 20 has Open vSwitch 2.3.0.

* XenAPI:

    * XenAPI Integration needs updates.

    * Currently OVS agent uses a special rootwrap [#xen_rootwrap]_
      [#xenapi_readme]_ to proxy ovs-ofctl/ovs-vsctl requests from
      neutron node to dom0.  This proposal replaces the use of ovs-ofctl.
      This proposal doesn't change ovs-vsctl part.

    * As we can assume IP reachability between neutron node and dom0
      [#xenapi_meeting_log]_ , a simple set-controller to teach OVS the
      IP address of the corresponding neutron node (thus the embedded
      OpenFlow controller) should be enough.

    * OVS-agent will have a new option to specify the address and port to
      listen for OpenFlow connections.

* Nicira extensions (NX)

    * Currently OVS agent uses Nicira extensions for OpenFlow 1.0 in
      a several places.  Even with OpenFlow 1.3, some of them still
      need the extensions.

    * The following is a list of Nicira extensions used by OVS agent.
      Ryu>=3.18 supports all of them.

        * Learn action

        * Local ARP responder uses Nicira extensions heavily:

            * Load
            * Move
            * NXM_NX_ARP_SHA
            * NXM_NX_ARP_THA

          For longer term, probably it might be cleaner to replace it with
          ofagent's packet-in based implementation entirely.  In that way,
          there's no need to use Nicira extensions.

        * The following Nicira extensions will be replaced by
          OpenFlow 1.3 native equivalents.

            * Tunnel support
            * Resubmit and tables

* OpenFlow channel establishment

    * Currently, on each invocation, ovs-ofctl command connects to the switch
      via unix domain socket.  Instead, this blueprint proposes to make OVS
      agent listen on a local TCP port, e.g. 127.0.0.1:6633, and wait for the
      switch to connect to that port.  TCP is the typical transport for an
      OpenFlow channel and removes a need of rootwrap.   It's the same as
      how ofagent currently works.

    * To make the above work, the switch needs to be configured to connect to
      that port.  OVS agent itself can configure the switch accordingly by
      accessing ovsdb on startup.

* Example of typical code changes

    * Current code::

        br.add_flow(priority=2,
                    in_port=in_port,
                    actions="output:%s" % out_port)

    * With Ryu::

        match = ofpp.OFPMatch(in_port=in_port)
        actions = [ ofpp.OFPActionOutput(port=out_port) ]
        instructions = [
            ofpp.OFPInstructionActions(ofp.OFPIT_APPLY_ACTIONS,
                                       actions)
        ]
        msg = ofpp.OFPFlowMod(dp,
                              priority=2,
                              match=match,
                              instructions=instructions])
        self.send_msg(msg)

    As you see, Ryu version is a straightforward representation of
    OpenFlow protocol.

Alternatives
------------

* Instead of making OVS agent switch to OpenFlow 1.3, improve Ryu's
  OpenFlow 1.0 support.  This would minimize changes in neutron.
  This isn't an attractive option though, given the limited functionality
  of OpenFlow 1.0.

* Fold ofagent into OVS agent.  It would bring in some additional ofagent
  improvements including drastic flow table design changes [#ofagent_flow]_
  and packet-in based local arp responder [#ofagent_ovs_comparison]_.
  If we want to merge these agents in a long term keeping ofagent's
  "avoid OVS-specific features as far as reasonable" property, this option
  would be the most straightforward and minimize the total amount of work a lot.
  If we go this route, probably the appropriate steps would be:

    * Port lacking features like DVR, canary table, etc to ofagent
    * Assuming it's more important to keep compatibility for OVS-agent (right?),
      resurrect features which was removed in ofagent. (br-ex support etc)
    * Once ofagent has feature parity, replace OVS-agent with it.

Data Model Impact
-----------------

none

REST API Impact
---------------

none

Security Impact
---------------

A local user on the node would be able to connect to the OpenFlow port
on which the agent listening and try to confuse the agent.  It isn't
a problem as the local management network is considered safe, though.

The OpenFlow channel can be protected using TLS if desirable.
Both of Open vSwitch and Ryu support TLS [#ryu_tls]_.
In that case, we need to introduce some agent options to specify certs.

Notifications Impact
--------------------

none

Other End User Impact
---------------------

none

Performance Impact
------------------

Per FlowMod overhead would be smaller than rootwrap and ovs-ofctl command.

IPv6 Impact
-----------

none

Other Deployer Impact
---------------------

Ryu SDN Framework will be required for OVS agent to work:

* The latest version can be pulled via pip.

* Ryu and its (non-optional) required packages are mostly pure-python
  [#ryu_req]_.  A few exceptions like eventlet are already covered by
  Neutron itself.

* Debian and Ubuntu have official packages too
  [#ryu_debian_package]_ [#ryu_ubuntu_package]_.
  However, Ryu version newer than the packages will likely be necessary
  for the Nicira extension support mentioned in "Proposed Changes" section.

* It isn't packaged for Fedora Rawhide yet.

Developer Impact
----------------

Developers of OVS agent need to be familiar with Ryu ofproto library API,
instead of ovs-ofctl command flow syntax.

Community Impact
----------------

none

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  yamamoto

Other contributors:
  kakuma

Work Items
----------

See "Proposed Change" section.

Dependencies
============

none

Testing
=======

Update and improve the existing tests if necessary.

Ryu SDN Framework will be required for gate jobs.

Experimental/non-voting jobs for the new driver will be needed.

Tempest Tests
-------------

The existing tempest tests should work as-is.

Functional Tests
----------------

Update and improve the existing tests if necessary.

There's an on-going effort for functional tests for drivers
[#ovs_agent_ofctl_functional_tests]_ .
Suggestions are welcome.

API Tests
---------

none

Documentation Impact
====================

User Documentation
------------------

A new option to specify address/port to listen for OpenFlow connections
needs to be documented.  It will be used by XenAPI integration.

If we want to support other transports like TLS, more options will be
needed:

* The option to specify the transport

* Transport specific parameters.  E.g. certs for TLS

Developer Documentation
-----------------------

It's nice to have a documentation on how to debug OpenFlow flows.
Some notes:

* Even after switching to the proposed Ryu-based driver, you can
  keep using ovs-ofctl command etc for debugging purposes as of today.

* Ryu has a functionality to JSON-serialize OpenFlow messages
  [#ryu_to_jsondict]_ .  It might be useful for logging purposes.

References
==========

.. [#vsctl_bp] https://blueprints.launchpad.net/neutron/+spec/vsctl-to-ovsdb

.. [#ryu] https://osrg.github.io/ryu/

.. [#ryu_ofproto] http://ryu.readthedocs.org/en/latest/ofproto_ref.html

.. [#ofagent] https://wiki.openstack.org/wiki/Neutron/OFAgent

.. [#ofagent_flow] https://github.com/openstack/neutron/blob/stable/juno/neutron/plugins/ofagent/agent/flows.py

.. [#ofagent_port_monitor_bp] https://blueprints.launchpad.net/neutron/+spec/ofagent-port-monitor2

.. [#ofagent_ovs_comparison] https://wiki.openstack.org/wiki/Neutron/OFAgent/ComparisonWithOVS

.. [#ovs_ubuntu_package] http://packages.ubuntu.com/trusty/net/openvswitch-switch

.. [#ryu_debian_package] https://packages.debian.org/sid/python-ryu

.. [#ryu_ubuntu_package] http://packages.ubuntu.com/vivid/python-ryu

.. [#ofagent_ci] https://wiki.openstack.org/wiki/ThirdPartySystems/OFAgent_CI

.. [#ryu_tls] http://ryu.readthedocs.org/en/latest/tls.html

.. [#ryu_old_api] http://sourceforge.net/p/ryu/mailman/message/31204306/

.. [#xen_rootwrap] https://github.com/openstack/neutron/blob/stable/juno/bin/neutron-rootwrap-xen-dom0

.. [#xenapi_readme] https://github.com/openstack/neutron/blob/stable/juno/neutron/plugins/openvswitch/agent/xenapi/README

.. [#ovs_agent_ovs_ofctl_driver] https://review.openstack.org/#/c/160245/

.. [#ovs_agent_ryu_driver] https://review.openstack.org/#/c/153946/

.. [#ovs_agent_ofctl_functional_tests] https://review.openstack.org/#/c/164584/

.. [#ryu_req] https://github.com/osrg/ryu/blob/master/tools/pip-requires

.. [#xenapi_meeting_log] http://eavesdrop.openstack.org/meetings/xenapi/2015/xenapi.2015-02-04-15.00.log.html

.. [#ryu_to_jsondict] http://ryu.readthedocs.org/en/latest/ofproto_base.html#ryu.ofproto.ofproto_parser.MsgBase.to_jsondict
