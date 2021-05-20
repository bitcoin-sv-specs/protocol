# P2P Double-Spend Notifications 

## Introduction
Transaction creators such as merchants can benefit from notification of double-spends. This specification defines a protocol for adding callback server details to a transaction using an OP_RETURN output; if a double spend is detected by a node in the network, it should directly notify the specified HTTP callback server.

Callback details for a Double Spend Notification (DSNT) are embedded in the transaction as follows.

> OP_FALSE OP_RETURN OP_PUSHDATA PROTOCOL_ID OP_PUSHDATA CALLBACK_MESSAGE

There should only be one DSNT output in a transaction. The node only attempts to process the first DSNT output (lowest index) in the event there are multiple DSNT outputs.

# PROTOCOL ID
The protocol ID is a 32-bit identifier for Double Spend Notifications: 0x64736e74 “dsnt” (in this byte order).
The dsnt protocol ID will be registered at the BSV OP_RETURN Protocol Repository: https://github.com/bitcoin-sv-specs/op_return/

# Callback Message
Binary encoded message format.

|**Field**|**Description**|**Size**|
| :- | :- | :- |
|**Version**|<p>Version field with signal bits</p><p>Bit 7 = 0 for IPv4, 1 for IPv6</p><p>Bit 6 = 0, reserved</p><p>Bit 5 = 0, reserved</p><p>Bits 4-0 = dsnt protocol version (currently 1, max 31)</p>|1 byte|
|**IP address count**|Varint, >1, where 0 is invalid|1 – 9 bytes|
|**List of IP addresses**|<p>List of IP addresses.</p><p>If IPv4, address is 32 bits, network byte order</p><p>If IPv6, address is 128 bits, network byte order</p>|Count \* address size|
|**Input count**|Varint, >=0, where 0 means notifications for all inputs, must be <= num tx inputs|1 – 9 bytes|
|**List of inputs**|List of varints, identifying the inputs requiring notification. This field only exists if input count > 0.|Count \* varint|

Future protocol versions may use the reserved bits (e.g. use signal bit 6 for SSL) and update the message format with extra fields (e.g. port numbers, payment reward details).

## Validating the Callback Message

Below are examples of malformed callback messages which should be treated as invalid:

|**Callback Message**|**Invalid Reason**|
| :- | :- |
|**OP_FALSE OP_RETURN 'dsnt'**|Missing version field|
|**OP_FALSE OP_RETURN 'dsnt' FF**|Invalid version field|
|**OP_FALSE OP_RETURN 'dsnt' 01 00**|IP address count of 0 is not allowed|
|**OP_FALSE OP_RETURN 'dsnt' 01 01**|IP address count is 1, but missing IP address|
|**... 00 AB**|Input count is 0, so no data should follow|
|**... 01 07 08**|Input count is 1, so only 1 input should follow|
|**... 03 02**|Input count is 3, but missing 2 inputs|

## Callback Server
 
The HTTP callback server is publicly accessible and listens on port 80 for HTTP (version 1 does not support HTTPS or using a custom port)

To help BSV nodes identify a callback server, every HTTP response MUST include the header field “x-bsv-dsnt: <value>” (where value is dependent on the endpoint). If deployment is behind a HTTP cache (e.g. Varnish), the cache must be configured so that it does not strip out this header field.

Remark:  RFC 6648 recommends against using “x-” prefix for application protocols. Use of “x-bsv-dsnt” may be deprecated in the future, for example, if “bsv-dsnt” is registered with IANA.

### Two-Phase Submit
The API implements a two-phase submit protocol which mandates a BSV node MUST first query the callback server, to ask if it wants to receive a proof, before submitting the proof. 
This two-phase protocol helps prevent an attacker from hijacking BSV nodes into launching a DDoS attack and sending transaction data to a third-party, by falsely identifying the third-party as a callback server.
### Namespace
The endpoints belong to the /dsnt namespace for domain separation from other services like /mapi.
### Versioning
The endpoints include the protocol version /dsnt/1, so future protocol versions can easily change URL parameters. The BSV node will extract the protocol version from the CALLBACK_MESSAGE.

