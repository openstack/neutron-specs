..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================
Moving to Keystone v3
=====================

https://blueprints.launchpad.net/neutron/+spec/keystone-v3

This blueprint is meant to capture the changes to the Neutron repos (neutron,
neutron-lib and python-neutronclient) needed to fully integrate Neutron with
the Keystone v3 API.

All the steps described in this document are applicable also to repos
neutron-fwaas, neutron-lbaas, neutron-vpnaas and neutron-dynamic-routing.
However, only the changes in neutron, neutron-lib and python-neutronclient
are required to consider this blueprint completed.


Problem Description
===================

The `Keystone v3 API`_ was released on February 20, 2013. In the Icehouse
release the `Keystone v2 API was deprecated`_. The deprecation had to be
`reverted`_ because many OpenStack projects did not support the v3 API yet.

In the Mitaka release Keystone is again `deprecating the v2 API`_. Projects will
be able to use the v2 API for the next four releases, but after that it will not
be supported. To be consistent with other OpenStack projects and to be able to
continue using the Identity Service, we must update Neutron to exclusively use
Keystone v3 when v2 is no longer supported.

With the Keystone v3 API, the concept of "tenant" is deprecated in favor of
"project". In particular, the attribute ``tenant_id`` is replaced with
``project_id``. Neutron must initially support ``project_id`` as an alias for
``tenant_id`` before finally replacing ``tenant_id`` altogether. Deprecation
will be kept as long as Neutron CLI will be supported.

In requests **to** Neutron, the neutron server already treats ``project_id`` as
an alias for ``tenant_id``. Internally, Neutron should eventually replace
``tenant_id`` with ``project_id`` in attributes and database fields.

In requests **from** Neutron to other OpenStack services, the neutron components
should use keystoneauth, which abstracts the v2/v3 differences.


Proposed Change
===============

Change all existing uses in Neutron code from Keystone v2 to Keystone v3.
Initially both versions of API will be supported. Migration will be handled in
several steps.

0. Update Neutron server to accept Keystone v3 contexts in requests. [DONE]
1. Update the API bindings in python-neutronclient to properly handle both
   project_id and tenant_id.
2. Update Neutron server to use only keystoneauth for requests to other
   OpenStack services.
3. Update documentation about deprecation of tenant_id attribute:

   a. OpenStack Networking API reference,
   b. Neutron and neutronclient developer references.

4. Create ``project_id`` alias for all ``tenant_id`` columns in DB.
5. Update codebase, changing all occurences of ``tenant`` with ``project``,
   where appropriate. Special care needs to be taken for:

   a. When renaming arguments for callables, some callers may pass them using
      kwargs notation (tenant_id=...) instead of positional.
   b. When changing code that is used by external projects, the change must not
      break the external project. Deprecation warnings should be set up.


Neutron API
-----------

The Neutron API accepts the ``project_id`` attribute and treats it as an alias
for the ``tenant_id`` attribute. This is `already implemented`_.

Also, ``tenant_id`` is used as a filter parameter in Neutron API. It is a
requirement that this use-case will continue to work.

Neutron API responses will be expanded to contain both ``tenant_id`` and
``project_id``. For now we are unable to phase out ``tenant_id`` since that
would require an API version change. When `microversioning`_ is adopted for the
Neutron API the task of removing deprecated attributes can be commenced.
Meanwhile we will expose ``project_id`` through yet another API extension,
enabled by default.


Neutron client
--------------

For CLI commands, the Neutron client is being deprecated in favor of `OpenStack
Client`_.  Therefore the CLI portion of python-neutronclient will not be
updated.  In the OpenStack Client the CLI commands have already adopted the
policy of using ``project_`` and do not support ``tenant_``.

The only part of python-neutronclient which will be modified is the Python API
bindings.  All the places where the bindings accept ``tenant_id`` and
``tenant_name`` need to be expanded to also accept ``project_id`` and
``project_name``.  The ``tenant_`` parameters will be marked as deprecated and
to be removed in the future.


OpenStack documentation
-----------------------

Neutron client CLI is being deprecated in favor of `OpenStack Client`_, thus
the only document update regarding its behavior will contain information about
deprecation.

The neutronclient Python API documentation will be updated to show
corresponding changes about bindings.


Neutron developer reference
---------------------------

The term ``tenant`` is used all across the documentation. All the places need to
be updated, with extra information about ``tenant_id`` deprecation in favor of
the ``project_id``.

Additional guidelines will be provided for all subprojects/affiliated projects,
about modifications required to be consistent with Neutron API changes.


Database
--------

Renaming columns from ``tenant_id`` to ``project_id`` and from ``tenant_name``
to ``project_id`` will require downtime and will be done in a contract
migration.

To allow changes in smaller chunks, and for backward compatibility, we will
temporarily use the `SQLAlchemy synonym feature`_ to make ``tenant_id`` a
synonym for ``project_id`` columns.

For some time, database operations will see ``tenant_id`` and ``project_id``
attributes separately coming up in lists of properties and such, and care will
need to be taken in how they are filtered and passed around.

Once all existing columns have been renamed to ``project_id``, all new columns
added to any Neutron code shall use only ``project_id``.  A hacking rule will
be added to prevent the reintroduction of ``tenant_id`` in source code.


Internal changes
----------------

Server code changes will be submitted in reviewable chunks, without interruption
to the external API or external project dependencies. Client code changes are
additive until the deprecated keywords are removed.


Testing
=======

API Tests
---------
Write tests to check responses for ``tenant`` and ``project`` parameters, to be
sure that introduced changes are consistent.

Functional Tests
----------------
* Test if ``project_id`` returns the same response as ``tenant_id``.
* Test if ``tenant_id`` is updated when ``project_id`` is updated.


References
==========
.. target-notes::

.. _`Keystone v3 API`: http://specs.openstack.org/openstack/keystone-specs/api/v3/identity-api-v3.html#what-s-new-in-version-3-0
.. _`Keystone v2 API was deprecated`: https://blueprints.launchpad.net/keystone/+spec/deprecate-v2-api
.. _`reverted`: http://lists.openstack.org/pipermail/openstack-dev/2014-March/031016.html
.. _`deprecating the v2 API`: http://lists.openstack.org/pipermail/openstack-dev/2015-November/080816.html
.. _`already implemented`: https://review.openstack.org/253782
.. _`SQLAlchemy synonym feature`: http://docs.sqlalchemy.org/en/rel_1_0/orm/mapped_attributes.html?highlight=synonym#synonyms
.. _`microversioning`: https://blueprints.launchpad.net/neutron/+spec/consolidate-extensions
.. _`OpenStack Client`: http://docs.openstack.org/developer/python-neutronclient/devref/transition_to_osc.html
