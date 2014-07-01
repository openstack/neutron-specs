..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================================================
Config option to control visibility of cisco-policy-profile resources for tenants
=================================================================================

https://blueprints.launchpad.net/neutron/+spec/cisco-policy-profile-visibility

Config option to control visibility of cisco-policy-profile resources for tenants



Problem description
===================

Currently, cisco-policy-profile resources must be explicitly assigned to tenants.


Proposed change
===============

This proposes adding a config option to Cisco N1KV plugin to control the visibility
of these resources.

The options may be True/False.
Selecting True would allow tenants access to all the cisco-policy-profile resources
without the need for explicit assignment.
The False option would turn off the visibility and require explicit assignment of
a cisco-policy-profile resources to a tenant.
Default value is True

Alternatives
------------

Admin needs to explicitly assign cisco-policy-profile resources to tenants.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

A new visibility option will be added in cisco_plugin.ini file.
Its default value will be True, which means it's enabled by default.
The change will take immediate effect after it's merged.

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  fenzhang

Work Items
----------

- add visibility option in cisco_plugins.ini file
- retrieve this option and control the visibility of cisco-policy-profile resources
  based on its value in neutron/plugins/cisco/db/n1kv_db_v2.py file


Dependencies
============

None


Testing
=======

The UT test cases will be added to cover this change.


Documentation Impact
====================

Will need to document it in user manual


References
==========

None
