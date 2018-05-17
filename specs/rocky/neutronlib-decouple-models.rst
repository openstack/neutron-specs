..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================================
Decoupling database Resource Model imports/access for neutron-lib
=================================================================

This work is not related to an enhancement request and therefore doesn't have
a RFE. Rather the intent herein is to discuss how we decouple ``neutron.db``
and related database imports/access as part of the overall neutron-lib effort.

Current neutron database access can be broken into the following high-level
categories:

- Database API & Utilities
- Core & Extension Database Mixins
- Database Resource Models
- Database Migration

As the database access patterns span a wide range of logic/code, a set of specs
will be proposed each focusing on a single access pattern.

This spec specifically addresses Database Resource Model access.

For current neutron-lib related blueprints, see [1]_ and [2]_.


Problem Description
===================

As part of our neutron-lib effort, we need to decouple out-of-tree networking
projects that depend on (e.g. ``import``) neutron. However, today a large
number of neutron consumers import database related neutron modules including
models [3]_ [4]_.

Access to database models by consumers is typically used for:

- Defining relationships between neutron and other project models. Note that
  while string names can be used when defining a relationship, this is not
  a viable solution for all cases due to [8]_.
- Building queries using models and/or their fields. With the introduction of
  versioned objects, consumers can effectively query using methods on objects.
  Therefore the deliverables of this spec enable such an approach by making
  versioned objects accessible via neutron-lib.
- Importing models to have them available for database tools. This includes
  database migration tools, so while this spec doesn't directly solve
  the migration access, it needs to lay a foundation for it (covered in a
  separate spec).

The intent of this spec is to propose how we can provide access to database
models without direct neutron imports by means of a bridge in neutron-lib
thereby breaking consumer's dependencies on neutron's internal database
models.

Proposed Change
===============

While one foreseeable solution would be to publish models using
discoverable entry points (stevedore) or register them using a factory,
these models must be versioned to allow consumers to determine compatibility.
Without versioning the underlying model can change without the consumers
knowledge, thereby breaking them.

Implementing versioned models is simple enough with some new facility, however
we already have such a versioning scheme; neutron versioned objects. Neutron
objects already contain a reference to the underlying model as well as a
version number that's incremented as the model changes [5]_. Therefore if we
can provide a way for consumers to get at neutron objects, they have access to
a version, corresponding model, etc..

This spec proposes we use a simple entry point scheme whereby all versioned
objects are defined as entry points that can then be looked up and handed
out by some bridging logic in neutron-lib. More specifically:

- Neutron versioned objects can be "published" via an entry point in
  ``setup.cfg`` where each entry point is a versioned object class. This
  exposes the objects to neutron-lib that can be discovered and loaded with
  ``stevedore``.
- A simple API in neutron-lib allowing consumers to retrieve versioned objects
  at runtime. In it's basic form a consumer asks for an object of type ``X``,
  whereupon neutron-lib looks it up from the entry point and returns the
  concrete object class to the consumer (e.g. ``load_class('Port')`` looks up
  and returns the versioned object class for ``Port``).
- If consumers need to create instances of the versioned object(s) returned
  from neutron-lib, they can invoke it's constructor directly to create a
  new instance. Since versioned object constructor's are based on the object's
  ``fields`` and the ``fields`` are tied to the model, consumers can query
  the object's ``VERSION`` to determine compatibility with the constructor.

The snippet below illustrates the API from a consumers point of view::

    from neutron_lib.objects import registry
    # .. other imports

    # Loading the object from neutron-lib uses stevedore to find the requested
    # object. Also note that under the covers the model is imported since the
    # concrete versioned object must import the model to ref it.
    port_ovo = registry.load_class(constants.PORT)

    if port_ovo.VERSION > '1.1':
        raise UnsupportedException("Can't use port objects greater than 1.1")

    # It's just an OVO class, so we can use it's static/class methods directly
    a_port = port_ovo.get_object(...)

    # Create a new versioned object; it's VERSION determines the constructor's
    # supported kwargs so consumers can detect compatibility
    new_port = port_ovo(context, project_id=...)

As mentioned earlier, while this solution doesn't completely solve database
migration access, it paves the way as we now have a way to find and load
versioned objects which must import the respective model.

Using this scheme, we should be able to eliminate consumers imports for:

- Existing ``neutron.objects`` imports [6]_. Consumers now get their object from
  the neutron-lib API provided herein.
- Building queries; version objects have methods allowing consumers to
  effectively query and there should be no need to build direct queries in
  consumers otherwise.

In addition, the rollout of this functionality should have minimal impact to
the main code paths; the only real change in neutron is to expose the objects
via ``setup.cfg``. The neutron-lib logic can be implemented, tested and rolled
out independently thereby reducing risk.

For a sample proof of concept, see the patches on [7]_ that use this approach
to remove/use versioned objects in a few of the vmware-nsx modules.

References
==========

.. [1] https://blueprints.launchpad.net/neutron/+spec/neutron-lib-networking-ovn
.. [2] https://blueprints.launchpad.net/neutron/+spec/neutron-lib-dynr
.. [3] http://codesearch.openstack.org/?q=from%20neutron.db.model
.. [4] http://codesearch.openstack.org/?q=from%20neutron.db%20import%20model
.. [5] https://docs.openstack.org/neutron/latest/contributor/internals/objects_usage.html
.. [6] http://codesearch.openstack.org/?q=from%20neutron.objects
.. [7] https://review.openstack.org/#/q/topic:bp/neutronlib-decouple-db
.. [8] https://stackoverflow.com/questions/45534903/python-sqlalchemy-attributeerror-mapper
