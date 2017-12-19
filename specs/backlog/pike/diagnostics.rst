..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Neutron Resource Diagnostics
============================

RFE: https://bugs.launchpad.net/neutron/+bug/1507499

Problem Description
===================

Cloud software is complex and problems are inevitable. Neutron is no exception
to this rule. To cope today we have helper scripts [1]_, blogs [2]_, diagnostic
tools [3]_, out-of-tree plugin utilities [4]_, etc. that could all benefit from
additional Neutron diagnostic data APIs. And while we certainly don't want
Neutron to become a new diagnostic system, other projects [5]_ have
found value in exposing some level of diagnostic data for users (typically
operators) needing to reach under the hood.

By the same token, Neutron offers a broad array of resources backed by a
heterogeneous set of technologies. As a result, standardizing a set of
concrete diagnostics across this diverse domain is no simple task.
Moreover, a Neutron resource may be realized through multiple components
(plugins, agents) spanning multiple nodes (hosts). Thus, any proposal
to collect resource diagnostic data must plan accordingly to support
multiple, potentially disparate, diagnostic participants.

Proposed Change
===============

This spec proposes to deliver a diagnostic framework consisting of the
following high level constructs:

- Diagnostic checks that are effectively micro-plugins containing concrete
  logic to carry out a diagnostic check for a given set of resource types
  (``ports``, ``networks``, etc.) as well as check specific metadata such
  as the check's name, description, etc.
- Diagnostic providers that are capable of discovering, managing and
  executing diagnostic checks. These providers are the sources for diagnostics
  and will allow neutron plugins and agents to plug-and-play with the
  framework.
- Diagnostics API extension that supports a ``/diagnostics`` URI for existing
  neutron resources and services the request by discovering and executing
  checks on applicable providers and rolling up the responses.

Each of the following constructs is discussed in greater detail in the following
sections.

Diagnostic Checks
-----------------

Diagnostic checks are effectively small plugins that provide the following:

- Check specific metadata including:

  - A unique name for the check.
  - (optional) A description of the check's diagnostics.
  - A boolean flag indicating if the check is required or not.
  - A set of resource types (``ports``, ``networks``, etc.) the check is
    applicable to.

