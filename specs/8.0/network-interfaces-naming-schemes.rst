..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Use the predictable network interfaces' names
=============================================

There is `the launchpad blueprint`_ about the spec.

Allow arbitrary names for network interfaces in Fuel. The practical reason is
to fix known issues for classical naming policy by using the predictable
interface naming policy.

-------------------
Problem description
-------------------

Currently Fuel relies on an old-fashioned (classical) *ethX* network interfaces
naming. Such a naming is known to be unstable in the following ways:

#) two machines with the same hardware can have different interface naming,
   consider two machines having an on board NIC and the one installed into
   PCI-E slot. Depending on luck the name of the interface corresponding to the
   on-board NIC can be eth0 on machine 1, and eth1 on machine 2.

#) the interfaces' names are not preserved across the reboots.

Switching to predictable interface naming policy doesn't require creating extra
udev rules pinning (classical ethX) interface name to the corresponding MAC
address to hold the same interface name after a reboot.

The 2nd problem can be solved by udev rule which pins the interface name to the
MAC address of the NIC. Solving the 1st problem requires a different naming
convention, such as the predictable network interfaces' names (as implemented
in udev >= 197). Modern Linux distributions (Ubuntu >= 15.04, CentOS >= 7) use
this scheme by default, and it can be enabled in Ubuntu 14.04 (and Debian
Jessie).

Hard-coded interface names ethX in Fuel code should be changed to support
arbitrary interface naming schemes.

By default, contemporary systems name Ethernet interfaces using different from
the conventional policy and can apply one of supported `naming schemes`_.

Example of classical interface naming, the interfaces are named as "eth0",
"eth1": ::

  $ ip -4 link
  2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP
     mode DEFAULT qlen 1000 link/ether 00:50:56:9c:74:4d brd ff:ff:ff:ff:ff:ff
  3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP
     mode DEFAULT qlen 1000 link/ether 00:50:56:9c:03:0a brd ff:ff:ff:ff:ff:ff

Example of interface naming based on physical location of the hardware (PCI
bus). The interfaces are named as "enp0s3" , "enp0s8": ::

  $ ip -o -4 link
  2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast
        state UP mode DEFAULT qlen 1000
        link/ether 08:00:27:4b:f0:40 brd ff:ff:ff:ff:ff:ff
  3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast
        state UP mode DEFAULT qlen 1000
        link/ether 08:00:27:bc:4d:85 brd ff:ff:ff:ff:ff:ff

Example of interface naming based on MAC addresses. The interfaces are named as
"enx0800274bf040" and "enx080027bc4d85"::

  $ ip -o -4 link
  2: enx0800274bf040: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
        qdisc pfifo_fast state UP mode DEFAULT qlen 1000
        link/ether 08:00:27:4b:f0:40 brd ff:ff:ff:ff:ff:ff
  3: enx080027bc4d85: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
        qdisc pfifo_fast state UP mode DEFAULT qlen 1000
        link/ether 08:00:27:bc:4d:85 brd ff:ff:ff:ff:ff:ff

Ethernet interfaces could also have names incorporating Firmware/BIOS provided
index numbers for on-board devices (example: eno1), names incorporating
Firmware/BIOS provided PCI Express hotplug slot index numbers (example: ens1).

----------------
Proposed changes
----------------

Fuel (master nodes, slaves, bootstraps, etc) shall support the predictable
interfaces' names and use it by default. Hard-coded interface names and regular
expressions bounded to a particular naming policy shall be avoided in the code.

Here are examples from the code (written in bash), which should be re-worked.
The "eth0", "eth1" and "eth." should be changed in the code below:

.. code-block:: bash

   # THIS CODE IS BROKEN
   ADMIN=eth0
   PUBLIC=eth1
   ALL_ETH_IP=$(ip -o -4 addr | grep "eth." | awk '{print \$4 }' | cut -d/ -f1)

There are default settings (expectations) in Fuel. For example, the interface
eth0 is expected to be an admin interface by default now.

Since Ethernet interfaces could have any names (we can't predict which one it
would be), and to preserve backward compatibility with the current classical
naming schema (ethX), the following approach is proposed for changes in code:

#. List of physical Ethernet interfaces should be collected from the system.
#. This list is sorted in alphabetic order.
#. The very first interface from the sorted list is assigned to the role of
   eth0, the second interface is assigned to the role of eth1 , etc ...

The function gathering Ethernet network interface names is required to
implement the feature. The function should collect information about only
physical Ethernet interfaces in the system (avoid virtual, e.g. loopback,
tunnel, VLAN, bound interfaces ... etc and wireless).

Web UI
======

The feature doesn't require changing the web UI, the possible impact is a
variable size for an interface names placeholder.

Nailgun
=======

