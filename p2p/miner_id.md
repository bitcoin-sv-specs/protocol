# Miner ID Related P2P Changes

## Abstract

This document describes the P2P messaging changes that have been made to help support some
features of miner ID.

## Introduction

In the current design, miners identify themselves by including some data in the malleable input
fields of the coinbase transaction whenever they mine a block. However, this data is not always
accurate and can be forged.

The miner ID change provides a way of cryptographically identifying miners. A miner ID is a public
key of an ECDSA keypair. It is used to sign a coinbase document which is included as part of a
transaction within the block, instead of providing unsigned arbitrary data. It should be noted
that miner ID is a voluntary extra service that miners can offer and is in no way mandatory. For
more details about miner ID, please see the specification published
[here](https://github.com/bitcoin-sv-specs/brfc-minerid).

Two features within the miner ID specification require extra support from the P2P layer, those
features being dataref transaction retrieval and miner ID revocation. These changes are described
here.

## Dataref transaction retrieval

Miner ID documents can include references to other so called dataref transactions, the contents
of which should be incorporated into the miner ID document to create the complete document.

SPV clients who wish to recreate a miner ID document containing such datarefs will need a way
to fetch those dataref transactions, which may have been mined in an earlier block than
the block from which they are referenced.

### P2P getdata message extension

SPV clients will become aware that they need dataref transactions to fully reconstruct a miner ID
document when they parse a miner-info transaction (as referenced by the block header and the
associated coinbase transaction) that contains those datarefs. The node that sent the SPV client
the block header and the miner-info transaction must have also validated that block and so will
have seen any dataref transactions contained in that block, plus any dataref transactions also
contained in earlier blocks, and so will have those dataref transactions stored in its dataref
index. The SPV client can therefore direct its requests for dataref transactions to the same node
that sent it the block header.

### Inventory vector new type

In order for SPV clients to request dataref transactions we specify here an extension to the
getdata P2P message. We will introduce a new inventory type for getdata called **MSG_DATAREF_TX**
with a value of **0x05** to indicate to the receiving node that the sender is requesting a dataref
transaction. The items in the inventory vector within the getdata message are then in the standard
format. I.e:

|Field size|Description|Field type|Comments|
|---|---|---|---|
| 4 | type | uint32_t | The type of the object being requested. In this case the value **0x05**. |
| 32 | hash | char[32] | The hash of the dataref transaction requested. |

If the requested dataref transaction(s) could not be found then a standard P2P **notfound** reply
should be returned. Otherwise, a series of **datareftx** P2P messages (format shown below) should
be sent back to the requester, one for each of the dataref transactions specified in the getdata
request.

### P2P datareftx message format

Dataref transactions are returned in a new P2P **datareftx** message, the format of which is as
follows:

|Field size|Description|Field type|Comments|
|----|----|----|----|
| variable | txn | char[] | The serialised dataref transaction in the standard transaction format as for the P2P **tx** message described [here](https://en.bitcoin.it/wiki/Protocol_documentation#tx). |
| variable | merkle proof | merkle_proof | A proof that the above dataref transaction is included in a block (see below for format). |

#### The merkle_proof

The merkle proof contents format should follow the TSC standard described
[here](https://tsc.bitcoinassociation.net/standards/merkle-proof-standardised-format/):

|Field size|Description|Field type|Comments|
|----|----|----|----|
| 1   | flags | char | Set to **0x00** to indicate this proof contains just the transaction ID, the target type is a block hash, and the proof type is a merkle branch. |
| 1+  | transaction index | varint | Transaction index for the transaction this proof is for. |
| 32  | txid | char\[32\] | Txid for the dataref transaction this proof is for. |
| 32  | target | char\[32\] | Hash of the block containing the dataref transaction. |
| 1+  | node count | varint | The number of following hashes from the merkle branch. |
| variable | node list | node\[\] | An array of node objects comprising the merkle branch for this proof. The format of the node object is as described in the standard (link above). |

## Miner ID revocation

A miner ID that a miner fears may have been compromised can be revoked and rotated in the next
block that miner mines. They may not be able to mine a block for some indeterminate amount of
time however, and so a P2P revokemid message has been defined to allow an early notification of
a revoked miner ID to be broadcast to the network.

### The P2P revokemid message

The format of the revokemid message is as follows:

|Field size |Description    |Data Type  |Comments|
|----|----|----|----|
| 4 | version | uint32_t | Identifies the miner ID protocol version. Currently always 0. |
| 33 | revocationKey | uint8_t[33] | The current public revocation key for the miner in compressed form as a 33 byte hex string. |
| 33 | minerID | uint8_t[33] | The current miner ID for the miner in compressed form as a 33 byte hex string. |
| 33 | revocationMessage | uint8_t[33] | The revocation message to be signed by the miner and identifying the ID (and all later IDs) to be revoked. |
| 142-148 | revocationMessageSig | uint8_t[] | This contains the 2 signatures from the miner required to certify the revocation. The format is detailed below. |

#### Format of the revocationMessageSig field

This field contains two signatures required to certify the miner ID revocation. Both signatures
made over the SHA256 hash of the revocationMessage field. The first signature is made
using the miners private revocation key (the corresponding private key to the public key
contained in the revocationKey field). The second signature is made using the miners private
miner ID key (the corresponding private key to the public key contained in the minerID field).

The two signatures, sig1 and sig2, are then encoded as 70-73 byte hex strings and concatenated
along with a single byte each indicating their length. Ie:

```
sig1 = HexStr(Sign(SHA256(revocationMessage), private revocationKey key))
sig2 = HExStr(Sign(SHA256(revocationMessage), private minerID key))

revocationMessageSig = concat(sig1.length, sig1, sig2.length, sig2)
```

## Miner ID block header retrieval

Peers (such as SPV clients) may wish to be able to retrieve sufficient details about blocks to
reconstruct the ID of the miner that created them without having to fetch the blocks in their
entirety. To support this, some enhancements have been made to the P2P block header retrieval
mechanisms. These changes are described [here](./miner_id_headers.md).

