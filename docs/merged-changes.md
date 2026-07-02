# Changes Merged Over Time and Why

---

## Day 0 Fixes — January 29, 2026

*Why this mattered: two tiny fixes merged on the same day as the original submission, establishing a fast-review cadence that carried through the rest of the spec's evolution.*

### PR #11205: Add missing elision of VERIFY frame data from signature hash

**Author**: fjl | **Merged**: Jan 29

- Bug: the original spec did not elide `frame.data` of VERIFY frames when computing the canonical signature hash
- This was critical because VERIFY frames contain the signature itself; by definition it cannot be part of the signature hash
- Merged same day

### PR #11209: Fix status field number

**Author**: kevaundray | **Merged**: Jan 29

- The TXPARAM table had an incorrect field number for the `status` parameter
- Simple numbering fix, approved by fjl

---

## APPROVE Relaxation — February 10, 2026

*Why this mattered: unblocked proxy-based smart accounts (Safe-style) from adopting the spec. Without this, accounts whose outer proxy predates `APPROVE` would have been locked out entirely.*

### PR #11297: Relax requirement that APPROVE must be called by top level call frame

**Author**: lightclient | **Merged**: Feb 10

- **Why**: Existing smart accounts (especially proxy-based ones like Safe) can change their implementation code but NOT their outer proxy contract. The outer proxy uses `RETURN`, not `APPROVE`. Requiring top-level `APPROVE` made adoption impossible for these accounts.
- **Change**: `APPROVE` became transaction-scoped, meaning it can be called from any depth and updates `sender_approved`/`payer_approved` directly, rather than requiring the top-level frame to exit with a special return code.

From lightclient's PR description:

> This allows existing smart accounts to more easily adopt EIP-8141. Before, the requirement was that accounts must exit the top level frame with APPROVE. Since APPROVE only exists with 8141, not smart accounts today support it. More importantly, smart accounts who are deployed with proxies _can_ change their smart account implementations, but still not the outer proxy which won't understand `APPROVE`.

---

## Bug Fixes & Clarifications — February-March 2026

*Why this mattered: caught a refactor-introduced CALLER/ADDRESS bug in APPROVE and settled ambiguities around frame reverts before downstream PRs built on the older, wrong assumptions.*

### PR #11344: Fix some issues with EIP-8141

**Author**: derekchiang | **Merged**: Mar 2

- **Fixed CALLER vs ADDRESS**: Changed `CALLER == frame.target` to `ADDRESS == frame.target` for APPROVE. In VERIFY frames, CALLER is ENTRY_POINT, not frame.target. This was a bug introduced during refactoring.
- **Removed APPROVE restriction to VERIFY frames**: lightclient wanted APPROVE available in any mode for private pool use cases (stateful approvals).
- **Clarified frame reverts**: Made explicit that a frame revert discards that frame's state changes but doesn't affect other frames.
- Notable discussion: nlordell asked about the TXPARAM numbering jump from 0x09 to 0x10; lightclient confirmed it was intentional to separate tx-level vs frame-level queries.

---

## EOA Support — March 5-10, 2026

*Why this mattered: the pivot. Made EOAs first-class users of frame transactions and reframed the spec from "smart-account-assumed" to "EOA-first." Every downstream design decision traces back to this merge.*

### PR #11379: Add EOA support

**Author**: derekchiang | **Merged**: Mar 10

- **Why**: The biggest driver was adoption: if most users are on EOAs, frame transactions need to work for them natively, without requiring smart contract deployment.
- **What changed**: Added "default code" behavior for EOAs:
  - In VERIFY mode: reads `frame.data` as a signature, supports SECP256K1 (0x0) and P256 (0x1) types, calls `APPROVE(scope)`
  - In SENDER mode: reads `frame.data` as RLP-encoded calls `[[target, value, data], ...]` and executes them
  - In DEFAULT mode: reverts
- Also initially added a `value` field to frames (later removed; the default code handles value via call encoding)

---

## Opcode Redesign — March 12, 2026

*Why this mattered: gave scalar and variable-length transaction data the opcodes they actually need. Scalar values no longer pay the cost of copy semantics meant for byte strings, and frame input data gets dedicated `FRAMEDATA*` opcodes that match its shape.*

### PR #11400: Clean up frame access opcodes

**Author**: fjl | **Merged**: Mar 12

- **Why**: Most TXPARAM values are 32 bytes or less, so `TXPARAMSIZE`/`TXPARAMCOPY` (designed for variable-length data) were unnecessary for most fields. The only variable-size data is frame input data.
- **Change**: Replaced `TXPARAMSIZE (0xb1)` and `TXPARAMCOPY (0xb2)` with dedicated `FRAMEDATALOAD (0xb1)` and `FRAMEDATACOPY (0xb2)`. Renamed `TXPARAMLOAD` to just `TXPARAM (0xb0)`.

From fjl's PR description:

> Most values returned by the TXPARAM opcode family is 32-bytes or less, so it makes no sense to be able to copy the data into memory. The only variable-size components of the transaction are the frame list and the input data of each frame. So I am defining specialized opcodes for reading the input data of the frame.

---

## Approval Bits — March 12-13, 2026

*Why this mattered: let users sign over their intended approval scope as part of the signed transaction payload, so smart accounts don't each have to implement bespoke scope-extraction logic. Simplified the default code and moved trust about scope from frame data to mode bits.*

### PR #11401: Add approval bits to frame mode

**Author**: fjl | **Merged**: Mar 12

- **Why**: Users needed a way to specify their intended approval scope **in the signed transaction data** so that smart accounts don't have to extract scope from `frame.data` (which is elided from the signature hash). Without this, accounts would need to compute `keccak(sighash | scope)`, which is repetitive logic every account must implement.
- **Change**: Added bits 9-10 of `frame.mode` as approval scope constraints. These constrain which `APPROVE` scopes are valid for that frame.
- Also shifted APPROVE scope values from 0-indexed (0x0, 0x1, 0x2) to 1-indexed (0x1, 0x2, 0x3) so that the mode bits can directly encode the constraint.

From fjl's PR description:

> The primary motivation for adding these bits is allowing the user to specify to their own account what should be approved, while retaining the ability to sign directly over the sighash of the transaction. However, the concept of mode bits is going to be useful for other things we may have to put in later, and it also gives some security benefits.

### PR #11402: Fix bit indices

**Author**: fjl | **Merged**: Mar 13

- Changed from 0-indexed to 1-indexed bit numbering (bits 9 and 10 instead of 8 and 9), since bits are "typically one-indexed."

---

## Atomic Batching — March 11-25, 2026

*Why this mattered: settled a month-long debate (atomic-by-default vs opt-in vs group-id) with a flag-based design that gave users atomic "approve + swap" semantics without breaking paymaster flows that need non-atomic defaults.*

### PR #11395: Add support for atomic batching

**Author**: derekchiang | **Merged**: Mar 25

- **Why**: Users need "all-or-nothing" semantics for related operations (e.g., ERC-20 approve + swap). Without this, a revert in one frame leaves prior frames' state changes applied, creating dangerous intermediate states.
- **Evolution**: Originally proposed a `SENDER_ATOMIC` mode. After extensive discussion:
  - pedrouid argued for atomic-by-default
  - derekchiang & frangio explained non-atomic is needed for paymasters
  - 0xrcinus proposed explicit group IDs
  - sm-stack suggested bit flags
- **Final design**: Bit 11 of `frame.mode` as the "atomic batch" flag. Consecutive SENDER frames with the flag set form a batch. If any frame in the batch reverts, all preceding frames are reverted and remaining frames are skipped.
- **Additional change**: Required `sender_approved` to be true before any SENDER frame can execute. This forbids APPROVE from being called in SENDER mode, preventing complexity where an atomic batch could revert a payment approval.

From derekchiang's PR description:

> This PR also adds the explicit requirement that `SENDER` frames can only be executed when `sender_approved` and `payer_approved` have been set to `true`, which essentially forbids `approve` from being called in `SENDER` modes. This is important because otherwise a `SENDER_ATOMIC` frame can `approve` gas payment, and then the payment frame may be reverted by other `SENDER_ATOMIC` frames, creating complexity for transaction validation.

---

## Mempool Policy — March 16-25, 2026

*Why this mattered: turned EIP-8141 from a consensus-layer spec into something clients could actually ship. Bounded validation cost, standardized the paymaster shape, and replaced ERC-7562's reputation/staking complexity with a canonical-paymaster code match.*

### PR #11415: Add mempool policy

**Author**: lightclient | **Merged**: Mar 25

