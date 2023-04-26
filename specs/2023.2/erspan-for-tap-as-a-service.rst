..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================================
Tunnel based mirroring (ERSPAN, GRE) for Tap-as-a-service
=========================================================

https://bugs.launchpad.net/neutron/+bug/2015471

Mirroring is a widely used tool to analyse traffic of switch ports.
The Tap-as-a-service project was created to allow admins to mirror traffic
of one Neutron port to another Neutron port.

Mirroring can also be done by encapsulating the traffic into a tunnel, like
GRE (``Generic Routing Encapsulation``) or ERSPAN (``Encapsulated Remote
Switch Port Analyzer``). ERSPAN first was used widely in Cisco switches, and
GRE is widely used as tunneling protocol.

ERSPAN protocol has 3 versions of which the last two is adapted, these are
``version 1`` and ``version 2`` (the other versioning uses TYPE I, II and III,
and TYPE II is version 1 and TYPE III is version 2)
For more details see the `ERSPAN draft from Cisco`_.

Since OVS 2.10 it is possible to use ERSPAN with OVS
(see `OVS basic configuration`_, and `OVS protocol header fields`_) both
ERSPAN v1 and v2, see `erspan NEWS update commit`_.

Since OVN v22.12.0 it is possible to create mirrors with OVN
(see `OVN 22.12 nbctl man page`_, and `OVN commit that introduced mirroring`_).

.. note::
    OVN only supports ERSPAN v1, and with OVN it is also possible to create
    a clean GRE type mirror.

This specification proposes an extension to the current tap-as-a-service
(TAAS) API to allow the users to create ERSPAN or GRE mirrors from Neutron
ports to a remote IP, and proposes the necessary backend changes to the
current OVS driver of TAAS and proposes a new driver for OVN to use ERSPAN
or GRE mirroring with OVN.

Problem Description
===================

Mirroring traffic can be useful in many situations for operators, for example
to debug network issues.

Tap-as-a-service provided a solution for traffic mirroring by allowing to
create tap-flows and mirror the traffic of them to the related tap-service.
Each tap-flow and tap-service can be attached to a Neutron port, so the port
attached to tap-flow is the source of the mirrored traffic and the port of
tap-service is the destination of the mirroring.
There is a N-1 relation between tap-flows and tap-services.
This mirroring model is mirroring traffic from one Neutron port to another
Neutron port over a Neutron network.

An ERSPAN or GRE mirror is a good solution when an operator needs to mirror
traffic from the cloud (from Neutron ports) to an analyser outside the cloud.

Use Cases
---------

* As an operator I want to mirror the traffic from a ``Neutron port`` to a
  network analyser that can be ``outside of my cloud``.

* As an operator I want to mirror the traffic from a ``Neutron port``
  to a ``Floating IP``.

* As an operator I want to mirror the traffic of a ``Neutron port`` to
  a ``dedicated infra network``, to avoid overloading the tenant networks.

* As an operator I want to use the extra headers ERSPAN provides (i.e.:
  original VLAN, original CoS in case of version 1 ERSPAN).

Proposed Change
===============

The proposal is to use OVS and OVN builtin ERSPAN version 1 and GRE mirroring
features. For using ERSPAN or GRE with OVS a port is added to an OVS bridge
with ``type=erspan`` or ``type=gre`` in case of GRE.

As ERSPAN is a modification of GRE, with some extra ERSPAN specific
headers (see `ERSPAN draft from Cisco`_), OVS creates the tunnel from the
previously created OVS port to the destination IP. This means that the mirrored
traffic is encapsulated to an ERSPAN/GRE tunnel.

Example wireshark dump of an ICMP echo reply::

    Frame 1: 148 bytes on wire (1184 bits), 148 bytes captured (1184 bits)
    Ethernet II, Src: RealtekU_79:ff:db (52:54:00:79:ff:db), Dst: RealtekU_91:2f:52 (52:54:00:91:2f:52)
    Internet Protocol Version 4, Src: 100.109.0.84, Dst: 100.109.0.142
    Generic Routing Encapsulation (ERSPAN)
    Encapsulated Remote Switch Packet ANalysis Type II
    Ethernet II, Src: fa:16:3e:d5:4b:c1 (fa:16:3e:d5:4b:c1), Dst: fa:16:3e:80:ed:09 (fa:16:3e:80:ed:09)
    Internet Protocol Version 4, Src: 10.0.0.39, Dst: 10.0.0.47
    Internet Control Message Protocol

This means that the source IP of the tunnel is the IP of the host on which the
ERSPAN port is created (in my virtual env it is 100.109.0.84).
In the above example the 2 inner IPs (10.0.0.47 and 10.0.0.39) are the fixed
IPs of 2 Openstack ports (VMs).

The outer protocol source IP (100.109.0.84) is the IP of the host on which the
mirror port is created, thus outside of the cloud, and in the control of the
admin.

