# What Changed From Original to Latest

---

## Structural Comparison

| Aspect | Original (Jan 29) | Latest (Aug 20) |
|---|---|---|
| **Opcodes** | `APPROVE`, `TXPARAMLOAD`, `TXPARAMSIZE`, `TXPARAMCOPY` (4) | `APPROVE (0xaa)`, `TXPARAM (0xb0)`, `FRAMEDATALOAD (0xb1)`, `FRAMEDATACOPY (0xb2)`, `FRAMEPARAM (0xb3)`, `SIGPARAM (0xb4)`, `SIGDATACOPY (0xb5)` (7, #12187). `SIGDATACOPY` currently collides with EIP-8272's `RECENTROOTREFLOAD (0xb5)` |
| **APPROVE mechanism** | Return codes 0-4 at top-level frame | Transaction-scoped with scope operand (0x1, 0x2, 0x3), callable at any depth, double-approval prevention |
| **APPROVE scope** | 0x0 (execution), 0x1 (payment), 0x2 (both) | 0x1 (payment), 0x2 (execution), 0x3 (both) |
| **APPROVE restriction** | Must be top-level frame | `ADDRESS == frame.target` only |
| **Frame structure** | `[mode, target, gas_limit, data]` | `[mode, flags, target, [execution_limit, state_limit], value, data]` (mode/flags split, per-frame value, explicit two-dimensional gas) |
| **Outer envelope signatures** | No outer signatures field; all signatures carried inside `frame.data` of VERIFY frames | `signatures` outer-envelope list carrying `(scheme, signer, msg, signature)` entries. `SECP256K1` and canonical low-`s` `P256` are protocol-validated; `ARBITRARY` carries custom witness bytes for account-code validation and costs 100 gas per entry. Default code reads index `0` when approving execution and index `1` for payment-only approval. Forward-compat hook for PQ signature aggregation (PR #11481, repaired by #11837/#11814, sponsorship-restored by #11954, and hardened by #11976/#11984) |
| **Mode field** | Just mode value (0, 1, 2) | Pure mode (0, 1, 2) with separate `flags` field |
| **Flags field** | N/A | Bits 0-1 = approval scope constraint; bit 2 = atomic batch flag, valid only for DEFAULT and SENDER |
| **Frame modes** | DEFAULT, VERIFY, SENDER | Same three modes plus an expiry-verifier shape (`VERIFY` with `target == EXPIRY_VERIFIER`) admitted by PR #11662 (merged May 14) |
| **Atomic batching** | Not supported | Bit 2 of flags on DEFAULT/SENDER frames. VERIFY cannot participate in or terminate a batch, and no frame belonging to a batch may carry approval scope, including its terminating frame (#12109) |
| **MAX_FRAMES** | `10^3` (1,000) | `64` |
| **Per-frame cost** | None | `FRAME_TX_PER_FRAME_COST = 475` gas |
| **Fee-field bounds** | Not explicit | Every execution and blob fee field must be less than `2**256` (PR #12005) |
| **Gas settlement** | One gas limit with no cross-frame refund model | Explicit per-frame execution/state pools under EIP-8037; `FRAME_TX_INTRINSIC_COST = 12,000`; EIP-7825 caps intrinsic plus execution budgets; state gas remains frame-isolated; the EIP-7623 floor applies to execution and net state gas is added; cross-frame state refills reduce the owning frame's receipt (#12062) |
| **APPROVE gas** | N/A | No base gas cost; only memory expansion, matching `RETURN` (PR #12003) |
| **Blob support** | Blob hashes present only through inherited EIP-4844 context | Full KZG versioned-hash validation, `BLOBHASH`, EIP-7594 pooled-transaction wrapper, and payer blob-fee settlement (PR #11985) |
| **EOA support** | None | Default code: ECDSA secp256k1 verification in VERIFY only, with canonical `v`/`r`/low-`s` encoding. Execution approval uses `tx.signatures[0]`; payment-only EOA sponsorship uses index `1`. P256 is protocol-validated in the outer list but not accepted by codeless EOA default code. `SENDER` and `DEFAULT` no longer revert (PR #11621): top-level value transfer to a default-code account completes. The earlier RLP-call-batch payload was removed by PR #11577 (Apr 29) once native batching plus per-frame `value` covered the multi-call use case |
| **Signature hash** | VERIFY data NOT elided (bug) | Raw `sig.signature` bytes are elided for signatures with empty `msg`; frame data stays covered. EIP-2718 type-byte prefix included (PR #11544, merged Apr 22). The Jul 6-7 repair (#11837/#11814) fixes the circular dependency introduced by the signatures-list merge |
| **Receipt status** | Not specified | `0x0` failure, `0x1` success, `0x2` skipped-batch (corrected by PR #11953, merged Jul 17) |
| **Expiry mechanism** | None | `EXPIRY_VERIFIER = address(0x8141)` canonical contract; an expiry-verifier `VERIFY` frame carries an 8-byte unix-seconds deadline as `frame.data`. Public-mempool admission MUST drop transactions whose expiry has passed; `TIMESTAMP` opcode gets a carve-out for this canonical runtime only. Public propagation requires the expiry frame to be first |
| **Mempool policy** | Not defined (just "Security Considerations" section) | Four prefixes, canonical paymaster, `MAX_VERIFY_GAS = 100,000`, `MAX_VERIFY_STATE_GAS = 500,000`, trace rules, deterministic replacement/eviction, and per-payer exposure. `SLOTNUM` is banned; deterministic `ORIGIN`, `TLOAD`, `TSTORE`, and `BLOBHASH` are allowed (#12167) |
| **Requires header** | `2718, 4844` | `1559, 2718, 2780, 3529, 3607, 4844, 7594, 7623, 7702, 7708, 7825, 8037` (two-dimensional gas and value accounting added the August dependencies) |
| **EIP-3607 origination check** | Inherited unconditionally (would block contract-account senders) | Carved out for frame transactions: `SENDER` frames may originate from contract accounts; non-frame txs unchanged (PR #11272, merged May 5) |
| **Authors** | 7 co-authors | 10 co-authors (derekchiang, nerolation, and svlachakis added) |
| **Receipt** | Not specified in detail | Includes `payer` and per-frame `[status, [execution_gas, state_gas], logs]`; no transaction-level status. Transaction logs concatenate frame logs in order, while failed-batch logs are discarded (#12008, #12061, #12062) |
| **Initial warm state** | Inherited implicitly | Sender, coinbase, and active precompiles start warm; storage keys start cold; frame targeting does not warm; payer warms when `APPROVE` touches it (#12113) |
| **SENDER frame requirements** | Could execute without prior approval | Requires `sender_approved == true` |
| **Value in frames** | Not in frame structure | Per-frame `value` field; non-zero only in SENDER frames. DEFAULT/VERIFY observe `CALLVALUE = 0` |
| **VERIFY frame behavior** | State changes allowed | Behaves as `STATICCALL`, no state changes. `APPROVE` requirement narrowed to `self_verify`/`only_verify`/`pay` shapes (PR #11662 relaxed the previous "every VERIFY frame must call APPROVE" rule to "if the frame reverts, the tx is invalid") |
| **Target resolution** | Direct use of `frame.target` | Explicit `resolved_target` (null target resolves to `tx.sender`) |
| **Deterministic deployer** | Not specified | EIP-7997 is the canonical-but-optional factory; any stateless factory qualifies under the deploy-frame trace rules (PR #11567, merged Apr 30) |
| **Deploy-frame mempool rule** | N/A | Trace-rule policy: write carve-out for `CREATE`/`CREATE2`/`SETDELEGATE` installing code at `tx.sender` and `SSTORE`s on `tx.sender`'s storage; any contract may be `frame.target` (PR #11567) |
| **Fork inclusion status** | N/A | CFI in Hegotá fork meta EIP-8081 (PR #11537, merged Apr 30) |
| **Sibling EIPs** | N/A | EIP-8250 Keyed Nonces, EIP-8266 Expiring Nonces, and EIP-8272 Recent Roots are merged siblings; EIP-8288 remains open. EIP-8250 and EIP-8272 now specify additive deltas and pinned system addresses. A new unresolved collision exists because both EIP-8141 `SIGDATACOPY` and EIP-8272 `RECENTROOTREFLOAD` claim `0xb5` |

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

Now APPROVE is a dedicated opcode that updates transaction-scoped approval context (`sender_approved` and `payer`) directly, from anywhere in the call stack. It's cleaner and more compatible with existing smart account patterns.

### 4. From Generic TXPARAM Opcodes to Specialized Data Access

The original `TXPARAMLOAD/SIZE/COPY` trio treated all transaction parameters uniformly. The redesign recognized that:
- Most parameters are scalar (32 bytes or less) → `TXPARAM` handles these
- Only frame data is variable-length → `FRAMEDATALOAD`/`FRAMEDATACOPY` handle this

This is more gas-efficient and easier to reason about.

### 5. From No Batching Control to Explicit Atomicity

The original spec had no mechanism for atomic multi-call. The latest provides fine-grained control via the atomic batch flag in the `flags` field (originally bit 11 of the packed `mode`, now bit 2 of the separate `flags` after the mode/flags split), allowing users to specify exactly which SENDER frames must succeed together. This was a response to the universal expectation from developers that "if I'm batching operations, they should be atomic," while preserving the non-atomic default needed for paymaster patterns.

### 6. Mode/Flags Split and Signature Hash Repair

The original spec simply checked `mode == VERIFY` to elide frame data from the signature hash. An intermediate version packed flags into the upper bits of mode, requiring `(frame.mode & 0xFF) == VERIFY` to mask them out. PR #11521 split mode and flags into separate fields, but the July signatures-list repair changed the hash model again: frame data is now covered, while raw `sig.signature` bytes are elided for signature entries whose `msg` is empty. That moves self-referential witness data into the outer `signatures` list. `SIGPARAM` exposes metadata and witness length; `SIGDATACOPY` copies custom bytes with a static stack shape.

### 7. From "No Value Field" to Per-Frame Value

The original spec deliberately had no `value` field in frames, on the principle that account code could send ETH via its own call encoding. That rationale held as long as every user had a smart account capable of encoding sub-calls. As the spec shifted toward an EOA-first, good-out-of-the-box experience (see Phase 1), the case for native per-frame `value` became hard to resist: a simple ETH transfer should not require the sender to construct an RLP-encoded call list, and wallets should not have to ship batching boilerplate to achieve parity with a regular transaction. PR #11534 (Apr 16) added a `value` field to the frame tuple, restricted to `SENDER` frames so that `VERIFY` stays `STATICCALL`-like and `DEFAULT` does not require `ENTRY_POINT` to fund transfers. The original rationale section was renamed from "No value in frame" to "Per-frame value."

### 8. From One Gas Limit to Explicit Execution and State Budgets

The original frame carried one gas limit. PR #12062 split that field into per-frame execution and state budgets to align EIP-8141 with EIP-8037 while preserving isolation between mutually distrusting frames. The two pools never borrow from each other. This lets builders reserve block capacity exactly, lets validation code inspect later frames' budgets before approving, and prevents a user frame from consuming state gas reserved for a paymaster post-op. The tradeoff is a wider envelope, two-dimensional estimation, and journaled cross-frame attribution when later execution clears state created by an earlier frame.

---

## Active Proposals That May Change the Comparison

As of August 20, 2026, these open PRs may extend the comparison table:

| Proposal | PR | Impact |
|---|---|---|
| **EIP-8288 PQ/STARK aggregation** | [#11772](https://github.com/ethereum/EIPs/pull/11772) | Still open after editorial requested changes and proof-security review. Would add `DEP_VERIFY_FRAME_MODE = 3`, block-level `recursive_stark`, and Lean Ethereum dependency schemes |
| **Keyed-nonce concurrency** | [#12039](https://github.com/ethereum/EIPs/pull/12039) | Would permit more than one pending transaction per sender when nonzero nonce keys are distinct and validation proves key ownership |
| **Canonical paymaster bytecode** | [#12041](https://github.com/ethereum/EIPs/pull/12041) | Replaces the closed Solidity draft with assembled reference runtime, storage layout, timelocked admin operations, and a pinned code hash |
| **Transaction Validity Proofs** | [#12075](https://github.com/ethereum/EIPs/pull/12075) | Draft EIP-8361 would attach a mempool-layer STARK proof of validation-prefix validity and potentially lift direct simulation limits without changing consensus |
| **VOPS Profiles for FOCIL** | [#12110](https://github.com/ethereum/EIPs/pull/12110) | Draft EIP-8369 defines end-of-payload and claimed-index validation profiles; transport for claimed indices remains an activation prerequisite |
| **Recent-root runtime** | [#12131](https://github.com/ethereum/EIPs/pull/12131) | Specifies EIP-8272 runtime bytecode, edge semantics, and deployment. It does not yet resolve the new `0xb5` collision |
| **Precompile frame dispatch** | [#12157](https://github.com/ethereum/EIPs/pull/12157) | Clarifies how active precompiles targeted by frames interact with default-code routing, value charges, and VERIFY invalidity |
| **Validation dependency indexing** | [#12160](https://github.com/ethereum/EIPs/pull/12160) | Enumerates validation-prefix dependencies so clients can revalidate affected transactions without blanket re-simulation |
| **Validation-stable concurrency** | [#12162](https://github.com/ethereum/EIPs/pull/12162) | Would relax the one-pending-transaction rule for accounts whose validation depends only on stable protocol state |
| **Expiry as a frame mode** | [#12198](https://github.com/ethereum/EIPs/pull/12198) | Replaces the special-target VERIFY shape with a fourth frame mode and updates EIP-8266/EIP-8272 references |
| **Expiry-verifier nonce** | [#12203](https://github.com/ethereum/EIPs/pull/12203) | Clarifies that activation installs runtime code without changing the existing account nonce |
| **Frame returndata opcodes** | Under discussion (post #137) | Proposed `FRAMERETURNDATASIZE`/`FRAMERETURNDATACOPY` to enable multi-step flows, no PR yet |
