# GetMiningCandidate API (GMC)
 
GMC is an improved API for Miners, ensuring they can scale with the Bitcoin network by mining large blocks without the limitations of the RPC interface interfering as block sizes grow. 
Based on and credited to the work of Andrew Stone and Johan van der Hoeven, GMC works by removing the list of transactions from the RPC request found in ‘getblocktemplate’ and supplying only the Merkle proof of all the transactions currently in the mempool/blockcandidate. 
 
The newly added RPC methods are getminingcandidate and submitminingsolution. Together they provide a replacement to the long standing getblocktemplate and submitblock. 
These RPC calls do not pass the entire block to mining pools (which is more than double the size of the actual block when transmitted over the RPC interface). Instead, the candidate block header, proposed coinbase transaction, and coinbase Merkle proof are passed. This is the approximately the same data that is passed to hashing hardware via the Stratum protocol. 

A mining pool uses getminingcandidate to receive the previously described block information and a tracking identifier (ID). The ID is to be supplied back to the bitcoin node with submitminingsolution along with the associated block candidate. By default, the outstanding ID’s are purged from the bitcoin node 30 seconds after a new block has been found. 

The miner can then create a new (or modify the supplied) coinbase transaction and many block header fields, to create different candidates for hashing hardware. It then forwards these candidates to the hashing hardware via Stratum. When a solution is found, the mining pool can submit the solution back to bitcoind via submitminingsolution. 

A few of the benefits when using RPC getminingcandidate and RPC submitminingsolution are: 
* Massively reduced bandwidth and latency, especially for large blocks. This RPC requires log2(blocksize) data, this is due to the size of the merkle branch. 
* Faster JSON parsing and creation 
* Concise JSON 

## License
GetMiningCandidate API (GMC) is released under the terms of the MIT license. See LICENSE for more information or see https://opensource.org/licenses/MIT.

## Example Implementation 
 
An example CPU-miner program is provided with bitcoind that shows a proof-of-concept use of these functions. The source code is located at src/bitcoin-miner.cpp. 
### Functional Test 
A python based test of these interfaces is located at /test/functional/mining2.py. This example may be of more use for people accessing these RPCs in higher level languages. 
## Function documentation: 
### RPC getminingcandidate
Arguments: - [bool] provide_coinbase_tx (Optional - default is false)  

#### Returns:

|Variable|Description|
|:-----:|:-----:|   
| [uuid] id | The assigned id from Bitcoind. This must be provided with submitting a potential solution |
|[hex string] prevHash | Big Endian hash of the previous block | 
| [hex string] Coinbase (optional) | Suggested coinbase transaction is provided. Miner is free to supply their own or alter the supplied one. Altering will require parsing and splitting the coinbase in order to splice in/out data as required.  Requires Wallet to be enabled. Only returned when provide_coinbase_tx argument is set to true. Returns error if Wallet is not enabled |
|[int32_t] version | The block version | 
|[int64_t] coinbaseValue | Total funds available for the coinbase tx (in Satoshis) | 
|[uint32_t] nBits | Compressed hexadecimal difficulty | 
|[uint32_t] time | Current block time | 
|[int] height | The candidate block height | 
|[array] merkleProof | Merkle branch/path for the block, used to calculate the Merkle Root of the block candidate. This is a list of Little-Endian hex strings ordered from top to bottom |

#### Example 
```
{ 
  "id": "a5f1f38b-2a00-4913-833a-bbcbb39d5d2c", 
  "prevhash": "0000000020493e205694c9fcb42f7d4ce5d85e230d52fccc90a6354e13940396", 
  "coinbase": "02000000010000000000000000000000000000000000000000000000000000000000000000ffffffff0503878b1300ffffffff01c5a4a80400000000232103b8310da7c413106c6ce63814dbcd366c55e8ae39c8c43c1fdaeb76df56e4ff7dac00000000", 
  "version": 536870912, 
  "nBits": "1c4877e8", 
  "time": 1548132190, 
  "height": 1280903, 
  "merkleProof": [ 
    "497d51f3a933dd6e933cd37a4a5799066086d4ff45dce23f0819c7a6c7174ccb", 
    "c2de445eda326b4afcec1291fc0dad3c526ddb551cbb01e2e10a10ebe79d2482", 
    "7f417e9de2e8c37566141e3057eec37747a924117413ee7c2b8f902dd81b095f", 
    "b25810a0b826ea8bf848d6e3f98f6c0bf4d097f0d1854d50c6e12988f29757d6" 
  ] 
}
```  
 

### RPC submitminingsolution 
Arguments: A JSON String containing the below fields.

|Variable|Description|
|:-----:|:-----:|
|[uuid] id | The id supplied by getminingcandidate | 
|[uint32_t] nonce |Miner generated nonce |  
|[hex string] coinbase | The crafted or modified coinbase transaction (Optional) |
|[uint32_t] time | Block time (Optional - must fall within the mintime/maxtime consensus rules ) |
|[uint32_t] version | Block version (Optional) | 



#### Example 
```
{   
  "id": a5f1f38b-2a00-4913-833a-bbcbb39d5d2c, 
  "nonce": 1804358173, 
   "coinbase": "...00ffffffff10028122000b2fc7237b322f414431322ffff...", 
   "time": 1528925410, 
   "version": 536870912 
}  
```
#### Returns: 
 
Success: returns true  
Failure: JSONRPCException or error reason comprising of but not limited to the following.

|Error|Description| 
|:-----:|:-----:|
|Block candidate ID not found|Required ID was not found, ID is supplied by the corresponding getminingcandidate call |
|nonce not found|The nonce was not supplied or not recognized | 
|coinbase decode failed|The supplied coinbase was unable to be parsed| 
 
Other possible errors can result from the SubmitBlock (BIP22) method in which submitminingsolution ultimately calls upon. 