.. note::

    This spec doesn't specify the source address (mac or IP) of the tunnel.
    So this is the task of the admin and out of scope for this spec.

The outer protocol destination IP (100.109.0.142) in this case is also outside
of the cloud, and Openstack Neutron control, and another host on which I can
run tcpdump.

REST API impact
---------------

The current API model of TAAS uses two high level objects: `tap-services` and
`tap-flows`. The tap-flow represents the source of the mirrored traffic,
and a tap-service represents the destination of the mirrored traffic. For
one tap-service multiple tap-flows can be attached. For details please check
the `tap-as-a-service API reference`_.
Both a tap-flow and a tap-service are referencing a Neutron port, and the
traffic on that port will be the source of the mirror (in case of a tap-flow),
or the destination of the mirror (in case of a tap-service).

.. warning::

    Mirroring only works for traffic which is allowed by security-groups.

The above API model is not useful when the intent is to implement mirroring
using ERSPAN or GRE tunnels, because the traffic source is a bridge port
(in the case of OVS) or a logical switch port (in the case of OVN) and the
destination is represented only by an IP address.

The proposal is to introduce a new high level API for ERSPAN or GRE mirroring:
``tap_mirror``.

This solution keeps the current API clean and not overloaded, and makes it
easier for operators to expect the right behaviour after API operations.

The proposed API is ``admin only``, to avoid the overloading of infrastructure
networks by tenants.

The suggested API request:

* ``POST /v2.0/taas/tap_mirrors``

  Create a tap mirror that mirrors traffic from a Neutron port to an
  external IP::

    {
        "tap_mirror": {
            "name": "mirror-traffic-of-server-a0",
            "description": "Mirror the traffic from server-a0",
            "directions": {"IN": 99, "OUT": 100},
            "port_id": "1a1a5a96-e8cb-11ed-9678-9b663820b519",
            "remote_ip": "172.31.1.1",
            "mirror_type": "erspanv1"|"gre"
        }
    }

* ``port_id`` is the source of the mirroring, this is a ``Neutron port``.
  One port can be attached to one tap_mirror only.

.. note::

    Only VM ports can be used as the source of the mirroring.

* ``remote_ip``: The IP of the remote end of the tunnel.

.. note::

    This API proposal keeps the current TAAS API's N-1 relationship between
    source and destination. Multiple source ports' traffic can be mirrored to
    one destination IP.

* ``mirror_type`` field is to select between ERSPAN and GRE.

* ``directions`` is a dictionary with `direction:tunnel_id` pairs.
  The current tap-as-a-service API allows the operator to select the direction
  when the `tap-flow` is created, it can be: IN, OUT, BOTH.
  This specification proposes to keep ``IN`` and ``OUT`` as the keys of the
  dictionary. Meaning of the directions:

  * ``IN``: the traffic towards the port, and into the VM attached to it
    (ingress traffic).

  * ``OUT``: traffic from the port, out of the VM attached to the port (egress
    traffic).

  The `tunnel_id` will be the value of the ``directions`` dictionary, this is
  the identifier of the ERSPAN or GRE session between the source and
  destination.

.. note::

    There is a big difference in the GRE and ERSPAN id size: GRE has 32 bits
    key size but ERSPAN has only 10 bits for ERSPAN session ID.
    This must be documented and validated on the API.

.. warning::

    It is not possible to create tunnel (GRE or ERSPAN) with the same
    tunnel_id to the same remote_ip from the same port with OVN. This means
    that the tunnel_id can't be the same for the 2 directions.

.. note::

    Both GRE and ERSPAN handle the fragmentation, so if the mirrored traffic's
    packet size with the extra headers is bigger than the MTU on the interface,
    the packet in the tunnel will be sent fragmented.

.. note::

    * For the GRE type mirroring 8 octet extra header is added over IP headers.

    * For ERSPAN 8 octet is added for GRE, 8 octet is added for ERSPAN and
      an extra trailing 4 byte CRC is added, so in summary 20 octets extra
      header is added in this case.

The proposed API definition::

    mirror_types_list = ['erspan', 'gre']

    RESOURCE_ATTRIBUTE_MAP = {
        'tap_mirror': {
            'id': {
               'allow_post': False, 'allow_put': False,
               'validate': {'type:uuid': None}, 'is_visible': True,
               'primary_key': True},
            'name': {
                'allow_post': True, 'allow_put': True,
                'validate': {'type:string': None},
                'is_visible': True, 'default': ''},
            'description': {
                'allow_post': True, 'allow_put': True,
                'validate': {'type:string': None},
                'is_visible': True, 'default': ''},
            'port_id': {
                'allow_post': True, 'allow_put': False,
                'validate': {'type:uuid': None},
                'enforce_policy': True, 'is_visible': True},
            'directions': {
                'allow_post': True, 'allow_put': False,
                'validate': 'type:dict': {
                    'IN': {'type:integer': None, 'default': None, 'required': False},
                    'OUT': {'type:integer': None, 'default': None, 'required': False}},
                'is_visible': True},
            'remote_ip': {
                'allow_post': True, 'allow_put': False,
                'validate': {'type:ip_address': None},
                'is_visible': True},
            'mirror_type': {
                'allow_post': True, 'allow_put': False,
                'validate': {'type:values': mirror_types_list},
                'is_visible': True,},
        }
    }

