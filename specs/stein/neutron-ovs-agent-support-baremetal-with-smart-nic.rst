..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================================
Neutron OVS Agent Support for Baremetal with Smart NIC
======================================================

https://bugs.launchpad.net/neutron/+bug/1785608

This spec describes proposed changes to the Neutron OVS mechanism driver and
the Neutron OVS agent to enable a generic, vendor-agnostic, baremetal
networking service running on smart NICs, enabling baremetal networking
with feature parity to the virtualization use-case.


Problem Description
===================

While Ironic today supports Neutron provisioned network connectivity for
baremetal servers through an ML2 mechanism driver, the existing support
is based largely on configuration of TORs through vendor-specific mechanism
drivers, with limited capabilities.


Proposed Change
===============

There is a wide range of smart/intelligent NICs emerging on the market.
These NICs generally incorporate one or more general purpose CPU cores along
with data-plane packet processing acceleration, and can efficiently run
virtual switches such as OVS, while maintaining the existing interfaces to the
SDN layer.

The proposal is to extend the Neutron OVS mechanism driver and Neutron OVS Agent
to bind the Neutron port for the baremetal host with smart NIC. This will allow
the Neutron OVS Agent to configure the pipeline of the OVS running on the
smart NIC and leverage the pipeline features such as: VXLAN, Security Groups and
ARP Responder.

This spec is complementary to the Ironic spec [1]_.

In this proposal, we address two use-cases:

#. Neutron OVS L2 agent runs locally on the smart NIC.
#. Neutron OVS L2 agent(s) run remotely and manages the OVS bridges for all
   the baremetal smart NICs.

Example of smart NIC model::

  +---------------------+
  |      baremetal      |
  | +-----------------+ |
  | |  OS Server    | | |
  | |               | | |
  | |      +A       | | |
  | +------|--------+ | |
  |        |          | |
  | +------|--------+ | |
  | |  OS SmartNIC  | | |
  | |    +-+B-+     | | |
  | |    |OVS |     | | |
  | |    +-+C-+     | | |
  | +------|--------+ | |
  +--------|------------+
           |

  A - port on the baremetal host.
  B - port that represents the baremetal port in the smart NIC.
  C - port that represents to the physical port in the smart NIC.

- Ironic creation of Neutron Port:

  #. Create Neutron port with new vnic_type called `smart-nic`.
  #. Add local_link_information with the following attributes:

    #. smart NIC hostname - the hostname of server/smart NIC where the Neutron
       OVS agent is running. (required)
    #. smart NIC port id - the port name that needs to be plugged to the
       integration bridge. B in the diagram above(required)
    #. smart NIC SSH public key - ssh public key of the smart NIC
       (required only for remote)
    #. smart NIC OVSDB SSL certificate - OVSDB SSL of the OVS in smart NIC
       (required only remote)

- Neutron OVS ML2 Mechanism Driver:

  The OVS ML2 will allow binding the `smart-nic` vnic_type. The rationale
  for creating new vnic_type and not using the barmetal one is that there is a
  wide range of mechanism drivers that use hierarchical port binding for
  configuring TOR switches and we want to allow this to work with smart NICs.
  for example mechanism_drivers=cisco_nexus,openvswitch

  The OVS ML2 mechanism driver will determine if the Neutron OVS Agent runs
  locally or remotely based on smart NIC configuration passed from ironic as
  described above.

  In case the Neutron OVS L2 agent runs locally on the smart NIC the OVS
  mechanism driver will locate the Neutron OVS agent by the smart
  NIC hostname attribute. For the remote case the changes are captured
  in this neutron spec [2]_.

- Neutron OVS Agent:

  Extend the port_update rpc method as following:

  .. code-block:: python

      def port_update(self, context, **kwargs):
        port = kwargs.get('port')
        # get the port data from cache
        port_data = self.plugin_rpc.remote_resource_cache. \
                get_resource_by_id(resources.PORT, port['id'])
        # if smart-nic port add it to updated_smart_nic_ports with
        # the required information for adding the port to OVSDB
        port_binding = port_data['bindings']
        if port_binding['vnic_type'] == 'smart-nic':
                ifname = smart_nic_port_data['port_binding']['profile'] \
                    ['local_link_information'][0]['port_id']
                mac = port_data['mac_address']
                node_uuid = port_data['device_id']
                port_binding['vif_type'] = vif_type
                self.updated_smart_nic_ports.append({
                                'mac': mac,
                                'node_uuid': node_uuid,
                                'iface_id': port['id'],
                                'iface_name': ifname,
                                'vif_type': vif_type})
        self.updated_ports.add(port['id'])

  When Neutron processes the ports the Neuton OVS agent will add the
  smart NIC port(s) to the OVSDB by ovs plugin in os-vif.

  Because RPC is not reliable we need to extend the full sync to do the following:

  #. when sync is True we will retrieve all the smart-nic ports for this agent.
     This requires to add another RPC call.
  #. We will compare the retrieved smart-nic ports for the Neutron server to the
     existing smart-nic port on the integration bridge.
  #. if the smart-nic port is only on the Neutron server we will add it to
     the added list in the port_info.
  #. if the smart-nic port is only on the integration bridge we will add it
     to the removed list in the port_info.


References
==========

.. [1] https://review.openstack.org/#/c/582767/

.. [2] https://review.openstack.org/#/c/595402/
