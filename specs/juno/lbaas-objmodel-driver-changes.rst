..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================
LBaaS Driver Interface changes for new Object Model
===================================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/lbaas-objmodel-driver-changes

The upcoming LBaaS object model and API changes for Juno will require some
driver interface changes, for new objects and backwards compatibility.

New interfaces include create_load_balancer (and the rest of crud) and
create_listener.  Create_vip is going away, and will be supported in a shim
layer for older drivers.  M:N health monitor associations are also going away.

The new object model and API blueprint:
https://blueprints.launchpad.net/neutron/+spec/lbaas-api-and-objmodel-improvement

This blueprint does not cover changes for the new TLS or L7 functionality.


Problem description
===================

With the upcoming LBaaS object model and API changes, we will need new
interfaces for the backend load balancing drivers, and a shim layer so that
things continue to work with older drivers during a depcrecation period.


Proposed change
===============

When loading the driver, neutron will determine if it is a new object model
driver or an older legacy driver, and pick an appropriate implemenation/shim
class depending on which.

The new driver interface methods will be passed objects with attributes,
instead of dictionaries as used in the icehouse drivers.  For expediency's
sake, initially these objects will be the standard neutron sqlalchemy objects,
but full sqlalchemy functionality is not guaranteed in the future.  Basic
attributes and dependent object lookups will be supported long-term.

The new driver interface will be made up of handler classes that implement
create, update, and delete methods.  Example, subject to change:


sample_driver.py::

    class SampleDriver(LoadBalancerAbstractDriver):
        def __init__(self):
            self.load_balancer = MyLoadBalancerManager(self)
            self.listener = MyListener(self)
            self.member = ef.MemberHandler(self)
            self.stats = xyz.foo.StatsGatherer(self)
            self.health_monitor = HMHandler(self)

    class MyLoadBalancerManager(BaseManager):
        def __init__(self, driver):
            self.driver = driver
        def create(self, context, lb_obj):
            print("create")
        def update(self, context, lb_obj):
            print("update")
        def delete(self, context, lb_obj):
            print("delete")


Supported driver handlers:
* load_balancer
* listener
* pool
* member
* health_monitor
* stats

The following older driver methods will be translated/supported.  Differences
from existing behavior are noted next to each method.

* create_vip - combination of one load_balancer and listener object from above.
* update_vip
* delete_vip
* create_pool - first object created
* update_pool
* delete_pool
* stats
* create_member
* update_member
* delete_member
* update_pool_health_monitor
* create_pool_health_monitor
* delete_pool_health_monitor

Limitations of older driver shim:

- Multiple listeners on a load balancer is not supported.
- Multiple pools on a listener is not supported.
- Many-to-many associations for health monitors is not supported.
- Subnet id's on member objects are not supported.

Driver breakage:

- Some references to ldb.Vip will break until their maintainers update the
  relevant driver (embrane, radware).  Tests for those drivers will be
  temporarily disabled until fixed.


Alternatives
------------

One proposed alternative for the backwards compatibility shim layer for the
drivers is to create a completely separate new LBaaS extension/plugin, with
completely separate database tables, and let both exist.  This would involve
duplicate drivers and db tables.  We decided not to use this approach, and
the general implementation plan for the lbaas api/driver changes:

* Create new LBaaS extension using database tables that have merged fields
  between old and new model.
* Change front-end of old extension to "translate" into operations on new model.
* Changer drivers to use new model.
* Deprecate / delete unused old model fields from database.
* Deprecate / delete old front-end.


Data model impact
-----------------

None, refer to object model blueprint.

REST API impact
---------------

None, refer to object model blueprint.

Security impact
---------------

None, same impact as current driver implementations.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

Performance Impact
------------------

None, same impact as current driver implementations.

Other deployer impact
---------------------

Upgrade to new LBaaS will involve a db migration; refer to object model
blueprint.  Older LBaaS drivers will continue to work, except that attempting
to add multiple listeners to a load balancer will fail.

Developer impact
----------------

Current LBaaS driver maintainers will need to eventually update their drivers
to the new interface.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  https://launchpad.net/~brandon-logan
  https://launchpad.net/~dougwig

Work Items
----------

- Modify layer above drivers to call new entry points.
- Modify layer above drivers to call old driver interface methods for older
  drivers, translating the newer object model as needed.
- New abstract_driver with new interfaces.

Updating the reference haproxy driver will be in another blueprint.

Dependencies
============

* https://blueprints.launchpad.net/neutron/+spec/lbaas-api-and-objmodel-improvement


Testing
=======

- Existing tests against older drivers will continue to pass, unless testing
  the M:N health monitor relationship.
- New tests will be added for new entry points will be added when the reference
  driver is updated.
- Broken drivers, as specified in "Proposed change", will result in disabled
  tests until the drivers are fixed.


Documentation Impact
====================

None.

References
==========

* https://blueprints.launchpad.net/neutron/+spec/lbaas-api-and-objmodel-improvement

* https://etherpad.openstack.org/p/1gsTm4GBdu

* https://etherpad.openstack.org/p/juno-lbaas-mid-cycle-hackathon
