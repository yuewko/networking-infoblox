============
Installation
============
The Infoblox driver should be installed on the controller nodes that are
runing your ``neutron-server``. The installation consists of the following
basic steps:

1) Configure Infoblox
2) Install the driver module on the controller nodes
3) Run database migrations to create the Infoblox tables
4) Modify neutron.conf and nova.conf
5) Start the Infoblox IPAM Agent
6) Restart the neutron-server
7) Restart nova-compute services


Configure Infoblox Grid
=======================
There are four steps to setting up your Infoblox grid to work with the IPAM
driver.

    a) Add an Infoblox user and optionally a group, and configure permissions
    b) Create Extensible Attribute definitions and associate Network Views with OpenStack using
       the create_ea_defs.py script
    c) Set the EAs to values representing the desired behavior
    d) Tag members that should serve OpenStack domains

Creating the User
-----------------
First, you should create an Infoblox user for the integration. If you have a Cloud
Network Automation license and/or are using Cloud Platform Appliances, this user should be assigned to the Cloud API Only
admin group. Otherwise, you may want to create a group specifically for this
integration. The group must be given the following permissions for full
IPAM/DHCP/DNS functionality to work:

.. list-table::
   :header-rows: 1
   :widths: 20 20 20 10

   * - Permission Type
     - Resource
     - Resource Type
     - Permission
   * - [DNS]
     - All A Records
     - A record
     - RW
   * - [DNS]
     - All AAAA Records
     - AAAA record
     - RW
   * - [DNS, DHCP, IPAM]
     - All Hosts
     - Host
     - RW
   * - [DHCP, DNS, IPAM]
     - All IPv4 Host Addresses
     - IPv4 Host address
     - RW
   * - [DHCP, DNS, IPAM]
     - All IPv6 Host Addresses
     - IPv6 Host address
     - RW
   * - [DHCP, IPAM]
     - All IPv6 Networks
     - IPv6 Network
     - RW
   * - [GRID]
     - All Members
     - Member
     - RW
   * - [DHCP, IPAM]
     - All IPv4 Networks
     - IPv4 Network
     - RW
   * - [DHCP, IPAM]
     - All Network Views
     - Network view
     - RW
   * - [DNS]
     - All PTR Records
     - PTR record
     - RW
   * - [DHCP]
     - All IPv4 Ranges
     - IPv4 range
     - RW
   * - [CLOUD]
     - All Tenants
     - Tenant
     - RW
   * - [DNS]
     - All DNS Views
     - DNS View
     - RW
   * - [DNS]
     - All Zones
     - Zone
     - RW

If you are testing IPAM only case which does not require Infoblox to serve DHCP and DNS, here is
the minimum set of required permissions.

.. list-table::
   :header-rows: 1
   :widths: 20 20 20 10 40

   * - Permission Type
     - Resource
     - Resource Type
     - Permission
     - Comment
   * - [GRID]
     - All Members
     - Member
     - RW
     - This can be set RO if `Report Grid Sync Time` is set to False.
   * - [CLOUD]
     - All Tenants
     - Tenant
     - RW
     -
   * - [DHCP, IPAM]
     - All Network Views
     - Network view
     - RW
     -
   * - [DHCP, IPAM]
     - All IPv4 Networks
     - IPv4 Network
     - RW
     -
   * - [DHCP, IPAM]
     - All IPv6 Networks
     - IPv6 Network
     - RW
     -

Create Extensible Attribute Definitions and Network View Associations
----------------------------------------------------------------------
The driver uses a variety of Extensible Attributes (EAs) to manage its
configuration. The needed extensible attributes may be created automatically
using the ``create_ea_defs.py`` script that can be found under the ``tools``
directory in the distribution::

    $ python create_ea_defs.py

The script will prompt you for the user name and password of an Infoblox superuser, which
is needed to create the EA definitions.

The script will also prompt you for association or un-association of
network views to OpenStack. This is an important step. You can use this script to select
network views explicitly to use in OpenStack. An associated network view will
have `Cloud Adapter ID` EAs stored on that network view. The `Cloud Adapter ID`
is equivalent to `cloud_data_center_id` defined in neutron.conf.

Setting EA values to Configure the Integration
----------------------------------------------
You must decide on the configuration you would like to use. For details on the
configuration options, please refer to the Infoblox Configuration Guide
(configuration_guide.rst).

