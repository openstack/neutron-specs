..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================
L3/BGP - Allow to configure the BGP peer connect mode
=====================================================

https://bugs.launchpad.net/neutron/+bug/2040001

This RFE intends to implement a configuration option to allow the Operator to
change the Neutron Dynamic Routing (n-d-r) speaker's default BGP connect mode
to all supported modes: ACTIVE, PASSIVE or BOTH.


Problem description
===================

When using n-d-r, the speaker's current behaviour is to work in ACTIVE mode
which will not create a local TCP socket on TCP 179 port. Therefore, in the
switch's BGP session, it needs to be configured as PASSIVE and not initiate
TCP connections.

Operators could have a switch brand that does not follow correctly the RFC 4271
[1]_ regarding with the BGP FSM (8.2).

When the n-d-r and the switch has a health BGP session, the Operator could run
a maintenance window in the switch or in the n-d-r agent and the session could
flap and go down for while and on the switch side, the BGP session state can
change to IDLE.

According to RFC 4271(8.2.2.), when a BGP session is in IDLE state, it refuses
all inbound BGP connection attempts, and starts a TCP connection to the
peer(n-d-r) and n-d-r speaker's will not answer because n-d-r is in ACTIVE mode
and has no listen port for this. If the switch correctly follow the RFC
regarding BGP PASSIVE mode when enabled, it will ignore it and accept the
incoming connection from the n-d-r and the connection will be restablished,
regardless of the IDLE state.

That is definitely a switch-side issue but when having this situation, we could
allow the Operator to change the default behaviour of n-d-r speaker's from
ACTIVE and set it as PASSIVE or BOTH using a n-d-r configuration option and
then open the local TCP socket on the TCP 179 port in order to anwser incoming
TCP connections.


Proposed change
===============

This proposal implements a configuration option to allow the Operator to change
the n-d-r speaker's default BGP connect mode from ACTIVE to PASSIVE or BOTH.
For Operators who do not have the problem described above, it is not necessary
to change anything as the BGP connection mode will remain as ACTIVE.

To change the n-d-r speaker's default BGP connect mode, a new configuration
option should be enabled via [BGP] section in the n-d-r agent configuration
file.

.. code::

  * ``bgp_connect_mode = both``

The `bgp_connect_mode` values supported would be:

* `active` initiate the TCP connection to establish the BGP session and do not
  require to open a local TCP socket on port 179. The default value for
  `bgp_connect_mode` must be `active` to keep the current behaviour.

* `passive` open a local TCP socket on port 179 and wait for an incoming TCP
  connection from the BGP peer.

* `both` open a local TCP socket on port 179 and wait for an incoming TCP
  connection from the BGP peer. Additionally, a connection could be initiated
  to establish the BGP session with the peer.

Neutron Dynamic Routing Impact
------------------------------

When the n-d-r agent is started:

* if the option `bgp_connect_mode` is ommited or has the value `active` the BGP
  connection mode will remain as ACTIVE and it will not create a local TCP
  socket on port 179.

* if the option `bgp_connect_mode` has the value `passive` the BGP connection
  mode will be used as PASSIVE and a local TCP socket on port 179 will be
  opened.

* if the option `bgp_connect_mode` has the value `both` the BGP connection mode
  will be used as BOTH and a local TCP socket on port 179 will be opened.

Because n-d-r could start the agent as an unprivileged user, systemd will
require for the n-d-r agent service to have net bind capability to allow for
creation of the local TCP socket on port 179.

Security Impact
---------------

The Operator will have to ensure the safety of the n-d-r agent when using
`bgp_connect_mode` as `passive` or `both`, and create local firewall rules to
only allow incoming TCP connection from BGP peers.


Implementation
==============

Assignee(s)
-----------

Primary assignees:
* Tiago Pires <tiagohp@gmail.com>
* Roberto Bartzen Acosta <rbartzen@gmail.com>

Work Items
----------

* Implement a new configuration option.

* Implement relevant unit and functional tests using the existing facilities
  in Neutron Dynamic Routing.

* Write documentation.


Testing
=======

* Unit/functional tests.


Documentation Impact
====================

User Documentation
------------------

* Information about the `bgp_connect_mode` configuration option.


References
==========

.. [1] https://datatracker.ietf.org/doc/html/rfc4271