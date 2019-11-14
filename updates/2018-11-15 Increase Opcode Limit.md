# Specification for the Increase of Script Non-Push Opcode Limit
November 2018 Bitcoin release.

Version 1.1

## Introduction
Some desirable Bitcoin transactions can require scripts with more than 201 (non-push) opcodes. There have been proposals 
to use a high-level language to generate complex scripts, as well as proposals to use in-script cryptographic techniques 
to implement complex behaviour. Note also that a single transaction with greater than 201 opcodes may be equivalent to 
many smaller transactions, but cheaper since the costs of processing transaction inputs is only charged once.

Other limits exist in Bitcoin:
* Max script element size = 520
* Max script size = 10000 items
* Max public keys per multisig = 20
* Max transaction size = 1MB
* Max SIG opcodes per transaction = 20,000
* Max SIG opcode per MB = 20,000
* Max stack size (std + alt) = 1000 elements
* Max element size = 520 bytes.

This specification describes the increase of the current limit on the number of non-push opcodes that can be included in 
a valid bitcoin script.

## New Behaviour
* The limit on the number of non-push opcodes per script will be raised to 500.

Other limits, including the the limits on the total script size, are not changed by this specification.
 
## Specification Status
This document is in Google Doc format for presentation and ease of commenting purposes. It will be made available for 
public comment.
 
This document is a specification document. It was produced following a proposal, community discussions in the 
“[WG] OpCodes” telegram group, and a workshop style meeting.

The proposal document is [here](https://docs.google.com/document/d/1WpbZZH-mDm0A75Q86RsQbk39IKlB6Udz6wY4aytgqtg/edit?usp=sharing).
 
## Document History
* 2018-10-15 - Daniel Connolly, nChain Ltd - updated to change removal of limit to an increase
* 2018-08-13 - Daniel Connolly, nChain Ltd -minor edits, grammar, typos
* 2018-08-03 - Shaun O’Kane, nChain Ltd - initial version
