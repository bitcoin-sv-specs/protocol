# P2P Large Message Support

## Abstract

This document describes a change to the P2P networking protocol introduced in
the BSV 1.0.10 node software to increase the maximum size of a P2P message.

## Introduction

Every P2P message on the network has the same basic structure; a 24 byte
header followed by some payload data. One of the fields within the header
describes the length of that payload, and is currently encoded as a
uint32_t. This therefore limits the maximum size of any message payload
to 4GB.

In order to support block sizes greater than 4GB, a change has been made to
the P2P message structure to overcome this limitation.

## Description

### P2P Protocol Versioning

As of the 1.0.10 release the P2P protocol version number has been bumped from
70015 to 70016. Doing this allows a node to know in advance whether a
connected peer will understand the new extended message format and therefore
avoid sending such messages to that peer. Conforming nodes must not send
messages in the extended format to peers with a version number lower than
70016, or they will be banned.

### Message Header Changes

In summary, the changes to the P2P message involve the use of special
values of fields within the existing P2P header as flags that can be
recognised by a peer that supports such changes to indicate that this
is a message with a large payload. These special values also allow a
peer that doesn't understand them to reject such a message and fail cleanly
if it were to come across one.

The existing P2P header contains a 12 byte message type field. We propose
to use a new message type in this field "**extmsg**" (short for extended message)
that when seen will indicate to the receiver that following this message
header are a series of new extended message header fields before the real
payload begins.

The proposed full extended message format is shown below:

| Field_Size | Name | Data_Type | Description |
| --- | --- | --- | --- |
| 4   | magic | uint32\_t | Existing network magic value. Unchanged in this proposal. |
| 12  | command | char\[12\] | Existing network message type identifier (NULL terminated). For new extended message this would take the value **extmsg**. |
| 4   | length | uint32\_t | Existing payload length field. Currently limited to a maximum payload size of 4GB. For new extended messages this will be set to the value **0xFFFFFFFF**. The real payload length will be read from the extended payload length field below. |
| 4   | checksum | uint32\_t | Checksum over the payload. For extended format messages this will be set to **0x00000000** and not checked by receivers. This is due to the long time required to calculate and verify the checksum for very large data sets, and the limited utility of such a checksum. |
| 12  | ext\_command | char\[12\] | The extended message type identifier (NULL terminated). The real contained message type, for example **block** for a > 4GB block, or could also conceivably be **tx** if we decide in future to support > 4GB transactions, or any other message type we need to be large. |
| 8   | ext\_length | uint64\_t | The extended payload length. The real length of the following message payload. |
| ?   | payload | uint8\_t\[\] | The actual message payload. |

