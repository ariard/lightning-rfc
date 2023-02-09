# Redemption Phase

This document specifies the Redemption protocol inside the Staking Credentials framework.

The messages data format, validation algorithms and implementation considerations are laid out.

This draft is version 0.1 of the Redemption phase protocol.

## Credentials Redemption

	+-------+				  		+-------+
	|	|				  		|	|
	|	|				  		|	|
	|	|--(1)--- redeem_credentials ------------------>|	|
	|   A   |				  		|   B   |
	|	|						|	|
	|	|				  		|	|
	|	|-------- contract_request (non-specified) ---->|	|
	|	|				  		|	|
	+-------+				  		+-------+

### The `redeem_credentials` message

This message contains a set of unblinded credentials, a set of corresponding credentials
signatures, a set of unsigned blinded credentials for the reward mode and a contract
identifier.

TODO: add the issuance pubkey to prevent issue with propagation delay

1. type: 37562 (`redeem_credentials`)
2. data:
    * [`credentiallen*byte`: `unblinded_credentials`]
    * [`credentiallen*signature`: `credentials_signature`]
    * [`credentiallen*byte`: `reward_blinded_credentials`]
    * [`32*byte`: `contract_identifier`]

The `contract_identifier` should pair with a random unique identifier provided by the
contract flow. In the context of channel jamming, the `contract_identifier` matches the
one provided in the onion.

#### Requirements

The sender:
   - MUST set the sum of `unblinded_credentials`, `credentials_signature` and `blinded_credentials` to less than `max_onion_size`
   - MUST sort the `credentials_signature` in the order of reception of `blinded_credentials` in the corresponding `request_credentials_authentication`

The receiver:
   - if the signatures are not valid for the `unblinded_credentials` sent in the corresponding `request_credentials_authentication`:
     - MUST reject this redeem.

#### Rationale

### Implementations Considerations

The replay of credentials redeemption against a ContractProvider could constitute a non-compensated usage
of the contract liquidity, therefore once redeemed the credentials should be logged and any secondary redeem
attempt should be rejected. The state of credentials usage should be maintained by the Issuer and queried
by the ContractProviders under scope.

An authenticated and encrypted communication channel should be maintained between the Issuer and the ContractProviders
to avoid man-in-the-middle, where a credential validaiton is faked.
