..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================
Distributed metadata datapath for Openvswitch Agent
===================================================

RFE: https://bugs.launchpad.net/neutron/+bug/1933222

When instances are booting, they will try to retrieve metadata from
Nova by the path of Neutron virtual switches(bridges), virtual devices,
namespaces and metadata-agents. After that, metadata agent has no other
functionalities. In large-scale scenarios, a large number of deployed
metadata agents will cause a waste of resources for hosts and message queue.
Because metadata agents will start tons of external processes based on
the number of users' resources, report state to Neutron server for keepalive.

We are going to overcome this. This spec describes how to implement an
agent extension for Neutron openvswitch agent to make the metadata
datapath distributed.

Problem Description
===================

How many metadata-agent should run for a large scale cloud deployment?
There is no exact answer to this question. Cloud operators may setup
metadata agent to all hosts, something like DHCP agent.

Config drive [1]_ can be an alternative for clouds to supply metadata for VMs.
But there are some limitations and security problem for config drive. For
instance, if you set config drive to use local disk, the live migration may
not be available. Config drive uses extra storage device and mounting it
inside VMs. Can not change userdata online for users specific scripts.
The security issue is that because the mounting FS can be access by all users,
if the metadata includes root password or key, the password and the key
can be accessed by none root users.

If you are running Neutron agents, there is no alternative to replace
metadata agent for cloud deployments.

As we can see, the metadata datapath is very long via many devices, namespaces
and agents. One metadata path goes down, such as agent down or external process
dead, will not only influence the host, but also all related hosts that will
boot new VMs on.

Proposed Change
===============

Adds an agent extension for Neutron openvswitch agent to make the metadata
datapath distributed.

.. note:: This extension is only for openvswitch agent, other mechanism drivers
          will not be considered, because this new extension will rely on the
          openflow protocol and principle.

Solution Proposed
-----------------

Metadata Special purpose IP and MAC
***********************************

Each VM in one host will generate a Meta IP + MAC pair which is unique among VM
on a host and which will be used to identify the metadata requestor
vm/port. The agent extension will do the generation work when handle port
in the first time, and store the Meta IP + MAC to ovsdb. When ovs-agent is
restarting, handle_port funtion will try to read the ovsdb first to reload
the Meta IP + MAC, if not exist, re-generate.

During the agent extension initialization, it will add a internal port named
"tap-meta" to the ovs-bridge br-meta. This tap device will have a Meta Gateway
MAC + IP.

Potential Configrations
***********************

Config options for Neutron openvswitch agent with this extension:

::

  [METADATA]
  provider_cidr = 100.100.0.0/16
  provider_vlan_id = 998
  provider_base_mac = "fa:16:ee:00:00:00"

* ``provider_cidr`` will be used as the IP range to generate the VM's metadata IP.
* ``provider_vlan_id`` will be a fixed vlan for data from local tap-meta dev.
* ``provider_base_mac`` will be the MAC prefix to generate the VM's META MAC.

We can reuse the existing config option from metadata agent:

::

  METADATA_PROXY_HANDLER_OPTS = [
      cfg.StrOpt('auth_ca_cert',
                 help=_("Certificate Authority public key (CA cert) "
                        "file for ssl")),
      cfg.HostAddressOpt('nova_metadata_host',
                         default='127.0.0.1',
                         help=_("IP address or DNS name of Nova metadata "
                                "server.")),
      cfg.PortOpt('nova_metadata_port',
                  default=8775,
                  help=_("TCP Port used by Nova metadata server.")),
      cfg.StrOpt('metadata_proxy_shared_secret',
                 default='',
                 help=_('When proxying metadata requests, Neutron signs the '
                        'Instance-ID header with a shared secret to prevent '
                        'spoofing. You may select any string for a secret, '
                        'but it must match here and in the configuration used '
                        'by the Nova Metadata Server. NOTE: Nova uses the same '
                        'config key, but in [neutron] section.'),
                 secret=True),
      cfg.StrOpt('nova_metadata_protocol',
                 default='http',
                 choices=['http', 'https'],
                 help=_("Protocol to access nova metadata, http or https")),
      cfg.BoolOpt('nova_metadata_insecure', default=False,
                  help=_("Allow to perform insecure SSL (https) requests to "
                         "nova metadata")),
      cfg.StrOpt('nova_client_cert',
                 default='',
                 help=_("Client certificate for nova metadata api server.")),
      cfg.StrOpt('nova_client_priv_key',
                 default='',
                 help=_("Private key of client certificate."))
  ]

