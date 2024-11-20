..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================================
Provide Port Binding Information for Nova Live Migration
========================================================

https://bugs.launchpad.net/neutron/+bug/1580880

Nova Live Migration consists of 3 stages:

* pre_live_migration - executed before migration starts; the migration target
  is already known, but the instance still resides on the source.

* live_migration_operation - migration itself which consists of 2 substages:

  * before the VM is running on the migration target compute.

  * after VM is running on the destination compute, but migration is still in
    progress (post copy migration).

* post_live_migration - executed after migration; source VM does not exist
  anymore. Used for finalizing the migration.

Today, port binding occurs on the target compute host during migration,
considered now in the post_live_migration stage.  This, unfortunately, is
too late in the process as there are cases where Nova requires this
information in the pre_live_migration stage.  We simply cannot move the port
binding to the pre_live_migration stage as the original port binding would be
deleted, causing issues due to the instance being active still on the original
host.  Further details are contained in the 'Alternatives' section.

Problem Description
===================

For live migration improvements, it is required that Neutron allows port
binding on the target compute host during the pre_live_migration stage, without
removing the original port binding on the source compute.

The proposal is to have a port binding on the source AND target compute hosts
where only one port binding is active.  After instance migration is complete,
the target port binding is activated and the source port binding is
deactivated, but not deleted.  The original port binding on the source compute
host is kept for a possible move of the instance back to the source, and will
only be removed during the post_live_migration stage.

The issues can be divided in 2 categories:

#1 Keep the source instance running if the port binding on the target fails:

* Instance error state when port binding fails:

    Today, the port binding is triggered in the post_live_migration stage,
    after the migration has completed.  If the port binding fails, the
    instance is then stuck in an error state.

    The solution is to move the port binding to the pre_live_migration stage,
    where there is an opportunity to fail the port binding earlier.  This would
    prevent the instance from being stuck in an errored state after the
    migration is complete and fail the migration at a pre_live_migration stage.

    The issue with moving the port binding to the pre_live_migration stage is
    that some drivers will shutdown the port binding on the source compute
    host, even though the instance is still active.  To achieve a cleaner
    migration solution, Neutron needs to be modified to allow a port binding
    on both the source and target compute hosts in an active/inactive state.

#2 Handling the differences in port binding details between hosts:

* Live migration between hosts running different l2 agents:

    Another case to consider is the chance that the migration occurs between
    two hosts running different l2 agents.  The requirement here is on Nova
    to update the instance definition before the migration is executed. In the
    case of libvirt, Nova would update the domain.xml with the target
    interface definition.  For more information, refer to `[3]`_ and `[4]`_.

    A special case to consider is where a migration occurs between agents
    with differing firewall drivers, i.e. from a host running the ovs
    hybrid-fw driver to a host running the new ovs conntrackd firewall driver.

    Such a migration must only be allowed with source and target binding using
    the same VNIC type.

* Live migration with MacVTap agent when different physnet mappings are used:

    MacVTap today has restrictions with live migration in certain situations
    `[1]`_ and requires an update to the instance definition (libvirt
    domain.xml) before starting a migration.

    Updating the instance definition occurs during the live_migration_operation
    stage, before the instance is running on the target compute host.  Two port
    bindings, one active at the source and one inactive at the target
    host are required for this operation to succeed.

Proposed Change
===============

For migration, Nova will require the port binding information from the target
compute during the pre_live_migration stage.  This can be achieved by allowing
a compute port to be bound to the migration source and the migration target
host.

This can be achieved by the following steps:

#1 Expand upon existing API entities under port, allowing CRUD bindings.

#2 Update ML2 to support the changes.

#3 Update the DB to support the changes.

Usage by Nova
-------------

Nova will utilize the expanded API, and the potential flow is as follows:

* pre_live_migration: Create the inactive binding for target host.

* live_migration_operation: Use information gathered from the inactive
  binding to modify the instance definition.

* live_migration_operation: Once the instance is active, set the inactive
  binding to active, and the previous active binding to inactive.

