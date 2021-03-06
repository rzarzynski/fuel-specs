..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================
Selectable offloading type
==========================

https://blueprints.launchpad.net/fuel/+spec/selectable-offloading-type

This blueprint describes a way to automate process of configuration offloading
modes for node's physical interfaces, and, to provide possibility for user to
configure supported offloading modes via API/CLI/UI.

Problem description
===================

In current implementation it's already possible to control specific offloading
type for node's physical interface via CLI. But, in this case user should care
about available offloading types of physical interfaces himself (check that
specified offloading modes is supported by Ethernet card, collect available
modes, etc.). This blueprint provide way to automate these boring actions.
Also, it allows to avoid changing of unsupported offloading modes.

Proposed change
===============

Firstly, it will be necessary to extend nailgun agent to collect
available offloading types for each node's interfaces during
the bootstrap stage. Currently we use ruby rethtool gem to
collect node's interfaces data. But, for some reasons, this library
doesn't provide offloading related interface information. So,
we can collect necessary information by parsing cli output of
ethtool for dedicated interface.

For example, ethtool provides following cli output for interface's
features::

    rx-checksumming: on
    tx-checksumming: on
        tx-checksum-ipv4: on
        tx-checksum-ip-generic: off [fixed]
        tx-checksum-ipv6: on
        tx-checksum-fcoe-crc: off [fixed]
        tx-checksum-sctp: on
    scatter-gather: on
        tx-scatter-gather: on
        tx-scatter-gather-fraglist: off [fixed]
    tcp-segmentation-offload: on
        tx-tcp-segmentation: on
        tx-tcp-ecn-segmentation: off [fixed]
        tx-tcp6-segmentation: on
    udp-fragmentation-offload: off [fixed]
    generic-segmentation-offload: on
    ..............................

In this case we should collect all offloading types what have not
'fixed' attribute during the bootstrap stage, alter on/off offloading
options to 'None' value, and then transform formatted data to Json object
and sent it to the Nailgun server.
Nailgun's NodeNICInterface and NodeBondInterface data models should be
extended with Json field containing array of supported offloading types
to properly receive nailgun's agent data. Json array will contain objects
having following structure::

  "name" - type string // offloading name
  "state" - type boolean (true/false/null) // offloading state
  "sub_modes": [] - type array // array of sub offloading modes

Json field in Nailgun db after agent passed interfaces data to Nailgun server::

  "offload_modes":
  [
      {
          "name": "rx-checksumming",
          "state": null,
          "sub_modes": []
      },
      {
          "name": "tx-checksumming",
          "state": null,
          "sub_modes": [
              {
                  "name": "tx-checksum-ipv4",
                  "state": null,
                  "sub_modes": []
              },
              {
                  "name": "tx-checksum-ipv6",
                  "state": null,
                  "sub_modes": []
              },
              {
                  "name": "tx-checksum-sctp",
                  "state": null,
                  "sub_modes": []
              }
          ]
      },
      {
          "name": "scatter-gather",
          "state": null,
          "sub_modes": [
              {
                  "name": "tx-scatter-gather",
                  "state": null,
                  "sub_modes": []
              }
          ]
      },
      {
          "name": "tcp-segmentation-offload",
          "state": null,
          "sub_modes": [
              {
                  "name": "tx-tcp-segmentation",
                  "state": null,
                  "sub_modes": []
              },
              {
                  "name": "tx-tcp6-segmentation",
                  "state": null,
                  "sub_modes": []
              }
          ]
      },
      {
          "name": "generic-segmentation-offload",
          "state": null,
          "sub_modes": []
      },
      ..............................
  ]

Initially all offloading types in array will have 'None' state's value.
Further, these offloading types may be modified to false/true values via
API/CLI/UI. Offloading types should be sorted alphabetically in UI before
it will be shown to user for more pretty and usable output.

Json field in Nailgun db after user configured interface's offloading types::

  "offload_modes":
  [
      {
          "name": "rx-checksumming",
          "state": true,
          "sub_modes": []
      },
      {
          "name": "tx-checksumming",
          "state": null,
          "sub_modes": [
              {
                  "name": "tx-checksum-ipv4",
                  "state": null,
                  "sub_modes": []
              },
              {
                  "name": "tx-checksum-ipv6",
                  "state": null,
                  "sub_modes": []
              },
              {
                  "name": "tx-checksum-sctp",
                  "state": null,
                  "sub_modes": []
              }
          ]
      },
      {
          "name": "scatter-gather",
          "state": null,
          "sub_modes": [
              {
                  "name": "tx-scatter-gather",
                  "state": null,
                  "sub_modes": []
              }
          ]
      },
      {
          "name": "tcp-segmentation-offload",
          "state": null,
          "sub_modes": [
              {
                  "name": "tx-tcp-segmentation",
                  "state": null,
                  "sub_modes": []
              },
              {
                  "name": "tx-tcp6-segmentation",
                  "state": null,
                  "sub_modes": []
              }
          ]
      },
      {
          "name": "generic-segmentation-offload",
          "state": null,
          "sub_modes": []
      },
      ..............................
  ]

