# Confiscation Transactions
This document provides an overview of confiscation transactions in BSV and BTC.

## Overview 
Confiscation transactions move funds from frozen UTXOs to a court designated
address. Since valid unlocking scripts for the frozen UTXOs are not available,
a mechanism is introduced which allows script verification to be skipped:

- Use new OP_RETURN protocol for Confiscation
- Contain a reference to identify the related ConfiscationOrder document

Unlocking scripts for confiscated inputs are not available, so script
verification will fail. Instead, we don’t perform script verification for
confiscation transactions.

- The transaction must be whitelisted to skip script verification
- Fees are paid from confiscated funds
    - Every input must be present on the confiscation order
    - Skipping script verification applies to every input of the transaction

### Segwit
The confiscation transaction is serialized as a normal (non-Segwit)
transaction. If the PX chain has Segregated Witness (Segwit) enabled, the
confiscation transaction would have to be processed as a Segwit transaction if
there are any Segwit inputs to be confiscated. By skipping script verification,
any Segwit specific witness validation logic can be skipped.

### Unspendable Outputs
Confiscation transactions can confiscate any UTXO that exists in the Node’s
UTXO set. Funds at provably unspendable outputs e.g. OP_FALSE OP_RETURN which
are not tracked by the Node’s UTXO set, cannot be confiscated. A malicious
attacker could steal funds and send them to an output like this and the funds
could not be recovered. 
Conversely, funds sent to unspendable outputs (not provable, but generally
considered to be unspendable) which do exist in the UTXO set, can be
confiscated (salvaged).  

## OP_RETURN protocol
### Introduction
A confiscation transaction is identified by the first transaction output being
an OP_RETURN output. The output follows the confiscation transaction protocol,
embedding the RIPEMD-160 of the confiscation order hash. This helps to create
an audit trail for observers of why funds moved.

Note: The Court order document is not embedded in the OP_RETURN data of the
confiscation transaction because not all blockchains support a large payload.
For example, the default data carrier size on BTC is 80 bytes, which means most
nodes will classify the transaction with large payload as non-standard, and
subsequently not accept the transaction in its mempool. So the key design
criteria for this protocol are:

- Compatibility with BTC and BSV
- Transaction is treated as standard

### Confiscation Protocol 
The OP_RETURN output uses a pushdata for the protocol id, and another pushdata
for the confiscation related data:

`[OP_FALSE] OP_RETURN OP_PUSHDATA <PROTOCOLID> OP_PUSHDATA <CONFISCATIONMESSAGE>`

Where CONFISCATIONMESSAGE is a binary encoded blob of data:

<VERSION || ORDERHASH || LOCATIONHINT> 

Where OP_FALSE is only required on BSV (see below).

### Can we use OP_FALSE with OP_RETURN?
BTC and BSV have diverged in how they determine if the OP_RETURN output script is standard script or not.

- With BTC an output script beginning with an OP_RETURN opcode is treated as a
  standard script. If it begins with OP_FALSE, the script is not standard.
- Since the BSV Genesis upgrade, the output must begin OP_FALSE OP_RETURN to be
  treated as standard script and to be provably unspendable. 
          
This means OP_FALSE OP_RETURN can only be used with BSV. On the BTC network,
this output would be considered non-standard and the entire transaction will be
rejected as being non-standard.

### How much data can we store? 
BTC has a default value for MAX_OP_RETURN_RELAY of 83 bytes. This is the
maximum size of the OP_RETURN script output.

- OP_RETURN (1 byte) + PUSHDATA1 (1 byte) + Length 0x50 (1 byte) + Payload (80 bytes) 

A BTC node can be configured to have a custom data carrier size limit. 

### We don’t use data framing for every field 
Note that OP_RETURN protocols on BSV, by convention, use OP_PUSHDATA for data
framing e.g the protocol ID and the payload are separate elements.  We could
use data framing with every field of the confiscation protocol, since BSV can
support large scripts, but we only frame the protocol ID and the payload
(Confiscation Message). This helps keep OP_RETURN script output <= 83 bytes in
size for compatibility with BTC. 

### Protocol Data Fields
Here are the fields in more detail:

Opcode/ Field Name | Size        | Description 
---                | ---         | ---
OP_FALSE           |1 byte (BSV) | 0x00 NOT FOR BTC. REQUIRED ONLY FOR BSV.
OP_RETURN          |1 byte       | 0x6a 
OP_PUSHDATA        |1 byte       | 0x04 (length of protocol ID) 
PROTOCOLID         |4 bytes      | Protocol ID. ASCII string: ‘cftx’ 
OP_PUSHDATA        |1 byte       | Length 1 byte Length of protocol specific payload. For BSV, upto 74 (0x4f) 
VERSION            |1 byte       | Protocol version number. 0x01 for version 1 
ORDERHASH          |20 bytes     | 160-bit hash value, RIPEMD160(ConfiscationOrderHash) where ConfiscationOrderHash is 256-bit hash value. 
LOCATIONHINT       |Varies       | For BTC, upto 54 bytes For BSV, upto 53 bytes (as 1 byte is required for OP_FALSE)

### How is the LOCATIONHINT used? 
The LOCATIONHINT size is limited to ensure the number of script bytes after
OP_RETURN is <= 80 bytes in total. The output script size must be <= 83 bytes
in total to maintain compatibility with BTC. The LOCATIONHINT is any
**partial data** that helps an observer to **reconstruct a URL** to locate and
download the ConfiscationOrder. By convention this might be the ASCII string
of the S3 bucket where the related court documents are - so the observer can
reconstruct the filename using the ORDERHASH e.g.

    S3Bucket/ORDERHASH_TypeOfDocument.json 

This field i.e. bucket, may become stale over time, but the ORDERHASH will
always be valid, so an observer should be able to find the document from the
NotaryTool (NT) operator, archival service or Google search. For example, an
observer would construct a download link for a CourtOrder document as follows:

The observer finds in the OP_RETURN:

- ORDERHASH is the RIPEMD160(ConfiscationOrderHash)
    - This is 160-bit hash value as raw data, taking up 20 bytes
- LOCATIONHINT contains ASCII string “s3://usa-nt-bucket/newyork"
    - Storage service identifier
    - Bucket name
    - Folder names

Using the order hash, the ConfiscationEnvelope containing the order document
has been uploaded to the S3 URL:

- s3://usa-nt-bucket/newyork/ORDERHASH 

So the observer can download the document over HTTP using the following URL:

- http://usa-nt-bucket.s3.amazonaws.com/newyork/ORDERHASH 

Remark: The node must validate the PROTOCOLID and VERSION are correct, and that
the OP_RETURN data payload is <= 80 bytes in total (and the OP_RETURN output
script size is <= 83 bytes in total). The node does not need to do anything
with the ORDERHASH and LOCATIONHINT, those fields are for interested third
parties.

## Whitelisting Confiscation Transactions 
The NT creates confiscation transactions. The BlacklistManager (BM) is
responsible for whitelisting the confiscation transactions in the Node:

- The NT is responsible for checking court orders in the NT system and
  identifying any transaction output conflict before letting a user create a
  confiscation order. The NT verifies:
    - Freeze Order outputs are still consensus frozen
        - If an Unfreeze Order is activated, outputs are no longer frozen
        - Freeze Order outputs are still unspent
            - If already spent, they cannot be confiscated
        - Freeze Order outputs do not appear on an existing Confiscation Order
            - If already being confiscated we don’t want to create a
              double-spend transaction
            - The only exception is if the Confiscation Order has been archived, in
              which case outputs can be made available for selection (see
              section on Archiving for why this is useful e.g. confiscation
              order was never activated)
- The BM is responsible for validating confiscating transactions and
  whitelisting confiscation txids, using new node RPC:
  addToConfiscationTxidWhitelist(), once the confiscation order is activated.
- The BM is responsible for submitting whitelisted confiscation transactions to
  the Node, which accepts them when block height >= EnforceAtHeight, using node
  RPC sendrawtransaction().
        - The BM must track current height
        - The BM can submit confiscation transactions at EnforceAtHeight-1, as
          the node will accept the confiscation transaction for mining into the
          next block which is EnforceAtHeight (the node will reject a block at
          EnforceAtHeight-1 containing the confiscation transaction)
- The Node is not responsible for determining if confiscation transactions are
  genuine or not. Like other RPC calls, the Node trusts the caller invoking the
  whitelist RPC.
- The Node is only responsible for validating the confiscation transaction
  e.g. valid OP_RETURN, double-spends, etc.

The BM can update a node’s whitelist by using a new RPC:
addToConfiscationTxidWhitelist() 

Later, when the BM submits the transaction to the node for mining, using
sendrawtransaction, or the transaction is received from a remote peer through
P2P, the node will perform validation as described above. 

The BM does not remove confiscation transactions from the whitelist, and the
node does not have an RPC to remove them. This is because once an UTXO moves
from state ConsensusFrozen to ConsensusConfiscation, the UTXO cannot be
unfrozen. The UTXO will remain in the ConsensusConfiscation state until it is
confiscated to a new address. 

