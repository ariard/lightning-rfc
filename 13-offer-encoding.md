# BOLT #13: Flexible Protocol for Lightning Payments

# Table of Contents

  * [Limitations of BOLT 11](#limitations-of-bolt-11)
  * [Payment Flow Scenarios](#payment-flow-scenarios)
  * [Encoding](#encoding)
  * [TLV Fields](#tlv-fields)
  * [Invoices](#invoices)
  * [Offers](#offers)
  * [Invoice Requests](#invoice-requests)

# Limitations of BOLT 11

The BOLT 11 invoice format has proven popular, but has several
limitations:

1. The entangling of bech32 encoding makes it awkward to send
   in other forms (i.e. inside the lightning network itself).
2. The signature applying to the entire invoice makes it impossible
   to prove an invoice without revealing its entirety.
3. Fields are not generally extractable for external use: the `h`
   field was a boutique extraction of the `d` field, only.
4. The lack of 'it's ok to be odd' rule makes backwards compatibility
   harder.
5. The 'human-readable' idea of separating amounts proved fraught:
   `p` was often mishandled, and amounts in pico-bitcoin are harder
   than the modern satoshi-based counting.
6. The bech32 encoding was found to have an issue with extensions,
   which means we want to replace or discard it anyway.
7. The `payment_secret` designed to prevent probing by other nodes in
   the path was only useful if the invoice remained private between the
   payer and payee.
8. Invoices must be given per-user, and are actively dangerous if two
   payment attempts are made for the same user.


# Payment Flow Scenarios

Here we use "user" as shorthand for the individual user's lightning
node, and "merchant" as the shorthand for the node of someone who is
selling or has sold something.

There are two basic payment flows supported by BOLT 13:

The general user-pays-merchant flow is:
1. A merchant publishes an *offer*, such as on a web page or a QR code.
2. Every user requests a unique *invoice* over the lightning network
   using an *invoice_request* message.
3. The merchant replies with the *invoice*.
4. The user makes a payment to the merchant indicated by the invoice.

The merchant-pays-user flow (e.g. ATM or refund):
1. The merchant provides a user-specific *invoice_request* in a webpage or QR code,
   with an amount (for a refund, also a reference to the to-be-refunded
   invoice).
2. A user sends an *invoice* for the amount in the *invoice_request* (for a
   refund, a proof that they requested the original)
3. The merchant makes a payment to the user indicated by the invoice.

## Payment Proofs and Payer Proofs

Note that the normal lightning "proof of payment" can only demonstrate that an
invoice was paid (by showing the preimage of the `payment_hash`), not who paid
it.  The merchant can claim an invoice was paid, and once revealed, anyone can
claim they paid the invoice, too.[1]

1. Sharing the minimum information required to prove sections of the
   invoice in dispute (e.g. the description of the item, the payment
   hash, and the merchant's signature).
2. Contain a transferrable proof that the user is the one who was
   responsible for paying the invoice in the first place.

Providing a key in *invoice_request* allows a user to prove that they were the one
to request the invoice.  In addition, the merkle construction of the BOLT 13
invoice signature allows the user to selectively reveal fields of the invoice
in case of dispute.

# Encoding

Each of the forms documented here are in
[TLV](01-messaging.md#type-length-value-format) format.

The supported ASCII encoding is the designated prefix, followed by a
`1`, followed by a bech32-style data string of the TLVs in order,
optionally interspersed with `+` (for indicating additional data is to
come).

FIXME: Encoding/decoding requirements.
- Any required padding bits are set to zero / reject if not zero.

## Rationale

The use of bech32 is arbitrary, but already exists in the bitcoin
world.  We omit the six-character trailing checksum since all uses
here involve a signature.

The use of `+` (which is ignored) allows use of invoices over limited
text fields like Twitter:

```
lno1xxxxxxxx+

yyyyyyyyyyyy+

zzzzz
```

## Signature Calculation

Each form is signed using one or more TLV signature elements; TLV types
240 through 1000 are considered signature elements.

The only currently defined signature TLV is the secp256k1 signature of
the SHA256(`LnPrefix` | prefix | merkle-root), where prefix is the designated
prefix, e.g. `lno`.

The formulation is similar to that proposed in [BIP-taproot], with the
insertion of alternate "dummy" leaves to avoid revealing adjacent
nodes in proofs.

The Merkle Tree's leaves are, in TLV-ascending order:
1. The SHA256 of: `LnLeaf` followed by the TLV entry.
2. The SHA256 of: `LnAll` followed all non-signature TLV entries appended in ascending order.

The Merkle tree inner nodes are SHA256('LnBranch' | lesser-SHA256 |
greater-SHA256); this ordering means that proofs are more compact
since left/right is inherently determined.

If there are not exactly a power of 2 leaves, then the tree depth will
be uneven, with the deepest tree on the lowest-order leaves.

e.g. consider the encoding of an `lno` form with TLVs TLV1, TLV2 and TLV3:

```
LALL=SHA256(`LnAll`|TLV1|TLV2|TLV3) 
L1=SHA256(`LnLeaf`|TLV1)
L2=SHA256(`LnLeaf`|TLV2)
L3=SHA256(`LnLeaf`|TLV3)

Assume L1 < LALL, and L2 > LALL.

   L1    LALL                        L2   LALL                   L3
     \   /                             \   /                     |
      v v                               v v                      v
L1A=SHA256('LnBranch'|L1|LALL)  L2A=SHA256('LnBranch'|LALL|L2)   L3
                 
Assume L1A < L2A:

       L1A   L2A                                 L3
         \   /                                    |
	      v v                                     v
  L1A2A=SHA256('LnBranch'|L1A|L2A)               L3
  
Assume L1A2A > L3:

  L1A2A=SHA256('LnBranch'|L1A|L2A)      L3
                          \            /
                           v          v
                Root=SHA256('LnBranch'|L3|L1A2A)

Signature = SIG(SHA256('LnPrefixlno' | Root), nodekey)
```

## Rationale

FIXME: some taproot, some about obscuring leaves in merkle proofs.

# Offers

Offers are a precursor to an invoice: readers will usually request an invoice
(or multiple) based on the offer.  An offer can be much longer-lived than a
particular invoice, so has some different characteristics; in particular it
can be recurring, and the amount can be in a non-lightning currency.  It's
also designed for compactness, to easily fit inside a QR code.

The designated prefix for offers is `lno`.

## TLV Fields for Offers

1. tlvs: `offer_tlvs`
2. types:
    1. type: 2 (`chains`)
    2. data:
        * [`...*chain_hash`:`chains`]
    1. type: 6 (`currency`)
    2. data:
        * [`...*byte`:`iso4217`]
    1. type: 8 (`amount`)
    2. data:
        * [`tu64`:`amount`]
	1. type: 10 (`description`)
    2. data:
		* [`...*byte`:`description`]
    1. type: 12 (`features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 14 (`expiry_timestamp`)
    2. data:
        * [`tu64`:`expiry_timestamp`]
    1. type: 16 (`blindedpath`)
    2. data:
        * [`point`:`blinding`]
        * [`...*onionmsg_path`:`path`]
	1. type: 20 (`vendor`)
    2. data:
		* [`...*byte`:`vendor`]
	1. type: 22 (`quantity_min`)
    2. data:
		* [`tu64`:`min`]
	1. type: 24 (`quantity_max`)
    2. data:
		* [`tu64`:`max`]
    1. type: 26 (`recurrence`)
    2. data:
	    * [`byte`:`time_unit`]
		* [`u32`:`period`]
		* [`tu32`:`limit`]
    1. type: 28 (`recurrence_base`)
    2. data:
		* [`u32`:`basetime`]
		* [`tu32`:`paywindow`]
    1. type: 30 (`node_id`)
    2. data:
        * [`pubkey32`:`node_id`]
    1. type: 240 (`signature`)
    2. data:
        * [`signature`:`sig`]

## Recurrence

Some offers are *periodic*, such as a subscription service or monthly
dues, in that payment is expected to be repeated.  There are many
different flavors of repetition, consider:

* Payments due on the first of every month, for 6 months.
* Payments due on every Monday, 1pm Pacific Standard Time.
* Payments due once a year:
   * which must be made on January 1st, or
   * which are only valid if started January 1st 2021, or
   * which if paid after January 1st you (over) pay the full rate first year, or
   * which if paid after January 1st are paid pro-rata for the first year, or
   * which repeat from whenever you made the first payment

Thus, each payment has:
1. A `time_unit` defining 0 (seconds), 1 (days), 2 (months), 3 (years).
2. A `period`, defining how often (in `time_unit`) it has to be paid.
3. An optional `number` of total payments to be paid (0 meaning unlimited).
4. An optional `basetime`, defining when the first payment applies
   in seconds since 1970-01-01 UTC.
5. An optional `paywindow`, defining how many seconds into the period
   a payment will be accepted: 0xFFFFFFFF being a special value meaning
   "any time during the period, but you will have to pay proportionally
   to the remaining time in the period".

Note that the `expiry_timestamp` field already covers the case where an offer
is no longer valid after January 1st 2021.

## Requirements For Offers

A writer of an offer:
  - MUST set `node_id` to the public key of the node to request the invoice from.
  - MUST specify exactly one signature TLV: `signature`.
    - MUST set `sig` to the signature using `node_id` as described in [Signature Calculation](#signature-calculation).
  - MUST set `description` to a complete description of the purpose
    of the payment.
  - if the chain for the invoice is not solely bitcoin:
    - MUST specify `chains` the offer is valid for.
  - otherwise:
    - the bitcoin chain is implied as the first and only entry.
  - if a specific minimum `amount` is required for successful payment:
	- MUST specify `amount` to the amount expected (per item).
    - if the currency for `amount` is that of the first entry in `chains`:
	  - MUST specify `amount` in multiples of the minimum lightning-payable unit
        (e.g. milli-satoshis for bitcoin).
	- otherwise:
	  - MUST specify `iso4217` as an ISO 4712 three-letter code.
      - MUST specify `amount` in the currency unit adjusted by the ISO 4712
        exponent (e.g. USD cents).
  - if it supports offer features:
    - SHOULD set `features` to the bitmap of offer features.
  - if the offer expires:
    - MUST set `expiry_timestamp` to the number of seconds after midnight 1
      January 1970, UTC that invoice_request should not be attempted.
  - if it is connected only by private channels:
    - MUST include `blindedpath` containing one or more paths to the node from
	  publicly reachable nodes.
  - otherwise:
    - MAY include `blindedpath`.
  - if it sets `vendor`:
    - MUST set it to a valid UTF-8 string.
    - SHOULD set it to clearly identify the issuer of the invoice.
  - if it can supply more than one item for a single invoice
	- if the minimum quantity is more than 1:
	  - MUST set that minimum in `quantity_min`
    - if the maximum quantity is known:
	  - MUST set that maximum in `quantity_max`
    - if neither:
	  - MUST set `quantity_min` to 1 to indicate `quantity` is supported.
  - MAY include `recurrence` to indicate offer should trigger time-spaced
    invoices.
  - if it includes `recurrence`:
      - MUST set `time_unit` to 0 (seconds), 1 (days), 2 (months), 3 (years).
	  - MUST set `period` to how often (per `time-unit`) it wants to be paid.
	  - if there is a maximum number of payments:
	    - MUST set `limit` to the maximum number of payments
	  - otherwise
	    - MUST set `limit` to 0.
	  - MAY include `recurrence_base` if invoices are expected at
        particular time offsets.
  - otherwise:
    - MUST NOT include `recurrence_base`.

A reader of an offer:
  - SHOULD gain user consent for recurring payments.
  - SHOULD allow user to view and cancel recurring payments.
  - if it uses `amount` to provide the user with a cost estimate:
	- MUST warn user if amount of actual invoice differs significantly
		from that expectation.
  - FIXME: more!

# Invoice Requests

Invoice Requests are a request for an invoice; the designated prefix for
invoices is `lnr`.

These can be spontaneous, such as a refund or exchange withdrawal, or in
response to an offer (usually via an `onion_message` `invoice_request` field).

## TLV Fields for `invoice_request`

1. tlvs: `invoice_request_tlvs`
2. types:
    1. type: 2 (`chains`)
    2. data:
        * [`...*chain_hash`:`chains`]
    1. type: 4 (`offer_id`)
    2. data:
        * [`sha256`:`offer_id`]
    1. type: 8 (`amount`)
    2. data:
        * [`tu64`:`amount`]
	1. type: 10 (`description`)
    2. data:
		* [`...*byte`:`description`]
    1. type: 12 (`features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 16 (`blindedpath`)
    2. data:
        * [`point`:`blinding`]
        * [`...*onionmsg_path`:`path`]
	1. type: 32 (`quantity`)
    2. data:
		* [`tu64`:`quantity`]
	1. type: 34 (`refund_for`)
    2. data:
        * [`sha256`:`refunded_payment_hash`]
    1. type: 36 (`invoice_request_recurrence`)
    2. data:
       * [`tu64`:`counter`]
    1. type: 38 (`payer_key`)
    2. data:
        * [`pubkey32`:`key`]

## Requirements for Invoice_Requests

The writer of an invoice_request:
  - MUST set `payer_key` to a transient public key.
  - MUST remember the secret key corresponding to `payer_key`.
  - if the invoice_request is a response to an offer:
    - MUST set `offer_id` to the merkle root of the offer as described in [Signature Calculation](#signature-calculation).
	- MUST NOT set or imply any `chain_hash` not set or implied by the offer.
	- MUST NOT set `description`.
	- if the offer had a `quantity_min` or `quantity_max` field:
	  - MUST set `quantity`
	  - MUST set it within that (inclusive) range.
    - otherwise:
	  - MUST NOT set `quantity`
  - otherwise (not responding to an offer):
    - MUST set `description` to a complete description of the purpose
      of the payment.

  - if there was no corresponding offer, or the offer did not specify `amount`:
    - MUST specify `amount` in multiples of the minimum lightning-payable unit
      (e.g. milli-satoshis for bitcoin) for the first `chains` entry.
  - otherwise:
    - MUST NOT set `amount`
  - if there was a corresponding offer, and the offer contained `recurrence`:
    - MUST set `recurrence` `index` to 0 for the first 
      invoice request, and increment it for each successful invoice paid.
    - MUST use the same `payer_key` for all recurring payments of
      this offer.
  - otherwise:
    - MUST NOT set `recurrence`.
  - if the invoice_request is for a partial or full refund for a previously-paid
    invoice:
    - SHOULD set `refunded_payment_hash` to the `payment_hash` of that
      invoice.
  - otherwise:
    - MUST NOT set `refunded_payment_hash`.


# Invoices

Invoices are a request for payment, and when the payment is made they
it can be combined with the invoice to form a cryptographic receipt.

The designated prefix for invoices is `lni`.  It can be sent in response
to an `invoice_request` using `onion_message` `invoice` field.

1. tlvs: `invoice_tlvs`
2. types:
    1. type: 2 (`chains`)
    2. data:
        * [`...*chain_hash`:`chains`]
    1. type: 4 (`offer_id`)
    2. data:
        * [`sha256`:`offer_id`]
    1. type: 8 (`amount`)
    2. data:
        * [`tu64`:`amount`]
	1. type: 10 (`description`)
    2. data:
		* [`...*byte`:`description`]
    1. type: 12 (`features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 16 (`blindedpath`)
    2. data:
        * [`point`:`blinding`]
        * [`...*onionmsg_path`:`path`]
    1. type: 18 (`blindedpay`)
    2. data:
        * [`...*blinded_payinfo`:`payinfo`]
	1. type: 20 (`vendor`)
    2. data:
		* [`...*byte`:`vendor`]
    1. type: 30 (`node_id`)
    2. data:
        * [`pubkey32`:`node_id`]
	1. type: 32 (`quantity`)
    2. data:
		* [`tu64`:`quantity`]
	1. type: 34 (`refund_for`)
    2. data:
        * [`sha256`:`refunded_payment_hash`]
    1. type: 38 (`payer_key`)
    2. data:
        * [`pubkey32`:`key`]
    1. type: 40 (`timestamp`)
    2. data:
        * [`tu32`:`timestamp`]
    1. type: 42 (`payment_hash`)
    2. data:
        * [`sha256`:`payment_hash`]
    1. type: 44 (`expiry`)
    2. data:
        * [`tu32`:`expiry_seconds`]
    1. type: 46 (`cltv`)
    2. data:
        * [`tu32`:`min_final_cltv_expiry`]
    1. type: 48 (`fallbacks`)
    2. data:
	    * [`u8`:`num`]
		* [`num*fallback_address`:`fallbacks`]
    1. type: 50 (`routehelp`)
    2. data:
		* [`...*payroute`:`routes`]
	1. type: 52 (`refund_signature`)
    2. data:
        * [`signature`:`payer_signature`]
    1. type: 240 (`signature`)
    2. data:
        * [`signature`:`sig`]

1. subtype: `payroute`
2. data:
   * [`point`:`blinding`]
   * [`u16`:`num_hops`
   * [`onionmsg_path`:`num_hops*path`]
   * [`blinded_payinfo`:`num_hops*payinfo`]

1. subtype: `blinded_payinfo`
2. data:
   * [`u32`:`fee_base_msat`]
   * [`u32`:`fee_proportional_millionths`]
   * [`u16`:`cltv_expiry_delta`]
   * [`u16`:`flen`]
   * [`flen*byte`:`features`]

1. subtype: `fallback_address`
2. data:
   * [`byte`:`type`]
   * [`u16`:`len`]
   * [`len*byte`:`address`]

## Requirements

A writer of an invoice:
  - if the `invoice_request` contained the same `offer_id`, `payer_key` and `recurrence` (if any) as a previous `invoice_request`:
    - MAY simply reuse the previous invoice.
  - otherwise:
    - MUST NOT reuse a previous invoice.
  - MUST set `timestamp` to the number of seconds since Midnight 1
    January 1970, UTC.
  - MUST set `payment_hash` to the SHA2 256-bit hash of the
    `payment_preimage` that will be given in return for payment.
  - MUST set `node_id` to the public key of the node to pay the invoice to.
  - MUST specify exactly one signature TLV: `signature`.
    - MUST set `sig` to the signature using `node_id` as described in [Signature Calculation](#signature-calculation).
  - if the chain for the invoice is not solely bitcoin:
    - MUST specify `chains` the offer is valid for.
  - otherwise:
    - the bitcoin chain is implied as the first and only entry.
  - if it has bolt9 features:
    - MUST set `features` to the bitmap of features.
  - if the expiry for accepting payment is not 604800 seconds after `timestamp`:
    - MUST set `expiry` to the number of seconds after `timestamp`
      that payment should not be attempted.
  - if the `min_final_cltv_expiry` for the last HTLC in the route is not 9:
    - MUST set `min_final_cltv_expiry`.
  - if it accepts onchain payments:
    - MAY specify `fallbacks`
    - MUST specify `fallbacks` in order of most-preferred to least-preferred
      if it has a preference.
    - for currency `bc`, it MUST set each `fallback_address` to one of:
      - `type` to a valid witness version and `address` to a valid
        witness program
      - `type` to `17` and `address` to a public key hash
      - `type` to `18` and `address` to a script hash.
  - if it is connected only by private channels:
    - MUST include a `blindedpath` containing one or more paths to the node.
  - otherwise:
    - MAY include `blindedpath`.
  - if it includes `blindedpath`:
    - MUST specify `path` in order of most-preferred to least-preferred if
      it has a preference.
    - MUST include `blinded_payinfo` with exactly one `payinfo` for
	  each `blindedpath` entry, in order.
  - otherwise:
    - MUST NOT include `blinded_payinfo`.
  - MUST set (or not set) `offer_id` exactly as the invoice_request did.
  - MUST set (or not set) `quantity` exactly as the invoice_request did.
  - MUST set (or not set) `refund_for` exactly as the invoice_request did.
  - MUST set (or not set) `recurrence` exactly as the invoice_request did.
  - MUST set `payer_key` exactly as the invoice_request did.
  - if it sets `refund_for`:
    - MUST set `refund_signature` to the signature of the
      `refunded_payment_hash` using the `payer_key`.
  - if it sets `offer_id`:
    - MUST set `vendor` exactly as the offer did.
    - MUST begin `description` with the `description` from the offer.
    - MAY append additional information to `description` (e.g. " +shipping").

  - if the invoice_request specified an `amount`:
    - MUST specify the same `amount`.
  - otherwise:
    - MUST specify `amount` in multiples of the minimum lightning-payable unit
      (e.g. milli-satoshis for bitcoin) for the first `chains` entry.
    - SHOULD derive `amount` using the `amount` and `currency` from
      the offer, and `quantity` from the invoice_request.

A reader of an invoice:

  - FIXME

## Rationale

Because the messaging layer is unreliable, it's quite possible to
receive multiple requests for the same offer.  As it's the caller's
responsibility not to reuse `payer_key` except for recurring invoices,
the writer doesn't have to check all the fields are duplicates before
simply returning a previous invoice.

The invoice duplicates fields rather than committing to the previous offer or
invoice_request.  This flattened format simplifies storage at some space cost, as
the payer need only remember the invoice for any refunds or proof.


FIXME: Possible future extensions:

1. The offer can require delivery info in the invreq.
2. An offer can be updated: the response to an invreq is another offer,
   perhaps with a signature from the original node_id
3. Any empty TLV fields can mean the value is supposed to be known by
   other means (i.e. transport-specific), but is still hashed for sig.

[1] https://www.youtube.com/watch?v=4SYc_flMnMQ
