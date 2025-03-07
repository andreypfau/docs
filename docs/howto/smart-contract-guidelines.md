# Smart contract guidelines
This page collects some recommendations and best practices that could be followed when developing new smart contracts for TON Blockchain.

## Internal messages

Smart contracts interact with each other by sending the so-called **internal messages**. When an internal message reaches its intended destination, an ordinary transaction is created on behalf of the destination account, and the internal message is processed as specified by the code and the persistent data of this account (smart contract). In particular, the processing transaction can create one or several outbound internal messages, some of which could be addressed to the source address of the internal message being processed. This can be used to create simple "client-server applications", when a query is incapsulated in an internal message and sent to another smart contract, which processes the query and sends back a response, again as an internal message.

This approach leads to the necessity of distinguishing whether an internal message is intended as a "query" or as a "response", or doesn't require any additional processing (like a "simple money transfer"). Furthermore, when a response is received, there must be a means to understand to which query it corresponds.

In order to achieve this goal, the following recommended internal message layout can be used (notice that the TON Blockchain does not enforce any restrictions on the message body, so these are indeed just recommendations):

1. The body of the message can be embedded into the message itself, or be stored in a separate cell referred to from the message, as indicated by the TL-B scheme fragment: 
   
`message$_ {X:Type} ... body:(Either X ^X) = Message X;`

  The receiving smart contract should accept at least internal messages with embedded message bodies (whenever they fit into the cell containing the message). If it accepts message bodies in separate cells (using the `right` constructor of `(Either X ^X)`), the processing of the inbound message should not depend on the specific embedding option chosen for the message body. On the other hand, it is perfectly valid not to support message bodies in separate cells at all for simpler queries and responses.

2. The message body typically begins with the following fields:

    * A 32-bit (big-endian) unsigned integer `op`, identifying the `operation` to be performed, or the `method` of the smart contract to be invoked.
    * A 64-bit (big-endian) unsigned integer `query_id`, used in all query-response internal messages to indicate that a response is related to a query (the `query_id` of a response must be equal to the `query_id` of the corresponding query). If `op` is not a query-response method (e.g., it invokes a method that is not expected to send an answer), then `query_id` may be omitted.
    * The remainder of the message body is specific for each supported value of `op`.

3. If `op` is zero, then the message is a "simple transfer message with comment". The comment is contained in the remainder of the message body (without any `query_id` field, i.e., starting from the fifth byte). If it does not begin with the byte `0xff`, the comment is a text one; it can be displayed "as is" to the end user of a wallet (after filtering invalid and control characters and checking that it is a valid UTF-8 string). For instance, users may indicate the purpose of a simple transfer from their wallet to the wallet of another user in this text field. On the other hand, if the comment begins with the byte `0xff`, the remainder is a "binary comment", which should not be displayed to the end user as text (only as hex dump if necessary). The intended use of "binary comments" is, e.g., to contain a purchase identifier for payments in a store, to be automatically generated and processed by the store's software.

   Most smart contracts should not perform non-trivial actions or reject the inbound message on receiving a "simple transfer message". In this way, once `op` is found to be zero, the smart contract function for processing inbound internal messages (usually called `recv_internal()`) should immediately terminate with a zero exit code indicating success (e.g., by throwing exception `0`, if no custom exception handler has been installed by the smart contract). This will lead to the receiving account being credited with the value transferred by the message without any further effect.

4. A "simple transfer message without comment" has an empty body (without even an `op` field). The above considerations apply to such messages as well. Note that such messages should have their bodies embedded into the message cell.

5. We expect "query" messages to have an `op` with the high-order bit clear, i.e., in the range `1 .. 2^31-1`, and "response" messages to have an `op` with the high-order bit set, i.e., in the range `2^31 .. 2^32-1`. If a method is neither a query nor a response (so that the corresponding message body does not contain a `query_id` field), it should use an `op` in the "query" range `1 .. 2^31 - 1`.

6. There are some "standard" response messages with the `op` equal to `0xffffffff` and `0xfffffffe`. In general, the values of `op` from `0xfffffff0` to `0xffffffff` are reserved for such standard responses.

    * `op` = `0xffffffff` means "operation not supported". It is followed by the 64-bit `query_id` extracted from the original query, and the 32-bit `op` of the original query. All but the simplest smart contracts should return this error when they receive a query with an unknown `op` in the range `1 .. 2^31-1`.
    * `op` = `0xfffffffe` means "operation not allowed". It is followed by the 64-bit `query_id` of the original query, followed by the 32-bit `op` extracted from the original query.

   Notice that unknown "responses" (with an `op` in the range `2^31 .. 2^32-1`) should be ignored (in particular, no response with an `op` equal to `0xffffffff` should be generated in response to them), just as unexpected bounced messages (with "bounced" flag set).

## Paying for processing queries and sending responses


In general, if a smart contract wants to send a query to another smart contract, it should pay for sending the internal message to the destination smart contract (message forwarding fees), for processing this message at the destination (gas fees), and for sending back the answer if required (message forwarding fees).

In most cases, the sender will attach a small amount of TON Coins (e.g., one TON Coin) to the internal message (sufficient to pay for the processing of this message), and set its "bounce" flag (i.e., send a bounceable internal message); the receiver will return the unused portion of the received value with the answer (deducting message forwarding fees from it). This is normally accomplished by invoking `SENDRAWMSG` with `mode = 64` (cf. Appendix A of TON VM documentation).

