..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================================
Add support for Keystone V3 APIs in the python-neutronclient.
==============================================================

URL of the launchpad blueprint:
********************************************************************************
https://blueprints.launchpad.net/python-neutronclient/+spec/keystone-api-v3-support

This blueprint is meant to capture the changes necessary to the
python-neutronclient to integrate with python-keystoneclient for authentication
and session management.  All clients have this requirement.


Problem description
===================
Python-neutronclient lacks Keystone V3 support. Furthermore, it is duplicating
python-keystoneclient logic by maintaining its own version of Keystone V2
authentication API and session management (i.e. endpoint lookup). A major
drawback with this approach is that it must be constantly updated in response
to any Keystone API changes. Maintenance is also a burden as authentication
and session management are not consistent across all OpenStack Python clients.


Proposed change
===============
Utilizing python-keystoneclient for authentication and session management
so that they are completely abstracted from python-neutronclient. The changes
are twofold, CLI (shell) and SDK (Client).

CLI
---
For CLI, the global identity arguments, which are common to all the OpenStack
Python clients, should be provided and facilitated by python-keystoneclient.
Python-neutronclient does not need to know about them. It simply need a way
to convey them to the end users. Therefore, the following global identity
arguments will be isolated and eventually be facilitated by
python-keystoneclient:

- *--os-auth-url*
- *--insecure*
- *--os-cacert*
- *--os-cert*
- *--os-key*
- *--os-token*
- *--os-username*
- *--os-user_id*
- *--os-password*
- *--os-user-domain-id*
- *--os-user-domain-name*
- *--os-tenant-name*
- *--os-tenant-id*
- *--os-project-name*
- *--os-project-id*
- *--os-project-domain-id*
- *--os-project-domain-name*
- *--os-region-name*
- *--os-service-type* (Default to ``network``)
- *--os-endpoint-type* (Default to ``publicURL``)
- *--os-url* (DEPRECATED, should be using *--os-endpoint* instead)
- *--os-endpoint*
- *--os-auth-strategy* (DEPRECATED, absence of *--os-auth-url* signify no auth)


Client
------
Use ``keystoneclient.session.Session`` for session management and
python-keystoneclient auth plugin for authentication. This is done by
introducing two optional arguments, ``session`` and ``auth``,  to
``neutronclient.common.clientmanager.ClientManager`` class::

    class ClientManager(object):
        """Manages access to API clients, including authentication.
        """
        neutron = ClientCache(neutron_client.make_client)
        # Provide support for old quantum commands (for example
        # in stable versions)
        quantum = neutron

        def __init__(self, token=None, url=None,
                     auth_url=None,
                     endpoint_type=None,
                     tenant_name=None,
                     tenant_id=None,
                     username=None,
                     user_id=None,
                     password=None,
                     region_name=None,
                     api_version=None,
                     auth_strategy=None,
                     insecure=False,
                     ca_cert=None,
                     log_credentials=False,
                     service_type=None,
                     session=None,
                     auth=None
                     ):

Where caller can optionally pass in an instance of
``keystoneclient.session.Session`` in ``session`` and an instance of
``keystoneclient.auth.base.BaseAuthPlugin`` in ``auth``.

If ``session`` is provided, we shall use it for HTTP session management instead
of ``neutronclient.client.HTTPClient``. This is done by providing shims for the
the existing ``neutronclient.client.HTTPClient`` to preserve backward
compatibility.

Changes to ``neutronclient.client``::

    class SessionHTTPClient(HTTPClient):
        """Shims for HTTPClient.

        Requests are delegated to keystoneclient Session.
        """

        def __init__(self, session, auth,
                     region_name=None,
                     service_type='network',
                     endpoint_type='publicURL'):

    def _construct_http_client(*args, **kwargs):
        session = kwargs.pop('session', None)
        auth = kwargs.pop('auth', None)
        if session:
            return SessionHTTPClient(session, auth, **kwargs)
        else:
            return HTTPClient(**kwargs)


For ``neutronclient.common.clientmanager.ClientManager`` and
``neutronclient.v2_0.client.Client``, instead of instantiating
``neutronclient.client.HTTPClient``, it will just call
``neutronclient.client._construct_http_client`` to get a HTTP client
object.

At some point in the future if we choose to completely remove the old HTTPClient,
we should also remove the ServiceCatalog class and all the home-grown parsing
that goes with it.  It's much cleaner to simply let the keystone client do
all that parsing.  bklei will add a fixme comment in the code to note that
for future cleanup.

Alternatives
------------
None -- this is a required change.


Data model impact
-----------------
None.


REST API impact
---------------
None.


Security impact
---------------
None.


Notifications impact
--------------------
None.


Other end user impact
---------------------
In order to authenticate with V3 in keystone, if a username is provided
for authentication, the user's domain name or id must also be provided.
Similarly, if a tenant/project name is provided, the tenant's domain name
or id must also be specified.

Performance Impact
------------------
Shouldn't be any -- the same calls to keystone are being made, just via
the keystone client instead of the neutron specific HTTPClient.


Other deployer impact
---------------------
None.


Developer impact
----------------
Same as the end user impact.


Implementation
==============

Assignee(s)
-----------
Bradley Klein (bklei)


Work Items
----------
Need to import the keystone client session and auth plugin, and construct
both to authenticate.


Dependencies
============
None, the keystone client already provides what is needed for this change.


Testing
=======
Unit testing comprehensively tests the keystone integration, those tests will
be modified/enhanced to also test the new V3 code.


Documentation Impact
====================
The new domain specific parameters for the neutron command should be documented.
It would also probably make sense to mention that the python-keystoneclient
supports both v2 and v3 auth based on the value provided by auth-url.


References
==========
None.
