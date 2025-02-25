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
  USLP:
    target: "<https://public.ccsds.org/Pubs/732x1b3e1.pdf>"
    title: Unified Space Data Link Protocol (USLP)
    date: 2024-06
    seriesinfo:
      "CCSDS": "732.1-B-3"
  TM:
    target: "<https://public.ccsds.org/Pubs/132x0b3.pdf>"
    title: Telemetry (TM) Space Data Link Protocol
    date: 2021-10
    seriesinfo:
      "CCSDS": "132.0-B-3"
  AOS:
    target: "<https://public.ccsds.org/Pubs/132x0b3.pdf>"
    title: Advanced Orbiting Systems (AOS) Space Data Link Protocol
    date: 2021-10
    seriesinfo:
      "CCSDS": "732.0-B-4"
  DVB-S2X:
    target: <https://www.etsi.org/deliver/etsi_en/302300_302399/30230702/01.04.01_60/en_30230702v010401p.pdf>
    title: >
      Digital Video Broadcasting (DVB);
      Second generation framing structure, channel coding and
      modulation systems for Broadcasting,
      Interactive Services, News Gathering and
      other broadband satellite applications;
      Part 2: DVB-S2 Extensions (DVB-S2X)
    date: 2024-08
    seriesinfo:
      "ETSI": "EN 302 307-2"
  5G: 3GPP.23.501
  IEEE.802.3: DOI.10.1109/IEEESTD.2022.9844436

--- abstract

TODO - I always do the abstract last ;)

--- middle

# Introduction

Bundle Protocol version 7 (BPv7) is defined in terms a layered logical architecture, detailed in {{!RFC9171}}, wherein the responsibility for the storing and routing of bundles lies with the Bundle Processing Agent (BPA), and the BPA relies upon Convergence Layer Adaptors (CLAs) to provide bundle transport between nodes. CLAs provide a unified interface to the BPA, allowing BPAs to be link-layer agnostic, but still use a diverse range of underlying link-layer protocols to transfer bundles between BPAs.

In the realm of near- and deep-space communication there are a number of standardized link-layer protocols, including {{USLP}}, {{TM}}, {{AOS}}, {{DVB-S2X}}, that share a set of common properties:

- They are unidirectional: data transfer occurs in one direction only, there is no in-band return path for data.
- They are frame-based: the link-layer protocol will guarantee that a frame of data is either delivered to the receiver in its entirety or not at all. Frames may be of fixed or variable length.
- They are point-to-point: although the medium over which the data transfer occurs may be broadcast in nature, the link-layer protocol provides an uncontested point-to-point communication channel between a single sender and a single receiver.

These characteristics provide a common baseline that allows the definition of a lightweight protocol for transferring BPv7 bundles meeting the requirements of a BPv7 CLA, and this document describes such a protocol, Bundle Transfer Protocol - Unidirectional (BTPU), suitable for implementation over any link-layer protocol that shares these characteristics.

Although primarily designed for space communication, the protocol is applicable to other link-layer technologies which can share these characteristics, for example 5G Unstructured PDUs {{5G}}, or {{IEEE.802.3}}, without requiring a full IP stack.

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