## /dsnt/1/query/txid
Purpose:
	Ask callback server if it wants to receive a proof for a dsnt-enabled transaction 

URL parameters:
1.	txid -  the dsnt-enabled transaction, hex string, 64 bytes in length

The BSV Node’s conflicting transactions may both be dsnt-enabled. In this scenario, the BSV node should submit the txid of the transaction in its mempool (first-seen rule).

Server response
*	HTTP status code
*	Header field “x-bsv-dsnt: <value>”. Value=0 if the server does not want the proof for the specified txid. Value=1 if the server wants the proof for the specified txid
*	Empty body

| HTTP | Status Code |	Reason |
|------|-------------|---------|
|200 |OK	|Query received |
|400 |Bad Request	|Bad query, txid is malformed|


### /dsnt/1/submit?txid=…&n=…&ctxid=...&cn=…
Purpose:
	Submit proof to callback server

URL parameters
1. txid -  the dsnt-enabled transaction, hex string, 64 bytes in length
2. n - input of dsnt-enabled transaction, uint >= 0
3. ctxid – the conflicting transaction (cannot be the same as txid), hex string, 64 bytes in length
4. cn - input of conflicting transaction, uint >= 0

HTTP Request Header Fields:
*	Content-Type: application/octet-stream
*	Content-Length: LENGTH

HTTP Body:
*	Proof in binary encoding format

Server Response:
*	HTTP status code
*	Header field “x-bsv-dsnt: <value>”. Value=0, server does not want the proof for the specified txid.	Value=1, server wants the proof for the specified txid
*	Empty body

It is a valid response for the server to send a HTTP response and then early disconnect without first reading the whole request. For example, a BSV node might encounter early disconnect with:
*	200 OK, “x-bsv-dsnt: 0” (server not interested in txid)
*	500 Internal Server Error, “x-bsv-dsnt: 1” (server is interested in txid, you can retry)

|**HTTP Status Code**|**Reason**|
| :- | :- |
|**100 Continue**|Response to client request with Expect: 100-continue, informing client to send body|
|**200 OK**|Received submission|
|**400 Bad Request**|<p>Catch-all for bad request</p><p>- malformed URL parameters</p><p>- bad headers</p><p>- content length is wrong</p><p>- content type is unsupported</p>|
|**500 Internal Server Error**|Something went wrong internally|

The server should consider allowing concurrent submissions of a proof, for a given txid, to mitigate against fake submissions and network issues slowing down the notification process.

## BSV Node and Invalid Callback Servers

The BSV node can identify an invalid third-party web server by examining HTTP responses
-	header field “x-bsv-dsnt” is missing
-	response codes not expected from a valid callback server
	-	301 Moved Permanently (many websites redirect HTTP requests to HTTPS)
	-	404 Not Found (dsnt endpoints don’t exist)

And also identify a bad server e.g. a valid callback server which is behaving badly
-	slow response time and high-latency data link
-	frequent use of response codes
	-	5xx – internal server error

## Double-Spend Proof
### Proof Format
For version 1, the proof format is the entire transaction, as the BSV Node supports notifications for non-standard UTXOs, where the corresponding input scriptSig could be large. E.g. locking script begins with OP_DROP.
Future protocol versions could use a more compact format, such as the conflicting input’s scriptSig along with any precomputed hashes of outputs needed for signature verification operations.

### Proof Validation
Validation only requires script validation of the utxo’s locking script and the conflicting input’s unlocking script. Validation does not require validation of other utxos in a dsnt-enabled transaction or other transaction wide checks.

For version 1, using the /submit parameters,
1.	Check sha256d(proof) == ctxid
2.	Check utxo spent by input txid:n is the same utxo being spent by input ctxid:cn
3.	The conflicting input script must validate.

## User-facing changes to BSV node

### New Config Option: dsnotifylevel=level

This option instructs the node on how to handle double spends, for a conflicting input, when at least one of the conflicting transactions is dsnt-enabled. The notification levels are:

0.  No notification (disables double-spend reporting)
1.  Notify only for standard UTXOs (new default behaviour)
2.  Notify for all UTXOs

### New Config Option: dsendpointmaxcount=n

This option sets a limit on the number of endpoints a node should attempt to notify for each dsnt-enabled transaction. The minimum value is 1, the default value is 3.

Example: A node, with a default setting of 3, will only attempt to contact the first 3 endpoints, even if the callback message contains 5 unique IP addresses.

Any bad endpoints (duplicates, uncontactable, etc.) in the IP address list will count towards the limit. Also, the node will not send multiple notifications to the same endpoint for duplicate IP addresses.

Example: A node, with a default setting of 3, will only contact the given endpoint once, if the callback message contains 2 duplicate IP addresses.

## Double-Spend Detection and Processing

### Detecting Double-Spends
When validating new transactions, there are three code paths which detect double-spends:

1.	Typically, when transaction tx1 is in the BSV node’s mempool and transaction tx2 is received from a peer, the node will drop tx2 without validating and report an internal error “txn-mempool-conflict”. No P2P message is sent to the peer.
2.	Conflicting transactions may be received and validated in parallel by the node. Tx1 may complete validation first and be added to the mempool before tx2 finishes validation. The node will drop tx2 and report an internal error “txn-mempool-conflict”. No P2P message is sent to the peer.
3.	However, if both conflicting transactions complete validation around the same time, before either one can be added to the mempool, the CTxnDoubleSpendDetector which synchronizes validation threads, will ensure only one transaction is added to the mempool. The other transaction is dropped and an external error “txn-double-spend-detected” is sent as a P2P reject message to the peer.

Double-spend transactions may be validated before they are even recognized as double-spends.

## Processing Double-Spend Notifications
When a double-spend has been detected, as detailed above, the BSV Node should:

1.	Check dsnotify configuration option to see if notifications are enabled
1.	If enabled but only for standard utxos, check if the utxo is standard
1.	Check the CALLBACK_MESSAGE is valid
	1.	At least one of the conflicting pair of txs will be dsnt-enabled with a CALLBACK_MESSAGE. If both are, use the tx which has been accepted into the mempool (first-seen rule)
1.	Find one conflicting input which is registered for notification in inputs list of CALLBACK_MESSAGE
1.	Check for duplicate notification: node may have already posted a notification for this txid
1.	Validate script: utxo locking script + input unlocking script
	1.	Skip if tx was already validated by the code path
1.	If script is valid,
	1.	generate proof
	1.	submit proof and callback details to a background thread responsible for HTTP communication with callback servers

The BSV node should continue with existing behaviour:
*	Drop conflicting transactions
*	If the callback server is notified, send P2P reject message “txn-double-spend-detected”
*	The node should continue to send notifications over ZMQ.

Remark: Transaction wide checks on a conflicting transaction are not necessary for double-spend detection. The existence of a conflicting input with valid script is proof that something fishy is going on, so the callback server should be notified. 

## Duplicate Notifications
The BSV node should keep a list of recent txids where a notification attempt has been made. If a new double-spend attempt is detected for a txid on the list, the BSV node should not notify the callback server.

## Script Validation Failure
If the script does not validate, the transaction itself is invalid. The peer should not have relayed this transaction to the node, so the peer should be banned.

## Script Validation Time-Out and Local Policy Violation
A peer may relay conflicting transactions where script validation fails due to time-out or a local policy violation. As the script cannot be validated, a notification cannot be sent.

If an honest peer with superior processing power repeatedly relays these transactions, the node should not ban the honest peer, but increase a temporary suspension score (similar to ban score).

This avoids wasting CPU time to validate scripts which fail locally, and helps mitigate against a resource exhaustion attack using double-spend transactions.

The policy limits as configured by the node are used, since the conflicting transaction would not have been accepted by the mempool, even if it had been processed before its counterpart.

## Document Information
Version 
* 2020-05-07 – Simon Liu – initial version.


