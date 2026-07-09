# Post-Quantum Roadmap

---

## TL;DR

EIP-8141 does not solve post-quantum security for Ethereum. It is the **first step** in a long roadmap requiring multiple coordinated protocol upgrades before Ethereum accounts are truly quantum-safe. The roadmap spans seven stages: decoupling accounts from ECDSA, PQ precompiles, ephemeral key rotation, encrypted mempools, complete secp256k1 revocability, native PQ transactions, and private L1 settlement.

The [Ethereum L1 Strawmap](https://strawmap.org/) identifies post-quantum L1 as one of five "north stars," targeting "centuries-long cryptographic security via hash-based schemes" through seven forks by end of 2029. The urgency is real: [Google's Q-Day research](https://blog.google/innovation-and-ai/technology/safety-security/cryptography-migration-timeline/) estimates elliptic curve cryptography could be broken with fewer than 500,000 physical qubits in minutes, with 2029 as the projected timeline. Each stage builds on the previous one; skipping stages creates gaps a quantum adversary could exploit.

EIP-8141 is the foundational primitive that makes the rest possible. Without account decoupling from ECDSA, none of the subsequent stages have a protocol surface to build on.

---

## 1. Foundation: Decoupling Accounts from ECDSA

**Status**: EIP-8141 (Draft), targeted for [Hegota fork](https://strawmap.org/) (H2 2026) as CFI. Added to the EIP-8081 meta EIP via [PR #11537](https://github.com/ethereum/EIPs/pull/11537), merged Apr 30, 2026, per ACDE #233 decisions.

Today every Ethereum account is bound to a secp256k1 key pair. Shor's algorithm can recover the private key from an exposed public key in polynomial time. EIP-8141 breaks this coupling: [VERIFY frames](/current-spec#frame-modes) let any account code define its own signature verification, and the outer `signatures` list can carry protocol-validated `P256` signatures or `ARBITRARY` custom witness bytes. [EOA default code](/eoa-support) gives existing codeless EOAs secp256k1 frame authentication without deploying a smart contract, but it does not make those EOAs passkey/P256-authenticated by default.

The frame architecture is deliberately generic, providing foundational primitives (VERIFY frames, APPROVE, per-frame gas) without prescribing which PQ scheme to use. This gives the ecosystem time to research PQ curves before committing.

**Delivers**: the protocol surface for PQ migration. **Does not deliver**: actual quantum safety. Default code still signs with secp256k1, and P256 is also quantum-vulnerable even where account code uses it. The ECDSA key remains the master key unless account code or a future EIP revokes it.

---

> **⎯ Where EIP-8141 ends, the roadmap begins ⎯**
>
> Stage 1 above is what EIP-8141 itself delivers and is what the rest of this site documents. Stages 2 through 7 below are **future research**: separate EIPs, separate forks, separate open questions. EIP-8141 is a prerequisite for each of them but does not implement any of them. If you are reading this page for what ships in Hegotá, stop at Stage 1; the rest is the direction of travel.

---

## 2. PQ-Ready Precompiles and Opcodes

**Status**: Research (EIP-8051, EIP-8052)

PQ signature schemes are expensive. Falcon-512 costs over 1M gas in the EVM; Dilithium and SPHINCS+ are similarly costly. Dedicated precompiles are needed, following the path P256 took via [RIP-7212](https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7212.md) (~3,450 gas).

- [EIP-8051](https://eips.ethereum.org/EIPS/eip-8051): gas reductions for Dilithium
- [EIP-8052](https://eips.ethereum.org/EIPS/eip-8052): gas reductions for Falcon

EIP-8141 decouples PQ scheme selection from the transaction format: as precompiles ship, accounts adopt them by updating VERIFY frame logic, no protocol changes needed. The [signatures list](/merged-changes) (PR [#11481](https://github.com/ethereum/EIPs/pull/11481), merged May 22; repaired by #11837/#11814 in July) adds forward-compatibility with signature aggregation, critical since PQ signatures are large (Falcon: ~666 bytes, Dilithium: ~2,420 bytes). The current design moves witness bytes out of `frame.data` into a dedicated outer `signatures` list: protocol-validated raw bytes remain hidden from EVM code for aggregation, while `ARBITRARY` bytes are exposed through `SIGPARAM` for custom verifiers. Raw signature bytes with empty `msg` are elided from the canonical hash.

PR [#11772](https://github.com/ethereum/EIPs/pull/11772) (proposed as EIP-8288, opened Jun 5 by vbuterin and Thomas Coratger) makes block-level aggregation concrete but remains open as of Jul 9. It introduces a new frame mode `DEP_VERIFY_FRAME_MODE = 3` that declares `(scheme, data_hash, verification_key)` dependencies and a top-level block-header field `recursive_stark` that aggregates every per-block dependency into a single recursive STARK proof using Lean Ethereum tooling (`LEANSPHINCS_SCHEME = 0x10` at 3000 gas, `LEANSTARK_SCHEME = 0x11` at 30000 gas). The current review blocker is proof security around recursive knowledge soundness, not the high-level frame-transaction integration. EIP-8288 is intended to be FOCIL-compatible by construction.

**Delivers**: practical gas costs for PQ verification, and (via EIP-8288 if merged) the recursive-STARK block-level aggregator that makes per-tx PQ signatures economically viable at scale. **Does not deliver**: mempool-level key protection or legacy key revocation.

**Address derivation gap**: [EIP-8215 Hash-Committed Account](https://github.com/ethereum/EIPs/pull/11480) is a complementary open proposal for new accounts whose addresses derive from a Merkle root of spending conditions rather than a public key. Its own framing is that EIP-8141 upgrades validation for existing accounts, while HCA lets new accounts be born without a public-key-derived address. It does not replace EIP-8141's transaction model; it fills a different PQ gap.

---

## 3. Ephemeral Key Pairs

**Status**: Research ([ethresear.ch](https://ethresear.ch/t/achieving-quantum-safety-through-ephemeral-key-pairs-and-account-abstraction/24273), March 2026)

Even with PQ precompiles, the public key is exposed in the mempool between broadcast and inclusion. A quantum adversary could recover the private key and frontrun.

Ephemeral key pairs (proposed by mvicari) make every signing key single-use: each transaction rotates the authorized signer to a fresh address, permanently retiring the previous key. An EOA that has never transacted is quantum-safe because its public key is hidden behind a hash. Ephemeral keys ensure each key is used exactly once.

Under EIP-8141, this is expressible natively: VERIFY frames check the current signer, SENDER frames rotate to the next one, key derivation follows BIP-44 paths. Gas overhead is ~136k per ERC-20 transfer in the PoC (~100k above baseline), reducible by stage 2 precompiles.

**Delivers**: quantum safety against mempool-level key extraction. **Does not deliver**: protection against builders/proposers who see plaintext transactions.

---

<span id="4-encrypted-mempools"></span>

## 4. Encrypted Mempools

**Status**: EIP-8184 (Draft, March 2026)

Ephemeral keys shrink the exposure window but do not eliminate it. EIP-8184 (LUCID) closes this gap with **sealed transactions** that propagate encrypted, revealed only after ordering is finalized:

1. User broadcasts a sealed transaction: signed ticket (for charging) + ciphertext envelope
2. Includers propagate via FOCIL inclusion lists
3. Builders commit to bundles via hashes before seeing plaintext
4. Key publishers release decryption keys after block commitment
5. Decrypted transactions execute against the prior block state

EIP-8184 is PQ-forward-compatible: its `signature_id` field is extensible beyond the current `EC_DSA_TYPE`, and it describes an explicit EIP-8141 integration path where sealed transactions appear inside frame transactions. It requires [EIP-7805 (FOCIL)](https://eips.ethereum.org/EIPS/eip-7805). See also [Mempool Strategy → Encrypted Mempool Compatibility](/mempool-strategy#encrypted-mempool-compatibility).

**Delivers**: complete concealment of signatures and public keys until after ordering. **Does not deliver**: revocation of the legacy secp256k1 key.

---

## 5. Complete secp256k1 Revocability

**Status**: [EIP-7851](https://eips.ethereum.org/EIPS/eip-7851), [EIP-8151](https://eips.ethereum.org/EIPS/eip-8151), [EIP-8164](https://github.com/prestwich/EIPs/blob/48a93b9a83d8cb3d78a983e4f9f60460651574ab/EIPS/eip-8164.md) (all Draft). EIP-7851/EIP-8151 cover 7702-delegated EOAs; EIP-8164 adds protocol-level Ed25519 migration for any EOA.

As long as the original secp256k1 key retains authority, the account has a quantum-vulnerable surface. Complete revocability means permanently removing the ECDSA key as an authorized signer:

- The protocol must support accounts with no ECDSA fallback
- Legacy transaction types (0, 1, 2) must be rejected for these accounts
- Recovery mechanisms for accounts that lose their PQ key must exist
- Deployed signature-checking contracts (e.g., ERC-2612 `permit` via `ecRecover`) must also stop honoring deactivated keys without needing redeployment

Three draft EIPs approach revocability from two angles:

- [EIP-7851](https://eips.ethereum.org/EIPS/eip-7851) (Deactivate a Delegated EOA's Key) — system contract where a 7702-delegated EOA can schedule an irreversible deactivation of its ECDSA key, finalized after a 7-day cancellation window. Once finalized, block processing, the transaction pool, and future 7702 authorizations all reject signatures from the deactivated key.
- [EIP-8151](https://eips.ethereum.org/EIPS/eip-8151) (Private Key Deactivation Aware `ecRecover`) — modifies the `ecRecover` precompile to return 32 zero bytes for keys deactivated under EIP-7851, so immutable contracts that rely on `ecRecover` for authorization (most visibly ERC-2612 `permit`) stop honoring deactivated keys without having to be redeployed. Requires EIP-7851.
- [EIP-8164](https://github.com/prestwich/EIPs/blob/48a93b9a83d8cb3d78a983e4f9f60460651574ab/EIPS/eip-8164.md) (Native Key Delegation for EOAs) — extends EIP-7702's `0xef01XX` designator space with `0xef0101` for Ed25519 native keys and a new tx type `0x05`. Once installed, all ECDSA paths are protocol-rejected for the account; unlike EIP-7851 no prior 7702 delegation is required, so it reaches the codeless EOA population. The crafted-signature creation path produces accounts whose ECDSA key never existed, a cryptographic guarantee of rootlessness rather than a procedural one.

EIP-8164 is currently specced as a *parallel track* to EIP-8141: `0xef0101` accounts have no code execution, so they cannot run EIP-8141 default code or originate frame transactions. The EIP defers a future combined designator that would make the two composable. This is likely the last stage before full migration: all preceding infrastructure (PQ auth, key rotation, mempool protection) must be proven before the ECDSA fallback can be safely removed protocol-wide.

**Delivers**: permanent removal of ECDSA authority for 7702-delegated EOAs (EIP-7851/EIP-8151) and any EOA via Ed25519 migration (EIP-8164). **Does not deliver**: a composable EIP-8141 + EIP-8164 path, recovery for lost PQ keys, or protocol-wide ECDSA removal.

---

## 6. Native Post-Quantum Transactions

**Status**: [Strawmap](https://strawmap.org/) target (end of 2029)

With all building blocks in place, Ethereum ships native PQ transactions as a protocol feature. The strawmap envisions **leanXMSS** for signatures and **STARK-based compression** via **leanVM**:

- **Fork I** (~H1 2027): Validator PQ key registration
- **Fork J** (~H2 2027): Gas efficiency for PQ signatures
- **Fork L** (~H2 2028): State compression via ZK proofs
- **Later forks**: Full PQ consensus

In practice: frame transactions with PQ precompile VERIFY frames, restrictive-tier mempool validation, ephemeral key rotation, encrypted propagation, and revoked ECDSA keys. EIP-8141 provides the transaction format this composes into, without requiring a new tx type for each capability.

---

## 7. Long-Term: Private L1 Settlement

**Status**: Research direction

The PQ roadmap converges with **native privacy via shielded transfers**, another strawmap north star. A fully private L1 combines PQ authentication, encrypted propagation, shielded transfers (concealed amounts/parties), and ZK state proofs.

Proposals pointing in this direction: [EIP-8224](/eip-8224) (fflonk ZK proofs for shielded gas funding), the [expansive mempool tier](/mempool-strategy#expansive-mempool-what-develops-in-parallel) for privacy protocols, and the [merkle branch escape hatch](/mempool-strategy#the-merkle-branch-escape-hatch) for witness data. EIP-8141's frame architecture provides the protocol surface that PQ verification, encrypted propagation, and shielded transfers all build on.

The [three-gates analysis](/mempool-strategy#privacy-pools-three-gates) identifies what must change for shielded-pool withdrawals to reach a block through public infrastructure rather than a private relayer: canonical-pool code-hash exemption, raised VERIFY caps (~400k per tx, 1M per inclusion list), and validation-index FOCIL enforcement.

---

## Roadmap Summary

| Stage | Component | Status | Depends on |
|---|---|---|---|
| 1 | Account decoupling from ECDSA | EIP-8141 (Draft) | - |
| 2 | PQ precompiles and opcodes | Research (EIP-8051, EIP-8052) | Stage 1 |
| 3 | Ephemeral key rotation | Research ([ethresear.ch](https://ethresear.ch/t/achieving-quantum-safety-through-ephemeral-key-pairs-and-account-abstraction/24273)) | Stages 1, 2 |
| 4 | Encrypted mempools | EIP-8184 (Draft) | Stage 1 |
| 5 | Complete secp256k1 revocability | EIP-7851, EIP-8151, EIP-8164 (all Draft; EIP-8164 currently parallel to EIP-8141, not yet composable) | Stages 1-4 |
| 6 | Native PQ transactions | [Strawmap](https://strawmap.org/) target (end of 2029) | Stages 1-5 |
| 7 | Private L1 settlement | Research direction | Stages 1-6 |

**EIP-8141 is necessary but not sufficient.** It builds the foundational first step by decoupling account authentication from ECDSA and providing the generic transaction primitives that all subsequent PQ work builds on. A long sequence of protocol upgrades, precompiles, and eventually full ECDSA deprecation must follow before Ethereum is truly post-quantum safe. This roadmap is measured in years, not months.

---

## Read Next

- [Current Spec → EOA Default Code](/current-spec#eoa-default-code) — where Stage 1 actually lives, in spec terms.
- [Mempool Strategy](/mempool-strategy) — how encrypted mempools (Stage 4) interact with the two-tier design.
- [Competing Standards](/competing-standards) — which alternatives are also PQ-ready, and which are not.

---

*Sources: [EIP-8141 Spec](https://eips.ethereum.org/EIPS/eip-8141), [EIP-8184 (LUCID)](https://eips.ethereum.org/EIPS/eip-8184), [Ephemeral Key Pairs (ethresear.ch)](https://ethresear.ch/t/achieving-quantum-safety-through-ephemeral-key-pairs-and-account-abstraction/24273), [Ethereum L1 Strawmap](https://strawmap.org/), [Post-Quantum Ethereum](https://pq.ethereum.org/), [Google Q-Day Timeline](https://blog.google/innovation-and-ai/technology/safety-security/cryptography-migration-timeline/), [EIP-8051](https://eips.ethereum.org/EIPS/eip-8051), [EIP-8052](https://eips.ethereum.org/EIPS/eip-8052), [EIP-7851](https://eips.ethereum.org/EIPS/eip-7851), [EIP-8151](https://eips.ethereum.org/EIPS/eip-8151), [EIP-8164](https://github.com/prestwich/EIPs/blob/48a93b9a83d8cb3d78a983e4f9f60460651574ab/EIPS/eip-8164.md)*
