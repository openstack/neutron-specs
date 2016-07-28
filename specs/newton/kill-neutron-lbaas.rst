..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Kill neutron-LBaaS, aka LBaaS project spinout
=============================================

The Octavia project has, as part of creating the new LBaaS reference
implementation, also implemented the LBaaS API as a standard pecan/wsgi
openstack endpoint.

The Octavia LBaaS API would be identical and compatible with the Neutron
LBaaS v2 API, excepting the /lbaas root url token being missing.

End users of the existing neutron API endpoint will see no differences
for two cycles, which means potential removal in P or Q, depending on when
this work lands. Note that if the shim is low-maintenance enough, it may
live for even longer; that will be at the discretion of the neutron PTL
and core team.

This spec describes the process for removing LBaaS as a neutron extension,
and making it a standalone project of its own.


Problem Description
===================

Octavia provides a virtually identical API and plugin mechanism that
neutron-lbaas provides without the tight coupling, which calls into question
why we are maintaining both, now that lbaasv1 is past deprecation.
The main goals of this spinout will be simplifying the administration
and implementation of both, while providing as seamless an experience
as possible to operators and end users.

There are several reasons for spinning out the LBaaS project:

* Having your service stuffed into the guts of a neutron-server, while
  your code is in a separate repo, has been a fairly brittle proposition.

  * Refactors to core neutron have a chance of breaking LBaaS.

  * Co-gate adds to instability of gate for both projects, and can be removed.

  * Neutron extensions get in the way of microversioning, and making
    that migration a mess.

* We would no longer be double-testing nearly the same functionality,
  and consuming more infra and testing resources.

* The active teams for LBaaS and neutron have virtually no overlap,
  so the added overhead of administering both is providing no benefit
  to neutron, and slowing down LBaaS.

* Octavia has its own REST API, so why maintain two, if we can make them
  compatible?

The reasons against:

* There is a certain amount of busywork required to finish getting ready
  to be a completely separate project, listed in proposed changes below,
  but the payoff in two simpler projects would seem worth it.

* There is some value in having all the "networking" APIs under one umbrella,
  though there is precedent for higher-level services being outside (DNSaaS).


Proposed Change
===============

This list of changes will be split into "phase one" and "phase two". Phase
one is the minimum list of changes necessary for a governance change and
to stop maintaining neutron-lbaas as it is today, and phase two are logical
next steps needed for any standalone openstack project.

Octavia is an API server, a service VM framework for LBaaS, and the
reference implementation for LBaaS. Currently only service VM centric
drivers are supported

Phase one:

* In Octavia, an interface needs to be added so that non-service VM
  drivers can be used behind the API server (e.g. hardware appliances or
  existing lbaas drivers.) This driver
  interface should accept current neutron LBaaS v2 drivers directly,
  provided that those drivers limit themselves to passed in objects and
  public lbaas plugin interfaces. The current neutron lbaasv2 drivers
  will be moved into the octavia repo, though any driver reaching into
  neutron's guts is likely to break. So, don't reach into guts.

* Modify the neutron-LBaaS API extension to be a direct proxy
  pass-through to the octavia api controller.

* Notify vendors to verify their drivers work with the Octavia API
  controller, and to move their third-party CIs.

* Notify other projects using LBaaS, notably Heat.

* Remove neutron-LBaaS/neutron co-gate.

* Verify minimum docs coverage from a user and API standpoint.

* Apply for governance change from neutron stadium to the big tent.
  This includes moving neutron-lbaas into octavia, unless the neutron
  /lbaas proxy is a pecan route in neutron proper, and then neutron-lbaas
  can simply be removed/sent to the attic.

Phase two:

* Create client library, and/or direct support in openstackclient/sdk.
  What happens to the existing 'openstack loadbalancer'? Is it just a neutron
  proxy for a few releases? Long-term, it will be an OSC plugin.

* Create/verify/move API/install guide/user docs that are already written
  for neutron-lbaas.

During this transition, and for sometime after, the neutron /lbaas proxy
will make this transparent to users.

References
==========

Spec that outlines specific changes needed in Octavia for this to happen:

https://review.openstack.org/#/c/310667/

Proposed neutron-lbaas proxy shim layer:

https://review.openstack.org/#/c/328053/

Brandon's work on making an lbaasv2 driver layer directly in Octavia:

https://review.openstack.org/#/c/311586/

WIP governance patch, which will remain WIP until phase one is nearing
completion:

https://review.openstack.org/#/c/313056/

Octavia's 'api_handler' interface, which is similar to the current lbaasv2
driver interface (see 311586 for work to make this seamless with current
drivers):

https://github.com/openstack/octavia/blob/master/octavia/api/v1/handlers/abstract_handler.py

