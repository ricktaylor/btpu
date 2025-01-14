---

title: "Bundle Transfer Protocol - Unidirectional"
abbrev: "BTPU"
category: std

docname: draft-taylor-btpu-latest
submissiontype: IETF # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3

# area: INT

# workgroup: DTN Working Group

keyword:

- DTN
- BPv7
- Bundle Protocol

venue:

# group: "Delay/Disruption Tolerant Networking"

type: "Working Group"
mail: "dtn@ietf.org"
arch: "https://mailarchive.ietf.org/arch/browse/dtn/"
github: "ricktaylor/btpu"
latest: "https://ricktaylor.github.io/btpu/draft-taylor-btpu.html"

author:

- fullname: Rick Taylor
  organization: Aalyria Technologies
  email: rtaylor@aalyria.com

normative:

informative:

--- abstract

TODO

--- middle

# Introduction

TODO Introduction

# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Terminology

Within the scope of this document, the following terms have specific meaning:

Bundle:
: A higher-layer protocol data unit, typically a BPv7 Bundle as defined in {{?RFC9171}}.

Link-layer PDU:
: The Link-layer Protocol data unit transferred from the Sender to the Receiver by the Link-layer protocol, excluding any Link-layer Protocol specific headers or metadata that makes up a complete transmission unit or frame as defined by the Link-layer Protocol specification.

Datagram:
: A series of one or more Messages, formatted according to the rules of this specification to fit within a single Link-layer PDU.

Message:
: A single protocol data item, see [](#message-definitions).

Transfer:
: The context in which the transmission of the Segments of a single Bundle occurs.

Segment:
: In order to transfer a Bundle larger than a Datagram, Bundles may be subdivided into Segments in order to fit within a Datagram.

# Protocol Applicability

## Assumptions

# Protocol Overview

## Segmentation and Transfers

### Interleaving Segments

### Cancelling Transfers

### Transfer Id Roll-over

## Handling Datagram Loss

# Message Definitions

All Protocol Messages, with the exception of the [Padding Message](#padding-message), follow the common "Type-Length-Value" formatting pattern, with each message being identified by a 32-bit header that encodes the type of the Message, and the length of the content of the message.

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Type  | Length (24-bit unsigned integer)                      |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                       ... Content ...                         :
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Type:
: The type of the message, allocated from IANA "BTPU Message Types" registry, expressed as a 4-bit unsigned integer in network byte order

Length:
: The length of the message in octets, excluding the 4 octets of the header itself, expressed as a 24-bit unsigned integer in network byte order.

Content:
: A sequence of octets of data, encoded according to the type of the Message.

## Complete Bundle Message

The Complete Bundle Message is used to encapsulate an entire bundle, and SHOULD be used by an implementation when a bundle will fit in its entirety in a single Link-Layer PDU to avoid the overhead of processing involved with segmentation, and reducing the risk of the total loss of a bundle if a one or more unnecessary segments of a bundle is lost.

A Complete Bundle Message has a type of seven (7). The Message Content MUST be a valid bundle.

## Transfer Start Message

The Transfer Start Message is used to encapsulate the first segment of a new multi-segment Bundle Transfer.

A Transfer Start Message has a type of five (5). The Message Content is formatted as follows:

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Transfer Id                                                   |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                       ... Content ...                         :
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Transfer Id:
: The numeric identifier of the new Transfer that is starting, encoded as a 32-bit unsigned integer in network byte order.

Content:
: The octets of the first Segment of the Transfer, with the length calculated as the Message content length minus the four (4) octets of the Transfer Id.

The Transfer Start Message does not include an explicit Segment Sequence Number as it is always zero (0).

## Transfer Segment Message

The Transfer Segment Message is used to encapsulate the next segment of an existing multi-segment Bundle Transfer.

A Transfer Segment Message has a type of one (1). The Message Content is formatted as follows:

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Transfer Id                                                   |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Segment Sequence Number                                       |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |               Content ...                                     :
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Transfer Id:
: The numeric identifier of the Transfer that this Segment is part of, encoded as a 32-bit unsigned integer in network byte order.

Segment Sequence Number:
: The sequence number of the Segment, encoded as a 32-bit unsigned integer in network byte order.

Content:
: The octets of a Segment of the Transfer, with the length calculated as the Message content length minus the eight (8) octets of the Transfer Id and Segment Sequence Number.

## Transfer Complete Message

The Transfer Complete Message is used to encapsulate the last segment of a multi-segment Bundle Transfer, and complete the Transfer.

A Transfer Complete Message has a type of three (3). The Message Content is formatted as follows:

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Transfer Id                                                   |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Segment Sequence Number                                       |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |               Content ...                                     :
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Transfer Id:
: The numeric identifier of the Transfer that is completing, encoded as a 32-bit unsigned integer in network byte order.

Segment Sequence Number:
: The sequence number of the final Segment, encoded as a 32-bit unsigned integer in network byte order.

Content:
: The octets of the final Segment of the Transfer, with the length calculated as the Message content length minus the eight (8) octets of the Transfer Id and Segment Sequence Number.

## Transfer Cancel Message

The Transfer Cancel Message is used to cancel an in-progress multi-segment Bundle Transfer.

A Transfer Cancel Message has a type of eleven (11). The Message Content is formatted as follows:

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Transfer Id                                                   |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Transfer Id:
: The numeric identifier of the Transfer that is cancelled, encoded as a 32-bit unsigned integer in network byte order.

The Transfer Cancel Message has no additional content, and hence has a fixed length of 4 octets.

## Padding Message

When the Link-layer Protocol requires Link-layer PDUs of a fixed size, the Padding Message SHOULD be used in order to increase the length of the Datagram to meet the Link-layer PDU size requirement.

A Padding Message has a type of zero (0), but in order to support a Padding Message with a minimum total length of one octet, the Message does not include a Length or Content field, and hence has the following layout:

     0 1 2 3 4 5 6 7
    +-+-+-+-+-+-+-+-+
    | Type  | MBZ   |
    +-+-+-+-+-+-+-+-+

Type:
: The type of the message, in this case zero (0).

MBZ:
: All bits MUST be zero (0).

A Padding Message MUST be the final Message in a Datagram. A sender SHOULD set the value of any remaining octets in the Datagram to zero (0), and a receiver MUST ignore any octets that follow the Padding Message.

When the Link-layer Protocol provides variable length Link-layer PDUs, Sender implementations SHOULD take into account the mechanisms used by the Link-layer Protocol to support variable length Link-layer PDUs, and emit Datagrams of a suitable size for the underlying Link-layer Protocol. For example, if variable length Link-layer PDUs are implemented by the Link-layer Protocol using a sub-framing mechanism, then emitting Datagrams of a single, or whole number of sub-frames may increase reliability. The Padding Message MAY be used to meet such requirements.

# Security Considerations

TODO Security

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments

TODO acknowledge.
