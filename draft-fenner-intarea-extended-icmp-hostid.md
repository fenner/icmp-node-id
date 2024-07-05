---
title: "Extending ICMP for Node Identification"
abbrev: "ICMP Node ID"
category: std

docname: draft-fenner-intarea-extended-icmp-hostid-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Internet"
workgroup: "Internet Area Working Group"
keyword:
 - ICMP
 - IPv6 nexthops
 - Node identification
venue:
  group: "Internet Area Working Group"
  type: "Working Group"
  mail: "int-area@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/int-area/"
  github: "fenner/icmp-node-id"
  latest: "https://fenner.github.io/icmp-node-id/draft-fenner-intarea-extended-icmp-hostid.html"

author:
- name: Bill Fenner
  org: Arista Networks
  street: 5453 Great America Parkway
  city: Santa Clara
  code: '95054'
  region: California
  country: USA
  email: fenner@fenron.com
- name: Reji Thomas
  org: Arista Networks
  street: Global Tech Park
  city: Bangalore
  region: Karnataka
  code: '560103'
  country: India
  email: reji.thomas@arista.com

normative:
  RFC3629:
  RFC2277:
  RFC4884:
  RFC5837:
  RFC7317:
  IANA.address-family-numbers:

informative:
  I-D.chroboczek-intarea-v4-via-v6:


--- abstract

RFC5837 describes a mechanism for Extending ICMP for Interface and Next-Hop Identification,
which allows providing additional information in an ICMP error that helps identify
interfaces participating in the path.  This is especially useful in environments
where each interface may not have a unique IP address to respond to, e.g., a traceroute.

This document introduces a similar ICMP extension for Node Identification.
It allows providing a unique IP address and/or a textual name for the node, in
the case where each node may not have a unique IP address (e.g., the
IPv6 nexthop deployment case described in draft-chroboczek-intarea-v4-via-v6).

--- middle

# Introduction

In addition to adding incoming interface information to a traceroute
using the mechanisms described in {{RFC5837}}, a network operator
may be interested in adding information to identify nodes themselves.
{{I-D.chroboczek-intarea-v4-via-v6}} describes a scenario in which individual
nodes do not have unique IPv4 addresses to use to reply to an IPv4
traceroute, so additional information is needed.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Node Identification Object

This section defines the Node Identification Object, an ICMP Extension
Object with a Class-Num (Object Class Value) of 5 that can be appended
to the following messages:

- ICMPv4 Time Exceeded

- ICMPv4 Destination Unreachable

- ICMPv4 Parameter Problem

- ICMPv6 Time Exceeded

- ICMPv6 Destination Unreachable

For reasons described in {{RFC4884}}, this extension cannot be appended
to any of the currently defined ICMPv4 or ICMPv6 messages other than
those listed above.

The extension defined herein MAY be appended to any of the above
listed messages and SHOULD be appended whenever required to identify
the node and when local policy or security
considerations do not supersede this requirement.

Similarly to the Interface Identification Object defined in {{RFC5837}},
there are two different pieces of information that can appear in a
Node Information Object.

1. An IP Address Sub-Object MAY be included, containing an address
   of sufficient scope to identify the node within the domain.
   The IP Address Sub-Object is defined in {{IPAddr}} of this memo.

2. A Node Name Sub-Object MAY be included, as specified in {{Name}},
   containing up to 63 octets of the yang sys:hostname or another
   appropriate name uniquely identifying the node.

## C-Type Meaning in a Node Identification Object

The C-Type contains a bitmask describing what information is included
in this Node Identification Object.

~~~~
Bit     0       1       2       3       4       5       6       7
    +-------+-------+-------+-------+-------+-------+-------+-------+
    |               Reserved                | IPAddr|  name | Rsvd2 |
    +-------+-------+-------+-------+-------+-------+-------+-------+
