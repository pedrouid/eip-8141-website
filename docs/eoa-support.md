# EOA Support

---

## TL;DR

EIP-8141 makes existing EOAs first-class AA users with no migration, no smart-account deployment, and no [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702) delegation. The mechanism is a protocol-level **default code** that runs whenever a frame targets an account with no deployed code, handling signature verification and call dispatch on the EOA's behalf.

- **Replaces EIP-7702 delegation for common cases**. No `set_code` tx, no persistent delegate, no extra signing ceremony.
- **Per-transaction, not persistent**. Default code is protocol logic, not state on the account. Nothing is recorded on-chain.
- **Composable per-transaction**. The wallet picks the frame composition for each transaction.
- **EOA can act as a paymaster**. An ETH-funded EOA can sponsor via payment-only default code using signature index `1`, while the sender uses index `0`.
- **Custom account code remains available**. Accounts that need richer logic can still deploy code, which overrides default code.
- **DEFAULT frames serve positional roles**. First frame for account deployment, last frame for paymaster post-op refunds.

Default code does NOT handle contract deployment (that uses a separate `deploy` frame), and does not run for 7702-delegated EOAs (the delegated code takes over).

---

## What EOAs Get For Free

Every EOA gets the following behavior without opt-in, deployment, or signed authorization:

### VERIFY mode

1. Require `frame.target == tx.sender` unless the approval scope is payer-only (`0x1`). Payment-only approval can target a different EOA paymaster.
2. Read approval scope from the flags field: `scope = frame.flags & 3`. If `scope == 0`, revert.
3. Select `sig_index = 0` when the scope includes execution; otherwise select index `1` for payment-only approval.
4. Require `tx.signatures[sig_index]` to be a `SECP256K1` entry with empty `msg` whose resolved signer equals the resolved frame target. Empty signer metadata resolves to `tx.sender`.
   - `0x1` (`SECP256K1`): require a 65-byte `v || r || s` signature, `v` in `{0, 1}`, canonical non-zero `r`, low-`s`, and `resolved_target == ecrecover(sig_hash, sig)`.
   - Anything else: revert.
5. Call `APPROVE(scope)`.

PR #11481 (merged May 22) moved per-tx signatures out of `frame.data` and into a dedicated outer `signatures` list. PR #11814 (merged Jul 7) replaced the linear scan with index `0`; PR #11954 (merged Jul 17) restored codeless EOA sponsorship by assigning payment-only approval to index `1`. PR #11937 (merged Jul 17) pinned secp256k1 recovery-id and canonical low-`s` encoding. The outer list has three schemes: `ARBITRARY (0x0)`, `SECP256K1 (0x1)`, and `P256 (0x2)`. P256 is protocol-validated in the list, but PR #11621 (merged May 11) removed the P256 branch from default code. Hardware wallets and passkey-based accounts that need P256 must still ship that logic themselves (via deployed code, `SIGPARAM`, and custom verification, or via a future extension EIP). secp256k1 ECDSA is the only signature scheme that ships in the protocol-level default code.

The split index rule supports two distinct codeless EOAs in one transaction: the sender's execution-approving signature occupies index `0`, and a payment-only EOA sponsor's signature occupies index `1`. A distinct sponsor entry carries the sponsor address in `signer`; leaving it empty would resolve to `tx.sender`. Combined execution-and-payment approval still uses index `0` because its scope includes execution.

### SENDER mode

Top-level value transfer succeeds. PR #11621 (merged May 11) changed default code so that `SENDER` frames no longer revert. The frame's value (per [PR #11534](https://github.com/ethereum/EIPs/pull/11534), Apr 16) is transferred to `tx.sender` at the top-level call boundary using ordinary CALL value-transfer semantics, then the frame returns with empty data. The earlier RLP-encoded call-list payload was removed by PR #11577 (Apr 29); multi-call sequences now compose at the frame-list level via batching, not inside a single frame's payload. Atomic semantics come from the [atomic batch flag (bit 2 of `flags`)](https://eips.ethereum.org/EIPS/eip-8141#mode-flags), which PR #11652 (May 12) extended to all frame modes.

