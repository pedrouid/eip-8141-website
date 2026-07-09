# What EIP-8141 Can Do Today and How It Works

---

## About this page

This is the primary reference for what EIP-8141 does today and how a frame transaction actually flows through the protocol. It's for readers who want the mechanism in detail: what a transaction looks like on the wire, what the EVM does with it, and what the mempool will and won't propagate.

Background assumed: comfortable reading an EIP, know what ECDSA and EOAs are. No prior exposure to ERC-4337, EIP-7702, or FOCIL needed; those are introduced where they come up.

## Transaction Lifecycle

A frame transaction arrives at a node and goes through four conceptual stages:

1. **Admission**. The node checks whether the transaction matches a shape the public mempool will propagate. The mempool only admits transactions whose *validation prefix* (the opening frames up to the point a payer is approved) fits one of four recognized patterns, stays under a gas cap, and reads only predictable state. If it doesn't fit, the transaction is still consensus-valid on-chain but has to reach a block builder through a private channel.
2. **Validation**. The protocol runs the opening frames. These are the frames whose job is to prove "this transaction is authorized and someone is paying for it." Approval-bearing validation frames call the `APPROVE` opcode before returning; expiry-verifier frames are the special timestamp-check exception. This is where signature checks, policy checks, and paymaster logic live.
3. **Execution**. Once a sender has been approved and a payer has been approved, the protocol runs the remaining frames as normal EVM calls. These are the frames that actually do what the user wanted: a token transfer, a swap, a contract deployment. Multiple execution frames can be grouped into an atomic batch so they all-or-none revert together.
4. **Settlement**. After the last frame runs, the protocol computes gas used, refunds the payer, and writes per-frame receipt entries. The transaction receipt records which address paid, which frames succeeded, and the logs each frame produced.

The rest of this page is the mechanical detail behind these four stages.

## Core Concept

EIP-8141 introduces **Frame Transactions**, a new transaction type (`0x06`) where a transaction consists of multiple **frames**, each with a purpose (verify identity, execute calls, deploy contracts). The protocol uses frame modes to reason about what each frame does, enabling safe mempool relay and flexible user-defined validation.

A key insight for understanding the current spec: **EIP-8141 is two specs in one**.

- The **execution model** says "validation and payment are programmable": any account code can verify any signature scheme and approve any payer, with arbitrary logic.
- The **mempool model** says "that programmability is only publicly relayable when it fits a small set of validation-prefix shapes and state-dependency rules."

The execution model defines what is *possible*; the mempool model defines what is *propagatable*. A frame transaction that doesn't match the mempool rules is still valid on-chain; it just can't be gossiped through the public p2p network and must reach a block builder through private channels.

## Transaction Structure

```
[chain_id, nonce, sender, frames, signatures, max_priority_fee_per_gas, max_fee_per_gas, max_fee_per_blob_gas, blob_versioned_hashes]

frames = [[mode, flags, target, gas_limit, value, data], ...]
signatures = [[scheme, signer, msg, signature], ...]
```

- `sender`: the 20-byte address of the account originating the transaction
- Each frame has a `mode` (lower 8 bits only), a `flags` field (approval scope + atomic batch), a `target` address (or null for sender), a gas limit, a `value` field (ETH to transfer in the top-level frame call, non-zero only in `SENDER` frames), and arbitrary data
- `signatures` is a list of signature objects on the outer transaction, verified before any frame executes for protocol-validated schemes. During execution, signer identity and signature validity are already established for `SECP256K1` and `P256`; custom schemes carry witness bytes as `ARBITRARY` signatures and validate them in account code.
- Up to 64 frames per transaction

Signature schemes:

| Scheme | Name | Protocol validation |
|---|---|---|
| `0x0` | `ARBITRARY` | No cryptographic validation by the protocol; `signer` must be empty. Raw bytes are introspectable through `SIGPARAM`. |
| `0x1` | `SECP256K1` | `v || r || s`, low-`s`, signer is a 20-byte address or empty for `tx.sender`. |
| `0x2` | `P256` | `r || s || qx || qy`, signer address is `keccak256(qx || qy)[12:]` or empty for `tx.sender`. |

