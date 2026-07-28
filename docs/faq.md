# Frequently Asked Questions

---

## 1. General

**1.1. What is EIP-8141?**

A new transaction type (`0x06`) that splits a transaction into multiple frames - each with a purpose (verify, execute, deploy), giving every account programmable validation and native batching at the protocol level. [Read the spec overview →](/current-spec)

**1.2. What problem does it solve?**

Today, advanced transaction features (gas sponsorship, batching, custom signatures) require off-chain infrastructure and smart contract middleware. EIP-8141 moves these capabilities into the protocol itself.

**1.3. What are frames?**

Ordered steps within a single transaction. Each frame has a mode - `VERIFY` (authenticate), `SENDER` (execute), or `DEFAULT` (deploy/post-op) - that tells the protocol what the frame does. [See frame modes →](/current-spec#frame-modes)

**1.4. Is EIP-8141 live on mainnet?**

No. It is a draft EIP under active development targeting a future hard fork. [Track progress on GitHub →](https://github.com/ethereum/EIPs/pulls?q=is%3Apr+8141)

**1.5. What new opcodes does it introduce?**

Six: `APPROVE`, `TXPARAM`, `FRAMEDATALOAD`, `FRAMEDATACOPY`, `FRAMEPARAM`, and `SIGPARAM` (signature metadata and custom witness bytes). [Details →](/current-spec#transaction-structure)

---

## 2. ERC-4337 & Bundlers

**2.1. Does EIP-8141 replace ERC-4337?**

In the long run, yes. EIP-8141 is the native protocol successor, moving account abstraction into the transaction layer and eliminating bundlers, the EntryPoint contract, and off-chain UserOperation infrastructure for new flows. In the short term, EIP-8141 is still a draft and ERC-4337 remains the production AA stack; existing 4337 wallets do not stop working when 8141 ships.

**2.2. Why are bundlers no longer needed?**

The protocol itself handles validation, gas payment, and execution ordering through frames. There is no separate UserOperation mempool - frame transactions use the standard public mempool.

**2.3. What about existing ERC-4337 wallets?**

Smart accounts built for ERC-4337 can migrate their validation logic into VERIFY frames. The core verification code (signature checks, access policies) is reusable - what changes is how it's invoked.

**2.4. Does this affect ERC-4337 paymaster contracts?**

Yes. EIP-8141 introduces its own [canonical paymaster mechanism](/current-spec#mempool-policy) at the protocol level, replacing ERC-4337's paymaster interface.

**2.5. What happens to bundler operators?**

The bundler role is absorbed by the protocol and standard block builders. There is no separate bundler market or infrastructure to maintain.

---

## 3. EIP-7702 & Account Delegation

**3.1. Does EIP-8141 replace EIP-7702?**

For most use cases, yes. EIP-7702 requires EOAs to permanently delegate to a smart contract, a persistent on-chain state change. EIP-8141 gives EOAs native AA without any delegation, code deployment, or state change. [See EOA default code →](/current-spec#eoa-default-code)

**3.2. Can 7702-delegated accounts still use EIP-8141?**

Yes, but with a caveat. A 7702-delegated account sends frame transactions like any other, yet the protocol's default code does not run; the delegated contract's code runs instead. If that contract does not implement the `APPROVE` opcode, the account loses the signature-verification path default code would have given it. This is a real interoperability gap flagged by DanielVF (posts #120, #122). EOAs that want default-code behavior should not 7702-delegate.

**3.3. Why is removing the 7702 dependency important?**

EIP-7702 relies on ECDSA for its authorization list, making it incompatible with post-quantum signature schemes. EIP-8141 has no authorization list - accounts choose their own cryptography. [See competing standards →](/competing-standards#ecdsa-decoupling)

---

<span id="4-users"></span>

## 4. Users

**4.1. What does EIP-8141 mean for regular users?**

Users get gas sponsorship, atomic batching, and secp256k1 default-code signing without needing to deploy smart contracts, migrate to new addresses, or rely on third-party relayers. Passkeys/P256 are supported as protocol-validated outer signatures, but codeless EOA default code does not accept them directly.

**4.2. Can I keep my existing EOA address?**

Yes. EOAs work natively with frame transactions - no code deployment, no delegation, no address change. Your account stays codeless before, during, and after the transaction.

**4.3. Can I pay gas in ERC-20 tokens?**

Yes, through non-canonical paymasters. The public-mempool shape lets a sponsor inspect sponsor data and the next ERC-20 transfer frame, then accept the risk that the user drains the token balance before inclusion. A trustless onchain balance-checking paymaster can read token storage, but that exceeds the restrictive tier and routes through the expansive tier, a private mempool, or direct-to-builder submission. Neither pattern relies on ERC-4337 infrastructure. [See Mempool Strategy → ERC-20 gas repayment →](/mempool-strategy#erc20-paymaster-patterns)

**4.4. Can I batch multiple actions in one transaction?**

Yes. Consecutive DEFAULT or SENDER frames with the atomic flag revert together if any fails; VERIFY frames cannot participate in or terminate a batch. [See atomic batching →](/current-spec#atomic-batching)

**4.5. Do I need a smart contract wallet to use this?**

No. The protocol has built-in default behavior for codeless accounts: secp256k1 ECDSA signature verification, plus native ETH transfer in SENDER frames via `frame.value`. [Details →](/current-spec#eoa-default-code)

**4.6. Is this compatible with passkeys / biometrics?**

Partly. The outer signatures list includes canonical low-`s` `P256 (0x2)`, but codeless EOA default code does not accept it. Passkey/WebAuthn authorization therefore needs deployed account code or a future extension EIP.

**4.7. How do I send ETH to someone with a frame transaction?**

Build a SENDER frame with `target = destination` and `value = amount`; no payload encoding needed. The per-frame `value` field was added by PR #11534 (Apr 16). Non-zero `value` is only valid in SENDER frames; DEFAULT and VERIFY frames must set `value = 0`. [See current spec →](/current-spec#transaction-structure)

**4.8. Can one codeless EOA sponsor another codeless EOA?**

Yes. Default code uses signature index `0` for the sender's execution approval and index `1` for a distinct payment-only EOA sponsor (PR #11954). The sponsor is non-canonical, so the public mempool allows only one pending transaction per sponsor.

**4.9. Can frame transactions carry blobs?**

Yes. They validate KZG versioned hashes, use the EIP-7594 pooled-transaction wrapper, expose `BLOBHASH`, and settle blob gas against the payer. [See blob support →](/current-spec#blob-support)

---

## 5. Wallet Developers

**5.1. How does EIP-8141 reduce wallet development overhead?**

Bundler infrastructure, UserOperation formatting, EntryPoint ABI compatibility, and ERC-4337 paymaster integration all go away. Wallets construct a standard transaction with frames instead.

**5.2. Do wallets still need to run or depend on bundlers?**

Not for frame transactions. They enter the public mempool like any other transaction, with no separate bundler endpoint, selection, or availability concerns. Wallets that still serve ERC-4337 UserOperations continue to need bundler infrastructure for that path; 8141 and 4337 can coexist during migration.

**5.3. Does this reduce vendor dependency?**

Yes. Today, wallets depend on bundler providers (Pimlico, Alchemy, etc.) for AA functionality. With EIP-8141, the protocol is the infrastructure - any Ethereum node can validate and propagate frame transactions.

**5.4. What about gas sponsorship infrastructure?**

For high-throughput ETH-funded sponsorship, wallets use the canonical paymaster. A codeless EOA can also sponsor through payment-only default code at signature index `1`, subject to the one-pending-transaction non-canonical cap. ERC-20 repayment retains the public risk-accepting versus private/expansive trustless split. [See 4.3 →](#4-users)

**5.5. Can wallets still build custom validation logic?**

Yes. VERIFY frames execute arbitrary account code - wallets can implement multisig, social recovery, session keys, or any scheme. The [mempool policy](/current-spec#mempool-policy) constrains what's publicly relayable, but custom logic is valid on-chain.

**5.6. What's the migration path from ERC-4337?**

Move validation logic from `validateUserOp` into VERIFY frame code that calls `APPROVE`. Replace bundler submission with standard transaction broadcasting. Replace EntryPoint paymaster calls with canonical paymaster VERIFY frames.

**5.7. Can I give an AI agent or session a scoped key that expires?**

Yes, via account code. Default code covers only secp256k1 at signature index 0 on the primary key; richer policies (expiry, per-call caps, allowlists) live in the account's own VERIFY logic and remain valid on-chain. Whether such a transaction propagates publicly depends on the [mempool tier](/mempool-strategy#two-tiers-in-one-mempool) it fits.

**5.8. How do custom verifiers get signature bytes without circular hashes?**

They put witness bytes in a 100-gas `ARBITRARY` signature entry. `SIGPARAM` exposes the bytes, and empty-`msg` entries elide them from the canonical hash so the witness does not commit to itself.

**5.9. Is there a standard for existing modular smart accounts to use frame transactions?**

Yes. ERC-8286 (draft, `requires: 7579, 8141`) standardizes how ERC-7579 modular accounts implement the frame validation flow: a validator module returns an approval mode the account applies via `APPROVE` in a VERIFY frame. See [Developer Tooling](/developer-tooling#where-fragmentation-risk-still-lives).

**5.10. Must wallets normalize P256 signatures?**

Yes. EIP-8141 requires canonical low-`s` P256 even though `P256VERIFY` accepts high-`s`; wallets must replace high `s` with `SECP256R1N - s`.

---

## 6. Post-Quantum Readiness

**6.1. Is EIP-8141 post-quantum safe?**

The transaction format itself has no ECDSA dependency. Accounts choose their own signature scheme in VERIFY frames, and custom witness bytes can live in `ARBITRARY` signature entries exposed by `SIGPARAM`. Any PQ algorithm can be used by account code without changing the frame transaction envelope.

**6.2. How does this compare to other proposals?**

EIP-8141 offers the most flexible PQ path (arbitrary account code plus `ARBITRARY` signature witnesses). EIP-8202 now includes native Falcon-512 support (updated from its original ephemeral-k1 design). EIP-8130 requires deploying PQ authenticator contracts and getting them into the canonical authenticator set. [Full comparison →](/competing-standards#ecdsa-decoupling)

---

## 7. Mempool & Network Health

**7.1. How do nodes validate frame transactions?**

Nodes simulate the validation prefix. If every prefix frame uses protocol-defined default, expiry-verifier, or canonical-paymaster code, they may evaluate it directly with identical dependencies, gas, and `MAX_VERIFY_GAS` results. [Mempool policy →](/current-spec#mempool-policy)

**7.2. What is the canonical paymaster?**

A standardized paymaster contract recognized by mempool policy. Nodes verify it by runtime code match and track reserved balances for pending transactions. [Spec details →](/current-spec#mempool-policy)

**7.3. What if wallets don't adopt the canonical paymaster?**

They can propagate only under the one-pending-transaction cap per non-canonical paymaster. Higher-volume or richer paymasters route through the expansive tier or private submission and lose the canonical paymaster's FOCIL-friendly path. [See open question →](/mempool-strategy#canonical-paymaster-adoption)

**7.4. Does this affect censorship resistance?**

Potentially. Mempool health is censorship resistance - if minimal nodes can't validate certain frame transactions, those transactions lose public propagation guarantees. [See open question →](/mempool-strategy#mempool-health-and-censorship-resistance)

**7.5. Can a frame transaction expire?**

Yes. PR #11662 (merged May 14) added an expiry-verifier frame: a `VERIFY` frame targeting `address(0x8141)` whose `frame.data` is an 8-byte unix-seconds deadline. The canonical runtime reverts if the deadline has passed, and the public mempool drops the transaction as soon as the deadline is in the past. At most one such frame is valid; public propagation requires it to be first. [Spec details →](/current-spec#expiry-verifier-frame)

**7.6. What's the difference between EXPIRY_VERIFIER and EIP-8266 expiring nonces?**

`EXPIRY_VERIFIER` (PR #11662, merged May 14) expires a single transaction: past the deadline, that specific transaction becomes invalid. EIP-8266 (PR #11692, merged May 22) expires a nonce slot: past the deadline, the keyed-nonce sequence as a whole ages out. They are complementary primitives.

---

## 8. Statelessness & Scalability

**8.1. How does EIP-8141 interact with statelessness goals?**

Frame transaction validation requires more state access than legacy transactions (~100k gas vs ~3k gas). This tensions with VOPS (Validity-Only Partial Statelessness) nodes that carry minimal state. [Details →](/vops-compatibility#validation-state-requirements)

**8.2. What is the "choose 2 of 3" trilemma?**

The observation that current designs cannot simultaneously deliver Frames/Native AA, Public Mempool/FOCIL, and Statelessness/VOPS - you can have at most two. [Details →](/vops-compatibility#the-frames-focil-vops-trilemma) The proposed resolution under a [two-tier mempool + VOPS+4 + merkle escape hatch](/mempool-strategy#resolving-the-trilemma) is detailed in Mempool Strategy.

**8.3. Is there a workaround for VOPS compatibility?**

Yes. The proposed [VOPS extension](/mempool-strategy#the-state-side-vops-4-slots) covers nonce, balance, code, and the first 4 storage slots per account, which handles most AA validation reads. Use cases that exceed this baseline include a [merkle branch](/mempool-strategy#the-merkle-branch-escape-hatch) (4-8 kB today, 1-2 kB after binary tree).

**8.4. What about encrypted mempools?**

Encrypted mempools (e.g., LUCID protocol) are fundamentally incompatible with the public restrictive mempool. The proposed framework routes them through the [expansive tier and onchain rebroadcaster contracts](/mempool-strategy#why-frame-transactions-dont-need-relayers) instead. [See open question →](/mempool-strategy#encrypted-mempool-compatibility)

**8.5. How much extra state do nodes need?**

At full AA adoption with 4 cached storage slots per account, VOPS nodes would need ~72 GB total - an 8x increase from today's ~10 GB floor. [Details →](/vops-compatibility#state-growth-at-scale)

**8.6. Can privacy pools use frame transactions today?**

Not through the public mempool or FOCIL. Frames structurally remove relayer trust (invalid or replayed proofs revert in VERIFY before gas is charged), but Groth16 verification exceeds the 100k VERIFY cap, per-IL gas budgets fit only ~2 frame transactions, and nullifier slots live outside VOPS+4. [See the three gates →](/mempool-strategy#privacy-pools-three-gates)

---

## 9. Competing Proposals

**9.1. What are the alternatives to EIP-8141?**

Four competing proposals: EIP-8175 (composable capabilities with programmable fee_auth), EIP-8130 (declared authenticators, no wallet-code execution in validation), EIP-8202 (scheme-agile flat transactions), and a Tempo-like proposal (fixed UX primitives). [Full comparison →](/competing-standards)

**9.2. Which is most likely to ship?**

Unclear. EIP-8141 is the most comprehensive but also the most complex. EIP-8130 has strong Base/Coinbase backing. EIP-8175, EIP-8202, and Tempo are earlier stage. [See activity comparison →](/competing-standards#summary)

**9.3. Can EIP-8130 be built on top of EIP-8141?**

In principle yes. Declared authenticators are a subset of what VERIFY frames can do, while the reverse is not true. This is a key argument from EIP-8141 proponents. EIP-8130 authors contest the framing: their case is that a narrower primitive with no wallet-code execution in validation is better suited to performance chains regardless of whether it can be rebuilt on 8141. Both proposals remain active.

**9.4. What is EIP-8223 and how does it relate to EIP-8141?**

A new proposal (PR #11509, Apr 11 2026) for static gas sponsorship via a canonical payer-registry predeploy at `0x13`. Its author explicitly frames it as complementary to EIP-8141 and EIP-8175, covering the narrow case where validation needs only an SLOAD plus a balance check. [See EIP-8223 →](/eip-8223)

**9.5. What is EIP-8224 and how does it relate to EIP-8141?**

A follow-up proposal (PR #11518, Apr 12 2026) for shielded gas funding via fflonk ZK proofs against canonical fee-note contracts. Solves the bootstrap problem of funding a fresh EOA without traceable on-chain links. Composes with EIP-8223 (one-shot bootstrap, then sponsored txs). [See EIP-8224 →](/eip-8224)

---

## 10. Implementation & Timeline

**10.1. Is implementation complexity a concern?**

Yes. Frame transactions touch consensus, mempool policy, p2p propagation, client state management, and wallet infrastructure simultaneously. This breadth is a contributing factor to Glamsterdam delays. [Details →](/vops-compatibility#implementation-complexity)

**10.2. Where can I follow the spec development?**

- [EIP-8141 spec on GitHub](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-8141.md)
- [All related PRs](https://github.com/ethereum/EIPs/pulls?q=is%3Apr+8141)
- [Ethereum Magicians discussion](https://ethereum-magicians.org/t/frame-transaction/27617)
- [Statelessness concerns on ethresear.ch](https://ethresear.ch/t/frame-transactions-through-a-statelessness-lens/24538)

**10.3. What is EIP-8141's Hegotá inclusion status?**

It remains Considered for Inclusion (CFI), not Proposed for Inclusion (PFI). ACDE #241 on Jul 16 described it as the native-AA placeholder; no AA-stack proposal was promoted.

**10.4. How are frame and signature bytes charged?**

They contribute to `standard_gas_limit` and the EIP-7623 `calldata_floor_gas`; `max_gas` reserves the larger amount. Final `gas_used` applies the EIP-3529 refund, then enforces the floor.

**10.5. What do frame receipt status values mean?**

`0x0` means failure, `0x1` success, and `0x2` skipped after an atomic-batch failure. Per-frame gas is gross before the transaction refund and may not sum to final transaction `gas_used`.

**10.6. Did the sibling EIPs collide with EIP-8141's introspection assignments?**

Yes, and the July fixes resolved them: EIP-8250 starts nonce-key selectors at `0x10` (#11966), while EIP-8272 uses count selector `0x0f` (#11930) and `RECENTROOTREFLOAD (0xb5)` (#11967).

**10.7. How much gas does APPROVE cost?**

`APPROVE` has no base charge and pays only memory expansion, matching `RETURN`; related nonce, payer, and escrow work is covered by intrinsic transaction costs.

---

## Read Next

- [Current Spec](/current-spec) — the mechanism behind the answers above.
- [Developer Tooling](/developer-tooling) — if you're building a wallet or app.
- [Competing Standards](/competing-standards) — if you're evaluating EIP-8141 against alternatives.
- [Feedback Evolution](/feedback-evolution) — if you want to see how the spec got here.

---

*For VOPS and statelessness details, see [VOPS Compatibility →](/vops-compatibility). For mempool open questions, see [Mempool Strategy →](/mempool-strategy#open-questions).*