The code related to the processing network interface names should be changed to
get rid using templates sticked to a particular interface naming schemes like
"eth*". The code of Nailgun doesn't require such changes, but Nailgun agent's
code should be improved. There is the `bug reported on nailgun-agent`_.

Interfaces' names are kept as metadata of clusters and nodes in the database.
Nailgun shall collect information from Nailgun agent on slaves about interface
names and detect the renaming and update (if it has not been implemented) the
names in the database using the MAC addresses to stick to.


Data model
----------

None.

REST API
--------

None.

Orchestration
=============

None.


RPC Protocol
------------

None.

Fuel Client
===========

None.

Plugins
=======

Plugins are ofter made by third-party teams. The code should be written in a
way avoiding stick to a particular interface naming schema.

Fuel Library
============

Fuel Library is abstract enough to be interface name agnostic. We use bridges
with names like 'br-mgmt' and this is matter of Nailgun to decide which port to
put into which bridge. The only problem here is to retain interface mapping
on-bootstrap/installation/post-installation phases which is a matter of
fuel-agent and Nailgun teams. Fuel Library is not affected by interface naming
unless there is some bug. We are using custom bridges names which attach
corresponding physical interfaces according to the info sent in network_scheme
which is currently generated by Nailgun. So far, there is no Library impact for
this feature.

------------
Alternatives
------------

We can stick to the classicall interface naming schema, but it does not allow
us to work with contemporary Linux distributions using another network
interface naming policies and prevent from including already deployed hosts
(like RHEL) to Fuel environments.

--------------
Upgrade impact
--------------

Upgrading from CentOS 6.x to CentOS 7.0 lead to change network interface
naming policy and can break a network configuration used before the upgrade.

The interfaces' names from slave nodes are kept as metadata in configs. The
additional step to match and update the new names in configs shall be
introduced during updgrade before the configs will be applied. The plausible
solution is to stick to hardware (MAC) addresses, when the naming of network
interfaces are changed. The (upgrade) scripts should collect information
about current network configuration and modify the configs in way preserving
current interfaces' roles.

It's possible to caught the case after an upgrade when the IBP images have
been built for classical interface naming schema, but bootstrap image uses
predictable interface naming policy. Since the software has a flaw, which
doesn't allow updated interfaces' names between provisioning and deployment
phases (in manifests), the document how to either rebuild IBP images or
suppress predictable interface naming policy on the bootstrap should be
provided for a customer. (The flaw is going to be fixed in the nearest
releases.)

---------------
Security impact
---------------

None.

--------------------
Notifications impact
--------------------

None.

---------------
End user impact
---------------

The user should be careful when assigning interface roles of several nodes at
once. Just because two nodes have the interface called enp2s0f0 doesn't mean
both these interfaces are attached to the same L2 network (unless the nodes'
hardware is the same).

Since the interfaces' names could be changed during upgrade, the user should
review the config files from the slave nodes to match new interfaces' names
with the old one.

------------------
Performance impact
------------------

None.

-----------------
Deployment impact
-----------------

None.

Switching to predictable interface naming may change interfaces' names during
deploying or upgrading a cluster. Potentially it is possible, that an interface
on a slave node get the name (during provisioning) differ from the name it has
previously taken on bootstrap. For example, if the bootstrap is based on a
differ OS, than OS using on provisioning (the boostrap is based on CentOS, but
Ubuntu is to be provisioned later and both using different naming policies).
But it doesn't impact on deployment if Nailgun agent passes to Nailgun correct
interface names, which shall be updated in the database before the
serialization the data has started.

----------------
Developer impact
----------------

The approach of using templates (like "eth*") for collecting information about
(physical) interface names should be avoided, because any network naming policy
could be used. The corresponding function returning list of (physical) network
interfaces should be implemented. It would be better to stick to MAC addresses
of network interfaces instead of interface names.

---------------------
Infrastructure impact
---------------------

Impact on fuel-devops
=====================

Please keep in mind, that achive predictable interface names two parameters
biosdevname=1 and net.ifnames=1 should be passed to the kernel on boot. The
next obligitory requirement (in case of deploying on VMs) is chose network
driver E1000, but not VMXNET2 or VMXNET3, using by default in most cases.


The main impact is to create interfaces in predictable way. Currently we
create interfaces based on infomation from fuel-devops/devops/settings.py,
lines are like: ::

  DEFAULT_INTERFACE_ORDER = 'admin,public,management,private,storage'
  'admin': ['eth0', 'eth1']

There is a work in fuel-qa/fuel-devops `template-based-testcases`_
to move that to yaml files with settings, so it is going to be fixed anyway.
Later changes will require simple rename in yaml files.

Interface order is used to correctly create interfaces in virtual domains.
Currently INTERFACE_ORDER is primary source of truth. Based on that ordering
we map networks to interfaces. ::

  fuel-devops/devops/models/environment.py