The configuration is captured within the various EAs that were created in the
previous step. In general, these EAs are set on the *grid master* member. To do
this, you navigate to Grid > Grid Manager > Members and click on the Gear icon
next to the grid master member. Choose the *Extensible Attributes* option. From
there you can create and modify various EA values that will apply to the entire
IPAM driver integration.

Installing the Driver
=====================
The driver need to be installed on each controller node that is running the
Neutron service. The driver is available from PyPi, and can be installed using
the ``pip install`` command.

Infoblox IPAM Agent Installation
--------------------------------
The ``infoblox-ipam-agent`` init script that is used to start the Infoblox IPAM
Agent will be installed as part of the ``pip install`` command.
By default, the ``infoblox-ipam-agent`` init script is installed as
``/usr/local/etc/init.d/infoblox-ipam-agent``. To install the script in ``/etc/init.d``,
specify ``--install-option`` as follow::

    $ sudo pip install --install-option="--install-data=/" networking-infoblox


Latest Release
--------------
To install the most recent production release, use the following command::

    $ sudo pip install networking-infoblox

Liberty and Mitaka
------------------
Version 2.x of the driver supports Liberty and Mitaka. Version 2.0.1 of the
driver can be installed using the following command::

    $ sudo pip install networking-infoblox==2.0.1

We strongly recommend to use 2.0.1 instead 2.0.0 because 2.0.1 includes
critical bug fixes that ensure its stability and has undergone important
database scheme change to support production usability like GM candidate
promotion, proxying support and better grid sync.

We do not support data migration from 2.0.0 to 2.0.1. Please use Version 2.0.1
or higher.

Creating the Infoblox Neutron Database
======================================
The driver uses a number of different Infoblox-specific tables to manage the
integration. These are created by running the `neutron-db-manage` after you
install the `networking_infoblox` module::

    $ sudo neutron-db-manage upgrade head

This should be done on one of the controller nodes, assuming all controller
nodes share a common database cluster.

Modify the OpenStack Configuration
==================================
The ``neutron.conf`` files on each controller node, as well as the
``nova.conf`` files on each compute node, must be updated as described below.

Neutron
-------
The grid connectivity and credentials configuration must be added to the
``neutron.conf`` file in `infoblox` and `infoblox-dc` stanzas. The `infoblox`
stanza contains a list of grids, and then each there is an `infoblox-dc`
containing the appropriate configuration for each grid. Support for multiple
grids is not yet available.

.. list-table::
   :header-rows: 1
   :widths: 10 90

   * - Option
     - Description
   * - cloud_data_center_id
     - An integer ID used for the data center. This is used to form the stanza
       name for the rest of the options. If you have multiple instances of
       OpenStack sharing the same Infoblox grid, this ID needs to be unique
       across the instances. We recommend the ID starting from 1 and increment
       by 1 as you add another Openstack instance. This ID is used to generate
       a unique ID for a network view that is cached in neutron database.
       Starting it with a very high number may exceed the max length of a
       network view id.
   * - grid_master_host
     - The IP address, hostname, or FQDN of the Grid Master (GM).
       Proxying is supported so this does not have to be the exact IP or
       hostname of the GM if you have a situation where you cannot reach the GM
       directly in your network. It can be any connection information that
       proxies to the GM.
   * - grid_master_name
     - The name of the Grid Master (GM)
       This has to be the exact GM name registered in the Infoblox grid.
   * - admin_user_name
     - The user name to use for the WAPI.
   * - admin_password
     - The password to use for the WAPI.
   * - wapi_version
     - The WAPI version to use. Version 2.2.2 or later is recommended, if your
       grid supports it (WAPI version 2.3 is supported in NIOS 7.3)
   * - wapi_max_results
     - The maximum number of objects to be returned by WAPI. If this is set to
       a negative number, WAPI will return an error when the number of returned
       objects would exceed the setting. If this is set to a positive number,
       the results will be truncated when necessary. The default is -1000.
       If you experience "Result set too large" error, increase this value.
   * - ssl_verify
     - Set to false if you use a self-signed SSL certificate, and true
       if you use a certificate signed by a known certificate authority. You
       can also set this to a path to a certificate file so that verification
       will be done even for a self-signed certificate. Using a value of False
       in a production environment is not secure.
   * - http_pool_connections, http_pool_maxsize, http_request_timeout
     - Optional parameters to control the HTTP session pool.

