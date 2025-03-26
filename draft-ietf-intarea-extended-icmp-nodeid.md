---
title: "Adding Extensions to ICMP Errors for Originating Node Identification"
abbrev: "ICMP Node ID"
category: std

docname: draft-ietf-intarea-extended-icmp-nodeid-latest
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
  latest: "https://fenner.github.io/icmp-node-id/draft-ietf-intarea-extended-icmp-nodeid.html"

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
  RFC0792:
  RFC3629:
  RFC2277:
  RFC4443:
  RFC4884:
  RFC5837:
  RFC7317:
  RFC7915:
  IANA.address-family-numbers:

informative:
  RFC6877:
  I-D.chroboczek-intarea-v4-via-v6:
  I-D.equinox-v6ops-icmpext-xlat-v6only-source:


--- abstract

RFC5837 describes a mechanism for Extending ICMP for Interface and Next-Hop Identification,
which allows providing additional information in an ICMP error that helps identify
interfaces participating in the path.  This is especially useful in environments
where a given interface may not have a unique IP address to respond to, e.g., a traceroute.

This document introduces a similar ICMP extension for Node Identification.
It allows providing a unique IP address and/or a textual name for the node, in
the case where each node may not have a unique IP address (e.g., a deployment
in which all interfaces have IPv6 addresses and all nexthops are IPv6 nexthops,
even for IPv4 routes).

--- middle

# Introduction

In addition to adding incoming interface information to a traceroute
using the mechanisms described in {{RFC5837}}, a network operator
may be interested in adding information to unambiguously identify nodes themselves.
For example, {{I-D.chroboczek-intarea-v4-via-v6}} describes a scenario in which individual
nodes do not have unique IPv4 addresses to use to reply to an IPv4
traceroute, so additional information is needed.
Another scenario is described in {{I-D.equinox-v6ops-icmpext-xlat-v6only-source}}:
when an IPv6-only node runs the customer-side translator (CLAT, {{RFC6877}}),
traceroute to an IPv4 destination can not represent intermediate IPv6-only routers.

This document defines an ICMP extension that fills that void.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

ICMPv4 is used to refer to Internet Control Message Protocol (ICMP) specified in {{RFC0792}}.

ICMPv6 is used to refer to Internet Control Message Protocol (ICMPv6)
for the Internet Protocol Version 6 (IPv6) specified in {{RFC4443}}.

ICMP is used to refer to both ICMPv4 and ICMPv6.


# Node Identification Object {#nodeid}

This section defines the Node Identification Object, an ICMP Extension
Object with a Class-Num (Object Class Value) of 5 (see {{sec-iana}}).

Similar to {{Section 4 of RFC5837}}, this object can be appended
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
considerations do not supersede this requirement.  See {{security}} for
suggested configuration regarding including these messages.

Similarly to the Interface Identification Object defined in {{RFC5837}},
there are two different pieces of information that can appear in a
Node Identification Object:

1. An IP Address Sub-Object MAY be included, containing an address
   of sufficient scope to identify the node within the domain.
   The IP Address Sub-Object is defined in {{IPAddr}} of this memo.

2. A Name Sub-Object MAY be included, as specified in {{Name}},
   containing up to 63 octets of the YANG sys:hostname ({{RFC7317}})
   or another appropriate name uniquely identifying the node.

## C-Type Meaning in a Node Identification Object

The C-Type contains a bitmask describing what information is included
in this Node Identification Object ({{ctypeFig}}).  The fields in this
bitmask are chosen so that the IPAddr and name bits overlap
with the same bits as defined in {{RFC5837}}, so that an implementation
that supports exactly these bits can reuse packet generation and parsing code.

~~~~
Bit     0       1       2       3       4       5       6       7
    +-------+-------+-------+-------+-------+-------+-------+-------+
    |               Unassigned              | IPAddr|  Name |  Un2  |
    +-------+-------+-------+-------+-------+-------+-------+-------+
