..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Extend management features of L3 metering API
=============================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/l3-metering-mgnt-ext

This blueprint aims to extend the current metering API in order to help
administrators to manage the metering labels/rules.


Problem description
===================

Currently the administrator has to associate metering label for each tenant
by the hand, there is no way to add a default metering label/rules which will
be automaticaly associated with all current tenants and also with the tenants
that will be created.

Use cases:

- The administrator wants to get traffic counters from the public for a billing
  purpose for all current tenants and for the future tenants. Currently each
  time a tenant is added the administrator has to create a new label for it.

- The administrator wants to get traffic counters from service networks for
  monitoring purpose, for instance from a provider network which exposes
  physical resources (ex: db, storage)


Proposed change
===============

The goal is to extend the current API to add an extra parameter for the
labels creation which specifies whether the label will be shared by all tenants
or not. A such label will be shared by all current tenants and also by the
tenants which will be created.

Alternatives
------------

An alternative could be to add an extra configuration file which set some
metering labels/rules at the metering agent startup.

Data model impact
-----------------

An extra field will be added to the MeteringLabel data model to specify whether
a label is shared or not. By default a metering label will be not shared.
The current labels will be unchanged.

REST API impact
---------------

A new shared attribute will be introduced to the current MeteringLabel model:

+----------+-------+---------+---------+------------+--------------+
|Attribute |Type   |Access   |Default  |Validation/ |Description   |
|Name      |       |         |Value    |Conversion  |              |
+==========+=======+=========+=========+============+==============+
|shared    |bool   |RW, admin|false    |N/A         |              |
+----------+-------+---------+---------+------------+--------------+

Security impact
---------------

No change, only admin users are allowed to create/delete labels/rules.

Notifications impact
--------------------

None

Other end user impact
---------------------

The shared parameter will be exposed to the end user through the neutron
client, ex:

neutron meter-label-create testlabel --shared

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

Sylvain Afchain <sylvain-afchain>

Work Items
----------

The work is split up into two parts:

1. API, Data model, Metering service plugin.

2. Neutron client update.


Dependencies
============

None


Testing
=======

For tempest test coverage, new API tests for the shared parameter will be
provided.


Documentation Impact
====================

Documentation and examples for the shared parameter will be provided.


References
==========

None
