..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================
Decoupling database imports/access for neutron-lib
==================================================

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

This spec specifically address the Database API & Utilities access.

For current neutron-lib related blueprints, see [1]_ and [2]_.


Problem Description
===================

As part of our neutron-lib effort, we need to decouple out-of-tree networking
projects that depend on (e.g. ``import``) neutron. However, today a large
number of neutron consumers import database related neutron modules [3]_ [4]_
[5]_. While some of these dependencies are rehomeable into neutron-lib without
significant effort, others create a large dependency chain with neutron
internals, coupling them to neutron itself.

The intent of this spec is to propose how we can rehome the neutron database
API and utilities into neutron-lib breaking consumer dependencies on these
neutron modules. The access patterns herein were gathered by inspecting usages
of the respective modules using [3]_, [4]_, [5]_, [6]_.

Proposed Change
===============

This spec proposes we rehome the generic database API/utility functionality
into neutron-lib. The following considerations must be taken into account
for all applicable code:

- If the functionality is private and not used outside of neutron today,
  it can likely stay in neutron as private to reduce public API surface area.
- If the functionality is private, but used outside of neutron, it's a
  candidate for rehoming.
- For code applicable to rehoming, determine the best form for the
  implementation. If used by non-OVO consumers then likely it'll be
  necessary to rehome as "generic database logic". However if applicable
  to OVO, it may make sense to incorporate the logic into neutron objects
  that will be rehomed in subsequent work.

When the work for this spec is done, we should no longer have any consumers
using neutron's database API or utils [3]_, [4]_, [5]_, [6]_, but rather
using them from neutron-lib.

Subsequent subsections describe each module in greater detail.

Note that ``provisioning_blocks`` will be addressed in subsequent specs
as it requires versioned objects and thus will need access to them via
neutron-lib in order to implement.

Database API
------------

The ``neutron.db.api`` module is used broadly today [3]_ and has already begun
rehoming into neutron-lib [7]_. Therefore this spec proposes we finish the
database API rehoming work by:

- Rehoming the remaining externally used functions in the API. This includes
  generic functions such as ``is_retriable`` as well as decorators like
  ``retry_db_errors`` and ``retry_if_session_inactive``.
- Removing deprecated and unused functions such as ``load_one_to_manys`` and
  ``get_session``.
- The osprofiler setup/initialization will need to be addressed to ensure
  proper profiling functionality.
- Unit tests will need to be in place for the rehomed functionality; rehomed
  and/or written as needed.
- Consuming the rehomed changes once released.

Generic Utils
-------------

The ``neutron.db._utils`` module contains generic logic and is thus used by
some consumers today [4]_. This spec therefore proposes we rehome the used
functionality into neutron-lib by:

- Rehoming the used functions such as ``resource_fields`` and
  ``get_marker_obj``
- Rehoming any functions use in the model query utils (see subsequent section)
  into neutron-lib.
- Ensuring the proper unit test coverage is in place for the rehomed code.
- Consuming the rehomed changes once released.

Model Query Utils
-----------------

While today ``neutron.db._model_query`` is private, some consumers use it [5]_.
As this module contains generic database logic, it's a candidate for rehoming
to neutron-lib as a generic model hook module by:

- Rehoming the logic in the ``_model_query`` module itself to neutron-lib.
- Ensuring the generic utils are in lib that are required for this module.
  See the previous section on generic utils.
- Rehoming the few used functions from ``neutron.common.utils`` used in the
  module query module. These are generic in nature and thus don't raise any
  red flags for rehoming.
- Rehoming ``neutron.objects.utils`` to neutron-lib as its used in the model
  query logic.
- Ensuring the proper unit test coverage is in place for the rehomed code.
- Consuming the rehomed changes once released.

Resource Extend
---------------

The ``neutron.db._resource_extend`` module is used by consumers today [6]_ and
as it contains generic functionality, can also move into neutron-lib by:

- Rehoming the resource extend module into neutron-lib.
- Rehoming the single generic ``neutron.common.util`` function required
  by the module into neutron-lib.
- Ensuring the proper unit test coverage is in place for the rehomed code.
- Consuming the rehomed changes once released.


References
==========

.. [1] https://blueprints.launchpad.net/neutron/+spec/neutron-lib-networking-ovn
.. [2] https://blueprints.launchpad.net/neutron/+spec/neutron-lib-dynr
.. [3] http://codesearch.openstack.org/?q=from%20neutron%5C.db%20import%20api
.. [4] http://codesearch.openstack.org/?q=from%20neutron%5C.db%20import%20_utils
.. [5] http://codesearch.openstack.org/?q=from%20neutron%5C.db%20import%20_model_query
.. [6] http://codesearch.openstack.org/?q=from%20neutron%5C.db%20import%20_resource_extend
.. [7] https://github.com/openstack/neutron-lib/blob/master/neutron_lib/db/api.py
