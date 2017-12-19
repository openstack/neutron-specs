..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================
Floating IP rate limit
======================

RFE: https://bugs.launchpad.net/neutron/+bug/1596611

Floating IP bandwidth is unrestricted now. But the NIC and the export of data
center may have a limitation for a cloud deployment. Then floating IP rate
limit is needed.

This spec describes how to add a built-in extension to limit the floating IP
bandwidth.


Problem Description
===================

Neutron now has floating IP whose bandwidth is not restricted, then there are
several reasons for adding rate limit to floating IP:

* a) Currently, neutron QoS implementation only affects neutron ports, more
  detail is that the bandwidth restriction is based on the neutron port, so all
  the VM traffic will be limited under that restriction, including L3 traffic.
  L3 needs its IP based implementation.

* b) North/South traffic always rely on infrastructure networking capacity.

* c) In case of Floating IP Traffic, cloud deployment do not have enough
  bandwidth to meet the total bandwidth requirement of all tenants at the
  same time. Due to this, the SLA of user's network traffic cannot be
  guaranteed. For example, if the VM NICs are given a high QoS value, then
  to meet the requirement of each tenant, the DC network would have a heavy
  load, thus it may be unable to facilitate additional requests and affect
  not only the tenant network bandwidth from North-South, but also East-West
  traffic.

* d) SNAT traffic in centralized network node will not meet the needs of all
  tenant bandwidth either. Because a NIC has limited bandwidth.


Proposed Change
===============

Overview
--------
We want to limit the bandwidth of floating IP. At the same time, the east/west
traffic should have no effect.

Some agreements:

* Allow binding QoS policy to floating IP.

* Each floating IP should have only one QoS policy.

* L3 agent needs a QoS extension to process L3 IP related QoS rules.

* QoS policy parameter is nullable. Floating IP can have no QoS policy
  assigned.


How the change works:

* Linux TC (Traffic Control) [1]_ will be used to implement such functionality.
  And for egress traffic a HTB [2]_ qdisc will be used to limit the floating
  IP outgoing bandwidth.


Solution Proposed
-----------------

Server side changes
+++++++++++++++++++

* New floating IP extension for QoS attribute (qos_policy_id).

* Adding a new relationship table between floating IP and QoS
  policy ``QoSFIPPolicyBinding``.

* L3 plugin will return the floating IP list with its bonding QoS policy
  during router sync RPC.


L3 agent side TC rules
++++++++++++++++++++++

Where to install the TC rules:

* For HA/legacy routers, rules for floating IP will be installed into network
  node qrouter namespace, and the traffic control device will be qg-device.

* For DVR routers, it's compute node qrouter namespace, but the floating IP
  traffic control device will be rfp-device (one of qrouter-namespace to
  fip-namespace pair).


For all IPs, in the corresponding namespace, the egress and ingress qdisc
rules are basically the same. The ``{qdisc_id}`` will be used to create the
IPs' tc filter. Some tips:

* Remember all the following commands will be executed in a namespace.
* [a-device] represents the qg-device or rfp-device in a namespace.
* {egress/ingress_qdisc_id} is the qdisc handle id.

Create qdisc commands:

::

  # egress
  tc qdisc add dev [a-device] root htb

  # ingress
  tc qdisc add dev [a-device] ingress

Actual qdisc rules:

::

  $ tc qdisc show dev [a-device]
  # egress
  qdisc htb {egress_qdisc_id} root refcnt 2 r2q 10 default 0 direct_packets_stat 400

  # ingress
  qdisc ingress {ingress_qdisc_id} parent ffff:fff1 ----------------


Then we can use that {ingress_qdisc_id} and {egress_qdisc_id} to create the
L3 IPs filters. Assuming we have a L3 IP 172.16.6.161, and it has a QoS policy
with rule {qos_rate_value: 1000Kbit, qos_burst_value: 1Mb}. The ingress and
egress tc filter create commands will be:

::

  # ingress
  $ tc filter add dev [a-device] parent {ingress_qdisc_id} protocol ip prio 1 \
    u32 match ip dst 172.16.6.161 police \
    rate 1000Kbit burst 1Mb drop flowid :1

  # egress
  $ tc filter add dev [a-device] parent {egress_qdisc_id} protocol ip prio 1 \
    u32 match ip src 172.16.6.161 police \
    rate 1000Kbit burst 1Mb drop flowid :1


Actual filter rules:

