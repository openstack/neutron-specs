..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Big Switch - Cleanup modules
============================

https://blueprints.launchpad.net/neutron/+spec/bsn-module-cleanup

The Big Switch plugin and ML2 have shared code that lives inside of the Big
Switch plugin file. This creates strange side effects when importing the plugin
from the ML2 agent when database modules are loaded. The Big Switch modules
need to be separated out better to prevent these cases and to cleanly express
what is shared between the ML2 driver and the plugin.


Problem description
===================

Importing the Big Switch plugin from the Big Switch ML2 driver requires some
ugly hacks since it causes all of the Big Switch plugin DB modules to be
imported.[1] This very tight integration makes updating the ML2 driver or the
plugin a delicate process because we have to make sure one doesn't break the
other.


Proposed change
===============

The shared code paths should be removed from the Big Switch plugin module and
placed into their own. Some of the shared methods should also be refactored to
make the two use cases easier to see and customize.

Sections to be moved to a new module:

- The entire NeutronRestProxyV2Base class.[2]


Shared methods to cleanup/refactor:

- The async_port_create method.[3] The name is misleading because it's not
  asynchronous. It just has the ability to be called asynchronously because it
  handles failures.
- The _extend_port_dict_binding method.[4] It has to have special logic to
  determine if it's being called from the driver or the plugin. There should be
  a different function for each one for the conditional parts and then the base
  method can contain the shared logic.
- Rename _get_all_subnets_json_for_network since it's not really returning
  JSON.[5]
- Move imports out of methods.[9][10]


Other:

- Remove the version code since it's not used any longer.[6][7]
- Move the router rule DB module into the DB folder.[8]


Alternatives
------------

Let the code be ugly


Data model impact
-----------------
N/A

REST API impact
---------------
N/A


Security impact
---------------
N/A


Notifications impact
--------------------
N/A

Other end user impact
---------------------
N/A

Performance Impact
------------------
N/A

Other deployer impact
---------------------
N/A

Developer impact
----------------

N/A

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  kevinbenton

Work Items
----------

* Separate code into modules
* Refactor shared methods between ML2 and plugin to make demarcation clear


Dependencies
============

N/A

Testing
=======

Existing tests should cover this refactor.

Documentation Impact
====================

N/A

References
==========

1. https://github.com/openstack/neutron/commit/1997cc97f14b95251ad568820160405e34a39801#diff-101742a6f187560f6c12441b8dbfb816
2. https://github.com/openstack/neutron/blob/c82e6a4d33c2a8dc51260fab4ad0cb87805b49de/neutron/plugins/bigswitch/plugin.py#L159
3. https://github.com/openstack/neutron/blob/c82e6a4d33c2a8dc51260fab4ad0cb87805b49de/neutron/plugins/bigswitch/plugin.py#L388
4. https://github.com/openstack/neutron/blob/c82e6a4d33c2a8dc51260fab4ad0cb87805b49de/neutron/plugins/bigswitch/plugin.py#L349
5. https://github.com/openstack/neutron/blob/c82e6a4d33c2a8dc51260fab4ad0cb87805b49de/neutron/plugins/bigswitch/plugin.py#L254
6. https://github.com/openstack/neutron/blob/c82e6a4d33c2a8dc51260fab4ad0cb87805b49de/neutron/plugins/bigswitch/version.py
7. https://github.com/openstack/neutron/blob/c82e6a4d33c2a8dc51260fab4ad0cb87805b49de/neutron/plugins/bigswitch/vcsversion.py
8. https://github.com/openstack/neutron/blob/c82e6a4d33c2a8dc51260fab4ad0cb87805b49de/neutron/plugins/bigswitch/routerrule_db.py
9. https://github.com/openstack/neutron/blob/c82e6a4d33c2a8dc51260fab4ad0cb87805b49de/neutron/plugins/bigswitch/db/porttracker_db.py#L25
10. https://github.com/openstack/neutron/blob/c82e6a4d33c2a8dc51260fab4ad0cb87805b49de/neutron/plugins/bigswitch/db/porttracker_db.py#L37
