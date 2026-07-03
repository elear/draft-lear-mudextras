---
title: "Some MUD Extensions and Clarifications"
abbrev: "More MUD"
category: std

docname: draft-lear-iotops-mudextras-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Operations and Management"
workgroup: "IOT Operations"
keyword:
 - MUD
 - IOT
venue:
  group: "IOT Operations"
  type: "Working Group"
  mail: "iotops@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/iotops/"
  github: "elear/draft-lear-mudextras"
  latest: "https://elear.github.io/draft-lear-mudextras/draft-lear-iotops-mudextras.html"

author:
 -
    fullname: "Eliot Lear"
    organization: Cisco Systems, Inc.
    email: "lear@cisco.com"

normative:

informative:

...

--- abstract

Manufacturer Usage Descriptions (MUD) provide a means to describe device network behavior.  This memo clarifies some aspects that may improve both usability and interoperability.  Some examples include how to handle IP-based access-lists, broadcasts and multicasts of various forms, and QoS.


--- middle

# Introduction

Manufacturer Usage Descriptions (MUD) {{!RFC8520}} provide a means to describe device network behavior.  Since its initial standardization, there have been a number of questions and clarifications that have arisen.  The goal of a MUD file is to accurately describe the network behavior of a device in a way that is independent of the network topology.  Primarily the topological concern is local; that is, within an enterprise or home network.  Certain cases make that challenging:

* Some devices make use of directed broadcasts, which are not supported in all networks.  When they are, there is a need to understand each and every network segment to which such a device may be attached.

* Some devices hardcode certain IP addresses.
MUD itself augments {{!RFC8519}}, which specifies an access control list (ACL) schema in YANG.  Within that context, therefore, it should be possible to employ ACLs that match that schema.

While one could argue whether or not either of these approaches should be used by device manufacturers, the reality is that they are used.  Therefore, MUD should provide a means to describe them.


## Conventions and Definitions

{::boilerplate bcp14-tagged}

# Clarifications

## ACLs in MUD

As mentioned above, RFC 8519 specifies a YANG schema for ACLs.  Nothing in the specification prevents the use of ACLs to describe the network behavior of devices in a MUD file.  For example, the following is a valid MUD file that describes a device that is allowed to connect to a cloud service.

