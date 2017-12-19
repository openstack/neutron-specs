..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Porting the Neutron Core to Python 3
====================================

https://blueprints.launchpad.net/neutron/+spec/neutron-python3


Problem Description
===================
Neutron currently only works with Python 2.x. As some OpenStack projects now
work with Python 3 (most of the Python clients, some of the oslo libraries...),
and since OpenStack should move to Python 3 [1]_, Neutron shall be ported to
Python 3.4.


Proposed Change
===============
This specification details the steps needed to port Neutron core to Python 3.

The goal here is only to be able to run all the unit tests with Python 3.4.
This may not be enough to make sure that Neutron is fully compatible with
Python 3.4: we will also need to run Neutron in a devstack that uses Python
3.x, but this is another issue.

Obviously, these changes must not break any of Neutron's reverse dependencies.


Data Model Impact
-----------------
None


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
None


Performance Impact
------------------
None


IPv6 Impact
-----------
None


Other Deployer Impact
---------------------
If deployers want to run services under Python 3, they should install Python 3
with the whole stack. There will not be any differences for those who want
to stick to Python 2, as we will make sure not to break anything.


Developer Impact
----------------
Once Neutron has been ported to Python 3, developers will not able to write
code that only works with Python 2, since the py34 gate will be voting.


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
Primary assignee:
  cyril-roelandt


Work Items
----------

* Fix issues with the dependencies that are not Python 3 compatible: see the
  "Dependencies" section.

* Make sure "tox -epy34" only runs tests that are specifically listed in
  tox.ini.

* Enable the py34 gate in Neutron and make it voting. We should now be able to
  avoid regressions while working.

* Fix the most obvious issues (print statements, failing imports, ...) that can
  probably be caught by 2to3. Checks for these might be added to
  tools/misc-sanity-checks.sh to avoid regressions. Once the py34 is voting and
  all the unit tests pass on Python 3, we will be able to remove them.

* Fix all the tests, one by one.

* Once they all work, remove the explicit list of tests to run when using
  Python 3 from tox.ini and add py34 to the default env list.


Dependencies
============

Neutron has dependencies on packages that currently do not work with Python 3:

* oslo.db: Works with Python 3.4, even though we need a new release. The only
  issue is MySQL-Python which is not Python 3 compatible yet, but one can
  use mysqlclient as a drop-in replacement (MySQL-Python + py3 support +
  fixes).
* olso.messaging: Thanks to the work of Victor Stinner, the RabbitMQ and ZMQ
  drivers and all the executors work on Python 3. Only the Qpid and AMQP 1.0
  drivers don't work on Python 3, but it should not block porting Neutron to
  Python 3. Currently, this work can only be found in the git repository, but a
  Python3-compliant version should be released in a near future.

All in all, there is probably no major issue with the dependencies.

Testing
=======

Tempest Tests
-------------
None


Functional Tests
----------------
None


API Tests
---------
None


Documentation Impact
====================

User Documentation
------------------
None


Developer Documentation
-----------------------
Developers might be interested in reading the official Python 3 page on the
Openstack wiki [2]_. It shows the current progress and details some common
issues that arise when porting code to Python 3.


References
==========
.. [1] http://techs.enovance.com/6521/openstack_python3
.. [2] https://wiki.openstack.org/wiki/Python3