~~~~
{: #ctypeFig title='C-Type for the Node Identification Object'}

The following are bit-field definitions for C-Type:

Reserved (bits 0-4): These bits are reserved for future use
and MUST be set to 0 on transmit and ignored on receipt.

IP Addr (bit 5) : When set, a Node IP Address Sub-Object is present.
When clear, an IP Address Sub-Object is not present.  The Node IP Address
Sub-Object is described in {{IPAddr}} of this memo.

Node Name (bit 6): When set, a Node Name Sub-Object is
included.  When clear, it is not included.  The Node Name Sub-Object is
described in {{Name}} of this memo.

Rsvd2 (bit 7): This bit is reserved for future use
and MUST be set to 0 on transmit and ignored on receipt.

The information included does not self-identify, so this
specification defines a specific ordering for sending the information
that must be followed.

If bit 5 (IP Address) is set, a Node IP Address Sub-Object MUST
be sent first.  If bit 6 (Name) is set, a Node Name Sub-Object
MUST be sent next.  The information order is thus: IP Address Sub-Object,
Node Name Sub-Object.  Any or all pieces of information may be
present or absent, as indicated by the C-Type.  Any data that follows
these optional pieces of information MUST be ignored.

It is valid (though pointless until additional bits are assigned by
IANA) to receive a Node Information Object where bits 5 and 6
are both 0; this MUST NOT generate a warning or error.

## Node IP Address Sub-Object {#IPAddr}

If the Node Identification Object identifies the node by
address, the Object Payload contains an address sufficient
to identify the node within the appropriate scope - global
or as otherwise configured - as depicted in {{addrFig}}.

{::comment}
protocol 'AFI:16,Reserved:16,Address...:32'
{:/comment}
~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|              AFI              |            Reserved           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Address...
~~~~
{: #addrFig title='Node Identification Object - C-Type 2 Payload'}

Payload fields are defined as follows:

* Address Family Identifier (AFI): This 16-bit field identifies
  the type of address represented by the Address field.
  Values for this field represent a subset of values
  found in the IANA registry of Address Family Numbers (available
  from  {{IANA.address-family-numbers}}).  Valid values are 1 (representing a
  32-bit IPv4 address) and 2 (representing a 128-bit IPv6 address).

* Reserved: This field MUST be set to 0 and ignored upon
  receipt.

* Address: This variable-length field represents an address
  of appropriate scope (global, if none other defined) that
  can be used to identify the node.

## Node Name Sub-Object {#Name}

{{nodeFig}} depicts the Node Name Sub-Object:

{::comment}
protocol 'Length:8,Node Name...:24'
{:/comment}
~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Length    |                  Node Name...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #nodeFig title='Node Identification Object Node Name Sub-Object' }

The Node Name Sub-Object MUST have a length that is a multiple
of 4 octets and MUST NOT exceed 64 octets.

The Length field represents the length of the Node Name Sub-
Object, including the length and the node name in octets.  The
maximum valid length is 64 octets.  The length is constrained to
ensure there is space for the start of the original packet and
additional information.

The second field contains the human-readable node name.  The node
name SHOULD be the sys:hostname {{RFC7317}}, if less than 64 octets,
or the first 63 octets of the sys:hostname, if the sys:hostname is
longer.  The node name MAY be some other human-meaningful name of
the node.  The node name MUST be padded with ASCII NUL characters
if the object would not otherwise terminate on a 4-octet boundary.

The node name MUST be represented in the UTF-8 charset {{RFC3629}}
using the Default Language {{RFC2277}}.

# Security Considerations

It may not be desirable to allow this information to be sent to
an arbitrary receiver.  The addition of this information SHOULD
be configurable, and MUST default to off.  An implementation
SHOULD determine what objects may be appended to a given message
based on the destination IP address of the ICMP message that will
contain the objects.

The intended field of use for the extensions defined in this document
is administrative debugging and troubleshooting.  The extensions
herein defined supply additional information in ICMP responses.
These mechanisms are not intended to be used in non-debugging
applications.

This document does not specify an authentication mechanism for the
extension that it defines.  Application developers should be aware
that ICMP messages and their contents are easily spoofed.

# IANA Considerations

This IANA has allocated the ICMP Extension
Object Class value 5 to the extension described above.  The corresponding
Class Sub-types Registry is as follows:

| C-Type (Value) | Description | Reference
| 0-4 | Unallocated - allocatable with Standards Action | \[This document]
| 5 | IP Address Sub-object included | \[This document]
| 6 | Name Sub-object included | \[This document]
| 7 | Unallocated - allocatable with Standards Action | \[This document]

--- back

# Change history

This section is to be removed before publishing as an RFC.

## Changes since draft-fenner-intarea-extended-icmp-hostid-00

- Instead of having two different messages with the same Class Value
  and different CType values, we copy the bitmap implementation
  from [RFC5837].  The re-use of bit positions means that packet
  parsing and generation code can be largely reused from existing
  [RFC5837] code.

## Changes since draft-fenner-intarea-extended-icmp-hostid-01

- Fixed several copy-pasta errors that still referred to
  interface names instead of node name.

# Acknowledgments
{:numbered="false"}

This document derives text heavily from {{RFC5837}}, since the
underlying mechanism is identical, and only the semantics of the
message differs.  Thanks are therefore due to that document's
authors: Alia K. Atlas, Ronald P. Bonica, Carlos Pignataro,
Naiming Shen and JR. Rivers.
