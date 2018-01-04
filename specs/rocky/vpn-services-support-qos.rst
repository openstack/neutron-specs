..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================
VPN Services Support QoS
========================

https://bugs.launchpad.net/neutron/+bug/1727578

Currently, there is no way to set VPN services' bandwidth. This specification
proposed a way to set the VPN services' bandwidth limit.


Problem Description
===================

Currently, Neutron VPNaaS provides site to site VPN services, but its bandwidth
consumption is not regulated, as the VPN tunnel will cost the bandwidth from
the outside public bandwidth provided by the ISP or other organizations. That
means it is not free. The OpenStack provider or users should pay for the
limited bandwidth. So it is necessary for saving the resources.

Currently, physical device configurations outside OpenStack environment cannot
be modified by OpenStack, but we can shape the outgoing traffic. VPN services
provide the security guarantee, but it shouldn't abuse the bandwidth of
underlay network that it will affects other traffic.

Use Case
--------

Consider that an OpenStack public cloud has been deployed in a real data
center. A cloud enterprise user has a requirement in which one of the
applications should connect to its private network 20.0.0.0/24, but for other
traffic flows, still use the original network functionality provided by
OpenStack. The network topology diagram is shown below::

                                                                                                             +
                           +--------------------+   +                                                        |DataCenter
                           |VM1                 |   |                                                        |external network
                           |nic IP: 10.0.0.11   |   |                 +-------------------------------+      |        +----+
                           |default via 10.0.0.1+---+                 |Default Router                 |      |        |    |
                           |                    |   +-----------------+Interface: 10.0.0.1            +------+        |    |
                           +--------------------+   |                 |gw: 172.24.4.11                |      |        |    |
    OpenStack                                       |                 |route: 20.0.0.0/24 via 10.0.0.5|      |        |    |
    private                                .        |                 |                               |      |        |    |
    network                                .        |                 +-------------------------------+      |        |    |
                                                    |                                                        |        |    |
    subnet: 10.0.0.0/24                             |                                                        |        |    |
                           +--------------------+   |                                                        |        |    |
                           |VM2                 |   |                                                        +--------+    |Internet
                           |nic IP: 10.0.0.12   |   |                                                        |        |    |
                           |default via 10.0.0.1+---+                                                        |        |    |
                           |                    |   |                                                        |        |    |
                           +--------------------+   |                                                        |        |    |
                                                    |                                                        |        |    |
                                           .        |                 +--------------------------------+     |        |    |       +-------------------------------+
                                           .        |                 |VPN Router                      |     |        |    |       |Vendor GW Router               |
                                                    |                 |Interface: 10.0.0.5             |     |        |    |       |peer vpn service running       |
                                                    +-----------------+gw: 172.24.4.12                 +-----+        |    +-------+hold private subnet 20.0.0.0/24|
                           +--------------------+   |                 |VPN service running             |     |        |    |       |                               |
                           |VMX                 |   |                 |                                |     |        |    |       |                               |
                           |nic IP: 10.0.0.13   |   |                 +--------------------------------+     |        +----+       +-------------------------------+
                           |default via 10.0.0.1+---+                                                        |
                           |                    |   |                                                        |
                           +--------------------+   |                                                        |
                                                    |                                                        |
                                                    +                                                        +

As this diagram shows, the enterprise user owned many VMs in the private
network and the allocated IPs are from the private subnet 10.0.0.0/24. There
are two routers connected, ``Default Router`` is used for the normal network
functions, such as NAT, Floating IP, Routing. ``VPN Router`` is mainly used for
VPN traffic process, also contains NAT and route for other traffic. Both of the
routers are attached to the external network which is the physical underlay
network in the real data center. The VPN Router in OpenStack and Vendor GW
Router in other site maintain a VPN peer relationship across the Internet.

There are two scenarios towards the VM outgoing traffic:

1. VMs in the OpenStack private network access the normal websites, first send
   the network packets to its gateway which is located on the
   ``Default Router``. Then send to the Internet by the data center devices.