Additionally, the `ipam_driver` option must be set in ``neutron.conf`` to
`infoblox`.

These settings must be done on *each controller* that runs the Neutron service.

Example (replace the ALL_CAPS values with those appropriate for your
installation):

.. code-block:: ini

   [DEFAULT]
   ipam_driver = infoblox

   [infoblox]
   cloud_data_center_id = 1

   [infoblox-dc:1]
   grid_master_host = GRID_MASTER_HOST
   grid_master_name = GRID_MASTER_NAME
   admin_user_name = USER
   admin_password = PASSWORD
   wapi_version = 2.2.2
   wapi_max_results = -50000

In addition to these options, you must enable the notifications options
within Neutron, if they are not already enabled.

.. code-block:: ini

   [DEFAULT]
   notification_driver = messagingv2
   notification_topics = notifications

Nova
----
On each controller node running the Nova service, as well as compute node
running nova-compute, you must configure Nova to send notifications.
These notifications are used by the Infoblox IPAM agent to manage DNS entries
and extensible attribute values for VMs. Set the following values in
``nova.conf``, if they are not already set.

.. code-block:: ini

   notification_driver = messagingv2
   notification_topics = notifications
   notify_on_state_change = vm_state

Start the Infoblox IPAM Agent
=============================
Depending on your distribution, you will need to create and configure
init.d and/or systemd service definitions for the ``infoblox-ipam-agent``.
Once that is done, you should start the agent.

To start it manually, without any init.d or systemd setup, you run the
following command as the same user that runs neutron-server::

    $ /usr/local/bin/infoblox-ipam-agent --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini >/var/log/neutron/infoblox-ipam-agent.log 2>&1

Restart the Services
====================
The appropriate services must be restarted to pick up the changes to the
configuration files.

Neutron
-------
Restart ``neutron-server`` on each node running it. The exact command may vary
based upon your distribution. In Ubuntu the command is::

    $ sudo service neutron-server restart

Nova
----
If you modified the Nova notification settings, you must restart the Nova
Compute service on each node running it. The exact command may vary based
on your distribution. In Ubuntu the command is::

    $ sudo service nova-compute restart

Running Data Migration
======================

Before installing networking-infoblox, you may have already created networks,
subnets and ports in OpenStack. If you wish to migrate those objects to the
Infoblox grid, you can run `sync_neutron_to_infoblox.py` script under
networking_infoblox\tools folder.

In order to run the script, you will need to create a keystone_admin file if
you don't have one already and source it so that you have the admin credential
variables available in the shell environment.

networking-infoblox should have been successfully configured before running the
migration script.

.. code-block:: console

    $ cat keystone_admin
    unset OS_SERVICE_TOKEN
    export OS_USERNAME=admin
    export OS_PASSWORD=admin
    export OS_AUTH_URL=http://10.39.12.161:5000/v2.0
    export PS1='[\u@\h \W(keystone_admin)]\$ '

    export OS_TENANT_NAME=admin
    export OS_REGION_NAME=RegionOne

    $ source keystone_admin

    # if you have not run infoblox-ipam-agent yet, then you need to run
    # infoblox_grid_sync.py to register the Infoblox grid members to Neutron.
    $ networking-infoblox(keystone_admin)]# python networking_infoblox/tools/infoblox_grid_sync.py

    $ networking-infoblox(keystone_admin)]# python networking_infoblox/tools/sync_neutron_to_infoblox.py

You can re-run the migration script as many times as needed.

Known Issues and Limitation
===========================

Issue #1
--------

We have discovered an issue with `A` DNS record during the floating IP
association. After a floating IP is associated, infoblox-ipam-agent updates
the record name from 'floating-ip-' prefixed name to 'host-ip-' prefixed name
to indicate that the floating ip is now associated with the instance.

After the name change happens, sometimes we see that all the EAs are cleared.

This happens when WAPI version 2.3 is used against NIOS 7.3.

The following grid configurations are needed to reproduce the issue:

 * `IP Allocation Strategy`: Fixed Address
 * `DNS Record Binding Types`: record:a, record:aaaa