* post_live_migration: Remove the inactive binding on the source
  compute host.

.. image:: /images/pike/nova-live-migration-flow.png

* If rollback is performed after the instance is active on target: From a
  Neutron standpoint, if the binding is active on the target host, Nova will
  need to set the source binding back to active::

    PUT /v2.0/ports/{port_id}/bindings/{host_id}/activate

For more details on the Nova implementation , see the related Nova
Blueprint `[3]`_ and its Spec `[4]`_.  Neutron will not dictate the implemented
capabilities of Nova live migration and will support either path, to rollback
or to not rollback.

Binding API Extension for Ports
-------------------------------

.. _list_binding:

List Bindings
~~~~~~~~~~~~~

GET /v2.0/ports/{port_id}/bindings

::

    {
        "bindings": [
            {
                "host_id": "source-host_id",
                "vif_type": "ovs",
                "vif_details": {
                    "port_filter": true,
                    "ovs_hybrid_plug": true
                    },
                "profile": {},
                "vnic_type": "normal",
                "status": "active"
            },
            {
                "host_id": "target-host_id",
                "vif_type": "bridge",
                "vif_details": {
                    "port_filter": true,
                    },
                "profile": {},
                "vnic_type": "normal",
                "status": "inactive"
            },
        ]
    }

..  list-table:: Response Parameters
    :header-rows: 1

    * - Parameter
      - Style
      - Type
      - Description
    * - bindings
      - plain
      - xsd:list
      - A list of *binding* objects

More parameters see :ref:`show_binding`

Important key features of list bindings:

* Compute bindings will currently be listed and a request for unsupported
  bindings will return 'NotImplemented' until the capability is introduced.

* All bindings will be listed and pagination will be used when many bindings
  are returned.

.. _show_binding:

Show Binding
~~~~~~~~~~~~

GET /v2.0/ports/{port_id}/bindings/{host_id}

..  list-table:: Response Parameters
    :header-rows: 1

    * - Parameter
      - Style
      - Type
      - Description
    * - binding
      - plain
      - xsd:dict
      - A *binding* object
    * - host_id
      - plain
      - xsd:string
      - Hostname where the port is allocated.
    * - vif_type
      - plain
      - xsd:string
      - The VIF type for this port binding determined during
        portbinding
    * - vif_details
      - plain
      - xsd:dict
      - A dictionary containing additional details for this specific
        binding. The details are set by a mechanism driver.
    * - vnic_type
      - plain
      - xsd:string
      - The VNIC type for this port binding.
    * - profile
      - plain
      - xsd:dict
      - A dictionary holding the vif profile.
    * - status
      - plain
      - xsd:String
      - Status of the binding :ref:`binding_status`

::

    {
        "binding": {
             "host_id": "target-host_id",
             "vif_type": "target-vif-type",
             "vif_details": {
                 "port_filter": true,
                 },
             "vnic_type": 'NORMAL',
             "profile": {},
             "status": "active"
         }
    }

Important key features of show binding:

* DVR ports exposed in this resource will show the real vif_type of
  'distributed' ports as they are stored in DistributedPortBindings, i.e. ovs.

.. _create_binding:

Create Binding
~~~~~~~~~~~~~~

POST /v2.0/ports/{port_id}/bindings

..  list-table:: Request Parameters
    :header-rows: 1

    * - Parameter
      - Style
      - Type
      - Description
    * - binding
      - plain
      - xsd:dict
      - A *binding* object
    * - host_id (mandatory)
      - plain
      - xsd:string
      - Hostname where the port is allocated.
    * - vnic_type (optional, default = 'normal')
      - plain
      - xsd:string
      - The VNIC type for this port binding.
    * - profile (optional)
      - plain
      - xsd:dict
      - A dictionary holding the vif profile.

::

    {
        "binding": {
            "host_id": "target-host_id"
        }
    }

Response parameters

see :ref:`list_binding`