2. VMs in the OpenStack private network access the other site private network
   20.0.0.0/24, still first send the network packets to its gateway 10.0.0.1,
   then check the route tables, the nexthop of 20.0.0.0/24 is 10.0.0.5 which
   is located on ``VPN Router``. The network traffic will be sent based on the
   existing VPN tunnel to Vendor private site.

Like the diagram shows, the QoS Policy should be set on the qg-XX port of the
VPN router for limiting the outgoing VPN traffic.

This spec focuses on the QoS of VPN outgoing traffic, so for neutron-vpnaas,
this spec will focus on the Router related with VPN services. And for the
general use cases which is that VPN service usually setup across the Internet
in tunnel mode, we will only introduce the QoS support on tunnel type VPN
services.


Proposed Change
===============

We propose the ``VPN Service`` resource accepts the Neutron QoS Policy. Once
the ``ipsec site connection`` is created, the QoS Policy will be applied on the
VPN router's qg-XX port, as the ESP encapsulation will use the qg-XX port's IP
to access other sites.

So there are three parts that require to work:

1. DB related changes, including new table ``qos_vpnservice_policy_bindings``
   addition and data model change.
2. API changes, including extend the API to accept the Neutron QoS Policy.
3. Introduce a new l3 agent extension to extend the ability to process the QoS
   policy installation on the router.

Alternatives
------------

* Accept the QoS parameters and implement the QoS function on our own.
* Apply QoS Policy on the Router interface directly, but this would affect the
  west-east traffic.

Data model impact
-----------------
In this spec, the QoS data model and function will be provided by Neutron, so
``vpnservices`` table need to maintain the relationship with Neutron QoS
Policy.

The following new table is added as part of the VPN QoS feature::

    CREATE TABLE `qos_vpnservice_policy_bindings` (
      `vpn_service_id` varchar(36) NOT NULL,
      `qos_policy_id` varchar(36) NOT NULL,
      UNIQUE KEY `vpn_service_id` (`vpn_service_id`),
      KEY `qos_policy_id` (`qos_policy_id`),
      CONSTRAINT `qos_vpn_service_policy_bindings_ibfk_1` FOREIGN KEY (
      `qos_policy_id`) REFERENCES `qos_policies` (`id`) ON DELETE CASCADE,
      CONSTRAINT `qos_vpn_service_policy_bindings_ibfk_2` FOREIGN KEY (
      `vpn_service_id`) REFERENCES `vpnservices` (`id`) ON DELETE CASCADE
    );

REST API impact
---------------

Proposed attribute::

        EXTEND_FIELDS = {
            'qos_policy_id':{'allow_post': True, 'allow_put': True,
                             'validate': {'type:uuid': None},
                             'is_visible': True,
                             'default': None}
        }


Some samples in ``VPN service`` create/update. Users are allowed to pass
``qos_policy_id``.

