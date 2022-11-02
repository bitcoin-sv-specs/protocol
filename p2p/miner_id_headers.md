# P2P messages sendhdrsen / gethdrsen / hdrsen

## Abstract

Remote peers can use the P2P message _gethdrsen_ to request block headers together with the
contents of the coinbase transaction and any miner-info transaction plus proof of their
inclusion. Headers with the coinbase and miner-info transactions and proof of inclusion
are returned to the requesting peer in P2P message _hdrsen_. Peers can also request to receive
block announcements in form of hdrsen P2P messages. Primary users of this P2P message are
SPV clients which may require simplified access to miner ID provided in miner-info transactions.

## Introduction

The P2P message _gethdrsen_ is an extension of the [_getheaders_](https://wiki.bitcoinsv.io/index.php/Peer-To-Peer_Protocol#getheaders)
message.

Specifically, the message _hdrsen_, which is sent back to the peer from which message _gethdrsen_
was received, contains the same data as _headers_ message with additional fields for the coinbase
transaction contents, any miner-info transaction referenced from the coinbase, and Merkle proof
of inclusion for the coinbase and miner-info transactions. Message _hdrsen_ may also be sent to
announce new blocks if a peer has requested this with message _sendhdrsen_.

All messages defined in this specification are pure additions to the P2P protocol and do not
change the semantics of other messages in any way. Changing the version of P2P protocol is not
required. Nodes are encouraged to implement these messages, but are not required to do so. Peers
sending them must be prepared to handle this situation (i.e. message may be ignored by node and
no response is sent back to the peer).

## Message gethdrsen

If a node receives _gethdrsen_ message from a remote peer, it responds with message _hdrsen_.
The message payload is the same as for message _getheaders_ and contains parameters to limit the
number of headers that will be returned. The semantics of the parameters used to specify which
headers are requested are also the same as for message _getheaders_. See specification
of P2P message [_getheaders_](https://wiki.bitcoinsv.io/index.php/Peer-To-Peer_Protocol#getheaders)
for further details.

## Message sendhdrsen

The _sendhdrsen_ message is defined as an empty message where pchCommand == "sendhdrsen".

Upon receipt of a _sendhdrsen_ message, the node will be permitted, but not required, to announce
new blocks by sending the enriched headers using message _hdrsen_ (defined below) of the new
block (along with any other blocks that a node believes a peer might need in order for the
block to connect).

This message is intended to have similar semantics as message [_sendheaders_](https://wiki.bitcoinsv.io/index.php/Peer-To-Peer_Protocol#sendheaders).

Nodes will not announce new blocks by sending enriched headers if any of the following are true:

* New block is not on node's active chain.
* Peer receiving the announcement does not have the header of parent block (as determined by
node sending the announcement). In this case new blocks will be announced as if _sendhdrsen_
message was never received (i.e. by sending _inv_ message).
* There are more than 8 blocks to announce. In this case new blocks will be announced as if
_sendhdrsen_ message was never received (i.e. by sending _inv_ message).
* Node has received _sendheaders/sendcmpct_ message from peer. If _sendheaders/sendcmpct_ message
is received (either before or after _sendhdrsen_), headers are announced as specified by
_sendheaders/sendcmpct_ message. This exception is needed to avoid changing semantics of
_sendheaders/sendcmpct_ message, which would require changing the protocol version.
* Size of _hdrsen_ message exceeds limit imposed by maxRecvPayloadLength parameter.
In this case new blocks will be announced as if _sendhdrsen_ message was never received (i.e.
by sending _inv_ message). This exception is needed to avoid pushing a large amount of data
to peers that cannot handle it. Unlike when enriched headers are explicitly requested with
_gethdrsen_, it also applies if there is only a single header to announce. Note that if several
headers need to be announced and the size of the corresponding _hdrsen_ message exceeds the size
limit but announcing each header separately would not, this cannot be done because messages
may arrive out of order. If _sendhdrsen_ message is received more than once from the same peer,
it may be treated as a protocol violation.

While nodes may use this message to receive new block announcements via _hdrsen_ message, they
are encouraged not to do so as it provides no benefits (nodes can obtain coinbase and miner-info
transactions plus their Merkle proofs during block validation) and unnecessarily increases
resource usage. Receiving block announcements with enriched headers is primarily intended for
SPV clients that do not validate every block.

## Message hdrsen

The _hdrsen_ message returns enriched block headers requested by a _gethdrsen_ message. It is
also used to announce new blocks to peers that have requested this with _sendhdrsen_ message.

If a peer receives this message without explicitly requesting it first, it can be ignored.
Receiving unrequested _hdrsen_ message should not be treated as protocol violation.

|Field size |Description    |Data Type  |Comments|
|----|----|----|----|
| 1+ | count | var_int | Number of enriched block headers.<br><br>May be 0 if no header matches parameters specified in _gethdrsen_ message.<br><br>No more than 2000 enriched block headers are returned in a single message.<br><br>Since the contents of the coinbase and miner-info transactions can be large, maximum size of _hdrsen_ message is limited to maximum packet size that was agreed on in protoconf message with maxRecvPayloadLength parameter (value is specified in peer configuration). The number of returned enriched block headers is reduced as needed to stay below this limit, but not below 1. This is so that one header requested by _gethdrsen_ message can be returned even if limit imposed by maxRecvPayloadLength parameter is exceeded.<br><br>This limit is always honored if message is sent to announce new blocks (i.e. new blocks will not be announced with this message if the size of the message would exceed the limit). |
| ? | enriched block headers | block_header_en[] | List of enriched block headers (see below). |

### Enriched block header

|Field size |Description    |Data Type  |Comments|
|----|----|----|----|
| 81+ | all block header fields | various | Same fields as in block header returned by [_headers_](https://wiki.bitcoinsv.io/index.php/Peer-To-Peer_Protocol#headers) message.<br><br>Note: Value of field txn_count (transaction count) in block header is typically set to 0 if header is not sent as part of block message (e.g. in _headers_ message). Here the value of this field is set to actual transaction count if that information is available (i.e. if the block was already validated). |
| 1 | no more headers | uint8_t | Boolean value indicating if there are more block headers available after the current header.<br><br>This value only equals true (1) for header of the block that is currently a tip of the active chain as seen by the node that is sending the message. |
| 1 | has coinbase and proof | uint8_t | Boolean value indicating if current block header has additional coinbase data following this field.<br><br>This value may be equal to false if the message is sent as response to message _gethdrsen_ and the node does not have the required data (e.g. if requested block is not yet fully validated, or if it was already pruned). This value is always true if message is sent to announce new blocks. |
| ? | coinbase txn | uint8_t[] | Serialised coinbase transaction. This field is only present if "has coinbase and proof" field was set to true. |
| ? | coinbase merkle proof | uint8_t[] | Merkle proof in binary format according to standard [TS 2020.010-31](https://tsc.bitcoinassociation.net/standards/merkle-proof-standardised-format/). This field is only present if "has coinbase and proof" field was set to true.<br><br>Value of flags field is zero.<br><br>Fields txOrId and target contain an ID of a coinbase transaction and a block hash, respectively.<br><br> Value of index field is zero since the proof is for coinbase transaction which is always the first transaction in a block.<br><br>Value of type field is zero in every node element in nodes field. |
| 1 | has miner-info and proof | uint8_t | Boolean value indicating if current block header has additional miner-info transaction data following this field. |
| ? | miner-info txn | uint8_t[] | Serialised miner-info transaction. This field is only present if "has miner-info and proof" field was set to true. |
| ? | miner-info merkle proof | uint8_t[] | Merkle proof in binary format according to standard [TS 2020.010-31](https://tsc.bitcoinassociation.net/standards/merkle-proof-standardised-format/). This field is only present if "has miner-info and proof" field was set to true.<br><br>Value of flags field is zero.<br><br>Fields txOrId and target contain an ID of a miner-info transaction and a block hash, respectively.<br><br> Value of index field is the position of the miner-info transaction within the block.<br><br>Value of type field is zero in every node element in nodes field. |

