..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Neutron Core and Vendor code decomposition
==========================================

https://blueprints.launchpad.net/neutron/+spec/core-vendor-decomposition

Contributing and reviewing existing and/or new vendor code in Neutron is painful
for a number of reasons. We are proposing changes to the existing structure
of the project to address these pain points. More precisely we are promoting
changes in the following areas:

* Code structure;
* Contribution process: this extends to the following areas:

  * Design and Development;
  * Testing and Continuous Integration;
  * Defect management;
  * Documentation.

Problem Description
===================

Vendor and community concerns are competing for scarce resources
because our process is currently one-size-fits-all. Holding
plugins/drivers close made sense in the early days, to build the community, but
Neutron has matured as a project.  We have the opportunity to evolve the
relationship between plugin/driver maintainers and the core to ensure that as
a community we can grow ever larger without incurring the bottlenecks and
issues we are currently facing.

Currently, we have 20+ plugins and 12+ ML2 mechanism drivers. There are
plenty of new plugin/drivers in the pipeline whose specifications and
code contributions need to go through the contribution process where:

* a developer submits a blueprint specification to the neutron-specs repo;
* a developer submits the code patch(es) to the neutron repo;

* at least two members from the core team need to (iteratively) review the spec;
* at least one member of the drivers team needs to approve the spec;
* at least two core reviewers need to (iteratively) review/approve the patch(es);

This model clearly does not scale because each core can only have limited
knowledge of some parts of the codebase and the sheer amount of vendor-specific
contributions that Neutron as a project has seen so far meant that no single
core reviewer can know it all. This model has also the following issues:

* Requires an artificial oversight from the core team, as core reviewers do
  not have the ability to fully understand and judge quality of vendor-specific
  source, sometimes in the form of interrelated patches made of many
  thousands lines of code; granted, a good static code review can help
  address issues like styling, use of data structures, algorithms,
  modularity, etc, but without the ability to fully understand all the moving
  parts it is sometime difficult to assess possible failure modes, potential
  bottlenecks, and even why certain fixes after the fact end up being necessary;
* Core review may sometimes require playing with the patch, which is made
  difficult, if not impossible, by the lack of access to proprietary and/or
  external systems;
* Prevents core reviewer resources from being solely focused on targeting core
  concerns of Neutron as a management service;
* Prevents vendors from having more control over the development and release of
  their own code;
* Prevents vendors from having different levels of engagement with the Neutron
  community: the existing model imposes a high acceptance bar. Vendor
  developers are invited to actively participate in the day-to-day management
  of the project, maintaining their code etc, even though they sometimes fail
  to do so. If the code contribution is substantial, it is only customary for
  these developers to participate in the cross-cutting issues that may arise
  (e.g. gate failures, oslo libraries refreshes, change in style guidelines,
  etc); in other words, the current process requires that a vendor generate
  goodwill through contribution to non-vendor initiatives such that cores are
  willing to review or address vendor's non-core contributions in a timely
  fashion. In an ideal world this would foster community development, but it
  presupposes that all vendors have similar resources to devote to Neutron and
  that the core team can scale non-linearly (neither of which is the case
  today);
* Does not promote clear separation of concerns, as the code is in one big
  bucket;
* The maintainance cost of the entire codebase is unevenly distributed across
  the Neutron team;
* Refactoring effort is hindered by the difficulty of handling code that the
  developer is not familiar with;
* Negatively impacts unit test feedback runtime, as a lot more tests need to run
  at once to for a specific change;

All these issues ultimately affects the pace at which the project can evolve.

It is worth noting that this problem, and potential solutions, may affect
different people in the OpenStack community in different ways. For instance:

* OpenStack Infra: what tooling would the team need to provide to help the
  project? Would they be impacted on a regular basis because of a change
  in the way the project decides to organize itself?
* Vendor Developers: how can a developer get his/her changes merged faster
  and with minimal overhead? Writing better code and participating more in
  core reviews certainly helps, but this is a bar that not every contributor
  can or is willing to set for himself/herself;
* Vendor Marketeers: how can a marketeer claim that his/her vendor plugin is
  OpenStack friendly? In this particular regard, the fact that vendor plugins
  are shipped as 'compatible' with OpenStack is largely independent from the
  code's origin. The perception that things in openstack.org trunk are
  'good' is in reality a misconception, and as such it is a task for the
  marketing team(s) to fix it.
* Distros: how can a distro package Neutron by capturing the code necessary,
  their external depedencies, and provide configuration tools that allow for
  easy consumption of the solution?
* Neutron developers: how can a developer support (non)vendor code
  contributions without burning out?
* Operators: do they even care so long as they can get access to a stable,
  upgradable and well documented product?
* OpenStack Technical Committee: would the TC members promote this effort,
  so long as OpenStack ideals like Openness, Transparency, Commonality,
  Integration and Quality are still enforced? As far as openness is concerned,
  any new project arrangement must guarantee that the four opens are still
  in place (open source, open design, open development, open community [17]);
  TC members also represent the Active Technical Contributor, so it is of
  paramount importance for them to alleviate the pain in contributing to
  OpenStack;
