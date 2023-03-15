# Onions Communications

This document specifies the usage of BOLT 4 onion messages and HTLC onions inside the Staking
Credentials framework.

The message data format, validation algorithms and implementations considerations are laid out.

This draft is version 0.1 of the Onions Communications.

## Credentials Issuance - Communication

The `request_credentials_authentication` message is included in the `onionmsg_payloads` for the
final hop. A blinded route from a given "introduction node" should be attached for replies.

The sender should conform to all the creator requirements described in BOLT4.

The onion messages are unreliable. In case of lack of answer from the credentials issuer, a new
onion message can be retry with an identical collateral identifiers (to avoid collateral deanonymization
attacks).

## Redemption - Communication

The `redeem_credentials` message is included in the HTLC `payload` protected under the packet-wide HMAC.

The HTLC `payload` should be maxed out to 32 KB to constitute a single anonymity set, independently of
the quantity of credentials attached.