~~~~
{: #ctypeFig title='C-Type for the Node Identification Object'}

The following are bit-field definitions for C-Type:

- Unassigned (bits 0-4): These bits are reserved for future use
    and MUST be set to 0 on transmit and ignored on receipt.

- IP Addr (bit 5) : When set, an IP Address Sub-Object is present.
    When clear, an IP Address Sub-Object is not present.  The IP Address
    Sub-Object is described in {{IPAddr}} of this memo.

- Name (bit 6): When set, a Name Sub-Object is
    included.  When clear, it is not included.  The Name Sub-Object is
    described in {{Name}} of this memo.

- Un2 (bit 7): This bit is reserved for future use
    and MUST be set to 0 on transmit and ignored on receipt.

The information included does not self-identify, so this
specification defines a specific ordering for sending the information
that must be followed.

If bit 5 (IP Address) is set, an IP Address Sub-Object MUST
be sent first.  If bit 6 (Name) is set, a Name Sub-Object
MUST be sent next.  The information order is thus: IP Address Sub-Object,
Name Sub-Object.  Any or all pieces of information may be
present or absent, as indicated by the C-Type.  Any data that follows
these optional pieces of information MUST be ignored.

It is valid (though pointless until additional bits are assigned by
IANA) to receive a Node Identification Object where bits 5 and 6
are both 0; this MUST NOT generate a warning or error.

### Behavior when additional bits are reserved {#fooblewomp}

Bit values SHOULD be assigned from left to right in the diagram
above, i.e., starting at zero.  The sub-objects associated with each
new bit MUST be placed in the packet after the sub-objects defined
in this memo.  For example, if bit 0 is assigned to the Fooblewomp,
a packet with bits 0 and 5 set MUST contain the IP Address
Sub-Object, followed by the Fooblewomp sub-object.

If a bit is set that a receiver does not support, followed by
a bit that the receiver does support, the receiver MUST ignore
all of the additional data, since the length of the unsupported
data is unknown.

## IP Address Sub-Object {#IPAddr}

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
{: #addrFig title='IP Address Sub-Object'}

Payload fields are defined as follows:

* Address Family Identifier (AFI): This 16-bit field identifies
  the type of address represented by the Address field.
  Values for this field represent a subset of values
  found in the IANA registry of Address Family Numbers (available
  from  {{IANA.address-family-numbers}}).  Valid values are as
  follows:

  + 1: 32-bit IPv4 address
  + 2: 128-bit IPv6 address.

* Reserved: This field MUST be set to 0 and ignored upon
  receipt.

* Address: This variable-length field represents an address
  of appropriate scope (global, if none other defined) that
  can be used to identify the node.  The length of this field
  is derived from the AFI (i.e., 32 bits if the AFI field is set to 1,
  and 128 bits if the AFI is set to 2).

## Name Sub-Object {#Name}

{{nodeFig}} depicts the Name Sub-Object:

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
{: #nodeFig title='Name Sub-Object' }

The Name Sub-Object MUST have a length that is a multiple
of 4 octets and MUST NOT exceed 64 octets. If the length field
exceeds 64 octets, the receiver MUST ignore the sub-object.

The Length field represents the length of the Name Sub-Object,
including the length and the node name, in octets.  The
maximum valid length is 64 octets.  The length is constrained to
ensure there is space for the start of the original packet and
additional information.

The second field contains the human-readable node name.  The node
name SHOULD be the YANG sys:hostname {{RFC7317}}, if less than 64 octets,
or the first 63 octets of the sys:hostname, if the sys:hostname is
longer.  The node name MAY be some other human-meaningful name of
the node.  The node name MUST be padded with ASCII NUL characters
if the object would not otherwise terminate on a 4-octet boundary.

The node name MUST be represented in the UTF-8 charset {{RFC3629}}
using the Default Language {{RFC2277}}.

# Addition of Node Identification Object by Intermediate Nodes {#nat}

An IP/ICMP translator MAY use this extension when translating
an ICMP message listed above to include the
pre-translation source address of a packet. When doing so, it MUST
include the IP Address Sub-Object.
If an ICMP Extension Structure is already present
in the packet being translated, this Extension Object is appended to
the existing ICMP Extension Structure and the checksum is updated.
If an ICMP Extension Structure is not present in the packet being
translated, one is added using the rules of {{RFC4884}}.
Further details of this mode of operation are outside the
scope of this memo.

# Security Considerations {#security}

A node name may reveal sensitive information, especially when it
encodes semantic information.
It may not be desirable to allow this information to be sent to
an arbitrary receiver.  The addition of this information to
the ICMP responses listed in {{nodeid}} is configurable, and
defaults to off, with the exception of IP/ICMP translators {{RFC7915}}.
Those translators SHOULD add the Node Identification Extension Object
with the IP Address Sub-Object, as described in {{I-D.equinox-v6ops-icmpext-xlat-v6only-source}}.
An implementation
may determine what objects may be appended to a given message
based on the destination IP address of the ICMP message that will
contain the objects.  Access control lists (ACLs) may be used to
filter the destinations to which this information may be communicated.

This document does not specify an authentication mechanism for the
extension that it defines.  Application developers should be aware
that ICMP messages and their contents are easily spoofed.

# IANA Considerations {#sec-iana}

This IANA has allocated the ICMP Extension
Object Class value 5 to the extension described above.  The corresponding
Class Sub-types Registry is as follows:

| C-Type (Value) | Description | Reference
| 0-4 | Unassigned - allocatable with Standards Action | \[This document]
| 5 | IP Address Sub-object included | \[This document]
| 6 | Name Sub-object included | \[This document]
| 7 | Unassigned - allocatable with Standards Action | \[This document]

As mentioned in {{fooblewomp}}, IANA is requested to assign additional
bits starting at zero.

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

## Changes since draft-fenner-intarea-extended-icmp-hostid-02

- Renamed to draft-ietf-intarea-extended-icmp-nodeid-00 to reflect
  adoption by WG

## Changes since draft-ietf-intarea-extended-icmp-nodeid-00

- Several edits suggested by Med Boucadair

- Added {{nat}} to address the needs of XLAT

- Changed title to "Adding Extensions to ICMP Errors for
  Originating Node Identification"

## Changes since draft-ietf-intarea-extended-icmp-nodeid-01

- Added the request to assign bits starting at 0 to the IANA
  Considerations

- Updated IANA Considerations wording based on RFC8126

- Shortened sub-object names to "IP Address" and "Name", eliminating
  "Node".

# Acknowledgments
{:numbered="false"}

This document derives text heavily from {{RFC5837}}, since the
underlying mechanism is identical, and only the semantics of the
message differs.  Thanks are therefore due to that document's
authors: Alia K. Atlas, Ronald P. Bonica, Carlos Pignataro,
Naiming Shen, and JR. Rivers.

Further thanks are due to the following who have provided
valuable contributions to this document: Med Boucadair,
Jen Linkova, and David Lamparter.
