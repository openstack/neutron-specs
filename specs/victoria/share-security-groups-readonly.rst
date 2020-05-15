..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Allow sharing security groups as read-only
==========================================

https://bugs.launchpad.net/neutron/+bug/1875516

Allow sharing security groups as read-only.

Problem Description
===================

Currently, security groups can be shared with the rbac system, but the only
valid action is `access_as_shared`, which allows the target tenant to
create/delete (only) new rules on the security group. This works fine for
use-cases where the group should be shared in a nearly equal way.

However, some users/services may want a security group to be visible, but
read-only. A prime example of this would be to enable ProjectB to add a
security group owned by ProjectA as a remotely trusted group on their own
security group.

The immediate need for this is found in an existing Octavia patch [1]_.
Octavia would like to share the security group it creates for each
load-balancer with the load-balancer's owner, so they can open access to their
backend members for only a specific load-balancer.

Proposed Change
===============

Add a new action type for security group RBAC: `access_as_readonly`. This
action would allow the target tenant to see the shared security group with
show/list, but not create/delete new rules for it or change it in any way.

Documentation Impact
====================

Neutron documentation about sharing security groups will need to be modified to
add the action type `access_as_readonly`.

Implementation
==============

Assignee(s)
-----------

* Adam Harwell

Work Items
----------

* Add new action type `access_as_readonly`
* Documentation update in config-rbac.rst [2]_ as seen in [3]_
* Create additional tempest tests in RbacSharedSecurityGroupTest class [4]_

References
==========

.. [1] https://review.opendev.org/723735

.. [2] https://github.com/openstack/neutron/blob/master/doc/source/admin/config-rbac.rst

.. [3] https://docs.openstack.org/neutron/train/admin/config-rbac.html#sharing-a-security-group-with-specific-projects

.. [4] https://github.com/openstack/neutron-tempest-plugin/blob/master/neutron_tempest_plugin/api/test_security_groups.py
