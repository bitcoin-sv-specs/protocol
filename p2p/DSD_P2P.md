# DSD P2P Message

## Abstract

This document describes the new optional DSD P2P protocol network message used to notify nodes of the existence of double-spend transactions in blocks in competing blockchains.

## Introduction

The Bitcoin SV blockchain, like other PoW blockchains, has been subject to Block Withholding attacks. That is, attackers use their hash power to create a hidden alternate chain of blocks parallel to the main chain. The hidden blockchain, which contains double-spends, is then revealed and becomes the main chain.

As part of the response to this situation, an application has been developed to monitor the blockchain and to generate DSD P2P messages to notify nodes of the presence of double-spends in alternate blockchains. The application only needs to notify one connected node which, after validation, relays the DSD P2P message to its peers. 

As part of the validation process, the DSD P2P message confirms that the specified double-spends do indeed exist in the specified blocks (See Merkle proofs).

Note: There is no need to notify peers of every double-spend contained in every block, it is sufficient for a notification to contain the details of a single transaction from each block containing such a conflict.

On receipt of a valid DSD P2P message, the node will use webhooks to notify listener(s) that double-spends in alternate blocks have been detected. Please consult SV Node documentation for further details.
 
## Message Format

The format of the new DSD P2P message is 

|Field size	|Description	|Data Type	|Comments|
|----|----|----|----|
|2	|version	|uint16_t	|Versioning information for this message. |Currently can only contain the value 0x0001|
|1+	|block count	|varint	|The number of blocks containing a double-spend transaction this message is reporting. |Must be >= 2.|
|variable	|block list	|block_details[]	|An array of details for blocks containing double-spend transactions|

block_details:

|Field size	|Description	|Data Type	|Comments|
|----|----|----|----|
|1+	|header count	|varint	|The number of following block headers (may not be 0).|
|variable	|header list	|block_header[]	|An array of unique block headers containing details for all the blocks from the one containing the conflicting transaction back to the last common ancestor of all blocks reported in this message. Note that we don't actually need the last common ancestor in this list, it is sufficient for the last header in this list to be for the block where this fork begins, i.e. the hashPrevBlock field from the last block header will be the same for all the blocks reported in this message.|
|variable	|Merkle proof	|merkle_proof	|The transaction contents and a proof that the transaction exists in this block. Follows TSC standard.|

block_header:

|Field size	|Description	|Data Type	|Comments|
|----|----|----|----|
|4	|version	|int32_t	|Block version information.|
|32	|hash previous block	|char[32]	|Hash of the previous block in the fork.|
|32	|hash merkle root	|char[32]	|The Merkle root for this block.|
|4	|time	|uint32_t	|Timestamp for when this block was created.|
|4	|bits	|uint32_t	|Difficulty target for this block.|
|4	|nonce	|uint32_t	|Nonce used when hashing this block.|

merkle_proof:

|Field size	|Description	|Data Type	|Comments|
|----|----|----|----|
|1	|flags	|char	|Set to 0x05 to indicate we are including the entire transaction the proof is for, the target type is a Merkle root and the proof type is a Merkle branch.|
|1+	|transaction index	|varint	|Transaction index for the transaction this proof is for.|
|1+	|transaction length	|varint	|Size in bytes of the following transaction.|
|variable	|transaction	|char[]	|Binary serialised double-spend transaction.|
|32	|target	|char[32]	|The expected calculated Merkle root. Should be the same as the Merkle root specified in the block header for the block containing the transaction.|
|1+	|node count	|varint	|The number of following hashes from the Merkle branch.|
|variable	|node list	|node[]	|An array of node objects comprising the Merkle branch for this proof. The format of the node object is as described in the standard (link above).|

The Merkle proof contents format should follow the TSC standard described [here](https://tsc.bitcoinassociation.net/standards/merkle-proof-standardised-format/).

## Activation
There are no special rules regarding activation of the P2P DSD message. The P2P protocol version remains unchanged. It is expected that old clients ignore unknown messages.