Basic backend config with http for haproxy will be:

::

  backend backend_{{ instance_x.uuid }}_{{ metadata_ip_x }}
  server metasrv http://nova_metadata_host:nova_metadata_port

In haproxy side if ssl verify is enabled, the backend will be:

::

  backend backend_{{ instance_x.uuid }}_{{ metadata_ip_x }}
  server metasrv https://nova_metadata_host:nova_metadata_port ssl verify required ca-file /path/to/auth_ca_cert

Or ssl with client certificate:

::

  backend backend_{{ instance_x.uuid }}_{{ metadata_ip_x }}
  server metasrv https://nova_metadata_host:nova_metadata_port ssl verify required ca-file /path/to/nova_client_cert crt /path/to/nova_client_priv_key

.. note:: If the cloud has its own client certificate, the ``crt`` parameter
          can point to your client certificate file. But this is supported
          since haproxy version >= 2.4.


Metadata data pipelines
***********************


Sample Resource Datas
~~~~~~~~~~~~~~~~~~~~~

* VM1 - network1 local_vlan_id=1, fixed_ip 192.168.1.10, port mac fa:16:3e:4a:fd:c1, Meta_IP 100.100.0.10, Meta_MAC fa:16:ee:00:00:11
* VM2 - network2 local_vlan_id=2, fixed_ip 192.168.2.10, port mac fa:16:3e:4a:fd:c2, Meta_IP 100.100.0.11, Meta_MAC fa:16:ee:00:00:22
* VM3 - network1 local_vlan_id=1, fixed_ip 192.168.1.20, port mac fa:16:3e:4a:fd:c3, Meta_IP 100.100.0.12, Meta_MAC fa:16:ee:00:00:33
* VM4 - network3 local_vlan_id=3, fixed_ip 192.168.3.10, port mac fa:16:3e:4a:fd:c4, Meta_IP 100.100.0.13, Meta_MAC fa:16:ee:00:00:44

* META Gateway IP 100.100.0.1, META Gateway MAC: fa:16:ee:00:00:01

TCP Egress
~~~~~~~~~~

HTTP request packets from VM direct to br-meta, and change IP headers to tap-meta,
add HTTP headers in host haproxy then go to nova-metadata API. Datapath:

::

  +----+ TCP +-----------------------------------+ TCP +---------------------------------------------+ TCP +------------------------+ TCP +-----------------------+
  |    +----->             Br-int                +----->                   Br-meta                   +----->        tap-Meta        +----->        Haproxy        |
  | VM |     | From VM port + 169.254.169.254:80 |     |   Source (VM MAC + IP --> Meta MAC + IP)    |     |  Meta Gateway MAC + IP |     |   Match Meta IP       |
  |    |     |          add local vlan           |     |  Dest (MAC + IP --> Meta Gateway MAC + IP)  |     |       Listened by      |     |     Add Http header   |
  |    |     |           to Br-meta              |     |                 to tap-Meta                 |     |        Haproxy         |     |  to Nova-Metadata-API |
  +----+     +-----------------------------------+     +---------------------------------------------+     +------------------------+     +-----------------------+

Flows (some keywords are pseudo code) on br-int:

::

  Table=0
  Match: ip,in_port=<of_vm1>,nw_dst=169.254.169.254 actions=mod_local_vlan:1,output:"To_br_meta"
  Match: ip,in_port=<of_vm2>,nw_dst=169.254.169.254 actions=mod_local_vlan:2,output:"To_br_meta"
  Match: ip,in_port=<of_vm3>,nw_dst=169.254.169.254 actions=mod_local_vlan:1,output:"To_br_meta"
  Match: ip,in_port=<of_vm4>,nw_dst=169.254.169.254 actions=mod_local_vlan:3,output:"To_br_meta"