The purpose of the protocol is to transfer a series of Bundles between two nodes. Because a Bundle is of variable length, which is unlikely to be exactly the same size as a Link-layer PDU, the protocol defines a mechanism to divide Bundles into Segments as required, such that each Link-layer PDU is efficiently filled with data, and one or more Bundles can be transferred in a minimal number of Link-layer PDUs, described in more detail in [](#transfers).

This segmentation is unrelated to BPv7 bundle fragmentation as defined in {{Section 5.8 of RFC9171}}.  Although BPv7 bundle fragmentation may be used to sub-divide oversized BPv7 bundles, the required duplication of metadata blocks can result in inefficiencies or fail to generate BPv7 bundle fragment small enough to fit in a single Link-layer PDU.

As a sender may prioritize the transfer of each Bundle differently, the protocol allows for the multiplexing of Bundle transfers, so that the transfer of higher priority Bundles may interrupt the transfer of other Bundles, avoiding "head of line blocking", see [](#interleaving-transfers) for more detail.

## Applicability

The protocol has been designed to transfer BPv7 formatted Bundles, however it is equally capable of transferring any kind of binary data, but includes no explicit discriminator of the type of a particular Bundle. If multiple different types of Bundle are to be transferred by a single implementation, this specification considers the differentiation between different Bundle types to be a matter for the implementation. For example, both BPv6 ({{?RFC5050}}) formatted Bundles and BPv7 Bundles can be multiplexed without issue, as the different formats can be distinguished by simple examination of the initial octets of a received Bundle by an implementation.  Additionally, the segmentation mechanism enables the use of this protocol with Bundle formats that do not support some form of fragmentation.

Although designed for any link-layer protocol that shares the characteristics defined in [](#introduction), additional specification or profiling may be required to map the constructs of the link-layer protocol to the mechanisms defined in this specification.

    +----------------------+
    |  DTN Application     |
    +----------------------+
    |  BPv7 / BPv6         |
    +----------------------+
    |  BTPU               |
    +----------------------+
    |  Link-layer Protocol |
    +----------------------+
{: #fig-stack title="The location of BTPU in relation to the Bundle Protocol and a Link-layer protocol" artwork-align="center" }

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

Each Segment is assigned a monotonically increasing integral sequence number, starting at zero (0).  In addition to a sequence number, every Segment is associated with a Transfer that provides context to the sequence of Segments to enable the correct reassembly of the original Bundle. Each Transfer is assigned a number as an identifier, with each identified Transfer mapping to the segmentation of a single Bundle.

The transfer of a sequence of Segments of a Bundle a sender MUST be emitted via a sequence of [Transfer Segment Messages](#transfer-segment-message) carrying the same Transfer identifier.  The end of a sequence of Segments MUST be indicated by emitting a [Transfer End Message](#transfer-end-message), including the final Segment and the identifier of the Transfer that is now complete.

The receiver reassembles the transferred Bundle by concatenating the Segments that share a common Transfer number in the order of their sequence number.  When all the Segments have been received and concatenated, the receiver is assumed to pass the recombined Bundle to an upper layer for further processing.

Transfer numbers are encoded using 32-bit unsigned integers. A sending implementation SHOULD choose a random value between 0 and 2^32-1 for the first Transfer number, and each subsequent Transfer MUST use the next numeric value in the sequence.  To avoid placing a limit on the total number of Transfers between peers, numbers are allowed to "roll-over" via zero and repeat, i.e. the next number in the sequence is the previous number incremented by one, modulo 2^32.

## Interleaving Transfers

In order to support the transmission of Bundles with different priorities, Transfer Messages associated with different Transfers, i.e. with different Transfer numbers, MAY be interleaved.  This allows senders to interrupt the emission of a sequence of Segments associated with one Transfer with one or more Segments of another Transfer, preventing a large lower priority Transfer blocking a higher priority Transfers.

## Cancelling Transfers {#cancelled}

A Transfer may be aborted by the sender while a Transfer is in progress by the emitting of a [Transfer Cancel Message](#transfer-cancel-message) containing the identifier of the Transfer to cancel.  The receiver of a Transfer Cancel Message SHOULD discard any cached segments already received and MUST ignore any further Messages associated with the Transfer.

# Transfer Window {#transfer-window}

Because Messages may be lost in transmission due to the loss of Link-layer PDUs, and a sender may emit duplicate Messages as a defense against loss, see [](#repetition), a sender MUST maintain a sliding Transfer Window that defines the maximum number of Transfers that can be simultaneously in progress.  As Transfers are identified by a monotonically increasing number, the size of the Transfer Window also strictly defines the range of identifiers of Transfers in progress.

The sender MUST maintain a reference to the greatest Transfer number used in any emitted Message, and MUST NOT emit any Message with a Transfer number less than or equal to the latest minus the size of the Transfer Window, taking into account the modulo 2^32 roll-over.

The receiver MUST maintain a reference to the greatest Transfer number received in any Message.  When a Transfer Message is received with a Transfer number greater than the greatest previously received, the new Transfer number is considered the greatest Transfer number, and Transfers with number less than or equal to the latest minus the size of the Transfer Window MUST be considered [cancelled](#cancelled).  Because of Transfer number roll-over, half the number space of 2^32 and window size is used to determine if a number is older or newer than the latest Transfer number.  Pseudocode for the algorithm is given in [](#fig-windowing).

The size of the Transfer Window SHOULD be the same at both receiver and sender, and MUST be configured via some out-of-band mechanism.  The Transfer Window size MUST be at least 4, MUST be less than 2^12, and is RECOMMENDED to be 16. [^1]

    const WINDOW_SIZE  # Configured transfer window size
    var GREATEST = NIL # Greatest received transfer number, initially NIL

    # Function to check if a transfer is valid within the current window
    FUNCTION isTransferValid(T):
        # Ensure Transfer T is within the
        #  sliding window defined by WINDOW_SIZE
        RETURN ((GREATEST - T + 2^32) MOD 2^32) < WINDOW_SIZE

    # Function to check if the transfer is considered a "new" transfer
    FUNCTION isNewTransfer(T):
        IF GREATEST IS NIL THEN
            # The first transfer is always considered new
            RETURN TRUE
        # Check if the transfer is within the valid window range
        #  (half of the number space + window size)
        RETURN ((T - GREATEST + 2^32) MOD 2^32) <
                         (2^32 / 2) + (WINDOW_SIZE / 2)

    # Main function to process a transfer and manage the sliding window
    FUNCTION processTransfer(T):
        IF isNewTransfer(T) THEN
            # New transfer, update the greatest received transfer number
            GREATEST ← T
            # Cancel transfers that are now outside the window
            CANCEL_OUTDATED_TRANSFERS()
        ELSE IF isTransferValid(T) THEN
            # Transfer is in progress, continue handling it
            CONTINUE_PROCESSING(T)
        ELSE
            # Transfer is invalid (outside the window), ignore it
            IGNORE_MESSAGE(T)
{: #fig-windowing title="The receiver's algorithm for determining Transfer number validity and sliding window" artwork-type="pseudocode"  }

[^1]: These are entirely arbitrary numbers, and need discussing by the WG.

# Handling Link-layer PDU Loss {#repetition}

Due to the unreliable nature of the link-layer protocol, Link-layer PDUs may be lost in transmission, resulting in the loss of the contained Messages.  Because the underlying link-layer is assumed to be unidirectional, the protocol does not include a mechanism to trigger the retransmission of lost Messages; instead the protocol allows the sender to repeat the transmission of Messages.

A sender MAY emit any Message multiple times in different Link-layer PDUs.  Although every Link-layer PDU transmitted may contain different Messages, any repeated Message MUST be an exact copy of an already emitted Message.  When segmenting bundles, not all Messages in a Transfer need be repeated the same number of times, and different Transfers may repeat Messages differently.

Although it is RECOMMENDED that [Transfer Segment Messages](#transfer-segment-message) are emitted in ascending order of sequence number; once emitted, any Message MAY be repeated any number of times, in any order.  The number of repetitions of a particular Message is an implementation matter that can be influenced by many factors, including:

- Offline analysis of the deployed environment may require a certain amount of Message repetition to reach some required certainty of transfer.
- A higher 'reliability' factor associated with a particular Bundle may result in more copies of each associated Transfer Message being emitted.
- Signalling from the link-layer protocol, or some other out-of-band mechanism, may trigger increased repetition of a subset Messages, to protect against some temporary spike in Link-layer PDU loss rate.

The provided protection mechanisms are logically separate from any facilities the underlying link-layer protocol may have to protect against information loss through redundancy and erasure coding, and may be used as required by a deployment.  If a link-layer protocol receives a duplicate transmission frame, it SHOULD be delivered to this protocol only once.

# Message Definitions

All protocol Messages except the [Indefinite Padding Message](#indefinite-padding-message) follow the common "Type-Length-Value" formatting pattern, with each Message being identified by a four octet header that encodes the type of the Message, and the length of the content of the Message.

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Type          | Length (24-bit unsigned integer)              |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                       ... Content ...                         :
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Type:
: The type of the Message, allocated from IANA "BTPU Message Types" registry, see [](#iana-considerations), expressed as a 8-bit unsigned integer in network byte order.

Length:
: The length of the Message in octets, excluding the 4 octets of the header itself, expressed as a 24-bit unsigned integer in network byte order.

Content:
: A sequence of octets of data of variable length determined by the corresponding Length field value, encoded according to the type of the Message.

## Bundle Message

The Bundle Message is used to encapsulate an entire Bundle, and SHOULD be used by an implementation when a Bundle will fit in its entirety in a single Link-layer PDU to avoid the overhead of segmentation, and reducing the risk of the total loss of a Bundle if one or more unnecessary segments of a Bundle is lost.

A Bundle Message has a type of 2. The Message Content MUST be a valid Bundle.

Emitting a Bundle Message with a Length field value of zero (0), i.e no Bundle content, only adds control-plane overhead and SHOULD NOT be used as an alternative form of padding.

## Transfer Segment Message

The Transfer Segment Message is used to encapsulate a segment of a multi-segment Bundle Transfer.

A Transfer Segment Message has a type of 3. The Message Content field is formatted as follows:

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
: The numeric identifier of the Transfer that this Segment is part of, encoded as a 32-bit unsigned integer in network byte order.

Segment Sequence Number:
: The sequence number of the Segment, encoded as a 32-bit unsigned integer in network byte order.

Segment Data:
: The octets of a Segment of the Transfer, with the length calculated as the Message content length excluding the eight (8) octets of the Transfer Number and Segment Sequence Number.

Transfer Segment Messages SHOULD NOT have zero octets of Segment Data, i.e. the total length of the Message SHOULD be greater than 12 octets.  Such Messages only add control-plane overhead and SHOULD NOT be used as an alternative form of padding.

## Transfer End Message

The Transfer End Message is used to encapsulate the last segment of a multi-segment Bundle Transfer, and complete the Transfer.

A Transfer End Message has a type of 4. The Message Content field is formatted as follows:

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
: The numeric identifier of the [in-progress](#transfer-window) Transfer that is completing, encoded as a 32-bit unsigned integer in network byte order.

Segment Sequence Number:
: The non-zero sequence number of the final Segment, encoded as a 32-bit unsigned integer in network byte order.

Segment Data:
: The octets of the final Segment of the Transfer, with the length calculated as the Message content length excluding the eight (8) octets of the Transfer Number and Segment Sequence Number.

Transfer End Messages SHOULD NOT have zero octets of Segment Data, i.e. the total length of the Message SHOULD be greater than 12 octets.  Such Messages only add control-plane overhead and SHOULD NOT be used as an alternative form of padding.

## Transfer Cancel Message

The Transfer Cancel Message is used to indicate that the indicated Transfer is being aborted, and any prior or later received Segments associated with the Transfer MUST be discarded by the receiver.

A Transfer Cancel Message has a type of 5. The Message Content field is formatted as follows:

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Transfer Number                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Transfer Number:
: The numeric identifier of the [in-progress](#transfer-window) Transfer that is cancelled, encoded as a 32-bit unsigned integer in network byte order.

The Transfer Cancel Message has no content, and hence has a fixed length of 4 octets.

A peer that receives a Transfer Cancel Message with a Transfer Number field value that does not match the numeric identifier of an [in-progress](#transfer-window) Transfer MUST ignore the Message.

## Definite Padding Message

The Definite Padding Message is used to add padding to a Link-layer PDU.

A Definite Padding Message has a type of 1. Any content it contains has no semantic meaning, and a sender SHOULD set the content to a sequence of zero (0) octets.  A receiver MUST ignore any Message content.

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

IANA is requested to create a new registry entitled "BTPU Message Types".  The registration policy for this registry, using terms defined in {{!RFC8126}}, is:

| Values | Registration Policy |
|:-----: |:------ |
| 0..0x6F | Standards Action |
| 0x70..0x7F | Private Use |
| 0x80..0xFF | Reserved for future expansion |
{: #tab-message-types-reg align="left" title="BTPU Message Types registration policies"}

The initial values for the registry are:

| Type | Message | Reference |
|:------:|:-----|:------|
| 0 | [Indefinite Padding Message](#indefinite-padding-message)  | This document |
| 1 | [Definite Padding Message](#definite-padding-message)  | This document |
| 2 | [Bundle Message](#bundle-message) | This document |
| 3 | [Transfer Segment Message](#transfer-segment-message) | This document |
| 4 | [Transfer End Message](#transfer-end-message) | This document |
| 5 | [Transfer Cancel Message](#transfer-cancel-message) | This document |
{: #tab-message-types-vals align="left" title="BTPU Message Types initial values"}

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

An example of the transmission of three Bundles of varying sizes and different priority in three Link-layer PDUs is shown in [](#fig-interleaved).

            +---------------------------+
            | Bundle B                  |  High Priority
            +---------------------------+
    +--------------+-----------------+
    | Bundle A     | Bundle C        |     Low Priority
    +--------------+-----------------+


    +----------------------+------------+----+----+------------+---------+
    | T1    | Transfer 2   | Transfer 2 | T1 | T3 | Transfer 3 | Padding |
    | S0    | Segment 0    | Segment 1  | S1 | S0 | Segment 1  |         |
    +----------------------+------------+----+----+------------+---------+

    :                      :                      :                      :

    +----------------------+----------------------+----------------------+
    | Link-layer PDU N     | Link-layer PDU N + 1 | Link-layer PDU N + 2 |
    +----------------------+----------------------+----------------------+
{: #fig-interleaved title="Interleaved segmentation of a sequence of Bundles of different priority" }

TODO: Description.

## Message repetition

DIAGRAM!

# Acknowledgments

EK, Brian Sipos, TCPCL authors

Chloe

TODO acknowledge.
