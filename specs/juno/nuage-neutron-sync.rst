=====================================================================
Add the functionality to sync resources between Neutron and Nuage VSD
=====================================================================

https://blueprints.launchpad.net/neutron/+spec/nuage-neutron-sync

The purpose of this blueprint is to add functionality to sync resources
between Neutron and Nuage VSD(Virtualized Services Directory).

Problem description
===================

If there is any inconsistency in the state of neutron and the VSD
database, there is no way to sync the resources between them.

Proposed change
===============

The proposed change is to introduce support for state synchronization between
neutron and VSD in the Nuage plugin

In case Nuage VSD loses its database or the database is restored using a backup
copy, there is a possibility that the resources in Neutron  and VSD are not in sync
and this can lead to unexpected behavior. The sync will run as part of Nuage plugin
at regular intervals.

Neutron is considered as the master. The sync logic will get the state from Neutron and will
send it to VSD. VSD will compare this state with its own state and if there are less resources
in VSD as compared to Neutron, the resources will be created in VSD. In case there are more
resources in VSD, they will be deleted from VSD.

Sync will be controlled by two config parameters "enable_sync" and "sync_interval".
If the "enable_sync" parameter is set to true, the sync will start when Neutron
server starts and will try to sync resources. The sync will run periodically at an
interval specified for the "sync_interval" parameter.

Assumption:
1. If the resource exists in VSD, all of its attributes are in sync with the corresponding
Neutron resource.

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
 sayaji15

Work Items
----------

 * Modify the nuage plugin to add sync logic

Dependencies
============

None

Testing
=======

Unit tests will be provided.

Documentation Impact
====================

None

References
==========

None
