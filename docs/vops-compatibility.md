# VOPS Compatibility

---

## TL;DR

Frame transactions require more validation state than legacy transactions. Legacy validation is a single account-trie lookup (~3,000 gas); frame validation executes arbitrary sender code via VERIFY frames (up to ~100,000 gas), pulling in bytecode, storage slots, and helper libraries. This page covers how EIP-8141 interacts with [VOPS](https://ethresear.ch/t/a-pragmatic-path-towards-validity-only-partial-statelessness-vops/22236) (Validity-Only Partial Statelessness), what state nodes need to carry, and where the boundaries are.

## Terms You'll See On This Page

Three terms do most of the heavy lifting here. Plain-English versions first, links to the formal definitions after.

**Validation state** — the data a node has to read from its copy of the blockchain to decide whether a transaction is well-formed *before* including it in a block. For a legacy Ethereum transaction, that's three fields on the sender's account: balance (can they pay gas?), nonce (is this in order?), and code (is this an EOA or a contract?). That's it, one lookup. For a frame transaction, the `VERIFY` frame can run arbitrary account code, so validation state now includes whatever slots and contracts that code touches. The tension on this page is: how much more state does a node need to hold to safely validate 8141 transactions, and where does the line stop being reasonable?

**VOPS** — short for *Validity-Only Partial Statelessness*. A node design where the node does **not** carry the full ~280 GB Ethereum state. Instead it carries a small "validity slice" that is enough to check that incoming transactions are valid and well-formed, but not enough to execute them. Full execution is delegated to other nodes or ZK proofs. The baseline VOPS slice is nonce + balance per account, ~10 GB for ~400 M accounts. The question for EIP-8141 is how much bigger that slice has to get so VOPS nodes can validate frame transactions, without pushing VOPS back toward "full state." See the [original VOPS thread](https://ethresear.ch/t/a-pragmatic-path-towards-validity-only-partial-statelessness-vops/22236).

**FOCIL** — short for *Fork-Choice-enforced Inclusion Lists*, formalized in [EIP-7805](https://eips.ethereum.org/EIPS/eip-7805). A censorship-resistance mechanism: validators publish inclusion lists of transactions they believe the next block *must* contain, and the fork choice rule penalizes blocks that omit them. For FOCIL to work, the attesters who build those lists have to be able to validate the transactions they're listing. This is why FOCIL and VOPS are tightly coupled, and why a frame transaction that's too expensive for a minimal node to validate also can't be enforced via FOCIL.

**Why the combination matters.** A frame transaction has to pass three gates to be included in a block through public infrastructure: the public mempool admits it, FOCIL attesters can validate it, and VOPS nodes can hold the state needed to check it. When all three conditions are met, the transaction is censorship-resistant and cheaply propagatable. When one fails, the transaction falls back to private channels. The rest of this page is about which classes of frame transaction pass all three, and which don't.

**Acronyms recap**: **VOPS** (~10 GB baseline for ~400M accounts), **FOCIL** (Fork-Choice enforced Inclusion Lists, [EIP-7805](https://eips.ethereum.org/EIPS/eip-7805)).

---

## Status

| Topic | Status |
|---|---|
| Validation state requirements | Bounded: [restrictive tier](/mempool-strategy#restrictive-mempool-what-ships-first) caps at 100k gas, sender-only storage reads |
| Bytecode availability | Proposed: [VOPS+4 extension](/mempool-strategy#the-state-side-vops-4-slots) includes code |
| State growth at AA scale | Proposed: ~72 GB under VOPS+4; extra-VOPS reads pay [merkle branch cost](/mempool-strategy#the-merkle-branch-escape-hatch) |
| Frames + FOCIL + VOPS trilemma | Proposed: [two-tier mempool + VOPS+4 + witness escape hatch](/mempool-strategy#resolving-the-trilemma) |
| Witness-based FOCIL | Proposed: 4-8 kB today, 1-2 kB after binary tree migration |
| Privacy pool state reads | Partial: EIP-8272 makes declared recent roots pre-validated and introspectable, but nullifier slots remain hash-keyed in external pool storage outside VOPS+4. Routed via [canonical-pool exemption + validation-index FOCIL](/mempool-strategy#privacy-pools-three-gates) |
| Implementation complexity | Open: compounds with other protocol changes |

---

## Validation State Requirements

Legacy transactions require a single account trie lookup. Frame transactions execute arbitrary sender code via VERIFY, requiring bytecode, storage slots, and helper library code up to the 100,000-gas validation cap. Most nodes in a post-ZKEVM stateless world cannot validate these without carrying more state than the VOPS baseline provides.

The [restrictive mempool tier](/mempool-strategy#restrictive-mempool-what-ships-first) bounds this: validation is limited to `tx.sender` storage, capped at 100,000 gas, with banned opcodes preventing reads outside the sender's state. The [VOPS+4 extension](/mempool-strategy#the-state-side-vops-4-slots) (nonce, balance, code, first 4 storage slots per account) covers well-designed AA wallets within a small constant-factor increase over the VOPS baseline.

## State Growth at Scale

At N=4 storage slots per account, full AA adoption results in ~72 GB total VOPS size, an 8x increase from the ~10 GB floor. This is still well below the ~280 GB full state, but it is a significant regression from the VOPS baseline. The original AA-VOPS proposal did not address bytecode availability.

derekchiang proposed (ethresear.ch [post #12](https://ethresear.ch/t/frame-transactions-through-a-statelessness-lens/24538/12), Apr 15) adding all contract bytecodes to VOPS directly (~10.55 GB), roughly doubling the baseline but staying well below full state. Extra-VOPS use cases pay a [merkle branch cost](/mempool-strategy#the-merkle-branch-escape-hatch) per item: 4-8 kB today, 1-2 kB after binary tree migration.

## The Frames + FOCIL + VOPS Trilemma

A recurring observation from the [ethresear.ch thread](https://ethresear.ch/t/frame-transactions-through-a-statelessness-lens/24538): current designs cannot simultaneously deliver Frames/Native AA, Public Mempool/FOCIL, and Statelessness/VOPS. You can have at most two.

The EIP-8141 co-authors argue all three are achievable per-transaction-class. The restrictive mempool constrains validation to bounded state access. VOPS extends to nonce, balance, code, and 4 slots. Extra-VOPS use cases include merkle branches. Complex policies move to the [expansive tier](/mempool-strategy#expansive-mempool-what-develops-in-parallel). The full resolution framework is in [Mempool Strategy](/mempool-strategy#resolving-the-trilemma).

## Witness Costs for Extra-VOPS Reads

For storage slots outside the VOPS+4 extension, transactions include a witness proving the state items they read. Simple cases (alternative sig algorithms, key rotation) add zero extra cost since their validation reads fit within VOPS+4. EIP-8272 avoids mutable-storage reads for declared recent application roots by moving them into a pre-validated transaction field, with `RECENTROOTREFLOAD (0xb5)` for validation-time access. Privacy protocol withdrawals still need nullifier state from external storage and add ~4 kB per proven item under the witness approach.

The infrastructure already exists in clients (witness machinery is needed for sync), and binary tree migration further reduces the per-item cost. The framework accepts this as the explicit per-transaction cost of the escape hatch, limited to transactions that need it.

**Counterpoint**: Witnesses solve the on-chain verifiability piece but not the mempool-admission piece. EIP-8272's recent-root path removes one shared-state read and now specifies bounded expiry/reorg eviction (PR #11961), but privacy withdrawals still fail on proof gas, FOCIL per-list budgets, and nullifier availability. See [Mempool Strategy → Privacy Pools and the Three Gates](/mempool-strategy#privacy-pools-three-gates) for the canonical-pool exemption, raised VERIFY cap, and validation-index FOCIL proposals.

## Implementation Complexity

Frame transactions touch consensus, mempool policy, p2p propagation, client state management, and wallet infrastructure simultaneously. This has been cited as adding scope to the Glamsterdam timeline. The interactions between VOPS, FOCIL, witness proofs, and the two-tier mempool architecture create a combinatorial testing surface that compounds with each open question.

---

## Summary

The VOPS+4 extension plus merkle witnesses covers the majority of frame transaction validation within a bounded state footprint. Status-quo accounts and well-designed AA wallets fit entirely within VOPS+4 at zero extra cost. Privacy protocols and complex validation pay per-transaction witness costs. The [two-tier mempool architecture](/mempool-strategy) keeps FOCIL-enforcing nodes on the restrictive tier by default while enabling an opt-in expansive tier for use cases that exceed the bounded state model.

**What to watch**: whether the VOPS+4 extension gets formal adoption, and whether the witness cost after binary tree migration is low enough to make the escape hatch practical at scale.

---

*Source: [ethresear.ch - Frame Transactions Through a Statelessness Lens](https://ethresear.ch/t/frame-transactions-through-a-statelessness-lens/24538)*
