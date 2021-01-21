..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================
L3 router support ECMP
======================

Blueprint:
https://blueprints.launchpad.net/neutron/+spec/support-for-ecmp

Launchpad Bug:
https://bugs.launchpad.net/neutron/+bug/1880532

ECMP is a kind of routing technology which allows traffic to reach the
same destination via multiple different links. Neutron does not need to
calculate the equivalent route path, but leave that part of the work to
those applications using ECMP API. Neutron just receives those parameters
and configures routers. Since we have "ip route" command provided by the
iproute2 utility in Linux, Neutron can simply address ECMP by using pyroute2
and adding route entry into Neutron router namespace.

This feature is currently designed to support Octavia's multi-active scheme,
allowing LoadBalancer in Octavia to have multiple amphoras at the same time.
By configuring the ECMP route in the router, multiple amphoras can have a
virtual IP at the same time to serve a set of functions that require high
concurrency support.

.. _P2:

.. note::

   Items marked with [`P2`_] refer to lower priority features
   to be designed / implemented only after initial release.

[`P2`_] Currently the equal cost route is a simple 5 tuple, that means if
we have one <nexthop> unreachable and remove it from ECMP routes, all
connections get redistributed. To avoid this, we intend to use a consistent
hashing instead of the original scheme. This scheme which can support
consistent hashing is based on hmark which was added in iptables-1.4.15 or
later. See the history file of the iptables on [1]_.

Then this spec describes how to implement ECMP in Neutron.


Problem Description
===================

Octavia has proposed an active-active load balancing design on [2]_.

Topology Description
--------------------

::




                                                             Tenant Backend
                                +----------------+              Network
                                |                |                 +
        Internet+-------------->+    router/gw   +----------------->
                                |                |    ECMP         |
                                +----------------+                 |
                                                                   |
        Management                                                 |
         Network                                                   |
            +                                                      |
            |                                                      |         +----------+
            |               +-----------------------+              |         |  Tenant  |
            |          +----+                  +---------+         <---------+Service(1)|
            |          |MGMT|  loadbalancer(1) | VIP|Back|         |         |          |
            <----------+ IP |                  |    | IP +--------->         +----------+
            |          +---------------------------------+         |
            |                                          |           |         +----------+
            |                                          |           |         |  Tenant  |
            |                                          |  ICMP     <---------+Service(2)|
            |                                          | DETECT    |         |          |
            |                                          |           |         +----------+
            |                                          |           |
            |               +-----------------------+  v           |         +----------+
            |          +----+                  +---------+         |         |  Tenant  |
            |          |MGMT|  loadbalancer(2) | VIP|Back|         <---------+service(3)|
            <----------+ IP |                  |    | IP +--------->         |          |
            |          +---------------------------------+         |         +----------+
            |                                          |           |
            |                                          |           |
            |         +-------------+                  |           |           ● ● ●
            |         |Octavia Lbaas|                  |           |
            <---------+ Controller  |   ● ● ●          |  ICMP     |
            |         +-------------+                  | DETECT    |         +----------+
            |                                          |           |         |  Tenant  |
            |                                          |           <---------+Service(M)|
            |                                          |           |         |          |
            |               +-----------------------+  v           |         +----------+
            |          +----+                  +---------+         |
            |          |MGMT|   loadbalancer(n)| VIP|Back|         |
            <----------+ IP |                  |    | IP +--------->
            |          +---------------------------------+         |
            +                                                      +

This program proposed such a scheme:

* Multiple load balancing servers in a vip-subnet, sharing one virtual IP
  and one or more back end pools to response clients' request, and each
  loadbalancer has its own IP address.

* Clients send requests to VIP, then the router distributes every single
  request to a load balancing server which has the correct VIP configured
  on it.

* Finally, the load balancing server distributes the request to a back end.
  The loadbalancers and tenant service vm can be in the same subnet or
  different networks.

In such a situation, Octavia needs the router to support ECMP for distributing
requests. So Octavia can send a request to Neutron for creating an ECMP route,
then Neutron L3 agent executes command in the Neutron router's namespace to
create an ECMP entry in it, using VIP as the destination IP of the route's
entry, and several load balancers' IP as nexthop IP. So those requests having
VIP as their destinations can be distributed to each loadbalancer.

