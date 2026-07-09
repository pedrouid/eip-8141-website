# What Changed From Original to Latest

---

## Structural Comparison

| Aspect | Original (Jan 29) | Latest (Jul 9) |
|---|---|---|
| **Opcodes** | `APPROVE`, `TXPARAMLOAD`, `TXPARAMSIZE`, `TXPARAMCOPY` (4) | `APPROVE (0xaa)`, `TXPARAM (0xb0)`, `FRAMEDATALOAD (0xb1)`, `FRAMEDATACOPY (0xb2)`, `FRAMEPARAM (0xb3)`, `SIGPARAM (0xb4)` (6) |
| **APPROVE mechanism** | Return codes 0-4 at top-level frame | Transaction-scoped with scope operand (0x1, 0x2, 0x3), callable at any depth, double-approval prevention |
| **APPROVE scope** | 0x0 (execution), 0x1 (payment), 0x2 (both) | 0x1 (payment), 0x2 (execution), 0x3 (both) |
| **APPROVE restriction** | Must be top-level frame | `ADDRESS == frame.target` only |
| **Frame structure** | `[mode, target, gas_limit, data]` | `[mode, flags, target, gas_limit, value, data]` (mode/flags split, per-frame `value`) |
| **Outer envelope signatures** | No outer signatures field; all signatures carried inside `frame.data` of VERIFY frames | `signatures` outer-envelope list carrying `(scheme, signer, msg, signature)` entries. `SECP256K1` and `P256` are protocol-validated; `ARBITRARY` carries custom witness bytes for account-code validation. Default code reads index `0`. Forward-compat hook for PQ signature aggregation (PR #11481, repaired by #11837/#11814) |
| **Mode field** | Just mode value (0, 1, 2) | Pure mode (0, 1, 2) with separate `flags` field |
| **Flags field** | N/A | Bits 0-1 = approval scope constraint; bit 2 = atomic batch flag |
| **Frame modes** | DEFAULT, VERIFY, SENDER | Same three modes plus an expiry-verifier shape (`VERIFY` with `target == EXPIRY_VERIFIER`) admitted by PR #11662 (merged May 14) |
| **Atomic batching** | Not supported | Bit 2 of flags; any frame mode may participate (PR #11652 merged May 12 lifted the previous SENDER-only restriction). Restrictive mempool tier separately forbids the flag inside the validation prefix |
| **MAX_FRAMES** | `10^3` (1,000) | `64` |
| **Per-frame cost** | None | `FRAME_TX_PER_FRAME_COST = 475` gas |
| **EOA support** | None | Default code: ECDSA secp256k1 (low-`s` enforced) verification in VERIFY only, requiring the matching signature at `tx.signatures[0]`. P256 is protocol-validated in the outer list but not accepted by codeless EOA default code. `SENDER` and `DEFAULT` no longer revert (PR #11621): top-level value transfer to a default-code account completes. The earlier RLP-call-batch payload was removed by PR #11577 (Apr 29) once native batching plus per-frame `value` covered the multi-call use case |
| **Signature hash** | VERIFY data NOT elided (bug) | Raw `sig.signature` bytes are elided for signatures with empty `msg`; frame data stays covered. EIP-2718 type-byte prefix included (PR #11544, merged Apr 22). The Jul 6-7 repair (#11837/#11814) fixes the circular dependency introduced by the signatures-list merge |
| **Receipt status** | Not specified | `0x0` failure, `0x1` success, `0x3` skipped-batch (introduced by PR #11621, merged May 11) |
| **Expiry mechanism** | None | `EXPIRY_VERIFIER = address(0x8141)` canonical contract; an expiry-verifier `VERIFY` frame carries an 8-byte unix-seconds deadline as `frame.data`. Public-mempool admission MUST drop transactions whose expiry has passed; `TIMESTAMP` opcode gets a carve-out for this canonical runtime only. If present, the expiry frame must be first |
| **Mempool policy** | Not defined (just "Security Considerations" section) | Comprehensive: validation prefixes, canonical paymaster, banned opcodes, MAX_VERIFY_GAS including intrinsic signature-validation cost, expiry-verifier admission, approval-scope flag matching, and no VERIFY frames after the validation prefix |
| **Requires header** | `2718, 4844` | `1559, 2718, 3607, 4844, 7623, 7702` (PR #11567 dropped 7997 on Apr 30; PR #11272 added 3607 on May 5 with an explicit carve-out for frame transactions; PR #11621 added 7623 and 7702 on May 11) |
| **EIP-3607 origination check** | Inherited unconditionally (would block contract-account senders) | Carved out for frame transactions: `SENDER` frames may originate from contract accounts; non-frame txs unchanged (PR #11272, merged May 5) |
| **Authors** | 7 co-authors | 8 co-authors (derekchiang added) |
| **Receipt** | Not specified in detail | Includes `payer` field and per-frame `[status, gas_used, logs]`; `status == 0x3` for skipped batch entries |
| **SENDER frame requirements** | Could execute without prior approval | Requires `sender_approved == true` |
| **Value in frames** | Not in frame structure | Per-frame `value` field; non-zero only in SENDER frames. DEFAULT/VERIFY observe `CALLVALUE = 0` |
| **VERIFY frame behavior** | State changes allowed | Behaves as `STATICCALL`, no state changes. `APPROVE` requirement narrowed to `self_verify`/`only_verify`/`pay` shapes (PR #11662 relaxed the previous "every VERIFY frame must call APPROVE" rule to "if the frame reverts, the tx is invalid") |
| **Target resolution** | Direct use of `frame.target` | Explicit `resolved_target` (null target resolves to `tx.sender`) |
| **Deterministic deployer** | Not specified | EIP-7997 is the canonical-but-optional factory; any stateless factory qualifies under the deploy-frame trace rules (PR #11567, merged Apr 30) |
| **Deploy-frame mempool rule** | N/A | Trace-rule policy: write carve-out for `CREATE`/`CREATE2`/`SETDELEGATE` installing code at `tx.sender` and `SSTORE`s on `tx.sender`'s storage; any contract may be `frame.target` (PR #11567) |
| **Fork inclusion status** | N/A | CFI in Hegotá fork meta EIP-8081 (PR #11537, merged Apr 30) |
| **Sibling EIPs** | N/A | EIP-8250 Keyed Nonces, EIP-8266 Expiring Nonces, and EIP-8272 Recent Roots are merged siblings. EIP-8288 PQ signature/STARK aggregation remains open with requested changes and proof-security review |

## Key Philosophical Shifts

The overall trajectory: **from expressive abstraction toward deployability, compatibility, and mempool safety**. The early drafts prioritized flexibility; the later drafts constrain that flexibility into something the network can reason about.

### 1. From Smart-Account-Only to EOA-First

The original spec assumed users would have smart accounts. The latest spec makes EOAs first-class citizens with default code, recognizing that most users won't migrate to smart accounts immediately.

This was the single biggest change to the EIP's trajectory, driven by adoption concerns raised by DanielVF and derek's commercial AA experience showing that hardware wallets and most consumer wallets are slow/unwilling to adopt smart contract accounts.

### 2. From Minimal Mempool Guidance to Full Policy

The original spec had a brief "Security Considerations" section with general warnings about DoS vectors. The latest has a comprehensive mempool policy with:
- Specific structural rules (four recognized validation prefixes)
- Banned opcodes list
- Gas caps (MAX_VERIFY_GAS = 100,000)
- A canonical paymaster contract (removing ERC-7562's reputation/staking complexity)
- Non-canonical paymaster handling
- Explicit acceptance and revalidation algorithms

### 3. From Top-Level APPROVE to Transaction-Scoped APPROVE

Originally, approval status was determined by the return code of the top-level frame, similar to how `RETURN` works but with extended codes (2, 3, 4). This had several problems:
- Proxy-based accounts couldn't adopt it (proxy returns 0 or 1)
- APPROVE in nested calls required awkward propagation
- Return codes >1 from `CALL` broke backwards compatibility assumptions

Now APPROVE is a dedicated opcode that updates transaction-scoped variables (`sender_approved`, `payer_approved`) directly, from anywhere in the call stack. It's cleaner and more compatible with existing smart account patterns.

### 4. From Generic TXPARAM Opcodes to Specialized Data Access

The original `TXPARAMLOAD/SIZE/COPY` trio treated all transaction parameters uniformly. The redesign recognized that:
- Most parameters are scalar (32 bytes or less) → `TXPARAM` handles these
- Only frame data is variable-length → `FRAMEDATALOAD`/`FRAMEDATACOPY` handle this

This is more gas-efficient and easier to reason about.

### 5. From No Batching Control to Explicit Atomicity

The original spec had no mechanism for atomic multi-call. The latest provides fine-grained control via the atomic batch flag in the `flags` field (originally bit 11 of the packed `mode`, now bit 2 of the separate `flags` after the mode/flags split), allowing users to specify exactly which SENDER frames must succeed together. This was a response to the universal expectation from developers that "if I'm batching operations, they should be atomic," while preserving the non-atomic default needed for paymaster patterns.

### 6. Mode/Flags Split and Signature Hash Repair

The original spec simply checked `mode == VERIFY` to elide frame data from the signature hash. An intermediate version packed flags into the upper bits of mode, requiring `(frame.mode & 0xFF) == VERIFY` to mask them out. PR #11521 split mode and flags into separate fields, but the July signatures-list repair changed the hash model again: frame data is now covered, while raw `sig.signature` bytes are elided for signature entries whose `msg` is empty. That moves self-referential witness data into the outer `signatures` list and exposes custom bytes through `SIGPARAM`.

### 7. From "No Value Field" to Per-Frame Value

The original spec deliberately had no `value` field in frames, on the principle that account code could send ETH via its own call encoding. That rationale held as long as every user had a smart account capable of encoding sub-calls. As the spec shifted toward an EOA-first, good-out-of-the-box experience (see Phase 1), the case for native per-frame `value` became hard to resist: a simple ETH transfer should not require the sender to construct an RLP-encoded call list, and wallets should not have to ship batching boilerplate to achieve parity with a regular transaction. PR #11534 (Apr 16) added a `value` field to the frame tuple, restricted to `SENDER` frames so that `VERIFY` stays `STATICCALL`-like and `DEFAULT` does not require `ENTRY_POINT` to fund transfers. The original rationale section was renamed from "No value in frame" to "Per-frame value."

---

## Active Proposals That May Change the Comparison

As of July 9, 2026, several open PRs propose changes that would extend this comparison table:

| Proposal | PR | Impact |
|---|---|---|
| **Precompile-based VERIFY** | [#11482](https://github.com/ethereum/EIPs/pull/11482) | Would allow VERIFY frames to target signature precompiles directly, changing the verification model (all reviewers approved) |
| **Guarantors** | [#11555](https://github.com/ethereum/EIPs/pull/11555) | Would introduce a "guarantor" payer that pays even if sender validation fails, letting mempool nodes skip sender simulation and admit shared-state-reading VERIFY frames |
| **Payer approves before sender** | [#11580](https://github.com/ethereum/EIPs/pull/11580) | Alternative to #11555: relaxes the ordering rule so a payer can approve before the sender, letting a payer commit to gas without simulating sender validation. Briefly auto-merged as #11575 on Apr 28 and reverted by #11579 on Apr 29; reopened as a draft |
| **Extend with Guarantors, Flexible Nonces, and Signer Binding** | [#11681](https://github.com/ethereum/EIPs/pull/11681) | Pedro Gomes's +810/-74 bundle folding guarantors (#11555), keyed nonces (EIP-8250-equivalent), and signer binding (EIP-8164-equivalent) into EIP-8141 via one `signer` envelope field and an `AUTH_MANAGER` system contract. Successor to the closed PR #11643 after PR #11662 (EXPIRY_VERIFIER) settled the expiry design as a verifier-frame contract; inverts the requires-chain layering that EIP-8250 established by absorbing the keyed-nonce and signer-binding features into EIP-8141 itself |
| **EIP-8288 PQ/STARK aggregation** | [#11772](https://github.com/ethereum/EIPs/pull/11772) | Still open after editorial requested changes and proof-security review. Would add `DEP_VERIFY_FRAME_MODE = 3`, block-level `recursive_stark`, and Lean Ethereum dependency schemes |
| **Frame returndata opcodes** | Under discussion (post #137) | Proposed `FRAMERETURNDATASIZE`/`FRAMERETURNDATACOPY` to enable multi-step flows, no PR yet |
