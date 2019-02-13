..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Neutron Stadium Evolution
=========================

The objective of this specification is three-fold:

* Define what the Neutron Stadium is.
* Define the meaning of the networking community within OpenStack.
* Define guidelines on how to maintain the Stadium over time.


Problem Description
===================

Neutron grew to become a big monolithic codebase, and its core team had a
tough time making progress on a number of fronts, like adding new
features, ensuring stability, etc. During the Kilo timeframe, a
decomposition `effort <http://specs.openstack.org/openstack/neutron-specs/specs/kilo/core-vendor-decomposition.html>`_
started, where the codebase got disaggregated into separate efforts, like
the high level services, and the various third-party solutions for L2
and L3 services.

The outcome of that initiative has yielded mixed results, nonetheless giving
the various individual teams that came out of it the opportunity to iterate
faster and reduce the time to feature. This has been due to the increased
autonomy and implicit trust model that made the lack of oversight of the
PTL and the Neutron drivers/core team acceptable for a small number of
initiatives. However, the proposed `arrangement <https://review.openstack.org/#/c/175952/>`_
made it possible for projects to be `automatically <http://git.openstack.org/cgit/openstack/governance/commit/?id=321a020cbcaada01976478ea9f677ebb4df7bd6d>`_
enlisted as a Neutron project based simply on description, and desire for
affiliation. With the ever growing number of inclusions, the mission of the
PTL/drivers team of ensuring consistency in the APIs, architecture, design,
implementation and testing of the project, as well as a level of quality in
all aspects, like documentation, inter-project communication, release
management, stable backports etc are dealt with have become harder and harder.
The point about uniform APIs is particularly important, because the Neutron
platform is so flexible that a project can take a totally different turn in
the way it exposes functionality, that it is virtually impossible for the
PTL and the drivers team to ensure that good API design principles are being
followed over time. In a situation where each project is on its own, that
might be acceptable, but allowing independent API evolution while still under
the Neutron umbrella is counterproductive.

In a nutshell, the existing arrangement of conceding autonomy, and striving
for consistency is clearly challenging because of the two conflicting forces
at play. A better balance must be struck in a way that Neutron development
can continue on its trajectory of agile development without compromising
consistency, and without making the PTL, and the Neutron team a bottleneck
in the day to day management duties of the project.


Proposed Change
===============

The Neutron Stadium is the list of projects that show up in the following document:

https://governance.openstack.org/tc/reference/projects/neutron.html

The list includes projects that the Neutron PTL and core team are directly
involved in, and manage on a day to day basis. To do so, the PTL and team
ensure that common practices and guidelines are followed throughout the Stadium,
for all aspects that pertain software development, from inception, to coding,
testing, documentation and more.

The Stadium is not to be intended as a VIP club for OpenStack networking
projects, or an upper tier within OpenStack. It is simply the list of projects
the Neutron team and PTL claim responsibility for when producing Neutron
deliverables throughout the release `cycles <https://github.com/openstack/releases>`_.

This spec proposes how the current list (as of May 2016) needs to be revised in
order to maintain the integrity of the Stadium, and how to ensure this
integrity be maintained over time when modifications to the list are required.

When is a project part of the Stadium?
--------------------------------------

