..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
Extend Quota API to report usage statistics
===========================================

https://bugs.launchpad.net/neutron/+bug/1599488

The current Neutron Quota API does not provide extensive information about quotas.
Providing a detailed information about quotas will be beneficial to Horizon web
app.

Problem Description
===================

The current Neutron Quota API just reports the resources and their limits for
a tenant. This current approach is inefficient for Horizon and quota users in general.
Quota APIs of other projects(nova and cinder) include usage details for
resources under a specific tenant.

There is a problem of manually counting all of neutron resources [1],
instead of using one Quota API call to get all detailed information
for resources under a specific project.

Proposed Change
===============

The "Quota Sets" extension provided by Nova[2] and Cinder[3]
should be adopted by Neutron, in order to provide better cross-project APIs
when it comes to resource quotas and more features.

The existing quota extension lists the current set of quota limits for a specified project.
We propose to add a new extension, quota_detail, which is available via the same endpoint.
This new extension will have a new action "detail" which lists quota details for a project.
Details like reserved, used and limit for a resource will be available.

Possible use cases of extended quota API is to simplify and provide
improvements in Horizon and OpenStack/Neutron client.

The new "Quota Extension" API extension in neutron will not have 'user-id' as part of the
arguments to list detail quotas.
This is because neutron supports only simple per-project quotas

REST API Impact
---------------

Currently, the Quota API provides the following REST API endpoint for
quota details for a specific tenant/project.


GET /v2.0/quotas/{tenant_id}

::

    {
        "quota": {
            "subnet": 10,
            "network": 10,
            "floatingip": 50,
            "subnetpool": -1,
            "security_group_rule": 100,
            "security_group": 10,
            "router": 10,
            "rbac_policy": -1,
            "port": 50
        }
    }

In the new Quotas API extension, a more featureful API would be introduced
that provides the *limit*, *reserved* and *in_use* counts for resources under
a specific project.
The current implementation of reservation in neutron is ephemeral and will not
up but show a default value of "0" to users.


GET /v2.0/quotas/{tenant_id}/detail

::

    {
        "quota": {
            "subnet": {
                "reserved": 0,
                "limit": 10,
                "in_use": 0
            },
            "network": {
                "reserved": 0,
                "limit": 10,
                "in_use": 0
            },
            "floatingip": {
                "reserved": 0,
                "limit": 50,
                "in_use": 0
            },
            "subnetpool": {
                "reserved": 0,
                "limit": -1,
                "in_use": 0
            },
            "security_group_rule": {
                "reserved": 0,
                "limit": 100,
                "in_use": 0
            },
            "security_group": {
                "reserved": 0,
                "limit": 10,
                "in_use": 0
            },
            "router": {
                "reserved": 0,
                "limit": 10,
                "in_use": 0
            },
            "rbac_policy": {
                "reserved": 0,
                "limit": -1,
                "in_use": 0
            },
            "port": {
                "reserved": 0,
                "limit": 50,
                "in_use": 0
            }
        }
    }



Command Line Client Impact
--------------------------

A new optional argument will be added to the openstack client,
for example:
$ openstack quota show {tenant_id/project_id} --detail

Horizon uses neutronclient to get quotas from neutron api.
Since CLI end of neutronclient will be deprecated soon, only the HTTP
side of neutronclient will be implemented for horizon use.

Security Impact
---------------

None

Notifications Impact
--------------------

None

Other End User Impact
---------------------

Changes in Horizon

out of scope for this spec.

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

Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

Prince Boateng(prince.a.owusu.boateng@intel.com)
Aradhana Singh(aradhana1.singh@intel.com)

Work Items
----------

1. Extend Quota DB driver
2. Extend current API
3. Create new Quota extension
4. Add related tests
5. Add CLI openstackclient

Dependencies
============

None

Testing
=======

Tempest Tests
-------------

Write tests to verify the correct data model is returned from API

Functional Tests
----------------

None

API Tests
---------

Write tests to check responses for ``reserved``, ``in_use`` and ``limits`` parameters for
resources are consistent with the changes introduced.

Documentation Impact
====================

Yes

User Documentation
------------------

Yes

Developer Documentation
-----------------------

Yes

References
==========

[1]: https://github.com/openstack/horizon/blob/10.0.0.0b1/openstack_dashboard/usage/quotas.py#L313-L336

[2]: http://developer.openstack.org/api-ref-compute-v2.1.html#listDetailQuotas

[3]: http://developer.openstack.org/api-ref-blockstorage-v2.html#showQuota

[4]: https://review.openstack.org/#/c/383673