If it is necessary to reset the whitelist, the BM can do so when calling RPC
clearBlacklists with parameter removeAllEntries:true, which clears the
blacklist and whitelists. 

If the court needs to transfer funds back to the owner, the court can order
that some of the funds at the new address are returned.

## Network Behaviour 
Only nodes connected to a BM will be able to support Confiscation transactions,
because they must be explicitly whitelisted by a BM. **The introduction of
Confiscation transactions requires nodes on the network to be connected to a BM
to avoid a chain split.** 

### Node behaviour with and without BM 
The following table shows what happens when regular non-confiscation and
confiscation transactions try to move frozen funds. 

Non-PX: Pre Phase 1
Phase 1: Freezing only
Phase 2: Freezing plus confiscation

| **Location of TX Moving Frozen Funds** | **Non-PX Node** | **Phase 1 Node Without BM** | **Phase 1 Node With BM** | **Phase 2 Node Without BM** | **Phase 2 Node With BM** |
| ---                                | ---         | ---                     | ---                  | ---                     | ---                  |
| Mempool: Regular Tx (inputs are consensus frozen) | Accept transaction into mempool | Accept transaction into mempool | Reject transaction as funds are known to be frozen | Accept transaction into mempool | Reject transaction as funds are frozen | 
| Block: Regular Tx (inputs are consensus frozen) | Accept block (until orphaned by PX miners) | Accept block (until orphaned by PX miners) | Reject block as funds are known to be frozen | Accept block (until orphaned by PX miners) | Reject block as funds are known to be frozen |
| Mempool: Confiscation Tx (inputs are consensus confiscated) | Reject transaction as confiscation transactions will be considered invalid | Reject transaction as confiscation transactions will be considered invalid | Reject transaction as confiscation transactions will be considered invalid | Reject transaction as there is no BM to whitelist transaction | Accept transaction only if BM has whitelisted transaction |
| Block: Confiscation Tx (inputs are consensus confiscated) | Reject block as confiscation transactions will be considered invalid | Reject block as confiscation transactions will be considered invalid | Reject block as confiscation transactions will be considered invalid | Reject block as confiscation transaction was not whitelisted by BM | Accept block only if BM has whitelisted transaction

### Mining nodes must run BM
Whitelisting confiscation transactions means that nodes must be connected to a
BM for a consistent view of the whitelist, in order to avoid a chain split.
Also, to be legally compliant, the miner may already be required to run a BM. 

### Listener nodes can run without BM 
However, this could be a burden for non-miners. Is is possible for a node to
remain on the network without a BM and not fall victim to fake confiscation
transactions. Listener nodes can be configured to run without a BM and stay on the
network, with assurance that confiscation transactions in blocks are genuine: 

Set new configuration option –assumeWhitelistedBlockDepth=0, and the node will:

- Have an empty blacklist and whitelist, since there is no BM updating the list
- Reject all confiscation transactions received via P2P or submitted directly
  to node, since whitelist is empty
- Accept blocks containing confiscation transactions
    - Depth 0 means confiscation transactions older than 0 blocks from tip
      are assumed to be whitelisted
    - Majority hashpower of honest miners will whitelist confiscation
      transactions and only build on top of valid blocks which don’t
      contain non-whitelisted confiscation transactions 

Also, the new ConfiscationMaturity consensus rule will help provide assurance
to listener nodes :

- A rogue miner trying to defraud an exchange by spending funds from fake
  confiscation transaction, would have to build a chain longer than
  ConfiscationMaturity blocks.
- Possible attack is that a rogue miner might include fake confiscation
  transaction which confiscates legitimate funds, to prevent owner from
  spending those funds, but this attack would be costly to maintain for
  ConfiscationMaturity blocks 

If the listener node thinks the above is not sufficient, they should run the BM.

## Syncing the Whitelist 
A chain split can occur if mining nodes have different whitelists, so it’s
important for mining nodes to have a consistent view of globally whitelisted
transactions. 

A mining node will mark a valid block as invalid, if it contains
a confiscation transaction which has not been whitelisted.  The node operator
will then have to manually reconsider the block. 

### Examples of when we need to sync 
When a node is launched for the first time, it will not have a whitelist. 

If a node is being restarted, any local whitelist database is stale. 

If a node has lost connectivity, when connectivity is restored, the whitelist
may be stale.

The node cannot reorg because a block on the target fork has been invalidated,
due to a stale whitelist. 

### BM periodically updates the whitelist 
The BM keeps track of the blacklist already sent to a node, and periodically
tries to update the blacklist for all connected nodes.