::

  # ingress
  $ tc -s -d -p filter show dev [a-device] parent {ingress_qdisc_id}
  ...
  filter protocol ip pref 1 u32 fh 800::800 order 2048 key ht 800 bkt 0 flowid :1  (rule hit 0 success 0)
    match IP dst 172.16.6.161/32 (success 0 )
   police 0x67 rate 1000Kbit burst 1Mb mtu 64Kb action drop overhead 0b
  ...

  # egress
  $ tc -s -d -p filter show dev [a-device] parent {egress_qdisc_id}
  ...
  filter protocol ip pref 1 u32 fh 800::800 order 2048 key ht 800 bkt 0 flowid :1  (rule hit 0 success 0)
    match IP src 172.16.6.161/32 (success 0 )
   police 0x68 rate 1000Kbit burst 1Mb mtu 64Kb action drop overhead 0b
  ...


The neutron basic workflow
--------------------------


Floating IP
+++++++++++
* 1. User create a floating IP.
* 2. Floating IP was associated with a port.
* 3. Create/find QoS policy and rule for floating IP.
* 4. Bind QoS policy to floating IP (floating IP update API).
* 5. L3 agent processes the router and its floating IP.
* 6. L3 agent sets the TC rules to the qrouter-namespace relevant device.

After this the floating IP will be set under a bandwidth restriction.


Example L3 agent side TC rules
------------------------------
Assuming that we have a legacy router: cf6951cb-b050-4543-9742-c63a4989edae,
gateway ip: 172.16.6.167, 1Mbps, floating IP: 172.16.10.69, 1Mbps, then in
its scheduled network node you can get the following rules:

::

  # ip netns exec qrouter-cf6951cb-b050-4543-9742-c63a4989edae ip a
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
      inet 127.0.0.1/8 scope host lo
  140: qr-4aba9b05-36: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN
      inet 192.168.233.1/24 brd 192.168.233.255 scope global qr-4aba9b05-36
  141: qr-aef0d42d-f9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN
      inet 192.168.232.1/24 brd 192.168.232.255 scope global qr-aef0d42d-f9
  143: qg-c99a5832-7f: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc htb state UNKNOWN
      inet 172.16.6.167/16 brd 172.16.255.255 scope global qg-c99a5832-7f
      inet 172.16.10.69/32 brd 172.16.10.69 scope global qg-c99a5832-7f
         valid_lft forever preferred_lft forever

  # ip netns exec qrouter-cf6951cb-b050-4543-9742-c63a4989edae tc qdisc show dev qg-c99a5832-7f
  qdisc htb 801b: root refcnt 2 r2q 10 default 0 direct_packets_stat 400
  qdisc ingress ffff: parent ffff:fff1 ----------------

  # ip netns exec qrouter-cf6951cb-b050-4543-9742-c63a4989edae tc -s -d -p filter show dev qg-c99a5832-7f parent 801b:
  ...
  filter protocol ip pref 1 u32 fh 800::800 order 2048 key ht 800 bkt 0 flowid :1  (rule hit 0 success 0)
    match IP src 172.16.6.167/32 (success 0 )
   police 0xa74 rate 1000Kbit burst 1Mb mtu 64Kb action drop overhead 0b
  ref 1 bind 1

   Sent 0 bytes 0 pkts (dropped 0, overlimits 0)
  filter protocol ip pref 1 u32 fh 800::801 order 2049 key ht 800 bkt 0 flowid :1  (rule hit 0 success 0)
    match IP src 172.16.10.69/32 (success 0 )
   police 0xa76 rate 1000Kbit burst 1Mb mtu 64Kb action drop overhead 0b
  ref 1 bind 1

   Sent 0 bytes 0 pkts (dropped 0, overlimits 0)

  # ip netns exec qrouter-cf6951cb-b050-4543-9742-c63a4989edae tc -s -d -p filter show dev qg-c99a5832-7f parent ffff:
  ...
  filter protocol ip pref 1 u32 fh 800::800 order 2048 key ht 800 bkt 0 flowid :1  (rule hit 0 success 0)
    match IP dst 172.16.6.167/32 (success 0 )
   police 0xa73 rate 1000Kbit burst 1Mb mtu 64Kb action drop overhead 0b
  ref 1 bind 1

   Sent 0 bytes 0 pkts (dropped 0, overlimits 0)
  filter protocol ip pref 1 u32 fh 800::801 order 2049 key ht 800 bkt 0 flowid :1  (rule hit 0 success 0)
    match IP dst 172.16.10.69/32 (success 0 )
   police 0xa75 rate 1000Kbit burst 1Mb mtu 64Kb action drop overhead 0b
  ref 1 bind 1

   Sent 0 bytes 0 pkts (dropped 0, overlimits 0)