In order to be considered part of the Stadium, a project must show a track
record of alignment with the Neutron core project. This means showing proof
of adoption of practices as led by the Neutron core team. Some of these
practices are typically already followed by the most mature OpenStack
projects and should not be perceived as particularly stringent:

 * exhaustive documentation: it is expected that each project will have a
   developer, user/operator and API guide, reachable from docs.o.o;
 * exhaustive OpenStack CI coverage (unit, functional, tempest), using
   OpenStack CI (upstream) resources (this means using infra CI, not just
   downstream CI, as Grafana support will be required); access to CI
   resources and historical data by the team is key to ensuring stability
   and robustness of a project;
 * good release footprint, according to the chosen release model;
 * adherence to deprecation and stable backports policies;
 * demonstrated ability to do `upgrades <https://governance.openstack.org/tc/reference/tags/assert_supports-upgrade.html>`_
   and/or `rolling upgrades <https://governance.openstack.org/tc/reference/tags/assert_supports-rolling-upgrade.html>`_,
   where applicable;
 * adoption of neutron-lib (with related hacking rules applied), and proof
   of good decoupling from Neutron core internals; having unit tests executed
   against neutron-lib changes is also beneficial in order to show stability
   of the project;
 * develop client bindings and CLI according to OSC transition `plan <https://github.com/openstack/python-neutronclient/blob/master/doc/source/devref/transition_to_osc.rst>`_
   (where applicable). This means that Neutron Stadium projects will have
   their client side hosted in python-neutronclient.
   The problem stems from the fact that some service extensions do not
   release on pypi (like fw, vpn or lb) thus it is difficult to get hold of
   client extensions where they need to be used. SFC and L2GW, for example,
   do release on pypi so that is not so much a problem for them. In theory
   their client extensions can stay colocated within their tree but that
   would make more difficult the alignment to the OSC plugin model. Since
   the recommendation is to keep extensions in the python-neutronclient
   tree, it is beneficial to have a consistent approach where all Stadium
   client extensions are part of the python-neutronclient tree to avoid
   confusion when dealing with exceptions to the rule;
 * adoption of neutron-api: submission of any API change must follow the
   Neutron `RFE submission process <http://docs.openstack.org/developer/neutron/policies/blueprints.html>`_,
   where specifications to neutron-specs may be required;
 * adoption of modular interfaces to provide networking services: this means
   that L2/7 services are provided in the form of ML2 mech drivers and
   service plugins respectively. A service plugin can expose a driver
   interface to support multiple backend technologies, and/or adopt the
   flavor framework as necessary.

The last two criteria are particularly important for the following reasons:

 * To provide a consistent API experience to Neutron users/operators and assist
   Neutron developers more closely. This can only be achieved when API proposals
   and changes are funneled through a single initiative.
 * To provide composable networking solutions: the ML2/Service plugin framework
   was introduced many cycles ago to enable users with freedom of choice. Many
   solutions have switched to using ML2/Service plugins for high level services
   over the years. Although some plugins still use the core plugin interface
   to provide end-to-end solutions, the criterion to enforce the adoption
   of ML2 and service plugins for Neutron Stadium projects does not invalidate,
   nor does make monolithic solutions deprecated. It is simply a reflection of the
   fact that the Neutron team stands behind composability as one of the promise of
   open networking solutions. During code review the Neutron team will continue
   to ensure that changes and design implications do not have a negative impact
   on out of tree code irrespective of whether it is part of the Stadium project
   or not.

When a project is to be considered part of the Stadium, proof of compliance to
the aforementioned practices will have to be demonstrated typically for at
least two OpenStack releases. Application for inclusion is to be considered
only within the first milestone of each OpenStack cycle, which is the time when
the PTL and Neutron team do release planning, and have the most time available
to discuss governance issues.

Projects part of the Neutron Stadium have typically the first milestone to get
their house in order, during which time reassessment happens; once removed, a
project cannot reapply within the same release cycle it has been evicted.

What if a project cannot be considered part of the Stadium?
-----------------------------------------------------------