::

    {
        "binding": {
            "host_id": "target-host_id",
            "vif_type": "ovs",
            "vif_details": {
                "port_filter": true,
                "ovs_hybrid_plug": true
                },
             "vnic_type": 'NORMAL',
             "profile": {},
             "status": "active"
        }
    }

If the binding fails, a new return code of 4xx or 5xx should be returned. This
differs from today where a failed binding returns a 2xx response code and the
vif_type is set to "binding_failed".

Important key features of update/create binding:

* By default, the status will be active when creating a port binding.  If
  a binding is created, doesn't exist already, and an existing binding is
  already active, the binding will default to inactive, requiring the
  operator to activate the new binding.

* If a binding being added already exists, a 4xx will be returned.

* A compute port can only have 1 active binding at a time. This is not an
  enforcement by Neutron, but a result of the operation surrounding
  PortBinding. This feature expands the capability of having multiple bindings,
  but will only allow for 1 active binding for compute ports.

* At this time, creation of a binding will be limited to compute ports.

* The existing API is not touched, which will return host_id:{host_id} as the
  current active binding.

* Activating an inactive compute binding will deactivate the current
  active binding.

Update Binding
~~~~~~~~~~~~~~

PUT /v2.0/ports/{port_id}/bindings/{host_id}

..  list-table:: Request Parameters
    :header-rows: 1

    * - Parameter
      - Style
      - Type
      - Description
    * - binding
      - plain
      - xsd:dict
      - A *binding* object

All create parameters are valid for update as well. See :ref:`create_binding`.
::

    {
        "binding": {
            "vnic_type": 'NORMAL',
            "profile": {}
        }
    }

Response parameters

see :ref:`show_binding`

::

    {
        "binding": {
            "host_id": "target-host_id",
            "vif_type": "ovs",
            "vif_details": {
                "port_filter": true,
                "ovs_hybrid_plug": true
                },
            "vnic_type": 'NORMAL',
            "profile": {"foo": "bar"},
            "status": "active"
        }
    }

On failed binding, a 4xx or 5xx return code should be returned.

.. _activate_binding:

Activating an Inactive Binding
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

PUT /v2.0/ports/{port_id}/bindings/{host_id}/activate

Response parameters

see :ref:`show_binding`

::

    {
        "binding": {
            "host_id": "target-host_id",
            "vif_type": "ovs",
            "vif_details": {
                "port_filter": true,
                "ovs_hybrid_plug": true
                },
            "vnic_type": 'NORMAL',
            "profile": {"foo":"bar"},
            "status": "active"
        }
    }

Important key features of activate binding:

* Activating a compute binding that is inactive will deactivate the existing
  active binding, as a compute port can only have 1 binding active at a time.

* Operation will be limited to compute ports.

* Attempting to activate an existing active binding will return a 4xx.

* Returns 5xx if activating the binding fails.

Delete Binding
~~~~~~~~~~~~~~

DELETE /v2.0/ports/{port_id}/bindings/{host_id}

This operation does not accept a request body and does not return a response
body.

Important key features of delete binding:

* Active/Inactive bindings can be removed.

* Deleting an active compute port binding, where an inactive binding
  exists does not activate the binding.  The operator will be required to
  explicitly activate the binding.

Overlap Between Existing vs New APIs
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

All the functionality of the existing API will be covered by the new API as
well. This section describes the overlap.

.. list-table:: Overlap existing vs. new API

  * - Existing API
    - New API
  * - Show port with active binding
    - n/a
  * - Create port: directly with an active binding
    - n/a
  * - Update port: add host_id (which adds the active binding)
    - Add Binding
  * - Create port: without any binding
    - n/a
  * - Update port: change host_id (re-trigger port binding for another host)
    - Update binding
  * - Update port: set host_id to ''(remove the active binding)
    - Delete active binding
  * - Delete port: Remove port with all its bindings
    - n/a

Effects on Existing APIs
~~~~~~~~~~~~~~~~~~~~~~~~

Slight adjustment to existing APIs:

