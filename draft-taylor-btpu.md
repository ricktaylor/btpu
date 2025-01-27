---

title: "Bundle Transfer Protocol - Unidirectional"
abbrev: "BTP-U"
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
mail: "<dtn@ietf.org>"
arch: "<https://mailarchive.ietf.org/arch/browse/dtn/>"
github: "ricktaylor/btpu"
latest: "<https://ricktaylor.github.io/btpu/draft-taylor-btpu.html>"

author:

- fullname: Rick Taylor
  organization: Aalyria Technologies
  email: <rtaylor@aalyria.com>

normative:

informative:

--- abstract

TODO - I always do the abstract last ;)

--- middle

# Introduction

Bundle Protocol version 7 (BPv7) is defined in terms a layered logical architecture, detailed in {{!RFC9171}}, wherein the responsibility for the storing and routing of bundles lies with the Bundle Processing Agent (BPA), and the BPA relies upon Convergence Layer Adaptors (CLAs) to provide bundle transport between nodes. CLAs provide a unified interface to the BPA, allowing BPAs to be link-layer agnostic, but still use a diverse range of underlying link-layer protocols to transfer bundles between BPAs.

In the realm of near- and deep-space communication there are a number of standard link-layer protocols, including USLP, TM, AOS, DVB-S2(X), that share a set of common properties:

- They are unidirectional. Data transfer occurs in one direction only, there is no in-band return path for data.
- They are frame-based. The link-layer protocol will guarantee that a frame of data is either delivered to the receiver in its entirety or not at all. Frames may be of fixed or variable length.
- They are point-to-point. Although the medium over which the data transfer occurs may be broadcast in nature, the link-layer protocol provides an uncontested point-to-point communication channel between a single sender and a single receiver.

These characteristics provide a common baseline that allows the definition of a lightweight protocol for transferring BPv7 bundles meeting the requirements of a BPv7 CLA, and this document describes a protocol, Bundle Transfer Protocol - Unidirectional (BTP-U), suitable for implementation over any link-layer protocol that shares these characteristics.

TODO: More here about not requiring a full IP stack.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Terminology

Within the scope of this document, the following terms have specific meaning:

Bundle:
: A higher-layer protocol data unit, typically a BPv7 Bundle as defined in {{?RFC9171}}.

Link-layer PDU:
: The protocol data unit, excluding any link-layer protocol specific headers or metadata, that makes up a complete transmission unit or frame, as defined by the link-layer protocol specification.