* OpenStack Foundation: so long as the job of promoting the development,
  distribution and adoption of the OpenStack cloud operating system is not
  hindered by any attempt at making a project functioning better, they should
  actually welcome the effort. The Foundation has also the duty, with the
  marketplace on openstack.org, to promote drivers compatibiility and soon
  there will also be RefStack [14];

Having said that, the proposed change will primarily address the development
challenges that the Neutron community faces today, leaving social or marketing
perceptions and concerns thereof, a solution to be sought with guidance of
the OpenStack Foundation.

Proposed Change
===============

The proposal being made here is an initial step, and something it has been
deemed attainable in the span of a single release. It is not by all means
the end goal, but a good balance that will let us find the golden mean [5]
in iterations. It is also worth noting that this effort is about the core
plugins, rather than advanced services like Load Balancer, VPN and Firewall.
Spinning off those services will be tracked with a different blueprint [8].

High Level Codebase Structure
-----------------------------

We propose that:

* The *monolithic plugins*, *ML2 MechanismDrivers*, and *L3 service plugins*
  become integration-only to code that lives outside the tree (more details
  provided in the section below); the same applies for any vendor-specific
  agents: the only part that will remain in the tree is the agent 'main' (a
  small python file that imports agent code from the vendor library and
  starts it). L3 being an integral part of Neutron core will go through the
  same process as core plugins and ML2 drivers. The 'outside the tree' can be
  anything which the vendor is comfortable with: it may be a stackforge repo
  for instance, a tarball, a pypi package, etc. It is then important that this
  vendor library be publicly accessible for a number of reasons:

  * Potential licensing conflicts, that might be solved only by getting access
    to the vendor library, but more importantly under no circumstances should
    a key OpenStack project like Neutron be encouraging closed source
    development of a critical component such a plugin or mechanism driver;
  * Ease of packaging support from distros; even though in most cases, distros
    will have a relationship with vendors they want to package and validate,
    there may still be pure open source players, that may be interested in
    getting to use the undelying technology, should that be open source itself.

  A plugin/drivers maintainer team self-governs in order to promote sharing,
  reuse, innovation, and release of the 'out-of-tree' backbone. It should not
  be required for any member of the core team to be involved with this process,
  although core members of the Neutron team can be in whichever capacity
  necessary for out-of-tree development.

Development Strategy
--------------------

* The following elements are suggested to remain in the tree for all plugins
  and drivers (called vendor integration hereinafter) - at least in the first
  iteration of this effort:

  * Data models;
  * Extension definitions;
  * Configuration files;
  * Requirements file targeting vendor code;

* Things that do not remain in the tree (called vendor library hereinafter):

  * Vendor specific logic;
  * Associated unit tests;

The idea here would be to keep in-tree the plugin/driver code that implements
an API, but have it delegate to out-of-tree code for backend-specific
interactions; the vendor integration will then typically involve minor
passthrough/parsing of parameters, minor handling of DB objects as well as
handling of responses, whereas the vendor library will do the heavylifting and
implement the vendor-specific logic. The boundary between the in-tree layer and
the out-of-tree one should be defined by the maintainer while asking these
types of questions:

  * If something changes in my backend, do I need to alter the integration
    layer drastically? Clearly, the least impact there is, the better the
    separation being achieved;
  * If I expose vendor details (e.g. protocols, auth, e.g.), can easily swap
    and replace (e.g. a hardware with a newer version being supplied) without
    affecting the integration too much? Clearly, the more reusable the
    integration the better separation.

Furthermore, we can observe that:

  * Regarding db models, we would like to keep backward compatibility
    (for already deployed systems) at the present time. There are a number of
    technical points to be addressed before moving db models out of tree
    completely, e.g., FK, joined query, db migration for existing deployments
    and so on, as currently alembic requires extra work to support multiple db
    migration paths onto a single database [16]. This technical challenge can
    be dealt with in due course, once the major parts of this proposal have
    been completed;
  * Moving config files elsewhere (or even relying on config autogeneration
    capabilities) could be explored as a next step. We have been tracking
    config changes with DocImpact flags to ensure that OpenStack Manuals are
    kept in sync, and we should not break this model;
  * The vendor code *must* be publicly available via pypi, or public
    source repository compatible with pip requirements (if packaging is
    desired); this requirements file should not be confused with the Neutron
    requirements file that lists all common dependencies); instead it is a
    file 'requirements.txt' that is located in neutron/plugins/pluginXXX/,
    whose content is something along the lines of
    'my_plugin_xxx_library>=X.Y.Z'. Vendors should be responsible
    for ensuring that their library did not depend on libraries conflicting
    with global requirements, but could include libraries not included
    in the global requirements. If the vendor library depended on a python
    package not captured in the global requirements, this is no different
    by what already happens today and we are not trying to address this
    in the scope of this effort.
  * Versioning can be used to pin vendor code to specific versions;