`SIGPARAM (0xb4)` exposes signature metadata to EVM code. It intentionally does not expose raw signature bytes for protocol-validated schemes, preserving future aggregation. For `ARBITRARY` entries, `SIGPARAM` can copy the raw witness bytes into memory so custom verifiers can validate schemes the protocol does not understand.

## Frame Modes

| Mode | Name | Behavior |
|---|---|---|
| 0 | `DEFAULT` | Called from `ENTRY_POINT`. Used for deployment or post-op tasks. |
| 1 | `VERIFY` | Read-only (STATICCALL semantics). Approval-bearing validation shapes must call `APPROVE`; expiry-verifier frames are the timestamp-check exception. Used for authentication & payment authorization. |
| 2 | `SENDER` | Called from `tx.sender`. Requires prior `sender_approved`. Executes user's intended operations. |

**Flags (separate field, introduced by PR #11521):**
- Bits 0-1: Approval scope constraint (limits which `APPROVE` scope can be used)
- Bit 2: Atomic batch flag (groups consecutive frames of any mode into an all-or-nothing batch; the restrictive mempool tier separately forbids the flag inside the validation prefix). PR #11652 (merged May 12) lifted the previous SENDER-only restriction.

## The APPROVE Mechanism

`APPROVE (0xaa)` is the central innovation. It's an opcode that:
1. Terminates the current frame successfully (like `RETURN`)
2. Updates transaction-scoped approval variables

**Stack**: `[offset, length, scope]` (with double-approval prevention: once a scope bit is set, it cannot be set again)

**Scopes**:
- `0x1`: Approve payment — increments nonce, collects gas fees from the account, sets `payer_approved = true`
- `0x2`: Approve execution — sets `sender_approved = true` (only valid when `frame.target == tx.sender`)
- `0x3`: Approve both execution and payment

**Security**: Only the resolved frame target can call `APPROVE` (`ADDRESS == resolved_target` check). `APPROVE` exceptional-halts outside frame-transaction execution. The requested scope must be non-zero and must fit the frame's allowed scope bits. `sender_approved` must be true before payment-only approval can set the payer.

## Execution Flow

For each frame transaction:

1. Check `tx.nonce == state[tx.sender].nonce`
2. Initialize `sender_approved = false`, `payer_approved = false`
3. For each frame, compute `resolved_target`: if `frame.target` is null, use `tx.sender`; otherwise use `frame.target`
4. Execute each frame sequentially with `CALLVALUE = frame.value` at the top-level call (which is `0` except in `SENDER` frames):
   - **DEFAULT mode**: caller = `ENTRY_POINT`, execute as regular call to `resolved_target`
   - **VERIFY mode**: caller = `ENTRY_POINT`, execute as STATICCALL to `resolved_target`; approval-bearing validation frames must call `APPROVE`
   - **SENDER mode**: requires `sender_approved == true`, caller = `tx.sender`, target = `resolved_target`. If caller lacks balance for `frame.value`, the frame reverts (ordinary CALL value-transfer semantics).
5. After all frames: verify `payer_approved == true`, refund unused gas to payer

## EOA Default Code

When `frame.target` has no code, the protocol applies built-in "default code" behavior:

**VERIFY mode:**
1. Resolve the frame target (`null` means `tx.sender`). Execution approval is only valid for the sender.
2. Read the approval scope from the flags: `allowed_scope = frame.flags & 3`. If `allowed_scope == 0`, revert.
3. Require `tx.signatures[0]` to be a `SECP256K1` signature with empty `msg` whose resolved signer equals the resolved frame target.
4. Rely on the protocol-validated secp256k1 signature over `compute_sig_hash(tx)`.
5. Call `APPROVE(allowed_scope)`.

PR #11481 (merged May 22) moved per-tx signatures out of `frame.data` and into a dedicated outer `signatures` list. PR #11814 (merged Jul 7) tightened default code to use the secp256k1 signature at index `0` instead of scanning for a matching signer. PR #11621 (merged May 11) removed the P256 branch from the protocol-shipped default code. P256 remains a protocol-validated outer signature scheme, but codeless EOAs still get only secp256k1 default-code authentication; accounts that need passkeys or other schemes use deployed code plus `SIGPARAM`/custom verification.

**SENDER and DEFAULT modes:** PR #11621 (merged May 11) changed default code so that `SENDER` and `DEFAULT` frames no longer revert. The previous behavior (revert unconditionally on both modes, plus an RLP-call-batch decoder removed by PR #11577 on Apr 29) blocked simple native ETH transfers to a fresh EOA via a frame transaction. After the merge, a `SENDER` or `DEFAULT` frame whose `resolved_target` has no code completes the value transfer and returns with empty data.

This means **any EOA can use frame transactions today**, no smart contract deployment needed.

## Atomic Batching

Consecutive frames with bit 2 of `flags` set form an atomic batch. Any frame mode can participate (PR #11652 merged May 12 lifted the previous SENDER-only restriction). The batch is the maximal contiguous run of flagged frames terminated by a frame without the flag:

```
Frame 0: (atomic flag set)   ─┐
Frame 1: (flag not set)      ─┘ Batch 1
Frame 2: (atomic flag set)   ─┐
Frame 3: (atomic flag set)   ─│ Batch 2
Frame 4: (flag not set)      ─┘
```

If any frame in a batch reverts, the state is restored to before the batch started, and remaining frames in the batch are skipped. Skipped frames record receipt status `0x3` (introduced by PR #11621, merged May 11). The restrictive mempool tier separately forbids the atomic-batch flag inside the validation prefix, so this protocol-level expansion does not loosen what the public mempool admits. This enables safe patterns like "approve + swap" where both must succeed.

## Gas Accounting

```
tx_gas_limit = (
    15000
    + 475 * len(tx.frames)
    + sum(calldata_cost(frame.data) for frame in tx.frames)
    + sum(calldata_cost(sig.signer) + calldata_cost(sig.msg) + calldata_cost(sig.signature) for sig in tx.signatures)
    + sum(signature_verification_cost(sig) for sig in tx.signatures)
    + sum(frame.gas_limit for frame in tx.frames)
)
```

Each frame incurs a per-frame cost of `FRAME_TX_PER_FRAME_COST` (475 gas) on top of the intrinsic cost. Gas is charged for the actual frame data and signature data bytes, not for an RLP-encoded aggregate payload. Protocol-validated signatures also add their intrinsic verification costs. Each frame has its own gas allocation. Unused gas is **not** carried to subsequent frames. After all frames execute, the total unused gas is refunded to the payer.

The total fee is:
```
tx_fee = tx_gas_limit * effective_gas_price + blob_fees
```

## Canonical Signature Hash

```python
def compute_sig_hash(tx):
    for sig in tx.signatures:
        if len(sig.msg) == 0:
            sig.signature = Bytes()  # elide raw signature bytes
    return keccak(bytes([FRAME_TX_TYPE]) + rlp(tx))
```

The transaction-type prefix aligns EIP-8141 with the EIP-2718 typed-transaction convention and prevents cross-type signature replay (PR #11544, merged Apr 22). Frame metadata and frame data, including a SENDER frame's `value` and an expiry-verifier deadline, remain covered by this hash.

Raw signature bytes are elided for signatures whose `msg` is empty because:
1. A signature over `compute_sig_hash(tx)` cannot commit to its own raw bytes.
2. Custom verifiers need to insert witness bytes after computing the canonical hash.
3. Protocol-validated signatures can remain aggregation-compatible, because contracts see metadata but not raw bytes.

This is the Jul 6-7 repair from PR #11837 and PR #11814. The older model elided `VERIFY` frame data; the current model keeps frame data in the hash and moves self-referential witness bytes to the `signatures` list.

## Expiry Verifier Frame

A `VERIFY` frame whose `frame.target` equals `EXPIRY_VERIFIER` (`address(0x8141)`) is an **expiry verifier frame**. It calls the canonical runtime code installed at that address; the calldata is interpreted as an 8-byte big-endian unix-seconds deadline, and the call reverts unless `block.timestamp <= expiry_timestamp`.

Constraints:
- `frame.flags == 0`
- `frame.value == 0`
- `len(frame.data) == EXPIRY_DATA_LENGTH` (`8`)
- At most one expiry-verifier frame per transaction
- If present, it may only be the first frame in the frame list
- The frame succeeds without calling `APPROVE`; only `self_verify`, `only_verify`, and `pay` frames are required to call `APPROVE` for transaction validity. PR #11662 relaxed the previously-uniform "every VERIFY frame must call APPROVE" rule to "if the frame reverts, the transaction is invalid".

Mempool admission: nodes MUST drop a transaction from the public mempool whose expiry is less than the node's current view of `block.timestamp` at any point. Expiry-verifier frames are exempt from validation trace rules, storage-dependency tracking, and `MAX_VERIFY_GAS`. The `TIMESTAMP` opcode is permitted only in an expiry-verifier frame executing the canonical runtime code. Clients may omit explicit EVM execution and perform the deadline check natively when the externally observable result is identical.

Validation-prefix shape matching treats expiry-verifier frames as transparent: `[expiry_verify, self_verify]` is recognized as `[self_verify]` for admission-rule purposes.

## Mempool Policy

The policy below is the **restrictive tier** of a [two-tier mempool architecture](/mempool-strategy). It is what clients ship first and what FOCIL nodes default to. An expansive tier (ERC-7562 / paymaster-extended) is intended to develop in parallel for use cases that exceed the restrictive policy (privacy protocols, multi-account paymasters).

The public mempool recognizes four validation prefixes:

**Self Relay:**
```
[self_verify]                    # basic
[deploy] → [self_verify]        # with account deployment
```

**Canonical Paymaster:**
```
[only_verify] → [pay]                    # basic
[deploy] → [only_verify] → [pay]        # with account deployment
```

Rules enforced during validation prefix:
- Must match one of the four prefixes above
- Sum of validation prefix gas plus intrinsic signature-validation cost <= 100,000
- `self_verify`, `only_verify`, and `pay` frame flags must match the scope of the `APPROVE` call; no frame in the validation prefix may set the atomic-batch flag
- No `VERIFY` frame may appear after the validation prefix
- Banned opcodes (ORIGIN, TIMESTAMP, BLOCKHASH, etc.). `CREATE`, `CREATE2`, and `SETDELEGATE` are banned outside the first `deploy` frame and allowed inside it (PR #11567, merged Apr 30)
- No state writes, except inside the first `deploy` frame for `CREATE`/`CREATE2`/`SETDELEGATE` operations that install code at `tx.sender`, or `SSTORE`s to `tx.sender`'s storage
- No storage reads outside `tx.sender`
- No calls to non-existent contracts or EIP-7702 delegations (except deploy-frame default-code behavior at `tx.sender`)
- Deploy-frame target may be any contract, provided its execution satisfies the trace rules above. EIP-7997 is the canonical-but-non-mandatory factory; cross-chain-stable factory addresses are typically acquired through it (PR #11567)

**Canonical paymaster**: verified by runtime code match, uses reserved balance accounting. The canonical paymaster handles ETH-funded sponsorship only (the paymaster covers gas from its own ETH balance, with no token-level repayment flow). ERC-20 gas repayment is a separate non-canonical paymaster design space: a public-mempool sponsor can inspect signature metadata and the next SENDER frame while accepting sponsee frontrunning risk; a fully trustless onchain balance-checking variant exceeds the restrictive tier and routes privately or through an expansive tier. See [Mempool Strategy → ERC-20 gas repayment](/mempool-strategy#erc20-paymaster-patterns).
```python
available = balance(paymaster) - reserved_pending_cost - pending_withdrawal_amount
```

**Non-canonical paymaster**: limited to 1 pending tx per paymaster in the mempool.

**Transaction origination**: EIP-3607's restriction (which forbids transactions whose `tx.sender` has non-empty, non-delegation code) does not apply to frame transactions. `SENDER` frames intentionally originate calls where `tx.sender` is a contract account, so the carve-out is required for native AA. Validation logic for non-frame transaction types is unchanged: a regular transaction is only valid if the sender account's code is empty or a valid delegation indicator (PR #11272, merged May 5).

## Practical Use Cases

### 1. Simple EOA Transaction (gas self-paid)

| Frame | Mode | Target | Data |
|---|---|---|---|
| 0 | VERIFY | sender | ECDSA signature |
| 1 | SENDER | target | call data |

Frame 0 verifies the signature and calls `APPROVE(0x3)`. Frame 1 executes.

### 2. Gas Sponsorship (ETH-funded, canonical paymaster)

| Frame | Mode | Target | Data |
|---|---|---|---|
| 0 | VERIFY | sender | Signature (approve execution) |
| 1 | VERIFY | canonical paymaster | Sponsor data (approve payment) |
| 2 | SENDER | target | User's call |
| 3 | DEFAULT | canonical paymaster | Post-op (settle gas refund) |

The canonical paymaster pays gas in ETH; the user pays nothing.

### 3. Atomic Approve + Swap

| Frame | Mode | Atomic | Target | Data |
|---|---|---|---|---|
| 0 | VERIFY | - | sender | Signature |
| 1 | SENDER | set | ERC-20 | approve(DEX, amount) |
| 2 | SENDER | not set | DEX | swap(...) |

If the swap reverts, the ERC-20 approval is also reverted.

### 4. Account Deployment + First Transaction

| Frame | Mode | Target | Data |
|---|---|---|---|
| 0 | DEFAULT | deployer | initcode + salt |
| 1 | VERIFY | sender | Signature |
| 2 | SENDER | target | User's call |

Frame 0 deploys the account; frames 1-2 validate and execute.

### 5. EOA Paying Gas in ERC-20s

| Frame | Mode | Target | Data |
|---|---|---|---|
| 0 | VERIFY | sender | Empty; uses sender signature entry at index 0 |
| 1 | VERIFY | sponsor | Sponsor data; checks signature metadata and next ERC-20 transfer |
| 2 | SENDER | ERC-20 | transfer(sponsor, fees) |
| 3 | SENDER | target | User's call |
| 4 | DEFAULT | sponsor | Optional post-op refund/conversion |

The sponsor pays ETH gas; frame 2 repays the sponsor in ERC-20 tokens.

> **Mempool note**: in the current public-mempool shape, the sponsor does not check the user's ERC-20 balance onchain during validation. It checks the frame structure and any sponsor authorization data, then accepts the risk that the sponsee may zero the ERC-20 balance before inclusion. A fully trustless onchain balance-checking paymaster remains consensus-valid, but it exceeds the restrictive tier because it reads external token storage.

## Key Design Properties

- **No authorization list**: Unlike EIP-7702, doesn't rely on ECDSA for delegation. Compatible with PQ crypto.
- **No access list**: Future optimizations covered by block-level access lists (EIP-7928).
- **Per-frame `value` field**: SENDER frames may transfer ETH natively via `frame.value` (added by PR #11534, Apr 16). `VERIFY` and `DEFAULT` frames must set `value = 0`, preserving `STATICCALL`-like behavior in `VERIFY` and keeping `ENTRY_POINT` free from funding transfers.
- **ORIGIN returns frame caller**: Changed from traditional tx.origin behavior (precedent set by EIP-7702).
- **Transient storage cleared between frames**: TSTORE/TLOAD state doesn't persist across frames.
- **Warm/cold state shared across frames**: Gas accounting for storage access is shared.
- **Requires**: EIP-1559, EIP-2718, EIP-3607, EIP-4844, EIP-7623, EIP-7702 (PR #11567 merged Apr 30 dropped EIP-7997; PR #11272 merged May 5 added EIP-3607 with an explicit carve-out; PR #11621 merged May 11 added EIP-7623 (calldata gas pricing) and EIP-7702 (delegation indicators), formalizing dependencies that were previously implicit in spec text).

## Related Proposals

| Proposal | Relationship |
|---|---|
| ERC-4337 | 8141 is the native protocol successor, removing the bundler intermediary |
| EIP-7702 | Complementary; 7702 accounts can also use frame transactions. Note: 7702-delegated accounts cannot currently use default code signature verification (gap identified by DanielVF) |
| ERC-7562 | 8141's mempool rules are inspired by but simpler than 7562 (no staking/reputation) |
| EIP-8175 | Competing alternative: flat capabilities + programmable fee_auth, 4 new opcodes |
| EIP-8130 | Coinbase/Base's alternative: declared authenticators (formerly verifiers), no wallet code execution during validation, active through Jul 8. See [Competing Standards](./competing-standards) |
| EIP-7997 | Canonical deterministic factory predeploy; recommended for cross-chain-stable factory addresses but no longer a hard dependency after PR #11567 (merged Apr 30) |
| EIP-7392 | Signature registry; interoperability PR #11455 was closed without merge on Apr 23 |
| EIP-8250 | Keyed-nonces sibling EIP (PR #11598 merged May 11; PR #11749 merged Jun 1 generalized single keyed nonce to a bounded key set). First EIP whose `requires` header includes EIP-8141; layers `(nonce_keys, nonce_seq)` replay protection on top of EIP-8141 via a `NONCE_MANAGER` system contract |
| EIP-8266 | Expiring-nonces sibling EIP (PR #11692 merged May 22, co-authored by nerolation and lightclient); second EIP in the compose-by-requires AA stack, with `requires` including both EIP-8141 and EIP-8250. Sentinel-mode (`tx.nonce == 2**64 - 1`) plus a `NONCE_RING` system contract; complements PR #11662's per-tx `EXPIRY_VERIFIER` frame by scoping nonces themselves to time windows |
| EIP-8272 | Recent-roots sibling EIP (PR #11726 merged Jun 5 by soispoke, vbuterin, nerolation); third compose-by-requires sibling, requires both EIP-7843 and EIP-8141. Adds a `recent_root_references` outer-envelope field, a `RECENT_ROOT_ADDRESS` system contract with 8192-slot ring, and a new `RECENTROOTREFLOAD` opcode. Note the opcode-space interaction: the latest EIP-8141 now assigns `0xb4` to `SIGPARAM`, so sibling opcode numbering needs coordination |
| EIP-8288 (pending) | PQ signature and STARK aggregation sibling EIP (PR #11772 opened Jun 5 by vbuterin and Thomas Coratger); fourth compose-by-requires sibling. Still open as of Jul 9 with requested changes and proof-security review. Adds a new `DEP_VERIFY_FRAME_MODE = 3` frame mode declaring `(scheme, data_hash, verification_key)` dependencies, and a block-header `recursive_stark` field that aggregates all per-block dependencies into one recursive STARK using Lean Ethereum tooling (`LEANSPHINCS_SCHEME`, `LEANSTARK_SCHEME`). FOCIL-compatible |
| ERC-8286 (draft) | First application-layer standard built on EIP-8141, and the first frame-transaction proposal in the ERCs repo (ERC PR #1794 opened Jun 3 by chiranjeev13). Defines how ERC-7579 modular accounts (validator, executor, hook, config modules) implement the frame-transaction validation flow: a validator module returns an approval mode the account applies via `APPROVE` during a VERIFY frame. Standardizes the permissions and session-key layer that protocol defaults leave open |

## Key Takeaway

A frame transaction is a sequence of purpose-labeled sub-calls. The protocol runs validation frames first, refuses to proceed unless approval-bearing validation frames call `APPROVE`, then runs execution frames. Default code makes EOAs first-class users without deployment. The mempool only propagates transactions whose validation prefix fits a constrained shape, which is what lets a stateless node safely accept them.

## Read Next

- [EOA Support](/eoa-support) — what existing codeless accounts get for free, and how default code replaces EIP-7702 delegation for common cases.
- [Feedback Evolution](/feedback-evolution) — how the spec got to its current shape through twenty-two phases of community review.
- [Mempool Strategy](/mempool-strategy) — why the validation prefix is the way it is, and how the two-tier mempool handles everything that doesn't fit.
- [Competing Standards](/competing-standards) — how EIP-8141 compares to EIP-8130, EIP-8175, EIP-8202, and the sibling proposals.
