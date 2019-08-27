# P2P protconf Message

## Status

## Abstract
This document proposes a new optional P2P protocol network message that can be used to advertise different protocol 
configuration parameters to remote peers.

## Introduction

As Bitcoin scales towards unlimited block sizes, several network protocol limits such as maximum P2P message size are 
being raised.  The current P2P protocol provides no way of discovering what limits are being used by a remote peer. 
A Bitcoin client is therefore left with several unsatisfactory options:

* Use minimum limits for all remote peers, giving up the benefits of higher limits.
* Use maximum limits for all remote peers risking being disconnected or even banned.
* Maintain internal database of all other implementations, their versions and limits used. This is complex, 
cumbersome and fails if some of the limits are configurable.

To solve this problem a new P2P message `protconf` is introduced. This message can be used to advertise protocol 
configuration parameters to remote peers.

## Definitions

*P2P protocol version* – version of Bitcoin P2P protocol exchanged in version field of the `version` P2P message.

*Message payload* – part of the P2P messages immediately following the message header up to the last byte of the message.

*Compact size encoding* – variable size encoding of integer values that is optimized for small values. See 
`WriteCompactSize` in `serialisation.h` of Bitcoin source code for definition.

*Protocol violation* – A divergence from this or other P2P protocol related specification. Examples: message with 
incorrect checksum, sending other messages before `version` message… When a node detects a protocol violation, it should 
terminate the connection to the remote peer.

### Message protconf Specification 

The protconf message contains different protocol configuration related parameters.

The message payload consists of the following fields:

| Field size | Data type | Name | Description |
|------------|-----------|------|-------------|
| variable   | Compact size unit | `nFields` | Contains number of fields following this field. Used for versioning of protconf message. This specification uses version 1. Version 0 is not allowed. | 
| variable   | Compact size unit | `MAX_PROTOCOL_RECV_PAYLOAD_LENGTH` | See section “Parameter MAX_PROTOCOL_RECV_PAYLOAD_LENGTH” below. |


### Versioning

Field `nFields` is used to indicate the version of protconf messages. This allows for adding new fields in the future 
without changing the P2P protocol version. When a new field is added, the `nFields` is increased by 1. Node 
implementation must therefore allow extra bytes following currently defined fields and ignore any fields it does now 
know about.

The maximum size of `protconf` message payload (without header) is 1.048.576 bytes.

### Rationale

Q: Why use `uint32_t` for maximum P2P message size? We want to scale to gigabytes, shouldn’t `unit64_t` be used instead?

A: Theoretical maximum message size is already limited to `uint32_t` by the field in P2P message header that describes 
payload length. Increasing this further would require changes to message header which would affect all the messages.

Q: Why is versioning implemented per message? Shouldn’t P2P protocol version be increased?

A: It is expected that new fields will be added in near future. Increasing the protocol version is required when 
modifying content or interpretation of an existing message that is not versioned by itself.

Q: Why use a static structure instead of dynamic key-value map with possibly generic data types. This would enable 
clients to parse all current and future configuration values.

A: There is no point parsing unknown keys and values (except for display purposes) if a client does now know how to 
handle them.  If a client wants to take an advantage of newly defined configuration key, client’s code needs to be 
updated anyway. Extending client’s message structure can be performed as part of this change. It is also much more 
straightforward to add a new value that uses a different data type to static structure than trying to encode different 
data types in a dynamic data structure.

### Configurable values

The `protconf` message can only be used to relax P2P protocol restrictions as implemented in the first release of 
Bitcoin SV (version 0.1.0). For example: maximum allowed size of P2P message can only be increased through `protconf` 
message.  The reason for this is to ensure the compatibility with old clients that do not handle `protconf` message.

This specification currently only defines one configuration parameter: `MAX_PROTOCOL_RECV_PAYLOAD_LENGTH`.

### Parameter `MAX_PROTOCOL_RECV_PAYLOAD_LENGTH`

The value of this parameter specifies the maximum size of P2P message payload (without header) that the peer sending the 
`protconf` message is willing to accept (receive) from another peer. The specified limit is used for all P2P messages 
except the following:

* Block related message. Block related messages are the following: `block`, `cmpctblock` 
* The protconf message itself, which is limited to 1.048.576 bytes.

The default value for this parameter is 1.048.576.

The minimum value for this parameter is 1.048.576. Advertising a lower value is treated as a protocol violation.

In addition to increasing maximum P2P message size, the presence of `MAX_PROTOCOL_RECV_PAYLOAD_LENGTH` also changes the 
current limit of `MAX_INV_SZ` (50.000) elements for `getdata` and `inv` messages. The `MAX_INV_SZ` limit is removed and 
the number of elements in those messages is only limited by `MAX_PROTOCOL_RECV_PAYLOAD_LENGTH` advertised by the 
receiving peer.

### Timing and compatibility

The `protconf` is an optional message. It is not required that node either sends or handles the reception of `protconf` 
message. If any of the two connected peers do not support `protconf` message or some of the configuration parameters 
present in the messages, default values will be used. Defaults values are chosen in such a way that they are compatible 
with old clients.

A node that sends the `protconf` message must follow the following rules:

* The `protconf` message must not be sent before sending `verack`. This ensures that the version handshake phase is completed 
before exchanging additional message. 
* The `protconf` message should be sent immediately after the `verack`. This enables remote peer to start using the 
relaxed limits as soon as possible.
* A node can send at most one `protconf` message during the lifetime of the P2P connection. Sending more than one 
`protconf` messages is considered a protocol violation. This means that configuration parameters cannot be changed after 
they were sent to a remote peer. However, if the same peer disconnects and reconnects later, it can send or receive 
different values for the new connection.

### Activation

There are no special rules regarding activation of the `protconf` message. The P2P protocol version remains unchanged. 
It is expected that old clients ignore unknown messages.

### Rationale

Q: Why not include protocol configuration information in `version` message? The `version` message has already been 
extended in the past.

A: The `version` message is used as part of initial handshake and should be kept stable. Some of the existing clients 
do not correctly handle `version` message with additional unknown fields.   The `protconf` message contains additional 
configuration parameters not directly related to the `version` of protocol and most of the existing clients ignore 
unknown messages. It is therefore expected that a new protocol message will cause less disruption than modified 
`version` message and/or increased protocol version.

Q: Why is there no acknowledgment message following the `protconf` message.

A: Because it is not needed. The `protconf` message is designed in such a way that it relaxes existing limits. It is up 
to the peer that receives `protconf` message to decide whether to take advantage of relaxed limits. If `protconf` 
message would tighten the limit then both peers would need to agree on new limit (possibly on per configuration key level).

### Unit tests

*Unit Test*: Node should respect messages up to its advertised `MAX_PROTOCOL_RECV_PAYLOAD_LENGTH`.

*Steps*:
1. Connect to a remote peer and perform handshake sequence (`version`, `verack`).

2. Remote peer must send `protconf` message.
 
2. Calculate how many elements (`maxInvElements`) are accepted by the node based on received 
`MAX_PROTOCOL_RECV_PAYLOAD_LENGTH`. When `MAX_PROTOCOL_RECV_PAYLOAD_LENGTH` is 2 MiB (2*1024*1024), max number of 
elements is 58254.

3. Send an `inv` message with maxInvElements - 1 elements.
4. Remote peer must accept the message.
4. Send an inv message with maxInvElements  elements.
4. Remote peer must accept the message.
5. Send an inv message with maxInvElements+1  elements.	
5. Remote peer should disconnect and ban the client.


*Unit Test*: Node should respect peers that advertise smaller message size.

*Steps*:	
1. Connect to a remote peer and perform handshake sequence (version, versionack).	
1. Handshake successful.
2. Send a protconf message with `MAX_PROTOCOL_RECV_PAYLOAD_LENGTH` set to 1MiB (remote peer default will remain 2MiB).
2. Remote peer must accept the `protconf` message.
3. Send an INV with payload size of 2 MiB (58254 elements).	
3. Remote peer should respond with `getdata` messages, each of them must not exceed  1MiB (without header). The number of elements are 29126, 29126 and 2 respectively.

*Unit Test*: Node should disconnect if it receives two protconf messages.

*Steps:*
1. Connect to a remote peer and perform handshake sequence (`version`, `verack`).
1. Remote peer must send `protconf` message after it send verack message.
2. Send a protconf message.
2. Remote peer must  accept the message.
3. Send another protconf message.
3. Remote peer must disconnect the client without banning it.

*Unit Test*: A `protconf` with payload larger than 1MiB must result in ban	

*Steps*:
1. Connect to a remote peer and perform handshake sequence (`version`, `verack`).
2. Successful handshake. 
2. Send a protconf message with payload of exactly 1MiB.	
2. Remote peer should accept the message. 
3. Connect to a remote peer and perform handshake sequence (version, verack).
3. Successful handshake. 
4. Verify that the remote peer responded with `protconf` message where `MAX_PROTOCOL_RECV_PAYLOAD_LENGTH` is set to 2MiB.
4. Appropriate protconf message should be received.
5. Send a `protconf` message with payload that is exactly 1MiB +1 byte long.
5. Remote peer should disconnect and ban node (despite the fact that it accepts 2MIB messages of other types).

*Unit Test*: Versioning - node should disconnect if it receives invalid numberOfFields.

*Steps*:
1. Connect to a remote peer and perform handshake sequence (`version`, `verack`)
2. Successful handshake. 
2. Send a protconf with `nFields` = 0.
2. Remote peer should disconnect the client without banning it.

*Unit Test*: Node should be able to receive protconf message with a higher version than it recognizes

*Steps*:
1. Connect to a remote peer and perform handshake sequence (`version`, `verack`).
2. Successful handshake.
2. Send a  protconf message with `nFields` = 2 and set `MAX_PROTOCOL_RECV_PAYLOAD_LENGTH`  set to 1 MiB + 36 bytes (1 element), which equals to 29127 elements.
2. Remote peer must  accept the message.
3. Send an INV with payload size 2 MiB (58254 elements)	
3. The remote peer should respect the limit and respond with two getdata messages. First should contain 29127 elements and the second 29127 elements.

*Unit Test*: Old versions of nodes should ignore unknown protconf message. 	

*Steps*:
1. Connect to a remote peer that does not support prtoconf message. Perform handshake sequence (`version`, `verack`).
2. Successful handshake.
2. Send a `protconf` message. 
3. The remote peer should not disconnect once it receives `protconf` message.