.. code-block:: python

   def create_interfaces(self, networks, node,
                         model=settings.INTERFACE_MODEL):
       interfaces = settings.INTERFACE_ORDER
       if settings.MULTIPLE_NETWORKS:
           logger.info('Multiple cluster networks feature is enabled!')
       if settings.BONDING:
           interfaces = settings.BONDING_INTERFACES.keys()

Also, IPMI driver is slightly affected: ::

  fuel-devops/devops/driver/ipmi/ipmi_driver.py

.. code-block:: python

   class DevopsDriver(object):
       interface_install_server='eth0',
       def _create_boot_menu(self, interface='eth0', ...

And node model. It is enough to rename eth0 to correctly mapped the
first interface: ::

  fuel-devops/devops/models/node.py

.. code-block:: python

   def pxe_boot_interface_is_eth0(self):
   @property
   def interfaces(self):
       return self.interface_set.order_by('id')

Impact on fuel-qa
=================

The main impact in fuel-qa is a communication with the nailgun. With current
nailgun scheme we need just to change interface information updates in
fuel-qa/fuelweb_test/models/fuel_web_client.py. There are 14 lines to send to
nailgun interfaces. Need to carefully update them with information from yaml
files and devops. For now we need to update info based on INTERFACE_ORDER and
test logic.

Simple fix in tests of fuel contrail plugini in fuel-qa repository::

  fuelweb_test/tests/plugins/plugin_contrail/test_fuel_plugin_contrail.py

.. code-block:: javascript

   raw_data = [{
              'mac': None,
              'mode': 'balance-rr',
              'name': 'bond0',
              'slaves': [
                  {'name': 'eth4'},
                  {'name': 'eth2'},
              ],

Also, simple fix in dhcrelay_check::

  fuel-qa/fuelweb_test/models/environment.py

.. code-block:: python

  def dhcrelay_check(self):
     with self.d_env.get_admin_remote() as admin_remote:
         out = admin_remote.execute("dhcpcheck discover "
                                 "--ifaces eth0 "

Simple replace of 'eth*' in currently used network templates, see::

 fuel-qa/fuelweb_test/network_templates/\*.yaml

This is a folder with yaml files which are going to be
base for tests. Same as above - replace 'eth*' is enough.

--------------------
Documentation impact
--------------------

Switching to predictable network interfaces' names may require some changes
in documentation. Diagrams and User Guide should be reviewed to support new
network schemes.

--------------------
Expected OSCI impact
--------------------

None.

--------------
Implementation
--------------

Assignee(s)
===========

Primary assignee:
  Albert Syriy <asyriy@mirantis.com>


Other contributors:
  Ivan Suzdal <isuzdal@mirantis.com>
  Alexei Sheplyakov <asheplyakov@mirantis.com>
  Alexandr Kostrikov <akostrikov@mirantis.com>
  Alexey Shtokolov <ashtokolov@mirantis.com>

Mandatory design review:
  Aleksandr Gordeev <agordeev@mirantis.com>
  Igor Kalnitsky <ikalnitsky@mirantis.com>
  Sergii Golovatiuk <sgolovatiuk@mirantis.com>
  Sergey Vasilenko <svasilenko@mirantis.com>
  Vladimir Kozhukalov <vkozhukalov@mirantis.com>
  Vladimir Kuklin <vkuklin@mirantis.com>

Work Items
==========

The following parts of the project require improving::

 * Nailgun
 * Fuel-main
 * Fuelmenu (LP#1512479)
 * Fuel-QA tests
 * Fuel scale tests

Dependencies
============

None.

-----------
Testing, QA
-----------

We are having impact on two subsystems: fuel-devops and fuel-qa. Jenkins jobs
are not affected.

Acceptance criteria
===================

Fuel should work well with different Ethernet interface naming policy. In
general Ethernet interface can have an arbitrary name.

----------
References
----------

.. _the launchpad blueprint: https://blueprints.launchpad.net/fuel/+spec/network-interfaces-naming-schema
.. _naming schemes: http://www.freedesktop.org/wiki/Software/systemd/PredictableNetworkInterfaceNames/
.. _bug reported on nailgun-agent: https://bugs.launchpad.net/fuel/+bug/1502198
.. _template-based-testcases: https://blueprints.launchpad.net/fuel/+spec/template-based-testcases
.. [1] `Reported bug for fuel-main <https://bugs.launchpad.net/fuel/+bug/1494223>`_
.. [2] `Bug. Undeterministic interface naming behaviour in Ubuntu <https://bugs.launchpad.net/mos/+bug/1487044>`_
.. [3] `Fix for the interface naming issue in fuel-main <https://review.openstack.org/#/c/223939>`_
.. [4] `Bug related fuel-menu <https://bugs.launchpad.net/fuel/+bug/1512479>`_