The aforementioned criteria imply that the Neutron PTL is unable to favorably
consider a project for inclusion, unless all the criteria are met. This in turn
imposes that new initiatives start as standalone projects hosted on the
openstack.org namespace, as outlined in the `creator guide <http://docs.openstack.org/infra/manual/creators.html>`_.
Furthermore, projects that interact with Neutron purely via its REST API should
also be incentivized to seek other forms of governance, such as applying as
top-level OpenStack project and follow the new project requirements as outlined
by the `OpenStack Governance <https://governance.openstack.org/tc/reference/new-projects-requirements.html>`_,
rather than going for Stadium inclusion.
Drivers that leverage proprietary software and/or hardware will also not be
considered for inclusion into the Neutron Stadium, due to the barrier on
access to the technology for dev/test and CI integration, as well as the inability
for the Neutron team to provide effective feedback during review for any aspect of
software development: from code to infrastructure and testing. This does not mean
that the Neutron team will stop collaborating with the individual teams supporting
these initiatives (as past experiences demonstrate), and in fact any team is
encouraged to seek collaboration in order to address integration issues with the
core Neutron platform. Teams for projects outside the Stadium are simply fully
responsible of dealing with the end-to-end SDLC of their solutions, and thus
empowered. The integration code to external systems can still be hosted on openstack.org
irrespective of their nature and get access to the same resources any other
OpenStack project uses. It is noteworthy that it may not be useful for them to seek
inclusion on governance.openstack.org in order to achieve 'recognition' in that
they are typically one layer underneath the projects that are considered for
inclusion by the TC. Besides, other programs, like the marketplace driver
certification program are more effective tools to reflect the level of quality
of a specific driver solution, as they also track the level of supportability
across the various OpenStack releases.

What if a project within the Neutron Stadium decides to include drivers for proprietary systems?
------------------------------------------------------------------------------------------------

Today's reality is that some Neutron projects still host drivers to proprietary
systems, like vpnaas, fwaas, and lbaas/octavia; even though the decision of accepting
new drivers lies with the respective teams, it is strongly encouraged that these be
exceptions rather than the norm. This point becomes less relevant if, in the long run
some of these efforts will no longer be considered responsibility of the Neutron team.

How do we build a Networking Community?
---------------------------------------

Neutron has been, since its inception, the networking project within
OpenStack. It has been *the* place where APIs and networking abstractions
are proposed and agreed on. There have been deadlocks and stalemates as
in any other project, and one way to resolve these stalemates is to
allow individual initiatives to experiment in isolation: ultimately code
wins. Even though this might seem okay at first, Neutron still lack the
ability to reconcile all these initiatives back together in a coherent
fashion.

In order to resolve this problem, the Neutron team should consider the
adoption of a neutron-api initiative where API proposals are consolidated.
The neutron-api project can be a component to be consumed by Neutron Stadium
projects, and to be started off as a spinout of the neutron api codebase such
as neutron/api, neutron/extensions, and any other project (e.g. bgp, l2gw,
sfc, etc) that is considered for Stadium inclusion. Contributions to this
module will follow the Neutron RFE process with submission of specification
documents to neutron-specs, if deemed required. The resulting code will end
up being located in the api module itself, whilst specs will be hosted on
neutron-specs. Each core team of projects belonging to the Stadium will
have +2 rights on `neutron-specs <https://review.openstack.org/#/admin/groups/314,members>`_.
The neutron drivers will continue to hold +W rights on both repos. It is noteworthy that
members of the individual core teams are and will be considered for membership
of the drivers team on a regular basis. Level of participation in reviewing
RFE submissions, experience with the Neutron codebase, and commitment to the
project are factors taken into account when identifying members of this team.
From an implementation perspective, the api module can either exist on its own,
as yet another repo within the Stadium list, or be incorporated as a /api module
under neutron-lib. Whilst the former approach allows finer control of the
release model, the latter has the obvious benefit of minimizing complexity in
landing end-to-end code changes.

The reason behind the existence of this effort and the requirement to go
through neutron-specs for API proposals is two-fold: a) ensure continuous
alignment across all APIs exposed from Neutron Stadium projects; b) provide
a place in the OpenStack networking community to discuss and achieve consensus
on networking abstractions; and c) as a robust foundation to build
interoperable APIs.

What happens to the current list of Neutron Stadium projects?
-------------------------------------------------------------

At the time of writing the Stadium contains the projects as outlined below.
Based on all the aforementioned considerations, all of individual teams will
have to work during the Newton timeframe to make the current proposal a reality.
More precisely, to be considered for inclusion (continue to be part of the
Stadium list), the following plan must be put in place:

* neutron: the core team will have to stand up neutron-api, and transition
  the api models/definitions out of the neutron repo. Developer documentation
  on the `Neutron Stadium <http://docs.openstack.org/developer/neutron/#neutron-stadium>`_
  will be revised according to the guidelines outlined in this proposal.

* neutron-fwaas: the fwaas core team will have to:

  * complete the transition to fwaas v2;
  * adopt neutron-api, and neutron-lib;
  * complete docs, end-to-end testing, and demonstrate ability to upgrade;

* neutron-lbaas: the existence of this project is now questioned by the
  presence of Octavia. This is elaborated more in `spinout <https://review.openstack.org/#/c/310805/>`_ spec.
  In a nutshell, this project will be phased out in favor of Octavia being
  an OpenStack top-level project.

* neutron-vpnaas: this project has seen the least amount of traction during
  the last few `cycle <http://docs.openstack.org/releasenotes/neutron-vpnaas/mitaka.html>`_.
  Testing and documentation is not adequate therefore the vpnaas core team
  will have to:

  * adopt neutron-api, and neutron-lib;
  * complete docs, end-to-end testing, and demonstrate ability to upgrade;
  * switch to OSC plugin;

* neutron-dynamic-routing: this is a spinout of BGP code that landed
  in the Mitaka timeframe. Currently in progress.

* networking-l2gw: this needs to:

  * adopt neutron-api, and neutron-lib;
  * complete docs, end-to-end testing, and demonstrate ability to upgrade;
  * move client side code to python-neutronclient and switch to OSC plugin;

* networking-bagpipe: the core team will have to:

  * adopt neutron-api, and neutron-lib;
  * complete docs, end-to-end testing, and demonstrate ability to upgrade;

* networking-bgpvpn: the core team will have to:

  * adopt neutron-api, and neutron-lib;
  * complete docs, end-to-end testing, and demonstrate ability to upgrade;
  * move client side code to python-neutronclient and switch to OSC plugin;

* networking-calico: the core team will have to:

  * adopt neutron-api, and neutron-lib;
  * complete docs, end-to-end testing, and demonstrate ability to upgrade;

* networking-midonet: the core team will have to:

  * adopt neutron-api, and neutron-lib;
  * complete docs, end-to-end testing, and demonstrate ability to upgrade;
  * move client side code to python-neutronclient and switch to OSC plugin;

* networking-odl: the core team will have to:

  * adopt neutron-api, and neutron-lib;
  * complete docs, end-to-end testing, and demonstrate ability to upgrade;

* networking-ofagent: this is deprecated and marked for removal.

* networking-onos: the core team will have to:

  * adopt neutron-api, and neutron-lib;
  * complete docs, end-to-end testing, and demonstrate ability to upgrade;

* networking-ovn: the OVN project will need to consider switching back to
  an ML2 driver. This is beneficial for the following reasons:

  * considered as a solution for multi-hypervisor deployments: this is
    currently only true if a hypervisor runs Open vSwitch.
  * present itself as a viable alternative to supplant the ovs mech_driver
    in the OpenStack gate: this can only be done  so if we are swapping
    like for like.
  * do bare-metal and virtual provisioning in the context of the same
    Neutron deployment; hardware vtep support only is limiting a number
    of potential bare metal scenarios that leverage third party solutions.
  * getting access to a number of performance/reliability enhancements
    that the Neutron team develops against ML2.
  * Composability of services across the entire L2-L7 stack.

* networking-sfc: the core team will have to:

  * adopt neutron-api, and neutron-lib;
  * complete docs, end-to-end testing, and demonstrate ability to upgrade;
  * move client side code to python-neutronclient and switch to OSC plugin;

* octavia: this is being tackled in the context of neutron-lbaas. In a
  nutshell, this will eventually be considered as top-level OpenStack
  project.