Message:
: A single protocol data item, see [](#message-definitions).

Transfer:
: The context in which the transmission of the Segments of a single Bundle occurs, see [](#transfers).

Segment:
: In order to transfer a Bundle larger than a Link-layer PDU, Bundles may be subdivided into Segments in order to fit within a Link-layer PDU, see [](#transfers).

# Protocol Overview

The purpose of the protocol is to transfer a series of Bundles between two nodes. Because a Bundle is of variable length, which is likely to not be exactly the same size as a Link-layer PDU, the protocol defines a mechanism to divide Bundles into Segments as required, such that each Link-layer PDU is efficiently filled with data, and one or more Bundles can be transferred in a minimal number of Link-layer PDUs, described in more detail in [](#transfers).

This segmentation is unrelated to BPv7 bundle fragmentation as defined in {{Section 5.8 of RFC9171}}.  Although BPv7 bundle fragmentation may be used to sub-divide oversized BPv7 bundles, the required duplication of metadata blocks can result in inefficiencies or fail to generate BPv7 bundle fragment small enough to fit in a single Link-layer PDU.

As a sender may prioritize the transfer of each Bundle differently, the protocol allows for the multiplexing of Bundle transfers, so that the transfer of higher priority Bundles may interrupt the transfer of other Bundles, avoiding "head of line blocking", see [](#interleaving-transfers) for more detail.

## Applicability

The protocol has been designed to transfer BPv7 formatted Bundles, however it is equally capable of transferring any kind of binary data, but includes no explicit discriminator of the type of a particular Bundle. If multiple different types of Bundle are to be transferred by a single implementation, this specification considers the differentiation between different Bundle types to be a matter for the implementation. For example, both BPv6 ({{?RFC5050}}) formatted Bundles and BPv7 Bundles can be multiplexed without issue, as the different formats can be distinguished by simple examination of the initial octets of a received Bundle by an implementation.  Additionally, the segmentation mechanism enables the use of this protocol with Bundle formats that do not support some form of fragmentation.

Although designed for any link-layer protocol that shares the characteristics defined in [](#introduction), additional specification may be required to map the constructs of the link-layer protocol to the mechanisms defined in this specification.

    +----------------------+
    |  DTN Application     |
    +----------------------+
    |  BPv7 / BPv6         |
    +----------------------+
    |  BTP-U               |
    +----------------------+
    |  Link-layer Protocol |
    +----------------------+
{: #fig-stack title="The location of BTP-U in relation to the Bundle Protocol and a Link-layer protocol" artwork-align="center" }

## Messages

The basic primitive of the protocol is the Message, a self-describing unit of protocol control information of variable length. The sender is responsible for composing one or more Messages as required, and packing them into a Link-layer PDU, such that a single PDU is optimally filled.  The receiving node parses the contained Messages from each received Link-layer PDU, and then processes them as individual control signals.  This sequence of Messages from sender to receiver is the logical control-plane used by the protocol.  This document uses the verb "emit" to describe to the writing of a new Message to a Link-layer PDU ready for transmission, to differentiate from the the transmission of the Link-layer PDU itself, as many Messages may emitted prior to the transmission of the containing PDU.

See [](#message-definitions) for detail of each type of Message.

## Padding

Because the size of a Bundle is not expected to exactly match the size of a Link-layer PDU, an implementation will likely need to add padding to the PDU so that the Link-layer PDU size requirements are met.  Two Messages are available for this purpose: The [Definite Padding Message](#definite-padding-message) and the [Indefinite Padding Message](#indefinite-padding-message).  Padding Messages are valid at any point within a Link-layer PDU.

It is RECOMMENDED that implementations use the Definite Padding Message to add padding to a Link-layer PDU, except when less than four octets of padding are required, as the minimum length of the Definite Padding Message is four octets.

When the link-layer protocol provides variable length Link-layer PDUs, implementations SHOULD take into account the mechanisms used by the link-layer protocol to support variable length Link-layer PDUs, and emit Link-layer PDUs of a suitable size for the underlying protocol. For example, if variable length Link-layer PDUs are implemented by the link-layer protocol using a sub-framing mechanism, then emitting Link-layer PDUs of a single, or whole number of sub-frames may increase reliability.

The algorithm used to pad and pack Messages efficiently into Link-layer PDUs is an implementation matter.

# Segmentation and Transfers {#transfers}

As described in the [Protocol Overview](#protocol-overview), in order to transfer Bundles larger than a single Link-layer PDU into multiple PDUs, Bundles are be divided into a sequence of Segments by the sender and each Segment is emitted in its own a Message. However, if a complete Bundle can fit in the next Link-layer PDU, then the Bundle SHOULD be transferred without segmentation, see the [Bundle Message](#bundle-message).

Each Segment is assigned a monotonically increasing integral sequence number, starting at zero (0).  In addition to a sequence number, every Segment has an associated Transfer that provides context to the sequence of Segments to enable the correct reassembly of the original Bundle. Each Transfer is assigned a monotonically increasing number as an identifier, with each identified Transfer mapping to the segmentation of a single Bundle.

The receiver reassembles the transferred Bundle by concatenating the Segments in the order of their sequence number.  When all the Segments have been received and concatenated, the receiver passes the recombined Bundle to the upper layer for further processing.

Transfer numbers are encoded using 32-bit unsigned integers, and to avoid placing a limit on the total number of Bundles that may be transferred between peers, numbers are allowed to "roll-over" to zero: the next number in the sequence is the previous number incremented by one, modulo 2^32.

## Interleaving Transfers

TODO: Transfer Messages associated with different Transfers, i.e with different Transfer Number field values, MAY be interleaved.

## Transfer Lifetime {#transfer-liveness}

The transfer of a sequence of Segments of a Bundle a sender MUST begin by emitting a [Transfer Start Message](#transfer-start-message), carrying the Transfer identifier and the first Segment, which indicates a new Transfer is starting.  Further Segments, excluding the final Segment, MUST be emitted via the [Transfer Segment Message](#transfer-segment-message), carrying the same Transfer identifier.  The end of a sequence of Segments MUST be indicated by emitting a [Transfer End Message](#transfer-end-message), including the final Segment and the identifier of the Transfer that is now complete.  A sender considers a Transfer to be "active" between emitting the first Transfer Start Message, and emitting the first Transfer End Message referencing the Transfer.

TODO: Because Messages may be in transmission lost due to the loss of Link-layer PDUs, and a sender may emit duplicate Message as a defense against loss, see [](#handling-loss), a receiver has a looser perspective on the liveness of a Transfer.

## Cancelling Transfers

A Transfer may be aborted by the sender while a Transfer is active by the emitting of a [Transfer Cancel Message](#transfer-cancel-message) containing the identifier of the Transfer to cancel.

# Handling Link-layer PDU Loss {#handling-loss}

Due to the unreliable nature of the link-layer protocol, Link-layer PDUs may be unexpectedly lost in transmission, resulting in the loss of the contained Messages.  Because the underlying link-layer is assumed to be unidirectional, the protocol does not include a mechanism to trigger the retransmission of lost Messages; instead the protocol allows the sender to repeat the transmission of Bundle segments.

The repetition of Segments is logically separate from any mechanism the underlying link-layer protocol may have to repeat the transmission of frames, although both aim to protect against information loss through redundancy and may be used as required by a deployment.  If a link-layer protocol receives a duplicate transmission frame, it SHOULD be delivered to this protocol only once.

TODO: The loss and repetition of Messages results in Transfer id being all over the place!

## Forward Error Correction

We could reference {{?RFC6363}}, and include a Transfer FEC Start and Segment Repair Message.[^1]

[^1]: This is a question for the WG

# Message Definitions

All protocol Messages except the [Indefinite Padding Message](#indefinite-padding-message) follow the common "Type-Length-Value" formatting pattern, with each Message being identified by a 32-bit header that encodes the type of the Message, and the length of the content of the Message.

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Type          | Length (24-bit unsigned integer)              |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                       ... Content ...                         :
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Type:
: The type of the Message, allocated from IANA "BTP-U Message Types" registry, expressed as a 8-bit unsigned integer in network byte order.

Length:
: The length of the Message in octets, excluding the 4 octets of the header itself, expressed as a 24-bit unsigned integer in network byte order.

Content:
: A sequence of octets of data of variable length determined by the corresponding Length field value, encoded according to the type of the Message.

## Bundle Message

The Bundle Message is used to encapsulate an entire Bundle, and SHOULD be used by an implementation when a Bundle will fit in its entirety in a single Link-layer PDU to avoid the overhead of segmentation, and reducing the risk of the total loss of a Bundle if one or more unnecessary segments of a Bundle is lost.

A Bundle Message has a type of TBD. The Message Content MUST be a valid Bundle.

Emitting a Bundle Message with a Length field value of zero (0), i.e no Bundle content, only adds control-plane overhead and SHOULD NOT be used as an alternative form of padding.

## Transfer Start Message

The Transfer Start Message is used to encapsulate the first segment of a new multi-segment Bundle Transfer.

A Transfer Start Message has a type of TBD. The Message Content field is formatted as follows:

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Transfer Number                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Total Transfer Length                                         :
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    :                         ... Total Transfer Length (continued) |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                    ... Segment Data ...                       :
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Transfer Number:
: The numeric identifier of the new Transfer that is starting, encoded as a 32-bit unsigned integer in network byte order.

Total Transfer Length:
: The total length of the reassembled Bundle, encoded as a 64-bit unsigned integer in network byte order.

Segment Data:
: The octets of the first Segment of the Transfer, with the length calculated as the Message content length excluding the four (4) octets of the Transfer Number.

So that a receiving implementation may preallocate the buffers required to reassemble the segmented Bundle, the Total Transfer Length field contains the total length of the Bundle to be reassembled.

To reduce signalling overhead, the Transfer Start Message does not include a Segment Sequence Number field as the sequence number is implicitly zero (0).

The Transfer Number field value MUST NOT match the numeric identifier of a currently [active](#transfer-liveness) Transfer.

Transfer Start Messages SHOULD NOT have zero octets of Segment Data, i.e. the total length of the Message SHOULD be greater than 16 octets.  The Total Transfer Length field value SHOULD NOT be zero, as a zero-length Bundle is not a valid Transfer.  Such Messages only add control-plane overhead and SHOULD NOT be used as an alternative form of padding.

## Transfer Segment Message

The Transfer Segment Message is used to encapsulate the next segment of an active multi-segment Bundle Transfer.

A Transfer Segment Message has a type of TBD. The Message Content field is formatted as follows:

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Transfer Number                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Segment Sequence Number                                       |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                    ... Segment Data ...                       :
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Transfer Number:
: The numeric identifier of the [active](#transfer-liveness) Transfer that this Segment is part of, encoded as a 32-bit unsigned integer in network byte order.

Segment Sequence Number:
: The non-zero sequence number of the Segment, encoded as a 32-bit unsigned integer in network byte order.

Segment Data:
: The octets of a Segment of the Transfer, with the length calculated as the Message content length excluding the eight (8) octets of the Transfer Number and Segment Sequence Number.

Transfer Segment Messages SHOULD NOT have zero octets of Segment Data, i.e. the total length of the Message SHOULD be greater than 12 octets.  Such Messages only add control-plane overhead and SHOULD NOT be used as an alternative form of padding.

## Transfer End Message

The Transfer End Message is used to encapsulate the last segment of a multi-segment Bundle Transfer, and complete the Transfer.

A Transfer End Message has a type of TBD. The Message Content field is formatted as follows:

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Transfer Number                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Segment Sequence Number                                       |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                    ... Segment Data ...                       :
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Transfer Number:
: The numeric identifier of the [active](#transfer-liveness) Transfer that is completing, encoded as a 32-bit unsigned integer in network byte order.

Segment Sequence Number:
: The sequence number of the final Segment, encoded as a 32-bit unsigned integer in network byte order.

Segment Data:
: The octets of the final Segment of the Transfer, with the length calculated as the Message content length excluding the eight (8) octets of the Transfer Number and Segment Sequence Number.

Transfer End Messages SHOULD NOT have zero octets of Segment Data, i.e. the total length of the Message SHOULD be greater than 12 octets.  Such Messages only add control-plane overhead and SHOULD NOT be used as an alternative form of padding.

## Transfer Cancel Message

The Transfer Cancel Message is used to indicate that the indicated Transfer is being aborted, and any prior or later received Segments associated with the Transfer MUST be discarded by the receiver.

A Transfer Cancel Message has a type of TBD. The Message Content field is formatted as follows:

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Transfer Number                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Transfer Number:
: The numeric identifier of the [active](#transfer-liveness) Transfer that is cancelled, encoded as a 32-bit unsigned integer in network byte order.

The Transfer Cancel Message has no additional content, and hence has a fixed length of 4 octets.

A peer that receives a Transfer Cancel Message with a Transfer Number field value that does not match the numeric identifier of an [active](#transfer-liveness) Transfer MUST ignore the Message.

## Definite Padding Message

The Definite Padding Message is used to add padding to a Link-layer PDU.

A Definite Padding Message has a type of TBD. Any content it contains has no semantic meaning, and a sender SHOULD set the content to a sequence of zero (0) octets.  A receiver MUST ignore any Message content.

It is valid for this Message to have no content, i.e. a Length field value of zero (0), adding a total of four (4) octets of padding to the Link-layer PDU.

## Indefinite Padding Message

An Indefinite Padding Message has a type of zero (0), and in order to support padding with a minimum total length of one octet, the Message does not include an explicit Length or Content field, and hence has the following layout:

     0 1 2 3 4 5 6 7
    +-+-+-+-+-+-+-+-+
    | Type          |
    +-+-+-+-+-+-+-+-+

Type:
: The type of the Message: zero (0).

The Indefinite Padding Message type field is followed by a sequence of zero or more zero (0) octets, ending at the first non-zero octet, or the end of the fixed-length Link-layer PDU.  The content of the Message has no meaning, and MUST be ignored by a receiver.

Note: When a Indefinite Padding Message terminates with a non-zero octet, the non-zero octet is the first octet of the subsequent Message.

# Security Considerations

This protocol does not define any measures to protect Messages or their content.  Although there may be link-layer mechanisms to protect the transmission of frames against over-hearing and interference, transport-layer security is considered out of scope for the protocol. Mechanisms such as BPSec, defined in {{!RFC9172}}, MUST be used to provide integrity and protection at the Bundle layer as required.

# Deployment Considerations

The following caveats should be considered before deploying instances of this protocol:

1. It is unreliable. Although there may be a link-layer protocol mechanism for a receiver to be notified that a frame has been lost in transmission, due to the unidirectional nature of the link-layer there is no in-band return path suitable for higher-layer acknowledgement of transfer success.  Any acknowledgement system designed to provide reliability MUST use a logically separate path from receiver back to sender.

1. It does not provide congestion control or signalling. The underlying link-layer is expected to provide an uncontested point-to-point channel, and hence such mechanisms are considered to be out of scope. The protocol MUST NOT be deployed in environments where congestion may occur, such as the public Internet, when the underlying link-layer, or a higher layer, does not provide suitable congestion control.

1. It requires an out-of-band mechanism for configuration. This can either be via pre-placed static configuration, a parallel dynamic control-plane protocol, or some other mechanism beyond the scope of this specification.

# IANA Considerations

TODO Need a new registry!

TODO Reserve some codepoints for CLs

--- back

# Examples

## Segmentation of a sequence of Bundles of equal priority

An example of the transmission of three Bundles of varying sizes and equal priority in three Link-layer PDUs is shown in [](#fig-sequential).

    +---------------------------+------------+-----------------+
    | Bundle A                  | Bundle B   | Bundle C        |
    +---------------------------+------------+-----------------+

    :                           :            :                 :

    +----------------------+----+------------+----+------------+---------+
    | Transfer 1           | T1 |  Complete  | T2 | Transfer 2 | Padding |
    | Segment 0            | S1 |  Bundle    | S0 | Segment 1  |         |
    +----------------------+----+------------+----+------------+---------+

    :                      :                      :                      :

    +----------------------+----------------------+----------------------+
    | Link-layer PDU N     | Link-layer PDU N + 1 | Link-layer PDU N + 2 |
    +----------------------+----------------------+----------------------+
{: #fig-sequential title="Segmentation of a sequence of Bundles of equal priority" }

Bundle A is transferred as two Segments, included in the first and second Link-layer PDU, as Transfer 1.  Bundle B fits completely in the second Link-layer PDU, and is therefore transferred without segmentation.  Bundle C is transferred as two Segments split between the second and third PDU, but padding is required to fill the third PDU.  An alternative algorithm could have selected to not segment Bundle C, but to pad the second PDU and include Bundle C without segmentation in the third PDU, without changing the semantics, as an implementation preference.

## Segmentation of a sequence of Bundles of different priority

DIAGRAM!

## Message repetition

DIAGRAM!

# Acknowledgments

EK, Brian Sipos, TCPCL authors

TODO acknowledge.
