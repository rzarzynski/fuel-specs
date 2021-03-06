..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
Implement nailgun extensions using stevedore
============================================

https://blueprints.launchpad.net/fuel/+spec/stevedore-extensions-discovery

--------------------
Problem description
--------------------

Nailgun has a possibility to extend its behaviour thanks to extensions system.
There are methods called on events like node create, node update, cluster
delete etc. which can be used to inject some logic. The problem is
that currently all extensions must be placed inside `extensions` module in
Nailgun's source code. Also in order to make extension visible for Fuel, it
must be imported and explicitly added to global `extensions` list.

It means that there is no elegant way for User to use extensions system
capabilities.

There may be some confusion between Nailgun `extension` and Fuel `plugin`.
Here are the main differences between them:

* Extensions system is mainly created for Fuel developers.
  It is an easy way to e.g. integrate other services with Nailgun which has a
  tremendous meaning in Fuel Modularization plan. But of course not only
  Fuel developers may use the extensions system.

* Extension's code is directly injected into Nailgun's source code. That
  allows developers to extends Nailgun features without modifying the source
  code itself. This is also the reason why extensions are `Python only`.

* Extensions are more like `middleware`. The main feature is that extensions
  system triggers handlers for specific events like `on_node_create`,
  `on_node_update`, `on_node_reset`, `on_cluster_delete` etc. Extension
  (which basically is just a subclass of `BaseExtension` class) can override
  this handlers and run some custom actions like e.g. informing other service
  about the data change to keep it up to date with Nailgun.

* Extensions can be distributed in every acceptable Python form which is
  python source code, egg, wheel, zip etc. They just have to use
  `nailgun.extensions` namespace to be visible for Nailgun.

----------------
Proposed changes
----------------

The extensions system must be refactored to meet the following conditions:

* It must be pluggable - User is able to write an extension, place it in
  separate package and add it to available extensions list just by running
  :code:`pip install <extension-name>`

* It must implement auto-discovery of extensions.

* Extension has `description` field which briefly describes its features.

The best solution here is to use stevedore - a manager for dynamic plugins in
Python. Stevedore uses namespaces to load the extensions so the proposed
namespace for nailgun extensions is `nailgun.extensions`.


Web UI
======

None


Nailgun
=======

Data model
----------

None


REST API
--------

None


Orchestration
=============


RPC Protocol
------------

None


Fuel Client
===========

None


Plugins
=======

None


Fuel Library
============

None

------------
Alternatives
------------

* We could write our own plugin system instead of using Stevedore. But:

  * In most cases it is not good to reinvent the wheel. It also applies for
    this one, since current extensions system doesn't need a lot of work to
    port it to Stevedore.

* We could use some other plugin system like `baseplugin` [#baseplugin]_. But:

  * As an OpenStack project we should reuse other OpenStack projects

  * Stevedore is already in global requirements.


--------------
Upgrade impact
--------------

* Extensions which are shipped with Fuel will be upgraded automatically.

* Extensions installed and managed separately from Nailgun won't be upgraded
  automatically and it's extension Developer responsibility to
  prepare right path for upgrade.

* Also all extensions which require database tables must provide alembic
  migration scripts.


---------------
Security impact
---------------

None


--------------------
Notifications impact
--------------------

None


---------------
End user impact
---------------

None

------------------
Performance impact
------------------

None

-----------------
Deployment impact
-----------------

The change is nailgun specific, so there's no Deployment impact.


----------------
Developer impact
----------------

Developer is able to extend Nailgun features by writing extension which uses
Nailgun's extensions base class and namespace which is `nailgun.extensions`.

It will be placed in separate package and the installation will be simple as
:code:`pip install <extension_name>`. Nailgun will detect new extension
automatically after restart.


---------------------
Infrastructure impact
---------------------

None


--------------------
Documentation impact
--------------------

Extensions mechanism should be described:

* How to write extension:

  * Where is the base class for extension

  * What is the minimal working extension (required properties etc.)

* What are the possibilities

* Nailgun namespace which is `nailgun.extensions`

* Example of simple extension with `logging` which logs appropriate message
  on every event like `on_node_create`, `on_node_update` etc.


--------------
Implementation
--------------

Assignee(s)
===========

Primary assignee: Sylwester Brzeczkowski <sbrzeczkowski@mirantis.com>

Other contributors:

  * Evgeny Li <eli@mirantis.com>

Mandatory design review:

  * Evgeny Li <eli@mirantis.com>
  * Igor Kalnitsky <igor@kalnitsky.org>


Work Items
==========

* Setup Nailgun with Stevedore. Add possibility to install extensions in
  separate packages

* Prepare simple `logging extension` as an example for documentation


Dependencies
============

* Stevedore module [#stevedore_docs]_.

* The change is related to Fuel integration with Bareon service
  [#bp_bareon_integration]_ which requires more pluggable extensions and at
  the same it is the perfect example of extensions system usage.


------------
Testing, QA
------------

Acceptance criteria
===================

* Install extension from separate package and check if it appears in an
  extensions list after Nailgun is restarted.


----------
References
----------

.. [#baseplugin] http://pluginbase.pocoo.org/
.. [#stevedore_docs] http://docs.openstack.org/developer/stevedore/index.html
.. [#bp_bareon_integration] https://blueprints.launchpad.net/fuel/+spec/fuel-bareon-api-integration