* Show Port will still just show the binding like today. For compute
  ports, it would only show the active binding.

* Create Port will create an unbound binding as before, but with the status of
  active.

* Update Port with host_id will still re-trigger port binding for a host.  The
  difference will be `update_port()` will only action on the active binding.

* vif_type is set to "binding_failed" and http code 2xx is still used
  on failed port binding when binding is triggered via the existing
  port binding extension.

API Visibility
~~~~~~~~~~~~~~

A normal user should not be able to trigger any create/update/delete/activate
actions. This should only be possible via some special service user or the
admin role.

Sub Resource Extension
----------------------

Neutron `ports` will be extended with a sub resource `bindings`, having a
member name of `port` to preserve portbindings and ports extensions.  The new
sub resource extension will be `portbindings_extended.py` and have a parent
resource of `ports`.

The following methods will be added to the newly created service plugin
`bindings_plugin.py`:

* `get_port_bindings()`

* `get_port_binding()`

* `create_port_binding()`

* `update_port_binding()`

* `delete_port_binding()`

* `update_port_binding_activate()`

ML2 Changes
-----------

Existing methods `create_port()` and `update_port()` will need to be updated
to support actioning only on `active` status bindings.  In addition, a new
status of `inactive` will be introduced to neutron-lib for use in PortBinding.

.. _binding_status:

Status Usage
~~~~~~~~~~~~

The status column in PortBinding will store an additional state 'inactive'
where the current states are 'active' and 'down'.  Neutron-lib will
only require the addition of PORT_BINDING_STATUS_ACTIVE and
PORT_BINDING_STATUS_INACTIVE.

::

    from neutron_lib import constants as const

    const.PORT_BINDING_STATUS_ACTIVE
    const.PORT_BINDING_STATUS_INACTIVE

Create/Update/Delete Port
~~~~~~~~~~~~~~~~~~~~~~~~~

New methods will be introduced in support of the new sub resource under ports,
but the current `create_port()`, `update_port()` and `delete_port()` will be
modified to only act on `active` bindings in the PortBinding table.

Today, `create_port()` adds an empty unbound binding in PortBinding and the
following changes will be made in support of this spec:

* Create an unbound binding with status `active` in the PortBinding table.

In addition, `update_port()` will be adjusted for `active` status with the
following changes:

* Update will only change binding information on the `active` binding in the
  PortBinding table.

Finally, `delete_port()` will be adjusted for `active` status with the
following changes:

* Delete will only act on the `active` binding in the PortBinding table.

Data Model Changes
------------------

The PortBinding table will expand the primary key to column `host`, allowing
selection of the binding based on `port_id` and `host`.

In addition, a `status` column will be introduced in the expansion where states
`active`, `down`, and `inactive` will be values.

Online upgrades, Blueprint `[9]`_, requires the addition of `host` to
primary_keys and a new field `status` for Port Binding OVO. Version of the
object should be bumped if push-notifications, under Blueprint `[10]`_,  will
be merged first, and PortBinding object will be present on the RPC wire.
Defining a default value for the `status` field would not require online
data migration.

Changes to Mechanism Drivers
----------------------------

A new mechanism driver method is required to determine if the new way
of binding things is supported. This must be validated for all binding
levels and allow fallback to previous methods.  By default, new method will
return unsupported.

Besides the in tree mechanism drivers for l2 agents (ovs, lb, sr-iov, macvtap)
the following drivers need to be considered:

* l2pop (however there are ideas to eliminate l2pop)

* ironic

* third party mechanism drivers

Activate RPC Port Update/Delete
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The existing `port_update` and `port_delete` RPC message will be adjusted to
send, when the agent retrieves device information with
`get_devices_details_list_and_failed_devices`, specific binding information
for the host regardless of binding state. This will allow the addition of
additional plumbing to occur as follows:

* Activate will result in a `port_update`, which will pass the relative binding
  information for the host and dictate the transition from inactive to active
  in the `get_devices_details_list_and_failed_devices` response. This will
  allow for a GARP to be sent out, updating the topology to a change in status.

