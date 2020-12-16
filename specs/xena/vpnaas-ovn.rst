..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================
VPNaaS for OVN Networking
=========================

This specification covers the support for VPNaaS with OVN networking by adding
a stand-alone VPN agent. This new agent will create a namespace on the node,
connect it with the OVN distributed logical router and run the Swan process in
the namespace.

Problem Description
===================

The existing VPNaaS service plugin only supports the reference Neutron software
routers, such as neutron L3 router using the L3 agent. This approach does not
work if there is no L3 agent. For OVN a different solution is required.

Proposed Change
===============

Add a new stand-alone VPN agent to support OVN+VPN. Add OVN-specific service
and device drivers that support this new VPN agent. This will have no impact
on the existing VPN solution, the existing L3 agent and its VPN extension will
still work.

Changes on neutron server
-------------------------

VPN service driver and Agent scheduler
++++++++++++++++++++++++++++++++++++++

The VPN service driver has different implementations for different VPNaaS
solutions. The existing implementation relies on the Neutron L3 router
scheduler to decide where the VPN service process is hosted and the VPN device
driver is part of the L3 agent. In OVN the L3 router scheduler and L3 agents
are not used anymore, so a replacement is required.

For the newly-introduced agent type "VPN agent"
we also add a corresponding VPN agent scheduler. Using the OVN-related
scheduler for the router gateway port was ruled out because the VPN agent
scheduler should also keep track whether the VPN agent(s) are still alive.

The new VPN agent scheduler will work on a per-router basis to have all IPSec
site connections of a router processed by the same agent.

VPN gateway and transit network
+++++++++++++++++++++++++++++++

In the existing implementation for ML2/OVS the VPNaaS shares the router gateway
IP address with router SNAT, but for OVN the gateway public IP address (SNAT)
can't be shared with the VPN because SNAT is not in the namespace context.
A new public IP address is needed for the VPNaaS namespace, so a VPN gateway
port is created in the external network of the router. This address will be
visible as the external_v4_ip/external_v6_ip of the VPN service.

To connect the router with the namespace a "Transit network" (169.254.0.0/30)
is created and attached to the router. It is used to route traffic between
virtual machine network(s) and the VPN process namespace. The respective routes
added to the router are (per peer CIDR):

- destination CIDR: peer CIDR
- next hop: IP address of a port in the transit network (169.254.0.2)

The new ports and the transit network and subnet are managed using the API of
the Neutron core plugin and OVN L3 router service plugin, not "under the hood"
with OVN functions, to avoid inconsistencies between the Neutron database and
OVN Northbound. This makes sure that scripts that sync Neutron DB and OVN
Northbound (like neutron-ovn-db-sync-util) don't delete lswitches that
were created for the VPNaaS transit network but have no corresponding Neutron
DB entries.

The IDs of the newly added ports, transit network and subnet are stored in a
new database table "vpn_ext_gws" (for VPN external gateways).
The gateway port and transit network are created automatically when the first
VPN service of a router is created and are deleted after the last VPN service
of the router is removed.

Naming of Neutron objects
+++++++++++++++++++++++++

- External gateway port for VPN: "vpn-gw-{router_id}" (with device owner
  "network:vpn_router_gateway", device ID is the router ID)

- VPN transit network: "vpn-transit-network-{router_id}"

- VPN transit subnet: "vpn-transit-subnet-{router_id}"

- Port in the VPN transit network, to be plugged by the agent in the VPN
  namespace: "vpn-ns-{router_id}" (with device owner "network:vpn_namespace",
  device ID is the transit subnet ID)

Changes on VPN agent
--------------------

For OVN the VPN functionality is implemented in a new dedicated VPN agent using
new OVN specific device drivers.

Configuration synchronization
+++++++++++++++++++++++++++++

The existing VPNaaS implementation uses rabbitmq RPCs to synchronize the VPN
configuration between the database and the IKE daemon (strongswan etc)
controlled by the agent.

Ideally a VPNaaS solution for OVN would avoid rabbitmq. But currently there
is no support for configuring VPNs in OVN (via north/southbound databases).
While it is possible to configure IPSec for tunnel ports, it's not possible
to set up and configure IPSec site connections to external peers.

