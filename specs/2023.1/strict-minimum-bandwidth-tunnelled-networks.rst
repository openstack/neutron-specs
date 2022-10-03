..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================================
Strict minimum bandwidth support for tunnelled networks
=======================================================

Launchpad bug: https://bugs.launchpad.net/neutron/+bug/1991965

The aim of the RFE is to improve the previous implemented RFE Strict minimum
bandwidth support [1]_.


Problem Description
===================

Since [1]_, Neutron has the ability to model the available bandwidth of the
physical interfaces (ingress and egress) connected to the physical networks
(flat and VLAN networks). This information is collected by Placement and
used to spawn virtual machines with network ports on compute nodes with
enough bandwidth. That guarantees a minimum port throughput.

This feature is currently implemented in three backends: ML2/SR-IOV, ML2/OVS
and ML2/OVN. The full list of related patches can be reviewed at [2]_.

However, most of the current deployments do not use physical backed networks
(flat or VLAN) but overlay networs (VXLAN and Geneve). Of course those
deployments use ML2/OVS and ML2/OVN; ML2/SR-IOV does not support tunnelled
networks. That leads to an existing gap in the currently implemented feature:
there is no way to model tunnelled networks.


Proposed Change
===============

This RFE proposes a way to model the available bandwidth for tunnelled
networks in compute nodes. This implementation will be limited to ML2/OVS
and ML2/OVN backends.

The referred backends handle the overlay traffic sending and receiving this
traffic from a host interface, that acts as a VTEP [3]_. This host interface
is identified by an IP address, known as "local_ip" in the ML2 plugin
configuration file [4]_.

This RFE proposes to use the same configuration options provided in [1]_,
adding a static string constant to define a resource provider in Placement
that could be configurable by the administrator. This string will be the
suffix of the resource provider name, same as Neutron uses the physical
bridge names to build their resource provider names. For example
"u20ovn:OVN Controller agent:rp_tunnelled", being "rp_tunnelled" the provided
string. The default value will be "rp_tunnelled". This configuration variable
will be stored in the ``[ml2]`` section and will be accesible from the Neutron
server and the OVS agent::

  [ml2]
  tunnelled_network_rp_name = custom_rp_name  # "rp_tunnelled" by default.


Example of ML2/OVS configuration section::

  [ovs]
  resource_provider_bandwidths = br0:EGRESS:INGRESS,rp_tunnelled:EGRESS:INGRESS


Example of ML2/OVN configuration, stored in the local database of the OVS
service::

  root@u20ovn:~# ovs-vsctl list open_vswitch .
  ...
  external_ids : {hostname=u20ovn,
                  ovn-bridge=br-int,
                  ovn-bridge-mappings="public:br-ex",
                  ovn-cms-options="enable-chassis-as-gw,
                                   resource_provider_bandwidths=br-ex:4001:5001;rp_tunnelled:4001:5001,
                                   resource_provider_inventory_defaults=allocation_ratio:1.0;min_unit:5,
                                   resource_provider_hypervisors=br-ex:u20ovn;rp_tunnelled:u20ovn",
                  ovn-encap-ip="192.168.10.40",
                  ovn-encap-type=geneve,
                  ovn-remote="tcp:192.168.10.40:6642",
                  system-id="a55c8d85-2071-4452-92cb-95d15c29bde7"}


Note that in ML2/OVN, it is mandatory to define the tunnelled resource provider assignation to
the host in the "resource_provider_hypervisors" list.

This new string constant cannot be used as a physical bridge name. To avoid
any possible clash, there will be a new check when parsing the physical
network bridge mappings.

A host with ML2/OVN backend with a physical network (mapped to the physical
bridge "br-ex") and a tunnelled network will report the following resource
provider tree::

  $ openstack resource provider list
  +--------------------------------------+------------------------------------------+------------+--------------------------------------+--------------------------------------+
  | uuid                                 | name                                     | generation | root_provider_uuid                   | parent_provider_uuid                 |
  +--------------------------------------+------------------------------------------+------------+--------------------------------------+--------------------------------------+
  | 8f0e060d-bf63-42a1-85e6-710c8b2fccc8 | u20ovn                                   |         10 | 8f0e060d-bf63-42a1-85e6-710c8b2fccc8 | None                                 |
  | cb101b60-527b-5264-8e7f-213c7b88e9e1 | u20ovn:OVN Controller agent              |          1 | 8f0e060d-bf63-42a1-85e6-710c8b2fccc8 | 8f0e060d-bf63-42a1-85e6-710c8b2fccc8 |
  | 521f53a6-c8c0-583c-98da-7a47f39ff887 | u20ovn:OVN Controller agent:br-ex        |         20 | 8f0e060d-bf63-42a1-85e6-710c8b2fccc8 | cb101b60-527b-5264-8e7f-213c7b88e9e1 |
  | dfdbf43f-f60b-577c-bae8-3dcea448c735 | u20ovn:OVN Controller agent:rp_tunnelled |          6 | 8f0e060d-bf63-42a1-85e6-710c8b2fccc8 | cb101b60-527b-5264-8e7f-213c7b88e9e1 |
  +--------------------------------------+------------------------------------------+------------+--------------------------------------+--------------------------------------+