* For the reference implementation: they will still follow the same model, but
  the implementation will remain in tree, due to gating requirements; the
  reason being that there is no way to externalize the reference implementation
  until we have backwards compatible interfaces in the mix. Having to
  coordinate the landing of a breaking change in core neutron repo and a fix
  in the reference implementation repo so as to avoid breaking openstack
  integration jobs would be an extremely tricky proposition.
  The proposal documented here is intended to encourage plugins/drivers to
  separate themselves into a part that integrates with Neutron and a part that
  integrates with a backend. Given that most breakage is going to be in the
  integration with Neutron (which remains in the tree) - and that breakage will
  only be allowed when fixes can reasonably be expected to be landed before the
  end of cycle (e.g. before milestone2) there should substantially less risk
  for non-gating plugins/drivers. Trunk will always work: for breaking changes
  that can be detected, we can ensure that at least the in-tree part of plugin
  drivers are fixed before merge. Plugin/driver-specific breakage will have to
  be fixed by the maintainer of that plugin/driver, but as a community we
  should address the potential for that kind of breakage separately.

* No DevStack changes required: Devstack already has the right hooks [6,7,10];
  moreover, the vendor is in full control of the 3rd party CI environment;
  they could just install the needed dependencies out of band, while the CI
  is setting up the environment and before stacking. Some of them already do
  all sorts of stuff.

For instance a vendor integration module can become as simple as one that
contains only the following:

* Registering config options;
* Registering the plugin class;
* Registering the models;
* Registering the extensions.

For instance:

::

>    class DummyPlugin(db_base_plugin_v2.NeutronDbPluginV2,
>                      your_mixin_here):
>
>    supported_extension_aliases = ["the_extension_you_implement"]
>
>    __native_bulk_support = True
>    __native_pagination_support = True
>    __native_sorting_support = True
>
>    def __init__(self):
>        super(DummyPlugin, self).__init__()
>        config.register_config_opts()
>        self.vendor_plugin_init()
>
>    db_base_plugin_v2.NeutronDbPluginV2.register_dict_extend_funcs(...)


The Nuage plugin [2, 3] follows a slightly similar approach. Since unit test
must be purely testing the units that make up for the vendor library, they no
longer make sense to be part of the Neutron codebase. Having said that, the
vendor integration should be tested via functional testing. The VMware plugin
was also going down that direction [4], where the plugin definition becomes
purely declarative.

Testing Strategy
----------------

The testing process will change as follows:

* There will be no unit tests for plugins and drivers in the tree, except the
  reference implementation. The expectation is that vendors would run unit test
  in their own external library (e.g. in stackforge where Jenkins setup is for
  free); For unit tests that validate the vendor library, it is responsibility
  of the vendor to choose what CI system they see fit to run them; there is no
  need or requirement to use OpenStack CI resources if they do not want to;
  ultimately these tests can run as part of the 3rd party CI system [9] that is
  currently required, based on specific filters, if necessary. It is noteworthy
  that this effort may unveil areas of the core code that turn out to be barely
  tested. We should strive to identify those, and improve their coverage so
  that, in turn, the improvements can be beneficial for the various
  vendor integrations as well, as added/revised tests will exercise the shared
  framework;
* 3rd Party CI will continue to validate vendor integration with Neutron via
  Tempest (e.g. functional testing); 3rd Party CI is a communication mechanism.
  This objective of this mechanism  is threefold:

  * it communicates to plugin/driver maintainers when someone has contributed
    a change that is potentially breaking. It is then up to the a given
    maintainer to determine whether the failure is transient or real, and
    resolve the problem if it is real;
  * it communicates to a patch author that they may be breaking a plugin/driver.
    If they have the time/energy/relationship with the maintainer of the
    plugin/driver in question, then they can (at their discretion) work to
    resolve the breakage;
  * it communicates to the community at large whether a given plugin/driver
    is being actively maintained.

  Gating on 3rd party CI *is* a terrible idea, though. It introduces too much
  variability into the merge process, and we already have more chaos than we
  can handle. The proposal in question attempts to minimize some breakage
  by keeping plugin/driver interfaces in-tree. We could add sanity unit tests
  of interface integration, but it appears that integration failures should
  be more effectively caught at a functional and integration level rather
  than unit one. Having said that, nothing outlined here would prevent to
  adjust test coverage over time.

Review and Defect Management Strategies
---------------------------------------

We propose that:

* The same bug management process applies: bugs that affect vendor code can be
  filed against the Neutron integration, if the integration code is at fault;
  otherwise, the code maintainer may decide to fix a bug without oversight, and
  issue a new version pin against the requirements file for the code in question
  in case the vendor library is being pinned; it makes sense to require 3rd
  party CI for a given plugin/driver to pass when changing their dependency
  before merging to any branch (i.e. both master and stable branches);
