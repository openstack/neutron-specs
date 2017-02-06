..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================
Port data plane status
======================

https://bugs.launchpad.net/neutron/+bug/1598081

Neutron may not be well equipped to detect data plane failures affecting the
underlying networking infrastructure. This spec addresses that issue by means
of allowing external tools to report to Neutron about faults in the underlying
data plane that are affecting the ports. A new REST API field is proposed to
that end.


Problem Description
===================

An initial description of the problem was introduced in bug #159801 [1_]. This
spec focuses on capturing one (main) part of the problem there described, i.e.
extending Neutron's REST API to cover the scenario of allowing external tools
to report network failures to Neutron. Out of scope of this spec are works to
enable port status changes to be received and managed by mechanism drivers.

This spec provides the plumbing to address bug #1575146 [2_] in subsequent
work. Specifically, and argued by the Neutron driver team in [3_]:

 * Neutron should not shut down the port completely upon detection of underlay
   network failure; connectivity between instances on the same node may still
   be reachable. External tools may or may not want to trigger a status change
   on the port based on their own logic and orchestration.

 * Port down is not detected when an uplink of a switch is down.

 * The physical network bridge may have multiple physical interfaces plugged;
   shutting down the logical port may not be needed when network redundancy is
   in place.


Use case
--------

The network elements of a cloud infrastucture is managed by two SDN
controllers: SDN controller A manages the virtual switches hosted on OpenStack
nodes, and SDN controller B manages Top of Rack (ToR) switches.

::


                       +---------------+
                       | Orchestrator/ |
           +-----------+  Fault Mmgmt  +---------+
           |           +---------------+         |
           |                                     |
           |                                     |
    +------+-------+                       +-----+----+
    |     SDN      |                       |  Neutron |
    | Controller B |                       +-----+----+
    +------+-------+                             |
           |                                     |
           |                                     |
           |                              +------+-------+
     +-----+------+                       |     SDN      |
     | ToR Switch |                       | Controller A |
     +------------+                       +------+-------+
                                                 |
                                                 |
                                                 |
                                            +----+----+
                                            | vSwitch |
                                            +---------+


SDN Controller B has network monitoring capabilities that enables northbound
users to be notified of topology changes, e.g. due to hardware failures or
cable pulled. In this use case, user is an orchestrator with fault management
capabilities that collects fault events from multiple data sources and updates
the status of affected cloud resources. The orchestrator can be a system
composed by various OpenStack projects such as Vitrage/Monasca for Root Cause
Analysis (RCA), Congress for cloud resource status updates and remediation
actions, and Aodh for event alarming.

One possible workflow is:

1. SDN Controller B detects ToR switch port down and reports to orchestrator;
2. Orchestrator finds affected cloud resources (e.g. (sub)set of Neutron ports
   attached to Nova instances) caused by the underlay data plane outage;
3. Orchestrator updates the data plane port status of those Neutron ports;
4. Neutron sends notification to the message bus;
5. Aodh and Congress consume Neutron notifications:

  a. Aodh sends an event alarm to affected Neutron port users;
  b. Congress triggers remediation actions (e.g. Congress policy actions) for
     switch-over to a standby instance.


A similar workflow was presented at the OpenStack Summit Barcelona keynote demo [6_].


Proposed Change
===============

A couple of possible approaches were proposed in [1_] (comment #3). This spec
proposes tackling the problem via a new extension API to the port resource.
The extension adds a new attribute ``data_plane_status`` to represent the
status of the underlay data plane. This attribute is to be managed by entities outside
of Neutron, while the 'status' attribute is managed by Neutron. Both status
attributes are independent from one another.

The field should be read-only by project users and read-write by any user with
a specific role.


Data Model Impact
-----------------

A new attribute will be added to the 'ports' table as a Neutron extension.

+-----------------+--------+----------------+--------+---------------+
|Attribute        |Type    |Access          |Default |Validation/    |
|Name             |        |                |Value   |Conversion     |
+=================+========+================+========+===============+
|data_plane_status|string  |R, project      |None    |None, "ACTIVE" |
|                 |        |RW, user w/role |        |"DOWN"         |
+-----------------+--------+----------------+--------+---------------+


REST API Impact
---------------

A new API extension to the ports resource is going to be introduced.

.. code-block:: python

  EXTENDED_ATTRIBUTES_2_0 = {
      'ports': {
          'data_plane_status': {'allow_post': False, 'allow_put': True,
                                'default': None, 'is_visible': True}

      },
  }


Examples
~~~~~~~~

Updating port data plane status to down:

.. code-block:: json

   PUT /v2.0/ports/<port-uuid>
   Accept: application/json
   {
       "port": {
           "data_plane_status": "DOWN"
       }
   }



Command Line Client Impact
--------------------------

::

  openstack port set [--data-plane-status <ACTIVE/DOWN>] <port>

Argument --data-plane-status is optional.


References
==========

.. [1] RFE: Port status update,
   https://bugs.launchpad.net/neutron/+bug/1598081

.. [2] RFE: ovs port status should the same as physnet
   https://bugs.launchpad.net/neutron/+bug/1575146

.. [3] Neutron Drivers meeting, July 21, 2016
   http://eavesdrop.openstack.org/meetings/neutron_drivers/2016/neutron_drivers.2016-07-21-22.00.html

.. [4] Neutron v2 API
   https://wiki.openstack.org/wiki/Neutron/APIv2-specification

.. [5] Neutron: resource status & admin state up
   https://docs.google.com/presentation/d/1-cex849lLsmRsZ302lqkwvXbexhe6cwuCZ2JSE8Rp2s

.. [6] Demo: OpenStack and OPNFV - Keeping Your Mobile Phone Calls Connected
   https://www.youtube.com/watch?v=Dvh8q5m9Ahk