If the receiver is unable to parse the received message and terminates with a non-zero exit code (for example, because of an unhandled cell deserialization exception), the message will be automatically "bounced" back to its sender, with the "bounce" flag cleared and the "bounced" flag set. The body of the bounced message will be the same as that of the original message; therefore, it is important to check the "bounced" flag of incoming internal messages before parsing the `op` field in the smart contract and processing the corresponding query (otherwise there is a risk that the query contained in a bounced message will be processed by its original sender as a new separate query). If the "bounced" flag is set, special code could find out which query has failed (e.g., by deserializing `op` and `query_id` from the bounced message) and take appropriate action. A simpler smart contract might simply ignore all bounced messages (terminate with zero exit code if "bounced" flag is set).

On the other hand, the receiver might parse the incoming query successfully and find out that the requested method `op` is not supported, or that another error condition is met. Then a response with `op` equal to `0xffffffff` or another appropriate value should be sent back, using `SENDRAWMSG` with `mode = 64` as mentioned above.

In some situations, the sender wants both to transfer some value to the sender and to receive either a confirmation or an error message. For instance, the validator elections smart contract receives an election participation request along with the stake as the attached value. In such cases, it makes sense to attach, say, one extra TON Coin to the intended value. If there is an error (e.g., the stake may not be accepted for any reason), the full received amount (minus the processing fees) should be returned to the sender along with an error message (e.g., by using `SENDRAWMSG` with `mode = 64` as explained before). In the case of success, the confirmation message is created and exactly one TON Coin is sent back (with the message transferring fees deducted from this value; this is `mode = 1` of `SENDRAWMSG`).

## Using non-bounceable messages


Almost all internal messages sent between smart contracts should be bounceable, i.e., should have their "bounce" bit set. Then, if the destination smart contract does not exist, or if it throws an unhandled exception while processing this message, the message will be "bounced" back carrying the remainder of the original value (minus all message transfer and gas fees). The bounced message will have the same body, but with the "bounce" flag cleared and the "bounced" flag set. Therefore, all smart contracts should check the "bounced" flag of all inbound messages and either silently accept them (by immediately terminating with a zero exit code) or perform some special processing to detect which outbound query has failed. The query contained in the body of a bounced message should never be executed.

On some occasions, non-bounceable internal messages must be used. For instance, new accounts cannot be created without at least one non-bounceable internal message sent to them. Unless this message contains a StateInit with the code and data of the new smart contract, it does not make sense to have a non-empty body in a non-bounceable internal message.

It is a good idea not to allow the end user (e.g., of a wallet) to send unbounceable messages carrying large value (e.g., more than five TON Coins), or at least to warn them if they try this. It is a better idea to send a small amount first, then initialize the new smart contract, and then send a larger amount.

## External messages


External messages are sent from the outside to the smart contracts residing in the TON Blockchain to make them perform certain actions. For instance, a wallet smart contract expects to receive external messages containing orders (e.g., internal messages to be sent from the wallet smart contract) signed by the wallet's owner; when such an external message is received by the wallet smart contract, it first checks the signature, then accepts the message (by running the TVM primitive `ACCEPT`), and then performs whatever actions are required.

Notice that all external messages must be protected against replay attacks. The validators normally remove an external message from the pool of suggested external messages (received from network); however, in some situations another validator could process the same external message twice (thus creating a second transaction for the same external message, leading to the duplication of the original action). Even worse, a malicious actor could extract the external message from the block containing the processing transaction and re-send it later. This could force a wallet smart contract to repeat a payment, for example.

The simplest way to protect smart contracts from replay attacks related to external messages is to store a 32-bit counter `cur-seqno` in the persistent data of the smart contract, and to expect a `req-seqno` value in (the signed part of) any inbound external messages. Then an external message is ACCEPTed only if both the signature is valid and `req-seqno` equals `cur-seqno`. After successful processing, the `cur-seqno` value in the persistent data is increased by one, so the same external message will never be accepted again.

One could also include an `expire-at` field in the external message, and accept an external message only if the current Unix time is less than the value of this field. This approach can be used in conjuction with `seqno`; alternatively, the receiving smart contract could store the set of (the hashes of) all recent (not expired) accepted external messages in its persistent data, and reject a new external message if it is a duplicate of one of the stored messages. Some garbage collecting of expired messages in this set should also be performed to avoid bloating the persistent data.

In general, an external message begins with a 256-bit signature (if needed), a 32-bit `req-seqno` (if needed), a 32-bit `expire-at` (if needed), and possibly a 32-bit `op` and other required parameters depending on `op`. The layout of external messages does not need to be as standardized as that of the internal messages because external messages are not used for interaction between different smart contracts (written by different developers and managed by different owners).

## Get-methods


Some smart contracts are expected to implement certain well-defined get-methods. For instance, any subdomain resolver smart contract for TON DNS is expected to implement the get-method `dnsresolve`. Custom smart contracts may define their specific get-methods. Our only general recommendation at this point is to implement the get-method `seqno` (without parameters) that returns the current `seqno` of a smart contract that uses sequence numbers to prevent replay attacks related to inbound external methods, whenever such a method makes sense.