The BM will maintain information on which nodes have been contacted and had
their whitelist updated. In the event that there are connectivity issues, the
BM will keep retrying until it successfully reaches the node. 

### Help a node recover and sync the whitelist 
To help a node recover to the correct state of the whitelist there is a manual
recovery procedure by BM operator:

- Call Node RPC getchaintips to check if node is on the same fork
- Call Node RPC reconsiderblock if node has seen correct chain tip but marked as invalid

Note that the BM is unaware of what a Node operator may have done. For example,
the Node operator may have used RPC invalidateblock on a blockhash belonging to
the BM’s best chain. 

Note that the BSV node software has introduced RPCs which are not available in
BTC, so they should not be relied upon.

- soft block orphaning: conditionallyinvalidateblock, reconsiderconditionallyinvalidatedblock
- safe mode: ignoresafemodeforblock, reconsidersafemodeforblock

## Mempool and Relay
When a node receives a confiscation transaction, it will only be accepted into the
mempool and subsequently relayed, if:

1. Txid has been whitelisted 
2. node block height H + 1 >= EnforceAtHeight 
    a. At block EnforceAtHeight-1 the node can accept a confiscation
    transaction which is valid for mining into EnforceAtHeight
3. Passes other validation checks 

Once accepted into the mempool, the confiscation transaction could expire. The
BM should ensure that the confiscation transaction is resubmitted. 

## Block containing Confiscation transactions 
When a node receives a block containing a confiscation transaction, the node should only
accept the block if: 

1. Txid has been whitelisted  
2. block height >= EnforceAtHeight

Note that the node’s ConsensusFreeze blacklist is used to reject a block that
contains a transaction spending a blacklisted (frozen) output. Currently, this
blacklist is not cleaned up, so the node must prioritize the whitelist to
override the consensus blacklist, to ensure a valid block containing
confiscation transactions is accepted.

## Reorgs and Confiscation transactions 
When there is a reorg, the node treats the confiscation transactions the same
as other transactions, and they are all added back to the mempool. 

Transactions should be validated before being added back to the mempool. 

Whenever an output is frozen (by policy or consensus), the mempool is scanned
and any transactions spending the frozen output are removed from the mempool.

If there is a reorg to a tip where a previously valid transaction is now
spending a frozen output, that transaction is not added back to the mempool
during reorg. If it already exists in the mempool, it is removed. 

When there is reorg to a new chain we must consider:

- Confiscation transaction is in mempool at height H1 of current best chain before reorg, so this means H1 + 1 >= EnforceAtHeight
- The confiscation transaction remains whitelisted by node, so the
      EnforceAtHeight is still valid and applies to the new best chain which is
      at height H2
- We intend to add confiscation transaction back to the mempool when all of
      the following are true:
    - H2 + 1 >= EnforceAtHeight
    - confiscation transaction has not been already mined in new chain
    - Inputs have not been double-spent in new chain by conflicting transaction
- We may encounter scenario where the confiscation transaction is not available to add back to the mempool:
    - During reorg, the mempool evicted transactions including confiscation
      transaction
    - Fails validation: H2 + 1 >= EnforceAtHeight , meaning it’s too early
      to add back 
    - The BM will need to resubmit the transaction 


Consider a simple scenario on the Best Chain 

| Height:                               | 100 | … | 148 | 149     | 150     | 151      | 152   |
|  ---                                  | --- |---| --- | ---     | ---     | ---      | ---   |
| Freeze Order Enforce At Height        | O1  |   |     |         |         |          |       |
| Confiscate Order Enforce At Height    |     |   |     |         |  O2     |          |       |
| NT Court Order TXO State              | F   | F | F   | F       | C       |  C       | C     |
| BM TXO State                          | F   | F | F   | F       | F       | F        | F     | 
| Node Blacklist                        | F   | F | F   | F       | F       | F        | F     | 
| Confiscation Tx                       |     |   |     | Mempool | Mempool |  Mempool | Mined |

Now let’s examine what happens where there is a reorg. 

Let’s say the best chain is at Height 149 and confiscation transaction is in
mempool, and then a reorg occurs. 

| Height:           | 100 | … | 148 | 149     |  
| ---               | --- |---|---  |---      |
| Confiscation Tx   |     |   |     | Mempool |

Scenario 1. 

Reorg chain is at Height 148. The confiscation transaction cannot be accepted
into mempool, since height 149 has not been reached yet. **The confiscation
transaction needs to be retained by the node somewhere, or if dropped, the BM
must resubmit at height 149.** 

| Height:         | 100 | … | 148 | 
| ---             | --- |---| --- |
| Confiscation Tx |     |   |     |