- The actual diagnostic check logic. This logic will be invoked when collecting
  diagnostics for a specific resource instance of a resource type that's
  supported by the check (as per the check's metadata). Checks return a
  well-formed response including a response code, (optional) text message, etc.
  If a check supports multiple resource types, its implementation can key off
  the resource type that'll be passed in by the framework.

As part of this work a sample check will be provided for both a server and
agent side plugin that illustrates how to use the framework.

Diagnostic Providers
--------------------

Diagnostic providers are simply 'sources' for diagnostic checks and contain
the logic to discover, manage and execute checks. Services in a deployment
wishing to contribute diagnostics will therefore implement a diagnostic
provider. For this effort we'll deliver a diagnostic provider binding
for both the neutron service as well as neutron based agents.

Diagnostic providers have the following characteristics:

- The means to discover and load known checks. More details in subsequent
  sections.
- An internal cache of loaded checks and public APIs to:

  - Determine if the provider has a check for a given resource type or list
    of resource types.
  - List the resource types the provider has checks for.
  - Execute all checks applicable to a specified resource type and resource
    instance, collect the well-formed results and return them.

The neutron server diagnostic provider binding delivered by this effort
will be implemented as a service plugin; consumers wishing to invoke it's
APIs can get a reference to the plugin instance via neutron manager. In
addition, other plugins can act as a diagnostic provider by supporting
the extension and implementing the respective diagnostic methods required.
This approach will also work for agent-less plugins and mimics what's typically
done today in such plugins when supporting extensions in their implementation.

The neutron agent based diagnostic provider binding delivered by this
implementation will use the agent extension framework; plumbing the support
for all neutron based agents. However since the agent binding is remote to the
neutron API server it has the following notable differences:

- To indicate the resource types the agent diagnostic provider supports,
  a set of (string) resource types is returned in the agent state report
  and stored in the agent database. On the agent, this set is built by
  collecting the supported resource types from the checks it manages. On the
  server side, the agent's supported resource list can be used by the API
  extension controller logic to determine if the agent is applicable
  to service specific diagnostic requests.
- The public APIs to invoke checks on a specific resource type and resource
  instance are remoteable so they can be called via RPC.

Diagnostic Providers: Loading Checks
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Diagnostic providers are responsible for loading checks upon start-up
(e.g. when service plugin starts or the agent starts). There are multiple
ways to support check plugins including:

- Having each check as a stevedore entry point; then loading and managing them
  via a driver manager.
- Defining a specific check directory that only contains python modules where
  each module is a check (plugin) and discovering + importing them from the
  directory.
- Statically defining the checks in the diagnostic provider binding code.
  This is just as it sounds; statically have a list of checks in the code
  rather than dynamically discovering + loading.

Each approach above has it's pros/cons. For the initial implementation we
suggest using the last approach of statically defining the checks in the code.
While this is not the most robust approach, it minimizes complexity and allows
us to get a tire-kicker more rapidly that we can then later iterate on once
we get some feedback from users.

Diagnostic Providers: Data Model
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As mentioned previously, neutron agent based diagnostic providers report
the resource types the checks they manage support. This is a list of
unique string resource types that is transported in a new key/value of
the agent state report. This list is stored in a new database table
in a new column called ``diagnostic_resource_types`` that maps to its
corresponding agent table entry.

None (default) or an empty list signifies the agent does not provide
diagnostic data and thus will not be called to service a diagnostic
request. Neutron (OVO) objects will be updated as necessary.

API Extension Plugin
--------------------

The implementation delivered will include an API extension plugin that acts
as the REST API controller for diagnostics. As described in the REST API
section, a ``/diagnostics`` URI is dangled off existing neutron resources
that only supports ``POST`` with an empty request body. Therefore this
controller needs to service diagnostic requests for a specific resource
type and ID.

The following pseudo code outlines the controllers logic for request handling:

- Get a list of all plugin instances from the neutron manager that support
  the diagnostic extension and filter them to only those that support
  the said resource type.
- Get a list of all active agents from the DB and filter to only those that
  have the said resource type in their ``diagnostic_resource_types`` column.
- From the list of plugin and agent providers that support the given resource
  type, invoke their API(s) to run the checks they know about for the said
  resource type and resource instance.
- Collect the diagnostic results, roll them into a nice response and return
  them to the caller.

REST API
--------

When enabled, this implementation dangles a ``/diagnostics`` URI off Neutron
resources. The only supported HTTP method for this URI is ``POST`` (with an
empty request body) that triggers diagnostic data collection from all
applicable registered diagnostic providers. While this spec proposes the
diagnostic collection be run synchronously for the initial iteration of the
implementation, future workings could make the collection job based run async
in the back ground.

The generic form of the diagnostics URL is::

    POST /v2.0/{resource}/{resource_id}/diagnostics

The response is a list of diagnostic (dict) objects, one object per
``diagnostic``. A diagnostic is an aspect of the ``{resource}`` checked,
and contains a ``description`` as well as a diagnostic ``status`` object
and list of individual ``checks`` run for the said diagnostic. Diagnostic
``status`` is set by the diagnostics framework based on the result of
all checks for the said resource ``diagnostic``.

The array of ``checks`` returned for each diagnostic includes high level
details about the check such as ``name``, ``description`` and ``provider``.
In addition, checks report their own ``status`` based on the result of
their check execution. If a check is not successful, the check must
return a ``remediation`` to describe potential ways to remediate the failed
check.

The ``status`` at the diagnostic level is handled by the framework and
can be one of the following:

- ``OK``: All checks for the diagnostic are successful.
- ``ERROR``: One or more checks failed.
- ``INACTIVE``: One or more providers is inactive and couldn't be invoked
  to run the diagnostic checks.
- ``DEGRADED``: Checks can be registered as optional. If a check is optional
  and fails, or is ``INACTIVE``, the diagnostic status will be ``DEGRADED``.

For example::

    POST /v2.0/subnets/315ec9bb-34f5-4f7a-a44c-b13015a26803/diagnostics
    EMPTY POST BODY
    ==> All successful DHCP diagnostics
    {
        "diagnostics": [
            {
                "diagnostic": "dhcp",
                "description": "Neutron network DHCP diagnostics.",
                "status": {
                    "code": "DS000",
                    "label": "OK",
                    "message": "All checks completed successfully."
                },
                "checks": [
                    {
                        "name:": "check1",
                        "description": "Check1 does this and that.",
                        "status": {
                            "code": "DS000",
                            "label": "OK",
                            "message": null
                        },
                        "provider": {
                            "name": "DHCP Agent",
                            "host": "dhcp-host1"
                        },
                        "remediation": {}
                    },
                    {
                        "name:": "check2",
                        "description": "Check2 does cool stuff.",
                        "status": {
                            "code": "DS000",
                            "label": "OK",
                            "message": null,
                        },
                        "provider": {
                            "name": "DHCP Agent",
                            "host": "dhcp-host2"
                        },
                        "remediation": {}
                    }
                ]
            }
        ]
    }

    POST /v2.0/subnets/315ec9bb-34f5-4f7a-a44c-b13015a26803/diagnostics
    EMPTY POST BODY
    ==> A failed dhcp diagnostic
    {
        "diagnostics": [
            {
                "diagnostic": "dhcp",
                "description": "Neutron network DHCP diagnostics.",
                "status": {
                    "code": "DS002",
                    "label": "ERROR",
                    "message": "Check 'check1' failed. See the check details for more info."
                },
                "checks": [
                    {
                        "name:": "check1",
                        "description": "Check1 does this and that.",
                        "status": {
                            "code": "DHCPE001",
                            "label": "ERROR",
                            "message": "The dnsmasq process is not running."
                        },
                        "provider": {
                            "name": "DHCP Agent",
                            "host": "dhcp-host1"
                        },
                        "remediation": {
                            "code": "DHCPR001",
                            "message": "Re-enable DHCP for this network, then rerun this check."
                        }
                    },
                    {
                        "name:": "check2",
                        "description": "Check2 does cool stuff.",
                        "status": {
                            "code": "DS000",
                            "label": "OK",
                            "message": null,
                        },
                        "provider": {
                            "name": "DHCP Agent",
                            "host": "dhcp-host2"
                        },
                        "remediation": {}
                    }
                ]
            }
        ]
    }

Access control to ``/diagnostics`` is handled via standard policy definition.
The default access control is ``admin_only``, but operators can change this
in ``policy.json`` as needed.


Benefits
--------

While the main purpose of this effort is to spearhead diagnostics in Neutron and
start building out the plumbing, this functionality is immediately valuable for
consumers and out-of-tree plugins alike.

For example:

- The python-don project [3]_ can implement diagnostic data collection for
  data used in its analysis.
- The vmware-nsx plugin can migrate some of its operator CLI functionality [4]_
  into diagnostic data.
- The implementation can be enhanced to collect interface stats similar to how
  Nova diagnostics [5]_ does.


Future work
-----------

- Once the API solidifies, a CLI can be added to support diagnostics.


References
==========

.. [1] https://github.com/openstack/osops-tools-generic/blob/master/neutron/listorphans.py
.. [2] http://www.tuxfixer.com/openstack-how-to-manually-delete-orphaned-neutron-port/
.. [3] https://github.com/openstack/python-don
.. [4] https://github.com/openstack/vmware-nsx/blob/master/vmware_nsx/shell/admin/README.rst
.. [5] https://wiki.openstack.org/wiki/Nova_VM_Diagnostics

Related Information
-------------------

- https://bugs.launchpad.net/neutron/+bug/1563538