When your VM trying to access 169.254.169.254:80, what should the dest
MAC + IP be? The dest IP is clear, it is 169.254.169.254. The complicated
case is the dest MAC. We have three scenarios:

a. if your VM has only one default route which point to gateway, so the request
dest MAC should be gateway MAC.

b. if your VM has a route which directly point to 169.254.169.254 (for instance,
to 169.254.169.254 via 192.168.1.2 <the DHCP port IP>, normally, this is set
by original DHCP-agent and metadata mechanism), so some ARP responder(s) will
be added for such DHCP port IPs, in case of upgrading. A fake mac will be
responded for these DHCP port IPs.

c. if your VM has a link route which is telling guest OS 169.254.169.254 is
directly reachable. So an ARP responder for 169.254.169.254 will be added.
So the dest MAC will be a fake one as well.

Flows on br-meta:

::

  Table=0
  Match: ip,in_port=<patch_br_int>,nw_dst=169.254.169.254 Action: resubmit(,80)

  Table=80
  Match: dl_vlan=<local_vlan_1>,dl_src=fa:16:3e:4a:fd:c1,nw_src=192.168.1.10 Action: strip_vlan,mod_dl_src:fa:16:ee:00:00:11,mod_nw_src:100.100.0.10, resubmit(,87)
  Match: dl_vlan=<local_vlan_2>,dl_src=fa:16:3e:4a:fd:c2,nw_src=192.168.2.10 Action: strip_vlan,mod_dl_src:fa:16:ee:00:00:22,mod_nw_src:100.100.0.11, resubmit(,87)
  Match: dl_vlan=<local_vlan_1>,dl_src=fa:16:3e:4a:fd:c3,nw_src=192.168.1.20 Action: strip_vlan,mod_dl_src:fa:16:ee:00:00:33,mod_nw_src:100.100.0.12, resubmit(,87)
  Match: dl_vlan=<local_vlan_3>,dl_src=fa:16:3e:4a:fd:c4,nw_src=192.168.3.10 Action: strip_vlan,mod_dl_src:fa:16:ee:00:00:44,mod_nw_src:100.100.0.13, resubmit(,87)

  Table=87
  Match: tcp,nw_dst=169.254.169.254,tp_dst=80 Action: mod_nw_dst:100.100.0.1, mod_dl_dst:fa:16:ee:00:00:01,output:"tap-meta"

TCP Ingress
~~~~~~~~~~~

HTTP packets come from tap-meta to br-meta directly, then go to br-int and
finnaly direct to VM. Datapath:

::

  +----+     +---------------------+     +---------------------------------------+     +---------------+     +------------------+
  |    |     |      Br-int         |     |                Br-meta                |     |    tap-Meta   |     |      Haproxy     |
  | VM |     | From patch_br_meta  |     |    Source (IP ---> 169.254.169.254)   |     |               |     |  Http Response   |
  |    | TCP |   mac_dst is VM     | TCP |  Dest(Meta MAC + IP ---> VM MAC + IP) | TCP |      To       | TCP |   To Client IP   |
  |    <-----+   output to VM      <-----+               to  Br-int              <-----+ Meta MAC + IP <-----+  (Meta MAC + IP) |
  +----+     +---------------------+     +---------------------------------------+     +---------------+     +------------------+

Flows on br-meta:

::

  Table=0
  Match: ip,in_port="tap-meta" actions=push_vlan,goto_table:91

  Table=91
  Match: dl_vlan=<998>,ip,nw_dst=100.100.0.10 Action: mod_vlan_vid:1,mod_dl_dst:fa:16:3e:4a:fd:c1,mod_nw_dst:192.168.1.10,mod_nw_src:169.254.169.254,output:"to-br-int"
  Match: dl_vlan=<998>,ip,nw_dst=100.100.0.11 Action: mod_vlan_vid:2,mod_dl_dst:fa:16:3e:4a:fd:c2,mod_nw_dst:192.168.2.10,mod_nw_src:169.254.169.254,output:"to-br-int"
  Match: dl_vlan=<998>,ip,nw_dst=100.100.0.12 Action: mod_vlan_vid:3,mod_dl_dst:fa:16:3e:4a:fd:c3,mod_nw_dst:192.168.1.20,mod_nw_src:169.254.169.254,output:"to-br-int"
  Match: dl_vlan=<998>,ip,nw_dst=100.100.0.13 Action: mod_vlan_vid:4,mod_dl_dst:fa:16:3e:4a:fd:c4,mod_nw_dst:192.168.3.10,mod_nw_src:169.254.169.254,output:"to-br-int"

