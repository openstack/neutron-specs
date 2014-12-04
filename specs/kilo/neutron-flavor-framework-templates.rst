..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Flavor framework - Templates and meta-data
==========================================

https://blueprints.launchpad.net/neutron/+spec/neutron-flavor-framework-templates

The Flavor Framework spec introduces a static framework for creating different
feature levels for a service.  This proposal adds the ability to have the
objects being created for the service, influence the flavor's behavior.

Problem Description
===================

The original flavor framework allows operators to create custom feature levels
for a given service.  But, some features require per-instance data to be
effectively deployed.

An example would be a load balancer that offers page caching.  With flavors,
you can specify one flavor for no page caching, and another for page caching
everything.  But, full page caching often breaks legacy applications, and
certain URLs often need to be exempted, on a per load-balancer basis, such
as "do page caching, but not for /legacy/bank/thingie".

Another example would be an operator choosing to enable DDoS protection for a
certain level of load balancer, but security features like those often need
to have a per-instance whitelist for the end-admin to be able to quickly deal
with false positives.

Proposed Change
===============

This proposal suggests two changes to the existing flavor framework:

* Add a model/table for "flavor object meta-data", which is meta-data that
  is stored for any given flavor-enabled neutron object, back-referenced with
  that objects' UUID, and a mixin (/me hides) that adds a "flavor_meta" attribute
  to said object.

* In the "metadata" field passed to drivers, support jinja templating syntax,
  with macro substitution available for the neutron model attributes or the
  above per-instance meta data.  Macro substitution will happen before the
  resulting "metadata" is passed to the backend plugin/driver, so drivers
  do not need to be aware of this templating.

This is intended to be a temporary mechanism for operators to use to enable
certain features that are otherwise missing from openstack, without directly
modifying neutron.

Examples:

With the static flavor framework, you might have a flavor named "AwesomeSauce",
which includes load balancing and DDoS protection, and the driver metadata might
look like this:

::

  {
    vendor_z23_ddos_protection: true
  }

and that is exactly what would be passed to the z23 lbaas driver.  With the
templating syntax, that example becomes:

::

  {
    vendor_z23_ddos_protection: true
    z23_funky_whitelist: [ {{ meta.whitelist }} ]
  }

With a corresponsing flavor_meta field on the LoadBalancer object being:

::

  {
    whitelist: "1.2.3.4, 8.8.8.8"
  }

and the resulting metadata passed to the driver would be:

::

  {
    vendor_z23_ddos_protection: true
    z23_funky_whitelist: [ '1.2.3.4', '8.8.8.8' ]
  }

Data Model Impact
-----------------

One new logical model will be added to the database.

::

  FlavorMeta
    id: uuid
    object_id: uuid
    key: string
    value: string

The ServiceProfile model will be modified to add an "allowed_meta_keys"
attribute, that when taken in conjunction with the template that can be
defined in the "metadata" attribute, defines the allowed additional of the
data that the end user can submit as flavor_meta data.

The following existing models will be modified with a 'meta' method which
will lookup associated FlavorMeta data:

LoadBalancer

As more features are made flavor aware, their root models should add the meta
method (mixin?).

REST API Impact
---------------

Supporting flavor-enabled models will add an attribute for flavor_meta.
Setting this attribute will create an associated FlavorMeta model.

+------------+-------+---------+---------+------------+--------------+
|Attribute   |Type   |Access   |Default  |Validation/ |Description   |
|Name        |       |         |Value    |Conversion  |              |
+============+=======+=========+=========+============+==============+
|flavor_meta |list   |RW, all  |''       |list        |per-object    |
|            |       |         |         |key/value   |meta data     |
+------------+-------+---------+---------+------------+--------------+

Resource: /service_profile

