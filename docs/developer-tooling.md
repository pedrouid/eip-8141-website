# Developer Tooling

---

## TL;DR

When AA ships without protocol-level defaults, every common feature (batching, sponsorship, permissions) must be standardized as an ERC. Those ERCs converge slowly and often fragment across wallet vendors. EIP-8141 addresses common features through protocol defaults: default EOA code for secp256k1 signing and frame execution, atomic batching in frame flags, and the canonical paymaster for ETH-funded sponsorship. Permissions, session keys, passkeys, and recovery remain outside default code and still need account code or ERC standardization. ERC-20 gas repayment is possible through non-canonical paymasters: the public shape accepts sponsor frontrunning risk, while a trustless balance-checking shape routes through the expansive tier or private mempool (see [ERC-20 gas repayment](/mempool-strategy#erc20-paymaster-patterns)).

---

## Practical Takeaways

**If you're building a wallet:**
- Plan to drop bundler, EntryPoint, and UserOperation infrastructure from the 8141 path. Frame transactions enter the public mempool directly.
- Existing EOA addresses keep working. No migration, no smart-account deployment, and no 7702 delegation required. Default code handles secp256k1 and ETH transfer out of the box. P256/passkeys are protocol-validated in the outer signatures list, but codeless EOA default code does not accept them.
- Session keys, multisig, social recovery, and richer permissions still need account code or ERC standardization. Protocol defaults do not cover these.
- For high-throughput gas sponsorship, prefer the canonical paymaster for the public-mempool path (ETH-funded only). A codeless EOA sponsor can use payment-only default VERIFY at signature index `1` while the sender uses index `0`, but remains subject to the one-pending-transaction non-canonical-paymaster cap.
- **ERC-20 gas repayment has a public/private split.** A public non-canonical sponsor can inspect the next ERC-20 transfer frame and propagate through the restrictive mempool, but it accepts the risk that the user drains the token balance before inclusion. A trustless balance-checking paymaster reads token storage and must route through the expansive tier or a private mempool. See [Mempool Strategy → ERC-20 gas repayment](/mempool-strategy#erc20-paymaster-patterns).

**If you're building an app:**
- Your contracts keep seeing `msg.sender = tx.sender` in SENDER frames, so token approvals, NFT ownership checks, and access control all work unchanged.
- "Approve + swap atomic" is a first-class public-mempool pattern, no RPC negotiation with a specific wallet vendor required.
- "Gas paid in ERC-20" is a valid on-chain pattern with two shapes: public non-canonical sponsors propagate with frontrunning risk, while trustless balance-checking paymasters route through private or expansive paths.
- Post-quantum migration is handled at the account level, not the app level. No app-side changes needed to support users migrating to PQ signatures.

**If you want the reasoning behind these claims:** the bull/bear cases and fragmentation analysis below.

---

## The Fragmentation Problem

Without protocol defaults, each AA feature becomes a wallet-level ERC that must be adopted by every major wallet to be useful to app developers. The pattern is well-documented:

**Slow convergence**. [ERC-5792](https://eips.ethereum.org/EIPS/eip-5792) (`wallet_sendCalls` for batch calls) took an extended period for wallets to align on, and adoption remains incomplete.

**Fragmented APIs**. Competing vendors steward parallel standards for the same feature:

| Feature | Standard A | Standard B |
|---|---|---|
| Permissions / session keys | MetaMask: [ERC-7710](https://eips.ethereum.org/EIPS/eip-7710) + [ERC-7715](https://eips.ethereum.org/EIPS/eip-7715) | Base: [ERC-7895](https://eips.ethereum.org/EIPS/eip-7895) + [Spend Permissions](https://docs.base.org/identity/smart-wallet/concepts/features/optional/spend-permissions) |
| Batch calls | [ERC-5792](https://eips.ethereum.org/EIPS/eip-5792) (converged, slowly) | - |
| Fee sponsorship | [ERC-7677](https://eips.ethereum.org/EIPS/eip-7677) (Coinbase lineage) | Wallet-specific proposals emerging |

Without protocol defaults, every wallet's policy is as valid as any other. That is how you get N parallel ERCs per feature and a multi-year convergence cycle per ERC.

---

## Bear Case: Complexity Without Defaults

*Position*: EIP-8141 is too unopinionated. [Source](https://x.com/_jxom/status/2043135281604464905)

A UX-focused protocol should ship conventional defaults covering ~80% of use cases, plus an escape hatch for the 20%. Without them, the ecosystem reproduces the ERC-5792/ERC-7895/ERC-7710 fragmentation for every AA feature. Permissions already show this: Base and MetaMask model them differently, neither has wider adoption, and apps must pick a side or ship two implementations.

Shipping EIP-8141 in wallet SDKs (e.g. [Viem](https://viem.sh)) depends on confidence that frame transactions become the native AA standard. If the spec ships without defaults for batching, sponsorship, and permissions, follow-on ERC work still has to happen with the same extended convergence cycle.

---

## Bull Case: Native AA With Powerful Defaults

*Position*: EIP-8141 ships with defaults that change the adoption calculus. Sources: [Decentrek — Powerful Defaults](https://x.com/decentrek/status/2036697881512701997), [Doris G — Your Wallet is About to Change Forever](https://dorisgxyz.substack.com/p/your-ethereum-wallet-is-about-to), [dicethedev — Bundler Bottleneck framing](https://hackmd.io/@dicethedev/HyhbyJA3bg).

EIP-8141 already provides protocol-level defaults for the most common features:

- **Default code for EOAs**: secp256k1 signature verification without account migration, smart account deployment, or EIP-7702 delegation. Existing EOAs send frame transactions and the protocol handles the default signature path automatically. Execution approval uses signature index `0`; payment-only EOA sponsorship uses index `1` (PR #11954). P256 is available as a protocol-validated outer signature scheme, but accounts need code to use it for authorization.
- **Native ETH transfers**: SENDER frames carry a `frame.value` field (PR #11534, Apr 16), so wallets build simple ETH sends as one SENDER frame with `target = destination, value = amount` rather than shipping RLP call-list boilerplate in default code.
- **ETH-funded gas sponsorship via the canonical paymaster**: a protocol-blessed sponsorship contract that the public mempool validates efficiently. Wallets routing through it inherit FOCIL compatibility. The canonical paymaster handles ETH-funded sponsorship only; ERC-20 gas repayment is a separate design space with two independent EIP-8141 patterns (see caveat below). Adoption risk is tracked as an [open question](/mempool-strategy#canonical-paymaster-adoption).
- **Atomic batching**: expressed via bit 2 of `frame.flags` on consecutive frames of any mode, with the restrictive mempool tier forbidding the flag inside the validation prefix. No wallet-level RPC standard needed, no separate ERC for batch semantics.
- **Escape hatch**: arbitrary EVM in VERIFY/SENDER frames for the configurability cases, routed via the expansive tier or private mempool when it exceeds restrictive rules.

The adoption cost reduces to "implement a new transaction type" rather than "deploy/audit a smart account and run relayer infrastructure on every chain." The "no relayer" claim is strongest for onchain variants that route through the expansive tier or private mempool, including privacy rebroadcasters and trustless ERC-20 balance-checking sponsors. Wallets that prefer the public mempool can use a non-canonical ERC-20 sponsor that checks frame shape, but that sponsor accepts sponsee frontrunning risk. See [Mempool Strategy](/mempool-strategy#why-frame-transactions-dont-need-relayers).

> **ERC-20 gas repayment caveat**: the canonical paymaster handles ETH-funded sponsorship only. ERC-20 repayment is a non-canonical paymaster pattern. The public version checks sponsor authorization data and the next ERC-20 transfer frame without reading token balances, so it propagates but leaves the sponsor with frontrunning risk. A trustless onchain balance-checking version reads external token storage and therefore routes through the expansive tier, a private mempool, or direct-to-builder submission. Neither pattern depends on ERC-4337 infrastructure. See [Mempool Strategy → ERC-20 gas repayment](/mempool-strategy#erc20-paymaster-patterns).

**"Bundler Bottleneck" framing** (dicethedev): the central wallet-developer claim is that ERC-4337 requires a bundler because validation runs off-protocol. Frame transactions run validation in-protocol, which removes the structural need for the bundler/EntryPoint/paymaster-service triad for the common cases. This is the same argument stated from the wallet side rather than the mempool side.

**Svalbard interop ratification** (Apr 27): the [native-AA breakout](https://hackmd.io/@nixorokish/svalbard-aa-breakout) at Svalbard listed protocol-level call batching, alternative signature schemes (multi-sig, post-quantum, R1), key management with account recovery, and **no relayers for core functionality** as the primary goals for native AA, with unified default-account treatment of EOAs and smart accounts as a stretch goal. ETH-denominated sponsorship is the core case; ERC-20 sponsorship is explicitly deferred to relayer-dependent paths. The list reads as wallet-side validation of the bull case framing: protocol defaults for the common features (batching, signatures, ETH sponsorship) plus an escape hatch for the rest. See [Feedback Evolution → Svalbard Interop AA Breakout](/feedback-evolution#svalbard-interop-aa-breakout) for the full constraint set.

**Synthesis framing** (Doris G): EIP-8141 converges programmable validation (ERC-4337), EOA compatibility (EIP-7702), and authorization-as-protocol-primitive (EIP-3074), and adds frame-based atomic batching on top. Prior standards are evolutionary predecessors, not competitors. Session keys and graduated-permission patterns (phone passkey for daily <1 ETH, hardware for ≤50 ETH, AI-agent session keys with expiry, multi-signer treasury policies) move from ERC-4337-smart-account-only to protocol-native; they can be expressed in default-code VERIFY paths plus account code, without a bundler or EntryPoint.

**What frames do not solve** (Doris G): cross-chain identity persistence. Per-transaction validation runs inside a frame; the mapping from "which keys are authorized for this user" to "this user's assets across chains" is a separate layer. A keystore registry is the complementary infrastructure for asset-signer separation. EIP-8141 is silent on this layer by design.

---

## Where Fragmentation Risk Still Lives

Protocol defaults cover batching, signatures, and ETH-funded sponsorship. They do not cover everything:

| Feature area | Default covers? | Public-mempool? | ERC work still needed? |
|---|---|---|---|
| Atomic batching | Yes (flags field, bit 2) | Yes | No |
| Gas sponsorship, native ETH | Yes (canonical paymaster) | Yes | No for basic case |
| Gas sponsorship, ERC-20 - public risk-accepting sponsor | Consensus-valid, not a protocol default | **Yes** (1 pending per non-canonical paymaster) | Sponsor risk controls / signing-infra work |
| Gas sponsorship, ERC-20 - trustless balance-checking sponsor | Consensus-valid, not a protocol default | **No** (expansive/private only) | Contract deployment; expansive-tier routing |
| secp256k1 signatures | Yes (default code) | Yes | No |
| P256 / passkeys | Protocol-validated outer signature scheme, not default-code authorization | Yes when account code fits the restrictive tier | Account-code or ERC work |
| Post-quantum signatures | Via account code or precompiles | Depends on scheme cost vs 100k cap | Scheme-by-scheme ERCs expected |
| Permissions / session keys | No | Depends on validation shape | Yes (ERC-7710/7715 vs ERC-7895 divergence exists) |
| Social recovery | No | Typically no | Yes |
| Multisig policies | No | Depends on validation shape | Yes |
| Wallet-to-app communication | Out of scope | N/A | Yes (ERC-5792 and successors) |

The bear case is not wrong, it is partially absorbed. Protocol defaults remove fragmentation pressure on the most common features. They do not remove it for permissions, passkeys, recovery, trustless ERC-20 gas repayment, or the wallet-to-app RPC layer.

A first datapoint on the direction of travel: ERC-8286 (chiranjeev13, [ERC PR #1794](https://github.com/ethereum/ERCs/pull/1794), draft, opened Jun 3 2026) standardizes how [ERC-7579](https://eips.ethereum.org/EIPS/eip-7579) modular accounts (validator, executor, hook, and config modules) implement the EIP-8141 validation flow: a validator module returns an approval mode the account applies via `APPROVE` inside a VERIFY frame. It is the first ERC built on top of EIP-8141 (`requires: 7579, 8141`), and it targets exactly the permissions and session-key layer that protocol defaults leave open. ERC-8286 was presented alongside EIP-8130 at [AllWalletDevs #40](https://ethereum-magicians.org/t/allwalletdevs-40-july-15-2026/28858) on Jul 15, moving the design from a repository draft into direct wallet-developer comparison. The signal is that the modular-account ecosystem is starting to organize around native AA as the base rather than fragmenting against it.

A weaker corroborating signal: ERC-8211 (Smart Batching, [ERC PR #1638](https://github.com/ethereum/ERCs/pull/1638), draft) is transport-agnostic, running over ERC-4337, ERC-7702, ERC-7579, or ERC-6900 today, but its Forward Compatibility section names EIP-8141 SENDER frames as a future execution path for the same `ComposableExecution[]` encoding. It does not take EIP-8141 as a dependency. As of this writing ERC-8286 is the only ERC that requires EIP-8141 in its header; ERC-8211 merely anticipates it.

---

## Summary

- The fragmentation concern is real. Wallet-level ERCs converge slowly and fragment across vendors.
- EIP-8141 addresses several features likely to fragment via protocol defaults: batching, signatures, and ETH-funded gas sponsorship, all reachable from existing EOAs on the public mempool.
- ERC-20 gas repayment has a public/private split. Public non-canonical sponsors propagate with frontrunning risk. Trustless balance-checking sponsors route through the expansive tier or private mempool.
- Permissions, session keys, passkeys, recovery, trustless ERC-20 gas repayment, and the wallet RPC layer remain outside protocol defaults. ERC-level fragmentation continues for those.
