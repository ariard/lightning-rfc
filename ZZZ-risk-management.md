# Recommendations on Contract Risks Management

This document describes the computation of contract risk coverage requirements
inside the Staking Credentials framework.

The data sources, set of events and algorithms for dynamic computation of `credential-to-liquidity`
ratio for the ContractProvider entity are laid out.

This draft is version 0.1 of the contract risk management recommendations.

## Contract fees vs Risk

The Staking Credentials framework aims to provide coverage for the non-payment of
contract fees between a Client and a ContractProvider. Bitcoin contracting protocols
rely fundamentally on the allocation of scarce resources (i.e satoshis liquidity)
between private entities. While the contracting protocols in themselves (e.g lightning
channels) are peer-to-peer and does not assume a priviliged counterparty, in real-world
deployment the execution of the contract often exposes a counterparty to a liquidity
risk (also called timevalue Dos or griefing attack).

This execution risk can be mitigated by a reputation strategy (where "honest" counterparty
are filtered before to enter into a contract) or a monetary strategy (where a risk fee
is paid before the contract funding).

If a risk fee is choosed as a mitigation strategy, the computation of this risk itself
stays an open question, as both counterparty track record and market force must be
processed by the ContractProvider in setting its `credential-to-liquidity` ratio
for its covered contracts.

TODO: introduce valuation process ?

### Gossip Monitoring

The gossip extension `credential_policy` and `contract_policy` messages are indicating
the liquidity risk price of each channel market-wide. There are market forces a ContractProvider
should account to bound its own risk levels.

### Channel Topology

In the context of HTLC forwarding, the local channel topology is a source of information
on the quality of the HTLC traffic, as if the local liquidity management is efficient, low-yield
channels should be closed.

### Channel Congestion

In the context of HTLC forwarding, the congestion rate and failure rate are indicative of the average
risk encumbered by a ContractProvider. Abnormal rate of congestion should be indicator of a
channel jamming attack, and therefore dynamically reflected by bumping the risk coverage requested.

### Mempool Congestion

In the context of HTLC forwarding, if the HTLC are stuck in flight, the cost of on-chain fees
to close channels is inflated and therefore the level of blockspace demand should be accounted
to request adequate risk coverage.
