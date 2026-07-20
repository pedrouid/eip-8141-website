# Competing Standards

---

## TL;DR

Five general-purpose proposals sit between maximum generality (EIP-8141: arbitrary EVM in VERIFY frames) and maximum constraint (Tempo-like: fixed UX primitives, no programmable validation). EIP-8141 and EIP-8130 are the most active and both PQ-ready: EIP-8130 wins on mempool simplicity; EIP-8141 wins on expressiveness and EOA defaults. Static sponsorship, shielded gas funding, and PQ-safe address derivation are complementary, not replacements.

---

## Competing vs Complementary

"Competing" is not a single relationship. The proposals on this page fall into two groups:

- **Competing general-purpose proposals** (EIP-8175, EIP-8130, EIP-8202, Tempo-like): these overlap with EIP-8141's scope and trade generality vs. constraint, EVM validation vs. declared authenticators, and recursive frames vs. flat capabilities. A chain adopts one of these, not several.
- **Complementary proposals** (EIP-8223, EIP-8224, EIP-8215/HCA): these sit *off* the generality spectrum and compose with any general-purpose format. They cover static sponsorship, shielded gas funding, and PQ-safe address derivation.

Read the tables below with this distinction in mind: the first half of the page compares EIP-8141 against its direct competitors; the sections on EIP-8223 and EIP-8224 describe how narrower primitives layer on top rather than replace.

---

## Comparative Analysis

### The Fundamental Tradeoff: Generality vs. Deployability

The five general-purpose proposals sit on a spectrum (alternatives, you choose one):

```
More General                                                                More Constrained
    |                                                                             |
 EIP-8141       EIP-8175       EIP-8130       EIP-8202                      EIP-XXXX
 (arbitrary     (flat caps,    (authenticator  (flat extensions,              (fixed UX
  EVM in         programmable   contracts,     scheme-agile auth,            primitives,
  VERIFY         fee_auth)      no wallet      single execution)             passkeys)
  frames)                       code)
```

Complementary proposals sit off the spectrum (narrower scope, compose with any of the above):

```
                Static Sponsorship                       Shielded Gas Funding
                ──────────────────                       ────────────────────
                 EIP-8223                                 EIP-8224
                 (canonical payer registry                (fflonk ZK proof against
                  at 0x13, one SLOAD                       canonical fee-note
                  + balance check,                         contracts, ~222K gas,
                  no EVM in validation)                    bootstrap problem)
```

EIP-8215/HCA is another related proposal outside the transaction-format spectrum: it addresses PQ-safe address derivation for new accounts by deriving addresses from hash commitments to spending conditions.

**EIP-8141** gives maximum flexibility: any account code can define any validation logic, multiple execution frames per transaction, atomic batching. The cost is mempool complexity (banned opcodes, gas caps, validation prefixes) and async execution incompatibility.

**EIP-8175** provides flat capability composition with programmable gas sponsorship via fee_auth, but limits sender validation to fixed signature schemes. It started as "simpler than 8141" but evolved to include 4 new opcodes and EVM execution in the fee_auth prelude.

**EIP-8130** makes validation programmable but operationally predictable through authenticator contracts: signature logic is expressed inside an authenticator, while the canonical set and known gas bounds give nodes predictable cost. It is also designed as a full cross-chain account standard. The constraint is that authenticators implement a pure `authenticate(hash, data) -> actorId` interface; richer policies are handled through actor scope/policy and execution-time gates.

**EIP-8202** takes a different angle entirely; it doesn't try to be an AA system. Instead, it solves signature agility and feature composition at the transaction envelope level. One execution payload, flat typed extensions, no new opcodes. The cost is no batching, no programmable validation, and no gas sponsorship (yet).

**EIP-XXXX (Tempo-like)** takes the most constrained approach, bundling the specific UX features wallets need today (batching, sponsorship, passkeys, validity windows, 2D nonces) into a single tx type with no new opcodes and no programmable validation. The cost is a fixed feature set that requires hard forks to extend, and no PQ strategy.

### ECDSA Decoupling

| Proposal | Approach | Programmable Validation |
|---|---|---|
| **EIP-8141** | Arbitrary EVM in VERIFY frames. Account code defines its own sig scheme. No ECDSA dependency in the tx format. | Yes: any logic via account code + APPROVE |
| **EIP-8175** | Fixed signature scheme set (secp256k1, Ed25519). New schemes require hard fork. | No: sender auth is cryptographic only |
| **EIP-8130** | Declarative authenticator contracts. Any sig scheme expressible inside an authenticator (including PQ schemes). Nodes maintain a canonical authenticator set. Fully PQ-ready once a PQ authenticator is canonical. | Yes: within the `authenticate(hash, data) -> actorId` interface plus actor policy gates |
| **EIP-8202** | Typed `scheme_id` in authorization (secp256k1, P256, Falcon-512). New schemes added per EIP. | No: fixed cryptographic check per scheme |
| **EIP-XXXX** | Fixed set: secp256k1, P-256, WebAuthn. New schemes require hard fork. | No |
| **EIP-8223** | secp256k1 sender only. Not a decoupling proposal. | No |
| **EIP-8224** | fflonk ZK proof verification. Not a decoupling proposal. | No |