Flows on br-int:

::

  Table=0
  Match: ip,in_port=<patch_br_meta>,dl_vlan=1,dl_dst=<vm1_mac_fa:16:3e:4a:fd:c1>,nw_src=169.254.169.254 actions=strip_vlan,output:<of_vm1>
  Match: ip,in_port=<patch_br_meta>,dl_vlan=2,dl_dst=<vm2_mac_fa:16:3e:4a:fd:c2>,nw_src=169.254.169.254 actions=strip_vlan,output:<of_vm2>
  Match: ip,in_port=<patch_br_meta>,dl_vlan=3,dl_dst=<vm3_mac_fa:16:3e:4a:fd:c3>,nw_src=169.254.169.254 actions=strip_vlan,output:<of_vm3>
  Match: ip,in_port=<patch_br_meta>,dl_vlan=4,dl_dst=<vm4_mac_fa:16:3e:4a:fd:c4>,nw_src=169.254.169.254 actions=strip_vlan,output:<of_vm4>

ARP for Metadata IPs
~~~~~~~~~~~~~~~~~~~~

Tap-meta device will be resident on host kernel IP stack, before the first
response of TCP, the host (protocol stack) needs to know the META_IP's MAC
address. So ARP reqeust is broadcast. ARP will be sent from tap-meta device
to br-meta responder. The ARP responder datapath:

::

  +---------------------------+      +---------------+
  |         Br-meta           |      |    tap-Meta   |
  | ARP Responder for Meta IP |      |     Learn     |
  |           to              | ARP  | Meta IP's MAC |
  |         INPORT            <------+     (ARP)     |
  +---------------------------+      +---------------+

The flows on br-meta will be::

  Ingress:
  Table=0
  Match: arp,in_port="tap-meta" Action: resubmit(,90)

  Table=90
  Match: arp,arp_tpa=100.100.0.10 Action: ARP Responder with Meta_MAC fa:16:ee:00:00:11,IN_PORT
  Match: arp,arp_tpa=100.100.0.11 Action: ARP Responder with Meta_MAC fa:16:ee:00:00:22,IN_PORT
  Match: arp,arp_tpa=100.100.0.12 Action: ARP Responder with Meta_MAC fa:16:ee:00:00:33,IN_PORT
  Match: arp,arp_tpa=100.100.0.13 Action: ARP Responder with Meta_MAC fa:16:ee:00:00:44,IN_PORT


Host haproxy configurations
***************************

The host haproxy is one only process which is used for all VMs. The host
haproxy will add HTTP headers to the metadata request which is needed for
metadata API. The headers have a fixed algorithm which is easy to
assemble. For each VM's request, haproxy will add an independent backend
and a match rule of checking the source IP (aka Meta_IP). While the request
from one VM's (Meta_IP) it will be send to the matched backend, which add
HTTP headers and then send to real nova-metadata-api.

Configurations:

