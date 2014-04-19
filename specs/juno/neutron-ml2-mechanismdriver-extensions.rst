..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Support for Extensions in ML2
=============================

https://blueprints.launchpad.net/neutron/+spec/extensions-in-ml2

This blueprint defines a pluggable framework for extending core resource
attributes in ML2.

Problem description
===================

In the current ML2 plugin implementation, only the extensions defined in the
Ml2Plugin class itself are available, and there is no way for core resources to
be extended with attributes needed by specific mechanism drivers. When such
extensions are needed, whether for new general purpose features that might be
incorporated in future core API versions, or to better support specific
networking technologies, the only current option is to implement a new
monolithic plugin.

Proposed change
===============

The following changes can be considered as a solution to support extensions in
mechanism driver:

1. Introduce ExtensionDriver API and ExtensionManager (similar to
MechanismDriver and MechanismManager).

2. Define new configuration parameter which contains list of extension drivers
to load (i.e. extension_drivers).

3. ExtensionDriver - Abstract class that defines the following interfaces for
ML2 extension drivers:

- initialize: Abstract method - Perform extension driver initialization
- extension_alias: Abstract property - Return supported extension aliases
- process_create_network: Process extended attribute for create network
- process_create_subnet: Process extended attribute for create subnet
- process_create_port: Process extended attribute for create port
- process_update_network: Process extended attribute for update network
- process_update_subnet: Process extended attribute for update subnet
- process_update_port: Process extended attribute for update port
- extend_network_dict: Add extended attributes to network dictionary
- extend_subnet_dict: Add extended attributes to subnet dictionary
- extend_port_dict: Add extended attributes to port dictionary

4. ExtensionManager - Manages loading and initializing extension drivers
similarly to the existing TypeManager and MechanismManager classes.
Provides methods that dispatch to the ordered list of registered extension
drivers for each of the process_create_<resource>, process_update_<resource>,
and extend_<resource>_dict abstract operations defined on ExtensionDriver.

5. Ml2Plugin will initialize the ExtensionManager at startup, which will load
and initialize the configured ExtensionDrivers.

6. Change Ml2Plugin's supported_extension_aliases abstract property to include
the extension_alias property of each registered ExtensionDriver.

7. In resource's create and update (i.e. network, port, subnet in Ml2Plugin)
before calling the pre/post_commit, process_create/update in extension
driver should be called to validate and persist any extended resource's
attributes defined by driver. Extended attribute must also be returned.

8. In each case where dictionaries are built by Ml2Plugin for network, subnet,
and port resources, the ExtensionManager's extend_<resource>_dict function will
be called so that the registered ExtensionDrivers can add their extended
attributes.

Alternatives
------------

Link below contains discussion on this subject in icehouse summit:

https://etherpad.openstack.org/p/icehouse-neutron-vendor-extension

Also, alternative of implementing extensions directly in mechanism drivers was
considered, but was rejected for various reasons, including no way to return
extended attributes from get operations

Data model impact
-----------------

ExtensionDriver implementations will add tables to store their extended
attributes, but no ExtensionDrivers are included in the BP.

REST API impact
---------------

Core resource extensions will now be possible with ML2, similar to what has
previously been possible with monolithic plugins.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

If a mechanism driver needs to add extended attribute to a resource, it needs
to create an extension driver (based on ExtensionDriver API) and add the name
to the setup.cfg and ml2_conf.ini.
The name of extension and path to the extension should be added in the
extension driver.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Nader Lahouti (nlahouti)

Other contributors:
  None

Work Items
----------

- ExtensionManager - new implementation
- ExtensionDriver - new implementation
- Add new method in Ml2Plugin class for adding supported extensions in
  mechanism driver to supported extension in the class.
- Invoke extension driver's method (e.g. process_create resource) in
  create/update/delete_<resource's name> to add/update/delete attribute in
  persistent table.

Dependencies
============

None

Testing
=======

The regular test for plugin still applies here. But new unit tests will be
added with a test ExtensionDriver that verifies that extended attributes are
properly processed in create and update operations, returned from create,
update, and get operations, and available from within MechanismDriver methods.

Documentation Impact
====================

The ExtensionDriver API will include docstrings describing the new API, so
generated documentation will cover it. Will need to update deployment docs for
the new config variable. Will mainly be covered by vendor docs whose mechanism
drivers require extension drivers.

References
==========

None