~~~~~
{
  "ietf-mud:mud": {
    "mud-version": 1,
    "mud-url": "https://iot.example.com/modelX.json",
    "mud-signature": "https://iot.example.com/modelX.p7s",
    "last-update": "2022-01-05T13:30:31+00:00",
    "cache-validity": 48,
    "is-supported": true,
    "systeminfo": "Sample IoT device",
    "mfg-name": "Example, Inc.",
    "documentation": "https://iot.example.com/doc/modelX",
    "model-name": "modelX",
    "from-device-policy": {
      "access-lists": {
        "access-list": [
          {
            "name": "mud-65443-v4fr"
          }
        ]
      }
    },
    "to-device-policy": {
      "access-lists": {
        "access-list": [
          {
            "name": "sample-ipv4-acl"
          }
        ]
      }
    }
  },
  "ietf-access-control-list:acls": {
    "acl": [
      {
        "name": "sample-ipv4-acl",
        "type": "ipv4-acl-type",
        "aces": {
          "ace": [
            {
              "name": "permit-https-to-my-cloud-service",
              "matches": {
                "ipv4": {
                  "protocol": 6,
                  "destination-ipv4-network": "10.1.2.3.4/32"
                },
                "tcp": {
                  "destination-port": {
                    "operator": "eq",
                    "port": 443
                  }
                }
              },
              "actions": {
                "forwarding": "accept"
              }
            }
          ]
        }
      }
    ]
  }
}
~~~~~
{:#figacl title="Example ACL in MUD file"}

A few cautions about using native IP addresses in MUD files:

* They should only ever refer to globally unique addresses that are coordinated by the device manufacturer.
* Address changes will necessitate a new MUD file, which must be signed retrieved by mud managers.

### Discussion

Some device manufacturers will use hardcoded IP addresses to bootstrap functions like the domain name system (DNS).  The use of anycast servers is not uncommon.  Others simply do not want to introduce a dependency on DNS.

## Directed Broadcasts

Some devices make use of directed broadcasts to communicate with devices either on the same subnet or on remote networks.  Directed broadcasts require local toplogical knowledge, specifically the subnet mask of the network to which the device is attached.  That information cannot be used across deployments, and so is inappropriate for MUD files.

To address this, we specify a MUD extension here that indicates that the device uses directed broadcasts.

Extension name: directed-broadcasts

A new object is added to the MUD file, called directed-broadcasts.  It contains two boolean elements:

* inbound: indicates whether the device must receive directed broadcasts.
* outbound: indicates whether the device must send directed broadcasts.

###  The directed-broadcast extension

The YANG tree for this extension is as follows:

~~~~
module: ietf-mud-directed-broadcasts

  augment /mud:mud:
    +--rw directed-broadcasts
       +--rw inbound?    boolean
       +--rw outbound?   boolean
~~~~
{:#figmud-directed-broadcasts title="YANG tree for directed-broadcasts extension"}


The YANG model for this extension is as follows:

~~~~~
module ietf-mud-directed-broadcasts {
  yang-version 1.1;
  namespace "urn:ietf:params:xml:ns:yang:ietf-mud-directed-broadcasts";
  prefix mud-directed-broadcasts;

  import ietf-mud {
    prefix mud;
    reference "RFC 8520: Manufacturer Usage Description (MUD)";
  }

  organization
    "IETF IOTOPS Working Group";
  contact
    "WG Web:   <https://datatracker.ietf.org/wg/iotops/>
     WG List:  <mailto:iotops@ietf.org>
     Author:   Eliot Lear <mailto:lear@lear.ch>";
  description
    "
     Copyright (c) 2026 IETF Trust and the persons identified as
     authors of the code.  All rights reserved.

     Redistribution and use in source and binary forms, with or
     without modification, is permitted pursuant to, and subject to
     the license terms contained in, the Revised BSD License set
     forth in Section 4.c of the IETF Trust's Legal Provisions
     Relating to IETF Documents
     (https://trustee.ietf.org/license-info).

     This version of this YANG module is part of RFC XXXX
     (https://www.rfc-editor.org/info/rfcXXXX); see the RFC itself
     for full legal notices.

    This module defines a MUD extension for directed broadcasts.";
  revision 2024-06-01 {
    description
      "Initial revision.";
    reference
      "RFC XXXX: Some MUD Extensions and Clarifications";
  }
  grouping directed-broadcasts-group {
    description
      "Indicates whether the device uses directed broadcasts.";
    container directed-broadcasts {
      description
        "Indicates whether the device uses directed broadcasts.";
      leaf inbound {
        type boolean;
        description
          "Indicates whether the device must receive directed broadcasts.";
      }
      leaf outbound {
        type boolean;
        description
          "Indicates whether the device must send directed broadcasts.";
      }
    }
  }
  augment "/mud:mud" {
    description
      "Augments the MUD model with directed-broadcasts.";
    uses directed-broadcasts-group;
  }
}
~~~~~
{:#figmud-directed-broadcasts-yang title="YANG model for directed-broadcasts extension"}

### Example

The following is an example of a MUD file that indicates that the device uses directed broadcasts in both directions.

~~~~~
{
  "ietf-mud:mud": {
    "mud-version": 1,
    "mud-url": "https://iot.example.com/modelX.json",
    "mud-signature": "https://iot.example.com/modelX.p7s",
    "last-update": "2022-01-05T13:30:31+00:00",
    "extensions": [
      "directed-broadcasts"
    ],
    "cache-validity": 48,
    "is-supported": true,
    "systeminfo": "Sample IoT device",
    "mfg-name": "Example, Inc.",
    "documentation": "https://iot.example.com/doc/modelX",
    "model-name": "modelX",
    "directed-broadcasts": {
      "inbound": true,
      "outbound": true
    }
  }
}
~~~~~
{:#figmud-directed-broadcasts-example title="Example MUD file with directed-broadcasts extension"}

Note that in all likelihood there would also be ACLs in the MUD file, but they are omitted here for brevity.

### Discussion {#dibroaddisc}

Directed broadcasts have well known security issues (see {{?RFC2644}}).  However, they are used in circumstances where a limited amount of configuration is considered acceptable, and where other mechanisms such as multicast cannot be expected to be available in **all** deployments.  The purpose of this extension is **not** to encourage the use of directed broadcasts, but rather to provide a means to describe them in MUD files when they are used.



# Security Considerations

The YANG module specified in this document defines a schema for data
that is designed to be accessed via network management protocols such
as NETCONF {{?RFC6241}} or RESTCONF {{?RFC8040}}.  The lowest NETCONF layer
is the secure transport layer, and the mandatory-to-implement secure
transport is Secure Shell (SSH) {{?RFC6242}}.  The lowest RESTCONF layer
is HTTPS, and the mandatory-to-implement secure transport is TLS
{{?RFC8446}}.

The NETCONF access control model {{?RFC8341}} provides the means to
restrict access for particular NETCONF or RESTCONF users to a
preconfigured subset of all available NETCONF or RESTCONF protocol
operations and content.

MUD files are intended to be retrieved from a web server, and so the security considerations of {{RFC8520}} apply.  In addition, see {{dibroaddisc}} for a discussion of the security implications of directed broadcasts.

# IANA Considerations

## Directed Broadcasts MUD Extension

IANA is requested to make the following additions to the "Manufacturer Usage Description (MUD) Extensions" registry:

~~~~~
Name: directed-broadcasts
Reference: [RFCXXXX] (this document)
~~~~~

The following YANG namespace is registered for the directed-broadcasts MUD extension:
* Namespace: urn:ietf:params:xml:ns:yang:ietf-mud-directed-broadcasts
* Prefix: mud-directed-broadcasts




--- back

# Acknowledgments

TODO acknowledge.

# Changes

* Initial revision.