Scenario 2. 

Reorg chain is at Height 149. During reorg, the confiscation transaction can be
added back to the mempool at height 149. Assume no mempool overflow. 

| Height:         | 100 | … | 148 | 149     | 
| ---             | --- |---| --- | ---     |
| Confiscation Tx |     |   |     | Mempool |

Scenario 3. 

Reorg chain is at Height 152. Confiscation transaction was already mined into
block 151. So the confiscation transaction does not need to be added to the
mempool, and will fail if tried. 

| Height:         | 100 | … | 148 | 149 | 150             | 151 | 152 |
| ---             | --- |---| --- | --- | ---             | --- | --- |
| Confiscation Tx |     |   |     |     | Not mined Mined |     |     |

Scenario 4. 

Reorg chain is at height 152. Confiscation transaction has not been mined yet.
It can be added to the mempool. Assume no mempool overflow. 

| Height:         | 100 | … | 148 | 149 | 150       | 151       | 152     |
| ---             | --- |---| --- | --- | ---       | ---       | ---     |
| Confiscation Tx |     |   |     |     | Not mined | Not mined | Mempool |

Scenario 5. 

Reorg chain is at Height 150. The confiscation transaction was double-spent by
a conflicting whitelisted transaction mined into block 150. So the confiscation
transaction will be rejected from mempool as some inputs already spent. 

| Height:         |100 | … | 148 | 149 | 150                                 | 
| ---             | ---|---| --- | --- | ---                                 |
| Confiscation Tx |    |   |     |     | Conflicting Tx Mined At This Height |


## IBD Mode 
When a node launches and is in IBD mode, how can the node verify that a
confiscation transaction received in a block is whitelisted, and that the block
should be accepted as valid? The solution is to rely on a combination of:

1. BM updating node whitelist via RPC **addToConfiscationTxidWhitelist** 
2. New configuration option **–assumeWhitelistedBlockDepth** (if a node is
   running without BM, they must rely on this) 

### New RPC: addToConfiscationTxidWhitelist 
When the node is launched, the BM should periodically update the node’s
whitelist by calling node RPC addToConfiscationTxidWhitelist. The BM should
also update the node’s Blacklist, to ensure the frozen TXO database is up to
date, since confiscation transactions are only valid if they are spending
frozen TXOs. 

By the time IBD mode reaches the first (new) confiscation transaction on the
blockchain, the node’s blacklist and whitelist *should* be up to date.

- If not, and there is a problem, the node operator will have to manually
  reconsider a block which the node mistakenly thinks is invalid because its
  whitelist was not up to date. This is a manual process documented in
  operations manual.

### New Config Option: assumeWhitelistedBlockDepth 
A new configuration option **–assumeWhitelistedBlockDepth** tells the node to
assume a confiscation transaction received in a block is whitelisted, when the
block is buried under enough PoW i.e. appears N blocks from chaintip. 

A configuration value N=10000 would mean confiscation transactions in blocks
older than 2.5 months are presumed to be whitelisted.

- This could be useful if in future, old whitelists are no longer available,
  which would result in IBD mode rejecting valid blocks. 

A configuration value N=0 would mean all confiscation transaction in blocks are
whitelisted

1. Useful if the node does not run with a BM, meaning the node’s whitelist is
   always empty and valid blocks containing confiscation transactions are
   rejected.

## Malleability
### Confiscation transactions
The confiscation transaction could be modified in-flight, changing the input
being confiscated, or the signature of an input which funds the transaction, or
the destination address. However, TXID (sha256d) covers the entire serialized
transaction, so any changes are easily detected as a modified transaction would
not have a whitelisted TXID. 

### Reward transactions 
For a timelocked confiscation chain with reward transactions, the reward
transactions are not whitelisted. If one transaction in the chain is malleated
so that it has a different txid, the next transaction in the chain will be
invalid (the input will refer to the original txid) and the rest of the reward
chain can no longer be mined. 

This does not impact the confiscation of funds, since the confiscation
transaction at the head of the chain has been mined. The only thing lost here
is the miner's reward. It is unlikely that a miner will malleate a reward
transaction and deprive themselves of potential reward. Though a rogue miner
might try to disrupt and annoy other miners. 

## Replay attacks 
The node does not download and verify the ConfiscationOrder and related
ConfiscatedTx documents referred to in the OP_RETURN data. An attacker could
try and re-use genuine OP_RETURN data in a new confiscation transaction to
steal funds from other (frozen) transaction outputs. This attack will fail due
to whitelisting, as the fake transaction will not have a txid which has been
whitelisted.
