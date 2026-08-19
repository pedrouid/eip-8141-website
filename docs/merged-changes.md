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
- **Change**: Default code's `SENDER` mode now simply reverts. The previous logic (which decoded `frame.data` as RLP `[[target, value, data], ...]` when `resolved_target == tx.sender`, and returned successfully with empty data when `resolved_target != tx.sender`) is removed. Net diff: +2/-15. This revert behavior was later superseded by PR #11621.
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
  - **Skipped status**: A distinct receipt status was introduced for frames skipped as part of an atomic batch (the May text used `0x3`; PR #11953 corrected it to `0x2` on Jul 17).
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

*Why this mattered: the compose-by-requires stack evolves on two fronts within one week. PR #11749 (June 1) ships EIP-8250's first post-merge revision, generalizing single keyed nonces to bounded key sets. PR #11726 / EIP-8272 (June 5) lands as the third sibling EIP in the stack with the broadest sibling-EIP spec change so far: new top-level transaction-payload field, new opcode (originally `RECENTROOTREFLOAD = 0xB4`, moved to `0xB5` by #11967), and a new system contract. The compose-by-requires pattern is now demonstrated to evolve in place (EIP-8250 itself updated post-merge) and to scale (three siblings, one with broad new spec surface), each with distinct authoring overlap among soispoke, vbuterin, nerolation, and lightclient.*

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
  - **New opcode**: `RECENTROOTREFLOAD` (gas 3) for validation-time read access to a verified reference's root; the June merge assigned `0xB4` and `TXPARAM(0x0D)` for the reference count. July PRs #11967 and #11930 moved those to `0xB5` and `0x0F` to resolve collisions. First sibling-EIP-introduced opcode in the frame-transaction stack.
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

---

## Signature-List Repair and Cleanup — July 6-7, 2026

Three July merges fixed the June signatures-list regression in the base spec. The current model is no longer "VERIFY data is elided"; frame data stays signed, raw signature bytes with empty `msg` are elided, and custom verifier witnesses live in `ARBITRARY` signature entries exposed through `SIGPARAM`.

### PR #11837: Allow arbitrary signature data for custom verifiers (merged July 6)

**Author**: lightclient

- **Why**: Fixes the circular dependency introduced by PR #11481. Custom verifiers need to sign over `compute_sig_hash(tx)` and insert witness bytes afterward; committing raw signature bytes into the same hash made that impossible.
- **What** (+44/-19): adds `ARBITRARY = 0x0`, shifts `SECP256K1` to `0x1` and `P256` to `0x2`, requires `ARBITRARY.signer` to be empty, adds `SIGPARAM (0xb4)` for signature metadata and `ARBITRARY` byte copying, and elides raw signature bytes for signatures with empty `msg`.
- **Why it matters**: custom schemes get post-hash witness insertion without moving signatures back into `frame.data`, while protocol-validated signatures remain compatible with future aggregation because their raw bytes are not introspectable.

### PR #11870: Gas-accounting nits and out-of-frame exceptional halts (merged July 6)

**Author**: lightclient

- **Why**: Clarifies implementer-facing behavior before the larger cleanup merged.
- **What** (+11/-4): `APPROVE` and frame/signature introspection exceptional-halt outside frame-transaction execution. Intrinsic gas now charges actual frame data and signature data bytes plus per-signature verification cost, rather than charging RLP-encoded aggregate lists.
- **Why it matters**: removes ambiguity for opcodes called in ordinary transactions and aligns intrinsic gas with the bytes that are semantically relevant.

### PR #11814: Spec cleanup and clarifications (merged July 7)

**Author**: morph-dev (Milos Stankovic)

- **Why**: Completes the signatures-list cleanup and resolves several public-mempool ambiguities that accumulated during the May-June merge tempo.
- **What** (+125/-111): signature entries use `scheme`; empty `signer` defaults to `tx.sender` for protocol-validated schemes; default code requires a `SECP256K1` signature at `tx.signatures[0]`; public-mempool expiry-verifier frames may appear only first; public-mempool rules require scope-matching flags, count intrinsic signature validation under `MAX_VERIFY_GAS`, and forbid `VERIFY` after the validation prefix.
- **ERC-20 example correction**: public sponsors no longer check the user's ERC-20 balance onchain during validation. They check signature/frame metadata and accept the frontrunning risk that the user can drain token balance before inclusion.
- **Why it matters**: turns the Phase 21 open question into settled spec text. The outer signatures list is now a concrete protocol surface with three scheme values, default-code index semantics, and explicit `SIGPARAM` introspection.

## Skipped Receipt Status Correction — July 17, 2026

*lightclient — PR #11953, merged July 17, 2026*

- **Receipt model** (+2/-2): skipped atomic-batch frames now record `status = 0x2`, correcting the draft's `0x3` even though `0x2` was the next available status.
- **Introspection**: `FRAMEPARAM(0x05)` now returns `0`, `1`, or `2` for failure, success, or skipped, so execution and receipt views agree.

**Key review discussion**: The correction was submitted with all required approvals already satisfied and auto-merged without substantive reviewer objection.

**Significance**: Changing a receipt-status constant is consensus-visible. Unlike the May cleanup that introduced a distinct skipped state, this merge pins its final wire value.

## Base-Spec Implementation Clarifications — July 17, 2026

Five additional implementation-review merges closed concrete interoperability, security, and accounting gaps in the base EIP. Together they repair one user-visible regression and pin behavior clients must agree on.

### PR #11954: Allow EOA sponsor again

**Author**: lightclient

- **Why**: PR #11814's universal index-0 default-code rule made it impossible for a codeless sender and distinct codeless EOA sponsor to authenticate in one transaction.
- **Change** (+2/-1): default VERIFY uses signature index `0` when approval includes execution and index `1` for payment-only approval. This restores contract-free EOA sponsorship while keeping fixed-position lookup.

### PR #11937: Pin secp256k1 signature encoding

**Author**: svlachakis

- **Why**: the spec passed `v`, `r`, and `s` to `ecrecover` without defining whether `v` was typed-transaction y-parity or precompile-style `27/28`, and without requiring canonical `r`/`s`.
- **Change** (+7/-2): `v` must be `0` or `1`; `r` must be in `(0, SECP256K1N)` and `s` in `(0, SECP256K1N/2]`. lightclient approved with “SGTM.”

### PR #11938: Specify frame-data operand order

**Author**: svlachakis

- **Why**: `FRAMEDATALOAD` and `FRAMEDATACOPY` were the only frame opcodes whose prose did not pin which operand sat on top of the stack.
- **Change** (+15/-5): explicit stack tables now define `offset` above `frameIndex` for load, and `memOffset`, `dataOffset`, `length`, `frameIndex` from top downward for copy. lightclient called it a “Nice catch.”

### PR #11939: Document transaction-wide execution approval

**Author**: svlachakis

- **Why**: `sender_approved` is global for the transaction, but the security section did not warn that one approval authorizes every later `SENDER` frame.
- **Change** (+8): validators must commit to the full frame list or otherwise constrain all later sender frames before approving execution; an explicit-message signature that does not bind the frames is insufficient by itself.

### PR #11941: Apply the EIP-7623 calldata floor

**Author**: svlachakis

- **Why**: the gas formula priced frame/signature bytes per token but omitted EIP-7623's defining minimum floor, allowing large payloads to pay only the standard branch when execution was cheap.
- **Change** (+26): introduces `calldata_tokens` and `charged_gas`; the data-plus-execution branch is floored by `TOTAL_COST_FLOOR_PER_TOKEN * calldata_tokens`, and the transaction must reserve enough gas for the floor.

## Recent-Root Selector Correction — July 18, 2026

*AnkushinDaniil — PR #11930, merged July 18, 2026*

- **Introspection** (+1/-1): `TXPARAM_RECENT_ROOT_REFERENCE_COUNT` moves from stale selector `0x0d` to `0x0f`, avoiding EIP-8250's nonce-key count assignment.
- **Layering**: the correction restores a unique selector map when EIP-8141, EIP-8250, and EIP-8272 are activated together.

**Key review discussion**: EIP-8272 author soispoke approved the one-line correction without requesting further changes.

**Significance**: This changes a transaction-introspection constant. It is the first of three July selector/opcode corrections needed to make the compose-by-requires stack internally consistent.

## Sibling Payload and Mempool Corrections — July 18, 2026

An implementation-driven audit propagated the repaired signatures-list and gas model into EIP-8250 and EIP-8272, while tightening activation and recent-root mempool handling.

### PRs #11960 and #11931: Activation and signature-cost fixes

**Author**: AnkushinDaniil

- **#11960** (+2/-2): aligns `NONCE_MANAGER` activation with EIP-8272's parent-boundary formulation so fork-boundary reorg behavior is explicit.
- **#11931** (+4/-1): restores EIP-8141 signature data and signature verification cost to EIP-8272's `tx_gas_limit` delta.
- **Review**: soispoke approved both changes without further objections.

### PRs #11959 and #11963: Restore signatures in sibling payloads (merged July 18)

**Author**: AnkushinDaniil

- **#11959** (+8/-8): restores `signatures` to EIP-8272's payload and aligns its sighash with EIP-8141 by eliding only empty-`msg` raw signature bytes, not VERIFY frame data.
- **#11963** (+3/-3): makes the same payload and signature-hash correction in EIP-8250, yielding ten fields after replacing `nonce` with `nonce_keys` and `nonce_seq`.
- **Review**: soispoke caught the remaining stale VERIFY-data-elision loop and missing signatures field before approving both fixes.

### PR #11961: Strengthen recent-root public-mempool handling (merged July 18)

**Author**: AnkushinDaniil

- **Why**: many pending transactions can share a root that expires on the same slot boundary, so soft eviction guidance left a predictable cleanup spike.
- **Change** (+5/-1): nodes SHOULD evict expired references, reject references near expiry using a small local margin, and index pending transactions by declared reference and expiry slot.
- **Review**: soispoke rejected a proposed per-`(source_id, slot)` pending cap because it would throttle privacy-pool concurrency; efficient indexing replaced the cap.

## Keyed-Nonce Gas Pricing — July 19, 2026

### PR #11958: Price keyed-nonce payload data

**Author**: AnkushinDaniil

- **Why**: EIP-8250 added up to sixteen 32-byte keys plus a sequence without charging their bytes in the base gas formulas.
- **Change** (+37/-9): defines `nonce_calldata = rlp(nonce_keys) || rlp(nonce_seq)`, prices it in `tx_gas_limit` and the execution branch of `charged_gas`, and includes it in the EIP-7623 floor; EIP-7623 is added to `requires`.
- **Key review discussion**: soispoke required exact encoding and consistent placement in all three gas quantities. nerolation questioned token-vs-byte pricing; reviewers retained EIP-7623 token treatment for consistency with the base EIP.

## Keyed-Nonce Selector Correction — July 19, 2026

*soispoke — PR #11966, merged July 19, 2026*

- **Introspection** (+8/-8): EIP-8250's `TXPARAM_NONCE_KEY_0` moves from base-owned `0x0b` to `0x10`, leaving the base selector range intact.
- **Approval model**: the same patch aligns nonce-consumption and rollback wording with EIP-8141's current transaction-scoped `payer` model.

**Key review discussion**: nerolation approved with “Lgtm” after soispoke marked the correction ready; no alternative assignment was proposed.

**Significance**: This changes an introspection constant and approval text. It resolves the keyed-nonce half of the collision set identified during the July implementation audit.

## Recent-Root Opcode Correction — July 20, 2026

*soispoke — PR #11967, merged July 20, 2026*

- **Opcode assignment** (+3/-1): EIP-8272's `RECENTROOTREFLOAD` moves from `0xb4`, now owned by EIP-8141's `SIGPARAM`, to `0xb5`.
- **Execution behavior**: stack semantics and gas cost remain unchanged; only the consensus-visible opcode byte moves.

**Key review discussion**: nerolation approved the collision fix after soispoke described it as straightforward; no reviewer argued for moving the base opcode instead.

**Significance**: Changing an opcode assignment is structural. It closes the final known `0xb*` collision across the merged frame-transaction sibling stack.

## Arbitrary-Signature Pricing — July 20, 2026

*lightclient — PR #11976, merged July 20, 2026*

- **Signature limits** (+3/-3): each `ARBITRARY` signature entry now adds 100 gas to intrinsic signature validation.
- **DoS resistance**: pricing bounds empty custom entries through the transaction gas limit without introducing a separate count constant.

**Key review discussion**: lightclient replaced PR #11935's proposed `MAX_SIGNATURES = 64` with per-entry pricing; reviewers accepted transaction gas as the bound and the explicit-cap draft closed.

**Significance**: This changes a protocol gas constant and closes the zero-cost signature-list decoding gap.

## Receipt Message Encoding — July 20, 2026

*svlachakis — PR #11942, merged July 20, 2026*

- Defines the frame receipt payload in the fork's `Receipts` message as `[tx-type, cumulative-gas, payer, [[status, gas-used, logs], ...]]`.
- Keeps the networking encoding in EIP-8141 until a matching devp2p protocol version owns it.

## VERIFY Batch Exclusion — July 21, 2026

*AnkushinDaniil — PR #11955, merged July 21, 2026*

- **Approval finality**: public-mempool simulation stops only after the frame that sets `payer` has completed successfully.
- **Atomic batching**: `VERIFY` frames cannot participate in atomic batches, eliminating the rollback path that could leave approval state inconsistent with escrow state.

**Key review discussion**: Nethermind implementation exposed a snapshot rollback bug. Review moved away from preserving approvals through batch rollback; lightclient requested the simpler structural exclusion that became the final patch.

**Significance**: This changes both consensus-valid batch shapes and the point at which mempool validation may stop.

## Transaction-Level Gas Refunds — July 21, 2026

*svlachakis — PR #11940, merged July 21, 2026*

- **Refund accounting**: one transaction-level refund counter uses EIP-3529's one-fifth cap.
- **Rollback**: reverted frames and unrolled batches restore their refund-counter changes.
- **Receipts**: per-frame `gas_used` remains gross, before transaction-level refunds.

**Key review discussion**: lightclient selected a transaction-level counter rather than per-frame refund settlement. soispoke required explicit cap, journal, and receipt wording; AnkushinDaniil clarified that the cap uses full pre-refund transaction gas.

**Significance**: This defines cross-frame refund state and deliberately separates receipt observability from final transaction gas.

## Canonical P256 Signatures — July 21, 2026

*svlachakis — PR #11984, merged July 21, 2026*

- **Signature validation**: protocol P256 entries require canonical low-`s` signatures.
- **Constants**: adds `SECP256R1N` so clients use the same curve-order bound.
- **Wallet behavior**: signers normalize high-`s` output to `n - s`.

**Key review discussion**: reviewers noted that `P256VERIFY` accepts both high- and low-`s` forms. The EIP therefore performs its own canonicality check instead of inheriting the precompile's broader acceptance.

**Significance**: This adds a consensus constant and removes a malleable outer-signature representation.

## Static Atomic-Batch Validation — July 21, 2026

*lightclient — PR #11987, merged July 21, 2026*

- **Frame flags**: the atomic flag is valid only on `DEFAULT` and `SENDER` frames.
- **Batch boundaries**: a flagged frame must be followed by another non-`VERIFY` frame, so `VERIFY` cannot enter or terminate a batch.

**Key review discussion**: the patch turns PR #11955's execution-time safety conclusion into a statically checkable transaction-validity rule; reviewers approved without proposing a competing shape.

**Significance**: This narrows the consensus-valid flag/mode matrix and makes batch safety decidable before execution.

## Blob Transaction Support — July 23, 2026

*svlachakis — PR #11985, merged July 23, 2026*

- **Envelope**: validates KZG versioned hashes and exposes them through `BLOBHASH`.
- **Networking**: adopts the EIP-7594 pooled-transaction sidecar and wrapper.
- **Fees**: the payer escrows and pays blob gas alongside execution gas.
- **Dependencies**: adds EIP-7594 and `VERSIONED_HASH_VERSION_KZG`.

**Key review discussion**: the draft initially forbade blobs. lightclient requested full support and rewrote the change around EIP-7594 rather than leaving frame transactions as a blobless exception.

**Significance**: This adds a new consensus constant and extends the envelope, networking, execution, and payer-fee surfaces.

## Direct Validation-Prefix Evaluation — July 23, 2026

*svlachakis — PR #12001, merged July 23, 2026*

- Allows nodes to evaluate protocol-defined validation prefixes directly when every prefix frame is default code, the expiry verifier, or the canonical paymaster.
- Direct evaluation must produce the same dependencies, gas accounting, and `MAX_VERIFY_GAS` result as EVM simulation.

## Zero-Base-Cost APPROVE — July 23, 2026

*AnkushinDaniil — PR #12003, merged July 23, 2026*

- **Opcode pricing**: `APPROVE` has no base gas charge and pays only memory expansion, matching `RETURN`.
- **Intrinsic accounting**: nonce, payer, and maximum-cost collection work remains covered by transaction intrinsic costs.

**Key review discussion**: reviewers accepted that the opcode's bookkeeping should not be charged twice because the transaction already prices the related protocol work intrinsically.

**Significance**: This changes the gas semantics of a core EIP-8141 opcode.

## Fee-Field Bounds — July 23, 2026

*svlachakis — PR #12005, merged July 23, 2026*

- **Validity**: each execution and blob fee field must be less than `2**256`.
- **Arithmetic**: the explicit bounds make fee multiplication and escrow checks well-defined across clients.

**Key review discussion**: the patch follows existing typed-transaction bounds and merged without disagreement.

**Significance**: This adds consensus-validity bounds to every fee field.

## Complete Fee Settlement — July 28, 2026

*soispoke — PR #11969, merged July 28, 2026*

- **Gas limits**: separates `standard_gas_limit`, `calldata_floor_gas`, and their maximum `max_gas`.
- **Refunds**: applies the EIP-3529 refund first, then enforces the EIP-7623 calldata floor to derive `gas_used`.
- **Payer settlement**: collects `max_cost`, charges execution gas at the effective price plus blob gas at the blob base fee, and returns `payer_refund`.
- **Receipts**: preserves gross per-frame gas even when frame totals do not sum to final transaction gas.

**Key review discussion**: review corrected the calldata-floor/refund order, blobless cases, receipt non-additivity, and the maximum-cost overflow check. lightclient rewrote the final formulas before merge.

**Significance**: This closes the full transaction settlement path across gas limits, refunds, receipts, payer escrow, and blob fees.

## Public-Mempool Lifecycle and Logs — July 28, 2026

*svlachakis — PRs #12007 and #12008, merged July 28, 2026*

- **Replacement**: identifies pending alternatives by `(sender, nonce)` and requires independent validity plus fee bumps.
- **Payer exposure**: reserves aggregate maximum cost against every payer and moves the reservation atomically when a replacement changes payer.
- **Eviction**: removes invalid transactions first, then the nearest expiry, then the lowest effective priority fee.
- **Logs**: defines transaction logs as the concatenation of frame-receipt logs and discards logs unrolled with a failed atomic batch.

**Key review discussion**: AnkushinDaniil and lightclient approved the policy framing. The rules remain public-mempool policy rather than consensus validity, while the log rule removes a cross-client receipt ambiguity.

## Additive Sibling Specifications — July 29, 2026

*soispoke — PRs #11968 and #11970, merged July 29, 2026*

- **EIP-8250**: restates only the nonce-key delta and its added cost terms.
- **EIP-8272**: restates only the recent-root field and its added cost and activation terms.

**Key review discussion**: authors approved defining siblings as deltas so future base-spec fields and settlement formulas are inherited rather than replaced by stale copies.

## Signature Copy Operand Order — July 30, 2026

*Marchhill — PR #12042, merged July 30, 2026*

- **SIGPARAM**: aligns the dynamic copy operand order with `CALLDATACOPY` and `FRAMEDATACOPY`.

**Key review discussion**: the normative one-line correction merged without disagreement and removed an interpreter-order ambiguity.

## Pinned Sibling System Addresses — August 3, 2026

*AnkushinDaniil — PRs #12067 and #12068, merged August 3, 2026*

- **EIP-8250**: pins `NONCE_MANAGER` to `0x...8250`.
- **EIP-8272**: pins `RECENT_ROOT_ADDRESS` to `0x...8272`.

**Key review discussion**: both changes replace placeholder identities with deterministic reserved addresses for client implementation.

## Validation Slot Safety and Reference Links — August 11, 2026

*AnkushinDaniil — PRs #12066 and #12121, merged August 11, 2026*

- **Mempool**: bans `SLOTNUM (0x4b)` during validation-prefix execution because the slot may change between simulation and inclusion.
- **Editorial**: links the first reference to each cited proposal.

**Key review discussion**: EIP-8272's use of slot identity made the environmental dependency concrete; the normative ban merged as a minimal safety correction.

## Explicit Two-Dimensional Frame Gas — August 13, 2026

*lightclient — PR #12062, merged August 13, 2026*

- **Envelope**: replaces each `gas_limit` with `limits = [execution, state]` and groups fee fields under `fees`.
- **Frame model**: gives every frame isolated execution and state-gas pools that cannot borrow across dimensions or frames.
- **Limits**: lowers `FRAME_TX_INTRINSIC_COST` to 12,000, applies the EIP-7825 execution cap, and adds `MAX_VERIFY_STATE_GAS = 500,000`.
- **State accounting**: attributes state growth to the creating frame; later refills reduce that owner's receipt without becoming spendable by another frame.
- **Receipts and introspection**: records `[execution, state]` per frame and adds state-limit/usage selectors to `TXPARAM` and `FRAMEPARAM`.
- **Deployment and value**: prices account creation and code deposit as state gas, adds the EIP-2780 value charge, and follows EIP-7708 transfer-log semantics.
- **Settlement**: floors the execution dimension, adds net state gas, and reserves block execution/state capacity independently under EIP-8037.

**Key review discussion**: jochem-brouwer identified journal-introspection leakage and the risk that untrusted calls could exhaust later state capacity. The final design isolates spendable pools and credits cross-frame refills only to the original frame's receipt. Helkomine questioned whether the EIP-7825 cap should be per frame; the merged text retains a transaction-wide execution cap while keeping budgets per frame.

**Significance**: This is the largest structural change since the April mode/flags rewrite and changes the envelope, execution model, receipts, block accounting, mempool limits, and wallet estimation surface.

## Receipt and Gas Clarifications — August 14, 2026

*AnkushinDaniil — PRs #12026 and #12061, merged August 14, 2026*

- **Signature validation**: protocol signature checks do not add the ECDSA or P256 precompiles to the block access list.
- **Receipts**: the payload has per-frame statuses only; interfaces derive any transaction-level status.

**Key review discussion**: both corrections came from client-implementation ambiguity and merged after the state-gas rewrite absorbed the earlier floor/value wording.

## Approval-Free Atomic Batches — August 14, 2026

*Marchhill — PR #12109, merged August 14, 2026*

- **Frame model**: every frame belonging to an atomic batch, including the unflagged terminating frame, must have approval-scope bits set to zero.
- **APPROVE**: cannot execute inside a batch because its required scope is absent.
- **Security**: batch rollback can no longer reverse nonce collection, payer escrow, or `sender_approved` and convert an ordinary failure into late invalidity.

**Key review discussion**: AnkushinDaniil argued that rollback-based handling made validity depend on executing most of a private transaction and left two possible payers for refund language. Marchhill replaced that design with the static full-scope ban; lightclient approved the result.

**Significance**: This changes both static frame validity and approval semantics to close a rollback-sensitive consensus path.

## Initial Warm Access Set — August 17, 2026

*Marchhill — PR #12113, merged August 17, 2026*

- **Access accounting**: initializes sender, coinbase, and active precompiles as warm and storage keys as empty.
- **Frame targets**: being a target does not itself warm an address.
- **Payer**: warms when `APPROVE` touches it; `ENTRY_POINT` remains cold unless otherwise accessed.

**Key review discussion**: AnkushinDaniil found overlap ambiguity for sender/coinbase/precompile targets and a payer divergence in sponsored post-op flows. The final wording pins all cases; lightclient approved it.

**Significance**: A one-line rule fixes consensus-visible gas divergence across every client implementation.

## Static Signature Data Copy — August 18, 2026

*lightclient — PR #12187, merged August 18, 2026*

- **Opcode set**: adds `SIGDATACOPY (0xb5)` for copying `ARBITRARY` witness bytes.
- **SIGPARAM**: now has a static two-item input and exposes metadata plus arbitrary-witness length only.
- **Sibling interaction**: the assigned byte collides with EIP-8272's current `RECENTROOTREFLOAD (0xb5)` assignment.

**Key review discussion**: the change responds to repeated implementation concern about dynamic stack arity. It merged without a recorded review objection, but the sibling collision remains unresolved as of August 20.

**Significance**: This adds a seventh EIP-8141 opcode and changes the contract interface used by every custom signature verifier.

## Deterministic Validation Opcodes — August 18, 2026

*Marchhill — PR #12167, merged August 18, 2026*

- **Mempool**: removes `ORIGIN`, `TLOAD`, `TSTORE`, and `BLOBHASH` from the banned list because their values are determined by the frame or signed payload.
- **Deployment**: permits `SSTORE` to `tx.sender` storage inside the first deploy frame, matching the existing trace-rule carve-out.
- **Security**: leaves environmental and third-party mutable-state dependencies banned.

**Key review discussion**: AnkushinDaniil and lightclient approved aligning the opcode list with the stated dependency model. Marchhill separately flagged a direct-evaluation dependency-set inconsistency, now tracked in open PR #12160.

**Significance**: This changes both deploy-frame capabilities and public-mempool validation policy while narrowing the ban to actual inclusion-time dependencies.

## Active/Open PRs

*As of August 20, 2026.*

### Base EIP-8141

- **[#12041](https://github.com/ethereum/EIPs/pull/12041), canonical paymaster bytecode** (Marchhill): assembled reference runtime, storage layout, timelocked admin operations, and pinned code hash; awaiting author review.
- **[#12157](https://github.com/ethereum/EIPs/pull/12157), precompile frame dispatch** (Marchhill): clarifies routing, value charges, and why a precompile-targeted VERIFY frame cannot approve.
- **[#12160](https://github.com/ethereum/EIPs/pull/12160), validation dependency set** (AnkushinDaniil): enumerates what clients index for targeted revalidation; lightclient questions whether this belongs in the EIP.
- **[#12162](https://github.com/ethereum/EIPs/pull/12162), validation-stable concurrency** (AnkushinDaniil): relaxes the sender cap for a narrow stable class; review warns against privileging protocol-nonce accounts.
- **[#12198](https://github.com/ethereum/EIPs/pull/12198), expiry as a frame mode** (nerolation): replaces the special-target VERIFY shape with a fourth mode and updates EIP-8266/EIP-8272.
- **[#12203](https://github.com/ethereum/EIPs/pull/12203), expiry-verifier nonce** (Marchhill): states that activation installs code without changing the existing nonce.

### Sibling and Related EIPs

- **[#11772](https://github.com/ethereum/EIPs/pull/11772), EIP-8288 PQ/STARK aggregation** (vbuterin, Thomas Coratger): open with recursive-proof and admitted-set completeness questions.
- **[#12039](https://github.com/ethereum/EIPs/pull/12039), EIP-8250 keyed concurrency** (soispoke): permits distinct nonzero nonce keys to have concurrent pending transactions; approved but unmerged.
- **[#12047](https://github.com/ethereum/EIPs/pull/12047), EIP-7819 trailing data** (shemnon): adds immutable delegation data and an ML-DSA/EIP-8141 validation example.
- **[#12075](https://github.com/ethereum/EIPs/pull/12075), EIP-8361 Transaction Validity Proofs** (Marchhill): mempool-layer STARK proofs for validation prefixes; editor review and CI cleanup remain.
- **[#12077](https://github.com/ethereum/EIPs/pull/12077), EIP-7793 TXINDEX** (Marchhill): uses a frame guard for paid positional enforcement; `TXINDEX` remains banned in validation.
- **[#12110](https://github.com/ethereum/EIPs/pull/12110), EIP-8369 VOPS Profiles** (soispoke): distinguishes mempool and FOCIL checks; claimed-index transport is an activation prerequisite.
- **[#12131](https://github.com/ethereum/EIPs/pull/12131), EIP-8272 recent-root runtime** (AnkushinDaniil): moves bytecode toward geas/sys-asm and synthetic deployment; the `0xb5` collision remains unresolved.
- **[#12139](https://github.com/ethereum/EIPs/pull/12139), EIP-8077 frame source/nonce** (nerolation): clarifies composition with EIP-8141 and EIP-8250; awaiting author review.

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

### PR #11932: Bound the signatures list (closed July 16)

**Author**: AnkushinDaniil

- Proposed `MAX_SIGNATURES = 64` for the unbounded outer list.
- Closed without comment the next day. The same proposal continues as open PR #11935 by svlachakis.

### PR #11957: Exclude keyed-nonce first-use gas from MAX_VERIFY_GAS (closed July 17)

**Author**: AnkushinDaniil

- Proposed excluding `KEYED_NONCE_FIRST_USE_GAS` from the public validation-prefix budget.
- Self-withdrawn eleven minutes after opening. The author concluded the surcharge was the only bound on keyed-read amplification under the current exempt protocol-bookkeeping model and proposed pricing reads directly before revisiting the split.

### PR #11964: Add EIP-8272 source_id test vector (closed July 18)

**Author**: soispoke

- Proposed a deterministic vector for `source_id = keccak256(source_address || salt)`.
- Closed by the EIP-8272 author because the specification already pins 20-byte addresses plus 32-byte salts, making the 52-byte preimage unambiguous.

### PRs #11972 and #11973: Activation and predeploy clarifications (closed July 19)

**Author**: soispoke

- #11972 proposed a `NONCE_MANAGER` collision guard and invalid-block rule for non-empty activation state; #11973 proposed non-mutating EIP-8272 sighash wording and a `RECENT_ROOT_ADDRESS != EXPIRY_VERIFIER` guard.
- Both draft PRs were self-closed within minutes without a recorded rationale. Neither changed the merged specifications.

### PR #11935: Bound the signatures list (closed July 20)

**Author**: svlachakis

- Proposed `MAX_SIGNATURES = 64` to bound decoding work.
- Closed when PR #11976 priced every `ARBITRARY` entry at 100 gas, using the transaction gas limit instead of a separate count constant.

### PR #11956: Batch sponsor repayment with user execution (closed July 21)

**Author**: AnkushinDaniil

- Proposed one atomic batch spanning ERC-20 repayment and user execution.
- lightclient closed it after PR #11955 chose to exclude `VERIFY` frames from atomic batches, making the proposed structure invalid and the separate patch unnecessary.

### PR #11971: Clarify decoding, signing, and activation (closed July 27)

**Author**: soispoke

- Proposed exact RLP shapes, non-mutating sighash construction, P256 validation, and expiry-verifier activation wording.
- Closed after PR #11984 resolved the P256 portion; lightclient questioned whether the remaining changes justified a combined patch.

### PR #12004: Add a canonical P256 pseudo-account (closed July 24)

**Author**: AnkushinDaniil

- Proposed a reserved pseudo-account for protocol P256 verification.
- lightclient argued that P256 is a cryptographic operation, not a complete account interface: the pseudo-account would lack ERC-1271, permit, callbacks, and self-deployment. The author agreed and closed it.

### PR #12010: Stabilize the canonical-paymaster identity (closed July 24)

**Author**: AnkushinDaniil

- Proposed recognizing a canonical paymaster through stable identity and deployment rules.
- Closed after review found its EIP-7702 recognition model unsound. The useful pinned-code-hash piece continues in draft PR #12012.

### PRs #12011 and #12012: Registry and paymaster drafts (closed July 29-30)

- **#12011**: closed after lightclient rejected an explicit protocol signature registry; new fixed-cost schemes should arrive through their own specifications.
- **#12012**: closed when the canonical paymaster implementation moved to assembled-runtime PR #12041.

### PRs #12086 and #12091: Inclusion checks (closed August 6 and 14)

- **#12086**: frame-aware FOCIL validity was superseded by the profile-based EIP-8369 in PR #12110 after review found the unpaid per-list validation work unbounded.
- **#12091**: block inclusion gating and payer solvency became redundant after PR #12062 specified exact two-dimensional block reservations.

### PRs #11482, #11555, #11580, and #11681: Precompile, guarantor, and bundled extensions (closed August 14)

- **#11482, #11555, #11681**: lightclient closed the stale branches and invited rebased proposals if the authors want to continue them.
- **#11580**: closed because the complexity of payer-before-sender approval was no longer justified.

### PR #12155: Batch approval duplicate (closed August 16)

- Superseded by merged PR #12109, which landed the same static no-approval rule for every batch frame.

### PRs #12161 and #12168: Wallet-specific admission rules (closed August 13-14)

- **#12161**: canonical passkey-wallet recognition overlapped earlier rejected P256-account directions; the general admission work continues in #12160 and #12162.
- **#12168**: signer-code restrictions were left to wallet validation rather than protocol-level `SIGPARAM` rules.

### PRs #12175 and #12176: Paymaster service and gas-estimation RPC drafts (closed August 17)

- Both unsolicited drafts closed within an hour without review. Neither is part of the current EIP or tooling surface.
