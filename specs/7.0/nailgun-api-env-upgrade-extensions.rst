..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
API Extensions For Environment Upgrade
======================================

https://blueprints.launchpad.net/fuel/+spec/nailgun-api-env-upgrade-extensions

Certain aspects of side-by-side upgrade procedure outlined in `this blueprint
<https://blueprints.launchpad.net/fuel/+spec/upgrade-major-openstack-environment>`_
have to be performed on Fuel side, especially those operations that require
database modifications. We propose extensions to Nailgun API that facilitate
creation of special type of environment to serve as a replacement for original
environment targeted for upgrade.


Problem description
===================

Current upgrade mode assumes that we create additional environment with
upgraded version of the Fuel Installer (also referred as Upgrade Seed or
shadow environment). This environment is used to install nodes with upgraded
versions of MOS and Operating System.

Upgrade Seed environment must have the same configuration as the original one
in terms of arhictecture options and settings for individual services. In the
current version of upgrade procedure we rely on Fuel API to provide complete
configuration of the environment. We tweak the downloaded configuration files
to upgrade attributes to the new version, using our best knowledge of changes
to settings derived from Nailgun codebase. Then we create a new environment
with given attributes and rely on Fuel to install nodes reassigned to Upgrade
Seed environment.

In addition, we need to have services credentials replicated to the shadow
environment. Currently we do it in Nailgun DB directly.

Finally, when we add Controller nodes to the Upgrade Seed environemnt, we make
changes to Nailgun DB to make sure that those nodes get the same IP addresses
as Controllers in the original environment. It is needed to transparently
reconnect Compute and Storage nodes to the new Contollers later in the upgrade
scenario.

However, this approach has a number of potential and actual problems:

* Lack of versioning in Cluster object causes huge work to be done to
  determine how attributes syntax and semantics changes between releases.

* Fuel API might not expose some attributes which default values might change
  between releases.

* Generated attributes of the environment are not exposed through API, so we
  have to reengineer the DB model and make changes to the DB directly, risking
  to create inconsistent records.

* IP addresses assignment is also done directly via Nailgun DB by external
  script. It doesn't reflect the schema changes and it might lead to incorrect
  assignment of addresses in other, 'clean' environments.

Proposed change
===============

During the upgrade procedure, we create an special environment that
complies to the following requirements:

* Installed a release you want to upgrade your original environment to.

* Has the same settings as the original environment in terms of
  selected components and architecture options.

* If the format of certain cluster attributes changed in the new release,
  those attributes updated while being copied from the original cluster.

* The same IP addresses allocated to the new environment as were allocated to
  the original environment.

We propose to extend definition of environment with Upgrade Seed environment
type. Such environment must refer to the original environment and have
settings copied from the original environment instead of generated in a usual
way.

Separate API call will be added to create Upgrade Seed environment. Handler to
that call must copy and upgrade settings of the original environment and
create a new environment with those settings, both editable and generated.

We must add Controller nodes to Upgrade Seed environment. Network and disk
settings for those nodes must be replicated from a Controller node in the
original environment during the assignment. IP addresses in Public and
Management networks allocated to those nodes must duplicate addresses
allocated to Controllers of original environment.

Alternatives
------------

Alternative implementation of Upgrade Seed environment logic is external
script that performs the following actions:

* Copy editable env settings via Nailgun API

* Copy generated settings from original to upgrade seed environment in Nailgun
  DB via ``psql`` client

* Modify IP address assignments in Nailgun DB via ``psql`` client

* Change deployment information for nodes in the Upgrade Seed environment to
  ensure changes in Fuel installer behavior during deployment of the Seed.

This methodology, while working and producing acceptable results, is difficult
to maintain outside of Fuel mainstream. Direct communications with database
pose data consistency threats. It will be hard to integrate with the Fuel Web
UI in future.

Data model impact
-----------------

* Create a new table to store mapping between clusters and their 'seeds',
  related as 1:1:

::

    class UpgradeRelation(BaseModel):
        __tablename__ = "upgrade_relation"
        id = Column(Integer, primary_key = True)
        seed_cluster_id = Column(Integer,
                                 unique = True)
        orig_cluster_id = Column(Integer,
                                 unique = True)


REST API impact
---------------

We propose to add the following extensions to the Nailgun API.

Upgrade an environment
++++++++++++++++++++++

This is a root resource for all methods related to upgrade. In future, when
single-click upgrade is developed, the single call to this resource will
upgrade the environment.

In 7.0, there are no handlers for this resource, thus any request to it shall
return error and inform that no methods are implemented for it. The resource
itself serves as a root for other resources that realize certain parts of
upgrade logic.