DB Impact
---------

To persist the new `tap_mirror` in the DB, a new table ``tapmirrors`` is
needed::

        op.create_table(
            'tapmirrors',
            sa.Column('id', sa.String(length=db_const.UUID_FIELD_SIZE),
                      primary_key=True, nullable=False),
            sa.Column('project_id', sa.String(
                      length=db_const.PROJECT_ID_FIELD_SIZE), nullable=False),
            sa.Column('name', sa.String(length=db_const.NAME_FIELD_SIZE),
                      nullable=True),
            sa.Column('description', sa.String(
                      length=db_const.DESCRIPTION_FIELD_SIZE), nullable=True),
            sa.Column('port_id', sa.String(db_const.UUID_FIELD_SIZE),
                      nullable=False, unique=True),
            sa.Column('directions', sa.String(255), nullable=False),
            sa.Column('remote_ip', sa.String(db_const.IP_ADDR_FIELD_SIZE)),
            sa.Column('mirror_type', mirror_type_enum, nullable=False)
        )

This also means that a new ``TapMirror`` DB model will be added.

OVN driver for mirroring
------------------------

The following shows the API call to create a tap_mirror based on version
1 ERSPAN and the corresponding backend changes::

    $ # REST API operation
    $ curl -g -i -X POST http://<host_ip:9696>/networking/v2.0/taas/tap_mirrors \
      -d '{"tap_mirror": {"name": "mirror1", "port_id": "54c4b09f-8b3d-4685-b66d-ce22c67956a9",
                          "directions": {"OUT": 42}, "remote_ip": "100.109.0.142",
                          "mirror_type": "erspan"}}'

    $ # backend changes
    $ sudo ovn-nbctl mirror-list
    mirror_out_297b12c0-e9a5-11ed-9f90-07946c615270:
      Type     :  erspan
      Sink     :  100.109.0.142
      Filter   :  from-lport
      Index/Key:  42

    $ sudo ovs-vsctl show
        Bridge br-int
            ...
            Port ovn-my_mirror2
                Interface ovn-mirror_out_297b12c0-e9a5-11ed-9f90-07946c615270
                    type: erspan
                    options: {erspan_idx="42", erspan_ver="1", key="42", remote_ip="100.109.0.142"}

.. note::

    Using GRE is very similar: the OVN mirror type will be GRE and the
    OVS port type will be GRE.

Description of the above port options:

* ``erspan_idx`` is a in hex, and it is the index field in ERSPAN header.

* ``erspan_ver`` is the version, for version 1 erspan it is 1.

* ``key`` is SpanID or Session ID field in the ERSPAN header.

.. note::

    ERSPAN ``index`` field is not specified as OVN doesn't allow to set it
    separately from the ``key-tunnel_id`` field. When OVN allows the setting
    of it, a future  TAAS change can handle it.

Example packet headers from Wireshark::

    Generic Routing Encapsulation (ERSPAN)
        Flags and Version: 0x1000
        Protocol Type: ERSPAN (0x88be)
        Sequence Number: 18
    Encapsulated Remote Switch Packet ANalysis Type II
        0001 .... .... .... = Version: Type II (1)
        .... 0000 0000 0000 = Vlan: 0
        000. .... .... .... = COS: 0
        ...0 0... .... .... = Encap: Originally without VLAN tag (0)
        .... .0.. .... .... = Truncated: Not truncated (0)
        .... ..00 0000 0010 = SpanID: 42
        0000 0000 0000 .... .... .... .... .... = Reserved: 0
        .... .... .... 0000 0000 0000 0010 0000 = Index: 66

    Index: 66: this is from ``erspan_idx=42``. Note that OVN's ``mirror-add``
    command (and the OVN ovn-nb.ovsschema) accepts one `index` parameter and
    uses that for OVS' ``key`` and ``erspan_idx`` parameters, but erspan_idx
    is a hex value, so this is probably a bug or not fully considered thing
    in OVN.

    SpanId: 42: this is from ``key=42``

With OVN to mirror both ingress and egress traffic of the source port
2 mirrors must be created (as the OVN mirror can have only ``from-lport`` or
``to-lport`` as direction), and attached to the port
(``logical-switch-port``), one with ``filter=from-lport`` and one
with ``filter=to-lport``.

