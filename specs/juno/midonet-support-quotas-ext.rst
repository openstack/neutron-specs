==========================================
Quota extension support for MidoNet plugin
==========================================

https://blueprints.launchpad.net/neutron/+spec/midonet-support-quotas-ext

Add support for quotas extension in MidoNet plugin.

Problem description
===================

Current MidoNet plugin does not support quotas extension, which can be
supported simply by adding "quotas" in supported_extension_aliases
field in the plugin and "quotas" table in the Neutron database.

Proposed change
===============

Add quotas in supported_extension_aliases in the plugin as well as
"quotas" table in the Neutron database.


Alternatives
------------
None

Data model impact
-----------------

"quotas" table will be created in the Neutron Database.


REST API impact
---------------
Users can start using quotas API with MidoNet plugin.


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
None

Developer impact
----------------
This introduces a new DB migration script.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  tomoe

Other contributors:

Work Items
----------

Add "quotas" in supported_extension_aliases
Add "quotas" table in the DB

Dependencies
============
None

Testing
=======

Tempest test tempest.api.network.admin.test_quotas covers this feature.

Documentation Impact
====================
None

References
==========
None