* The same review process applies: vendor specific code follow the same review
  guidelines as any other code in the tree, however, the code being reviewed
  strictly applies to integration code or requirements' version bumps; as for
  the vendor library, it becomes an external repo; the vendor can choose anyone
  to approve/merge changes in this repo;
* The same release process applies: this does not change;
* The same security vulnerability process applies: this does not change. However,
  since the attack surface is a lot larger for the vendor library, it is most
  likely that the vulnerability is found there, and the maintainer will need to
  issue a fix according to his/her own procedures; if a vulnerability is found in
  the vendor integration, the vulnerability fix will need to follow the same
  procedures are defined by OpenStack. Vendor may decide to publish the
  vulnerability on LP or any other tracking system: completely up to them.

Blueprint Spec Submission Strategy
----------------------------------

We propose that:

* Provided vendors adhere to the limited development footprint laid out in the
  proposal, they should not be required to follow the spec process for changes
  that only affect their vendor integration and library.
* Proposal of new vendor plugins and drivers should no longer follow the
  spec submission process in place for any other Neutron contribution, so long
  as the vendor plugin/driver adopts the guidelines outlined in this proposal.
  New contributions could simply be submitted for code review, with the proviso
  that adequate documentation and 3rd CI party is supplied at the time of
  the code submission. For tracking purposes, the review itself can be tagged
  with a Launchpad bug report, marked as wishlist; design documents can still
  be supplied in form of RST documents, within the same vendor library repo,
  for documentation purposes;
* Blueprint specifications for new vendor plugin/drivers that target the
  current release cyle for Neutron will therefore be evaluated with the
  assumption that this proposal goes ahead, and hence marked for abandonment.
  Code submission can still go ahead, and evaluated according to the
  guidelines laid out in this proposal. Code inclusion will most likely
  considered in time for the third milestone of the cycle to allow enough
  time for the core team to adequately demonstrate this proposal in practice.

Documentation Strategies
------------------------

There is a sister proposal for docs referenced in [15]. The intention going
forward is that the documentation team will fully document the reference
plugins/drivers and add short sections for vendor plugins and drivers, in
the same spirit promoted by this proposal (about code decomposition). Existing
plugins and drivers will have to add references to additional docs, just like
new ones will, once their vendor integration merges in the Neutron repo.

Adoption And Deprecation Policy
-------------------------------

We propose that:

* All new (core or L3) plugins and drivers follow this model from Kilo;
* It is strongly recommended that existing plugins and drivers adopt
  this model in the Kilo cycle, and structure the code according the
  guidelines outlined above;
* No in-tree features can be added to existing plugins and drivers: this
  means that the feature of existing plugins/drivers code in the main
  Neutron tree will be frozen for the Kilo release; some may argue that the
  freeze might not be necessary and we could still rely on some review
  best effort. However, it is important to recognize that strict discipline
  is required to see this proposal through;
* Only critical bug fixes will be permitted to existing plugins and drivers;
  ML2 needs to be excluded, at least for now, because we gate on ML2+OVS
  which precludes splitting any part of it out in advance of backwards
  compatible interfaces being introduced. It is nonetheless proposed that we
  refactor the OVS ML2 driver such that the drivers themselves contain the
  same minimum footprint proposed for all drivers and that they are cleanly
  separated from the backend implementation.
* Plugins and drivers that are still 'as is' (i.e. not complying with this
  proposal) at the end of Kilo will be evaluated for deprecation and removal;
  maintainers will need to make a positive effort to start this process in
  the Kilo timeframe; process well underway but to complete in the Lxxx
  will still be acceptable; however, it is paramount that maintainers
  prioritize their resources so that the rewriting process is not deferred
  until Lxxx, to minimize the impact of the code freeze, and to mitigate the
  potential danger of deprecation and removal in time for Mxxx.

Having said that, the deprecation target and freeze will need to be assessed
at the various Kilo milestones, to establish how close to completion this
proposal is, and how far each vendor maintainer is in completing the tasks
outilined in the 'Work Items' section below. Should this proposal get the go
ahead, then the clock will be set and actions will be tracked for each plugin,
ML2 driver, as well as the exent of progress being made (e.g. setting up of
repos, patches addressing the relevant code, etc).

NOTE: No action or lack of engagement from the interested parties will need to
be tracked to ensure that, deprecation is considered a last resort measure.

Summary
-------

The approach being proposed here tackles all the issues (outlined above) that
Neutron developers have grown so tired of; it does so by delegating some
control to the vendor, whilst retaining some of degree of visibility into what
is considered part of the Neutron project.

FAQ
---

Add your question here and we will try to answer to best way we can:

* Q: If ML2 plugin refactoring is pursued by the core team. Code-changes for
  the same can affect the interfaces and behaviour of existing large set of
  vendor mech-drivers in the neutron tree. So, does the vendor mech-drivers
  continue to remain in core tree.