.. note::

    OVN implementation of mirroring allows the user to create a
    mirror by selecting a direction as ``filter`` with ovn-nbctl,
    or via ovsdb (see `ovn-nb.ovsschema mirror table`_)

So if the user creates a tap_mirror with direction ``IN`` the filter will be
``to-lport``, if ``OUT`` the filter will be ``from-lport`` and in case of
both ``IN`` and ``OUT`` in the ``directions`` dictionary 2 mirrors will be
created one with ``to-lport`` and one with ``from-lport``.

The above means that in case of mirroring both ingress and egress traffic
tap-as-a-service will create 2 ERSPAN or GRE ports on br-int for each
tap_mirror.

OVS driver changes
------------------

To keep consistency between the 2 drivers, this specification proposes to use
GRE and ERSPAN version 1 for OVS drive also.

The end-to-end call will look like this::

    $ # REST API operation
    $ curl -g -i -X POST http://<host_ip:9696>/networking/v2.0/taas/tap_mirrors \
      -d '{"tap_mirror": {"name": "mirror1", "port_id": "54c4b09f-8b3d-4685-b66d-ce22c67956a9",
                          "direction": "IN", "remote_ip": "100.109.0.142", "tunnel_id": "42",
                          "mirror_type": "erspan"}}'

    $ # Backend changes
    $ sudo ovs-vsctl show
        Bridge br-tap
            ...
            Port mirror_in_ed6046d
            Interface mirror_in_ed6046d
                type: erspan
                options: {erspan_idx="42", erspan_ver="1", key="2", remote_ip="100.109.0.84"}

    $ sudo ovs-ofctl dump-flows  br-tap
       ...
       ... priority=20,dl_dst=fa:16:3e:d3:3a:d1 actions=output:"mirror_in_ed6046d"

For details on what the port properties ``key``, ``erspan_idx`` mean, see
`OVN driver for mirroring`_ .

For the details on how the case will be handled when both ``IN`` and ``OUT``
direction will be selected by the user see `OVN driver for mirroring`_.
OVS driver will create 2 OVS ports (type=erspan or type=gre) for the 2
directions like OVN does.

For the two directions 2 different flows will be installed on `br-tap` with
different output port in the action field::

    $ # Direction IN
    $ sudo ovs-ofctl dump-flows  br-tap
    ...
    ... priority=20,dl_dst=fa:16:3e:d3:3a:d1 actions=output:"mirror_in_ed6046d"

    $ # Direction OUT
    $ sudo ovs-ofctl dump-flows  br-tap
    ...
    ... priority=20,dl_src=fa:16:3e:d3:3a:d1 actions=output:"mirror_out_ed6046d"

Out of Scope
============

This specification is not proposing to make the OVN driver fully compatible
with the current OVS or SRIOV driver. So the proposed OVN driver will
implement only ERSPAN.

To make OVN driver fully feature compatible with the current OVS or SRIOV
driver can be part of a future specification.

Implementation
==============

Assignee(s)
-----------

* Lajos Katona (~lajoskatona) <lajos.katona@est.tech>, <katonalala@gmail.com>

Work Items
----------

* Add new REST API extension for tap-as-a-service, neutron-lib and
  tap-as-a-service changes.

* Change tap-as-a-service db schema accordingly.

* Adapt ovsdbapp to make it possible to manipulate both ovsdb and ovn-northd
  and create mirrors.

* Change OVS driver.

* Create a new ERSPAN only OVN tap-as-a-service driver.

* Adapt the documentation.

* Implement the necessary tests.

  * end-to-end test in tempest can be done using Floating IPs.

* Adapt OpenstackSDK and the necessary CLI code.

* Adapt Heat to make it possible to create ERSPAN mirrors.

References
==========

.. _ERSPAN draft from Cisco: https://datatracker.ietf.org/doc/id/draft-foschiano-erspan-02.txt
.. _OVS basic configuration: https://docs.openvswitch.org/en/latest/faq/configuration/
.. _OVS protocol header fields: http://www.openvswitch.org//support/dist-docs/ovs-fields.7.txt
.. _erspan NEWS update commit: https://github.com/openvswitch/ovs/commit/4ee9f056871872c3758abd291ccba9710b0c0479
.. _OVN commit that introduced mirroring: https://github.com/ovn-org/ovn/commit/323f978cbf4599568fcca9edec8ed53c076d2664
.. _OVN 22.12 nbctl man page: https://www.ovn.org/support/dist-docs-branch-22.12/ovn-nbctl.8.html
.. _tap-as-a-service API reference: https://docs.openstack.org/api-ref/network/v2/index.html#tap-as-a-service
.. _ovn-nb.ovsschema mirror table: https://github.com/ovn-org/ovn/blob/v22.12.0/ovn-nb.ovsschema#L309-L324