The whole process implements two levels of load balancing, i.e. load balancing
between multiple loadbalancers and load balancing between the backend
real servers

[`P2`_] Based on current public cloud operator implementations in production
environments, tenants usually only see IPs in the same network, so
considering the same broadcast domain, the router needs to enable proxy
ARP on the corresponding interface.(Users need to disable the proxy ARP
capability of vms in nexthops by themselves)

User Workflow
-------------

Generally, users can use the ECMP function for their own purposes.
For putting an ECMP entry into the router namespace,
user can set routes with same destination by using command::

  openstack router add route \
  --route destination=20.0.20.0/24,gateway=12.0.0.11 \
  --route destination=20.0.20.0/24,gateway=12.0.0.12 router-ecmp

And withdraw the ECMP entry with::

  openstack router add route \
  --route destination=20.0.20.0/24,gateway=12.0.0.11 \
  --route destination=20.0.20.0/24,gateway=12.0.0.12 router-ecmp

For more information about router related OSC, please read [3]_.

An integrated sequence diagram of the Octavia's use case is here:

::

  +------+      +--------+     +-------+   +--------+ +-------+ +------------+
  |client|      |Octavia |     |Neutron|   |LB Node | |qrouter| |service pool|
  +------+      +---+----+     +---+---+   +---+----+ +---+---+ +------+-----+
    |create LB      |              |           |          |            |
    +-------------> | create ecmp  |           |          |            |
    |service        +-------------->           |          |            |
    |               | LB server boot           |          |            |
    |               +--------------+---------->+          |            |
    |               |              | set ecmp route       |            |
    |               | ecmp done    +-----------+--------->+            |
    |               +<-------------|           |          |            |
    |               | LB server boot done      |          |            |
    |LB service done+<-------------+-----------+          |            |
    +<--------------+              |           |          |            |
    |               |              |           |          |            |
    |               |              |           |          |            |
    |sending request|              |           |          |            |
    +---------------------------------------------------->|            |
    |               |              |           |  pick a LB node       |
    |               |              |           +<---------|            |
    |               |              |           | pick a service node   |
    |               |              |           +---------------------->+
    |               |              |           |          |response    |
    |               |              |           +<----------------------+
    |               |  response    |           |          |            |
    +<-----------------------------------------+          |            |
    |               |              |           |          |            |
    |               |              |           |          |            |
    v               v              +           v          v            v


Suppose a user has a set of services that require a multi-active
load-balancing scheme, so the user send a request to Octavia to create a
loadbalancer, specifying topology as multi-active. And post a vip-subnet
to Octavia to assign an IP or directly post a virtual port, which is
defined by Octavia, and then users need to submit parameters such as
pool, member, listener, etc., but the latter are irrelevant to Neutron,
you can find them in Octavia document.

While Octavia is creating a loadbalancer, it will also send an `update_router`
request or an `add_extraroutes` request to Neutron, post severval `routes`
entries with same `destination` param, and load balancers' IPs as
`nexthop` param.

Neutron receives the request from Octavia, determines whether to add an ECMP
route by calculating whether there are multiple routes with the same
destination address, making sure the router will distribute those packets
with vip as their destination.

Those ECMP routes will be removed when user drops the multi-active
loadbalancer, and it could be modified when adding or removing a load balancing
node.


Data flow
---------

* [`P2`_] (If on a same network, use ARP proxy) A client requests mac
  address of the VIP and accesses the service based on this mac address.
  the router will use gateway MAC address to respond.

* The client's datagram will be transmitted to the router first.

* The router gateway checks ECMP routing entries then forwards the
  client's packets to the load balancers.

* Load balancer accepts connections from clients, receives traffic, then
  distributes it to the back-end server pool.

* The reply traffic from the back-end server pool go through load balancers
  and then comes to the router (directly comes back to intranet clients if on
  a same network), these packets are eventually forwarded back by the router.

Proposed Change
===============

Overview
--------

In Server Side
~~~~~~~~~~~~~~

* There are no changes that have to be made in server side.

In Agent Side
~~~~~~~~~~~~~

Modify the logic of processing router_update event in L3 agent to
support adding ECMP routes in routers.
The `routes_updated` function in RouterInfo will behave as below:

* When more than one route is found to have the same destination, L3
  agent should execute a pyroute2 code, which looks like

::

  ip.route('replace', dst='<destination_ip>',multipath=[{"gateway":
  "<nexthop1>"},{"gateway":"<nexthop2>"}])

* Then there will be an ip route entry in the namespace, which looks like

::

  <vip> proto static
      nexthop via <nexthop_ip1> dev qr-xxxxxxxx-nn weight 1
      nexthop via <nexthop_ip2> dev qr-xxxxxxxx-nn weight 1

Then router will randomly pick a <nexthop_ip> and fill its mac address into
the package's dst_mac address when it wants to get to the <destination_ip>.

[`p2`_]For keeping connection while removing a load balancing node, use
iptables instead of simply a ip route entry.

- Use `HMARK` to mark flows in mangle table, the `fwmark` values
  determined by the source address.
- Distribute flows to different tables by `fwmark` values.
- There is a mapping between the `fwmark` values and the table values
- For each table, give it a default nexthop ip.
- Modify the mapping between `fwmark` values and table values
  when a `nexthop` is unreachable.

[`p2`_]In order to let traffic from the same network to pass through the
router, L3 agent will also let router to use Proxy ARP by setting command::

  sysctl -w net.ipv4.conf.<NIC_1>.proxy_arp_pvlan=1

* <NIC_1> is the name of the router interface to which the destination
  subnet is connected. For example, router `R1` is connected to a
  subnet `sub-1` whose cidr is `10.10.10.0/24`, so there will be a
  virtual network interface device `qr-abcdefgh` in the router related
  namespace as the gateway for the subnet `sub-1`, then add an
  ECMP route with a destination like `10.10.10.5/32` which is in the
  scope of the subnet `sub-1`, at this point, the above command
  will be executed and <NIC_1> will be `qr-abcdefgh`.

* For making the ARP proxy optional, add an config option in L3Agent.ini::

    [ECMP]

    router_interface_arp_proxy = True


Data Model Impact
-----------------

None

REST API Impact
---------------


Following REST APIs wil be affected::

  PUT /v2.0/routers/<router_id>/add_extraroutes

  PUT /v2.0/routers/<router_id>/remove_extraroutes

  PUT /v2.0/routers/<router_id>

The above three APIs are the current methods used to add/remove custom
routes. See the usage of `extraroutes` on [4]_. (The third API
`PUT /v2.0/routers/<router_id>` is not recommended for adding routes)

Before the ECMP routing Implementation, when L3 agent receive several route
entries with same destination and different nexthops, it will only keep one
entry of them, or replace the existing route with a new one. But now after
these changes, there will be an ECMP route in the router. So you can add an
ECMP route entry like this:

::

  PUT /v2.0/routers/{router_id}/add_extraroutes

  { "router":
    { "routes":
      [ { "destination": "192.168.1.6/32",
          "nexthop": "192.168.1.88" },
        { "destination": "192.168.1.6/32",
          "nexthop": "192.168.1.99" }
        ...
      ]
    }
  }

Then you can find the ECMP route in router related namespace:

::

  #ip route

  192.168.1.6/32 proto static
    nexthop via 192.168.1.88 dev qr-9adb238b-c2 weight 1
    nexthop via 192.168.1.99 dev qr-9adb238b-c2 weight 1

To make this behavior change discoverable, a shim extension called
'ecmp_routes' will be added.
[`p2`_]To make ARP proxy behavior discoverable, a shim extension called
'ecmp_arp' will be added, it will be removed dynamically when related option
`router_interface_arp_proxy` in config file is `False`.


Implementation
==============

Assignee(s)
-----------

* XiaoYu Zhu

Work Items
----------

* L3 Agent Update
* Tests
* Documentation


Testing
=======

Tempest Tests
-------------
* Tempest tests

Functional Tests
----------------
* New tests need to be written


Documentation Impact
====================

User Documentation
------------------
* User documentation
* API reference

Developer Documentation
-----------------------
* Needs devref documentation


References
==========

.. [1] http://netfilter.org/projects/iptables/files/changes-iptables-1.4.15.txt

.. [2] https://review.opendev.org/723864

.. [3] https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/router.html

.. [4] https://specs.openstack.org/openstack/neutron-specs/specs/train/improve-extraroute-api.html