* A: The proposal is not to move drivers out of the tree. Rather, the
  intention is to encourage a separation of concerns such that plugins and
  drivers are as lightweight as possible, and serve primarily to integrate
  with libraries external to the tree that provide backend-specific
  implementation. This would have the advantage of allowing vendors greater
  control over their backend-specific code and and offer the possibility of
  allowing changes to plugins/drivers with a lighter-weight process (no spec
  process and faster merging).
* Q: Wouldn't it be easier if the three code artifacts (Model, Extensions,
  and config files) also lived in the vendor repositories? By leaving the
  above three in-tree, the proposal still puts significant burden on the
  Neutron cores to review vendor code artifacts.
* A: Short answer is No, as (hopefully) the artifacts are fairly small and
  easy to understand. The long answer is: the effort required for full
  separation is considerable. The suggested approach would allow plugin/driver
  maintainers to gain control over their efforts faster and with less risk. It
  will most likely take the entire cycle to get everyone on board and adjusted
  to the new model, which does not require any additional work, like new hooks,
  different DB timelines, etc. but just the 'mechanical' translation.
  Once that is cleared out, we can then focus on refining the model and pursue
  further delegation to vendor code.
  We also need to consider the migration path: we have a couple of things
  to consider on splitting db model: for data models, we need to consider the
  db migration path and FK relationship to Neutron core models (networks,
  ports, etc). Coming up with a concrete plan on how this can be handled ahead
  of this proposal is considered a risk worth deferring. API extensions and
  configuration files may be able to be moved out, but it has been considered
  useful documentation material, at least for the first iteration.
* Q: The DB models for each vendors will stay in the main repo. This probably
  avoid the alembic challenge for migrations, but it also confuses me a bit
  when it comes to synchronizing models in master and versioning how the plugin
  library. In other words, if you release a version of a library which requires
  changes in the data model, then you will have to wait for data model changes
  to merged before pinning Neutron to that version of the library. What should
  the process be in this case?
* A: Ideally, the model changes are backward compatible, but if they were not,
  then, as outlined, the process is still pretty straighforward.
* Q: What is the distro packaging strategy? For instance, do we need to define
  versioning policy for a vendor plugin module? For example, vendor plugin
  module for Kilo should have a version of 2015.1.N?
* A: We do not require openstack specific versioning of other python
  dependencies, and we should not require it of the plugin/driver dependency
  libraries either. We could rely on the per-plugin/driver requirements files
  in a given Neutron release to authoritatively determine the version of the
  library required for a given plugin/driver for that release, and the version
  could be updated on stable branches as required.
* Q: It seems that the plugin library maintainer is expected to provide
  packaging for the appropriate distros. Is this true?
* A: Just as distros are responsible for determining how to package
  plugin/driver dependencies today, they would continue to do so under this
  proposal. Their job would be made easier by the addition of a plugin/driver
  requirements file that could be used to determine plugin/driver dependencies
  that currently require manual discovery. The expectation here is that the
  plugin business logic will have its own lifecycle in a library which is not
  part of OpenStack. The relationship between Neutron and the vendor library
  would be for instance the same that there is between Neutron and sqlalchemy.
  Therefore versioning for the vendor library can be whatever one sees fit.
  The plugin itself however, stays in the main neutron repository, but it is
  minimised in a way that it could be just a stub. We will still have a Neutron
  "vendor X" plugin for openstack 2015.1
* Q: How do we deal with Oslo integrations?
* A: When oslo integrations occur, 3rd party CI (and a plugin/driver maintainers
  internal testing) should detect any breakage in the external dependencies of
  the plugins/drivers. We may want to announce when these kinds of changes are
  expected such that plugin/driver maintainers can respond in a timely fashion
  and minimize the duration of job failure. Only if plugin/driver maintainers
  were completely unresponsive to breaking changes in the Neutron tree would
  the possibility exist of those changes making it all the way to production.
  Were a maintainer to be that unresponsive, the inclusion of their plugin
  in the tree would likely be in question. The aim here is to relieve the core
  team of some of the maintainance burder in addressing these types of issues.
* Q: How can we ensure that API's will not be changed and versioned? Say a
  internal method of a mixin is updated - then a vendor overwriting or using
  this may miss this. In other words, we do not have stable API's, therefore
  in the current model breaking changes can be prevented by doing a one
  sweep-fix-all type of patch. With the model being proposed here, how are we
  going to deal with this type of issue?
* A: The proposal is to keep as strict a separation as possible between Neutron
  and vendor concerns, and continuing to develop our test suite to catch
  breakage (API or otherwise) when it occurs. Provided plugin/driver maintainer
  is diligent in following up on breakage, the only thing that will change
  under this proposal is how fast breakage will be fixed (same patch vs one
  patch to Neutron + one patch per external plugin dependency). The fact that
  the cost of breakage will be increased could actually be a positive thing if
  it can encourage the community to take more care in providing cleaner
  separation between concerns (via APIs or otherwise). Today both the
  integrated gate and non-gated plugins happen to break from time to time; so
  long as there is a prompt reaction to the issue, we do not see this model
  exacerbating this issue further. As a matter of fact, CI's are getting
  better and the team is getting better at catching these problems and fixing
  them swiftly; all we need is to be engaged!
