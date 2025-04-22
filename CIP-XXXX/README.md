---
CIP: ?
Title: Payload Codecs Dispatch on the Protocol Version
Category: Consensus
Status: Draft
Authors:
    - Nicolas Frisby <nick.frisby@iohk.io> <nicolas.frisby@moduscreate.com>
Implementors: []
Discussions:
    - https://github.com/cardano-foundation/CIPs/pull/?
Created: 2025-04-21
License: CC-BY-4.0
---

# Abstract

Today's codecs for messages exchanged by Cardano nodes begin with an integer value that unnecessarily depends on implementation details of the today's Cardano node.
This CIP proposes to replace that integer with the major component of the protocol version, something all node implementations must necessarily agree on.

# Motivation: why is this CIP necessary?

When Cardano hard forked from Byron to Shelley, the node's codecs needed to differentiate between the two kinds of message payload: Byron blocks versus Shelley blocks, Byron transactions versus Shelley transactions, etc.
Since the codebase with the Shelley details was defined as separately as possible from the Byron codebase, this separation was preserved in the codecs: every object's codec now involved a wrapper that begins with an integer tag, 0 for Byron and 1 for Shelley (one slight exception explained below).
This new wrapper and tag are referred to as the _Hard Fork Combinator wrapper/tag_ or more often the _HFC wrapper/tag_.

