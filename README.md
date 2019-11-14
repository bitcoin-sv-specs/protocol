# Bitcoin SV Specification & Infrastructure Standards

The Bitcoin SV Specification precisely and completely specify the rules that define Bitcoin SV. 
These rules are commonly referred to as the Consensus Rules.

In addition to the Bitcoin Specification there are a number of Infrastructure Standards that have emerged
over the years. The P2P Network standard enables infrastructure components to communicate with each
other. The RPC Standard defines a common set of methods to control and interact with a Bitcoin node.

The target audience for these documents are IT professionals that already have a solid understanding of Bitcoin. They 
specify the details of the infrastructure layer, namely the Bitcoin SV blockchain and Bitcoin SV network. The documents 
do not describe emergent behaviour based on these specifications, nor do they describe how applications or services
may be built upon the infrastructure layer and are not intended to be used as an introduction to Bitcoin.

## Bitcoin Specification
A full Bitcoin Specification has not been completed yet. At this time we only have specifications for the updates
that have been applied.

Updates to the Bitcoin Specification (ordered by descending activation date):
* 2019-07-24 - [Quasar Protocol Upgrade](updates/2019-07-24%20Quasar%20Protocol%20Upgrade.md)
* 2018-11-15 - [2018 November Bitcoin Cash Upgrade Specification](updates/2018-11-15%20BCH%20Upgrade%20Spec.md), 
[Specification for Re-enabling old Opcodes](updates/2018-11-15%20Re-enabling%20Old%20Opcodes.md),
and [Specification for the Increase of Script Non-Push Opcode Limit](updates/2018-11-15%20Increase%20Opcode%20Limit.md)
* 2018-05-15 - [May 2018 Hardfork Specification](updates/may-2018-hardfork.md) and [Restore disabled script opcodes, May 2018](updates/may-2018-reenabled-opcodes.md)
* 2017-11-13 - [November 13th Bitcoin Cash Hardfork Technical Details](updates/nov-13-hardfork-spec.md)
* 2017-08-01 - [UAHF Technical Specification](updates/uahf-technical-spec.md) and [UAHF Test Plan](updates/uahf-test-plan.md)

## P2P Network Standard
Full documentation of the P2P Network Standard has not been completed yet. At this time we only have documentation for the updates
that have been applied.

* [Protocol Configuration Message (protoconf)](p2p/protoconf.md)

## RPC Standard
Full documentation of the RPC Standard has not been completed yet. At this time we only have documentation for the updates
that have been applied.

* [getminingcandidate](rpc/GetMiningCandidate.md)

# Document Information
Authors: Daniel Connolly, nChain Ltd; Shaun O’Kane, nChain Ltd

Version:
* 2019-11-14 - Daniel Connolly - initial version, pulling in various previous specifications

All documents in this collection, are released under the terms of the Open BSV License. See the file [LICENSE](LICENSE) 
for more information.

The files may-2018-hardfork.md, may-2018-reenabled-opcodes.md, nov-13-hardfork-spec.md, uahf-technical-spec.md,
uahf-test-plan.md, all of which are located in the updates directory, were copied from the GitHub repository at 
https://github.com/bitcoincashorg/bitcoincash.org/ on 14th November 2019. The files were originally licensed under the 
MIT License, which is available in the file [MIT_LICENSE](MIT_LICENSE). The authors of these files were not specified 
within the files and are not included in the list of authors above. 