To still be able to offer VPNaaS some other means than configuration via
OVN north/southbound tables is necessary. Possible solutions could be:

1. Use rabbitmq RPCs as in the existing VPNaaS plugin.
2. Instead of rabbitmq, implement REST APIs on both ends (neutron server and
   VPN agent). This includes API endpoints that let the agent know about
   configuration changes (server -> agent), that allow agents to fetch the
   current configuration (agent -> server) and that allow agents to report
   state changes (agent -> server)

The least complex implementation approach is (1) to rely on existing code
even if it is still using rabbitmq instead of rewriting the server/agent
communication with new REST APIs.

VPN namespace management
++++++++++++++++++++++++

The agent creates a VPN namespace and plugs two ports:

- the external gateway port
- a port in the transit network bound to 169.254.0.2

There will be one VPN namespace per router and it's scheduled on exactly one
VPN agent. The VPN agent shall make sure that only those VPN namespaces exist
on the node which are scheduled there. I.e. when the agent starts it will
synchronize with the controller and potentially delete superfluous VPN
namespaces and/or create missing ones.

VPN namespaces are created by the VPN agent / device driver when the first
IPSec site connection of a router is created and are deleted if all IPSec site
connections of the router were deleted.

Routes are configured in the namespace to route incoming traffic to the local
subnets (destination: local CIDR, next hop: router port of the transit network,
i.e. 169.254.0.1). These routes are updated whenever the IPSec site connection
configuration changes.

The agent will

1. create a namespace called "qvpn-{router_id}"
2. create two interfaces in the namespace:

   - vgxxxxxxxx-xxx, where xxxxxxxx-xxx is the prefix of the port UUID of the
     VPN external gateway port (IP address in the external network)
   - vrxxxxxxxx-xxx, where xxxxxxxx-xxx is the prefix of the port UUID of the
     VPN transit network port (169.254.0.2)

3. plug the two interfaces
4. add routes

Liveness of the VPN agent
+++++++++++++++++++++++++

There are two ways how liveness checks could be implemented:

1. Traditional way using rabbitmq. The VPN agent periodically reports its state
   ("report_state" RPC). The server side plugin utilizes the
   AgentSchedulerDbMixin functionality to keep track of alive agents and will
   potentially reschedule VPN services if an agent is down.
2. Liveness check using OVN similar to OVN Metadata agent. The server sets
   a new value of the "neutron:liveness_check_at" external_id in NB_Global,
   the agent monitors SB_Global and will set an external_id in its chassis row
   (external_id "neutron:ovn-vpnagent-sb-cfg")

The second approach is more in line with the management of agents in the Neutron
OVN mech driver.

VPN plugin configuration for OVN
--------------------------------

The OVN VPN plugin is an extension of the existing one in order to add the VPN
agent scheduler and status checks.

The service plugin to be added to neutron.conf is
neutron_vpnaas.services.vpn.ovn_plugin.VPNOVNDriverPlugin

For OVN there is a new dedicated service driver, so the service_provider
setting in neutron_vpnaas.conf will be:
VPN:openswan:neutron_vpnaas.services.vpn.service_drivers.ovn_ipsec.IPsecOvnVPNDriver:default

There is a new agent type "VPN agent"

- The agent binary is neutron-vpn-agent
- The agent configuration file is /etc/neutron/vpn_agent.ini
- The agent uses the VPN device driver configured in vpnagent/vpn_device_driver.
  An example device driver entry is
  vpn_device_driver = neutron_vpnaas.services.vpn.device_drivers.ovn_ipsec.OvnStrongSwanDriver

Database impact
---------------

Two new tables are added to the neutron database:

- vpn_ext_gws: keeps the IDs of the additional network items needed for VPN
  services of a router (gateway port, transit network, subnet, port)
- routervpnagentbindings: keeps the VPN agent ID per router

REST API and CLI impact
-----------------------

There are no changes to the REST API or CLI of VPNaaS or Neutron.

The external IP address of the VPN service is visible in its
external_v4_ip/external_v6_ip and will be different than the one of
the router.

References
==========

* https://bugs.launchpad.net/neutron/+bug/1905391
