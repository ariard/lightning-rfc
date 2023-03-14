# Credential Issuance

This document specifies the Credentials Issuance protocol inside the Staking Credentials framework.

The messages data format, validation algorithms and implementation considerations are laid out.

This draft is version 0.1 of the Credential Issuance protocol.

## Credentials Issuance


	+-------+						  +-------+
	|	|						  |	  |
	|	|						  |	  |
	|	|--(1)--- request_credentials_authentication ---->|	  |
	|   A   |						  |   B	  |
	|	|						  |	  |
	|	|<-(2)--- reply_credentials_authentication -------|	  |
	|	|						  |	  |
	|	|						  |	  |
	+-------+						  +-------+



### The `request_credentials_authentication` message

This message contains a proof of asset as a base collateral for the credentials
and a set of unsigned blinded credentials.

TODO: add a random session id to enable processing parallelization ?

1. type: 37560 (`request_credentials_authentication`)
2. data:
    * [`assetlen*byte`: `collateral_assets`]
    * [`credentiallen*byte`: `blinded_credentials`]

#### Requirements

The sender:
  - MUST NOT send `collateral_assets` if they have not been announced by a previous `credential_policy` issued by this node.
  - MUST NOT send `blinded_credentials` of a format which not been previously announced by a `credential_policy` issued by this node.
  - MUST set `blinded_credentials` to less than `max_onion_size`.

The receiver:
  - if `collateral_asset` is not supported:
    - MUST reject this authentication.
  - if `blinded_credentials` format is not supported:
    - MUST reject this authentication.
  - if `blinded_credentials` size is equal or superior to `max_onion_size`:
    - MUST reject this authentication.
  - if the `collateral_asset` amount is not covering the quantity of `blinded_credentials` requested for signature as announced by `credential_policy`:
    - MUST reject this authentication.

#### Rationale


### The `reply_credentials_authentication` message

This message contains an issuance pubkey and a set of signatures for each credential
from `request_credentials_authentication`.

1. type: 37561 (`reply_request_credentials_authentication`)
2. data:
    * [`point` : `issuance_pubkey`]
    * [`credentiallen*signature`:`credentials_signature`]

#### Requirements

The sender:
  - MUST set `credentials_signatures` to less than `max_onion_size`.
  - MUST sort the `credentials_signature` in the order of reception of `blinded_credentials` in the corresponding `request_credentials_authentication`

The receiver:
  - if `credentials_signature` size is equal or superior to `max_onion_size`:
    - MUST reject this authentication reply.
  - if the `issuance_pubkey` is not matching the announced key in `credentials_policy`:
    - MUST reject this authentication reply.
    - MAY add this Issuer identity on a banlist.
  - if the signatures are not valid for the `blinded_credentials` sent in the corresponding `request_credentials_authentication`:
    - MUST reject this authentication reply.
  
#### Rationale

### Implementations Considerations

The credential signature request could constitute a CPU DoS vector, therefore the Issuer
should ensure the minimal proof of assets scores for an amount high-enough to deter an adversary.

If on-chain transaction is a supported proof of asset, it should be confirmed with few blocks
to avoid swallow reorgs leading to a free dump of counter-signed collaterals.

Eclipse attack: the Committer is blinded from the honest issuance pubkey to conduct a deanonymization
attack.