All Cardano node implementations must agree on the message codecs in order to communicate, but right now the HFC wrapper codec depends on an implementation detail of today's Cardano node.
That dependence is unnecessary and introduces undue complexity in codec specifications such as the [Cardano Blueprint](https://github.com/cardano-scaling/cardano-blueprint) and alternative node implementations.

# Specification

This CIP proposes one principal change and some follow-up changes.

- **Principal Change**.
  The HFC tag's current values are reflective of how the implementation of today's Cardano node has (so far) organized its code for the various protocol versions.
This CIP would redefine the actual values within the HFC tag to simply be the major component of the protocol version.
Alternative node implementations might organize their code differently, but the protocol version is a principal piece of the ledger state, something all node implementations must necessarily agree on regardless of how their code is organized.

- **Follow-Up Change: Storage Layer**.
  The Cardano node currently stores each block's HFC tag alongside it in the on-disk block database.
  As part of this CIP, that annotation would also change to (or at least additionally include) the protocol version, since that's the datum the node needs to send alongside blocks to other (new) nodes.

- **Follow-Up Change: Backwards-Compatibility Conversions**.
  New nodes will need to demote protocol versions to HFC tags to send payloads to old nodes and promote HFC tags received from old nodes.
  - The demotion logic is trivial, since every protocol version maps unambiguously to today's HFC tags: it's simply that protocol version's corresponding index into the Cardano node's [sum type](https://en.wikipedia.org/wiki/Tagged_union).
  - The promotion for headers and blocks is trivial, since they identify both a ledger state (via prev-hash) and a slot that extends that ledger state, which together exactly determine a protocol version.
  - The promotion for queries is trivial, since [`MsgAcquire`](https://github.com/IntersectMBO/ouroboros-network/blob/a487121fed89283d52da8efda4044aa361a3a3e5/ouroboros-network-protocols/src/Ouroboros/Network/Protocol/LocalStateQuery/Type.hs#L139-L144) specifies a ledger state whose protocol version should be used to decode the query.
    Query responses never require promotion, since the node only sends them without ever receiving them.
  - The promotion of transactions is similar to queries.
    The node should simply decode the transaction according to the protocol version of the ledger state that it's attempting to apply the transaction to (in the mempool).
    This does introduce some ambiguity near an epoch boundary that increments the protocol version, but the Ledger Team is careful to avoid ambiguity amongst the codecs for adjacent protocol versions.

- **Follow-Up Change: Transactions**.
  This CIP would remove the HFC wrapper for transactions from messages that are submitting transactions to a node.
  (TODO This change could happen without the rest of this CIP.)

# Rationale: how does this CIP achieve its goals?

There are no competing proposals in this same design space, so each change in the "Specification" section above can be judged on its inherent benefits.

## Principal Change & Follow-Up Change: Backwards-Compatibility Conversions

This CIP removes some details of one implementation from the codecs that all implementations must agree on.
It also specifies how the change would be rolled out such that new nodes could continue interoperating with old nodes.

The protocol version is the canonical value to use for distinguishing codec evolutions.
Using other values for the HFC tag would either be excessively granular or be some abstraction of the protocol version.
Today's HFC values are such an abstraction, but not necessarily a fundamental one that every node implementations would necessarily have independently decided upon.
Even if every node implementation did choose to use the same sum type as today's node does, the codec specification itself can be much simpler if it doesn't have to refer to/motivate that sum type.

One apparent alternative choice would be to remove the HFC wrapper entirely.
However, this would reduce modularity.
Because the HFC wrapper controls the prefix all of the node's codecs, the wrapped Byron and Shelley codecs can exist completely independently from each other---the HFC wrapper completely disambiguates them.
The same is true for the later hard forks to Allegra, Mary, Alonzo, and so on: each one is assigned a unique value for the HFC tag.
While the wrapper allows the wrapped codecs to be independent, it doesn't force them to be.
Codecs are still free to share code, which has been useful and reasonable for all the relatively-incremental changes that hvae been made since the big Byron-to-Shelley hard fork.
But a big change in the future (perhaps Ouroboros Leios?) might benefit from the independence the HFC wrapper ensures.

## Follow-Up Change: Storage Layer

Altering the database schema has consequences for tools (such as Mithril) that currently depend on the exact content of the node's private on-disk files --- this kind of disruption is [a known issue](https://github.com/cardano-foundation/CIPs/pull/974#issuecomment-2624935823) with the current Mithril design.
Making the on-disk data more objective will make future disruptions to such tools less likely.

In fact, on-disk blocks (and so blocks arriving via BlockFetch) are the slight exception foreshadowed in the "Motivation" section above.
Even before Shelley, Byron blocks were already wrapped with a tag that differentiated Epoch Boundary Blocks (0) from regular blocks (1), so the new option of Shelley (2) was merely added to that existing tag instead of nesting the old Byron wrapper within the new wrapper.
In particular, combining those two wrappers avoided needing to migrate the node's on-disk files to a new schema.
But since this CIP requires a change to the schema, it's also an opportunity to eliminate that special case: if a schema change is already happening, the marginal overhead of nested wrappers for Byron no longer justifies the special case.

None of these tags --- even the original Byron block tag --- have ever influenced hashes, so it is possible for their schema to change; it's simply never happened before.

## Follow-Up Change: Transactions

Removing the HFC wrapper from transactions is an improvement in a few ways.
- Just as today's HFC tags leak implementation details into the codecs, today's `cardano-cli` command-line interface leaks those same implementation details by forcing the user to pick the HFC tag for their transaction.
  (This is not apparent as a burden for users today, since there's essentially only one CLI interface for Cardano.)
- As explained in the "Follow-Up Change: Backwards-Compatibility Conversions" bullet above, the node doesn't actually need an HFC wrapper for incoming transactions: the current state of the mempool determines which protocol version to use when decoding the bytes of the incoming transaction.
- It would simplify the specification for submitting transactions.
  Some users have written their own code to do so, and some have reported [difficulties](https://github.com/IntersectMBO/ouroboros-consensus/issues/137).

If the Ledger Team does accidentally leave ambiguity among the codecs of two adjacent protocol versions, then there would be a risk of misinterpreting a submitted transaction near a protocol version boundary.
Even today, however, that (unlikely) risk instead motivates a dedicated new field specifying the intended protocol versions that is within the scope of the transaction's signature to prevent the adversary from tampering---the HFC wrapper itself is the wrong approach to mitigating that risk, since it's not signed.

# Path to Active

## Acceptance Criteria

- The latest release of all Cardano node implementations:
    - switches on the protocol version instead of switching on some abstraction of the protocol version,
    - stores annotations alongside their persisted blocks with that same protocol version instead of some abstraction thereof,
    - and disallows HFC wrappers for submitted transactions.
- All nodes also maintain the older codecs for backwards-compatibility for at least one year or until the next hard fork, whichever comes first.
  Beware: if the next hard fork comes soon after this CIP is implemented, then all node implementations would be forced to implement this CIP before being able to follow that hard fork, since the negotiated handshake versions fully sequentialize _all_ major changes to the node's messages across _all_ node implementations (TODO same concern re-raised in the next section).

## Implementation Plan

Each node implementation team would implement this CIP in their own node.
Because all implementations currently constrain the single integer negotiatied as of the node's handshake, whichever value of that integer is assigned to this CIP's changes would mean that a node implementation would be forced to implement this CIP before negotiating any greater version.
Therefore subsequent features that require greater handshake versions would be blocked by the implementation of this CIP, once it is assigned to a value of that integer.
(TODO has this challenge already been explicitly called out anywhere?
Perhaps we should branch the negotated version so that it "knows" about different implementations?
That doesn't sound sustainable either...)

# Copyright

This CIP is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/legalcode).
