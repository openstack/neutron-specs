====================================
Allow to specify floating IP address
====================================

Launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/allow-specific-floating-ip-address

Problem Description
===================

IP Address of floating ip is automatically allocated from IP pool.

In some cases, user want to specify a IP Address of floating ip.

Proposed Change
===============

Adding the floating ip address as an attribute to the API for creating floating ip. An attribute name is floating_ip_address.

Attributes of API are limited by "policy.json". By default, Non-admin user cannot use floating_ip_address. Admin can use it. ("rule:admin_only")

Alternatives
------------

None

Data Model Impact
-----------------

None

REST API Impact
---------------

* floating_ip_address will be allowed to POST /v2.0/floatingips
* It is not updatable. i.e. cannot use for PUT /v2.0/floatingips

The attribute map will be changed as the following.

.. code-block:: python

  RESOURCE_ATTRIBUTE_MAP = {
      ...
      'floatingips': {
          ...
          'floating_ip_address': {'allow_post': True, 'allow_put': False,
                                  'validate': {'type:ip_address_or_none': None},
                                  'is_visible': True, 'default': None,
                                  'enforce_policy': True},
          ...
      },
      ...
  }

Security Impact
---------------

An administrator of OpenStack should consider whether this feature is exposed to tenant user.

If users have a permission to use this feature, Users can control public ip addresses.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

None

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

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  fujioka-yuuichi-d

Work Items
----------

1. Add attribute to API
2. Create test case on tempest

Dependencies
============

None

Testing
=======

Tempest Tests
-------------

The following tests will be added.

* API Testing

  * Allowed users can use the new parameter

* Scenario Testing

  * A created floating ip address is reachable

Functional Tests
----------------

The following tests will be added.

* Allowed users can use the new parameter

API Tests
---------

The following tests will be added.

* All patterns that parameter is passed/not passed.

Documentation Impact
====================

User Documentation
------------------

New attributes will be added to API documentation.

Developer Documentation
-----------------------

None

References
==========

https://review.openstack.org/#/c/70286/
