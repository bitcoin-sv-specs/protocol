# P2P Multi-Streams

## Abstract
This document describes an enhancement to the P2P networking layer that allows
nodes to create additional connections to their peers and designate those
connections for particular categories of traffic.

## Introduction
When communicating with a peer over a network all messages to that peer that
are waiting to be sent have to be queued. Similarly, all messages from that
peer that are awaiting processing also have to be queued. If all we have is
a single communications channel between peers, then we run the risk that
messages we deem to be of higher priority may end up stuck in a queue behind
lower priority messages.

For example; consider the situation where we have a queue of several large
transaction messages we are streaming, and then a new block arrives. We would
like to be able to notify our peer of the new block as soon as possible,
without having to wait for the large transactions to finish being sent.

Note that it is not enough simply to use prioritised message queues to solve
this problem because we cannot easily interrupt a message that is midway
through the process of being transmitted in order to send a higher priority
one and then switch back to sending the first message again.

The solution in this case is to allow nodes to create multiple connections
(**streams**) to their peers, and to designate those streams for the transmission
of particular categories of traffic. Since each stream has both a dedicated
message queue and TCP socket, it both allows prioritisation within the
application message queues and avoids TCP's head-of-line blocking.

## Description
As stated above; the general idea is that instead of a single socket per node
we are connected to we may have multiple sockets each corresponding to a
stream of communications to that peer. The aim is to model something similar
to SCTP's “*association*” concept with multiple streams contained within an
association.

The first stream established is always designated a control/general message
channel, and it will be possible to send any message type over it. This means
that it is entirely optional whether to open further streams beyond the first
one and legacy nodes can continue to operate by just sending all their data
over a single socket as before.

New nodes may however then open further streams to their peers and designate
those new channels just for the exchange of particular data messages. It is
currently possible to create up to four additional streams per peer, the exact
number of which is decided by an active **stream policy**.

The stream policy determines not only the number of additional streams created
but how those streams are used, thus allowing the node to categorise and
prioritise the P2P traffic and hopefully allow more efficient use of the
resources available.

Currently there are only two stream policies implemented and available for
use; the `Default` and `BlockPriority` policies. The Default policy creates
no additional streams beyond the first and just sends all traffic over that
initial general stream, thus acting just like a traditional P2P connection.
The BlockPriority policy creates one additional stream which it uses to
transfer high priority block and connection control messages, thus prioritising
them over other high volume data such as transactions.

Additional stream policies may be implemented in future if such a need arises.

### Connection Establishment
To support the concept of an association between nodes that may contain
multiple streams a new field has been added to the existing `version` message
and two entirely new P2P messages have been created.

#### Version Message Extension
The version message has been extended to include a new `association ID` field
after the existing `relay` field. The full new version message format is shown
below:

| Field Size | Name | Data Type | Description |
|------------|------|-----------|-------------|
| 4 | version | int32_t | Identifies protocol version being used by the node |
| 8 | services | uint64_t | Bitfield of features to be enabled for this connection |
| 8 | timestamp | int64_t | Standard UNIX timestamp in seconds |
| 26 | addr_recv | net_addr | The network address of the node receiving this message |
| 26 | addr_from | net_addr | The network address of the node sending this message |
| 8 | nonce | uint64_t | Random nonce used to detect connections to self |
| 1+ | user_agent | var_str | User agent string |
| 4 | start_height | int32_t | The last block received by the sending node |
| 1 | relay | bool | Whether the remote peer should announce relayed transactions (BIP 0037) |
| 1+ | association_id | var_bytes | ID to use to identify this new association |

The association ID is an opaque value which can be used to identify multiple
connections for streams within the same association. If a version message is
received containing an association ID on an incoming peer connection, then the
corresponding version message sent back will echo the same association ID. The
receipt of a matching association ID on an incoming version message can
therefore be used as confirmation that the node we connected to understands how
to handle associations and we can proceed to setup further streams.

#### New CreateStream Message
A new `createstream` P2P message has been defined. A createstream message will
only be accepted for peers that have already connected to us with a version
message that specifies a new association ID, and will be sent as the first
message on a new connection instead of the version message if that connection
is desired to setup a new stream within an existing association.

The format of the createstream message is shown below:

| Field Size | Name | Data Type | Description |
|------------|------|-----------|-------------|
| 1+ | association_id | var_bytes | ID of the existing association this new stream belongs to |
| 1 | stream_type | byte | Enumeration to identify the type of this new stream (1 - 4) |
| 0+ | stream_policy | var_str | (Optional) Name of the stream policy to use on this association |

*Note 1:* The `stream_type` field will be used by the currently active stream
policy to determine which of the active streams to a peer should be used for
sending them messages of a particular type. It is also expected (but not
required) that they would use the same appropriate stream type for sending
messages to us.

*Note 2:* It is expected that the `stream_policy` field within the createstream
message will only need to be sent in a single createstream message if
multiple such messages are required to setup all streams within an association.

On successful receipt and processing of a createstream message a new
`streamack` message is returned to the sender. Until a streamack has been
received the sender should not assume that the new stream has been instantiated
by the other peer. If there was a problem processing the createstream message
then a standard `reject` response will be issued and the connection closed.

#### New StreamAck Message
The format of the new `streamack` message is shown below:

| Field Size | Name | Data Type | Description |
|------------|------|-----------|-------------|
| 1+ | association_id | var_bytes | ID of the association the stream we are ack'ing belongs to |
| 1 | stream_type | byte | The type of the stream we are ack'ing |

The streamack message simply echos back the association ID and stream type from
the createstream message.

### Protoconf Message Extension
A new field has also been added to the `protoconf` message. This field is
labelled `streamPolicies` and contains the names of all the stream policies
the sending node supports as a comma separated list. This field can be used
to help pick a stream policy both peers support that should be used on an
association.

For more details about the protoconf message see [here](./protoconf.md).
