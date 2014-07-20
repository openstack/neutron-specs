..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================================
Add the capability to sync neutron resources to the N1KV VSM
============================================================

https://blueprints.launchpad.net/neutron/+spec/cisco-n1kv-full-sync

The purpose of this blueprint is to add support to synchronize the state of
neutron database with Cisco N1KV controller (VSM).

Problem description
===================

Today if there is any inconsistency in the state of neutron and the VSM
databases, there is no way to push all the neutron configuration back into the
VSM.


Proposed change
===============

The proposed change is to introduce support for state synchronization between
neutron and VSM in the N1KV plugin.

Creates and updates of resources are rolled back in neutron if an error is
encountered on the VSM. In case the VSM loses its config, a sync is triggered
from the neutron plugin. The sync compares the resources present in neutron DB
with those in the VSM. It issues creates or deletes as appropriate in order to
synchronize the state of the VSM with that of the neutron DB.

Deletes cannot be rolled back in neutron. If a resource delete fails on the VSM
due to connection failures, neutron will attempt to synchronize that resource
periodically.

The full sync will be triggered based on a boolean config parameter
i.e. enable_sync_on_start. If "enable_sync_on_start" is True, neutron will
attempt to synchronize its state with that of VSM. If "enable_sync_on_start"
is set to False, neutron will not attempt any state sync.
This blueprint will introduce a bare minimum capability of synchronizing
resources. It does not cover out-of-sync detection logic.

Alternatives
------------

None

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

None

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  abhraut

Work Items
----------

 * Add logic to the plugin module to push database state to the VSM.
 * Add configuration parameter in cisco_plugins.ini

Dependencies
============

None

Testing
=======

Unit tests will be provided.

Documentation Impact
====================

Update documentation to reflect the new configuration parameter.

References
==========

None
