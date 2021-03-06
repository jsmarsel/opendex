# BOLD #4: Swap Protocol

## Overview

Once an order match is found in the taker’s order book, the swap protocol should be initiated. The current swap protocol assumes that the taker and maker are connected via a payment channel network (e.g. [Lightning](http://lightning.network/) or [Connext](https://connext.network/)) with sufficient balance available for the swap. The following is the swap protocol's "happy" flow:

1. Taker finds a match, e.g. buying 1 BTC for 10k DAI
2. Taker creates the private `r_preimage` and the public `r_hash` for the atomic swap
3. Taker sends the `SwapRequest` message to the maker, which includes `r_hash`
4. Maker confirms full or partial quantity in the `SwapAccepted` message
5. Taker starts the swap by dispatching the first-leg HTLCs on the DAI payment channel to the maker end, using `r_hash`
6. Maker listens for an incoming HTLC on the DAI payment channel. Once it arrives he verifies price and quantity and then dispatches the second-leg HTLCs on the BTC payment channel to the taker end.
7. Taker listens for an incoming HTLC on the BTC payment channel. Once it arrives he releases `r_preimage`. This allows **both** the taker and the maker payments to finalize.
8. Both nodes locally mark the swap as completed once the respective HTLC is resolved and the payment is finalized.

Possible misbehaviors and their outcome:

| Misbehavior                                                                        | Outcome                                               | Effect on payment channels 
|-------------------------------------------------------------------------------------|-------------------------------------------------------|------------------------------------------------------------|
| Maker doesn't respond to the `SwapRequest` message                                  | Taker should timeout the swap and penalize the maker  | None                                                       |
| Taker doesn't start the swap after receiving the `SwapAccepted` message             | Maker should timeout the swap and penalize the taker  | None                                                       |
| Maker receives the first-leg HTLC with insufficient amount or incorrect CLTV delta  | Maker should send the taker a `SwapError` message   | Taker funds are locked until HTLC expiration               |
| Maker doesn't continue the swap after receiving the first-leg HTLC                  | Taker should timeout the swap and penalize the maker  | Taker funds are locked until HTLC expiration               |
| Taker receives the second-leg HTLC with insufficient amount or incorrect CLTV delta | Taker should sends the maker a `SwapError` message  | Both Taker and Maker funds are locked until HTLC expiration|
| Taker doesn't release `r_preimage` after receiving the second-leg HTLC              | Maker should timeout the swap and penalize the taker  | Both Taker and Maker funds are locked until HTLC expiration|

## Swap Protocol
### SwapRequest Message (0x0c)

	`string id = 1`
	The message's globally unique identifier, generated by the sender 

    `uint64 proposed_quantity = 2`
    The proposed quantity

    `string pair_id = 3`
    The trading pair for the swap

    `string order_id = 4`
    The unique identifier of the maker order

	`string r_hash = 5`
	The taker preimage hash (in hex)

	`uint32 taker_cltv_delta = 6`
    The CLTV delta from the current height that should be used to set the timelock for the final hop when sending to the taker

The `SwapRequest` message is sent by the taker to the maker to start the swap negotiation. 

### SwapAccepted Message (0x0d)

    `string id = 1`
	The message's globally unique identifier, generated by the sender 
	
    `uint64 req_id = 2`
    The id from the received SwapRequest message

	`string r_hash = 3`
    The taker’s preimage hash (in hex) from the received SwapRequest message

	`string quantity = 4`
	The accepted quantity (which may be less than the proposed quantity)

	`uint32 maker_cltv_delta = 5`
    The CLTV delta from the current height that should be used to set the timelock for the final hop when sending to the maker

The `SwapAccepted` message is sent by the maker to the taker to accept the swap request.

### SwapFailed Message (0x0f)

	`string id = 1`
	The message's globally unique identifier, generated by the sender 

    `uint64 req_id = 2`
    An optional id from the received SwapRequest message. Otherwise, this field is empty

	`string r_hash = 3`
	The taker’s preimage hash (in hex)

	`string error_message = 4`
	Additional information regarding the failure reason

	`uint32 failure_reason = 5`
	The failure reason

The `SwapFailed` message can be sent by either side of the swap protocol, at any time, to announce the swap termination.

`failure_reason` is an optional parameter for specifying the failure reason:

| Failure Reason | Meaning                       | Description                                                                                              |
|----------------|-------------------------------|----------------------------------------------------------------------------------------------------------|
| `0x00`         | Order Not Found               | Could not find the order specified by a swap request                                                     |
| `0x01`         | Order On Hold                 | The order specified by a swap request is on hold for a different ongoing swap                            |
| `0x02`         | Invalid Swap Request          | The swap request contained invalid data                                                                  |
| `0x03`         | Swap Client Not Setup         | We are not connected to both swap clients, or we are missing public key identifiers for the peer's nodes |
| `0x04`         | No Route Found                | Could not find a route to complete the swap                                                              |
| `0x05`         | Unexpected Client Error       | A swap client call failed for an unexpected reason                                                       |
| `0x06`         | Invalid Swap Message Received | Received a swap message with invalid data                                                                |
| `0x07`         | Send Payment Failure          | The call to send payment failed                                                                          |
| `0x08`         | Invalid Resolve Request       | The swap resolver request was invalid                                                                    |
| `0x09`         | Payment Hash Reuse            | The swap request attempts to reuse a payment hash                                                        |
| `0x0a`         | Swap Timed Out                | The swap timed out while we were waiting for it to complete execution                                    |
| `0x0b`         | Deal Timed Out                | The deal timed out while we were waiting for the peer to respond to our swap request                     |                                         
| `0x0c`         | Unknown Error                 | The swap failed due to an unrecognized error  