* Q: Can you please elaborate a little more on the governane of new repos?
  There is at least 1 maintainer needed. How many minimum number of members?
  If there is 1 member, he/she will be committing, reviewing and merging
  his/her own code?
* A: This proposal does not mandate any kind of governance (or even distribution
  via a repo) of a given plugin/driver dependency library. We would continue to
  require an in-tree maintainer/point of contact for each plugin/driver, as we
  do today, and that maintainer would be responsible for coordinating with
  out-of-tree development.
* Q: Previously, if there was any change affecting plugins/drivers, it was done
  in all the code paths to-be-affected to make sure nothing breaks. However,
  with split repos, how to make third party CIs pass on changes which break
  vendor code and require a change across two or more repos? Looks like we
  will have to override third party CI vote either in Neutron or vendor
  plugin/driver repo? If the override is absolutely needed, where should it
  be done?
* A: 3rd party CI votes but does not gate, and this proposal in no way changes
  that. When breaking changes land in the Neutron tree, it will be up to the
  plugin/driver maintainers to resolve the breakage on their own timeline. In
  the short-term we are introducing coordination overhead by externalizing
  backend-specific code, but in the long-term this should encourage a decrease
  in coupling that minimizes such costs.
* Q: How do we define some vendor plugin is a part of OpenStack release? Do we
  define it? Defined by third party testing integration? It seems related to
  distro packaging policy to some extent.
* A: We can define whether a plugin is part of a given release is the answer to
  the question: 'does the plugin/driver exist in the Neutron tree?'. The
  question that we do need to answer is 'under what conditions is a
  plugin/driver allowed to be in the tree'? We are going to have to rely on a
  combination of relationship management (PTL <-> plugin/driver maintainer) and
  3rd party CI to answer that question, and this proposal does not need to
  answer this question. Some plugins already rely on a variation of the
  proposed model, and there does not seem to be any issue with distribution.
* Q: Should Third party CI result be posted to Neutron core change? We need to
  define third party testing requirements?
* A: They can choose to do so, if they believe the change may break their
  support, but this already happens today; 3rd party CI requirements need to
  be clarified regardless of this proposal.
* Q: Does this proposal make any effort into an ML2 Agent no longer feasible?
* A: No, effort into developing an ML2 agent would not be the best use of
  community resources. For historic reasons Neutron has grown to be not only a
  networking orchestration project but also a reference implemention that is
  resembling what some might call an SDN controller. We need to move away from
  this model, and for these reasons, pursuing an ML2 agent is no longer viable.
* Q: What about Linux Bridge?
* A: Linux Bridge is not currently the option we choose to focus both the
  development and testing; any effort in maintaining it at par with OVS would
  require resources that need to be identified and quantified. In lack of those
  resources, Linux Bridge has already fallen behind and will not be considered
  in the refactoring effort, unless someone else is willing to sponsor it.
* Q: How do we deal with imports of Neutron specifics (like utility functions,
  constants, base artifacts, etc) in the vendor library?
* A: External libraries that implement backend-specific interaction for
  plugins/drivers are expected to explicitly depend on Neutron to allow them
  to reference things like db models, utility functions and constants. Since
  the Neutron's dependency on the external libraries is not explicit
  (plugin/driver requirements file will not be considered by pip), a circular
  dependency will not result. The model would follow loosely what any OpenStack
  project already does when referencing Oslo. When it comes to pinning to a
  specific 'snapshot' of the core codebase, git hashes could be used. The model
  would need to ultimately evolve so that a better demarcartion will occur
  between core and vendor specifics, which makes the dependency less of a
  potential problem.
* Q: What if a vendor library uses an API defined in the core?
* A: Reuse of some plugin methods by an external library is possible through
  dependency injection, which are fancy words for 'pass a reference to the
  plugin to the code that needs to call its methods'.
* Q: With this approach, would packagers ignore some of the plugins in tree by
  not making a package for their library?
* A: Before the split, all plugins were 'magically' packaged by distributions,
  at least their source code. The packager then explicitely needed to add
  'control' rules to integrate the various parts together with the OS, and/or
  config management tools to reflect the configuration choices in the respective
  configuration files. This does not necessarily endorse validity of the
  vendor solution on the platform, but packagers were left with no choice but
  to follow this route in order to avoid dangling code being packaged and
  distributed. With this approach, consuming the actual source code becomes
  optional, and it is up to the vendor to decide to continue to preserve the
  existing packaging model by pulling from the various sources. In other words,
  this opens up the question whether all distributions should consider if it is
  worth the effort of shipping plugins that are not explicitely covered by
  relationship agreements with vendors. Every distro has different internal
  processes for tracking packages and releases, and it sensible that distros
  take the appropriate actions to ensure the overall quality of the Neutron
  solution they sell and support. We should emphasize that according to this
  proposal, long term, some plugins may loose packaging and distribution in the
  commercial editions of these distros, but at the same time, open source
  initiatives can still go ahead unaffected.
