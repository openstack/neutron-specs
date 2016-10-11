..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Postmortem documentation
==========================================

During the RC window of each release the PTL, with the help of anyone
who collaborated on the release, produces a detailed view of what was
tackled during the release. This document will capture what was completed,
if partially, what is still missing, and what areas deserve more attention.

The objective of this document is three-fold:

* It helps the PTL better identify how to improve planning for
  the forthcoming release, and better summarize the outcome of
  the release under target.
* It helps Neutron developers know where to focus energy for the
  initiatives that have partially completed, or that was planned
  but got deferred.
* It helps other members of the OpenStack community get a more
  cohesive picture of the planned features, and navigate through
  the many fragments that make up the entire picture for a feature.

Each entry in this document will capture only RFEs and Blueprints. The report
format would be like the following:

[Blueprint | RFE ] 'Title'
~~~~~~~~~~~~~~~~~~~~~~~~~~

* Status: [ Complete | Deferred | Partially Complete ]
* Assignee: The Launchpad handle for the assignee
* Link: The Launchpad link to the feature

  * CLI support: if the feature requires client support, report the status.
    Supporting notes about client support may be required.
  * Server/Agent support: if the feature spans both server and agents, API
    or model changes, report the status.
  * Testing coverage: report level of coverage and whether it is deemed adequate.
    Supporting notes on the level of coverage (api vs functional vs unit) may be
    provided.
  * Documentation: report the developer and/or user documentation, if required.

To the exception of documentation (for which the presence of pending patches
may suffice), the aforementioned areas of development are the ones that must be
marked Complete (i.e. patches merged) in order to claim the feature 'done'
for a given release. In addition to these, other areas to be documented may be:

  * Advanced/Sub-project support: for cross-project features, report the status
    of adoption for other Neutron repos besides 'neutron'.
  * Other Projects support: a feature may require integration with other
    OpenStack projects, e.g. Nova or Ironic. Report status of the integration.
    Supporting notes may be required.
  * OpenStack Infra support: some features may need to be tested in Gate.
    Report status in relation to infra support.
  * DevStack/Grenade support: some features may require DevStack or Grenade support.
    Report the status.
  * Horizon Support: some features may require Horizon support. Report the
    status.

Possible values to assign to these items are:

 * Completed (as planned).
 * Incomplete (planned, but failed to complete to be considered functional).
 * In progress (actively worked now at the time of writing).
 * Optional (not planned, but left for future planning).
 * Not applicable.