* Specification for the method

  * Upgrades a given cluster by installing new controllers and upgrading state
    and configurations of the original OpenStack cloud.

  * Method type: POST

  * Normal http response code(s): N/A

  * Expected http response code(s):

    * 404 Not Found
      In Fuel 7.0, this resource is not implemented

  * URL for the resource: ``/cluster/<cluster_id>/upgrade``

  * Parameters which can be passed via the url:

    * ``cluster_id``: ID of the cluster to upgrade

  * JSON schema definition for the body data if allowed: N/A

  * JSON schema definition for the response data if any: N/A

Clone upgraded environment
++++++++++++++++++++++++++

This is the first step in process of upgrade of MOS environment. Creates
Upgrade Seed cluster with configuration that matches configuration of the
original cluster, but has a new release version.

* Specification for the method

  * Create a new cluster with settings and attributes copied from the
    specified cluster, including generated attributes (i.e. service passwords
    and other credentials).

  * Method type: POST

  * Normal http response code(s): 200 OK

  * Expected error http response code(s)

    * 400 Bad Request
      Malformed request body or missing parameters.

    * 404 Not Found
      A cluster or release with given ID was not found in database.

    * 409 Conflict
      The cluster with given ID has attributes incompatible with the upgrade
      procedure (e.g. deprecated or deleted attributes)

    * 405 Method Not Allowed
      The cluster with given ID already being upgraded, i.e. a 'shadow' cluster
      was created already

  * URL for the resource: ``/cluster/<cluster_id>/upgrade/clone``

  * Parameters which can be passed via the url:

    * ``cluster_id``: ID of the cluster to copy parameters from it

  * JSON schema definition for the body data:

::

    {
         "$schema": "http://json-schema.org/draft-04/schema#",
         "title": "Cluster Clone Parameters",
         "description": "Serialized parameters to clone clusters",
         "type": "object",
         "properties": {
             "name": {"type": "string"},
             "release_id": {"type": "number"},
         },
    }

  * JSON schema definition for the response data:

::

    {
        "$schema": "http://json-schema.org/draft-04/schema#",
        "title": "Cluster",
        "description": "Serialized Cluster object",
        "type": "object",
        "properties": {
            "id": {"type": "number"},
            "name": {"type": "string"},
            "mode": {
                "type": "string",
                "enum": list(consts.CLUSTER_MODES)
            },
            "status": {
                "type": "string",
                "enum": list(consts.CLUSTER_STATUSES)
            },
            "net_provider": {
                "type": "string",
                "enum": list(consts.CLUSTER_NET_PROVIDERS)
            },
            "grouping": {
                "type": "string",
                "enum": list(consts.CLUSTER_GROUPING)
            },
            "release_id": {"type": "number"},
            "pending_release_id": base_types.NULLABLE_ID,
            "replaced_deployment_info": {"type": "object"},
            "replaced_provisioning_info": {"type": "object"},
            "is_customized": {"type": "boolean"},
            "fuel_version": {"type": "string"},
            "original_cluster_id": {"type": "number"}
        }
    }

Directly assign node to Upgrade Seed cluster
++++++++++++++++++++++++++++++++++++++++++++

This method assigns a node to Upgrade Seed cluster without deleting it from
database. This allows to keep ID of the node and IP address assigned to it,
given the network settings are the same in original and 'shadow' cluster.

Only nodes from the original cluster for the given Upgrade Seed cluster can be
assigned with this call.

* Specification for the method

  * Assign a node without changing roles to the Upgrade Seed environment. IP
    addresses assignment and ID of the node do not change. 

  * Method type: POST

  * Normal http response code(s): 200 OK

  * Expected error http response code(s)

    * 400 Bad Request
      Malformed request body or missing parameters.

    * 404 Not Found
      A cluster or a node with given ID was not found in database.

    * 405 Method Not Allowed
      A node identified by ``node_id`` parameter in the request data is not
      allocated to an original cluster, based on the mapping in table
      ``upgrade_relation``.

    * 409 Conflict
      One or more roles assigned to the node in the original cluster are not
      defined in the Upgrade Seed cluster.

  * URL for the resource: ``/cluster/<cluster_id>/upgrade/assign``

  * Parameters which can be passed via the url:

    * ``cluster_id``: ID of the Upgrade Seed cluster

  * JSON schema definition for the body data:

::

    {
         "$schema": "http://json-schema.org/draft-04/schema#",
         "title": "Cluster Clone Parameters",
         "description": "Serialized parameters to clone IPs",
         "type": "object",
         "properties": {
             "node_id": {"type": "number"},
         },
    }

  * JSON schema definition for the response data:

::

    {
        "$schema": "http://json-schema.org/draft-04/schema#",
        "title": "Node",
        "description": "Serialized Node object",
        "type": "object",
        "properties": {
            "mac": base_types.MAC_ADDRESS,
            "ip": base_types.IP_ADDRESS,
            "meta": {
                "type": "object",
                "properties": {
                    "interfaces": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "properties": {
                                "ip": base_types.NULLABLE_IP_ADDRESS,
                                "netmask": base_types.NET_ADDRESS,
                                "mac": base_types.MAC_ADDRESS,
                                "state": {"type": "string"},
                                "name": {"type": "string"},
                                "driver": {"type": "string"},
                                "bus_info": {"type": "string"},
                                "pxe": {"type": "boolean"}
                            }
                        }
                    },
                    "disks": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "properties": {
                                "model": base_types.NULLABLE_STRING,
                                "disk": {"type": "string"},
                                "size": {"type": "number"},
                                "name": {"type": "string"},
                            }
                        }
                    },
                    "memory": {
                        "type": "object",
                        "properties": {
                            "total": {"type": "number"}
                        }
                    },
                    "cpu": {
                        "type": "object",
                        "properties": {
                            "spec": {
                                "type": "array",
                                "items": {
                                    "type": "object",
                                    "properties": {
                                        "model": {"type": "string"},
                                        "frequency": {"type": "number"}
                                    }
                                }
                            },
                            "total": {"type": "integer"},
                            "real": {"type": "integer"},
                        }
                    },
                    "system": {
                        "type": "object",
                        "properties": {
                            "manufacturer": {"type": "string"},
                            "version": {"type": "string"},
                            "serial": {"type": "string"},
                            "family": {"type": "string"},
                            "fqdn": {"type": "string"},
                        }
                    },
                }
            },
            "id": {"type": "integer"},
            "status": {"enum": list(consts.NODE_STATUSES)},
            "cluster_id": base_types.NULLABLE_ID,
            "name": {"type": "string"},
            "manufacturer": base_types.NULLABLE_STRING,
            "os_platform": base_types.NULLABLE_STRING,
            "is_agent": {"type": "boolean"},
            "platform_name": base_types.NULLABLE_STRING,
            "group_id": {"type": "number"},
            "fqdn": base_types.NULLABLE_STRING,
            "kernel_params": base_types.NULLABLE_STRING,
            "progress": {"type": "number"},
            "pending_addition": {"type": "boolean"},
            "pending_deletion": {"type": "boolean"},
            "error_type": base_types.NULLABLE_ENUM(list(consts.NODE_ERRORS)),
            "error_msg": {"type": "string"},
            "online": {"type": "boolean"},
            "roles": {"type": "array"},
            "pending_roles": {"type": "array"},
            "agent_checksum": {"type": "string"}
        },
    }

Upgrade impact
--------------

This patch set will extend the standard Nailgun API and will be a subject to
modification during the upgrade procedure as a part of Nailgun codebase.

Security impact
---------------

Clone environment call creates a copy of cluster's generated attributes, which
include sensitive data like passwords for system users. Sensitive data cannot
be accessed directly using this API call.

Notifications impact
--------------------

No impact.

Other end user impact
---------------------

This change will not have impact on python-fuelclient in 7.0 release cycle.
Functions implemented in this change shall be added to python-fuelclient in
future release cycles.

Performance Impact
------------------

No impact.

Plugin impact
-------------

No impact.

Other deployer impact
---------------------

No impact.

Developer impact
----------------

No impact.

Infrastructure impact
---------------------

This change will require additional system test to verify that a clone of the
cluster was created successfully.

This change must be also tested against upgrade tests in a sense that it
properly creates a clone of the cluster with new release version.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ikharin (Ilya Kharin)

Other contributors:
  yorik.sar (Yuriy Taraday)

Mandatory design reviewers:
  mscherbakov (Mike Scherbakov)
  rpodolyaka (Roman Podolyaka)
  enikanorov (Eugene Nikanorov)

QA:
  smurashov (Sergey Murashov)

Work Items
----------

* implement API handler for url ``/cluster/<id>/upgrade``.

* implement API handler for url ``/cluster/<id>/upgrade/clone``.

* implement API handler for url ``/cluster/<id>/upgrade/assign``.

Dependencies
============

None.

Testing
=======

This change will require unittest coverage.

This change will require development of new functional tests for 3 API calls
listed in Work Items section above.

This change will require additional system test to verify that a clone of the
cluster was created successfully.

This change must be also tested against upgrade tests in a sense that it
properly creates a clone of the cluster with new release version.

Acceptance criteria for the cluster clone feature is a successful creation of
an environment with the upgraded release and cloned attributes. This cluster
must have a corresponding entry in table ``upgrade_relation`` set to proper
values.

Acceptance criteria for assignment feature is successful addition of Contoller
nodes to the environment with proper attributes in deployment settings.

Documentation Impact
====================

The feature will be documented along with the other API handlers.

References
==========

