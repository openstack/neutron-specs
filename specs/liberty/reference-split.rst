..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Neutron ML2 Reference Implementation Split
==========================================

https://blueprints.launchpad.net/neutron/+spec/reference-implementation-split

This spec tracks the splitting out of the ML2+[OVS,LB], L3 and DHCP reference
implementations from the main neutron source tree into a separate git
repository under the neutron project. This is being done to ensure the focus
of Neutron Core is squarely on being a management layer, and to ensure the
existing RPC and agent-based SDN reference implementation can evolve on it's
own. This also puts the existing reference implementation on equal footing
with the other Open Source implementations. As Neutron is a platform, competing
with those technologies it wants to enable has always seemed strange.

Problem Description
===================

Neutron has grown large. It also has developed an in-tree reference
implementation. Some of the problems with this model are:

* The in-tree reference implementation was meant as a proof of concept early
  on. It has taken many cycles to stabilize it.
* New features are proposed which sometimes ignore the logical model and
  instead focus on the in-tree reference implementation.
* The skills necessary to develop an SDN controller (what the in-tree
  reference implementation is) are different from those necessary to build
  a scalable API server. This has resulted in core reviewers who focus on
  one area or the other.
* By having an in-tree reference implementation which is open source, we are
  competing with other open source as well as vendor implementations. This
  goes against a core tenant of Neutron, which is to be a scalable networking
  API, not an SDN controller.

By factoring out the reference implementation, we will put Neutron back on firm
ground as an API server with a DB layer. We will no longer be directly competing
with the implementations who chose to implement the Neutron API. And we will allow
a natural split of core reviewers across the two projects.

Proposed Change
===============

We will create a new git repository for neutron called "neutron-reference."
The following directories will be migrated out and into this repository:

* neutron/agent
* neutron/plugins/linuxbridge
* neutron/plugins/ml2
* neutron/plugins/openvswitch
* neutron/plugins/sriovnicagent
* neutron/scheduler/dhcp_agent_scheduler.py
* neutron/scheduler/l3_agent_scheduler.py
* neutron/services/l3_router/l3_router_plugin.py
* neutron/tests/functional/agent

This move will result in the SDN agent-based implementation existing in the
new repository. As such, changes to devstack will be required to install
the agent-based implementation from the new repository.

Note we will be moving the functional tests around the agent into the new
repository. For now, we'll leave the Tempest tests for Neutron in-tree, as
they do not require a specific backend implementation. The goal is to
enhance those long-term so they can become a sort of qualification test
for backend implementations.

Please note that new features which require a backing implementation of their
logical models will still require changes in the built-in (and now spun out)
reference implementation.

Data Model Impact
-----------------

The new repository wil always use the Neutron DB but will get it's own alembic
branch.

REST API Impact
---------------

None

Security Impact
---------------

None

Notifications Impact
--------------------

None

Other End User Impact
---------------------

End users will now have to install the reference implementation from a separate
git repository. Thus, packagers will need to package a new git repository as
well.

Performance Impact
------------------

None

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

Deployers will need to be made aware of the fact they will need to deploy
from a separate git repository if they are using the existing in-tree
implementation.

Developer Impact
----------------

By splitting out the reference implementation, we will be increasing
developer velocity by having a smaller core team focused on this bit
of code.

One anticipated change is that some networking-foo projects will need to
update their requirements if they depend on parts of the neutron code
base which will be split out.

Community Impact
----------------

This will be inflating Neutron's Big Tent a bit more, but adding another
git repository. I expect it to increase the community, as we grow and
add more developers looking to develop SDN controllers. If you use the
broader Big Tent analogy [1]_ as an example, we will be broadening
Neutron's tent while at the same time focusing the core of Neutron.

Alternatives
------------

There are no alternatives other than keeping the in-tree reference
implementation as it is. But given the issues with this, moving it out
is the only realistic way forward.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Kyle Mestery: https://launchpad.net/~mestery

Other contributors:
  Doug Wiegley: https://launchpad.net/~dougwig
  Henry Gessau: https://launchpad.net/~gessau
  Gal Sagie: https://launchpad.net/~gal-sagie
  Miguel Angel Ajo: https://launchpad.net/~mangelajo

Work Items
----------

* Move code around in-tree to prepare for the decomposition.
* Create the new git repository with the existing history of the directories
  proposed above.
* Ensure basic UT works, including pep8.
* Ensure devstack is updated to install the code from the new repository.
* Ensure all gate jobs work as expected, moving those which make sense to
  targetthe new repository.
* Profit.

Dependencies
============

There may be a dependency on the neutron-lib work [2]_ being done. The new code
will be a user of some neutron features, much like the advanced services. We'll
know more about this being a hard dependency soon.

Testing
=======

Testing is critical for this to be a success. Given we'll be moving the code
which runs for all neutron jobs, among others, it's critical we ensure a
smooth experience. The gate will likely be affected during the transition
window, but the goal will be to get things moving as quickly as we can.

Once the code is moved, the hope is to pin the neutron requirement in the
new agent repo to a specific version of Neutron, and migrate that pin at
times throughout the cycle. In this way, we will keep things stable and
inflict any pain of moving the neutron pin to a few times each cycle. This
has the added benefit of allowing iteration in the two repositories at
separate levels of speed.

Tempest Tests
-------------

The goal is keep all existing Tempest tests running.

Functional Tests
----------------

The goal is keep all existing Functional tests running. The new repository
will add new functional test as needed.

API Tests
---------

None


Documentation Impact
====================

This will require documentation changes, as the documentation changes assumes
the reference implementation. We will not change that aspect of the
documentation, only where to install the reference implementation from.

User Documentation
------------------

See above.

Developer Documentation
-----------------------

None

References
==========

.. [1] http://superuser.openstack.org/articles/openstack-as-layers-but-also-a-big-tent-but-also-a-bunch-of-cats
.. [2] https://review.openstack.org/#/c/154736/