- **Why**: Without clear mempool rules, node operators would each implement their own policies, creating inconsistency and potentially enabling DoS attacks. The policy section was inspired by ERC-7562 but simplified.
- **Key innovations**:
  - **Validation prefix**: Only the frames up to `payer_approved = true` are subject to mempool rules. Post-payment frames are arbitrary.
  - **Canonical paymaster**: A standardized paymaster contract verified by runtime code match. Removes the entire reputation/staking system from ERC-7562.
  - **Non-canonical paymaster limit**: At most 1 pending tx per non-canonical paymaster
  - **Banned opcodes**: ORIGIN, GASPRICE, BLOCKHASH, COINBASE, TIMESTAMP, NUMBER, PREVRANDAO, GASLIMIT, BASEFEE, BLOBHASH, BLOBBASEFEE, GAS (with exceptions), CREATE, CREATE2 (with exceptions), INVALID, SELFDESTRUCT, BALANCE, SELFBALANCE, SSTORE, TLOAD, TSTORE
  - **MAX_VERIFY_GAS**: 100,000 gas cap for validation prefix
  - **Four recognized validation prefixes**: self-relay (basic & deploy), canonical paymaster (basic & deploy)
- **Notable review feedback**: jochem-brouwer provided detailed review of the canonical paymaster Solidity contract, catching PUSH0 optimization, stack order documentation, withdrawal overwrite handling, and chain ID checks.

From lightclient's PR description:

> One important change vs. 7562 is relying on a canonical paymaster contract. This point is open for discussion generally, but by having a canonical contract, we can remove the complex reputation system around the staking. It's also possible to add more functionality here over time.

---

## Default Code Update — March 26, 2026

*Why this mattered: kept the reference implementation in sync after approval bits changed how scope is encoded. Without this, every implementer would have had to reconcile an outdated default code against the latest approval-bit semantics on their own.*

### PR #11448: Update default code to match latest spec

**Author**: derekchiang | **Merged**: Mar 26

- **Why**: The approval bits addition meant the default code could be simplified: the scope is now read from the mode bits instead of being encoded separately.
- Updated the default code Python reference implementation to match all recent spec changes.

---

## Header Metadata Fix — April 8, 2026

*Why this mattered: a simple dependency-header fix that had been open for two months. The merge is notable mostly for how uncontroversial it was, and how long such a trivial change can sit waiting for author attention.*

### PR #11251: Add EIP-1559 to requires header

**Author**: BonyHanter83 | **Merged**: Apr 8 (opened Feb 4)

- **Why**: The spec uses EIP-1559's `max_priority_fee_per_gas` and `max_fee_per_gas` fields and explicitly states that `effective_gas_price` is calculated per EIP-1559, but the header didn't list it as a dependency.
- **Change**: Added `1559` to the `requires` header field alongside `2718` and `4844`.
- Approved by lightclient.
- This PR was open for over two months before being merged. It was a simple metadata fix that had no controversy.

---

## Broad Spec Tightening — April 14, 2026

*Why this mattered: the broadest single-PR restructuring since approval bits. Introduced a fifth opcode (`FRAMEPARAM`), hardened both default-code signature paths, reduced `MAX_FRAMES`, and locked deterministic deployment to EIP-7997. Consolidated several open threads into one coherent update.*

### PR #11521: Tighten spec

**Author**: benaadams (Ben Adams) | **Merged**: Apr 14

A 295-line spec-hardening PR consolidating several open threads. Changes:

- **Frame model**: Split the packed `mode` field into separate `mode` + `flags` fields. Added `FRAMEPARAM (0xb3)` opcode for frame introspection (renamed from `FRAMEINFO` at derekchiang's suggestion for consistency with `TXPARAM`). Introduced explicit `resolved_target` used consistently throughout execution.
- **APPROVE/VERIFY semantics**: Defined approval scopes as a bitmask with double-approval prevention. Aligned public-mempool prefixes. Made the VERIFY/STATICCALL carve-out explicit. Clarified that payment scopes collect `TXPARAM(0x06)`. Allowed third-party EOA paymasters in the default-code path.
- **Default code hardening**: Low-`s` enforcement for secp256k1. Reject failed `ecrecover`. Added P256 address-domain separation (`0x04` prefix before `qx|qy`). Require `P256VERIFY` to reject invalid public keys.
- **Limits and accounting**: Reduced `MAX_FRAMES` from `10^3` to `64`. Added `FRAME_TX_PER_FRAME_COST` (475 gas). Bounded frame gas totals. Clarified gas semantics for `FRAMEDATACOPY` and `TXPARAM`/blob access.
- **Deployment**: Locked deterministic deployment to EIP-7997. Made EIP-7702 interaction explicit. Added EIP-7997 to the `requires` header.
- **Security notes**: Stronger warnings around VERIFY-data malleability, `DELEGATECALL` + `APPROVE`, deploy-frame front-running, explicit-sender state-read amplification, and validation-time cross-frame data visibility.
- **Canonical paymaster**: Updated to use `TXPARAM(0x08)`. Documented that the current canonical implementation is single-signer secp256k1 only.

Key review discussion: fjl questioned lowering MAX_FRAMES from 1000 to 64, noting the high limit was intentional for native batching/bundling. benaadams responded that journaling carries across frames, creating up to 2000 effective call depth, and that it's easier to increase later after empirical measurement than to decrease. lightclient and derekchiang both approved.

This is the broadest restructuring since PR #11401 (approval bits) and the first PR to add a fifth opcode to the spec.

---

## Per-Frame Value — April 16, 2026

*Why this mattered: resolved the longest-running ergonomic ask. Wallets can now build a simple ETH transfer as one SENDER frame with `target = destination, value = amount`, instead of encoding an RLP call list inside default code. Ended the "frames are dumb message pipes" criticism from wallet developers.*

### PR #11534: Add value field to frame

**Author**: lightclient | **Merged**: Apr 16

Resolves the long-running community request for native ETH value transfer in frame transactions. Changes:

- **Frame model**: Extended the frame tuple from `[mode, flags, target, gas_limit, data]` to `[mode, flags, target, gas_limit, value, data]`. The `value` field is interpreted as the top-level call value in wei.
- **Validity rules**: Added `frame.value < 2**256` and `frame.mode == SENDER or frame.value == 0`. Non-zero `value` is only valid in `SENDER` frames; `DEFAULT` and `VERIFY` frames must set `value = 0`.
- **Execution semantics**: In the top-level frame call, `CALLVALUE = frame.value`. If the caller lacks sufficient balance to transfer `frame.value`, the frame reverts (ordinary CALL value-transfer semantics). `ENTRY_POINT` is now documented as observing `CALLVALUE = 0` for DEFAULT/VERIFY frames.
- **FRAMEPARAM extension**: Added `FRAMEPARAM(0x08, frameIndex)` returning the frame's `value`.
- **TXPARAM clarification**: `TXPARAM(0x06)` (max gas and blob fees) explicitly does not include any `frame.value` transfers.
- **Default code change**: In `SENDER` mode when `resolved_target != tx.sender`, the default code now returns successfully with empty data instead of reverting, because the top-level `frame.value` transfer has already been applied by the frame call. This matches the behavior of calling an empty-code account.
- **Signature hash coverage**: Documented that non-`VERIFY` frame metadata, including a `SENDER` frame's `value`, remains covered by the canonical signature hash.
- **Gas accounting**: Any `frame.value` transferred by a `SENDER` frame is separate from `tx_fee` and follows ordinary CALL value-transfer gas semantics.
- **Rationale**: Renamed the "No value in frame" rationale section to "Per-frame value" and explained that restricting non-zero `value` to `SENDER` frames keeps `VERIFY` and `DEFAULT` side-effect-free with respect to ETH transfers, preserves the `STATICCALL`-like behavior of `VERIFY`, and avoids requiring `ENTRY_POINT` to fund top-level ETH transfers.
- **Examples**: All transaction examples updated with a `Value` column. Example 1a (Simple ETH transfer) was restructured: instead of instructing the sender account to send ETH via payload decoding, the `SENDER` frame now sets `target = destination` and `value = amount` directly.

Key review discussion: lightclient's PR description notes the original resistance to a `value` field (preferring user-operation execution to be handled by the account) and the reversal now that frames are targeted at a good out-of-the-box experience without requiring wallet-side batching. Auto-merged after all reviewers approved; no debate on the merged PR.

Context: this resolves the "VALUE in SENDER frames" pending proposal that had accumulated support from rmeissner (Safe), DanielVF, frangio, 0xrcinus, derek, and matt across posts #124-134.

---

## Sighash Type Prefix — April 22, 2026

*Why this mattered: aligned EIP-8141's canonical sighash with the EIP-2718 typed-transaction convention, closing a cross-type signature replay vector.*

### PR #11544: Mix in transaction type to the sighash

**Author**: derekchiang | **Merged**: Apr 22

One-line change in `compute_sig_hash`:

- Replaced `keccak(rlp(tx_copy))` with `keccak(bytes([FRAME_TX_TYPE]) + rlp(tx_copy))`.
- Prefixes the type byte (`FRAME_TX_TYPE = 0x06`) before RLP-encoding, matching the EIP-2718 convention used by every other typed transaction type since EIP-1559.
- Without the prefix, a signature over a frame-transaction sighash could in theory be reused against another transaction type sharing the same RLP body.

All reviewers approved by Apr 18; auto-merged on Apr 22 with no further debate.

---

## Default Code Cleanup and Payer-Ordering Misadventure — April 28-29, 2026

*Why this mattered: with native frame batching and per-frame `value` both in the spec, lightclient pruned the now-redundant RLP call-batch decoding from the default account. In the same window, an alternative to derekchiang's guarantors PR (allow payer to approve before sender) was auto-merged in error and reverted within hours, then reopened as a draft. Net spec impact: only the RLP-batch removal landed.*

### PR #11577: Remove RLP call batch from default account

**Author**: lightclient | **Merged**: Apr 29

- **Why**: PR #11395 (Mar 25) introduced atomic batching at the frame-list level, and PR #11534 (Apr 16) added a per-frame `value` field. Together they cover the multi-call and ETH-transfer use cases that the default-code SENDER path was originally added to handle.
- **Change**: Default code's `SENDER` mode now simply reverts. The previous logic (which decoded `frame.data` as RLP `[[target, value, data], ...]` when `resolved_target == tx.sender`, and returned successfully with empty data when `resolved_target != tx.sender`) is removed. Net diff: +2/-15.
- Auto-merged after all reviewers approved; no review debate.

### PRs #11575 / #11579: Allow payer to approve before sender (merged in error, reverted same window)

**Author**: lightclient | **Merged-then-reverted**: Apr 28-29

- PR #11575 was an alternative to derekchiang's guarantors PR (#11555), proposed as a simpler relaxation: drop the rule that the sender must approve before the payer, so a payer can commit to gas without depending on sender-validation outcome. From lightclient's PR description: *"I think it is simpler to just allow the payer to approve before the sender instead of adding the full guarantor role."*
- The auto-merge bot fired on Apr 28 once reviewers approved. lightclient had intended the PR to remain a draft for further iteration, and opened PR #11579 the next day reverting the change with the note *"Meant to only create this as a draft."*
- The same content is now open as draft PR #11580. Net spec impact of #11575/#11579: zero. Listed here for traceability of the spec history; the live proposal sits in [Active/Open PRs](#active-open-prs) below.

---

## Mempool Factory Relaxation and Hegotá CFI Inclusion — April 30, 2026

*Why this mattered: two same-day merges that move EIP-8141 forward on parallel tracks. PR #11567 reframes the deploy-frame mempool rule from a named-contract whitelist into a stateless-trace policy and drops EIP-7997 as a hard dependency, removing the spec's only same-fork hard requires entry. PR #11537 lands the formal Hegotá CFI status that ACDE #233 had already signaled, completing the governance step that was outstanding from Phase 6.*

### PR #11567: Relax mempool rules to not require a specific factory

**Author**: derekchiang | **Merged**: Apr 30 (opened Apr 24)

- **Why**: The pre-merge spec hard-coded the EIP-7997 deterministic factory predeploy as the only valid `frame.target` a mempool node would propagate a deploy frame to, and listed EIP-7997 in `requires`. The actual safety invariant the restrictive tier needs is that deploy-frame outcome is independent of chain state outside `tx.sender`. Pinning the rule to one named contract conflated convenience with safety and blocked alternative stateless factories, custom CREATE2 deployers, and EIP-7702 delegation installation as deploy-frame primitives.
- **Spec changes** (+11/-8):
  - Drops `7997` from the `requires` header (now `1559, 2718, 4844`)
  - Mempool deploy-frame rule rewrites: any contract may be `frame.target`, provided the frame's execution satisfies the validation trace rules
  - Write policy expands from "deterministic deployment performed by the first `deploy` frame through a known deployer" to "inside the first `deploy` frame, (a) `CREATE`, `CREATE2`, or `SETDELEGATE` operations that install code at `tx.sender`, or (b) `SSTORE`s to `tx.sender`'s storage"
  - Banned-opcode allowlist: `CREATE` (0xF0) and `SETDELEGATE` (0xF6, EIP-7819) join `CREATE2` (0xF5) as exceptions inside the first `deploy` frame
  - Deploy-frame outcome rule: now satisfied by any non-empty code at `tx.sender`, including conventional contract code or an EIP-7702 delegation indicator (was: "non-empty, non-delegated code")
  - The `deploy` mode description in the structural rules table softens from "Deploys a new smart account using the EIP-7997 deterministic factory predeploy" to "Deploys a new smart account, typically via a deterministic factory such as the EIP-7997 predeploy"
  - Front-running rationale rephrased so initcode safety is generalized to "the deploy frame's calldata (and any initcode it carries) must be safe to submit by any party"
- **Key review discussion**: lightclient approved the same day with "SGTM" and the auto-merge bot fired on Apr 30. No public debate on the diff; the conceptual change had been telegraphed in the PR description for six days.
- **Consequence**: EIP-7997 becomes the canonical-but-non-mandatory factory. The spec drops its only same-fork hard dependency. Smart-account deployment and EIP-7702 delegation installation now flow through the same deploy-frame primitive, with the mempool treating delegation-indicator installation as a legitimate deployment outcome. This is the broadest mempool-policy change since PR #11415 (Mar 25 mempool policy) and the first to retract a `requires` entry.

### PR #11537: Add EIP-8141 to CFI in EIP-8081 Hegotá meta EIP

**Author**: dionysuzx | **Merged**: Apr 30 (opened Apr 17)

- **Why**: ACDE #233 ([forkcast t=5871](https://forkcast.org/calls/acde/233#t=5871)) and ACDC #177 ([t=3532](https://forkcast.org/calls/acdc/177#t=3532), [t=3853](https://forkcast.org/calls/acdc/177#t=3853)) landed the call decisions to add EIP-8141 to the Hegotá `Considered for Inclusion` list and EIP-7716 / EIP-8205 to `Proposed for Inclusion`. The PR formalizes the meta EIP record.
- **Spec changes**: 5 added lines in `EIPS/eip-8081.md` only. No change to EIP-8141's spec text.
- **Significance**: Governance milestone, not a spec change. EIP-8141 is now formally CFI for Hegotá; movement to PFI/SFI requires further client-readiness signals on subsequent ACD calls.

---

## EIP-3607 Carve-Out for Frame Transactions — May 5, 2026

### PR #11272: Disable EIP-3607 check for frame transactions

**Author**: Thegaram | **Merged**: May 5 (opened Feb 6)

- **Why**: EIP-3607 forbids transactions whose `tx.sender` has non-empty, non-delegation code, since a contract account cannot sign a regular ECDSA transaction. Frame transactions intentionally allow `SENDER` frames to originate calls from contract accounts (the whole point of native AA), so applying the EIP-3607 check unconditionally would have blocked smart-account use cases. The discussion sat dormant from Feb 6 to early May; lightclient dismissed an earlier review on Apr 8 and re-approved the cleaned-up version on May 5.
- **Spec changes** (+7/-1):
  - Adds `3607` to the `requires` header (now `1559, 2718, 3607, 4844`)
  - New "Transaction origination" subsection in mempool policy: "Do not apply the restriction put in place by EIP-3607 to frame transactions. Specifically, `SENDER` frames originate calls where `tx.sender` is a contract account. Validation logic for other transaction types remains unchanged, i.e. the transaction is only valid if the sender account's code is either empty or a valid delegation indicator."
- **Key review discussion**: The PR was opened Feb 6 with a single comment from Thegaram pointing at the [magicians thread post #26](https://ethereum-magicians.org/t/eip-8141-frame-transaction/27617/26) for context. It went idle through February-April; lightclient's first approval was dismissed on Apr 8 after later spec churn. Thegaram refreshed the diff in late April and lightclient re-approved on May 5.
- **Significance**: Small in line count (+7/-1) but resolves the longest-pending open spec gap from the original Jan 29 thread. The EIP-8141 ↔ EIP-3607 conflict was the first cross-EIP compatibility issue raised by external reviewers; closing it explicitly (rather than silently) means clients can implement the carve-out without inferring intent. Also makes EIP-3607 the first cross-EIP requirement that EIP-8141 explicitly *opts out* of in its `requires` list, with the carve-out documented in spec text.

---

## Frames Cleanup and Keyed Nonces EIP — May 11, 2026

*Why this mattered: two large merges land together within minutes of each other on May 11. lightclient's PR #11621 is the largest spec-text refactor since PR #11521 (Apr 14), restructuring the spec body and shipping a handful of small functional tweaks. soispoke's PR #11598 lands the standalone Keyed Nonces proposal as EIP-8250, the first EIP to require EIP-8141 as a dependency.*

### PR #11621: Frames cleanup

**Author**: lightclient | **Merged**: May 11 (opened May 7)

- **Why**: Two months of high-velocity merges (Phase 5 through Phase 9) left the spec text full of duplicated reasoning, stale section orderings, and inconsistencies between rationale and behavior. Opened explicitly as a readability sweep: "improve the EIP's readability without changing much functionality." A handful of small functional changes ride along where they fall out of the cleanup naturally.
- **Spec changes** (+185/-345, net -160 lines):
  - **Restructure**: Spec body reorganized under `### Frame Transaction` with `#### Payload Encoding` and `#### Field Definitions` subsections. Field definitions are now centralized into a single bulleted list per object (outer payload, frame object) instead of scattered prose.
  - **Skipped status**: Receipt status `0x3` introduced for frames skipped as part of an atomic batch (previously skipped frames had no distinct status).
  - **FRAMEPARAM operand order**: Order of `FRAMEPARAM` operands explicitly defined (was implicit and inconsistent across rationale).
  - **Default code**: P256 signature scheme removed from default code (only ECDSA secp256k1 remains in the protocol-shipped default code).
  - **Default code on SENDER/DEFAULT**: default code does not revert on `SENDER` or `DEFAULT` frames so top-level value transfers to a default-code account work correctly. This is the visible functional change: the previous default code reverted unconditionally on those modes, breaking simple ETH transfers to a fresh EOA via a frame transaction.
  - **Requires header**: adds `7623` (calldata gas pricing) and `7702` (delegation indicators); both were already implicit in the spec text but not declared.
  - **Abstract and Motivation**: rewritten to lead with the "frames" structural concept and then the post-quantum off-ramp, rather than the other way around. New motivation bullets call out native key rotation, simpler/safer smart accounts via batching, and decentralized fee payment.
- **Key review discussion**: bot reported "✅ All reviewers have approved" the same day the PR opened, no public review comments. samwilsn followed up four days later with an editorial review (EthMagicians post #149) flagging an `APPROVE_PAYMENT_AND_EXECUTION` naming-vs-evaluation-order mismatch, an undefined "paymaster frame" term, a question whether all five new opcodes earn their permanent slot, and a substantive `FRAMEDATACOPY`-reverts-vs-`CALLDATACOPY`-zero-pads design question. None of those gated the merge; auto-merged on May 11 alongside #11598.
- **Significance**: largest spec-text refactor since PR #11521 (Apr 14 broad spec tightening). The "removed P256 from default code" change retracted the hardware-wallet / passkey bridge that had been part of the EOA-support story since PR #11379 (Mar 10); the rationale for dropping it is not in the PR description. The "default code no longer reverts on SENDER/DEFAULT" change is small in implementation but visible to users and indexers (native ETH transfer to a fresh EOA via a frame transaction now succeeds rather than reverting).

### PR #11598: Add EIP — Keyed Nonces for Frame Transactions

**Authors**: soispoke (Thomas Thiery), nerolation, lightclient, vbuterin | **Merged**: May 11 (opened May 4)

- **Why**: A single linear sender nonce blocks privacy-pool flows, smart-wallet session keys, and shared-sender relayer designs from running concurrent transactions. The keyed-nonce proposal was first sketched as a delta against EIP-8141 in PR #11584 (closed in favor of this EIP) and then packaged as a separate Standards Track EIP that requires EIP-8141 rather than as a delta to it.
- **Spec changes**: New EIP at `EIPS/eip-8250.md` introducing `(nonce_key, nonce_seq)` replay-protection. `nonce_key == 0` aliases the legacy account nonce; non-zero keys live in storage of a `NONCE_MANAGER` system contract (revert-only runtime code `0x60006000fd`), keyed by `keccak256(left_pad_32(sender) || uint256_to_bytes32(nonce_key))`. `nonce_seq` is `uint64`, with `MAX_NONCE_SEQ = 2**64 - 1` reserved for exhausted state. Nonce consumption is lifted into the payment-approval transition (the unique `APPROVE` whose scope includes `APPROVE_PAYMENT`) so the spent-once guarantee is atomic with payment, surviving later-frame reverts and `SENDER` atomic-batch rollback. `KEYED_NONCE_FIRST_USE_GAS = 20000` (zero-to-nonzero `SSTORE` reference) is charged on first use of a non-zero key. New `TXPARAM(0x0B)` returns `tx.nonce_key`; `TXPARAM(0x0C)` returns the pre-state legacy sender nonce.
- **Key review discussion**: abcoathup left an approving non-editor review on May 6 ("Looks good enough for a draft", with a small preference for *transaction pool* over *mempool*) and noted explicitly that an editor would still need to sign off; lightclient (as EIP editor) approved on May 11 and auto-merge fired the same minute. The CI flag on commit `4b0dcbfc` (initial commit history contained the unrelated `eip-FOCIL.md` change inherited from #11597's branch) was resolved without forcing a fresh PR.
- **Significance**: first EIP to require EIP-8141 in its dependency header, making the EIP-8141 + EIP-8250 pair the first compose-by-requires AA stack in the EIP series. Establishes the protocol-vs-mempool layering pattern at the EIP level: the mempool one-pending-per-sender rule lives in EIP-8141, the parallel-sequence primitive lives in EIP-8250, and a future keyed-aware mempool policy can compose them without re-litigating EIP-8141's payload schema.

---

## Atomic Batching Extended to All Frame Modes — May 12, 2026

### PR #11652: Support atomic batching with any frames

**Author**: derekchiang | **Merged**: May 12 (opened same day)

- **Why**: Atomic batching was introduced in PR #11395 (Mar 25) limited to consecutive `SENDER` frames. EthMagicians posts #146-147 (alex-forshtat-tbk, derek, May 5) and #150 (alex-forshtat-tbk, May 10) argued the protocol should not constrain which frame modes can batch; the restriction belongs in mempool policy where validation-prefix safety matters, not in protocol semantics. Credits to forshtat for the suggestion (per derek's PR description).
- **Spec changes** (+9/-10, net -1 line):
  - Drops the `frame.mode == SENDER` and `tx.frames[i + 1].mode == SENDER` assertions from atomic-batch validity, allowing the flag on any mode.
  - Generalizes the atomic-batch definition: "a maximal contiguous sequence of frames `[i, j]` where `j > i`, frames `i` through `j - 1` have `ATOMIC_BATCH_FLAG` set, and frame `j` does not". The previous wording required all frames to be `SENDER`.
  - Mempool admission rule expanded: "No frame in the validation prefix may have the `ATOMIC_BATCH_FLAG` set." This is the mempool-side carve-out that keeps the restrictive tier safe while the protocol-level restriction lifts.
  - Atomic-batching rationale rewritten to drop the SENDER-mode language ("multiple frames" rather than "multiple `SENDER` frames").
- **Key review discussion**: lightclient approved within 30 minutes ("LGTM"), auto-merge fired the same day. EthMagicians post #152 (derek, May 12) explained the VERIFY-frame exclusion logic: `SSTORE` on `VERIFY` is banned so the invariant "removing VERIFY frames doesn't change tx behavior" holds, which lets builders aggregate signature verification. Letting VERIFY frames participate in atomic batches would create a path for a later frame to revert a VERIFY frame's effects, breaking the invariant. Derek's post #153 noted no frame currently reverts a VERIFY frame's effects, and post #154 announced the merge.
- **Significance**: small in line count but architecturally important. Encodes the protocol-vs-mempool layering pattern (PR #11580, forshtat's posts) at the spec level: atomic batching is now a protocol primitive applicable to all frame modes, with the restrictive-tier mempool policy carving out validation-prefix atomic batches separately. Opens DEFAULT-frame and (under permissive-tier propagation) VERIFY-frame batching patterns for private pools, post-op cleanup, and revert-protected validation sequences.

---

## EXPIRY_VERIFIER Frame Added — May 14, 2026

### PR #11662: Add EXPIRY_VERIFIER frame for tx expiry

**Author**: nerolation (Toni Wahrstätter) | **Merged**: May 14 (opened May 13)

- **Why**: Frame transactions previously had no protocol-level expiration. Senders relied on either (a) externally-managed off-chain dead-mans-switch deadlines or (b) a custom `VERIFY` frame that read `TIMESTAMP`, which the restrictive mempool tier forbids. Without a sanctioned deadline mechanism, transactions can sit in the mempool indefinitely and be inserted out of the sender's intended time window. PR #11662 introduces a single canonical address whose verifier semantics are codepath-pinned, so deadline checks ride alongside any frame transaction without re-introducing the validation-time `TIMESTAMP` hazard.
- **Spec changes** (+88/-33 lines):
  - New constants: `EXPIRY_VERIFIER = address(0x8141)` and `EXPIRY_DATA_LENGTH = 8`.
  - New "Expiry Verifier Frame" section: a `VERIFY` frame whose `frame.target == EXPIRY_VERIFIER` is the expiry-verifier frame. `frame.data` is interpreted as an 8-byte big-endian unix-seconds deadline; the canonical runtime code at `EXPIRY_VERIFIER` reverts unless `block.timestamp <= expiry_timestamp`. Constraints: `frame.flags == 0`, `frame.value == 0`, `len(frame.data) == 8`; at most one expiry-verifier frame per transaction.
  - Canonical runtime code shipped inline (28-byte sequence `0x60083614600a575f5ffd5b5f3560c01c4211601657005b5f5ffd`); clients may omit explicit EVM execution and perform the deadline check natively provided externally observable behavior is identical.
  - **Sighash change**: expiry-verifier `frame.data` is *not* elided from the signature hash (every other `VERIFY` frame's data is). The deadline is a sender-authored commitment that must not be malleable in transit.
  - **VERIFY semantics relaxed**: "If the frame does not successfully call `APPROVE`, the transaction is invalid" softens to "If the frame reverts, the transaction is invalid". A VERIFY frame can now exit cleanly (without `APPROVE`) and be valid; only an expiry-verifier frame uses this path, but the framing change is general. The static-validation rule that required at least one approval bit on every `VERIFY` frame is removed.
  - **Mempool rules**: validation-prefix dependency list adds "the block timestamp as read by an expiry verifier frame". Public mempool admission MUST drop transactions whose expiry is less than the node's current view of `block.timestamp`. Expiry-verifier frames are exempt from validation trace rules, storage-dependency tracking, and `MAX_VERIFY_GAS`. The `TIMESTAMP` opcode ban gets a single carve-out: permitted in an expiry-verifier frame executing the canonical runtime code at `EXPIRY_VERIFIER`.
  - **Structural rules table**: new `expiry_verify` shape entry (mode `VERIFY`, "Calls the expiry verifier contract"). Mempool-recognized validation-prefix shapes skip expiry-verifier frames when matching (e.g., `[expiry_verify, self_verify]` is recognized as `[self_verify]`).
  - Banned-opcode VERIFY rule for `self_verify`/`only_verify`/`pay` rephrased to "a `self_verify`, `only_verify`, or `pay` frame exits without its required `APPROVE`" (drops the broader VERIFY-without-APPROVE rejection, since expiry-verifier frames are now legitimate VERIFY frames without `APPROVE`).
- **Key review discussion**: lightclient approved on May 14 ("This is great! Thanks Toni!" with a rocket reaction). Toni's PR description flagged one open question: whether the canonical runtime reads `TIMESTAMP` (current draft) or the block header directly. The submitted version reads `TIMESTAMP` and adds the explicit carve-out to the `TIMESTAMP` ban; this choice was not separately debated before merge. Auto-merge fired the same day.
- **Significance**: first new frame shape since the original Jan 29 design and the first time the restrictive mempool tier admits a controlled dependency on `block.timestamp`. The "pinned target address whose runtime is fixed at activation" pattern (similar to `ENTRY_POINT`, EIP-4788, EIP-2935) becomes the second protocol-codepath inside EIP-8141 after the default code. The sighash-non-elision for expiry-verifier `frame.data` is the first carve-out from "VERIFY frame data is elided" since PR #11205 (Jan 29 day-0 fix); future system-frame designs (paymaster reservation, key delegation, etc.) likely follow the same pattern.

---

## Signatures List Merged and EIP-8266 Expiring Nonces Sibling — May 22, 2026

*Why this mattered: two structurally significant merges landed within sixteen hours of each other on May 22. PR #11481 (open since Apr 2) finally lands the `signatures` outer-transaction field, the most structural change to the transaction format since per-frame `value` (PR #11534) and the load-bearing forward-compat hook for PQ signature aggregation. PR #11692 lands as EIP-8266 (Expiring Nonces for Frame Transactions, co-authored by nerolation and lightclient), a second sibling EIP whose `requires` header includes EIP-8141, expanding the compose-by-requires AA stack first established by EIP-8250.*

### PR #11481: Add signatures list to outer tx

**Author**: lightclient | **Merged**: May 22 (opened Apr 2)

- **Why**: Forward-compatibility with PQ signature aggregation. PQ signatures are large (Falcon: ~666 bytes, Dilithium: ~2,420 bytes), and aggregating them will be critical as users migrate. A pre-execution `signatures` field lets future block-level aggregated witnesses elide individual signatures from block serialization while preserving the per-tx signature commitments.
- **Spec changes** (+199/-61, the largest spec-text addition since PR #11521 Apr 14 broad tightening):
  - **New outer field `signatures`**: a list of signature objects on the outer transaction. Each entry carries the signature itself plus metadata (algorithm, message, signer). Verified before any frame executes; during execution, signature validity and signer identity are already established and the contract only needs to check authority.
  - **Default code updated**: secp256k1 VERIFY now reads from the outer `signatures` list rather than `frame.data`. The default code loops through the list to find the entry whose signer matches `tx.sender` (the ergonomic weakness derekchiang flagged on Apr 9 was not separately addressed before merge).
  - **Forward-compat hook**: the design accommodates a future block-level aggregated witness that could replace the per-tx signature list entirely; individual signatures would be elidable from the on-wire form.
- **Key review discussion**: derekchiang's Apr 9 comment on the signature-index discovery problem was the only substantive review thread on the PR. Three subsequent forum comments (0xrcinus, morph-dev, 0xrcinus again, Apr 15-16) debated a default-code optimization that would skip the per-frame VERIFY when `tx.sender` was found in the outer list with a matching sighash; morph-dev rejected the shortcut citing key-rotation cases where the sender's signing key is not the smart-wallet's validation key. jochem-brouwer left a final editorial review on May 22 ("Some small questions and comments!") and the bot fired the auto-merge minutes later. The index-discovery ergonomic question lands as a known limitation rather than a blocker.
- **Significance**: largest spec-text addition since PR #11521 (Apr 14). The signatures field is the first protocol-level structure aimed specifically at PQ aggregation forward-compat (rather than at per-tx flexibility), and the first outer-envelope field added since the original Jan 29 design. The Apr 9 ergonomic concern (index discovery in default code) remains unresolved and is the leading candidate for a follow-up PR.

From lightclient's PR description:

> Any important goal of 8141 is to be forward compatible with signature aggregation techniques, especially with respect to PQ signatures. As those signatures are quite large, aggregating them may become very important as many users begin migrating.

### PR #11692: Add EIP-8266 — Expiring Nonces for Frame Transactions

**Authors**: nerolation (Toni Wahrstätter), lightclient | **Merged**: May 22 (opened May 19)

- **Why**: PR #11662 (May 14) shipped protocol-level transaction expiry as a verifier-frame contract (`EXPIRY_VERIFIER` at `address(0x8141)`). EIP-8266 introduces a complementary primitive: a short-lived replay-protection mode for senders that want to admit multiple pending transactions within a deadline window without growing per-tx state unboundedly. For short-lived intents (atomic swaps, time-boxed sponsorships) the only replay risk worth defending against is rebroadcast inside the deadline window; expiring nonces trade per-tx state growth for a fixed-capacity ring buffer.
- **Spec changes**: New EIP at `EIPS/eip-8266.md` (+162/-0 lines). `requires` header includes EIP-8141 and EIP-8250, making EIP-8266 the second EIP in the compose-by-requires AA stack established by EIP-8250 on May 11. Mechanism in three parts:
  - **Sentinel-mode selection**: a transaction is in expiring-nonce mode when `tx.nonce == 2**64 - 1`. Reusing the existing `nonce` field keeps the payload schema unchanged and lets the canonical signature hash continue to commit to the mode marker.
  - **Ring-buffer state**: a `NONCE_RING` system contract (runtime `0x60006000fd`, revert-only) holds a fixed `RING_CAPACITY = 2**18` slot ring. Consumption happens atomically on the unique payment-approving `APPROVE`. A flat `EXPIRING_NONCE_GAS = 13000` covers the read/write set; the zero-to-nonzero `SSTORE_SET` premium is intentionally omitted because the ring's leaf count is invariant in steady state (paired set+clear keeps trie leaves invariant per consumption).
  - **Deadline enforcement**: the deadline is enforced by reusing PR #11662's `EXPIRY_VERIFIER` frame (8-byte big-endian unix-seconds, capped at `MAX_EXPIRY_SECS = 60`). The sizing invariant `MAX_EXPIRY_SECS × peak_tps ≤ RING_CAPACITY` keeps the ring from evicting a live deadline (~4369 sustained TPS headroom).
- **Composition with EIP-8250**: explicitly non-normative. If both ship, the sentinel collapses into EIP-8250's keyed-nonce framing as a reserved `nonce_key == 2**256 - 1`; `NONCE_RING` storage moves under a distinct prefix inside `NONCE_MANAGER`. Mempool nodes MAY admit multiple pending expiring-nonce transactions per sender, reserving `TXPARAM(0x06)` against the payer's balance for each, breaking with EIP-8141's one-pending-per-sender guidance.
- **Key review discussion**: soispoke (EIP-8250 lead author) left a substantive five-comment inline review on May 19 within hours of submission. Two correctness bugs were caught and acknowledged by nerolation: (1) a same-block replay window at the `d == block.timestamp` boundary, since the strict-inequality check `stored < now` permits a second inclusion in the same block; soispoke's fix is `stored == 0 || stored < now` for stateful validity plus `oldDeadline >= now` for eviction. (2) Pricing concern that the 13000 charge underprices fresh `slot_seen(h)` writes; nerolation reframed the justification to "paired set+clear keeps trie leaves invariant" rather than "overwriting existing slots." soispoke also walked back the privacy framing: the replay key is `compute_sig_hash(tx)`, which only prevents same-hash replay and not nullifier reuse via envelope perturbation. nerolation acknowledged and retracted the privacy use case. abcoathup assigned EIP number 8266 on May 20 with the standard editor note. jochem-brouwer approved on May 22 ("Editorial: LGTM! I'll leave some review comments on EthMagicians later") and auto-merge fired. After the merge, kdenhartog raised a concern (May 24) about ECDSA nonce-reuse attacks and asked for security-considerations text; he edited his own comment shortly after to retract, noting it did not apply to EIP-8141's replay-protection model.
- **Significance**: second sibling EIP in the EIP-8141 compose-by-requires stack (after EIP-8250). The architectural pattern is now consolidated: EIP-8141 owns the protocol primitive (transaction shape, mempool policy, default code); sibling EIPs layer specific replay-protection and expiry policies on top through `requires`. This is the opposite of the absorb-into-base packaging carried by PR #11681. With #11481 also merged the same day, the absorb-into-base proposal now competes against a two-sibling baseline rather than one.

The EthMagicians thread ([topic 28575](https://ethereum-magicians.org/t/eip-8266-expiring-nonces-for-frame-transactions/28575)) was created May 20 with the initial announcement (1 post so far). Discussion of EIP-8266 design questions now happens on this thread rather than the original 27617 Frame Transaction thread.

---

## EIP-8250 Nonce Key Sets and EIP-8272 Recent Roots Merged — June 1 and June 5, 2026

*Why this mattered: the compose-by-requires stack evolves on two fronts within one week. PR #11749 (June 1) ships EIP-8250's first post-merge revision, generalizing single keyed nonces to bounded key sets. PR #11726 / EIP-8272 (June 5) lands as the third sibling EIP in the stack with the broadest sibling-EIP spec change so far: new top-level transaction-payload field, new opcode (`RECENTROOTREFLOAD = 0xB4`), and a new system contract. The compose-by-requires pattern is now demonstrated to evolve in place (EIP-8250 itself updated post-merge) and to scale (three siblings, one with broad new spec surface), each with distinct authoring overlap among soispoke, vbuterin, nerolation, and lightclient.*

### PR #11749: Add support for nonce key sets (EIP-8250 update)

**Author**: soispoke | **Merged**: June 1 (opened same day)

- **Why**: EIP-8250 originally bound each frame transaction to a single keyed nonce stream via `(nonce_key, nonce_seq)`. Some use cases (multi-device wallets coordinating one session, relayer accounts arbitrating shared inclusion, session-key bundles) need to advance multiple keyed sequences in lockstep within a single transaction. The single-key shape forced these flows into either a chain of dependent transactions or off-chain coordination, neither of which composes cleanly with EIP-8141's atomic-frame semantics.
- **Spec changes** (+64/-40, single file `EIPS/eip-8250.md`): generalizes the envelope from `(nonce_key, nonce_seq)` to `(nonce_keys, nonce_seq)`, where `nonce_keys` is a bounded set. All keys in the set advance their sequence atomically as part of the payment-approving `APPROVE`; partial-advancement is not possible. The single-key path remains expressible as a one-element set, preserving backward compatibility with EIP-8266 (the expiring-nonce sentinel still composes with single-key streams).
- **Key review discussion**: auto-merged within 28 minutes of opening with the eth-bot "✅ All reviewers have approved" comment, no public review thread. The PR is the first post-merge revision to any sibling EIP and establishes the precedent that sibling EIPs evolve in place rather than through chain-of-superseding EIPs.
- **Significance**: small in line count but architecturally informative. Demonstrates that sibling EIPs are first-class evolving documents in the EIP-8141 stack, not frozen one-shot proposals. The pattern now visible: EIP-8141 (base) + EIP-8250 (keyed nonces, with key-set support) + EIP-8266 (expiring nonces, composing with EIP-8250) + EIP-8272 (recent roots, composing with EIP-7843). The compose-by-requires camp continues to consolidate.

### PR #11726: Add EIP-8272 — Recent Roots for Frame Transactions

**Authors**: soispoke (Thomas Thiery), vbuterin (Vitalik Buterin), nerolation (Toni Wahrstätter) | **Merged**: June 5 (opened May 25)

- **Why**: EIP-8141's restrictive mempool tier forbids validation from reading arbitrary storage controlled by another account, but some validation rules legitimately need to depend on recent application state (privacy-pool tree roots, wallet authorization roots, account validation roots). Pre-EIP-8272 options were either re-implementing signed-snapshot logic per application (with no shared verification surface) or staying locked out of the public mempool entirely. EIP-8272 makes the snapshot a protocol primitive.
- **Spec changes** (+394/-0, single file `EIPS/eip-8272.md`):
  - **New outer-envelope field**: top-level `recent_root_references` list (max 16 entries per tx), each a `(source_id, slot, root)` tuple. Source identifiers are `keccak256(source_address || salt)` so a single writer publishes to multiple independent root streams. References target slots strictly before `current_slot`, which clients MUST obtain from the EIP-7843 `slotNumber` field rather than deriving from `block.timestamp`.
  - **New system contract**: `RECENT_ROOT_ADDRESS` accepts 64-byte calldata (`salt || root`); `msg.sender` becomes the source address; the entry is stored at `entries[S mod RECENT_ROOT_LENGTH]` keyed by `keccak256(RECENT_ROOT_STORAGE_DOMAIN || source_id || uint64_be(i))`. `RECENT_ROOT_LENGTH = 8192` slots per source. Per-source storage is bounded; total state grows with the number of distinct writers, not with reference volume.
  - **New opcode**: `RECENTROOTREFLOAD = 0xB4` (gas 3) for validation-time read access to a verified reference's root; `TXPARAM(0x0D)` returns the reference count. First sibling-EIP-introduced opcode in the frame-transaction stack and the fifth opcode in the `0xB*` range after `TXPARAM`, `FRAMEDATALOAD`, `FRAMEDATACOPY`, `FRAMEPARAM`.
  - **Validation semantics**: references are checked after the EIP-8141 nonce check and before frame execution, against the transaction pre-state. Competing blocks at the same slot validate against their own pre-states. References are included in `compute_sig_hash(tx)` (not elided by the VERIFY-frame data-elision rule) and frame data MUST NOT mutate the reference set during execution.
  - **Requires header**: `7843, 8141`. EIP-7843 is the first non-8141 requirement in the sibling-EIP stack.
- **Key review discussion**: abcoathup left a non-editor approving review on May 26 ("Looks good enough to merge as a draft"). The CI commit-graph errors from the initial submission were addressed in subsequent commits. lightclient signed off as EIP editor on June 5 and the bot fired the auto-merge minutes later.
- **Significance**: third sibling EIP in the compose-by-requires stack and the first to extend the transaction-payload schema, the opcode space, and the dependency graph beyond EIP-8141 itself. EIP-8250 reshaped an existing field (`nonce`); EIP-8266 used a sentinel; EIP-8272 introduces a brand-new top-level field, a brand-new opcode, and a brand-new non-EIP-8141 dependency. The compose-by-requires pattern is now demonstrated as a general extension surface, not a nonce-mechanism convention. The "pinned target address whose runtime is fixed at activation" pattern (`ENTRY_POINT`, `EXPIRY_VERIFIER`, `NONCE_RING`, `NONCE_MANAGER`, now `RECENT_ROOT_ADDRESS`) is the dominant deployment idiom for sibling-EIP system contracts.

From soispoke's PR description:

> Proposal to add Recent Root References for Frame Transactions.

The EthMagicians thread ([topic 28621](https://ethereum-magicians.org/t/eip-8272-recent-roots-for-frame-transactions/28621)) was created May 20 (1 post, the initial announcement).

---

## Behavior Section Restored and EIP-2542 Withdrawn — June 26 and 30, 2026

Two merges closed out June: one restoring spec text lost in the signatures-list refactor, one marking the first formal supersession of an older proposal by EIP-8141.

### PR #11810: Restore Behavior section (merged June 26)

**Author**: nerolation (Toni Wahrstätter). Merged June 26 after sitting fully approved since June 17.

- **What**: +23/-0, re-adds the `APPROVE` Behavior subsection accidentally deleted by the signatures-list refactor (PR #11481, merged May 22). The restored text defines per-scope semantics: `APPROVE_EXECUTION` requires `resolved_target == tx.sender` and sets `sender_approved`; `APPROVE_PAYMENT` requires prior sender approval and sufficient payer balance, increments the sender's nonce, and collects the transaction's maximum cost from the payer; `APPROVE_EXECUTION_AND_PAYMENT` combines both, with double-approval reverts throughout.
- **Why it matters**: no semantic change beyond restoration, but it completes the first half of the June 17 cleanup pair (with PR #11814) that followed matt's commitment in magicians post #164 after the signatures-list regression. lightclient signed off June 17; nerolation triggered the merge bot June 26.

### PR #11773: EIP-2542 moved to Withdrawn (merged June 30)

**Author**: forshtat (Alex Forshtat, an EIP-8141 co-author). Not a change to EIP-8141 itself, but a governance signal worth recording: EIP-2542 (TXGASLIMIT and CALLGASLIMIT opcodes, a 2020 proposal for transaction and frame gas-limit introspection) was moved to Withdrawn with the header `withdrawal-reason: Superseded by EIP-8141`. Frame transactions expose the same information through `TXPARAM`/`FRAMEPARAM` introspection, covering the use case the old opcodes targeted. This is the first EIP formally withdrawn in favor of EIP-8141.

## Active/Open PRs

*As of July 2, 2026.* These PRs represent active design proposals that may change the spec in the near future.

### PR #11482: Allow using precompiles for VERIFY frames (open since Apr 2)

**Author**: derekchiang

- **Why**: Allow both EOAs and contract accounts to use precompiles for verification, enabling key rotation and shared verification logic.
- **Proposed change**: Designate "signature precompiles" that VERIFY frames can target natively. The precompile reads the public key commitment from storage.
- **All reviewers approved** (as of April 14), awaiting merge. May need rebasing after PR #11521 (Apr 14) and PR #11534 (Apr 16).

From derekchiang's PR description:

> This will allow a contract account to use precompiles for verification, while still having code that serves other purpose (e.g. for execution). As a side benefit, this also enables key rotation, since the precompile reads the public key commitment from storage.

### PR #11555: Add support for guarantors (open since Apr 22)

**Author**: derekchiang

- **Why**: Provides a public-mempool path for transactions whose sender VERIFY logic reads shared state (ERC-20 balances, environmental opcodes), which otherwise violates the restrictive tier's `storage reads only on tx.sender` rule.
- **Proposed change**: Introduce a "guarantor" payer that pays for the transaction *even if sender validation fails*. When a guarantor is present, mempool nodes may skip sender-validation simulation entirely and propagate the transaction on the strength of the guarantor's signature alone.
- **Consequence**: a transaction with a guarantor can use any sender validation logic (including shared-state reads and environmental opcodes) and still propagate through the public mempool. This opens a third path for ERC-20 gas repayment alongside [live (offchain) and permissionless (onchain) paymasters](/mempool-strategy#erc20-paymaster-patterns).
- **Status**: Early proposal; Derek's description notes the authors are still iterating on the idea.

From derekchiang's PR description:

> The idea is to introduce the concept of a "guarantor," which is a payer that pays for the transaction *even if sender validation fails*. As long as a transaction has a guarantor, mempool nodes do not need to check if the sender validation succeeds, and can skip simulation for sender validation altogether. As a result, transactions with a guarantor can use any sender validation logic, including access to shared state, environmental opcodes, etc., and still safely pass through the public mempool.

### PR #11580: Allow payer to approve before sender (open as draft since Apr 29)

**Author**: lightclient

- **Why**: An alternative to derekchiang's guarantors PR (#11555). Instead of introducing a new "guarantor" role with its own approval semantics, lightclient proposes simply relaxing the ordering rule so a payer can call `APPROVE_PAYMENT` before the sender approves execution. A payer that commits to paying gas before sender validation runs absorbs the same economic risk a guarantor would, without a new role in the spec.
- **History**: same content was briefly auto-merged as #11575 on Apr 28 and reverted by #11579 on Apr 29 (lightclient intended it as a draft). Reopened as draft #11580 the same day.
- **Status**: draft; the choice between this and #11555 (guarantors) is the open question for the next sync.

From lightclient's PR description (carried over from #11575):

> Alternative to #11555. I think it is simpler to just allow the payer to approve before the sender instead of adding the full guarantor role.

### PR #11681: Extend with Guarantors, Flexible Nonces, and Signer Binding (open since May 16)

**Author**: pedrouid (Pedro Gomes)

- **Why**: Replaces the closed PR #11643 (Extended Feature Set, May 11 – May 18) after PR #11662 landed EXPIRY_VERIFIER on May 14. With protocol-level expiry now shipped as a verifier-frame contract, the envelope-expiry field in #11643 became redundant. PR #11681 drops the `expiry` envelope field and retains the three remaining features: guarantors, keyed nonces, and signer binding, packaged as a single EIP-8141 amendment rather than a requires-chain of sibling EIPs.
- **Proposed change** (+810/-74, 3 files):
  - **Guarantors**: adopts PR #11555 verbatim. New approval scope `APPROVE_GUARANTEE = 0x4`, a `compute_frame_sig_hash` helper, a `guarantor_approved` transaction-scoped flag, and a canonical-paymaster guarantor mode with a `bumpNonce` entry point. The mempool tier that today drops shared-state-reading sender validation can admit those transactions when a guarantor signature carries the risk.
  - **Keyed Nonces**: mirrors EIP-8250 semantics with one shape change, a single `uint64 signer` envelope field instead of `(nonce_key, nonce_seq)`, so the same identifier indexes both the keyed nonce stream and the registered pubkey. `signer == 0` aliases the legacy account nonce. The position taken in the PR description is that keyed nonces belong inside EIP-8141 rather than as a sibling EIP, because the upgrade path is more efficient when guarantors, keyed nonces, and signer binding ship together and share one system contract.
  - **Signer Binding**: a transaction-scoped `verified_signers` table populated by non-secp256k1 `VERIFY` frames that prove `(digest, address)` against a registered pubkey. `ECRECOVER` consults the table on the hit path; the miss path is byte-identical to upstream, so unrelated contracts behave the same.
  - **Envelope changes**: one new outer field, `signer` (uint64). No `expiry` field; protocol-level expiry is delegated to PR #11662's `EXPIRY_VERIFIER` frame.
  - **System contract**: one `AUTH_MANAGER` at a reserved address (EIP-4788 / EIP-2935 pattern), holding both keyed nonce streams and registered pubkey signers under one storage layout.
  - **Surface area kept small**: zero new opcodes, zero new precompiles, zero account-RLP changes.
- **Relationship to EIP-8250**: if PR #11681 lands, it supersedes EIP-8250 by absorption. The PR description argues explicitly against the requires-chain layering EIP-8250 introduced, on the grounds that a bundled upgrade is more efficient than three sibling EIPs with overlapping system contracts. This is the open architectural question on the table: compose-by-requires (EIP-8250's pattern) vs absorb-into-base (PR #11681's pattern).
- **Status**: Open since May 16, with no activity since May 18 (only bot comments). As of June 15 the PR has passed the one-month-idle mark with no reviewer signoffs, while the compose-by-requires sibling stack grew to four EIPs over the same window. CI initially flagged commit-graph errors which were addressed in subsequent commits. Bot reports 1 more reviewer needed. Sits alongside #11555 (guarantors) as the active packaging question; #11555 may fold into #11681 if the absorption framing converges.

From pedrouid's PR description:

> Guarantors: a payer primitive that admits a transaction to the public mempool even when the sender's `VERIFY` frame is unsafe to simulate. Adopts PR #11555 verbatim. Keyed Nonces: independent replay-protection sequences per `(sender, signer)`. Mirrors EIP-8250 semantics. Diverges only in shape: one `uint64 signer` envelope field instead of `(nonce_key, nonce_seq)`. Signer Binding: tx-scoped `verified_signers` table populated by non-secp256k1 `VERIFY` frames.

### ERC PR #1794: Add ERC-8286 — Modular Accounts for Frame Transactions (open since June 3)

**Author**: chiranjeev13 (node.cm)

- **Why**: Defines how [ERC-7579](https://eips.ethereum.org/EIPS/eip-7579) modular smart accounts implement the EIP-8141 frame-transaction validation flow. ERC-7579 stays the core module system (validator, executor, hook, config modules); ERC-8286 adds the validation path for frame transactions. First application-layer standard built on EIP-8141 and the first frame-transaction proposal tracked in the ERCs repo rather than EIPs. The file's `requires: 7579, 8141` header makes it the only ERC to date that takes EIP-8141 as a hard dependency (ERC-8211 Smart Batching, PR #1638, only mentions EIP-8141 as a forward-compatibility transport, without requiring it).
- **Proposed change** (+425 lines, new file `ERCS/erc-8286.md`): a validator module returns an approval mode that the account applies via the `APPROVE` instruction during a VERIFY frame, mapping ERC-7579's module interfaces onto EIP-8141's validation semantics. Targets the permissions and session-key layer that EIP-8141 protocol defaults deliberately leave open.
- **Status**: Draft; requires one more Editor review (g11tech, jochem-brouwer, samwilsn, xinbenlv) and CI flagged commit errors. EthMagicians thread at [topic 28695](https://ethereum-magicians.org/t/erc-8286-modular-accounts-for-frame-transactions/28695) (1 post, the June 3 announcement, no discussion yet).

### PR #11772: Add EIP-8288 — Frame type for PQ sig and STARK aggregation (open since June 5)

**Authors**: vbuterin (Vitalik Buterin), Thomas Coratger

- **Why**: Post-quantum signature schemes (lattice-based, hash-based) cost ~150-200k gas each to verify and weigh 2-3 kB on the wire. STARK proofs are even worse (128-512 kB on-wire, millions of gas to verify). Including these directly per transaction makes privacy protocols impractical and incompatible with FOCIL and Frame Transactions. EIP-8288 lets transactions declare lightweight `(scheme, data_hash, verification_key)` dependencies that get aggregated off-chain into a single recursive STARK at the block level.
- **Proposed change**: New sibling EIP (`eip-8288.md`, +508 lines) requiring `2718, 2929, 2930, 7702, 8141`. Three moving pieces:
  - **New frame mode `DEP_VERIFY_FRAME_MODE = 3`**: declares a set of dependencies as `(scheme, data_hash, verification_key)` triples (96 bytes each, up to `MAX_DEPENDENCIES_PER_FRAME = 256`). These are not executed as EVM code; they are recorded for the block-level recursive proof. The schemes are `LEANSPHINCS_SCHEME = 0x10` and `LEANSTARK_SCHEME = 0x11` (Lean Ethereum hash-based signatures and STARK proofs respectively, with gas constants `3000` and `30000`).
  - **Block-header field `recursive_stark`**: new top-level header entry `[stark_proof, block_deps_hash]`. Block validity requires the single recursive STARK proves all declared dependencies; `block_deps_hash` commits to the concatenation of all dependency triples. `AGGREGATED_VK` is the fixed verifying key for the recursive scheme.
  - **EVM introspection**: dependencies are visible during execution via existing EIP-8141 `FRAMEPARAM` / `FRAMEDATASIZE` / `FRAMEDATACOPY` opcodes, so contracts can iterate dependency frames and read declared triples. Per-tx caps: `MAX_SIGS_PER_TX = 16`, `MAX_STARKS_PER_TX = 1`.
- **Mempool wrapper**: introduces a wrapper format that bundles transactions with their dependencies and witnesses, with recursive aggregation supported at the mempool layer so nodes can combine proofs to reduce gossip bandwidth. Explicitly compatible with FOCIL (EIP-7805).
- **Significance**: fourth compose-by-requires sibling EIP after EIP-8250, EIP-8266, EIP-8272. First sibling to add a new frame mode (a structural extension to EIP-8141's mode enum, not just an opcode or system contract), and first to introduce a new top-level block-header field. Authored by Vitalik Buterin directly and Thomas Coratger (new contributor), bringing the sibling-author roster beyond the soispoke/nerolation/lightclient cluster. Strategically, the design positions Frame Transactions as the protocol-layer carrier for Ethereum's post-quantum signature transition and for STARK-based privacy/L2 settlement, both load-bearing for the next-major-fork roadmap.
- **Status**: Open since June 5. The file is now committed as `eip-8288.md` (the `eip-9999.md` placeholder was renamed). abcoathup left clarifying comments through June 15, derekchiang reviewed June 22, and EIP editor jochem-brouwer requested editorial changes June 30 ("Exciting EIP"), putting it one editorial pass from merge. EthMagicians thread at [topic 28723](https://ethereum-magicians.org/t/eip-frame-type-for-quantum-resistant-signature-and-stark-aggregation/28723) (6 posts): vbuterin answered the SPHINCS-vs-lattice question in post #5 (Jun 15), defending a purely hash-based base layer; pipavlo82 followed in post #6 (Jun 17) proposing hash-only dependency receipts to address omission accountability.

From vbuterin's PR description:

> Adds a frame type for quantum-resistant Signature and STARK Aggregation. This supports signatures and STARKs (for privacy or for eg. L2s) in a post-quantum world in a highly gas-efficient way, by providing a way for transactions to declare them as "dependencies", in a way that allows the mempool and the block builder to replace them with a recursive STARK proving that they all exist.

### PR #11814: Spec cleanup and clarifications (open since June 17)

**Author**: morph-dev (Milos Stankovic, new contributor)

- **Why**: Follows matt's June 17 commitment (magicians post #164) to restore arbitrary signature-byte support after the signatures-list merge (PR #11481) regressed custom schemes like passkeys. Surfaced by nlordell and DanielVF in posts #161-163.
- **Proposed change** (+103/-73):
  - Raw `sig.signature` bytes are elided from every `sig_hash`. Because all signatures must be valid for the tx to be valid, committing to `sig.msg` indirectly commits to the signatures; eliding the raw bytes keeps signatures insertable after hashing (fixing the circular-dependency regression) and preserves future signature-aggregation compatibility.
  - `sig.signer` may be empty and defaults to `tx.sender`.
  - Mempool: the EXPIRY_VERIFIER frame may only be the first frame in the frame list.
  - Moves several sections to more appropriate locations.
- **Status**: Open; requires one more Author review. morph-dev left several `TODO` markers flagging clarifications to resolve before merge, so this is a cleanup pass rather than a finished fix. Rebased June 30 with morph-dev reporting most TODOs resolved and a few remaining.

### PR #11837: Allow arbitrary signature data for custom verifiers (open since June 25)

**Author**: lightclient

- **Why**: The substantive fix for the signatures-list regression (PR #11481, magicians posts #161-164). Custom verifiers need to sign over the canonical signing hash, but the merged signatures list committed raw signature bytes into that hash, creating the circular dependency nlordell flagged in post #161.
- **Proposed change** (+44/-19): re-adds arbitrary signature data to the outer signatures list, with tweaks to the original idea, so a signature can be computed over the canonical hash and inserted afterward.
- **Status**: Open; eth-bot reports all reviewers approved. Overlaps PR #11814's sig-hash elision approach; how the two compose, or which supersedes, is the open question for the next sync.

From lightclient's PR description:

> I originally intended for this to be in #11481, but at the last minute somehow convinced myself it was unneeded. We realized it is actually required if we want to allow custom verifiers to be able to use the canonical signing hash. I have re-added it here, plus a few tweaks to original idea.

---

## Rejected/Closed PRs

### PR #11404: Simplify approval bits (closed Mar 26)

**Author**: derekchiang

- Proposed an alternative approach to approval bit handling
- Superseded by the mode flags approach (PR #11401)
- Sparked useful discussion: 0xrcinus questioned whether bits were needed at all, Meyanis95 reviewed edge cases

### PR #11408: Migrate EOA default code to EIP-7932 registry (closed Mar 21)

**Author**: SirSpudlington

- Proposed using EIP-7932's signature registry for default code, citing P256 malleability fixes
- lightclient rejected: "We want to reserve the ability to define custom behavior in 8141 default contract and we don't want to rely on another EIP/precompile like this."

### PR #11455: Small tweaks to default code for EIP-7392 compatibility (closed Apr 23)

**Author**: SirSpudlington

- Spiritual successor to the closed PR #11408. No dependency introduced; just aligned default-code values with EIP-7392 for interoperability.
- Never gathered the required reviewer approvals from core authors. Closed without merge after ~4 weeks open.

### PR #11597: Add EIP — Keyed Nonces for Frame Transactions (closed May 4, same day)

**Authors**: soispoke, nerolation, lightclient, vbuterin

- Same content as #11598. Closed without merge because the PR accidentally included an unrelated `eip-FOCIL.md` change in the diff, which broke CI. Resubmitted as PR #11598 the same day with a clean single-file diff.

### PRs #11310, #11314, #11321: Fix broken links (all closed)

**Author**: marukai67

- Three separate PRs attempting to fix allegedly broken links in the spec (to ERC-7562, EIP-2718, and other references)
- All rejected by lightclient with variants of "It's not broken" / "Not broken, thanks though"
- The links use relative paths that work in the EIPs rendering system but may look broken locally

### PR #11584: Add 2D nonces (closed May 8)

**Author**: nerolation (Toni Wahrstätter)

- Sketched `(nonce_key, nonce_seq)` per-sender parallel sequences as a delta against EIP-8141 (28-line draft, opened Apr 30).
- Closed without merge with a one-line "Closing in favor of the EIP for now." after the standalone Keyed Nonces EIP (PR #11598) gathered the same idea into a separate Standards Track proposal with concrete `NONCE_MANAGER` semantics.
- Outcome: keyed-nonce design moves entirely to PR #11598; the delta-against-8141 framing is abandoned.

### PR #11643: Extended Feature Set (closed May 18)

**Author**: pedrouid

- Opened May 11 (+843/-69) bundling guarantors, keyed nonces, signer binding, and envelope expiry into EIP-8141 via two new envelope fields (`signer`, `expiry`) and an `AuthManager` system contract.
- Closed by the author on May 18 in favor of PR #11681. The deciding factor was PR #11662 (EXPIRY_VERIFIER frame, merged May 14): with protocol-level expiry now shipped as a verifier-frame contract, the `expiry` envelope field in #11643 was redundant. PR #11681 drops the expiry field and retains the other three features.
- Net spec impact: zero. The substantive proposal lives in [PR #11681](#pr-11681-extend-with-guarantors-flexible-nonces-and-signer-binding-open-since-may-16).

### PR #11488: Fix spec inconsistencies (closed May 14)

**Author**: chiranjeev13

- Proposed three fixes: a static `VERIFY` frame count check (`<= 2`); stale APPROVE-scope value updates in structural rules (`self_verify` → `APPROVE(0x3)`, `only_verify` → `APPROVE(0x2)`, `pay` → `APPROVE(0x1)`); and removal of the `frame.target != tx.sender` check from default `VERIFY` code to allow any EOA as paymaster. Inspired by node.cm's EthMagicians posts #135-136.
- Sat open from Apr 6 with no reviewer activity. Closed without merge on May 14, three days after PR #11621 (frames cleanup) landed and absorbed the structurally compatible portions of the proposal. The remaining changes were either covered by #11521 (Apr 14) or no longer applied to the current spec.