* python-neutron-pd-driver: this is a project that depends on Neutron IPv6
  support and introduces another prefix delegation driver besides the one
  available in tree. With the in-tree implementation of dibbler this is
  not as necessary any more. Dropping it in O-1 is an acceptable outcome
  if no-one picks it up by then.

Contributing the respective API extensions of these individual projects
to neutron-api does not mean reviewing the APIs from scratch as these
APIs already have working solutions behind them. Having said that, any
revision required, as potentially identified through review, will have
to be followed up according to RFE submission process.
These projects and their respective core teams will be given time to comply
until Ocata Milestone-1, after which, if not compliant, they will be removed
from the Neutron Stadium. All things considered, a team can decide not to
adopt the aforementioned plan and ask to be removed during the Netwon
timeframe.

This deadline is considered to be firm, but will be revised according to
how fast we can consolidate APIs and neutron-lib mature to a point of
effective consumption by the `neutron <https://github.com/openstack/neutron>`_
project and once thorough documentation is provided to help the various
parties through the transition.

How can an existing OpenStack project apply for inclusion?
----------------------------------------------------------

There are projects currently hosted on OpenStack that are not part of the
Neutron governance list, for example tap-as-a-service, and there may be more
that may be created in the future. Such initiatives should consider being
part of the Neutron Stadium if and only if there is a mutual and beneficial
collaboration already established across the teams involved. As mentioned
above, the Stadium is not an elite club but simply the projects that the
PTL and the Neutron team take responsibility for. The more involvement and
mutual collaboration exist over time, the more likely it is that there
is good alignement and thus Stadium inclusion becomes a a little more than
a formality.
In order to succeed in an application, a project should first and foremost
make sure that these criteria are met:

* documentation, testing and upgrade strategy are in good order as outlined
  earlier;
* client extensions available as OSC plugins;
* adoption of neutron-lib;

The request for application would then need to be submitted with an RFE and
subsequent spec submission where the following information is provided:

* pointers to user and developer documentations;
* pointers to server and client side internals;
* pointers to CI jobs (e.g. unit, tempest, grenade, etc) as well as Grafana
  dashboards;
* pointers to stable backports and release deliverables;
* pointers to API specification and definitions;

Once evaluated and successfully approved via RFE process, the teams will work
together to transition API and client components respectively to api module
and python-neutronclient repos. The newly added project pledge to
adopt the Neutron RFE submission process for API changes (and API changes
alone) and spec submission to neutron-specs where deemed necessary from that
point forward.

Why is this Stadium arrangement beneficial?
-------------------------------------------

The Stadium creates a place for like-minded people to collaborate, innovate,
and iterate on networking solutions, and provides a degree of guarantee that
a project belonging to the Stadium behaves similarly to any other project
within the Stadium, thus inducing a more sustainable maintenance effort on
the Neutron team. On the other hand, the Neutron team takes responsibility of
the deliverables produced by Stadium project (docs, release notes, etc), and
therefore giving these initiatives higher visibility to downstream consumers.
Contributors of Neutron Stadium projects participate in the Neutron PTL election
process and have the same status of any other OpenStack Active Technical
Contributor.

What are the drawbacks of this proposal?
----------------------------------------

The cost of sanitizing the Stadium and keeping it sane over time is non
negligible, though the criteria have been designed to be lightweight and impose
minimal burden on the teams once the Stadium is up and running. This is the
trade-off to keep all the networking related efforts working in symbiosis.
Hopefully, it should be clear by now that associating a bad connotation to
projects that are not part of the Stadium is simply a misconception: some
projects can indeed apply to be top-level projects in OpenStack, and others
are simply better off by being independent initiatives.


Are there any alternatives worth considering?
---------------------------------------------

Open to suggestions.

References
==========

* https://review.openstack.org/#/q/topic:stadium-implosion
* https://review.openstack.org/#/q/topic:stadium-redux
* http://lists.openstack.org/pipermail/openstack-dev/2015-December/080865.html
* http://lists.openstack.org/pipermail/openstack-dev/2016-April/093561.html
