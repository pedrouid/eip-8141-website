# Mempool Strategy

---

## TL;DR

EIP-8141 separates **what is consensus-valid** (programmable, broad) from **what the public mempool propagates** (constrained, opinionated). The proposed strategy uses two tiers in parallel:

- **Restrictive mempool** ships in clients first. Covers status-quo accounts, bounded custom validation, protocol-validated outer signatures, and gas sponsorship via canonical or low-volume non-canonical paymasters. This is the [public mempool policy already specified](/current-spec#mempool-policy) in the current spec.
- **Expansive mempool** develops in parallel, opt-in per node. Built on the [ERC-7562](https://eips.ethereum.org/EIPS/eip-7562) lineage. Handles privacy protocols and complex validation. FOCIL nodes default to the restrictive set and may add expansive channels at their discretion.

For state, the proposal extends the [VOPS](/vops-compatibility#state-growth-at-scale) baseline to include **nonce, balance, code, and the first 4 storage slots per account**. Use cases outside this extension pay an explicit cost: a **merkle branch** (4-8 kB today, 1-2 kB after binary tree migration) per extra-VOPS state item. See the [AA-VOPS thread](https://ethresear.ch/t/a-pragmatic-path-towards-validity-only-partial-statelessness-vops/22236) for origin.

The framing is borrowed from Bitcoin: **let consensus rules do a lot, but restrict at the mempool layer**. Bitcoin Core's policy rules are upgradable without hardforks, which is why Bitcoin has evolved for 15+ years while keeping the consensus footprint narrow. EIP-8141 applies the same pattern.

A consequence: **the proposal claims frame transactions reduce the structural need for off-chain relayers**. Public ERC-20 sponsorship can be expressed as a non-canonical paymaster that checks frame shape and accepts frontrunning risk, while fully trustless balance-checking variants route through the expansive tier or private submission because they read external token storage. See [ERC-20 gas repayment](#erc20-paymaster-patterns).

---

## Two Tiers in One Mempool

> **Note**: The restrictive tier is in the current EIP-8141 spec. The expansive tier is a proposed framework, not a shipped feature.

A frame transaction that does not match the mempool rules is still **consensus-valid on-chain**. It just cannot be gossiped through the public p2p network. This separation makes the two-tier strategy possible.

| Tier | What it carries | Validation cost | Status |
|---|---|---|---|
| **Restrictive** | Status-quo accounts, protocol-validated outer signatures, canonical paymaster, non-canonical (1 pending tx) | Bounded: 4 prefix shapes, 100k gas including intrinsic signature validation, banned opcodes, sender-only reads | **Specified in EIP-8141** |
| **Expansive** | Privacy protocols, multi-paymaster, arbitrary VERIFY policies | Higher: ERC-7562 staking/reputation, full simulation | **Proposed**, opt-in per node |

---

## Restrictive Mempool: What Ships First

The restrictive tier covers a small surface:

- **Self relay**: account validates itself and pays its own gas
- **Canonical paymaster sponsorship (ETH-funded)**: a paymaster matching the canonical runtime code pays gas from its own ETH balance
- **Account deployment**: deploy frame as the first frame, targeting any factory whose execution satisfies the deploy-frame trace rules. EIP-7997 is the canonical-but-non-mandatory factory after PR #11567 (merged Apr 30) dropped it from `requires` and rewrote the rule as a stateless-trace policy.
- **Non-canonical paymaster**: bounded to 1 pending tx per paymaster

Validation constraints: one of four prefix shapes, validation-prefix gas plus intrinsic signature-validation cost no more than 100k, banned opcodes (ORIGIN, TIMESTAMP, BLOCKHASH, BALANCE, SSTORE, etc.), storage reads only on `tx.sender`, no calls to non-existent contracts, matching approval-scope flags, no atomic-batch flag inside the validation prefix, and no VERIFY frame after the validation prefix. `CREATE`, `CREATE2`, and `SETDELEGATE` (EIP-7819) are banned outside the first deploy frame and allowed inside it for installing code at `tx.sender` (PR #11567); `SSTORE`s on `tx.sender`'s storage are also allowed inside the deploy frame.

**Expiry-verifier frames** (PR #11662, merged May 14; tightened by #11814 on Jul 7) are admitted as a special case: a `VERIFY` frame targeting `EXPIRY_VERIFIER = address(0x8141)` carries an 8-byte unix-seconds deadline as `frame.data`, may appear only as the first frame, and is exempt from validation trace rules, storage-dependency tracking, and `MAX_VERIFY_GAS`. The `TIMESTAMP` opcode is permitted only for the canonical runtime code at this address. Public-mempool nodes MUST drop transactions whose expiry is in the past relative to their current view of `block.timestamp`. Validation-prefix shape matching treats expiry-verifier frames as transparent (`[expiry_verify, self_verify]` matches `[self_verify]`).

What this enables: default-code secp256k1 EOAs, smart accounts with bounded validation, protocol-validated outer signatures (`SECP256K1` and `P256`), `ARBITRARY` witness bytes for custom schemes, ETH-funded gas sponsorship via the canonical paymaster, and non-canonical sponsors that fit the one-pending-tx cap.

### ERC-20 gas repayment {#erc20-paymaster-patterns}

"Pay gas in ERC-20" is not covered by the canonical paymaster; it only covers ETH-funded sponsorship. EIP-8141 still supports ERC-20 repayment through non-canonical paymasters, with a clear public/private split.

<div class="balanced-columns">

| Dimension | Public ERC-20 sponsor | Trustless balance-checking sponsor |
|---|---|---|
| VERIFY logic | Checks sponsor authorization or signature metadata, then inspects the next SENDER frame for the ERC-20 transfer shape | Introspects frames and reads ERC-20 contract storage to confirm the user can pay |
| Offchain service | Optional; the sponsor can pre-authorize terms offchain | None |
| Mempool tier | **Restrictive** (public), as a non-canonical paymaster subject to `MAX_PENDING_TXS_USING_NON_CANONICAL_PAYMASTER = 1` per paymaster | **Expansive**, private mempool, or direct-to-builder submission |
| VOPS-compatible | Yes, if it avoids external token storage reads during validation | No (reads state outside the VOPS+4 slice) |
| Trust model | Sponsor absorbs frontrunning risk: the user can zero the ERC-20 balance before inclusion | Trustless with respect to token balance, but not publicly propagatable under the restrictive tier |
| Unique to EIP-8141 | No (any signature-checking paymaster works this way) | Yes (enabled by frame introspection) |

</div>

Example 5 in the spec ([EOA Paying Gas in ERC-20s](/current-spec#5-eoa-paying-gas-in-erc-20s)) is now the public sponsor shape: it checks signature metadata and the next ERC-20 transfer frame, not the user's token balance.

The restriction on the trustless balance-checking pattern is deliberate. The restrictive tier forbids arbitrary external storage reads during validation in order to preserve VOPS compatibility (see [VOPS Compatibility](/vops-compatibility)): a node running with a partial-statelessness slice cannot safely validate a transaction whose inclusion depends on state outside its slice. The current public workaround is economic, not cryptographic: the sponsor prices or limits the risk that the user can drain the token balance before inclusion. Active design directions for bringing stricter variants onto the public mempool in a future revision include **AMM paymaster contracts** (liquidity providers absorb the frontrunning risk in exchange for a fee), **ERC-7562-style validation rules** allowing narrowly-scoped shared-state reads under staking or reputation constraints, and **guarantor payers** (open in [PR #11555](https://github.com/ethereum/EIPs/pull/11555), Apr 22) where a guarantor commits to paying gas even if sender validation fails, letting mempool nodes skip sender simulation entirely and admit VERIFY frames that read shared state. These are open, not specified.

---

## Expansive Mempool: What Develops in Parallel

For use cases exceeding restrictive policy. Principal examples: **privacy protocols** that must read state outside `tx.sender` to verify nullifiers, and **trustless ERC-20 balance-checking paymasters** whose VERIFY frames read ERC-20 contract storage to authorize payment (see [ERC-20 gas repayment](#erc20-paymaster-patterns)).

The expansive tier accepts ERC-7562-style validation with staking/reputation, paymaster-extended policies, and arbitrary VERIFY logic subject to the node's resource budget. It is not a precondition for shipping EIP-8141. Clients ship restrictive first; the privacy/complex-validation community develops expansive independently. No hardfork dependency between the two.

Transactions that do not fit the restrictive tier are still consensus-valid on-chain. They just do not propagate through the public p2p network and must reach a block builder through private channels (direct submission, bundled relays, opt-in expansive-tier peers).

---

## The State Side: VOPS + 4 Slots

The proposed extension for frame transactions:

| State item | Per account | Notes |
|---|---|---|
| Nonce | Already in VOPS | Standard EOA validation |
| Balance | Already in VOPS | Standard EOA validation |
| Code | Already in VOPS | Needed to detect smart accounts |
| **First 4 storage slots** | New | Covers most common AA validation reads |

A small constant-factor increase over the VOPS baseline, covering the validation reads of well-designed AA wallets (signer set, sequencer key, replay-protection counter).

---

## The Merkle Branch Escape Hatch

For use cases reading state outside VOPS+4, the transaction includes a merkle branch proving the state items it reads.

| Property | Today (MPT) | After binary tree |
|---|---|---|
| Branch size per item | 4-8 kB | 1-2 kB |
| Items typically proved | 1-2 | 1-2 |

Status-quo and AA-VOPS-friendly transactions pay zero extra. Privacy protocol transactions pay one or two branches. The infrastructure already exists in clients (witness machinery for sync, standardized RPC methods for users to obtain witnesses).

---

## Resolving the Trilemma

The ["choose 2 of 3" trilemma](/vops-compatibility#the-frames-focil-vops-trilemma) (Frames + FOCIL + VOPS) resolves under this strategy:

| Transaction class | Restrictive? | FOCIL? | VOPS+4? | Extra cost |
|---|---|---|---|---|
| Status-quo accounts | Yes | Yes | Yes | None |
| AA wallets reading 4 own slots | Yes | Yes | Yes | None |
| AA wallets reading > 4 own slots | Yes | Yes | Yes (witness) | 1-2 branches |
| Canonical paymaster (ETH-funded) | Yes | Yes | Yes | None |
| Non-canonical, low volume | Yes (1 pending) | Yes | Yes | None |
| Non-canonical, high volume | No (expansive) | Opt-in | N/A | Expansive tier |
| Public ERC-20 sponsor | Yes (1 pending) | Yes | Yes | Sponsor accepts frontrunning risk |
| Trustless ERC-20 balance-checking sponsor | No (expansive/private) | Opt-in | Yes | Expansive tier or private mempool |
| Privacy protocol | No (expansive) | Opt-in | Yes (witness) | Branches + expansive |

Frames + FOCIL + VOPS coexist for the majority of traffic. Edge cases pay per-tx cost or move to the expansive tier.

---

## Why Frame Transactions Don't Need Relayers

Anything a relayer does for EIP-4337 can be expressed as a pure onchain smart contract under EIP-8141.

**Privacy rebroadcasters** become onchain contracts observing the canonical mempool and repackaging transactions with witness branches. **ERC-20 gas fronting** takes two shapes under EIP-8141: a public non-canonical sponsor that checks frame shape and accepts balance-drain risk, and a trustless balance-checking sponsor that routes through the expansive tier or private submission. See [ERC-20 gas repayment](#erc20-paymaster-patterns).

This is the structural argument against EIP-4337 + EIP-7702: in those designs, the relayer is required because validation does not run in-protocol. EIP-8141 brings validation in-protocol, which the proposal claims removes the structural need for out-of-protocol actors. Whether on-chain substitutes match bundlers' operational properties in practice is an open question.

**External ratification (Apr 27)**: the [Svalbard interop AA breakout](https://hackmd.io/@nixorokish/svalbard-aa-breakout) added "no relayers for core functionality" to the primary-goal list for native AA, and explicitly deferred ERC-20 sponsorship to relayer-dependent paths. The constraint set the breakout converged on (statelessness compatibility, public-mempool admissibility, FOCIL compatibility, ETH-denominated sponsorship as the core case) is the same set this two-tier strategy is designed against. See [Feedback Evolution → Svalbard Interop AA Breakout](/feedback-evolution#svalbard-interop-aa-breakout).

The practical implication is central to the [Developer Tooling bull case](/developer-tooling#bull-case-native-aa-with-powerful-defaults): the wallet adoption cost reduces to "implement a new transaction type."

---

## Open Questions

### Mempool Health and Censorship Resistance

Nodes that cannot validate frame transactions cannot maintain healthy mempools, propagate them, or enforce FOCIL inclusion lists covering them. This degrades to reliance on specialized infrastructure. The two-tier architecture mitigates this: the restrictive tier is validatable by minimal nodes, and FOCIL nodes default to that tier. Whether this is sufficient depends on FOCIL adoption breadth.

### Canonical Paymaster Adoption

The restrictive tier relies on a canonical paymaster recognized by runtime code match. If wallets prefer non-canonical paymasters (richer features, multi-token payment), those transactions are limited to 1 pending tx each and cannot be enforced by FOCIL. Historical precedent: ERC-4337 paymaster diversity is high, with no dominant canonical variant. The expansive tier is the proposed long-term answer, but it is opt-in. This remains market-driven, not protocol-enforceable.

### Encrypted Mempool Compatibility

Encrypted mempools (like [LUCID/EIP-8184](https://eips.ethereum.org/EIPS/eip-8184)) encrypt transaction contents before inclusion. Nodes cannot check even minimal fields for DOS prevention. The proposed answer routes encrypted transactions through the expansive tier and [onchain rebroadcasters](#why-frame-transactions-dont-need-relayers), not the restrictive tier. See also [PQ Roadmap → Stage 4](/pq-roadmap#4-encrypted-mempools).

### Transaction Propagation Fragility

The restrictive tier's propagation guarantee rests on every client implementing the same policy rules. If implementation drift occurs (one client ships early, another is slow, a third adds tweaks), the effective propagation surface shrinks to the intersection. This is a well-known failure mode for non-consensus policy rules. Mitigations are social: reference implementations, conformance tests, client-developer consensus on canonical paymaster bytecode.

### Privacy Pools and the Three Gates {#privacy-pools-three-gates}

Nero_eth's [Three Gates to Privacy](https://ethresear.ch/t/frame-transactions-and-the-three-gates-to-privacy/24666) (Apr 16 2026) argues that shielded-pool withdrawals must pass three independent validation gates to reach a block under current rules, and all three reject them by default:

| Gate | What it checks | Why privacy withdrawals fail |
|---|---|---|
| **Public mempool** | VERIFY ≤ 100k gas, reads bounded to `tx.sender` storage, banned opcodes | Groth16 pairing check ~250k gas exceeds cap; nullifier slots live in the pool contract, not `tx.sender` |
| **FOCIL enforcement** | Per-tx 100k cap and per-IL 250k budget | Cumulative budget fits ~2 frame transactions; append-loop builder compliance imposes quadratic overhead |
| **VOPS / AA-VOPS node validation** | Nodes carry only balance/nonce (VOPS) or a few slots per account (AA-VOPS) | Nullifier lookups are hash-keyed slots in an external contract; no fixed N suffices, AA-VOPS cannot help |

What frame transactions already give privacy flows: invalid-proof and replayed-proof cases both cause `VERIFY` to revert before gas is charged, so a sponsor can be paid from the withdrawn amount itself with zero trust assumption and no out-of-protocol relayer. The gates, not the flow itself, are what block inclusion today.

The post proposes five protocol changes to create a viable inclusion path:

1. Extend the canonical-contract code-hash exemption to recognized privacy pools.
2. Raise the per-transaction `VERIFY` gas cap to ~400k for canonical frames.
3. Adopt validation-index FOCIL enforcement: builders publish `(tx_hash, claimed_index)` pairs and attesters validate once at the claimed state, breaking the append-loop quadratic.
4. Raise `MAX_VERIFY_GAS_PER_INCLUSION_LIST` to 2^20 (1M gas).
5. Relax the bounded state-access rule for canonical privacy pools so nullifier reads are propagatable.

Tradeoff acknowledged in the post: under the validation-index model, attesters absorb up to ~28% of block gas in the worst case within the 4-second attestation deadline. Whether that remains feasible under stressed network conditions is open. The proposal also assumes PS (partially stateful) nodes voluntarily track privacy pools, an infrastructure coordination problem that protocol changes alone do not solve.

See [VOPS Compatibility → Status](/vops-compatibility#status) for the state-side row on privacy-pool reads.

---

## Summary

- **Two-tier mempool** (proposed): Restrictive (in spec, common case) and Expansive (parallel, opt-in, privacy and complex validation).
- **VOPS extension** (proposed): nonce, balance, code, first 4 slots per account.
- **Merkle branch escape hatch** (proposed): 4-8 kB today, 1-2 kB after binary tree. Cost falls on transactions that need it.
- **Trilemma**: Frames + FOCIL + VOPS coexist for majority of traffic. Edge cases pay per-tx cost or use the expansive tier.
- **Bitcoin pattern** (analogy): permissive consensus + restrictive mempool gives upgradability without hardforks.
- **Relayer reduction** (proposed claim): privacy rebroadcasters and trustless ERC-20 balance-checking sponsors are expressible as onchain contracts running through the expansive tier or private mempool. Public ERC-20 sponsorship can propagate through the restrictive mempool as a non-canonical paymaster, but the sponsor accepts frontrunning risk. Whether onchain variants match bundler operational properties is open.
- **Open questions**: canonical paymaster adoption (market-driven), propagation fragility (cross-client alignment), encrypted mempool routing (expansive tier), mempool health (FOCIL adoption), privacy pools and the [three gates](#privacy-pools-three-gates) (canonical-pool exemption + validation-index FOCIL + raised VERIFY caps proposed).

---

## Read Next

- [VOPS Compatibility](/vops-compatibility) — the state-side view of the same trilemma, with plain-English definitions of VOPS, FOCIL, and validation state.
- [Current Spec → Mempool Policy](/current-spec#mempool-policy) — the restrictive tier as it appears in the spec itself.
- [PQ Roadmap](/pq-roadmap) — how the mempool strategy interacts with encrypted mempools and the longer PQ arc.