::

    global
        log         /dev/log local0 {{ log_level }}
        user        {{ user }}
        group       {{ group }}
        maxconn     {{ maxconn }}
        daemon

    frontend public
        bind            *:80 name clear
        mode            http
        log             global
        option          httplog
        option          dontlognull
        maxconn         {{ maxconn }}
        timeout client  30s

        acl instance_{{ instance_1.uuid }}_{{ metadata_ip_1 }} src {{ metadata_ip_1 }}
        acl instance_{{ instance_2.uuid }}_{{ metadata_ip_2 }} src {{ metadata_ip_2 }}
        acl instance_{{ instance_3.uuid }}_{{ metadata_ip_3 }} src {{ metadata_ip_3 }}
        acl instance_{{ instance_4.uuid }}_{{ metadata_ip_4 }} src {{ metadata_ip_4 }}

        use_backend backend_{{ instance_1.uuid }}_{{ metadata_ip_1 }} if instance_{{ instance_1.uuid }}_{{ metadata_ip_1 }}
        use_backend backend_{{ instance_2.uuid }}_{{ metadata_ip_2 }} if instance_{{ instance_2.uuid }}_{{ metadata_ip_2 }}
        use_backend backend_{{ instance_3.uuid }}_{{ metadata_ip_3 }} if instance_{{ instance_3.uuid }}_{{ metadata_ip_3 }}
        use_backend backend_{{ instance_4.uuid }}_{{ metadata_ip_4 }} if instance_{{ instance_4.uuid }}_{{ metadata_ip_4 }}

    backend backend_{{ instance_1.uuid }}_{{ metadata_ip_1 }}
        balance         roundrobin
        retries         3
        option redispatch
        timeout http-request    30s
        timeout connect         30s
        timeout server          30s

        http-request set-header X-Instance-ID {{ instance_1.uuid }}
        http-request set-header X-Tenant-ID {{ instance_1.project_id }}
        http-request set-header X-Instance-ID-Signature {{ instance_1.signature }}

        server metasrv ...

    backend backend_{{ instance_2.uuid }}_{{ metadata_ip_2 }}
        balance         roundrobin
        retries         3
        option redispatch
        timeout http-request    30s
        timeout connect         30s
        timeout server          30s

        http-request set-header X-Instance-ID {{ instance_2.uuid }}
        http-request set-header X-Tenant-ID {{ instance_2.project_id }}
        http-request set-header X-Instance-ID-Signature {{ instance_2.signature }}

        server metasrv ...

    backend backend_{{ instance_3.uuid }}_{{ metadata_ip_3 }}
        ...

    backend backend_{{ instance_4.uuid }}_{{ metadata_ip_4 }}
        ...

IPv6 metadata
*************

The metadata for IPv6 [2]_ only network has similar address ``fe80::a9fe:a9fe``,
so all these works can be mirrored for IPv6. For IPv6 the generator
will use the range ``fe80:ffff:a9fe:a9fe::/64`` to allocate Meta_IPv6 address.
The Meta_gateway_IPv6 address will be ``fe80:ffff:a9fe:a9fe::1``, Gateway MAC is
still the same one ``fa:16:ee:00:00:01``. NA responder
for Meta_IPv6 address ``fe80:ffff:a9fe:a9fe::abcd`` and Meta_MAC
``fa:16:ee:00:00:11`` will be:

::

  Table=0
  Match: icmp6,icmp_type=135,icmp_code=0,nd_sll=fa:16:ee:00:00:01,
  actions=set_field:136->icmpv6_type,set_field:0->icmpv6_code,set_field:2->icmpv6_options_type,goto_table:90

  Table=90
  Match: icmp6,icmp_type=136,icmp_code=0,nd_target=fe80:ffff:a9fe:a9fe::abcd
  actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],set_field:fa:16:ee:00:00:11->eth_src,
  move:NXM_NX_IPV6_SRC[]->NXM_NX_IPV6_DST[],set_field:fe80:ffff:a9fe:a9fe::abcd->ipv6_src,
  set_field:fa:16:ee:00:00:11->nd-tll,set_field:0xE000->OFPXMT_OFB_ICMPV6_ND_RESERVED,IN_PORT



Implementation
==============

Assignee(s)
-----------

* LIU Yulong <i@liuyulong.me>


Work Items
----------

* Adding flows action for ovs bridges
* Adding host haproxy manager for ovs-agent
* Adding host metadata IP and Mac generator with ovsdb settings
* Adding ovs-agent extension to set up flows for VM ports
* Testing.
* Documentation.

Dependencies
============

None

Testing
=======

Test cases to verify the metadata can be set properly, this can be done by
using existing jobs with new new metadata driver.

References
==========

.. [1] https://docs.openstack.org/nova/latest/user/metadata.html
.. [2] https://bugs.launchpad.net/neutron/+bug/1460177