Here we have 'true' value for enabled offloading modes, 'false' for disabled
modes, and, 'null' for untouched modes (information about this modes will not
be passed to serialized deployment info).
As if we have hierarchical structure additional dependencies will be present::

  * when we disable offloading mode all its sub modes should be disabled
  * offloading mode should be disabled if all its sub modes are disabled

These dependencies should be supported via CLI/API/UI.
Extra 3-state checkbox for each incoming offloading type should be added to
node's interfaces UI tab to configure offloading types for physical
interfaces/bonds.
Checkbox state will be based on offloading mode's state from "offload_modes"
field::

  * true - value for enabled offloading modes
  * false - value for disabled offloading modes
  * null - value for default offloading modes

It may be hidden by default, and will be invoke in case if
user touch specific button.
Fix frontend to calculate available modes for bond interfaces
properly. UI should calculate intersection (or union) of offloading
types available when setup is being performed for a set of nodes
(every of which could have different offloading types supported for
the NICs with same names).
Currently, selectable offloading types are already supported by
puppet manifests. It will be enough to generate proper Hash field
via Nailgun and deliver it to the puppet manifests as it is.

Also, I want to add several examples regarding to changes in
node's yaml file and how to nailgun should serialise data to make
it handled properly via puppet.
We have two types of interfaces from the API/CLI/UI side of view:
physical interfaces and bond interfaces ( if we are not going to hack
transformations section using CLI ). The offloading types tuning is
similar for physical and bond interfaces in the fact that we have the
identical ethtool injection format for both cases.

For example::

  ethtool:
    offload:
        rx-checksumming:              true (or false)
        tx-checksumming:              true (or false)
        tcp-segmentation-offload:     true (or false)
        udp-fragmentation-offload:    true (or false)
        generic-segmentation-offload: true (or false)
        ....

In case if we are going to change offloading configuration for
physical interface we should add corresponding offload option
as the additional property of the interface object:

For example::
  ['network_scheme']['interfaces']['#interface_name'][ethtool]

In case if we are going to change offloading configuration for
bond interface we should add corresponding offload option
as the additional property of the bond interface object:

For example::
  ['network_scheme']['transformations'][#action_id]\
    ['#interface_properties'][ethtool]

It means that you should find needful #action_id using bond name
if you want to change it's offloading configuration. This change
will be applied for all bonded physical interfaces.

Alternatives
------------

None

Data model impact
-----------------

Nailgun's NodeNICInterface and NodeBondInterface data models should
be extended with Json field containing array of supported offloading
types. This field will be empty initially, and it's supposed to be filled
using nailgun agent data during the bootstrap stage for physical interfaces.
In case of bond interface this property will be filled during the environment
configuration process.

REST API impact
---------------

NodeValidator should be extended to handle incorrect node's offloading
types data.

Upgrade impact
--------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

User will be able to select physical interfaces offloading type via UI and CLI.

Performance Impact
------------------

Network performance may be increased due to more flexible offloading
types configuration.

Plugin impact
-------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

Nailgun's NodeNICInterface data model will be extended with
new Json field.

Infrastructure impact
---------------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Valyavskiy Viacheslav <slava-val-al>

Work Items
----------

* Extend nailgun agent to collect available offloading
  types for each node's interface during the bootstrap
  stage
* Extend Nailgun's NodeNICInterface data model to add
  one more Json field containing array of supported offloading
  types
* Add 3-state checkbox for each incoming offloading type
  should be added to node's interfaces UI tab to
  configure offloading types for physical interfaces/bonds
* Fix frontend to calculate available modes for bond
  interfaces properly

Acceptance criteria
-------------------

User is able to configure supported offloading modes for node's physical
interfaces via API/UI/CLI.
User is able to configure offloading modes for node's bond interfaces
via API/UI/CLI (available modes will be based on supported offloading modes
of the bonded interfaces).


Dependencies
============

None

Testing
=======

As there is no option to emulate different drivers with different offloading
options supported, we should test current feature using bare metal servers
with various set of network cards supporting different offloading types.

Documentation Impact
====================

Ability to control physical interface's offloading type should be
documented in Fuel Deployment Guide.

References
==========

None
