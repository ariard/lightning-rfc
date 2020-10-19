# BOLT #12: Flexible Protocol for Lightning Payments

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

There are two basic payment flows supported by BOLT 12:

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
to request the invoice.  In addition, the merkle construction of the BOLT 12
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

All signatures are created as per [BIP-340], and tagged as recommended
there.  Thus to sign a message `msg` with `tag`, `m` is
SHA256(SHA256(`tag`) || SHA256(`tag`) || `msg`).  The notation used
here is `SIG(tag,msg,key)`.

Each form is signed using one or more TLV signature elements; TLV types
240 through 1000 are considered signature elements.  For these the
tag is `LnPrefix` | prefix, and `msg` is the merkle-root; 
the prefix is the designated prefix listed below, e.g. `lno`.

The formulation of the merkle tree is similar to that proposed in
[BIP-taproot], with the insertion of alternate "dummy" leaves to avoid
revealing adjacent nodes in proofs.

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

Signature = SIG('LnPrefixlno', Root, nodekey)
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
    1. type: 16 (`paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
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
        * [`tu32`:`period`]
    1. type: 64 (`recurrence_paywindow`)
    2. data:
        * [`u32`:`seconds_before`]
        * [`byte`:`proportional_amount`]
        * [`tu32`:`seconds_after`]
    1. type: 66 (`recurrence_limit`)
    2. data:
        * [`tu32`:`max_period`]
    1. type: 28 (`recurrence_base`)
    2. data:
        * [`u32`:`basetime`]
        * [`byte`:`start_any_period`]
    1. type: 30 (`node_id`)
    2. data:
        * [`pubkey32`:`node_id`]
    1. type: 240 (`signature`)
    2. data:
        * [`signature`:`sig`]

1. subtype: `blinded_path`
2. data:
   * [`point`:`blinding`]
   * [`u16`:`num_hops`]
   * [`num_hops*onionmsg_path`:`path`]

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
3. An optional `recurrence_limit` of total payments to be paid.
4. An optional `recurrence_base`:
   * `basetime`, defining when the first period starts
      in seconds since 1970-01-01 UTC.
   * `start_any_period` if non-zero, meaning you don't have to start
      paying at the period indicated by `basetime`, but can use
      `recurrence_start` to indicate what period you are starting at.
5. An optional `recurrence_paywindow`:
   * `seconds_before`, defining how many seconds prior to the start of
      the period a payment will be accepted.
   * `proportional_amount`, if set indicating that a payment made during
      the period itself will be charged proportionally to the remaining time
      in the period (e.g. 150 seconds into a 1500 second period gives a 10%
      discount).
   * `seconds_after`, defining how many seconds after the start of the period
      a payment will be accepted.
  If this field is missing, payment will be accepted during the prior period and
  the paid-for period.

Note that the `expiry_timestamp` field already covers the case where an offer
is no longer valid after January 1st 2021.

## Offer Period Calculation

Each period has a zero-based index, and a start time and an end time.
Because the periods can be in non-seconds units, the duration of a
period can depend on when it starts.  The period with index N+1 begins
immediately following the end of period with index N.

- if an offer contains `recurrence_base`:
  - the start of period #0 is `basetime` seconds since 1970-01-01 UTC.
- otherwise:
  - the start of period #0 is the time of issuance of the first
    `invoice` for this particular offer and `payer_key`.

To calculate the start of period #`N` for `N` > 0:
- if `time_unit` is 0:
  - period `N` starts at period #0 start plus `period` multiplied by `N`,
    in seconds.
- otherwise, if `time_unit` is 1:
  - calculate the offset in seconds within the day of period #0 start.
  - add `period` multiplied by `N` days to get the day of the period start.
  - add the offset in seconds to get the period end in seconds.
- otherwise, if `time_unit` is 2:
  - calculate the offset in days within the month of period #0 start.
  - calculate the offset in seconds within the day of period #0 start.
  - add `period` multiplied by `N` months to get the month of the period start.
  - add the offset days to get the day of the period start.
    - if the day is not within the month, use the last day within the month.
  - add the offset seconds to get the period start in seconds.
- otherwise, if `time_unit` is 3:
  - calculate the offset in months within the year of period #0 start.
  - calculate the offset in days within the month of period #0 start.
  - calculate the offset in seconds within the day of period #0 start.
  - add `period` multiplied by `N` years to get the year of the period start.
  - add the offset months to get the month of the period start.
  - add the offset days to get the day of the period start.
    - if the day is not within the month, use the last day within the month.
  - add the offset seconds to get the period start in seconds.
- otherwise, the time is invalid.

Note that offset seconds can overflow only if the period start is in a
leap second; we ignore this!

If a period starts at 10:00:00 (am) UTC on 31 December, 1970 (31485600
seconds since epoch), the following demonstrates period ends:
* time_unit 0:
  - period 1: 31 Dec 1970 10:00:01 UTC (31485601)
  - period 10: 31 Dec 1970 10:00:10 UTC (31485610)
  - period 100: 31 Dec 1970 10:01:40 UTC (31485700)
  - period 1000: 31 Dec 1970 10:16:40 UTC (31486600)
* time_unit 1:
  - period 1: 1 Jan 1971 10:00:00 UTC (31572000)
  - period 10: 10 Jan 1971 10:00:00 UTC (32349600)
  - period 100: 10 Apr 1971 10:00:00 UTC (40125600)
  - period 1000: 26 Sep 1973 10:00:00 UTC (117885600)
* time_unit 2:
  - period 1: 31 Jan 1971 10:00:00 UTC (34164000)
  - period 10: 31 Oct 1971 10:00:00 UTC (57751200)
  - period 100: 30 Apr 1975 10:00:00 UTC (168084000)
* time_unit 3:
  - period 1: 31 Dec 1971 10:00:00 UTC (63021600)
  - period 10: 31 Dec 1980 10:00:00 UTC (347104800)

If a period starts at 10:00:00 (am) UTC on 29 February, 2016 (1456740000):
* time_unit 0:
  - period 1: 29 Feb 2016 10:00:01 UTC (1456740001)
  - period 10: 29 Feb 2016 10:00:10 UTC (1456740010)
  - period 100: 29 Feb 2016 10:01:40 UTC (1456740100)
  - period 1000: 29 Feb 2016 10:16:40 UTC (1456741000)
* time_unit 1:
  - period 1: 1 Mar 2016 10:00:00 UTC (1456826400)
  - period 10: 10 Mar 2016 10:00:00 UTC (1457604000)
  - period 100: 08 Jun 2016 10:00:00 UTC (1465380000)
  - period 1000: 25 Nov 2018 10:00:00 UTC (1543140000)
* time_unit 2:
  - period 1: 29 Mar 2016 10:00:00 UTC (1459245600)
  - period 10: 29 Sep 2016 10:00:00 UTC (1475143200)
  - period 100: 29 Jun 2024 10:00:00 UTC (1719655200)
* time_unit 3:
  - period 1: 28 Feb 2017 10:00:00 UTC (1488276000)
  - period 10: 28 Feb 2026 10:00:00 UTC (1772272800)

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
    - MUST include `paths` containing one or more paths to the node from
      publicly reachable nodes.
  - otherwise:
    - MAY include `paths`.
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
        - MUST include `recurrence_limit` with `max_period` set to the maximum number of payments
        - MUST NOT set `max_period` to 0.
      - otherwise:
        - MUST NOT include `recurrence_limit`.
      - if periods are always at specific time offsets:
        - MUST include `recurrence_base`
        - MUST set `basetime` to the initial period time in number of
          seconds after midnight 1 January 1970
        - if the first paid-for-period does not have to be the initial period:
          - MUST set `start_any_period` to 1.
        - otherwise:
          - MUST set `start_any_period` to 0.
      - otherwise:
        - MUST NOT include `recurrence_base`.
      - if payments will be accepted for the current or next period:
        - MAY include `recurrence_paywindow`
      - otherwise:
        - MUST include `recurrence_paywindow`
      - if it includes `recurrence_paywindow`:
        - MUST set `seconds_before` to the maximum number of seconds prior to
          a period for which it will accept payment for that period.
        - MUST set `seconds_after` to the maximum number of seconds into to a
          period for which it will accept payment for that period.
        - SHOULD NOT set `seconds_after` to greater than the maximum number of
          seconds in a period.
        - if it `amount` is specified and the node will proportionally reduce
          the amount charged for a period payed after the start of the period:
          - MUST set `proportional_amount` to 1 
        - otherwise:
          - MUST set `proportional_amount` to 0
  - otherwise:
    - MUST NOT include `recurrence_base`.
    - MUST NOT include `recurrence_paywindow`.
    - MUST NOT include `recurrence_limit`.

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
        * [`tu64`:`msat`]
    1. type: 10 (`description`)
    2. data:
        * [`...*byte`:`description`]
    1. type: 12 (`features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 16 (`paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
    1. type: 32 (`quantity`)
    2. data:
        * [`tu64`:`quantity`]
    1. type: 34 (`refund_for`)
    2. data:
        * [`sha256`:`refunded_payment_hash`]
    1. type: 36 (`recurrence_counter`)
    2. data:
        * [`tu32`:`counter`]
    1. type: 68 (`recurrence_start`)
    2. data:
        * [`tu32`:`period_offset`]
    1. type: 38 (`payer_key`)
    2. data:
        * [`pubkey32`:`key`]
    1. type: 242 (`recurrence_signature`)
    2. data:
        * [`signature`:`sig`]

## Requirements for Invoice Requests

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
    - MUST specify `amount`.`msat` in multiples of the minimum lightning-payable unit
      (e.g. milli-satoshis for bitcoin) for the first `chains` entry.
  - otherwise:
    - MUST NOT set `amount`
  - if there was a corresponding offer, and the offer contained `recurrence`:
    - for the initial request:
      - MUST use a unique `payer_key`.
      - MUST set `recurrence_counter` `counter` to 0.
    - for any successive requests:
      - MUST use the same `payer_key` as the initial request.
      - MUST set `recurrence_counter` `counter` to one greater than the highest-paid invoice.
    - if the offer contained `recurrence_base` with `start_any_period` non-zero:
      - MUST include `recurrence_start`
      - MUST set `period_offset` to the period the sender wants for the initial request
      - MUST set `period_offset` to the same value on all following requests.
    - otherwise:
      - MUST NOT include `recurrence_start`
    - MUST set `recurrence_signature` `sig` as detailed in
      [Signature Calculation](#signature-calculation) using the `payer_key` 
      and prefix 'recurrence_signature'.
    - SHOULD NOT send an `invoice_request` for a period which has
      already passed.
    - if the offer contains `recurrence_paywindow`:
      - SHOULD NOT send an `invoice_request` for a period prior to `seconds_before` seconds before that period start.
      - SHOULD NOT send an `invoice_request` for a period later than `seconds_after` seconds past that period start.
    - otherwise:
      - SHOULD NOT send an `invoice_request` with `recurrence_counter`
        is non-zero for a period whose immediate predecessor has not
        yet begun.
  - otherwise:
    - MUST NOT set `recurrence_counter`.
    - MUST NOT set `recurrence_signature`.
    - MUST NOT set `recurrence_start`
  - if the invoice_request is for a partial or full refund for a previously-paid
    invoice:
    - SHOULD set `refunded_payment_hash` to the `payment_hash` of that
      invoice.
  - otherwise:
    - MUST NOT set `refunded_payment_hash`.

The reader of an invoice_request:
  - MUST fail the request if `payer_key` is not present.
  - MUST fail the request if `chains` does not include (or imply) a supported chain.
  - MUST fail the request if `features` contains unknown even bits.
  - if `offer_id` is not present:
    - MUST fail the request if `description` is not present.
    - FIXME: refunds and more!
  - otherwise:
    - MUST fail the request if the `offer_id` does not refer to an unexpired offer.
    - if the offer had a `quantity_min` or `quantity_max` field:
      - MUST fail the request if there is no `quantity` field.
      - MUST fail the request if there is `quantity` is not within that (inclusive) range.
    - otherwise:
      - MUST fail the request if there is a `quantity` field.
    - if the offer included `amount`:
      - MUST fail the request if it contains `amount`.
      - MUST calculate the invoice amount using the offer `amount`.
      - if offer `currency` is not the invoice currency, convert to the
        invoice currency.
      - if request contains `quantity`, multiply by `quantity`.
    - otherwise:
      - MUST fail the request if it does not contain `amount`.
      - MUST use the request `amount` as the invoice amount.
    (Note: invoice amount can be further modiifed by recurrence below)
    - if the offer had a `recurrence`:
      - MUST fail the request if there is no `recurrence_counter` field.
      - MUST fail the request if there is no `recurrence_signature` field.
      - MUST fail the request if `recurrence_signature` is not correct.
      - if the offer had `recurrence_base` and `start_any_period` was 1:
        - MUST fail the request if there is no `recurrence_start` field.
        - MUST consider the period index for this request to be the
          `recurrence_start` field plus the `recurrence_counter` `counter`
          field.
      - otherwise:
        - MUST fail the request if there is a `recurrence_start` field.
        - MUST consider the period index for this request to be the
          `recurrence_counter` `counter` field.
      - if the offer has a `recurrence_limit`:
        - MUST fail the request if the period index is greater than `max_period`.
      - MUST calculate the period using the period index as detailed in [Period Calculation](#offer-period-calculation).
      - if `recurrence_counter` is non-zero:
        - MUST fail the request if the no invoice for the previous period
          has been paid.
        - if the offer had a `recurrence_paywindow`:
          - SHOULD fail the request if the current time is before the start of
            the period minus `seconds_before`.
          - SHOULD fail the request if the current time is equal to or after the
            start of the period plus `seconds_after`.
          - if `proportional_amount` is 1:
            - MUST adjust the invoice amount proportional to time remaining in
              the period.
        - otherwise:
          - if `counter` is non-zero:
            - SHOULD fail the request if the current time is prior to the start
              of the previous period.
    - otherwise (the offer had no `recurrence`):
      - MUST fail the request if there is a `recurrence_counter` field.
      - MUST fail the request if there is a `recurrence_signature` field.

## Rationale

We insist that recurring requests be in order (thus, if you pay an
invoice for #34 of a recurring offer, it implicitly commits to the
successful payment of #0 through #33).

The `recurrence_paywindow` constrains how far you can pay in advance
precisely, and if it isn't in the offer the defaults provide some
slack, without allowing commitments into the far future.

To avoid probing (should a payer_key become public in some way), we
require a signature for recurring invoice requests; this ensures that
no third party can determine how many invoices have been paid already.

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
        * [`tu64`:`msat`]
    1. type: 10 (`description`)
    2. data:
        * [`...*byte`:`description`]
    1. type: 12 (`features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 16 (`paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
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
    1. type: 36 (`recurrence_counter`)
    2. data:
       * [`tu32`:`counter`]
    1. type: 68 (`recurrence_start`)
    2. data:
        * [`tu32`:`period_offset`]
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
    1. type: 52 (`refund_signature`)
    2. data:
        * [`signature`:`payer_signature`]
    1. type: 240 (`signature`)
    2. data:
        * [`signature`:`sig`]

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
  - if the `invoice_request` contained the same `offer_id`, `payer_key` and `recurrence_counter` (if any) as a previous `invoice_request`:
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
  - if the invoice corresponds to an offer with `recurrence`:
    - SHOULD set `expiry` to the number of seconds after `timestamp` that
      payment for this period will no longer be accepted.
  - if the expiry for accepting payment is not 7200 seconds after `timestamp`:
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
    - MUST include a `blinded_path` containing one or more paths to the node.
  - otherwise:
    - MAY include `blinded_path`.
  - if it includes `blinded_path`:
    - MUST specify `path` in order of most-preferred to least-preferred if
      it has a preference.
    - MUST include `blinded_payinfo` with exactly one `payinfo` for
      each `onionmsg_path` in `blinded_path`, in order.
  - otherwise:
    - MUST NOT include `blinded_payinfo`.
  - MUST set (or not set) `offer_id` exactly as the invoice_request did.
  - MUST set (or not set) `quantity` exactly as the invoice_request did.
  - MUST set (or not set) `refund_for` exactly as the invoice_request did.
  - MUST set (or not set) `recurrence_counter` exactly as the invoice_request did.
  - MUST set (or not set) `recurrence_start` exactly as the invoice_request did.
  - MUST set `payer_key` exactly as the invoice_request did.
  - if it sets `refund_for`:
    - MUST set `refund_signature` to the signature of the
      `refunded_payment_hash` using the `payer_key`.
  - if it sets `offer_id`:
    - MUST set `vendor` exactly as the offer did.
    - MUST begin `description` with the `description` from the offer.
    - MAY append additional information to `description` (e.g. " +shipping").

  - if the invoice_request specified an `amount`:
    - MUST specify the same `msat`.
  - otherwise:
    - MUST specify `amount`.`msat` in multiples of the minimum lightning-payable unit
      (e.g. milli-satoshis for bitcoin) for the first `chains` entry.
    - SHOULD derive `msat` using the `amount` and `currency` from
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
