..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Adopt Oslo Guru reports
=======================

https://blueprints.launchpad.net/neutron/+spec/guru-meditation-report

This spec proposes adoption for Oslo Guru reports in Neutron. The new feature
will enhance debugging capabilities of all official Neutron services, by
providing an easy way to gather runtime data about current threads and
configuration, among other things, to developers, operators, and tech support.


Problem Description
===================

Currently Neutron does not have a way to gather runtime data from active
service processes. The only information that is available to deployers,
developers, and tech support to analyze is what was actually logged by the
service in question. Additional data could be usefully used to debug and solve
problems that occur during Neutron operation. Among other things, we could be
interested in stack traces of all (both green and real) threads, pid/ppid info,
package version, configuration as seen by the service, etc.

Oslo Guru reports provide an easy way to add support for gathering this kind of
information to any service. Report generation is triggered by sending a special
(USR1) signal to a service. Reports are generated on stderr, that can be piped
into system log, if needed.

Guru reports are extensible, meaning that we will be able to add more
information to those reports in case we see it needed. For example, we could
add a new report section for L2 Open vSwitch agent to dump flows and other
relevant networking data to the report. Note that this spec does not cover any
of possible extensions and just lays the foundation for them.

Guru reports are supported by Nova.


Proposed Change
===============

First, a new oslo-incubator module (reports.*) should be synchronized into
neutron tree. Then, each service entry point should be extended to register
reporting functionality before proceeding to its real main(). This may be
achieved through utilizing neutron/cmd/__init__.py or a similar way that would
automatically expose the functionality to all services.

After neutron core repository adopts the feature, advanced services'
repositories will be similarly updated to utilize the module.


Data Model Impact
-----------------
None.


REST API Impact
---------------
None.


Security Impact
---------------
In theory, the change could expose service internals to someone who is able to
send the needed signal to a service. That said, we can probably assume that the
user is already authorized to achieve a lot more than just having an access to
stack traces and configuration used. Also, if deployers are afraid of the
information leak for some reason, they could also make sure their stderr output
is channeled into safe place.


Notifications Impact
--------------------
None.


Other End User Impact
---------------------
None.


Performance Impact
------------------
The feature does not require any additional resources until it's triggered by
the user. Default report generation is not expected to take too long. Report
extensions will need to be assessed on case by case basis. In any case, reports
are not expected to be generated too often and are assumed to be debugging
capability, not something to trigger once per minute just in case.


IPv6 Impact
-----------
None.


Other Deployer Impact
---------------------
Deployers may be interested in making sure those reports are collected
somewhere (e.g. stderr should be captured by syslog).


Developer Impact
----------------
None.


Community Impact
----------------
None.


Alternatives
------------
We could reimplement the wheel, but we hopefully won't.


Implementation
==============

Assignee(s)
-----------
ihrachyshka


Work Items
----------
* sync reports.* module from oslo-incubator
* adopt it in all neutron services using a wrapper located under
  neutron/cmd/...

Note: the feature was once proposed for inclusion but was later abandoned:
https://review.openstack.org/#/q/project:openstack/neutron+topic:bp/guru-meditation-report,n,z


Dependencies
============
reports.* module is currently going thru graduation consideration. In case it's
graduated into oslo.reports library before neutron switches to it, we won't
actually need to sync any code from oslo-incubator but instead add a new
external oslo dependency. If Neutron switches to the module before graduation
is complete, then we'll need to adopt oslo.reports later as part of usual oslo
liaison effort.


Testing
=======
Ideally, the feature would be tested using full stack testing framework that
was expected to enter the tree in Kilo but was postponed to Liberty. Note that
the framework is not yet reproposed for Liberty.


Tempest Tests
-------------
None.


Functional Tests
----------------
None.


API Tests
---------
None.


Documentation Impact
====================

User Documentation
------------------
Documentation should be extended to describe the new feature.


Developer Documentation
-----------------------
Developer documentation should be updated to include information on how to add
support for the reporting feature. Special attention should be made on those
who implement Neutron extensions out of tree.


References
==========
* oslo-incubator module: http://git.openstack.org/cgit/openstack/oslo-incubator/tree/openstack/common/report
* blog about nova guru reports: https://www.berrange.com/posts/2015/02/19/nova-and-its-use-of-olso-incubator-guru-meditation-reports/
* oslo.reports repo: https://github.com/directxman12/oslo.reports
* full stack testing: https://blueprints.launchpad.net/neutron/+spec/integration-tests