+-----------------+-------+---------+---------+------------+--------------+
|Attribute        |Type   |Access   |Default  |Validation/ |Description   |
|Name             |       |         |Value    |Conversion  |              |
+=================+=======+=========+=========+============+==============+
|allowed_meta_keys|list   |RW, admin|''       |list        |keys for      |
|                 |       |         |         |            |flavor_meta   |
+-----------------+-------+---------+---------+------------+--------------+


Security Impact
---------------

None

Notifications Impact
--------------------

None

Other End User Impact
---------------------

This is an additional hook for operators to enable. If present, users can
interact with the flavor_meta attribute via the API.

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

Services/models that want to support flavors and this templating mechanism will
need to add the appropriate model entry and db migration.

Community Impact
----------------

This change allows operators greater flexibility in enabling advanced services
within the Neutron framework.

Many of the features that can be enabled via this mechanism can and should be
added as features in their corresponding service projects. This proposal
specifies a mechanism for operators and vendors to enable features which,
for whatever reason, are not being addressed by the OpenStack dev community yet.
This could be because the feature is too niche to be broadly applicable,
the release timeline is too far in the future, or a good open-source
alternative simply does not exist yet.  This proposal adds a mechanism to
handle the interim, until the mainline feature is supported.

Best practices:

* If you need to migrate flavors, the prior decision is that if you have to
  create a new service entity with a new flavor. They can not be modified
  at this time. This applies to all flavors, not just this proposal.

* If an equivalent community feature exists, it is encouraged that operators
  use that feature/give feedback/contribute. The recommended best practice
  for drivers is for features exposed in the main-line to override features
  exposed through flavor meta-data

* It should be self-evident that enabling a feature offered by a single vendor
  does two things for the Operator:  1. It allows the operator to utilize that
  feature and expose it to their users. 2. If users use said feature, the
  operator may find it more difficult to switch vendors if they choose to do
  so at a later date. Furthermore, for features offered by multiple vendors,
  care must be taken in abstracting those features, if the operator desires to
  support multiple vendors.

Alternatives
------------

* The first alternative is to do nothing.  This results in what many vendors are
  doing today, which is to brew up proprietary neutron solutions in order to
  expose more advanced features.  This results in inconsistent solutions for
  operators, more difficulty tracking trunk, and vendor lock-in.

* Another alternative is the same as this proposal minus the templating on the
  flavor metadata.  Since the flavor metadata is tied to a particular driver,
  and thus vendor specific, removing the templating would force vendors to expose
  vendor specific goo to their end users.  In addition, since multiple service
  profiles (drivers/vendors) can be used to implement a single flavor,
  not having templating would mean that that multiple backend support would break
  unless those backends supported the exact same back-end metadata, which
  is unlikely and impractical.

* Finally, the most straightforward alternative is to implement each feature
  natively into the services API.   In the aforementioned page caching example,
  add page caching as a feature in the API, with a URL exception list, and
  wait for all drivers to implement to that backend.  This adds maintenance
  and development load to the entire community, and means that Neutron will have
  a built-in lag for adding new features that appear in the marketplace.

Implementation
==============

Assignee(s)
-----------

https://launchpad.net/~dougwig

Work Items
----------

* Add new data model and migration.

* Add mixin for neutron models, with 'meta' method.

* Add mixin to LoadBalancer, and its REST attributes.

* Modify flavors to apply jinja templating to service profile metadata attr.

Dependencies
============

* Main flavors framework
* LBaaS v2


Testing
=======

Tempest Tests
-------------

Flavor tests need to be enhanced to include per-object meta-data and some basic
templated flavor metadata, and verify that substituted data is passed to
the backend.

Functional Tests
----------------

Tests to verify the flavor_meta field in models, and that the jinja substitution
is happening properly in the flavors code before being passed to backends.

API Tests
---------

Modify flavor API tests to include flavor_meta field for objects.


Documentation Impact
====================

User Documentation
------------------

This change is invisible to end users.

Developer Documentation
-----------------------

Deployers will need documentation for the new API fields and the templating syntax.

References
==========

* Flavors framework - https://review.openstack.org/#/c/102723
