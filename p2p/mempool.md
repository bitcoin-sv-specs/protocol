# Mempool Message

## Abstract

This document describes an extension to the existing `mempool` P2P network
message used to enable synchronisation of peers mempool contents.

## Introduction

The Bitcoin SV P2P protocol already includes a `mempool` message that has historically
been available to allow nodes to request details for the contents of
a peer's mempool.

An extended version of the `mempool` message is described here that also includes
an optional **age** field. Nodes willing to process mempool messages from peers
should restrict the details for transactions they send back to only those
transactions that have been in their mempool for longer than the specified
age.

## Message Format

The format of the extended `mempool` message is:

|Field size |Description    |Data Type  |Comments|
|----|----|----|----|
|0+  |age |uint64_t |Minimum mempool age for requested transaction details. |

## Activation

There are no special rules regarding activation of the new format P2P `mempool` message.
Old nodes can continue to use the old version of the message without modification.