* Activate will result in a `port_delete` rpc call to the source host, removing
  the source VIF.  This will need to be accomplished due to Nova not being able
  to issue a delete port, as the port still exists on a different host.  The
  binding will remain in an `inactive` state where binding information has
  been populated, but the port, from an agent perspective, will not exist.
  In addition, the transition from active to inactive will be indicated in the
  rpc call, influencing the `update_device_list` to not update the port state.

.. image:: /images/pike/rpc-portbinding-flow.png

In the case where push-notifications are implemented for ports under Blueprint
`[10]`_ the `get_devices_details_list_and_failed_devices` would not be adjusted
for transition state.  Instead, the binding transition state would be sent to
the agent as part of the port object.  The remaining actions are the same.

Other changes
-------------

* Neutron/Openstack Python Client support.

* Neutron-Lib support of new constant PORT_STATUS_INACTIVE
  see :ref:`activate_binding`.

Command Line Client Impact
--------------------------

Support for port bindings will be needed in OSC.  The following will be added::

    $ openstack port binding list {ARGS} <port>

    $ openstack port binding show {ARGS} <port> <host>

    $ openstack port binding create {ARGS} <port>

    $ openstack port binding update {ARGS} <port> <host>

    $ openstack port binding delete {ARGS} <port> <host>

    $ openstack port binding activate {ARGS} <port> <host>

Security Impact
---------------

None.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None.

Performance Impact
------------------

There should be no performance impact.

IPv6 Impact
-----------

None.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

Impact to Nova live migration, and is directly in support of their efforts.

Community Impact
----------------

Yes.  This change has been discussed on the ML, in Neutron meetings
(especially ML2), at mid-cycles, and at the design summit.

Alternatives
------------

An alternative is to use the current resources under ports to facilitate this
change to live migration.  The problem is, current dict structures would need
to be expanded to accommodate the 'bindings' key.  This may cause some
confusion as the user already receives 'bindings:profile' and various other
values.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
* `Jakub Libosvar <https://launchpad.net/~libosvar>`_

Please add your name here and attend the `ML2 Subteam Meeting
<https://wiki.openstack.org/wiki/Meetings/Neutron-ML2-Subteam>`_ if you'd
like to contribute.

Dependencies
============

None.

Testing
=======

Tempest Tests
-------------

Addition of a scenario test, `test_bindings.py`, to walk through the creation
of a source migration instance, creation of the inactive binding on a secondary
host, creation of a secondary target migration instance, activating the
inactive binding, deactivating the source migration active binding, and then
validating connectivity is still working.

Functional Tests
----------------

Additional functional tests will be added to `ml2/test_plugin.py` to expand
on the current port binding tests. This will accommodate for a status check
in the case of adding an inactive binding.

API Tests
---------

- Bindings resource (CRUD)

- Bindings resource (CRUD) and validation of active binding under current
  ports extension.

Documentation Impact
====================

Yes.

User Documentation
------------------

None.

Developer Documentation
-----------------------

- Create detailed explanation of active/inactive binding operation in
  `devref\ml2_port_bindings.rst`.  This should detail changes to ml2, and the
  extended `ports` resource.

References
==========

.. _[1]: https://bugs.launchpad.net/neutron/+bug/1550400
.. _[2]: https://bugs.launchpad.net/neutron/+bug/1367391
.. _[3]: https://blueprints.launchpad.net/nova/+spec/migration-use-target-vif
.. _[4]: https://review.openstack.org/#/c/301090/
.. _[5]: https://bugs.launchpad.net/neutron/+bug/1367391
.. _[6]: https://review.openstack.org/#/c/340031/
.. _[7]: https://bugs.launchpad.net/neutron/+bug/1595043
.. _[8]: https://review.openstack.org/340410
.. _[9]: https://blueprints.launchpad.net/neutron/+spec/adopt-oslo-versioned-objects-for-db
.. _[10]: https://review.openstack.org/#/c/225995/
