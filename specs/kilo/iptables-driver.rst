..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
New iptables firewall driver
============================

The goal of this spec is to implement new iptables firewall driver which will
be more maintainable, readable and testable with better OOP design.
The functionality of driver is preserved and can achieve the same configuration
as with current IptablesManager and IptablesFirewall.


Problem Description
===================

Nowadays, the biggest problem in current implementation is performance of
IptablesManager discovered by Rally project. The bottleneck in this
particular case is algorithm of parsing the output of iptables-save command
[1]. Other problems are code complexity of IptablesManager and how
IptablesFirewall uses internal attributes of IptablesManager without any
encapsulation on the side of IptablesManager.


Proposed Change
===============

Current IptablesFirewall implementation that manages both IpsetManager and
IptablesManager. IptablesManager uses dictionaries for ipv4 and ipv6 and their
filters. Because of that, IptablesFirewall must know internals of
IptablesManager when creating rules and thus there is no encapsulation. In
comparison to this model I propose follwing:

IptablesDriver
--------------

New IptablesDriver class will be introduced providing API for controlling
netfilter via given CLI. This API will be similar to the CLI. There will be
common class from which other classes will inherit and declare executable and
defaults. e.g. IptablesDriver will contain executable 'iptables' and 'raw,
mangle, filter, nat' tables while Ip6tablesDriver will contain 'ip6tables'
executable and 'raw, mangle, filter' tables.

In order to preserve atomicity when applying rules, similar approach to the
current one will be used. That means required changes to iptables are cached
and applied at once by save/restore mechanisms. In order to improve performance
here, an iptables parser will be implemented giving basic internal structure of
IptablesDriver, described at :ref:`objectmodel` section.

Parser will use regular expressions to distinguish which line is being parsed
and according its state will be created needed object. Suggested expressions
are in :ref:`regularexpression` section. For example: if parser
already hit table "A" then all following chains belong to that table until
new table is hit.

When rules are about to be applied, changes are taken from operations cache in
the same order in which they were added and applied on already parsed data.
This will effectively replace current implementation with _modify_rules()
avoiding expensive mangling of packet counters.

.. _regularexpression:

Regular expressions
-------------------

::

 TABLE_LINE_PATTERN = re.compile(r'^\*(?P<table>[a-z]+)')
 CHAIN_LINE_PATTERN = re.compile(r'^:(?P<chain_name>\S+) (?P<default_rule>\S+)'
                                  ' \[(?P<packets>\d+)\:(?P<bytes>\d+)\]')
 RULE_LINE_PATTERN = re.compile(r'^\[(?P<packets>\d+)\:(?P<bytes>\d+)\] '
                                 '-A (?P<chain>\S+) (?P<rule>.*)')

IptablesFirewall
----------------

As IptablesDriver is designed only to communicate with iptables CLI, logic of
creation of top chains will be moved to new IptablesFirewall. For wrapping
chain names will be created also separate class as this logic doesn't belong to
driver either. The new driver will provide same functionality, API and output
as current IptablesFirewall.

.. _objectmodel:

Object Model
------------

::

 +-----------------------------------------------------------------------+
 |                               IptablesDriver                          |
 +-----------------------------------------------------------------------+
 | - tables: OrderedDict                                                 +----+
 | - operations: list                                                    +--+ |
 +-----------------------------------------------------------------------+  | |
 | - load()                                                              |  | |
 | + apply()                                                             |  | |
 | + add_chain(table_name:str=filter,chain_name:str)                     |  | |
 | + delete_chain(table_name:str=filter,chain_name:str)                  |  | |
 | + set_chain_policy(table_name:str=filter,chain_name:str,policy:str)   |  | |
 | + append_rule(table_name:str=filter,chain_name:str,rule:IptablesRule) |  | |
 | + insert_rule(table_name:str=filter,chain_name:str,rule:IptablesRule, |  | |
 |               position:int=0)                                         |  | |
 | + delete_rule(table_name:str=filter,chain_name:str,rule:IptablesRule) |  | |
 +-----------------------------------------------------------------------+  | |
                                                                            | |
                      +-----------------------------------------------------+ |
                      |                                                       |
                      |                                                       |
                      |                                                       |
   +------------------+----------------+                                      |
   |           IptablesOperation       |                                      |
   +-----------------------------------+                                      |
   | - action: method                  |                                      |
   | - arguments: tuple (args, kwargs) |                                      |
   +-----------------------------------+                                      |
                                                                              |
                                                 +----------------------------+
                                                 |
                                                 |
                 +-------------------------------+---------------------------------+
                 |                          IptablesTable                          |
                 +-----------------------------------------------------------------+
                 | - name: str                                                     |
                 | - chains: OrderedList                                           +--+
                 +-----------------------------------------------------------------+  |
                 | + add_chain(chain_name:str)                                     |  |
                 | + delete_chain(chain_name:str)                                  |  |
                 | + set_chain_policy(chain_name:str,policy:str)                   |  |
                 | + append_rule(chain_name:str,rule:IptablesRule)                 |  |
                 | + insert_rule(chain_name:str,rule:IptablesRule, position:int=0) |  |
                 | + delete_rule(chain_name:str,rule:IptablesRule)                 |  |
                 +-----------------------------------------------------------------+  |
                                                                                      |
                                                                                      |
                           +----------------------------------------------------------+
                           |
                           |
                           |
 +-------------------------+------------------------+
 |                   IptablesChain                  |
 +--------------------------------------------------+
 | - name: str                                      |
 | - rules: list                                    +--+
 | - policy: str                                    |  |
 | - packets: int                                   |  |
 | - bytes: int                                     |  |
 +--------------------------------------------------+  |
 | + append_rule(rule:IptablesRule)                 |  |
 | + insert_rule(rule:IptablesRule, position:int=0) |  |
 | + delete_rule(rule:IptablesRule)                 |  |
 +--------------------------------------------------+  |
                                                       |
                                                       |
                                              +--------+--------+
                                              |  IptablesRule   |
                                              +-----------------+
                                              | - source        |
                                              | - destination   |
                                              | - target: str   |
                                              | - protocol      |
                                              | - in_interface  |
                                              | - out_interface |
                                              | - match: str    |
                                              | - packets: int  |
                                              | - bytes: int    |
                                              +-----------------+

Data Model Impact
-----------------

None

REST API Impact
---------------

None

Security Impact
---------------

None

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

Expected performance boost if iptables contains lot of chains. The performance
will be gained in usage of iptables-save output parsing.

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

None

Developer Impact
----------------

None

Community Impact
----------------

None

Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
    libosvar

Work Items
----------

 1) Make a test with current firewall driver managing iptables
 2) Parser for output of iptables-save
 3) New IptablesDriver driver passing test from 1)
 4) New IptablesFirewall mapping security groups with new IptablesDriver

Dependencies
============

None

Testing
=======

Tempest Tests
-------------

No need of new Tempest tests as this spec doesn't actually implement any new
functionality.


Functional Tests
----------------

Testing current IptablesManager and new IptablesDriver - testing that rules can
block specific ports, ip addresses, incoming and outgoing connections,
translate addresses and ports in nat table.

Some ongoing work can be observed at https://review.openstack.org/#/c/120349/ .

API Tests
---------

None

Documentation Impact
====================

User Documentation
------------------

Operators will be notified about new firewall driver.

Developer Documentation
-----------------------

None


References
==========

[1] https://gist.github.com/anonymous/e3250001212ea8e40e1d