* Q: The fact that many vendor-specific plugins/drivers are currently in-tree,
  creates the impression that a new vendor has 'failed' in some sense if they
  do not get their new plugin/driver in tree as well - where 'failed' has
  various possible connotations including 'not being guaranteed to work with
  OpenStack, and to continue to work as OpenStack evolves', and 'not having
  demonstrated sufficient engagement with the OpenStack community'. Does this
  proposal help in addressing this aspect of code contribution?
* A: Absolutely! First of all, let us be clear of one thing: in-tree does not
  equate success, and out-of-tree does not equate failure. This is a perception
  that somewhat built over time into people's minds, because of past events
  that led to the removal of code of questionable quality or because of lack
  of testing (most notably Hyper-V in Nova). Today, the reality is completely
  different, OpenStack has evolved and 3rd party CI plays a fundamental role
  in establishing the sanity of all the components in motion in an OpenStack
  based cloud. Furthermore, with this proposal, the failure for not getting
  new vendor support in the tree is mitigated by the fact that it is a lot
  easier for the core team to go through the checklist of steps to be taken
  to consider a vendor integrated with Neutron, without being encumbered by
  the need of going through multiple review cycles to get the plugin/driver
  in the tree.

Data Model Impact
-----------------

N/A

REST API Impact
---------------

N/A

Security Impact
---------------

N/A

Notifications Impact
--------------------

N/A

Other End User Impact
---------------------

From an end-user standpoint (the tenant or admin that interact with the Neutron
API); this approach has no impact on them.

Performance Impact
------------------

N/A

IPv6 Impact
-----------

N/A

Other Deployer Impact
---------------------

Minimal. Depending on how Neutron is consumed, deployers need to pull an
extra dependency, which is the vendor library of the plugin of their choice.
This dependency can be pulled in in various ways, with or without the actual
support from the OS/Distro the deployer is using to consume Neutron.

It is worth mentioning that this approach empowers the distros (namely the
packagers) to choose where to focus resources more effectively. This means
that they can now choose to target packaging only for specific vendor
solutions more explicitely. Although this may sound like a setback for some,
the benefit of automatic packaging was only giving false reassurance that the
overall Neutron solution was going to be free of supportability issues, or
that it was implicitely validated by either the vendor or the distro. Any
customer should do the appropriate due diligence when adopting the combination
of Neutron+Backend, should they choose to access the overall Neutron solution
via two distinct channels. This proposal does hinder this process.

Developer Impact
----------------

As outlined in this proposal.

Community Impact
----------------

Very minimal, or nearly none in terms of diverging drastically from the way
we operate today. As a matter of fact, we believe there will be a positive
community impact:

* Cores able to iterate faster on the core;
* Better separation of responsibilities;
* Vendors able to iterate faster into their plugin;

Alternatives
------------

Alternatives could be explored with or without tooling assistance:

* Increase the number of Neutron cores (1)

  We could allow every vendor to have core(s) reviewers so that they can
  approve only their own code (i.e. subtree delegation). This poses an
  interesting set of challenges:

   * What do we do with new contributors?
   * What does core developer actually mean?
   * How can we ensure a coherent standard of code reviews?

* Rubberstamping from Neutron cores (2)

  Neutron cores can become paper pushers and blindly approve changes
  that pertains only to a specific vendor. This poses an interesting
  set of challenges:

  * How do we ensure that cross-cutting concerns are not overlooked?
  * How do we ensure that vendor maintainers are still held accountable?

* Adopt a different decomposition of the source tree

  The existing source tree could be decomposed in separate repositories:

  * One repo hosts core elements (DB, API, RPC, etc) and fully open source
    implementation of L2/L3 API that is currently gated;
  * One repo for each vendor, that has contributed one or more (monolithic)
    plugin;
  * One repo for all ML2 mechanism drivers; A vendor that contributed both
    a monolithic plugin and an ML2 driver should decide where the ML2 driver
    needs to belong;

  Ideally this is decomposition that makes the most sense, if it was not for
  the political and social rules that we are constrained by. In other words,
  from a pure software development standpoint, this decomposition makes the
  most sense, and any objection to this approach could be overcome with
  technical solutions. That said, it may be too hard to pursue this approach
  in the span of a single release. It might make sense to reassess this
  approach in a release from now, once the codebase has gone through this
  initial transformation and other efforts (e.g. advanced service spin-off,
  API refactoring, etc.) have been completed.

* Using git submodules to decompose the source tree

  This has been strongly discouraged by OpenStack Infra, because of the
  complications of managing git submodules:  not only does it add potential
  complications in CI tooling, but it is nearly indistinguishable from using
  separate repositories and as such does not really address any of the
  concerns raised with separate repositories.