A new static trait will be added to represent this resource provider:
"CUSTOM_NETWORK_TUNNEL_PROVIDER". This is what identify that this resource
provider is for tunnelled networks. E.g.::

  $ openstack resource provider trait list $rp_tun
  +----------------------------------+
  | name                             |
  +----------------------------------+
  | CUSTOM_VNIC_TYPE_NORMAL          |
  | CUSTOM_VNIC_TYPE_DIRECT          |
  | CUSTOM_VNIC_TYPE_DIRECT_PHYSICAL |
  | CUSTOM_VNIC_TYPE_MACVTAP         |
  | CUSTOM_VNIC_TYPE_VDPA            |
  | CUSTOM_VNIC_TYPE_REMOTE_MANAGED  |
  | CUSTOM_VNIC_TYPE_BAREMETAL       |
  | CUSTOM_NETWORK_TUNNEL_PROVIDER   |
  +----------------------------------+


This is an example of a port resource request, sent to Nova when creating a
virtual machine. The port has a minimum bandwidth rule of 500 kbps, egress
direction::

  port['resource_request'] = {
      'request_groups': [
          {'id': 'c51c6f07-8e01-548c-9756-d5e54a780bb6',
           'required': ['CUSTOM_NETWORK_TUNNEL_PROVIDER', 'CUSTOM_VNIC_TYPE_NORMAL'],
           'resources': {'NET_BW_EGR_KILOBIT_PER_SEC': 500}
          }
      ],
      'same_subtree': ['c51c6f07-8e01-548c-9756-d5e54a780bb6']
  }


.. note::

   This spec is not considering the case of shared resource providers. For
   example when the same interface is shared between a VLAN/flat network and
   and overlay network. What this spec is proposing is to provide the
   scheduling functionality to ports in overlay networks. In case of having
   shared resources, the administrator will need to split bandwidth assignation
   between resource providers. Currently Placement API nor Neutron cannot
   provide a way to model a shared resource.


REST API Impact
---------------

This RFE does not introduce any API change.


Data Model Impact
-----------------

This RFE does not introduce any model change.


Security Impact
---------------

None.


Performance Impact
------------------

None.


Other Impact
------------

Currently there is no support for minimum bandwidth QoS rules for tunnelled
networks, neither in Placement nor in the ML2 backend (OVS, OVN). However,
it is possible to have ports with those type of QoS rules (maybe inherited
from the network QoS policy). With this feature, the minimum bandwidth QoS
rules won't be discarded, like now, when the port resource request is built
(that is the Placement blob to request a specific bandwidth in a specific
network).

A new check will be added to inform about those ports located on
tunnelled networks with minimum bandwidth QoS rules. The output of this check
will be a log with the list of ports, their networks and QoS policies. This
spec is considering the current Neutron implementation:

* ML2/OVS rejects the assignation of a QoS policy with minimum bandwidth rules
  and prevents from binding a port with them.
* The ML2/OVN mechanism driver implemented the minimum bandwidth rule support
  recently and does not prevent this scenario. However this functionality was
  implemented in Zed release; it is unlikely that many deployments are in this
  state (with ports located in overlay networks with QoS policies and minimum
  bandwidth rules).

This spec does not consider the rebuild of the current allocations. Any port
already present in a host that creates a new resource provider for tunnelled
networks, won't be allocated. Once there is standard a procedure to perform
this action, a new spec/bug will be created to track this improvement, but
this is out of scope in this spec.

Part of this RFE will be to document the alternatives the user has to, in
case of having a port with minimum bandwidth rules before enabling this
feature, create the needed allocations:

* Live migrate the VM with the port. That will trigger the Placement
  scheduling and the allocation creation.
* Detach and attach again the port to the VM. That will have the same
  effect. This functionality was added to Nova in Wallaby.


Implementation
==============

Assignee(s)
-----------

Primary assignees:
  Rodolfo Alonso Hernandez <ralonsoh@redhat.com> (IRC: ralonsoh)

Work Items
----------

* ML2 plugin update.
* Migration script to log those existing ports with minimum bandwidth rules
  in tunnelled networks.
* Documentation, including the methods to re-created the allocations for pre-
  created ports.
* Tests and CI related changes.


Testing
=======

* Unit/functional tests.
* Fullstack tests: increase the current fullstack tests coverage to check
  this new feature.
* Tempest tests: create a VM with a minimum bandwidth port, update the
  QoS policy and minimum bandwidth rule limits, unset the QoS policy,
  migrate the VM.


Documentation Impact
====================

User Documentation
------------------

Ammend the "Strict minimum bandwidth support" [1]_ documentation, adding this
new improvement.


References
==========

.. [1] [RFE] Strict minimum bandwidth support (egress)
       https://bugs.launchpad.net/neutron/+bug/1578989
.. [2] https://review.opendev.org/q/%25231578989
.. [3] https://networklessons.com/cisco/ccnp-encor-350-401/introduction-to-virtual-extensible-lan-vxlan
.. [4] https://github.com/openstack/neutron/blob/599c81767ea7aa3bde7a64ff57b20f34fb314548/neutron/conf/plugins/ml2/drivers/ovs_conf.py#L44-L50