Both EIP-8141 and EIP-8130 are PQ-ready, through different mechanisms. EIP-8141 uses arbitrary account code plus `ARBITRARY` witnesses. EIP-8130 uses authenticator contracts added to a canonical set. The tradeoff is expressiveness vs. operational predictability: EIP-8141 can run any EVM logic, while EIP-8130 constrains validation to `authenticate(hash, data) -> actorId` for bounded cost.

On key rotation: EIP-8130 has native onchain key rotation via `owner_config` changes, portable across chains. EIP-8141 delegates key management entirely to account code, with no protocol-level rotation mechanism. EIP-8223 offers key rotation for sponsored accounts via `authorize(newEOA)`.

### Gas Sponsorship

| Proposal | Sponsorship Model | Canonical Paymaster |
|---|---|---|
| **EIP-8141** | VERIFY frame authorizes payer. Canonical paymaster recognized by runtime code match. Non-canonical limited to 1 pending tx. A codeless sender uses signature index `0`; a payment-only codeless EOA sponsor uses index `1`. | Yes: protocol-blessed, mempool-validated |
| **EIP-8175** | Programmable `fee_auth` contract with `RETURNETH` escrow. Sponsor state persists even if main tx reverts. | No: fee_auth is per-sponsor |
| **EIP-8130** | `payer` + `payer_auth` fields. Payer authenticated via same authenticator infrastructure as sender. | No: uses canonical authenticator set |
| **EIP-8202** | `ROLE_PAYER` reserved but not yet defined. No sponsorship today. | No |
| **EIP-XXXX** | `fee_payer_signature` field (secp256k1 only). Fee payer pays all gas. | No: direct co-signing |
| **EIP-8223** | Canonical payer registry at `0x13`. Static validation (1 SLOAD + balance check). `tx.to` pays gas. | Yes: predeploy registry |
| **EIP-8224** | Shielded gas funding via ZK proofs. One-shot bootstrap, then transition to EIP-8223. | No: fee-note contract |

### Transaction Batching

| Proposal | Batching Model | Atomicity Control |
|---|---|---|
| **EIP-8141** | Multiple frames per tx. SENDER frames execute sequentially with per-frame gas. | Flags field bit 2 on consecutive frames of any mode; validation-prefix batches are excluded from the restrictive mempool |
| **EIP-8175** | Typed capabilities list (CALL, CREATE). Sequential execution. | All-or-nothing: if any capability reverts, remaining are skipped |
| **EIP-8130** | Call phases (array of arrays). Completed phases persist if later phases revert. | Per-phase atomicity: calls within a phase are atomic |
| **EIP-8202** | Single execution payload. No native batching. | N/A: one call per tx (multicall wrappers needed) |
| **EIP-XXXX** | `calls` list of `[to, value, input]` tuples. | All-or-nothing: entire tx reverts on any call failure |
| **EIP-8223** | Not addressed. Single call per tx. | N/A |
| **EIP-8224** | Not addressed. Single call per tx. | N/A |

### Developer Tooling

| Proposal | Wallet Integration Cost | Infrastructure Required |
|---|---|---|
| **EIP-8141** | Implement new tx type. Default code covers EOAs with no smart account needed. | No bundlers, no relayers. Public mempool. |
| **EIP-8175** | Implement new tx type. Compose capabilities + signatures. | No bundlers. fee_auth contracts for sponsorship. |
| **EIP-8130** | Implement new tx type. Register actors via Account Configuration Contract. | Authenticator contracts. Canonical-set coordination. |
| **EIP-8202** | Implement new tx type. Existing secp256k1 EOAs keep their address. | Minimal: no new contracts. |
| **EIP-XXXX** | Implement new tx type. P-256/WebAuthn create new addresses. | Minimal: no new contracts. |
| **EIP-8223** | Implement new tx type. Payer calls `authorize(sender)` on registry. | Payer registry predeploy only. |
| **EIP-8224** | Implement new tx type. User deposits into fee-note contract, generates ZK proof. | Fee-note contracts + proof generation tooling. |