Create/Update ``VPN service`` Request::

        POST /v2.0/vpn/vpnservices
        {
            "vpnservice": {
                "subnet_id": null,
                "qos_policy_id": "a36c20d0-18e9-42ce-88fd-82a35977ee8c",
                "router_id": "66e3b16c-8ce5-40fb-bb49-ab6d8dc3f2aa",
                "name": "myservice",
                "admin_state_up": true
            }
        }

        Response:
        {
            "vpnservice": {
                "router_id": "66e3b16c-8ce5-40fb-bb49-ab6d8dc3f2aa",
                "status": "PENDING_CREATE",
                "name": "myservice",
                "external_v6_ip": "2001:db8::1",
                "admin_state_up": true,
                "subnet_id": null,
                "project_id": "10039663455a446d8ba2cbb058b0f578",
                "tenant_id": "10039663455a446d8ba2cbb058b0f578",
                "external_v4_ip": "172.32.1.11",
                "id": "5c561d9d-eaea-45f6-ae3e-08d1a7080828",
                "description": "",
                "qos_policy_id": "a36c20d0-18e9-42ce-88fd-82a35977ee8c"
            }
        }

        PUT /v2.0/vpn/vpnservices/{service_id}
        {
            "vpnservice": {
                "name": "NEW VPN SERVICE NAME",
                "description": "Updated description",
                "qos_policy_id": "a36c20d0-18e9-42ce-88fd-82a35977ee8c"
            }
        }

        Response:
        {
            "vpnservice": {
                "router_id": "881b7b30-4efb-407e-a162-5630a7af3595",
                "status": "ACTIVE",
                "name": "NEW VPN SERVICE NAME",
                "admin_state_up": true,
                "subnet_id": null,
                "project_id": "26de9cd6cae94c8cb9f79d660d628e1f",
                "tenant_id": "26de9cd6cae94c8cb9f79d660d628e1f",
                "id": "41bfef97-af4e-4f6b-a5d3-4678859d2485",
                "description": "Updated description",
                "qos_policy_id": "a36c20d0-18e9-42ce-88fd-82a35977ee8c"
            }
        }


QoS Policy Application Details
------------------------------

The reason for introducing this, for example, we change the use case below, we
deploy the vpn service on the ``Default Router``, delete the ``VPN Router``.
That means the general traffic and VPN traffic will pass through the
``Default Router``, then we apply the Neutron QoS policy on the qg-XX port of
the ``Default Router``, it will limit all the bandwidth, so the VPN's bandwidth
may have a lower performance, or we can say it is not consistent with
expectations.

Currently, Neutron provides the QoS function but not for some interest streams.
Here we will focus on the VPN traffic. For this function, we will combine
the ``iptables`` and ``tc`` together. The reason for choosing them is that,
``iptables`` could mark the VPN interest stream by the ipsec VPN transform
protocols(such as esp, ah-esp protocols), the interface that the packets
will go out and the local encapsulated IP if running in tunnel mode. Also we
need to shape the vpn traffic before send out to the underlay network, so some
new ``iptables`` rules will be installed on mangle table in the router's
namespace. Also the ``fwmark`` is eaiser to extend, such as ipchains.

And we will introduce a new ``tc`` wrapper which will use ``htb`` and it will
provides classification algorithm. Then developers can easily implement other
complex traffic control. That means we will extend the current tc_lib in
Neutron repo. And ``vpn_qos`` will based on this.

Just like above description, a new L3 agent extension will be introduced like
fip_qos done. We suggest to name it ``vip_qos``, it will install the
appropriate ``iptables`` rules in the router's namespace which binds with
``VPN service``. Then users or managers could use the QoS function to the
``Default Router`` and not affect other network streams.


Security impact
---------------
None

Notifications impact
--------------------
No expected change.

Other end user impact
---------------------
Users will be able to specify qos_policy during create/update ``VPN service``.

Performance Impact
------------------
It will save the bandwidth of the underlay network in data center.

Other deployer impact
---------------------
None

Developer impact
----------------
Developer may use the new ``tc`` wrapper to do other things, as it is powerful
to support other stream control functionality.
But there may be a conflict with openstack/neutron-classifier, as it provides
defining the traffic. So we may reconsider that if possible.

Implementation
==============

Assignee(s)
-----------
zhaobo

Work Items
----------
* Add the DB model and extend the table column.
* Extend VPN API to accept QoS policy.
* Extend new tc wrapper which support classification algorithm based on
  traffic classifier feature.
* Extend new L3 agent extension ``vip_qos``.
* Add API validation code to validate access/existence of the qos_policy which
  created in Neutron.
* Add UTs to Neutron-vpnaas.
* Add API tests.
* Update CLI to accept QoS fields.
* Documentation work.

Dependencies
============
None

Testing
=======
Unit tests, functional tests, API tests and scenario tests are necessary.

Documentation Impact
====================
The Neutron API reference will need to be updated.

References
==========
