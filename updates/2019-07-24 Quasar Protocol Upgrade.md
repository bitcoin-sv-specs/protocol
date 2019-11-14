# Quasar Protocol Upgrade

## Summary
When the median time past(1) of the most recent 11 blocks (MTP-11) is greater than or equal to UNIX timestamp 1563976800 
(2019-07-24T14:00:00), the default configuration values should change as follows:

* Set the default maximum generated block size to 128MB
* Set the default maximum accepted block size to 2GB

## References
1 - Median Time Past is described in [here](https://en.bitcoin.it/wiki/Block_timestamp). 

## Document Information
Author: Daniel Connolly, nChain Ltd
