..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================
Multiple routed provider segment per host
=========================================

https://bugs.launchpad.net/neutron/+bug/1764738

The proposed spec is to extend the current feature routed provider
networks [0]_ to allow provisioning more than one segment per physical
network.

There is currently a limitation for a compute node to only have an
interface on one segment in a multisegment network. Operators that
want to extend IP range will have to provision new networks which
degrade user experience, or increase the broadcast domain which does
not scale.


Problem Description
===================

As an operator I want to extend IP pool without creating multiple
networks. I also want to limit the brodcast domain.


Proposed Change
===============

The change is limited to ``OVS agent``, it will be still possible to
implement it for ``linux-bridge``. That is said, this implementation
is out of the scope of the proposed spec.

The main purpose is to add the support for an interface of a compute
node to be attached to more than one segment of a given network.

This implies changes on different components. The DHCP agent currently
only supports one interface per network. It will be necessary to
extend it to support more domains per network. The VLAN manager that
maintains details of VLAN ports for OVS by using network id will have
to be updated to use segment id instead. The agent will have to be
updated to support the feature and finally the limitation that rejects
such case will have to be removed.


VLAN Manager
------------

The current implementation for ``VLAN manager`` that maintains
internal VLAN mapping to external network segmentation ids is using
the ``network ids`` as principal identifier [1]_.  This will have to
be updated to use a combination of ``network ids``, ``segment ids``.
This should not implies upgrade impact as they are in-memory and
recalculated after restarts.


DHCP Agent
----------

Using multiple segments per network implies more than one network
domain. The current DHCP agent supports one domain per network
[2]_.

A DHCP port is plugged to the internal bridge using the local VLAN
provided by the VLAN manager for a given network. This means that the
``DHPC Process`` running can address reauest for one domain. To reflect
the current desire of multiple domains per network, a DHCP port should
be plugged to br-int per combination of ``network id / segmentation
id``.

The proposed implementation is to avoid changing ``DHCP Process``
logic to make it supporting more than one interface per VLAN as it
will add complexity and may imply upgrade impact.  Instead, it has
been considered to keep the well trusted logic of the ``DHCP Process``
to use one per segmentation id. One may in future consider to revisit
this implementation.

The benefit of having one ``DHCP Process`` running per local VLAN will
limit the changes and avoid upgrade impact. For a deployment already
running routed network provider, two ``DHCP process`` will be
running. One considered as legacy running under ``/net-id/pid``, and
an other running at ``/seg-id/net-id/pid``. It will involve the
operator to remove the legacy port using opentack API.

Limitations
-----------

 * The proposed spec only implies OVS support but it should be
   possible to provide the same kind of support for
   ``linux-bridge``. A change has been proposed with [3]_ with related
   spec [5]_ It implies an upgrade impact which can be mitigated like
   for this proposed OVS implementation. Also that the
   ``linux-bridge`` implementation can take benefit of the change in
   DHCP agent which is missed in the proposed change for
   ``linux-bridge``.
 * For a given fixed ip, during scheduling Nova will not be able
   creating a port a the segment related to the subnet of the fixed
   ip. It’s an already known limitation. Bug 1979959 [9]_ has been
   reported to address the issue.
 * A regression has been noted since the bug 1952730 [8]_, with routed
   provider segments created are not taken into account. Bug 1979958
   [7]_.

Notes
-----

 * In its process Nova needs to retrieve network details [4]_. In case
   of a given network with multiple segments the process will use the
   first segment that it discovers with a ``physcal_network``. This is
   a design issue that is already known. The process change is not
   expected to fix it.

 * To have Nova schedule instances on hosts based on segments where
   ports are attached to. Aggregates are created per segment [6]_. The
   process is using segment ids as names. It is still expected to
   continue creating aggregates with the new segment created for a
   given network. This should not bring any issue.

Implementation
==============

Primary Assignees
-----------------

 * Sahid Orentino Ferdjaoui

References
==========

.. [0] https://docs.openstack.org/neutron/latest/admin/config-routed-networks.html
.. [1] https://github.com/openstack/neutron/blob/master/neutron/plugins/ml2/drivers/openvswitch/agent/vlanmanager.py#L87
.. [2] https://github.com/openstack/neutron/blob/master/neutron/agent/linux/dhcp.py#L286
.. [3] https://review.opendev.org/c/openstack/neutron/+/623115
.. [4] https://github.com/openstack/nova/blob/master/nova/network/neutron.py#L2141
.. [5] https://review.opendev.org/c/openstack/neutron-specs/+/657170
.. [6] https://opendev.org/openstack/neutron/src/branch/master/neutron/services/segments/plugin.py#L180
.. [7] https://bugs.launchpad.net/neutron/+bug/1979958
.. [8] https://bugs.launchpad.net/neutron/+bug/1952730
.. [9] https://bugs.launchpad.net/nova/+bug/1979959