On EVM changes: EIP-8141 requires 6 new opcodes and a new frame execution model. EIP-8175 requires 4 new opcodes. EIP-8130 and all other proposals require zero EVM modifications, which simplifies client implementation and reduces the cross-client coordination burden. EIP-8130 achieves comparable programmability to EIP-8141 with a smaller protocol-change surface. EIP-8141's advantage is that default code gives EOAs AA features without smart account deployment or registration.

### Mempool Strategy

| Proposal | Validation Cost | Mempool Complexity |
|---|---|---|
| **EIP-8141** | EVM execution (capped at 100k gas) | High: validation prefix shapes, banned opcodes, canonical paymaster |
| **EIP-8175** | Crypto sig verification + fee_auth EVM prelude | Medium: stateless sigs, but fee_auth simulation needed |
| **EIP-8130** | STATICCALL to authenticator (or native impl) | Medium: canonical authenticator set, account lock optimization |
| **EIP-8202** | ecrecover / P256VERIFY / Falcon-512 verify (deterministic) | Low: purely cryptographic, no EVM |
| **EIP-XXXX** | ecrecover / P-256 / WebAuthn (bounded) | Low: deterministic crypto, bounded sig sizes |
| **EIP-8223** | ecrecover + 1 SLOAD at `0x13` + balance check | Minimal: static reads only, no EVM |
| **EIP-8224** | fflonk proof verify (~176K gas) + EXTCODEHASH check + fixed storage reads | Minimal: bounded crypto + static reads, no EVM |

On tracing: EIP-8141 requires full EVM tracing during validation (banned opcodes, storage access, prefix matching). EIP-8175 requires partial tracing for fee_auth. EIP-8130 avoids tracing for canonical authenticators because nodes can implement hot-path signature algorithms natively. Non-canonical authenticators remain usable in ordinary execution, not direct 8130 authentication.

### Other Capabilities

| Proposal | EOA Support | Account Creation | Async Execution | Cross-Chain |
|---|---|---|---|---|
| **EIP-8141** | Protocol-native default code (ECDSA secp256k1; P256 outer signatures require account code) | DEFAULT deploy frame through any trace-compatible factory | Incompatible | Not addressed |
| **EIP-8175** | secp256k1 native; Ed25519 creates new addresses | Not addressed | Partially (fee_auth needs EVM) | Not addressed |
| **EIP-8130** | Implicit EOA authorization, auto-delegation | CREATE2 via `account_changes` | Compatible | Yes: `chain_id = 0` replays |
| **EIP-8202** | secp256k1 EOAs keep address; P256/Falcon create new | Not addressed | Compatible | Not addressed |
| **EIP-XXXX** | secp256k1 native; P-256/WebAuthn create new addresses | Not addressed | Compatible | Not addressed |
| **EIP-8223** | EOA delegates via 7702, then authorizes sponsor | Not addressed | Compatible | Not addressed |
| **EIP-8224** | Fresh EOA bootstrapping via ZK proof | Not addressed | Compatible | Cross-chain via `chain_id` binding |

## Individual Proposals

Each proposal is covered in detail on its own page:

- [EIP-8175: Composable Transaction](/eip-8175)
- [EIP-8130: AA by Account Configuration](/eip-8130)
- [EIP-8202: Scheme-Agile Transactions](/eip-8202)
- [EIP-XXXX: Tempo-like Transactions](/eip-xxxx)
- [EIP-8223: Contract Payer Transaction](/eip-8223)
- [EIP-8224: Counterfactual Transaction](/eip-8224)
- [EIP-8215: Hash-Committed Account](https://github.com/ethereum/EIPs/pull/11480) (related PQ address derivation)


---

## Cross-Proposal Commentary

### What EIP-8175's Ecosystem Says About EIP-8141

In the [Frame Transactions vs. SchemedTransactions comparison thread](https://ethereum-magicians.org/t/frame-transactions-vs-schemedtransactions-for-post-quantum-ethereum/28056), Giulio2002 ([post #1](https://ethereum-magicians.org/t/frame-transactions-vs-schemedtransactions-for-post-quantum-ethereum/28056/1)) argues that frame transactions impose a "smart wallet tax" of ~30,000-48,000 gas for contract dispatch, storage reads, and EVM context, making PQ verification via VERIFY frames (~63,000 gas) roughly 2x more expensive than direct SchemedTransaction verification (~29,500 gas for Falcon). The deeper argument: every real signature scheme ends up needing a precompile anyway, so frames become "an unnecessary indirection."

EIP-8141 defenders counter that the gas figure conflates ERC-4337 EntryPoint overhead with frame overhead (ch4r10t33r, [post #17](https://ethereum-magicians.org/t/frame-transactions-vs-schemedtransactions-for-post-quantum-ethereum/28056/17)); that existing EOAs cannot use new crypto schemes under SchemedTransactions without changing addresses, which matt ([post #15](https://ethereum-magicians.org/t/frame-transactions-vs-schemedtransactions-for-post-quantum-ethereum/28056/15)) flagged as an adoption blocker; and that hybrid classical+PQ signing is trivial in VERIFY frames but impossible in flat schemes. EIP-8175 and EIP-8202 share overlapping authors and advocate a consistent "flat composition" position against EIP-8141.

### What EIP-8130's Author Says About EIP-8141

From the [Biconomy blog analysis](https://blog.biconomy.io/native-account-abstraction-state-of-art-and-pending-proposals-q1-26/):
> "Base's position: 'We can heavily optimize this and build out performant mempool/block builder implementations,' something they can't do with EIP-8141's arbitrary validation frames."

The EIP-8130 position: EIP-8141 loses on most operational metrics (higher gas cost for non-EOA validation, mempool tracing, no native key rotation, 6 new opcodes), while authenticator contracts deliver comparable programmability with native hot-path implementations, no tracing, and built-in account management.

At [AllWalletDevs #40](https://ethereum-magicians.org/t/allwalletdevs-40-july-15-2026/28858) on Jul 15, EIP-8130 and ERC-8286 were presented side by side. The meeting summary records Chris Hunter saying Base plans to launch EIP-8130 in its September fork. A companion metadata layer also appeared as [ERC-8340](https://github.com/ethereum/ERCs/pull/1883), an open draft defining deterministic CBOR for EIP-8130's opaque `metadata` field. These are implementation and tooling signals, not changes to the core 8130 validation model.

EIP-8141 supporters counter that EIP-8130 can be built atop EIP-8141 (authenticators are a subset of what VERIFY frames can do) but not vice versa; that default code gives EOAs immediate AA without registration; and that VERIFY frames enable stateful and multi-step validation the pure authenticator interface cannot express. The tension is structural: constrained pure-function validation for performance, or general EVM validation for expressiveness.

### What EIP-8202's Authors Say About EIP-8141

From the spec's motivation and security considerations:
> "This EIP is intentionally not a frame transaction system. [...] It intentionally forbids recursive transaction composition. That avoids drifting into frame-transaction validation complexity, where mempool validation may require richer execution-aware processing before inclusion."

EIP-8202 also explicitly acknowledges EIP-8141 as a potential extension: "An EIP-8141-style frame extension could attach nested execution frames to the same transaction envelope without requiring a new top-level transaction type." The positioning is complementary-but-skeptical: solve the envelope and signature agility problem first, add frames later if needed.

---

## Summary

The landscape splits along two axes:

**Generality vs. deployability**: EIP-8141 is the most general (arbitrary EVM in VERIFY frames); constraint rises through EIP-8175, EIP-8130, EIP-8202, and EIP-XXXX. More generality means richer validation at the cost of mempool complexity and async-execution tradeoffs.

**Scope**: EIP-8223, EIP-8224, and EIP-8215/HCA sit off the generality spectrum. They address sponsored gas, shielded gas funding, and PQ-safe address derivation, and compose with whatever general-purpose AA design ships.

Key practical splits:

- **PQ readiness**: EIP-8141 and EIP-8130 are fully PQ-ready through programmable validation/authentication. EIP-8215/HCA addresses a different PQ gap: new account addresses that are not derived from exposed public keys. See [PQ Roadmap](/pq-roadmap).
- **Mempool simplicity**: EIP-8130 and the envelope-only proposals avoid EVM tracing during validation. EIP-8141 requires it, bounded by the restrictive tier's rules.
- **EOA support**: EIP-8141's default code gives codeless EOAs the cheapest path. Other proposals require smart accounts, new addresses for new schemes, or delegation.
- **Key rotation**: EIP-8130 has it natively (onchain `owner_config`). EIP-8141 delegates to account code. EIP-8223 offers it for sponsored accounts.
- **Complementary stack**: EIP-8141 can compose with narrower primitives such as EIP-8223, EIP-8224, and EIP-8215/HCA rather than forcing every sponsorship, privacy, or PQ-address concern into the base transaction type.

---

## Read Next

- [Current Spec](/current-spec) — what EIP-8141 specifically does, for direct comparison with the alternatives above.
- [EIP-8130](/eip-8130) or [EIP-8175](/eip-8175) — the two most active competitors.
- [EIP-8223](/eip-8223) and [EIP-8224](/eip-8224) — the complementary proposals that layer on top of any general-purpose AA.
- [Developer Tooling](/developer-tooling) — the wallet-adoption angle on why general vs constrained matters.