**Native ETH transfer**: A simple ETH transfer is one SENDER frame with `target = destination`, `value = amount`, `data = empty`. The protocol transfers ETH at the top-level frame call boundary; the wallet does not construct any default-code payload. After PR #11621, a transfer to a fresh EOA (default-code target) also completes cleanly rather than reverting in default code.

### DEFAULT mode

Default code returns with empty data without performing any other action. PR #11621 also relaxed the previous unconditional revert on `DEFAULT` for default-code targets, which lets a default-code account receive a `DEFAULT`-mode call (e.g. as a value transfer terminator). DEFAULT frames are typically used to target deployed contracts (first-frame account deployment, last-frame paymaster post-op) but the default-code path no longer breaks if a frame lands on a codeless account in that mode.

---

## DEFAULT Frames in Practice

| Position | Use case | Why DEFAULT mode |
|---|---|---|
| **First frame** | Account deployment | Account doesn't exist yet, so EntryPoint is the only meaningful caller. |
| **Last frame** | Paymaster post-op | Paymaster gates refund logic on `caller == ENTRY_POINT`. |

**Account deployment**: targets a stateless factory (EIP-7997 is the canonical-but-non-mandatory predeploy after PR #11567 merged Apr 30; any factory whose execution satisfies the deploy-frame trace rules now qualifies). Creates the account before VERIFY frames run. The mempool recognizes two deploy-prefixed validation shapes (`deploy → self_verify` and `deploy → only_verify → pay`). Inside the first deploy frame, `CREATE` (0xF0), `CREATE2` (0xF5), and `SETDELEGATE` (0xF6, EIP-7819) may install code at `tx.sender` (including an EIP-7702 delegation indicator), and `SSTORE`s to `tx.sender`'s storage are permitted.

**Paymaster post-op**: the paymaster charges upfront, the protocol refunds unused gas as ETH, then a DEFAULT frame calls the paymaster to refund the user's ERC-20 overpayment. The post-op frame is part of execution (after `payer` has been set) and not subject to validation rules.

---

## Why This Replaces EIP-7702 for Common Cases

| Property | EIP-7702 + delegate | EIP-8141 default code |
|---|---|---|
| Onchain footprint | Persistent delegation header | None |
| Authorization tx | Required `set_code` signing | None |
| Delegate deployment | Required (deploy + audit) | None (protocol logic) |
| Per-tx flexibility | Limited to delegate's features | Wallet composes per transaction |
| Signature schemes | Whatever delegate implements | secp256k1 baked in for codeless EOAs; P256 is an outer signature scheme but not accepted by default code |
| Gas sponsorship | Delegate must implement | Canonical paymaster, or EOA default VERIFY with sender at index `0` and payment-only sponsor at index `1` |
| Reversal cost | Another `set_code` authorization | Nothing to reverse |

The cost of EIP-7702 in production: (1) wallet must develop and audit a smart account, (2) wallet must run relayer infrastructure on every chain, (3) user must sign a delegation, (4) delegation is persistent so revocation requires another authorization. Default code eliminates all four for the common case. See [Developer Tooling → Bull Case](/developer-tooling#bull-case-native-aa-with-powerful-defaults).

---

## Per-Transaction Composability

EIP-8141's EOA support is **per-transaction, not per-account**. The same EOA can, across consecutive transactions: send a simple transfer, do an atomic approve-and-swap, accept ETH-funded gas sponsorship from a canonical paymaster, or deploy a smart account in the same transaction that uses it.

Each composition is a distinct frame transaction. Nothing on the account changes between them. Feature rollout ships in the wallet's frame-construction logic, not in a smart account redeploy. New AA features become available the moment the wallet supports them, for every existing EOA, without any user action.

---

## EOA as Paymaster

An EOA can act as an ETH-funded paymaster. The default VERIFY logic supports `APPROVE(scope)` for payment scope (`0x1`) and combined scope (`0x3`). A distinct sponsor signs a payment-only VERIFY frame through `tx.signatures[1]` with `signer = sponsor`; the sender can independently approve execution through index `0`. The sponsor's ETH balance covers the user's gas, with no paymaster contract required. High-throughput public sponsorship still favors the canonical paymaster because non-canonical paymasters are limited to one pending transaction each.

This composes with the [restrictive mempool tier](/mempool-strategy#restrictive-mempool-what-ships-first) under the `MAX_PENDING_TXS_USING_NON_CANONICAL_PAYMASTER = 1` rule per sponsor. The canonical paymaster contract exists for high-throughput ETH-funded sponsorship.

### ERC-20 repayment: two independent paymaster patterns

A related but distinct pattern is "user pays the sponsor back in ERC-20 tokens" (spec [Examples 2 and 5](/current-spec#practical-use-cases)). The current public-mempool shape is a **risk-accepting ERC-20 sponsor**: the sponsor's VERIFY frame checks sponsor authorization data and inspects the next SENDER frame to confirm it is an ERC-20 transfer of the right shape, but it does not read the user's token balance during validation. That propagates as a non-canonical paymaster subject to the one-pending-tx cap, while the sponsor accepts the frontrunning risk that the user can zero the ERC-20 balance before inclusion. A **trustless onchain balance-checking paymaster** remains consensus-valid, but because it reads external token storage during validation it routes through an expansive tier, private mempool, or direct-to-builder path. Both patterns are native to EIP-8141 and do not rely on ERC-4337. See [Mempool Strategy → ERC-20 gas repayment](/mempool-strategy#erc20-paymaster-patterns).

---

## When You Still Need Custom Account Code

Default code covers the common case but not everything:

- **Multisig authorization**: more than one signer
- **Social recovery**: trusted parties rotating the signing key
- **Session keys**: scoped, time-bounded keys with per-call rules
- **Custom signature schemes** beyond default-code secp256k1 (including P256/passkeys in account code, despite P256 being protocol-validated in the outer signatures list)
- **State-dependent validation**: rules reading more than `tx.sender`'s storage
- **Non-trivial paymaster logic**: rate limiting, allowlists, etc.

Default code is the floor, not the ceiling. Custom validation that exceeds the restrictive mempool's bounds routes through the [expansive tier](/mempool-strategy#two-tiers-in-one-mempool).

---

## What Default Code Doesn't Do

**Contract deployment**: uses a separate `deploy` frame targeting a stateless factory. EIP-7997 is canonical but non-mandatory after PR #11567; any factory satisfying the deploy-frame trace rules works, including custom CREATE2 deployers and EIP-7702 delegation installation. Default code's DEFAULT mode returns empty data; it does not deploy code by itself.

**7702-delegated EOAs**: if an EOA has signed a `set_code` authorization, the delegate's code runs instead of default code. This is a real interoperability gap [identified by DanielVF](/current-spec#related-proposals): a wallet that 7702-delegates is on the hook for reimplementing what default code provided. EOAs that want default code behavior should not 7702-delegate.

---

## Summary

EIP-8141 makes EOAs first-class AA users through protocol-level default code, eliminating the authorization-transaction, smart-account-deployment, and relayer overhead that EIP-7702 + EIP-4337 requires. Default code is the floor: accounts that need multisig, recovery, or exotic validation still deploy custom code. DEFAULT frames serve two positional roles (deployment and post-op), both gated on `caller == ENTRY_POINT`.

Default code handles per-transaction validation. It does not handle cross-chain identity persistence, meaning the mapping from "which keys are authorized for this user" to "this user's assets across chains." That layer is addressed by keystore registries, which are complementary to frame transactions rather than a replacement (see [Developer Tooling → Bull Case](/developer-tooling#bull-case-native-aa-with-powerful-defaults) for the asset-signer-separation framing).