* Using feature branches

  This could be achieved by using long lived branches (similar to the Linux
  kernel model where subteam maintainers have there own branch). Short lived
  branches do not really lend themselves to representing the lifespan of a
  plugin/driver. Even though one of the perks of feature branches is that
  they can own different ACLs so a different group of people can get +2/+W
  on them, the drawback to this approach is that it does not promote clear
  separation and responsibilities between the various parts of the system.
  In other words, if we had stable interfaces between these parties, using
  feature branches would not provide any benefits more than having these
  parties reside in different repositories.

Commentary

While approaches 1 and 2 may help address change velocity, they may
ultimately lead to an increased instability of the codebase, they do
not promote cleaner interfaces, and they weight down the Neutron team
that would need to deal with a much, much bigger codebase. In other
words, even though these approaches may make sense in the short term,
they may seriously affect the project in the long run.

Implementation
==============

Assignee(s)
-----------

For the reference implementation refactoring, code and test re-org:

* Armando Migliaccio
* Maru Newby

For examples of how this proposal develops in practice:

* Kevin Benton (Big Switch)
* Doug Wiegley (A10)
* Kyle Mestery (OpenDayLight ML2 Mech Driver)
* Yamamoto Takashi (ofagent)
* Gary Kotton (VMware)
* <add your name here>

For new plugin/driver that want to pioneer this model:

* Sandhya Dasu (Cisco UCS Manager ML2 mech driver)
* Vivekanandan Narasimhan (OVSvApp [12, 13])
* <add your name here>

Work Items
----------

* Announce new policy and deprecation timeline for existing plugins/drivers;
* Separate reference implementation in a similar manner but keep the code in
  tree (ML2 OVS mech driver + L3 service pluign with l3-agent, which is what
  it is currently gated on); This is of paramount importance in order to
  flesh out any remaining technical details and pave the way for vendor code
  maintaners to follow in the their own code.

  * the plugin becomes the integration point to the bulk of the code held in
    something like neutron/reference_code.
* Suggested work items for each vendor:

  * Setup a repo for hosting the vendor library (stackforge is useful because
    it provides jenkins integration); there is no naming convention forced
    down on vendors, however, should the stackforge route be pursued, vendor
    should avoid naming the repo with explicit reference to Neutron: this
    protects the vendor in case new naming issues arise (anyone remembers the
    Quantum naming debacle?), and it allows the vendor to have the freedom
    to target the same Python bindings for potentially different CMS;
  * Setup code in new repo: it is worth noting that if code is being copied
    out of neutron core and into these drivers, the git subtree tool can
    preserve history across this operation. OpenStack infra is doing something
    similar with puppet modules on [11];
  * Setup unit tests;
  * Publish code to pypi;
  * Change neutron driver/plugin to import pypi module and remove all vendor
    internal logic;

Dependencies
============

N/A

Testing
=======

For more details, see section on Testing Strategy.

Tempest Tests
-------------

N/A

Functional Tests
----------------

N/A

API Tests
---------

N/A

Documentation Impact
====================

For further details, see section on Documentation Strategies.

User Documentation
------------------

After this proposal, the overall Neutron system will not change in terms of how it
functions. From a user perspective, once the system has been deployed, configured
and up and running, no difference can be perceived. However, a sibling proposal
has been documented in [15]. Operators will be expected to follow documentation
and/or pointers to external references provided by the Upstream OpenStack docs
to understand how to install and configure the system to use specific plugins and
drivers.

Developer Documentation
-----------------------

No API change has been proposed here, so providing guidelines for contributing
a plugin/driver will suffice. This proposal already outlined quite extensively
some of the work involved, and will be used as base for the in-tree developer
documentation.

References
==========

* [1]  https://etherpad.openstack.org/p/aE7ydRU35m
* [2]  https://github.com/openstack/neutron/tree/master/neutron/plugins/nuage
* [3]  https://github.com/openstack/neutron/tree/master/neutron/tests/unit/nuage
* [4]  https://github.com/openstack/neutron/blob/master/neutron/plugins/vmware/plugin.py
* [5]  http://en.wikipedia.org/wiki/Golden_mean_(philosophy)
* [6]  https://github.com/openstack-dev/devstack/tree/master/lib/neutron_plugins
* [7]  https://github.com/openstack-dev/devstack/tree/master/lib/neutron_thirdparty
* [8]  https://review.openstack.org/#/c/136835/
* [9]  http://ci.openstack.org/third_party.html
* [10] https://review.openstack.org/#/c/137054
* [11] http://specs.openstack.org/openstack-infra/infra-specs/specs/puppet-modules.html
* [12] https://review.openstack.org/#/c/136091/
* [13] https://review.openstack.org/#/c/104452/
* [14] https://wiki.openstack.org/wiki/RefStack
* [15] https://review.openstack.org/#/c/133372/
* [16] https://github.com/stackforge/group-based-policy/tree/master/gbp/neutron/db/migration/alembic_migrations
* [17] https://wiki.openstack.org/wiki/Open
