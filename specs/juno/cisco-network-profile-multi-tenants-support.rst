..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================================
Ability to assign cisco nw profile to multi-tenants in single request
=====================================================================

https://blueprints.launchpad.net/neutron/+spec/cisco-network-profile-multi-tenants-support


Add the support to assign a cisco network profile to multiple tenants in a single request



Problem description
===================

Currently with Cisco N1kv plugin, admin is only able to assign cisco network profile to one
tenant in a request. So admin has to send multiple requests to assign a cisco network profile
to multiple tenants.


Proposed change
===============

The proposed change is to make the create network profile and update network profile functions to
take a list of tenant ids, and then update the profile-tenant binding info accordingly.


Alternatives
------------

The Alternative way is to send multiple requests, assigning to one tenant per request.
This alternative is obviously too tedious for admin users, especially when a admin has a lot
of tenants to manage.

Data model impact
-----------------

None

REST API impact
---------------

for network_profile resource, 'add_tenant' and 'remove_tenant' attributes are changed to
'add_tenants' and 'remove_tenants'. And Instead of taking one tenant id, now they are taking
a list of tenant ids.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

There is a corresponding change in python-neutronclient:
In cisco network profile create cli, admin can add repeated --add-tenant
option;
In cisco network profile update cli, admin can add repeated --add-tenant and
repeated --remove-tenant option

in horizon:
admin can select multiple tenants during creating cisco network profile;
admin can select or deselect multiple tenants during updating cisco network profile.


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
  fenzhang

Work Items
----------

- modify create and update network profile functions in neutron/plugins/cisco/db/nikv_db_v2.py file
- modify network profile attributes in neutron/plugins/cisco/extensions/network_profile.py file

Dependencies
============

blueprint in python-neutronclient and horizon:
https://blueprints.launchpad.net/horizon/+spec/cisco-network-profile-multi-tenants-support
https://blueprints.launchpad.net/python-neutronclient/+spec/cisco-network-profile-multi-tenants-support

Testing
=======

The UT test cases will be added to cover this change.

Documentation Impact
====================

None

References
==========

None