And DVR router 0580788c-c919-447c-aea1-87d415aa173a with floating IP
172.16.6.161 1Mbps in a compute node:

::

  # ip netns exec qrouter-0580788c-c919-447c-aea1-87d415aa173a ip a
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
      inet 127.0.0.1/8 scope host lo
  24: rfp-0580788c-c: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc htb state UP qlen 1000
      inet 169.254.106.114/31 scope global rfp-0580788c-c
      inet 172.16.6.161/32 brd 172.16.6.161 scope global rfp-0580788c-c
  102: qr-43185e93-af: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN
      inet 192.168.199.1/24 brd 192.168.199.255 scope global qr-43185e93-af
  104: qr-51635a61-46: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN
      inet 192.168.198.1/24 brd 192.168.198.255 scope global qr-51635a61-46

  # ip netns exec qrouter-0580788c-c919-447c-aea1-87d415aa173a tc qdisc show dev rfp-0580788c-c
  qdisc htb 801e: root refcnt 2 r2q 10 default 0 direct_packets_stat 5
  qdisc ingress ffff: parent ffff:fff1 ----------------

  # ip netns exec qrouter-0580788c-c919-447c-aea1-87d415aa173a tc -s -d -p filter show dev rfp-0580788c-c parent 801e:
  ...
  filter protocol ip pref 1 u32 fh 800::800 order 2048 key ht 800 bkt 0 flowid :1  (rule hit 0 success 0)
    match IP src 172.16.6.161/32 (success 0 )
   police 0x68 rate 1000Kbit burst 1Mb mtu 64Kb action drop overhead 0b
  ref 1 bind 1

   Sent 0 bytes 0 pkts (dropped 0, overlimits 0)

  # ip netns exec qrouter-0580788c-c919-447c-aea1-87d415aa173a tc -s -d -p filter show dev rfp-0580788c-c parent ffff:
  ...
  filter protocol ip pref 1 u32 fh 800::800 order 2048 key ht 800 bkt 0 flowid :1  (rule hit 0 success 0)
    match IP dst 172.16.6.161/32 (success 0 )
   police 0x67 rate 1000Kbit burst 1Mb mtu 64Kb action drop overhead 0b
  ref 1 bind 1

   Sent 0 bytes 0 pkts (dropped 0, overlimits 0)


IPv6 Impact
-----------
Only IPv4 has the concept of floating IP.


Data Model Impact
-----------------
For floating IP:
A new table, QoSFIPPolicyBinding, will be created to represent the 1-1
association between floating IP and QoS policy, each policy has rules in both
ingress and egress direction.


REST API Impact
---------------
New floating IP API extension:

::

  floatingip: {
         ...
         'qos_policy_id': {'allow_post': True, 'allow_put': True,
                           'validate': {'type:uuid_or_none': None},
                           'is_visible': True,
                           'default': None},}


New floating IP commands for creation, deletion and update elements with
QoS property:

::

  openstack floating ip create --qos-policy policy_id <network>
  openstack floating ip set --qos-policy policy_id floating_ip_id
  openstack floating ip unset --qos-policy floating_ip_id



Implementation
==============

Assignee(s)
-----------

* LIU Yulong <i@liuyulong.me>


Work Items
----------

* Create model tables and API extensions.
* New TC command wrapper for floating IP rate limit.
* L3 agent extension for QoS rule installation.
* CLI support.
* Testing.
* Documentation.

Dependencies
============

None

Testing
=======

Functionality
-------------
The bandwidth of floating IP should be restricted properly. Adding new
fullstack and tempest scenarios to test the bandwidth of ingress and egress.


References
==========

.. [1] `Linux Traffic Control <http://www.tldp.org/HOWTO/Traffic-Control-HOWTO>`_

.. [2] `Linux HTB Queuing <http://www.linuxjournal.com/article/7562>`_

Related Information
-------------------

- `Floating IP rate limit <https://review.openstack.org/#/c/424466/>`_
- `Router gateway rate limit (abandoned) <https://review.openstack.org/#/c/424468/>`_
- `Adding L3 rate limit TC lib <https://review.openstack.org/#/c/453458/>`_
