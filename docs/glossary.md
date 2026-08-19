# Glossary

A reference for the jargon that appears across this site. Entries are grouped by category and alphabetical within each group. Links point to the page where the concept is explained in depth.

---

## Contents

- [EIP-8141 core concepts](#eip-8141-core-concepts)
- [Frame modes and opcodes](#frame-modes-and-opcodes)
- [Mempool and propagation](#mempool-and-propagation)
- [Statelessness](#statelessness)
- [Cryptography and signatures](#cryptography-and-signatures)
- [Account abstraction ecosystem](#account-abstraction-ecosystem)
- [Related EIPs](#related-eips)
- [Alternative AA proposals](#alternative-aa-proposals)
- [Governance and timeline](#governance-and-timeline)

---

## EIP-8141 core concepts

**Absorb-into-base** — The packaging pattern proposed by PR #11681 to fold guarantors, keyed nonces, and signer binding directly into EIP-8141. The PR closed as stale on Aug 14; compose-by-requires remains the shipped sibling pattern.

**APPROVE** — The central new opcode (`0xaa`). Terminates the current frame successfully and updates transaction-scoped approval flags (`sender_approved` and/or payer). Takes scope `0x1` payment, `0x2` execution, or `0x3` both. It has no execution-gas base charge; if nonce increment creates the sender, the frame pays 183,600 state gas. See [Current Spec → APPROVE mechanism](/current-spec#the-approve-mechanism).

**Approval scope** — The subset of transaction-level approval a VERIFY frame grants. Encoded as bits 0-1 of `frame.flags`. Scopes are `0x1` (payment), `0x2` (execution), `0x3` (both). Double-approval prevention: once a scope bit is set, it cannot be set again.

**Atomic batch** — A run of consecutive DEFAULT or SENDER frames with bit 2 of `frame.flags` set. VERIFY cannot participate, and every batch member, including its unflagged terminator, must have zero approval scope. If any frame fails, batch state, logs, and state-gas receipt mutations revert and remaining frames are skipped.

**AUTH_MANAGER** — The combined nonce/signer system contract proposed by the now-closed PR #11681. It is not part of current EIP-8141.

**Blob sidecar** — The EIP-7594 pooled-transaction wrapper carrying blob transactions plus cells and proofs. Frame transactions support the same KZG versioned hashes, `BLOBHASH` access, networking path, and blob-fee settlement after PR #11985.

**calldata_floor_gas** — The EIP-7623 minimum execution gas for frame and signature payload bytes. Final `tx_execution_gas` cannot fall below it; net state gas is added on top.

**Canonical paymaster** — A standardized paymaster whose runtime code is recognized by mempool policy. It bypasses the one-pending non-canonical-paymaster cap and uses explicit balance reservations. Open PR #12041 replaces closed draft #12012 with assembled runtime, storage, signer, withdrawal, and code-hash rules.

**Canonical signature hash (sighash)** — `keccak(bytes([FRAME_TX_TYPE]) + rlp(tx_copy))` where `tx_copy` has raw `signature` bytes replaced with empty bytes for signature entries whose `msg` is empty. Raw signature bytes are elided so a signature over the canonical hash does not cover itself. The type-byte prefix follows the EIP-2718 convention for cross-type replay protection (PR #11544).

**Compose-by-requires** — The architectural pattern where EIP-8141 owns the protocol primitive and sibling EIPs (EIP-8250, EIP-8266, EIP-8272, EIP-8288) layer specific policies on top via the `requires` header, instead of bundling them into the base spec. Contrast absorb-into-base.

**Default code** — Protocol-level logic that runs when a frame targets an account with no deployed code and no EIP-7702 delegation. VERIFY uses secp256k1 at signature index `0` when approval includes execution and index `1` for payment-only EOA sponsorship (PR #11954); SENDER and DEFAULT return successfully after ordinary top-level call behavior. Makes EOAs first-class frame-transaction users. See [EOA Support](/eoa-support).

**Direct evaluation** — A mempool optimization for protocol-defined default, expiry, and canonical-paymaster prefixes. Clients may compute the result without EVM simulation only if dependencies and both execution/state cap results remain identical.

**ENTRY_POINT** — A protocol-defined distinguished caller address (`0xaa`) used as `CALLER` in DEFAULT and VERIFY frames. Not a deployed contract or precompile; contracts must not assume anything about its code, balance, or caller type beyond address equality. `CALLVALUE = 0` when the caller is `ENTRY_POINT`.

**EXPIRY_VERIFIER** — The canonical frame contract at `address(0x8141)` (PR #11662, merged May 14). A VERIFY frame targeting it treats `frame.data` as an 8-byte unix-seconds deadline and reverts if it has passed; the mempool drops expired transactions. At most one per transaction.

**Execution gas** — The EIP-8141 gas dimension for computation, data, calls, and account access. Each frame declares `limits.execution`; all execution budgets plus intrinsic gas are subject to EIP-7825.

**fees** — The grouped outer-envelope list `[max_priority_fee_per_gas, max_fee_per_gas, max_fee_per_blob_gas]` introduced by PR #12062.

**Frame** — A sub-call with tuple `[mode, flags, target, [execution_limit, state_limit], value, data]`. A transaction can hold up to `MAX_FRAMES = 64` frames.

**Frame transaction** — The EIP-8141 transaction type (`0x06`). An ordered list of frames plus chain, nonce, sender, fee, and blob fields. Splits a transaction into distinct validation, execution, and deployment phases, each of which the protocol can reason about structurally.

**FRAME_TX_INTRINSIC_COST** — `12,000` execution gas per frame transaction before per-frame, signature, calldata, and value-transfer charges. PR #12062 lowered it from `15,000` while adopting EIP-2780 pricing.

**FRAME_TX_PER_FRAME_COST** — `475` gas per frame added to intrinsic cost. Covers fixed CALL execution-context overhead (100) plus `G_log` (375) for the per-frame receipt sub-entry.

**FRAMEDATALOAD / FRAMEDATACOPY** — Opcodes `0xb1` and `0xb2` that read a frame's `data` field. `FRAMEDATALOAD` pops `offset` then `frameIndex`; `FRAMEDATACOPY` pops `memOffset`, `dataOffset`, `length`, then `frameIndex` (PR #11938). They replaced the earlier TXPARAMSIZE/TXPARAMCOPY opcodes.

**FRAMEDATASIZE** — An earlier/draft frame-introspection name for reading frame data length. The current introspection surface uses `FRAMEDATALOAD`, `FRAMEDATACOPY`, `FRAMEPARAM`, `SIGPARAM`, and `SIGDATACOPY`; sibling drafts using the old name need reconciliation.

**FRAMEPARAM** — Opcode `0xb3`. Reads frame metadata including target, execution/state limits, mode, flags, value, status, and completed-frame execution/state usage.

**gas_used** — Final `tx_execution_gas + tx_state_gas`. The execution component applies the EIP-3529 refund and EIP-7623 floor; state usage is net of journaled refills and rollbacks.

**Guarantor (APPROVE_GUARANTEE)** — A proposed payer that would pay even if sender validation fails, letting nodes skip sender simulation. PRs #11555 and #11681 closed as stale on Aug 14; no guarantor scope exists in current EIP-8141.

**Keyed nonces (2D nonce / nonce key set)** — Replay-protection sequences indexed by `(sender, key)` so an account can keep multiple independent in-flight nonce streams. EIP-8250's primitive, generalized to a bounded key set by PR #11749.

**limits** — The per-frame envelope field `[execution, state]`. Each component reserves an independent gas dimension and cannot borrow from another frame or dimension.

**max_cost** — The payer escrow collected when payment is approved: `max_gas * max_fee_per_gas + blob_gas * blob_base_fee`. The final difference is returned as `payer_refund`.

**MAX_FRAMES** — The per-transaction frame count limit. Currently `64`, reduced from `10^3` by PR #11521. Raisable later after empirical measurement; harder to lower once ecosystems depend on it.

**max_gas** — The maximum of `standard_gas_limit` and `calldata_floor_gas + total_declared_state_gas`. Used to compute payer escrow; blocks reserve execution and state capacity separately.

**MAX_VERIFY_GAS** — The 100,000 execution-gas cap on validation-prefix budgets plus protocol signature validation.

**MAX_VERIFY_STATE_GAS** — The 500,000 state-gas cap declared across the validation prefix. It bounds public-mempool state growth, not node simulation work.

**NONCE_MANAGER** — The system contract introduced by EIP-8250, holding `(nonce_keys, nonce_seq)` keyed-nonce streams per sender.

**NONCE_RING** — The system contract introduced by EIP-8266; a fixed-capacity ring buffer backing sentinel-mode (`tx.nonce == 2**64 - 1`) time-windowed replay protection.

**payer** — Transaction-scoped address initialized to `None`. A successful payment-scoped `APPROVE` sets it to the resolved frame target, consumes the sender nonce, and collects the transaction's maximum cost. It must be non-null after all frames. Older revisions described the same condition as `payer_approved`.

**Payer exposure** — The sum of pending maximum costs reserved against one payer. PR #12007 requires it not to exceed that payer's balance and moves reservations atomically on replacement.

**payer_refund** — The difference between the payer's collected `max_cost` and the final charged execution plus blob fee. It is returned after execution.

**recent_root_references** — EIP-8272's outer-envelope field: a bounded list of `(source_id, slot, root)` tuples letting validation check application-state roots without reading mutable storage.

**RECENT_ROOT_ADDRESS** — The system contract introduced by EIP-8272; an 8192-slot ring storing recent application-state roots, read during validation via the RECENTROOTREFLOAD opcode.

**RECENTROOTREFLOAD** — EIP-8272 opcode currently assigned `0xb5`. PR #11967 moved it from `0xb4`, but EIP-8141's later `SIGDATACOPY` merge now also claims `0xb5`; the collision is unresolved.

**recursive_stark** — EIP-8288's proposed block-header field `[stark_proof, block_deps_hash]` aggregating all per-block DEP_VERIFY dependencies into one recursive STARK proof.

**refund_counter** — One transaction-level EIP-3529 storage-refund counter shared across frames. Reverted frames and unrolled batches roll back their contributions; the final refund is capped at one-fifth of pre-refund gas.

**resolved_target** — `frame.target if frame.target is not null else tx.sender`. Explicit name for the target-resolution rule used consistently throughout execution (introduced by PR #11521).

**Self-relay** — The simplest recognized validation-prefix shape: a frame transaction whose sender pays its own gas (with a deploy-prefixed variant). One of the four restrictive-tier prefixes alongside the canonical-paymaster shapes.

**sender_approved** — Transaction-scoped boolean flag flipped when a VERIFY frame with execution scope calls `APPROVE`. Must be `true` before any SENDER frame can execute. Paired with the transaction-scoped `payer` address; execution approval can only be set once per transaction.

**Signatures list** — The outer-transaction `signatures` field (PR #11481, repaired by #11837/#11814): a list of `(scheme, signer, msg, signature)` objects. Protocol-validated schemes are `SECP256K1 (0x1)` and canonical low-`s` `P256 (0x2)`; `ARBITRARY (0x0)` carries custom witness bytes and costs 100 gas per entry. Default code reads index `0` for execution scope and index `1` for payment-only EOA sponsorship after PR #11954.

**Signer binding** — A transaction-scoped `verified_signers` table (PR #11681) populated by non-secp256k1 VERIFY frames that prove `(digest, address)` against a registered pubkey, so `ECRECOVER` can return non-secp256k1-authenticated addresses on the hit path.

**SIGDATACOPY** — EIP-8141 opcode currently assigned `0xb5`. Copies raw bytes from an `ARBITRARY` signature entry with `CALLDATACOPY`-style stack order. Its assignment conflicts with EIP-8272 `RECENTROOTREFLOAD`.

**SIGPARAM** — Opcode `0xb4`. Reads resolved signer, scheme, message, or `ARBITRARY` witness length with a static two-item stack input. Raw bytes are copied by `SIGDATACOPY`; protocol-validated bytes remain opaque.

**State gas** — The EIP-8037 dimension for durable state growth. Each frame declares isolated `limits.state`; charges and refills are journaled to the frame that owns the state creation.

**standard_gas_limit** — Intrinsic execution gas plus all declared frame execution and state budgets. One input to payer `max_gas`; block reservations split it back into exact dimensions.

**TOTAL_COST_FLOOR_PER_TOKEN** — The EIP-7623 calldata-floor multiplier applied to the combined frame and signature payload token count when computing `calldata_floor_gas`.

**TX_MAX_GAS_LIMIT** — The EIP-7825 per-transaction execution cap, currently `16,777,216`. EIP-8141 applies it to intrinsic gas plus declared execution budgets and separately to the calldata floor.

**TXPARAM** — Opcode `0xb0`. Reads transaction scalars including sender, nonce, grouped fees, blob count, sighash, frame/signature counts, current frame index, and current `state_gas_left`.

**Validation prefix** — The opening sequence of frames up to and including the successfully completed frame that sets `payer`. Only these frames are subject to public-mempool policy; post-payment frames are arbitrary. Recognized prefixes: self-relay, canonical paymaster, and deploy-prefixed variants.

**VERSIONED_HASH_VERSION_KZG** — The KZG version byte required for every frame-transaction blob hash. Added with full blob support in PR #11985.

---

## Frame modes and opcodes

**DEFAULT (mode 0)** — A frame called from `ENTRY_POINT` with regular-call semantics. Used positionally: first frame for account deployment through any factory satisfying the deploy-frame trace rules, and last frame for paymaster post-op refunds. For codeless targets, default code returns with empty data; deployed contracts can also implement DEFAULT frames.

**DEP_VERIFY (mode 3)** — `DEP_VERIFY_FRAME_MODE = 3`, a frame mode proposed by EIP-8288 (pending, PR #11772). Declares `(scheme, data_hash, verification_key)` dependency triples for block-level recursive-STARK aggregation instead of executing EVM code.

**SENDER (mode 2)** — A frame called from `tx.sender`. Requires `sender_approved = true` before execution. The frames that do what the user actually asked for: transfers, swaps, contract calls. The only mode that may carry non-zero `frame.value`.

**VERIFY (mode 1)** — A frame called from `ENTRY_POINT` with `STATICCALL` semantics (no state writes, except `APPROVE`'s protocol-defined effect). Approval-bearing validation shapes must call `APPROVE`; expiry-verifier frames are the timestamp-check exception. The home of signature verification, paymaster authorization, and custom validation policy.

---

## Mempool and propagation

**Banned opcodes** — Environmental or state-mutating instructions forbidden inside the validation prefix. The list includes block/fee fields, `SLOTNUM`, `BALANCE`, `SELFBALANCE`, creation outside deploy, and `SSTORE` outside the sender-storage deploy carve-out. `ORIGIN`, `TLOAD`, `TSTORE`, and `BLOBHASH` are allowed after PR #12167 because their values are frame- or transaction-determined.

**Encrypted mempool** — A mempool design (e.g., [LUCID/EIP-8184](https://eips.ethereum.org/EIPS/eip-8184)) that hides transaction contents until inclusion. Incompatible with the restrictive tier's static checks; routed through the expansive tier and onchain rebroadcasters instead.

**Expansive tier** — The opt-in, ERC-7562-style mempool tier that accepts arbitrary VERIFY logic subject to a node's resource budget. Handles privacy protocols, multi-paymaster flows, and anything exceeding restrictive-tier bounds. Not specified by EIP-8141; develops in parallel. See [Mempool Strategy](/mempool-strategy#expansive-mempool-what-develops-in-parallel).

**FOCIL** — *Fork-Choice-enforced Inclusion Lists*, formalized in [EIP-7805](https://eips.ethereum.org/EIPS/eip-7805). Validators publish lists of transactions the next block must include; the fork-choice rule penalizes blocks that omit them. For FOCIL to work, attesters must be able to validate listed transactions, which is why FOCIL and VOPS are tightly coupled.

**Inclusion list** — The ordered list of transactions a FOCIL attester proposes must appear in the next block. Bounded by per-tx and per-list gas budgets (100k per tx, 250k per list today; raised caps proposed in the three-gates analysis).

**Non-canonical paymaster** — Any paymaster whose runtime code does not match the canonical paymaster's code hash. Limited to `MAX_PENDING_TXS_USING_NON_CANONICAL_PAYMASTER = 1` pending transaction per paymaster in the public mempool. Beyond one, these users route privately or via the expansive tier.

**Restrictive tier** — The EIP-8141 public-mempool policy. Admits four prefix shapes under 100,000 execution gas and 500,000 declared state gas, with bounded dependencies and sender-only storage reads. See [Mempool Strategy](/mempool-strategy#restrictive-mempool-what-ships-first).

**Two-tier mempool** — The architecture where restrictive (in-spec, common case) and expansive (opt-in, privacy and complex validation) tiers run in parallel. FOCIL nodes default to restrictive; the expansive tier is a separate opt-in layer. See [Mempool Strategy](/mempool-strategy#two-tiers-in-one-mempool).

---

## Statelessness

**AA-VOPS** — VOPS extended to cover account-abstraction validation state. The practical question is how many per-account storage slots a VOPS node must carry beyond nonce and balance to validate frame transactions from smart accounts. EIP-8141 proposes `N = 4` (see VOPS+4).

**Binary tree migration** — The planned transition from Ethereum's hexary Merkle-Patricia Trie to a binary verkle/patricia tree. Reduces witness size per item from 4-8 kB today to 1-2 kB, making the merkle-branch escape hatch cheaper at scale.

**Merkle branch (witness)** — A cryptographic proof that a specific storage slot holds a specific value in the current state trie. Frame transactions that need to read state outside VOPS+4 can include witnesses for those reads, paying the proof size as explicit per-tx cost.

**PS node (Partially Stateful)** — A node that carries state beyond the VOPS baseline for a specific use case (e.g., a node that tracks a canonical privacy pool's nullifier slots). Not a formal protocol role; infrastructure coordination assumed.

**Validation state** — The data a node reads from its copy of the chain to decide whether a transaction is well-formed before including it in a block. For a legacy tx: three fields on the sender's account (balance, nonce, code). For a frame tx: whatever the VERIFY frame's code touches, bounded by the restrictive-tier rules.

**VOPS** — *Validity-Only Partial Statelessness*. A node design that carries only a small "validity slice" of the full state (nonce + balance, ~10 GB for ~400M accounts) and delegates full execution to other nodes or ZK proofs. See the [original VOPS thread](https://ethresear.ch/t/a-pragmatic-path-towards-validity-only-partial-statelessness-vops/22236).

**VOPS+4** — The proposed extension adding 4 storage slots per account to the VOPS baseline: nonce, balance, code, and the first 4 storage slots. Scales to ~72 GB at full AA adoption. Covers well-designed AA wallets' validation reads. See [Mempool Strategy → VOPS+4](/mempool-strategy#the-state-side-vops-4-slots).

---

## Cryptography and signatures

**BN254** — An elliptic curve used by EIP-8224's fflonk proofs and by the `ecPairing` precompile. Not quantum-safe on its own; used here for efficient pairing-based verification.

**ARBITRARY signature** — Signature-list scheme `0x0`. The protocol performs no cryptographic validation, requires empty `signer`, and charges 100 gas. Account code reads length through `SIGPARAM` and bytes through `SIGDATACOPY`. Empty-`msg` bytes are elided from the sighash, so verifiers must enforce length and canonical encoding.

**Dilithium** — A lattice-based post-quantum signature scheme (NIST FIPS 204). Candidate for a future PQ precompile alongside Falcon.

**ECDSA** — *Elliptic Curve Digital Signature Algorithm*. Ethereum's incumbent signature scheme, deployed over the `secp256k1` curve. Vulnerable to Shor's algorithm on a sufficiently large quantum computer.

**Ephemeral keys** — Single-use key material that is destroyed or rotated per transaction. Explored in Stage 3 of the [PQ roadmap](/pq-roadmap) as a way to reduce long-term exposure of secp256k1 keys before full PQ migration.

**Falcon-512** — A lattice-based post-quantum signature scheme (NIST FIPS 206). Smaller signatures than Dilithium but slower to sign. Proposed for native support in EIP-8175 and EIP-8202; EIP-8141 accommodates it via VERIFY-frame code or a future precompile (EIP-8052).

**fflonk** — A succinct ZK proving system used by EIP-8224 for shielded-gas-funding proofs. Universal trusted setup (reuses powers-of-tau), two-pair pairing verification on BN254, ~176K gas per proof.

**Groth16** — A pairing-based succinct ZK proving system commonly used by privacy pools. A withdrawal proof typically costs ~250K gas, exceeding the 100K `MAX_VERIFY_GAS` cap; this is one of the three gates privacy flows hit in the restrictive mempool.

**Lean Ethereum (LEANSPHINCS / LEANSTARK)** — Hash-based signature and STARK tooling for post-quantum Ethereum. EIP-8288 uses the schemes `LEANSPHINCS_SCHEME = 0x10` (gas 3000) and `LEANSTARK_SCHEME = 0x11` (gas 30000).

**Low-s enforcement** — A rule requiring ECDSA signatures to use the canonical, lower-half `s` value. Prevents signature malleability. PR #11937 pins EIP-8141's outer secp256k1 encoding to recovery id `0/1`, non-zero in-range `r`, and low-`s` for every protocol-validated entry.

**Nullifier** — A unique per-spend identifier used by privacy pools to prevent double-spending a shielded note. Stored in the pool contract's storage; reads are keyed by hash, making slot positions unpredictable and incompatible with fixed-N statelessness windows.

**P256 (secp256r1)** — The NIST curve used by Apple/Google passkeys, WebAuthn, and hardware secure enclaves. Supported as protocol-validated outer signature scheme `0x2`, with canonical low-`s` required even though the precompile accepts high-`s`. It is not accepted by codeless EOA default code and is not post-quantum safe. Requires the [EIP-7951 P256 precompile](https://eips.ethereum.org/EIPS/eip-7951).

**Passkey** — A platform-managed credential using WebAuthn + P256 signatures, typically stored in a device's secure enclave. EIP-8141 can carry P256 signatures in the outer list, but passkey-authenticated account authorization requires deployed account code or a future extension because default code accepts only secp256k1.

**Poseidon commitment** — A hash-based commitment using the Poseidon hash, ZK-friendly and efficient inside proof circuits. EIP-8224 uses Poseidon commitments to represent fee notes privately.

**Recursive STARK aggregation** — Verifying many per-transaction PQ-signature or proof dependencies in a block with one recursive STARK in the block header (EIP-8288), instead of verifying each per transaction. The concrete realization of the outer signatures-list aggregation hook.

**secp256k1** — The Koblitz curve used by Ethereum's EOA signatures. Paired with ECDSA; not quantum-safe. EIP-8141 scheme `0x1` uses a 65-byte `v || r || s` encoding with `v` in `{0, 1}` and canonical low-`s` values.

**Signature aggregation** — Combining many individual signatures into a single succinct validity proof that the protocol checks once. Strategically important for PQ signatures (which are large); the VERIFY-frame architecture deliberately preserves the path forward. The outer `signatures` list ([PR #11481](https://github.com/ethereum/EIPs/pull/11481), merged May 22) adds the schema-level hook: a future block-level aggregated witness can elide individual per-tx signatures while preserving the commitments.

**Signature-scheme registry** — The governed set of fixed-cost protocol signature IDs proposed by PR #12011. IDs `0x03` through `0xff` would pin the verifier, signer derivation, upgrade path, and aggregation behavior while retaining `ARBITRARY` as the custom-code escape hatch.

**SPHINCS+** — A hash-based post-quantum signature scheme (NIST FIPS 205). Larger signatures than Dilithium/Falcon; referenced as one of the PQ candidates in the roadmap.

---

## Account abstraction ecosystem

**Account Abstraction (AA)** — The umbrella term for moving validation and payment logic out of hardcoded protocol rules and into user-defined code. EIP-8141 calls itself *native AA*: the validation logic runs in-protocol via the EVM rather than out-of-protocol via bundlers.

**Bundler** — The off-chain actor in ERC-4337 that collects UserOperations, runs simulation, and packages them into transactions. EIP-8141 eliminates the role by bringing validation in-protocol; frame transactions use the standard mempool.

**EntryPoint** — The singleton contract in ERC-4337 that all UserOperations flow through. Handles validation, payment collection, and dispatch. EIP-8141's `ENTRY_POINT` address (`0xaa`) is a *protocol-defined caller*, not a deployed contract; the names are similar but the concepts differ.

**EOA (Externally Owned Account)** — An Ethereum account controlled by a private key rather than deployed code. Historically second-class in AA schemes; EIP-8141's default code makes EOAs first-class users of frame transactions without migration.

**Keystore** — An L1 registry that stores multiple keys (passkeys, hardware, backup, session) for a given user and answers "can key X sign for user Y?" Complementary to frame transactions: frames handle per-transaction validation; keystores handle cross-chain identity persistence. EIP-8141 does not include a keystore.

<span id="public-erc-20-sponsor"></span>

**Public ERC-20 sponsor** — An EIP-8141 non-canonical paymaster pattern where the sponsor's VERIFY frame checks sponsor authorization data and inspects the next SENDER frame for an ERC-20 transfer of the right shape. It does not read the user's token balance during validation, so it can propagate through the restrictive public mempool subject to `MAX_PENDING_TXS_USING_NON_CANONICAL_PAYMASTER = 1`. Trust model: the sponsor absorbs frontrunning risk because the user can drain the token balance before inclusion. Independent of ERC-4337.

**Modular account** — A smart account whose validation and execution logic is composed from pluggable modules (validators, executors, hooks), standardized by ERC-7579 and ERC-6900. ERC-8286 defines how such accounts implement EIP-8141 frame validation.

**Paymaster** — A contract that pays gas on behalf of a transaction's sender. In ERC-4337, paymasters implement a `validatePaymasterUserOp` interface and are gated by the EntryPoint. In EIP-8141, paymasters are plain contracts targeted by a VERIFY frame with payment scope; the canonical paymaster is a runtime-code-recognized ETH-funded variant. ERC-20 repayment is non-canonical and splits between [public risk-accepting sponsors](#public-erc-20-sponsor) and [trustless balance-checking sponsors](#trustless-erc-20-balance-checking-sponsor).

<span id="trustless-erc-20-balance-checking-sponsor"></span>

**Trustless ERC-20 balance-checking sponsor** — An EIP-8141 paymaster pattern where a self-contained on-chain contract uses frame introspection and ERC-20 storage reads to confirm the user can repay before approving payment. No off-chain service participates, and the sponsor does not accept balance-drain risk. Because validation reads external token storage, this exceeds the restrictive-tier rule `storage reads only on tx.sender` and does not propagate through the public mempool; it routes through the expansive tier, a private mempool, or direct-to-builder submission. Independent of ERC-4337.

**Relayer** — A third-party service that accepts signed user operations off-chain and submits them on-chain. EIP-8141 argues the role is structurally reduced because validation runs in-protocol: privacy rebroadcasters and trustless ERC-20 balance-checking sponsors are expressible as onchain contracts that route through the expansive tier or private mempool, while public ERC-20 sponsors can propagate through the restrictive mempool by accepting frontrunning risk.

**Session key** — A scoped, time-bounded key that can sign a limited set of operations on behalf of a primary account. Popular pattern for AI agents, games, and graduated-permission wallets. Not a protocol default; implemented in account code or via ERCs like ERC-7710/7715 and ERC-7895.

**Smart account (smart contract account)** — An Ethereum account whose address holds deployed code that defines custom validation and execution logic. The ERC-4337 default. EIP-8141 supports both smart accounts (via VERIFY-frame account code) and EOAs (via default code).

**UserOperation** — The pseudo-transaction object in ERC-4337 carrying sender, call data, signature, and paymaster data. Processed by bundlers, not the public mempool. EIP-8141 eliminates it; frame transactions are real transactions in the standard mempool.

---

## Related EIPs

Proposals EIP-8141 depends on, supersedes, or interacts with. For full context on any of these, consult the linked spec.

**[EIP-1559](https://eips.ethereum.org/EIPS/eip-1559)** — Fee market with `max_priority_fee_per_gas` + `max_fee_per_gas`. EIP-8141 inherits the fee model; listed in `requires`.

**[EIP-2718](https://eips.ethereum.org/EIPS/eip-2718)** — Typed transaction envelope. EIP-8141 is transaction type `0x06`. The type-byte sighash prefix (PR #11544) follows the EIP-2718 convention.

**[EIP-2780](https://eips.ethereum.org/EIPS/eip-2780)** — Intrinsic and value-transfer gas repricing. EIP-8141 uses its 12,000 base cost, 6,000 value cost, and runtime new-account charge after PR #12062.

**[EIP-2542](https://eips.ethereum.org/EIPS/eip-2542)** — 2020 proposal for `TXGASLIMIT`/`CALLGASLIMIT` gas-introspection opcodes. Moved to Withdrawn with `withdrawal-reason: Superseded by EIP-8141` (PR #11773, merged Jun 30, 2026), since `TXPARAM`/`FRAMEPARAM` cover the use case. The first EIP formally withdrawn in favor of frame transactions.

**[EIP-3074](https://eips.ethereum.org/EIPS/eip-3074)** — `AUTH`/`AUTHCALL` opcodes giving EOAs the ability to delegate authorization to contracts. Never shipped; its design principles feed into EIP-8141.

**[EIP-3529](https://eips.ethereum.org/EIPS/eip-3529)** — Storage-refund rules, including the one-fifth refund cap. EIP-8141 uses one transaction-level refund counter across all frames after PR #11940.

**[EIP-3607](https://eips.ethereum.org/EIPS/eip-3607)** — Rejects transactions from senders that have deployed code. Conflicted with frame transactions allowing contract-account senders; [PR #11272](https://github.com/ethereum/EIPs/pull/11272) (merged May 5, 2026) added EIP-3607 to the `requires` header with an explicit carve-out: the origination check does not apply to frame transactions, while non-frame transaction validation is unchanged.

**[EIP-4844](https://eips.ethereum.org/EIPS/eip-4844)** — Blob transactions for L2 data availability. EIP-8141 validates KZG versioned hashes, exposes `BLOBHASH`, and includes blob gas in payer settlement.

**[EIP-7594](https://eips.ethereum.org/EIPS/eip-7594)** — PeerDAS networking and pooled-transaction wrapper used to propagate frame transactions with blob cells and proofs (PR #11985).

**[EIP-7623](https://eips.ethereum.org/EIPS/eip-7623)** — Calldata cost floor. EIP-8141 applies its token floor to frame and signature data through `calldata_floor_gas`; EIP-8250 also prices nonce-key data under the same rule after PR #11958.

**[EIP-7702](https://eips.ethereum.org/EIPS/eip-7702)** — Lets an EOA delegate its code to a smart contract, transiently or persistently. EIP-8141's default code replaces EIP-7702 for common cases. 7702-delegated EOAs can still send frame transactions, but default code does not run; their delegated contract must implement `APPROVE`.

**[EIP-7708](https://eips.ethereum.org/EIPS/eip-7708)** — ETH transfer log convention. Non-zero EIP-8141 value transfers to non-sender targets emit its transfer log after PR #12062.

**[EIP-7805 (FOCIL)](https://eips.ethereum.org/EIPS/eip-7805)** — Fork-choice-enforced inclusion lists. Tightly coupled with VOPS; determines censorship resistance of the restrictive mempool.

**[EIP-7825](https://eips.ethereum.org/EIPS/eip-7825)** — Transaction execution-gas cap. EIP-8141 limits intrinsic plus all declared execution budgets, and the calldata floor independently, to `16,777,216`.

**[EIP-7843](https://eips.ethereum.org/EIPS/eip-7843)** — Slot-number-as-data EIP. Exposes the current slot number to the EVM; EIP-8272 (Recent Roots) requires it so validation can derive `current_slot` without reading `block.timestamp`.

**[EIP-7928](https://eips.ethereum.org/EIPS/eip-7928)** — Block-level access lists. EIP-8141 intentionally has no transaction-level access list; block-level ALs handle optimization.

**[EIP-7997](https://eips.ethereum.org/EIPS/eip-7997)** — The deterministic factory predeploy. EIP-8141's canonical-but-non-mandatory factory for account deployment via deploy frames. Listed in `requires` only between PR #11521 (Apr 14) and PR #11567 (Apr 30); the latter dropped it from `requires` and rewrote the deploy-frame mempool rule as a stateless-trace policy any factory can satisfy.

**[EIP-8037](https://eips.ethereum.org/EIPS/eip-8037)** — Two-dimensional gas model separating execution from state growth. EIP-8141 declares both budgets per frame rather than using EIP-8037's reservoir compatibility model.

**[EIP-8081 (Hegotá meta)](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-8081.md)** — The Hegotá fork meta-EIP tracking CFI/PFI/SFI/DFI status for candidate EIPs. EIP-8141 was added to CFI via [PR #11537](https://github.com/ethereum/EIPs/pull/11537), merged Apr 30, 2026.

**[EIP-8184 (LUCID)](https://eips.ethereum.org/EIPS/eip-8184)** — Encrypted mempool proposal. Incompatible with the restrictive tier; routes through expansive tier and onchain rebroadcasters.

**[EIP-8215 (Hash-Committed Account)](https://github.com/ethereum/EIPs/pull/11480)** — Open complementary PQ address-derivation proposal. New account addresses derive from a Merkle root of spending conditions rather than a public key; positioned alongside EIP-8141, not as a replacement for frame transactions.

**[EIP-8250 (Keyed Nonces)](https://eips.ethereum.org/EIPS/eip-8250)** — First sibling EIP requiring EIP-8141. Adds `(nonce_keys, nonce_seq)` replay-protection streams via `NONCE_MANAGER`. PR #11968 made its changes additive; PR #12067 pinned the system address to `0x...8250`.

**[EIP-8266 (Expiring Nonces)](https://eips.ethereum.org/EIPS/eip-8266)** — Second sibling EIP (`requires` EIP-8141 and EIP-8250). Sentinel-mode (`tx.nonce == 2**64 - 1`) time-windowed nonces via a `NONCE_RING` system contract (PR #11692 merged May 22).

**[EIP-8272 (Recent Roots)](https://eips.ethereum.org/EIPS/eip-8272)** — Third sibling EIP. Adds `recent_root_references`, a pinned `RECENT_ROOT_ADDRESS`, and validation-time root access. Its current `RECENTROOTREFLOAD (0xb5)` assignment conflicts with EIP-8141 `SIGDATACOPY`.

**[EIP-8288 (PQ frame mode)](https://ethereum-magicians.org/t/eip-frame-type-for-quantum-resistant-signature-and-stark-aggregation/28723)** — Fourth sibling EIP, pending (PR #11772 opened Jun 5, now committed as `eip-8288.md`; editorial review Jun 30). Adds frame mode `DEP_VERIFY_FRAME_MODE = 3` and a block-header `recursive_stark` field for post-quantum signature and STARK aggregation.

**[EIP-8361 (Transaction Validity Proofs)](https://github.com/ethereum/EIPs/pull/12075)** — Open draft for mempool-layer STARK proofs of EIP-8141 validation-prefix validity. It does not change consensus.

**[EIP-8369 (VOPS Profiles)](https://github.com/ethereum/EIPs/pull/12110)** — Open draft defining end-of-payload and builder-claimed-index profiles for frame-transaction FOCIL eligibility.

**[EIP-7951](https://eips.ethereum.org/EIPS/eip-7951)** — P256 precompile. EIP-8141 can use it for protocol-validated P256 outer signatures and account-code passkey/WebAuthn verification, but codeless EOA default code no longer relies on it.

**[ERC-4337](https://eips.ethereum.org/EIPS/eip-4337)** — The off-chain AA standard deployed today via bundlers, EntryPoint, and paymasters. EIP-8141 is its native protocol successor.

**[ERC-5792](https://eips.ethereum.org/EIPS/eip-5792)** — The `wallet_sendCalls` standard for wallet batch calls. Cited as the canonical slow-convergence/fragmentation precedent that protocol-level batching defaults aim to avoid.

**[ERC-6900](https://eips.ethereum.org/EIPS/eip-6900)** — A modular smart-account standard (validation/execution plugins). Listed alongside ERC-7579 as an account standard the transport-agnostic ERC-8211 can run over.

**[ERC-7562](https://eips.ethereum.org/EIPS/eip-7562)** — The validation-rules framework for ERC-4337 UserOperations. EIP-8141's restrictive-tier mempool policy is inspired by ERC-7562 but simpler (no staking, no reputation).

**[ERC-7579](https://eips.ethereum.org/EIPS/eip-7579)** — Minimal modular smart-account standard (validator, executor, hook, config modules). ERC-8286 builds the EIP-8141 frame-validation flow on top of it.

**[ERC-7710 / ERC-7715](https://eips.ethereum.org/EIPS/eip-7715)** — MetaMask's delegation and permissions-request standards for session keys. Contrasted with Base's ERC-7895 as evidence of session-key fragmentation absent protocol defaults.

**[ERC-7895](https://eips.ethereum.org/EIPS/eip-7895)** — Base's wallet permissions (Spend Permissions) standard. The ERC-7710/7715-vs-7895 split is the divergence protocol-level defaults aim to avoid.

**[ERC-8211 (Smart Batching)](https://ethereum-magicians.org/t/erc-8211-smart-batching/28135)** — Transport-agnostic `ComposableExecution[]` batch-encoding ERC (draft). Names EIP-8141 SENDER frames as a future execution path without taking it as a dependency.

**[ERC-8286 (Modular Accounts for Frame Transactions)](https://ethereum-magicians.org/t/erc-8286-modular-accounts-for-frame-transactions/28695)** — Merged ERC, `requires: 7579, 8141`. Standardizes how ERC-7579 validator modules return the approval mode applied through `APPROVE` in a VERIFY frame.

**[ERC-8340 (Transaction Metadata Encoding)](https://github.com/ethereum/ERCs/pull/1883)** — Draft application-layer companion to EIP-8130. Defines deterministic CBOR for its opaque `metadata` field, including attribution, memos, call/phase scope, and selectively disclosed offchain commitments.

---

## Alternative AA proposals

Each has a dedicated page in the [Alternatives](/competing-standards) sidebar group.

**[EIP-8130](/eip-8130)** — AA by Account Configuration (Chris Hunter, Coinbase/Base). Declarative authenticator-based validation instead of arbitrary wallet-code execution. Most direct competitor to EIP-8141.

**[EIP-8175](/eip-8175)** — Composable Transaction (Dragan Rakita). Flat list of typed capabilities plus separated signatures and programmable `fee_auth`. The flat-composition counterpoint to frame-based AA.

**[EIP-8202](/eip-8202)** — Scheme-Agile Transactions (Giulio Rebuffo, Ben Adams). Single execution payload with scheme-agile authorization list. Narrowest bet: PQ signatures on L1 without general AA.

**[EIP-8223](/eip-8223)** — Contract Payer Transaction (Ben Adams). Static gas sponsorship via a canonical payer registry at `0x13`. **Complementary** to EIP-8141, not competing.

**[EIP-8224](/eip-8224)** — Counterfactual Transaction (Ben Adams). Shielded gas funding via fflonk ZK proofs against canonical fee-note contracts. **Complementary** to EIP-8141 and EIP-8223; solves the bootstrap problem.

**[EIP-XXXX (Tempo-like)](/eip-xxxx)** — Constrained-scope transaction type (Georgios Konstantopoulos, Paradigm/Reth, pre-draft gist). Fixed UX primitives (batching, windows, passkeys, sponsorship, 2D nonces, access keys) with no programmable validation.

---

## Governance and timeline

**ACDE / ACDC** — *All Core Devs Execution* / *All Core Devs Consensus*. The two biweekly calls where protocol changes are discussed and moved between governance statuses. Meeting numbers (e.g., ACDE #233) are cited when decisions are captured.

**CFI / PFI / SFI / DFI** — The four governance statuses tracked in the Hegotá meta-EIP ([EIP-8081](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-8081.md)): *Considered for Inclusion*, *Proposed for Inclusion*, *Scheduled for Inclusion*, *Declined for Inclusion*. EIP-8141 reached CFI status via PR #11537 (merged Apr 30, 2026).

**EVVM** — A contract-native account-abstraction framework (~200 deployed instances since 2023). Its co-founder contributed a production-perspective comparison (Magicians post #148) on per-environment policy, async execution, and reservation primitives.

**Forkcast** — The site tracking Ethereum fork planning, ACDE/ACDC call timestamps, and governance decisions. Cited alongside ACDE numbers.

**Hegotá** — The next scheduled Ethereum hard fork after Glamsterdam. Target timeframe H2 2026. EIP-8141 is listed as CFI in its meta-EIP.

**Strawmap** — The [Ethereum L1 Strawmap](https://strawmap.org/) identifying five "north stars" for the protocol: PQ L1, native privacy, and others. Defines the multi-year roadmap EIP-8141 fits into.

**Svalbard** — The April 2026 native-AA breakout at the Svalbard interop event that ratified the constraint set EIP-8141 converged toward: no relayers for core functionality, statelessness and FOCIL compatibility, and ETH-funded sponsorship as the core case.

---

## Where else to look

- Opcode-level details, execution rules, and the mempool policy live in [Current Spec](/current-spec).
- The tension between VOPS, FOCIL, and frames gets its own plain-English explainer at the top of [VOPS Compatibility](/vops-compatibility).
- Short-answer form for common questions lives in the [FAQ](/faq).
- External sources, PR timeline, and discussion threads live in the [Appendix](/appendix).
