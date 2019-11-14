# 2018 November Bitcoin Cash Upgrade Specification

## Summary
When the median time past(1) of the most recent 11 blocks (MTP-11) is greater than or equal to UNIX timestamp 1542300000, 
Bitcoin Cash will execute an upgrade of the network consensus rules according to this specification. Starting from the 
next block these consensus rules changes will take effect:
* Re-enable the opcodes OP_MUL, OP_INVERT, OP_LSHIFT & OP_RSHIFT
* Increase the limit on the number of opcodes allowed per script to 500

The following are recommended changes, not consensus changes:
* Set the default maximum generated block size to 32MB
* Set the default maximum accepted block size to 128MB

## Re-enable opcodes OP_MUL, OP_INVERT, OP_LSHIFT, & OP_RSHIFT
The specification for these opcodes is [here](2018-11-15 Re-enabling Old Opcodes.md).
 
## Increasing the limit on the number of opcodes per script
The specification for this change is [here](2018-11-15%20Increase%20Opcode%20Limit.md).

## References
1 - Median Time Past is described in [here](https://en.bitcoin.it/wiki/Block_timestamp).

## Document Information
Author: Daniel Connolly, nChain Ltd
