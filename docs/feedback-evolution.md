# How Feedback Evolved Over Time

---

The feedback on EIP-8141 arrived in distinct waves, each pushing the spec along a clear trajectory: **from expressive abstraction toward deployability, compatibility, and mempool safety**. The early drafts prioritized flexibility; the later drafts are about making that flexibility survivable for clients, wallets, and the p2p network.

---

## Phase 1: Conceptual & Compatibility Scrutiny (Jan 29 – Feb 10)

*In this phase, the thread tested whether the `APPROVE` mechanism and the frame abstraction held up under proxy-account, nested-call, and ecosystem-compatibility pressure. The core design question was: does this new opcode actually behave safely across the call-graph shapes real accounts use?*

### APPROVE Propagation Debate

*nlordell, frangio, fjl — EthMagicians posts #16-32*

The first major discussion was about how `APPROVE` interacts with nested call frames. Key questions:
- Does `APPROVE` auto-propagate through `RETURN`?
- Can inner contracts accidentally approve transactions?
- How do proxy-based smart accounts (like Safe) use `APPROVE` if it requires top-level invocation?

Resolution: The authors decided **against** auto-propagation. `APPROVE` is transaction-scoped and updates `sender_approved`/`payer_approved` directly. Only `frame.target` (checked via `ADDRESS == frame.target`) can call `APPROVE`. This was later relaxed (PR #11297) to allow `APPROVE` from nested calls, enabling proxy-based accounts to adopt 8141 more easily.

### "Why Not Simpler Alternatives?"

*Helkomine — posts #6-14*

Helkomine argued that command-oriented architectures like Uniswap's UniversalRouter could achieve the same functionality. Matt (lightclient) explained: frames exist for **protocol-level introspection**, allowing the p2p layer to reason about transaction validity bounds. The previous attempt (EIP-2938) had failed precisely because it lacked this structured introspection, making safe mempool relay rules impossible.

Key quote from matt (post #9):

> Frames are required to support introspection *by the protocol*. It's not about supporting multiple calls at the EVM layer. It's about allowing end users to flexibly define the way their transactions should be handled. The protocol can in turn, use the modes we're introducing to reason about the transaction and safely bound the resources needed to validate and propagate abstract transactions over p2p.

### EIP-3607 Compatibility

*thegaram33 — post #26, PR #11272*

Peter Garamvölgyi identified that EIP-3607 (which rejects transactions from senders with deployed code) would block 8141's `SENDER` frames for smart accounts. He submitted PR #11272 to disable this check for frame transactions. This PR remains open.

### APPROVE Security in Non-Account Contracts

*thegaram33 — posts #37-41*

thegaram33 raised the concern that any contract containing the `APPROVE` opcode could potentially be used as a frame target, creating an alternative authorization pathway. fjl responded that `APPROVE` is literally just `RETURN` with extra semantics, and execution approval only works when `frame.target == tx.sender`. This concern actually **reinforced** the design decision to restrict `APPROVE` to `frame.target`.

**What changed because of this phase**: `APPROVE` semantics locked down to transaction-scoped with an `ADDRESS == frame.target` check, then relaxed via PR #11297 to allow `APPROVE` from nested calls so proxy-based accounts (Safe-style) could adopt the spec. EIP-3607 compatibility opened as PR #11272.

---

## Phase 2: Adoption & UX Concerns (Feb – Mar 10)

*The thread pivoted from "does this work?" to "will anyone use it?" Monad's adoption data and Derek Chiang's commercial-AA experience reframed the question: if nearly all frame transactions will come from EOAs, the spec has to serve EOAs natively or lose to a simpler alternative.*

### EIP-7702 Adoption Data Battle

*DanielVF vs matt — posts #64-71*

DanielVF (from Monad) presented data showing only ~0.28% of transactions in 1000 blocks were EIP-7702. Matt countered that this metric was wrong: most 7702 usage goes through entrypoint contracts and "relayed actions" not direct transactions. BundleBear data showed 4M+ operations/week from 7702-enabled accounts. The disagreement highlighted a key insight: **measuring AA adoption requires looking at operations, not raw transaction counts**.

### The Adoption Critique

*DanielVF — post #64*

DanielVF made a comprehensive argument across three posts:
1. Smart contracts as asset owners is the "adoption killer": wallets are expensive/dangerous to build, users get locked into vendors, no interoperability
2. EIP-1559 succeeded because it was simple and standardized; 4337/7702 failed because they required each wallet to build custom, dangerous smart contracts
3. If 99%+ of frame txns will be from EOAs, why not just build a simpler tx type?

Key quote from DanielVF (post #64):

> What are the bytes that a wallet sends over a transaction to authorize moving 1 Eth from one address to another? EIP-8141 at this moment doesn't define this. The authorizing address contract code could have any code under the sun, so we don't know what the wallet should send.

This argument directly influenced the spec's evolution toward EOA support.

### Derek Chiang's Response and EOA Support

*derek — posts #60, #65*

Derek Chiang, who had 3 years of commercial AA experience, largely agreed with DanielVF's adoption concerns and proposed EOA support for 8141. His argument:
- EOAs can enjoy AA benefits (gas abstraction, sponsorship) without smart contract risks
- Wallets integrate by supporting a new tx type, not building/auditing smart accounts
- Works consistently across all EVM chains without contract deployments
- Still preserves the option for advanced users to use smart contract accounts

Key quote from derek (post #62):

> It flips the default from "your EOA can't use any AA features unless you delegate to a smart account" to "your EOA can use most AA features by default, and can do more if you delegate to a smart account."

This led to PR #11379 (merged Mar 10), a pivotal change.

**What changed because of this phase**: EOA default code added via PR #11379 (Mar 10). This was the single biggest shift in the spec's trajectory, moving it from "smart-account-assumed" to "EOA-first." Every downstream design decision (default VERIFY code, default SENDER code, paymaster-by-EOA) followed from here.

---

## Phase 3: Operational Constraints & Mempool Safety (Mar 10 – Mar 25)

*With EOA support landed, attention shifted to making frame transactions safely propagatable and performant at the p2p layer. The async-execution critique (from Monad), the atomic-batching debate, and the mempool policy dominated. These are all operational questions about whether clients and builders can actually ship this.*

### Async Execution Incompatibility

*DanielVF, pdobacz — posts #53, #79, #119-123*

The most structural criticism: frame transactions require EVM execution to determine transaction inclusion validity, which is fundamentally incompatible with async execution models (Monad, and potentially future Ethereum via EIP-7886). Traditional transactions only need 3 state reads (balance, nonce, code) for inclusion checks.

Impact assessment:
- Monad: frame txns are "mutually incompatible" with their async execution model
- Base: drafted EIP-8130 specifically as an alternative to EVM-driven inclusion
- Ethereum: constrains future performance optimizations

However, the authors argued this is manageable: FOCIL provides censorship resistance, mempool rules cap validation gas at 100k, and the `VERIFY` frame structure allows the protocol to bound validation costs.

### Atomic Batching Debate

*pedrouid, 0xrcinus, derekchiang, frangio — posts #72-89, PR #11395*

pedrouid argued SENDER frames should always be atomic. derekchiang explained why non-atomic is the better default:
- Non-atomicity is more general (OR logic) while atomicity is more specific (AND logic)
- A paymaster using VERIFY needs the guarantee that the ERC-20 transfer SENDER frame won't revert just because a subsequent user operation reverts
- Having top-level frames default to non-atomic while providing atomic opt-in is more elegant

Key quote from frangio (post #73, PR #11395 comment):

> The main use case I've heard for non-atomic frames is using ERC-20 for gas payment. The payer needs the guarantee that once it has accepted payment, the next frame will do an ERC-20 transfer and not revert.

0xrcinus proposed a GROUP-based approach (explicit group IDs). The final design used bit flags in the mode field: a consecutive run of SENDER frames with the atomic batch flag (bit 11) set forms an atomic batch.

### P256 Scope Creep

*shemnon, frangio — posts #78, #94-99*

Danno Ferrin flagged the addition of P256 signature support in the default code as "significant scope creep." frangio noted P256 accounts (via passkeys) don't support key rotation, undermining one of AA's motivations. The authors decided to keep P256 in the default code but acknowledged the tradeoff: it provides immediate utility for passkey users but creates accounts that can't be migrated to PQ-secure schemes without additional EIPs.

Key quote from shemnon (post #99):

> P256 on its own is a fine discussion to have, I am concerned that 8141 is becoming a kitchen sink EIP that will fall apart once one interest doesn't get what they want.

### PQ Signature Aggregation Path

*fjl — post #23*

A less discussed but strategically important point: fjl noted that the VERIFY frame design deliberately enables future **signature aggregation**. Because VERIFY frames cannot change execution outcomes and their data is elided from the signature hash, a block builder could theoretically strip all VERIFY frames from transactions and replace them with a single succinct proof of validity. This is a key part of the long-term PQ strategy: PQ signatures are large, and aggregation could dramatically reduce their on-chain cost. The precise mechanism isn't worked out yet, but the frame architecture was designed to keep this path open.

### Mempool Rules as Turning Point

*lightclient — post #112, PR #11415*

lightclient's mempool policy PR was a turning point. DanielVF called it "a big big step forwards" (post #114). Key innovations:
- **Canonical paymaster**: removes complex reputation/staking from ERC-7562, replacing it with a standardized paymaster contract that nodes verify by code match
- **Validation prefix**: only the frames up to `payer_approved = true` are subject to mempool rules
- **Capped validation gas**: MAX_VERIFY_GAS = 100,000
- **Structural templates**: four recognized validation prefixes for public mempool acceptance

Alternative and competing proposals (EIP-8175, EIP-8130, Tempo) emerged around this period; they are documented in [Competing Standards](/competing-standards) and their per-EIP pages rather than as timeline entries here.

**What changed because of this phase**: atomic-batch flag added (PR #11395, Mar 25). Comprehensive mempool policy merged (PR #11415, Mar 25) with canonical paymaster, validation-prefix templates, banned opcodes, and MAX_VERIFY_GAS = 100k. These turned EIP-8141 from a spec into something clients could begin implementing.

---

## Phase 4: Forward-Compatibility Extensions & Open Gaps (Mar 26 – Apr 10)

*With the core spec landed, the thread moved into extension territory: forward-compat hooks for PQ signature aggregation and precompile-based verification, plus a steady stream of open gaps (EIP-7702 interaction, EIP-3607, signature-index discovery, frame-returndata) that needed decisions before the spec could be called complete.*

### VALUE in SENDER Frames

*rmeissner, DanielVF, frangio, 0xrcinus, derek, matt — posts #124-134*

rmeissner (Safe team) identified a gap: SENDER frames have no `value` property, preventing native ETH transfers without custom execute methods in smart accounts. This sparked strong consensus among participants.

DanielVF argued that without value, frames become "dumb message pipes" requiring full-featured smart contract wallets for basic operations. frangio agreed, noting it enables accounts to focus purely on verification. 0xrcinus called the value-in-frame approach "more intuitive."

Two approaches were proposed:
- **Option A**: Add a `value` field to frames + use the atomic flag for safety (preferred by Safe/rmeissner, 0xrcinus, and majority)
- **Option B**: Allow SENDER frames to `DELEGATECALL` a precompile implementing execute functionality

Key quote from rmeissner (post #130):

> Safe's preference is for removing DELEGATECALLs entirely, favoring value field with atomic flag instead. Formal verification is easier and stronger security guarantees.

Key quote from DanielVF (post #132):

> Value in SENDER frames enables clean separation between what the transaction does and authorization/payment. Removing value limits frames to "dumb message pipes," harming adoption and requiring full-featured smart contract wallets.

matt (post #134) confirmed the authors support including value in frames now that atomic batching exists, noting the previous hesitation was that value only works in SENDER frames, creating field mismatches. This is a strong signal that a value field will be added.

### Signature Aggregation Forward-Compatibility

*lightclient — PR #11481, April 2*

lightclient proposed adding a `signatures` field to the outer transaction object. The motivation is PQ forward-compatibility: PQ signatures are large, and aggregation will be critical as users migrate. The proposed design:

- A new `signatures` list in the outer transaction, each containing the signature, algorithm metadata, message, and signer
- Signatures are verified *before* frame execution, so frames can assume validity and just check authority
- In the future, block-level aggregated witnesses could elide individual signatures entirely

From lightclient's PR description:

> Any important goal of 8141 is to be forward compatible with signature aggregation techniques, especially with respect to PQ signatures. As those signatures are quite large, aggregating them may become very important as many users begin migrating.

This is the most structurally significant open proposal; it would change the transaction format itself.

### Precompile-Based Verification

*derekchiang — PR #11482, April 2*

derek proposed allowing VERIFY frames to target precompiles directly. This extends the default-code verification logic to contract accounts, enabling:

- Contract accounts to use precompiles for verification while having code for other purposes
- Key rotation (the precompile reads the public key commitment from storage)
- Shared verification logic between EOAs and contract accounts

This PR is still being worked on but represents a meaningful expansion of the verification model.

### EOA + EIP-7702 Delegation Compatibility

*DanielVF — posts #120, #122*

DanielVF identified that accounts with EIP-7702 delegated code cannot use signature-based authorization with frame transactions. When an EOA has delegated its code to a smart contract, the default code path isn't invoked, but the delegated contract may not implement `APPROVE`. This is a gap that needs addressing.

### Async Execution Compatibility (Continued)

*DanielVF, derek — posts #121-123*

The async execution thread continued. DanielVF pointed derek to ethresearch threads and EIP-7886, mentioning an upcoming EthCC talk. derek asked for more resources to ensure frame transactions remain compatible with Ethereum's potential move toward async execution.

### Spec Consistency Fixes

*node.cm, chiranjeev13 — posts #135-136, PR #11488*

node.cm (new participant) identified that VERIFY frames are implicitly capped at 2 per transaction, since only two approval flags exist (`sender_approved` and `payer_approved`) and each can only be set once. They recommended making this explicit in the Constraints section.

chiranjeev13 followed up with PR #11488 fixing multiple spec inconsistencies:
- Add static VERIFY frame count check (`<= 2`) to constraints
- Fix stale APPROVE scope values in the structural rules
- Remove `frame.target != tx.sender` check from default VERIFY code to allow any EOA as paymaster

### EIP-3607 Compatibility Status Update

*lightclient — PR #11272, Apr 8*

lightclient's earlier review on the EIP-3607 compatibility PR (#11272, disabling the EIP-3607 check for frame transactions) was dismissed on Apr 8. The PR remains open without resolution. The interaction between EIP-3607's sender-has-code rejection and frame transactions for smart accounts is still an unresolved design question.

### Signature Index Discovery Problem

*derekchiang — PR #11481 comment, Apr 9*

derekchiang raised a practical concern with lightclient's signatures list proposal (PR #11481): smart contracts leveraging outer signatures have no way to know which index their signature occupies in the list. Since a transaction may have any number of signatures in arbitrary order, a contract can't hardcode an index. The updated default code has to loop through the entire signature list to find the relevant entry, an ergonomic and gas-efficiency weakness that may need addressing before the proposal can be finalized.

### Frame Return Data Access

*jacopo-eth — post #137, Apr 10*

Jacopo raised that access to frame returndata would enable using it as input in multi-step flows without requiring wrapper contracts (similar to the motivation behind ERC-8211). He proposed native support via `FRAMERETURNDATASIZE` and `FRAMERETURNDATACOPY` opcodes, with per-byte gas cost and a cap per frame. No author response yet.

**What changed because of this phase**: community consensus built around adding a per-frame `value` field (posts #124-134). Signature-aggregation (PR #11481) and precompile-VERIFY (PR #11482) proposals entered review, both with all-reviewer approval pending merge. Spec-consistency PR #11488 opened. No merges landed yet. This phase set up the pivot that followed.

---

## Phase 5: Sibling EIPs and Broad Spec Tightening (Apr 11 – Apr 15)

*Ben Adams produced three PRs in five days: two narrower sibling EIPs (EIP-8223 for static sponsorship, EIP-8224 for shielded gas funding) and a 295-line tightening PR that restructured the spec's internal consistency. The character of the thread shifted from "proposals and debates" to "structural changes landing."*

### Contract Payer Transaction (EIP-8223)

*benaadams — PR #11509, Apr 11*

In parallel with the tightening PR, Ben Adams submitted EIP-8223 (PR #11509), a narrower sponsored-transaction proposal where gas fees are charged to `tx.to` via a canonical payer-registry predeploy at `0x13`. Validation requires one SLOAD and one balance check with no EVM execution, making it FOCIL/VOPS-compatible. The PR description explicitly positions EIP-8223 as complementary to EIP-8141 and EIP-8175 rather than competing: EIP-8223 covers the narrow case where static validation is sufficient, while frame-based proposals handle the general case. The registry mechanism could also be expressed as a capability or frame mode within those formats.

### Counterfactual Transaction (EIP-8224)

*benaadams — PR #11518, Apr 12*

A day later, Ben Adams submitted EIP-8224 (PR #11518), addressing the bootstrap problem that remains even after EIP-8223: a fresh EOA with no ETH cannot pay gas privately, because receiving ETH from any source creates a traceable on-chain link. EIP-8224 introduces a new transaction type (`0x08`) carrying an fflonk ZK proof (over BN254) that the sender owns an unspent fee note in a canonical fee-note contract (recognized by code hash). Validation is bounded cryptographic computation plus fixed storage reads, no EVM execution, ~222K gas typical. The intended composition: one-shot bootstrap via EIP-8224 to fund a smart account, then transition to cheap sponsored transactions via EIP-8223. Together with EIP-8141, the three proposals form a layered stack for general-purpose AA, static sponsorship, and shielded gas funding.

### Validation Frame Ordering Within a Block

*fvictorio — post #138, Apr 13*

Franco Victorio asked whether validation frames of frame transactions are executed first within a block, before non-frame transactions and non-validation frames, drawing an analogy to how ERC-4337 separates validation and execution phases. The question is about block-level scheduling, not transaction-level ordering, and remains unanswered in the thread.

### Broad Spec Tightening (Merged)

*benaadams — PR #11521, submitted Apr 13, merged Apr 14*

Ben Adams submitted a 295-line spec tightening PR consolidating several open threads: splitting `mode`/`flags`, introducing `FRAMEPARAM` and `resolved_target`, hardening default secp256k1/P256 paths (low-`s`, P256 domain separation), reducing `MAX_FRAMES` from 1000 to 64, adding per-frame gas costs, locking deterministic deployment to EIP-7997, and strengthening security warnings around VERIFY-data malleability and DELEGATECALL + APPROVE. lightclient and derekchiang both approved. fjl questioned lowering MAX_FRAMES, but benaadams argued that journaling carries across frames (up to 2000 effective call depth) and it is easier to increase later after measurement. This is the broadest restructuring since PR #11401 (approval bits) and the first to add a fifth opcode (`FRAMEPARAM`) to the spec.

### Bytecodes in VOPS Proposal

*derekchiang — ethresear.ch post #12, Apr 15*

derekchiang proposed adding contract bytecodes to the VOPS baseline, noting that total contract bytecode is ~10.55 GB (as of 2024). Including it roughly doubles the VOPS size but stays well below the ~280 GB full state. This directly addresses the bytecode availability gap identified in [VOPS state growth](/vops-compatibility#state-growth-at-scale) and the [AA-VOPS discussion](https://ethresear.ch/t/a-pragmatic-path-towards-validity-only-partial-statelessness-vops/22236): the original VOPS proposal bounded storage reads to N slots per account but did not address how AA-VOPS nodes obtain delegate bytecodes. Including bytecodes in the baseline resolves this without new opcodes or rent mechanisms.

**What changed because of this phase**: PR #11521 merged Apr 14, bringing the mode/flags split, the `FRAMEPARAM` opcode (fifth opcode in the spec), `MAX_FRAMES` reduced to 64, per-frame gas cost added, and default-code hardening for secp256k1 and P256. EIP-8223 (PR #11509) and EIP-8224 (PR #11518) submitted as complementary sibling proposals rather than competitors. Bytecodes-in-VOPS reframing surfaced on ethresear.ch.

---

## Phase 6: Value Field and Privacy/Delegation Threads (Apr 16 – Apr 19)

*The pending `value` field consensus landed and external analysis (Nero_eth's three-gates post) began shaping the next wave of discussion around privacy-pool flows. Forum debate reopened the default-code-vs-7702 delegation interaction. The Hegotá CFI inclusion PR also opened in this window but did not merge until Apr 30; that landing is recorded in Phase 7.*

### Per-Frame Value (Merged)

*lightclient — PR #11534, submitted and merged Apr 16*

lightclient merged the long-requested `value` field two days after PR #11521 landed, resolving the consensus that had built up across posts #124-134 (rmeissner, DanielVF, frangio, 0xrcinus, derek, matt). The PR description captures the reversal plainly: the authors originally resisted a frame-level `value` because user operations were expected to be handled by the account itself, but with frames now aimed at delivering a good out-of-the-box experience without wallet-side batching, native `value` became "a critical field for the SENDER frame." Non-zero `value` is restricted to `SENDER` frames so that `VERIFY` remains `STATICCALL`-like and `DEFAULT` does not require `ENTRY_POINT` to fund transfers. A subtle default-code consequence: when a SENDER frame's `resolved_target != tx.sender`, the default code now returns successfully with empty data (matching an empty-code account) rather than reverting, because the top-level `value` transfer has already been applied by the frame call. The PR was auto-merged after all reviewers approved; no debate on the merged diff.

### Three Gates to Privacy

*Nero_eth — [ethresear.ch post](https://ethresear.ch/t/frame-transactions-and-the-three-gates-to-privacy/24666), Apr 16*

Nero_eth framed privacy-pool inclusion as a three-gate problem: public mempool (100k VERIFY cap rejects Groth16 ~250k gas), FOCIL enforcement (250k per-IL budget fits roughly two frame transactions), and VOPS/AA-VOPS node validation (nullifier slots are hash-keyed in an external pool contract, outside the VOPS+4 window). The useful observation for EIP-8141: frames structurally remove relayer trust in privacy flows, because invalid or replayed proofs revert in VERIFY before gas is charged, so a sponsor can be paid from the withdrawn amount itself with zero trust assumption. Five protocol changes proposed: canonical-pool code-hash exemption, ~400k per-tx VERIFY cap for canonical frames, validation-index FOCIL enforcement `(tx_hash, claimed_index)`, `MAX_VERIFY_GAS_PER_INCLUSION_LIST = 2^20`, and relaxed state access for canonical pools. Acknowledged tradeoff: attesters absorb up to ~28% block gas in the worst case under the validation-index model. The proposals are routed into the research repo under [Mempool Strategy → Privacy Pools and the Three Gates](/mempool-strategy#privacy-pools-three-gates).

### Value Field Announced on Forum

*derek (post #139, Apr 17), DanielVF (post #140, Apr 17)*

derek announced the value-field merge on the forum, linking the commit. DanielVF welcomed it and named two remaining priorities he wants addressed before the proposal is production-ready: (1) the default signature-handling path should become an explicit opt-in rather than a protocol default, to future-proof transactions against sig-scheme changes, and (2) atomic batching of frames needs to become practically usable, not just structurally defined. These are framed as the last blockers from the wallet/adoption side before he considers the spec complete for deployment.

### Default Code vs 7702 Delegation Interaction

*DanielVF, derek, alex-forshtat-tbk — posts #141-145, Apr 17-19*

DanielVF raised that a 7702-delegated EOA today can choose per-transaction whether to invoke its delegation or send a "regular transaction" that bypasses the delegation code, because the transaction-type envelope encodes that intent explicitly. Under the current EIP-8141 rule ("if there's code, use the code, otherwise use the default code"), a 7702-delegated EOA loses that choice: the delegation code always wins. DanielVF argued for restoring explicit opt-in via a flag byte in `frame.data` selecting between default-code paths and the 7702 delegation. derek agreed the inconsistency is interesting but questioned the implementation. alex-forshtat-tbk observed that the existing `signature_type` first byte already acts as an EOA-scoped flag (`0x0` for ecrecover, `0x1` for P256VERIFY) and proposed extending it with `0x2` meaning "use 7702 code." No PR yet; sits on top of the 7702-delegation + default-code gap already flagged in the current-spec Related Proposals table.

**What changed because of this phase**: per-frame `value` merged (PR #11534, Apr 16). EIP-8141 formally submitted to the Hegotá CFI list via PR #11537 on Apr 17 (merged Apr 30, recorded in Phase 7). The next wave of discussion opened with Nero_eth's three-gates analysis, focused on how frame transactions interact with privacy pools under FOCIL and VOPS constraints.

---

## Phase 7: Mempool Risk-Shifting and Svalbard Ratification (Apr 22 – Apr 27)

*Phase 7 opens with a same-day pairing on Apr 22: a small security cleanup aligning the sighash with EIP-2718, and the first of two mempool-policy proposals from derekchiang that reframe long-standing restrictions. The guarantors PR proposes an economic-risk workaround (a payer that commits to paying gas even if sender validation fails) that lets mempool nodes skip sender simulation entirely, which would open ERC-20 gas repayment with trustless onchain verification to public propagation without the VOPS/statelessness tradeoffs that motivated the original restriction. Five days later, the Svalbard interop hosts a native-AA breakout that ratifies the constraint set EIP-8141 has been converging toward: no mandatory relayers, statelessness and ZK-EVM compatibility, public-mempool admissibility, FOCIL compatibility, and ETH-denominated sponsorship as the core case (ERC-20 explicitly deferred to relayer-dependent paths). Phase 7's character is design-level: an early proposal for shifting mempool risk onto a payer, and a core-dev consensus event that puts the constraints under which proposals like guarantors must hold up on the record. Net spec impact in Phase 7 itself: the sighash type-byte fix. The follow-on cleanup, the factory relaxation, and the Hegotá CFI landing all fall in the next phase.*

### Transaction-Type Sighash Fix (Merged)

*derekchiang — PR #11544, submitted Apr 18, merged Apr 22*

derekchiang opened a 1-line PR fixing a cross-type signature replay weakness in `compute_sig_hash`: the existing `keccak(rlp(tx_copy))` omits the `FRAME_TX_TYPE` byte that EIP-2718 typed transactions conventionally prefix before RLP. The fix is a direct `keccak(bytes([FRAME_TX_TYPE]) + rlp(tx_copy))`. Approved by all reviewers within hours; auto-merged on Apr 22 with no further debate. A small but security-relevant alignment with the EIP-2718 convention that other typed transactions already follow.

### Guarantors Proposal

*derekchiang — PR #11555, Apr 22*

derekchiang opened an early proposal introducing a "guarantor" payer: a payer that covers gas even if sender validation fails. When a transaction has a guarantor, mempool nodes may skip sender-validation simulation entirely, so the sender's VERIFY frame is free to read shared state (ERC-20 balances, environmental opcodes, anything the restrictive-tier rules currently forbid) and still propagate through the public mempool. If inclusion reveals that sender validation would have failed, the guarantor absorbs the gas cost. Guarantors are expected to be either accounts the user controls (self-guarantee) or third parties with a commercial or trust relationship to the user. The PR description explicitly frames the proposal as still iterating. The mempool-strategy impact is substantial: this would open a third path for ERC-20 gas repayment alongside the live (offchain) and permissionless (onchain) paymaster patterns, by moving the shared-state read problem from a mempool-policy violation into an economic-risk problem the guarantor absorbs.

Why this matters for statelessness: the original reason the restrictive tier bans shared-state reads during validation is VOPS compatibility. A node carrying only a partial-statelessness slice cannot safely validate a transaction whose inclusion depends on state outside its slice. Guarantors route around that by making the guarantor's commitment (which *is* inside every node's slice, since it is a paymaster-like signature check) sufficient grounds to admit the transaction. The VOPS invariant is preserved; the economic risk shifts from the protocol to the guarantor.

**What to watch**: whether guarantors gather author consensus beyond derekchiang; how the economic model handles third-party guarantor markets (AMM-backed, staked, or something else); whether the design generalizes beyond ERC-20 to privacy flows (nullifier reads, shielded withdrawals) and complex AA validation; and the interaction with VOPS/FOCIL. Guarantor-backed transactions sidestep sender simulation, but the guarantor's own validation still has to fit the restrictive tier for the mempool to admit the transaction at all.

### Svalbard Interop AA Breakout

*Hosts: Matt and Felix — [breakout notes](https://hackmd.io/@nixorokish/svalbard-aa-breakout), Apr 27*

A native-AA breakout at the Svalbard interop produced a constraint-and-goals framework that closely tracks EIP-8141's current shape. The session was attended by core contributors including Vitalik and Potuz alongside the wallet-side hosts, and the resulting list reads as external ratification of the design line EIP-8141 has been on, rather than a proposal for new directions.

**Primary goals**: alternative signature schemes (multi-sig, post-quantum, R1) with the open question of EVM-writable validation versus precompile-only; protocol-level call batching, motivated by wallet demand for top-level batching; key management and account recovery (multiple keys per account plus recovery mechanisms); post-transaction assertions for user-declared outcomes; and **no relayers for core functionality**, with users submitting through the L1 mempool without a mandatory relayer.

**Stretch goals**: unified default-account treatment of EOAs and smart accounts via a minimal default feature set; ETH-denominated gas sponsorship (ERC-20 sponsoring **explicitly deferred** to relayer-dependent paths); flexible nonces for non-sequential ordering use cases such as privacy pools; and a keystore wallet for cross-chain account sync.

**Hard constraints**: walk-away test (minimal future protocol churn); statelessness compatibility; ZK-EVM compatibility, with hash-based PQ signatures supported up front; public-mempool admissibility; FOCIL compatibility.

**Technical threads**:
- *Post-quantum*. Consensus to start with hash-based signatures. Lattice schemes need more work. Vitalik sketched a vector-math precompile for asymptotic gas-efficiency improvements.
- *Validation gas*. Separating a validation phase from execution gives the protocol the bounded-cost guarantees it needs while keeping validation logic flexible in the EVM, the same split EIP-8141 already encodes via APPROVE and VERIFY scopes.
- *L2 DoS vector*. Potuz raised sequencer DoS via invalid transactions with high computation cost. Mitigations discussed: validation-gas bounds and multi-dimensional gas accounting.
- *Privacy pools*. Promoted to primary goals with a witness-based validation approach. The canonical privacy-pool handler remains undefined, the same gap Nero_eth's [Three Gates to Privacy](/mempool-strategy#privacy-pools-three-gates) post named on Apr 16.
- *ERC-20 sponsorship*. Deferred to relayer-dependent implementations because of statelessness and mempool constraints. Core protocol focus stays on ETH-denominated gas abstraction without relayers, which lines up with EIP-8141's [two paymaster patterns](/mempool-strategy#erc20-paymaster-patterns) and the canonical-paymaster-is-ETH-only stance.

**Why this matters for EIP-8141**: the breakout's primary-goal list (alternative sigs, batching, key management, no-relayer-core, post-tx assertions) maps cleanly onto features EIP-8141 already specifies or has open PRs for. The hard constraints (statelessness, ZK-EVM, public mempool, FOCIL, walk-away test) are the same constraints driving the restrictive-tier mempool policy and the guarantors/payer-before-sender debates straddling this phase boundary. The unresolved items (canonical privacy-pool handler, validation-gas bounds for L2 sequencer DoS, EVM-writable versus precompile-only validation) are open in EIP-8141 too. Phase 9's keyed-nonces work, opened three days later, falls directly under the breakout's "flexible nonces" stretch goal.

**Action items from the session**: transcribe and publicize the constraint/goal framework; specify a canonical privacy-pool handler; continue work on PQ signature aggregation; evaluate competing EIPs against the framework. The competing-standards evaluation, in particular, becomes the lens for reading the alternatives tracked in this repo (EIP-8130, EIP-8175, EIP-8202, EIP-8223, EIP-8224, the Tempo-like draft).

---

## Phase 8: Payer Ordering, Default Code Cleanup, and Hegotá CFI (Apr 28 – Apr 30)

*Phase 8 is the implementation tail of Phase 7's design debate. lightclient floats a simpler framing of the guarantors idea: rather than a new role, just allow the payer to approve before the sender. The PR is briefly auto-merged by mistake, reverted the next day, and reopened as a draft. In parallel, a small default-code cleanup completes the multi-call transition by removing the legacy RLP call batch from the default `SENDER` mode, since the per-frame `value` and atomic-batch flag have superseded it. The phase closes on Apr 30 with two same-day landings: derekchiang's deploy-frame factory relaxation merges (dropping EIP-7997 from `requires` and rewriting the rule as a stateless-trace policy), and dionysuzx's long-pending Hegotá meta-EIP PR formalizes EIP-8141's CFI status in EIP-8081. Net spec impact across the phase: RLP call batch removed from default code, the factory-relaxation rewrite, and CFI status in the Hegotá fork meta. The guarantors framing remains open as a draft (#11555), paired with lightclient's payer-before-sender alternative (#11580); the choice between them is the open question heading into Phase 9.*

### Payer-Before-Sender Alternative to Guarantors

*lightclient — PRs #11575, #11579, #11580, Apr 28-29*

In parallel with derekchiang's still-iterating guarantors PR (#11555), lightclient floated a simpler framing: rather than introduce a new "guarantor" role, just relax the ordering rule so the payer can call `APPROVE_PAYMENT` before the sender approves execution. A payer that commits to gas before sender validation absorbs the same economic risk a guarantor would, without a new role. PR description: *"I think it is simpler to just allow the payer to approve before the sender instead of adding the full guarantor role."*

PR #11575 auto-merged on Apr 28; lightclient had intended a draft, and opened #11579 the next day reverting. The content is now open as draft #11580. Net spec impact: zero. The next-sync question is which framing (#11555 guarantors or #11580 ordering relaxation) gathers author consensus; both attack the same problem of admitting shared-state sender validation in the public mempool by shifting risk onto a payer.

### RLP Call Batch Removed from Default Code (Merged)

*lightclient — PR #11577, merged Apr 29*

A small cleanup completing the transition of multi-call out of the default-code payload and into the frame list. Default-code `SENDER` mode previously decoded `frame.data` as RLP `[[target, value, data], ...]` and returned successfully on cross-EOA targets; now it simply reverts. The use cases are covered by atomic frame batching (PR #11395) and per-frame `value` (PR #11534). Auto-merged with no debate.

### Deploy-Frame Factory Relaxation (Merged)

*derekchiang — PR #11567, opened Apr 24, merged Apr 30*

derekchiang's second structural mempool proposal merged six days after opening: drop the hard-coded EIP-7997 deterministic factory requirement for `deploy` frames. The pre-merge spec pinned EIP-7997 both as a `requires` entry and as the only valid `frame.target` a mempool node would propagate a deploy frame to. The merge removes both. Any contract can be the factory target, provided the deploy frame's execution still satisfies the validation trace rules. The mempool write policy is rewritten as an explicit carve-out: `CREATE`, `CREATE2`, or `SETDELEGATE` operations that install code at `tx.sender`, plus `SSTORE`s to `tx.sender`'s storage. `CREATE` (0xF0) and `SETDELEGATE` (0xF6, EIP-7819) join `CREATE2` (0xF5) in the deploy-frame opcode carve-out, and installed code may be either conventional contract code or an EIP-7702 delegation indicator.

Why this matters for the mempool model: EIP-7997 was a convenience dependency, not a safety dependency. The actual safety property the restrictive tier needs is that deploy-frame outcome is independent of chain state outside `tx.sender`. PR #11567 reifies that property directly in the trace rules rather than encoding it as "must call a specific contract." Any stateless factory qualifies. The PR also blurs the line between smart-account deployment and EIP-7702 delegation installation: both now flow through the same deploy-frame primitive, with the mempool treating delegation-indicator installation as a legitimate deployment outcome. Net spec impact: 11 added, 8 removed lines. lightclient approved with a one-word "SGTM"; auto-merged the same day.

This is the broadest mempool-policy change since PR #11415 (Mar 25 mempool policy) and the first PR to retract a `requires` entry. Open follow-up questions: how the new policy interacts with PR #11482's precompile-targeting VERIFY frames, and how it composes with the default-code-vs-7702 thread from Phase 6 now that delegation installation flows through the same primitive.

### Hegotá CFI Inclusion (Merged)

*dionysuzx — PR #11537, opened Apr 17, merged Apr 30*

The fork-meta PR that had been waiting on a final reviewer since Phase 6 merged on Apr 30 after ralexstokes approved. The diff is 5 added lines in `EIPS/eip-8081.md` only: EIP-8141 added to `Considered for Inclusion`, EIP-7716 and EIP-8205 added to `Proposed for Inclusion`. Decisions had been captured at ACDE #233 (t=5871) and ACDC #177 (t=3532, t=3853) two weeks earlier. The merge formalizes EIP-8141's CFI status for the Hegotá fork in the meta EIP record. Movement to PFI/SFI on a future ACD call requires further client-readiness signals, not another spec PR.

---

## Phase 9: Keyed Nonces and Parallel Sequences (Apr 30 – May 5)

*A new design line opens with two co-evolving proposals lifting EIP-8141's single linear sender nonce into a `(nonce_key, nonce_seq)` pair so a single sender can run independent sequences in parallel. The motivating use cases are privacy protocols sharing one onchain sender across many users, smart-wallet session keys, and relayer-style senders that today bottleneck on the linear nonce. Phase 9 is at its earliest moment: one delta PR (#11584, Apr 30) and one standalone Standards Track EIP (#11598, May 4, resubmitted from the briefly-open #11597) by overlapping authorship including soispoke, nerolation, lightclient, and vbuterin. The standalone EIP framing is the polished version, with explicit `NONCE_MANAGER` system-contract storage, atomic-with-payment-approval consumption semantics, and a 20k first-use gas surcharge tied to zero-to-nonzero `SSTORE` pricing. A May 5 forum question also reopened the scope of atomic batching itself, asking whether DEFAULT frames need transaction-level post-operation semantics rather than limiting atomic groups to SENDER frames.*

### 2D Nonces Sketch (Closed in Favor of Standalone EIP)

*nerolation — PR #11584, opened Apr 30, closed May 8*

Toni Wahrstätter opened a 28-line sketch as a delta against EIP-8141: replace the single sender nonce with `(nonce_key, nonce_seq)`. `nonce_key < 2**256`; per-key sequences run independently. `APPROVE_PAYMENT` and `APPROVE_PAYMENT_AND_EXECUTION` increment the per-key nonce. `TXPARAM(0x0B)` returns `nonce_key`. The PR sketches a tiered first-use gas cost (0 / 5000 / 22100, SSTORE-pricing-shaped) with a placeholder note that the figure is to be updated. Mempool guidance: one pending tx per `(sender, nonce_key)`, enabling parallel sequences. `nonce_key = 0` represents the legacy nonce slot for backward compatibility.

Closed by nerolation on May 8 with a one-line "Closing in favor of the EIP for now." once the standalone Keyed Nonces EIP (PR #11598) gathered the same idea into a separate Standards Track proposal with concrete `NONCE_MANAGER` semantics. The delta-against-8141 framing is abandoned; the design line continues entirely as PR #11598.

### EIP-8250: Keyed Nonces for Frame Transactions (Opened in This Phase, Merged Phase 12)

*soispoke, nerolation, lightclient, vbuterin — PR #11598, opened May 4*

Four days after the #11584 sketch, the same idea returned as the standalone Standards Track EIP-8250. PR #11598 opened May 4 and merged May 11 (lightclient editor approval); see Phase 12 for the merge writeup. The standalone framing keeps the parallel-sequence motivation and adds the implementation detail the sketch deferred:

- **State location**: non-zero keys live in storage of a `NONCE_MANAGER` system contract whose runtime code is the four-byte revert-only `0x60006000fd`. Slot derivation is `keccak256(left_pad_32(sender) || uint256_to_bytes32(nonce_key))`. Direct calls to `NONCE_MANAGER` revert; only protocol bookkeeping writes the slots, and the writes do not warm the address or slot under EIP-2929 nor charge under EIP-2200 SSTORE pricing.
- **Sequence width**: `nonce_seq` is `uint64`, with `MAX_NONCE_SEQ = 2**64 - 1` reserved as exhausted state. `nonce_key` is full `uint256` so privacy protocols can derive keys directly from 32-byte nullifiers, commitments, or hash outputs, rather than packing them into ERC-4337's 24-byte key field.
- **Atomicity with payment**: nonce consumption moves from a frame-execution side effect to a payment-approval transition. The unique successful payment-scoped `APPROVE` (scope 0x1 or 0x3) runs `consume_nonce` after EIP-8141's standard exceptional-condition checks but before any approval effect commits. `consume_nonce`, max-cost collection, payer recording, first-use gas charging, and approval-flag updates are journaled outside the frame's revert journal and outside `SENDER` atomic-batch snapshots, so they survive later-frame reverts and atomic-batch rollback. This is the load-bearing property for nullifier-style applications: a single-use key is atomically spent the moment payment is approved, not after the rest of the transaction succeeds.
- **First-use surcharge**: 20,000 gas, deducted from the frame executing the payment-scoped `APPROVE` only on the zero-to-nonzero transition for a non-zero key. Reuses the SSTORE state-creation reference cost. Subsequent increments of a consumed key are not separately surcharged.
- **TXPARAM additions**: `TXPARAM(0x0B)` returns `nonce_key`; `TXPARAM(0x0C)` returns the pre-state legacy sender nonce (transaction-scoped, not updated by approval, deployment, `CREATE`, or `CREATE2` mid-transaction). The latter exists so VERIFY code that previously relied on `TXPARAM(0x01)` being the legacy nonce can migrate cleanly.
- **Mempool**: does not relax EIP-8141's one-pending-tx-per-sender public-mempool guidance, but removes the protocol-level obstacle to a future keyed-aware policy that admits multiple pending transactions per sender on distinct non-zero keys.

The security section flags one subtlety worth tracking: a non-zero-key transaction does not advance the sender's legacy account nonce, so `CREATE` addresses computed from the legacy nonce can shift if another transaction advances the legacy nonce before inclusion. Applications relying on `CREATE` addresses must use `CREATE2` or authenticate `TXPARAM(0x0C)` explicitly. The "send another transaction with the same legacy nonce" cancellation pattern also does not work on keyed transactions; replacement requires the same `(sender, nonce_key, nonce_seq)`.

### Atomic Batching Scope: DEFAULT Frames and the VERIFY Carve-Out (Open)

*alex-forshtat-tbk and derek — EthMagicians posts #146, #147, May 5*

Alex Forshtat asked why atomic batching is limited to `SENDER` frames. The concern is that `DEFAULT` frames may want transaction-level post-transaction assertions or cleanup hooks, but the current atomic-batch flag only groups consecutive `SENDER` frames. Derek replied the same day with a finer-grained framing: VERIFY frames were excluded because letting them participate in atomic batches makes mempool validation harder (a VERIFY frame's outcome could be reverted by a later frame in the same batch), but he sees no protocol reason to exclude DEFAULT frames. He went further and proposed allowing *any* frame, including VERIFY, to be atomically batched at the protocol level, with the mempool restrictive tier separately forbidding VERIFY-in-batch. That way private pools could use atomically-batched VERIFY frames for revert-protection patterns without changing protocol rules. PR #11580's payer-before-sender ordering relaxation already used this same protocol-vs-mempool layering, so the pattern is consistent.

The exchange does not change the spec yet but reframes the design space: atomic batching as a protocol primitive applicable to all frame modes, with the restrictive-tier mempool policy carving out the cases that complicate validation.

**What to watch**: which of #11584 and #11598 the core authors converge on (the standalone EIP framing is more complete and avoids re-litigating EIP-8141's payload schema mid-fork, but it adds a new system contract); whether the 20k first-use surcharge survives review; whether the mempool one-pending-per-sender guidance gets relaxed in a follow-up; how the spent-once-with-payment property composes with the guarantors (#11555) and payer-before-sender (#11580) proposals from Phase 7, since both also want to commit to payment under specific conditions; and whether atomic batching opens up to DEFAULT and VERIFY frames at the protocol level with mempool policy carving back out the validation-time hazards.

---

## Phase 10: EIP-3607 Carve-Out and EVVM External Perspective (May 5 – May 7)

*Phase 10 closes a long-pending open issue and brings an external production data point onto the thread. Thegaram's PR #11272 (open since Feb 6) finally lands, explicitly carving frame transactions out of the EIP-3607 origination check, with EIP-3607 added to `requires` so the carve-out is declared rather than implied. The same week, ariutokintumi, co-founder of EVVM (a contract-native AA framework with ~200 deployments since 2023), reads through the full thread and contributes a contract-vs-protocol comparison covering institutional policy, async-execution compatibility, batch success granularity, and reservation primitives. Both events sit at the consolidation edge of the high-velocity Phase 5-9 churn: a long-overdue compatibility fix on one side and an outside production reference point on the other.*

### EIP-3607 Carve-Out for Frame Transactions (Merged)

*Thegaram — PR #11272, opened Feb 6, merged May 5*

The longest-pending open spec PR finally landed. EIP-3607 forbids transactions whose `tx.sender` has non-empty, non-delegation code, on the assumption that a contract account cannot sign a regular ECDSA transaction. Frame transactions deliberately allow `SENDER` frames to originate calls from contract accounts, so applying EIP-3607 unconditionally would have blocked native AA. The fix is two-line: add `3607` to the `requires` header and add a "Transaction origination" subsection documenting the carve-out: "Do not apply the restriction put in place by EIP-3607 to frame transactions. Specifically, `SENDER` frames originate calls where `tx.sender` is a contract account. Validation logic for other transaction types remains unchanged."

The compatibility issue had been raised on the magicians thread on day one ([post #26](https://ethereum-magicians.org/t/eip-8141-frame-transaction/27617/26)) and sat in the open-PR backlog through the high-velocity Phase 5-9 spec churn. lightclient's first approval was dismissed on Apr 8 after the spec moved underneath the PR; Thegaram refreshed the diff in late April and lightclient re-approved on May 5. Net spec impact is small (+7/-1) but architecturally clean: EIP-3607 becomes the first cross-EIP requirement that EIP-8141 explicitly opts out of in its `requires` list, with the carve-out stated in spec text rather than inferred by clients.

### EVVM Production Perspective on Frame vs Contract-Layer AA (External)

*ariutokintumi — EthMagicians post #148, May 7*

German Abal (co-founder, architect of EVVM, a contract-native AA framework deployed across ~200 instances on 10+ EVM chains since 2023) read the full thread and posted a four-paragraph comparison of EIP-8141's protocol-layer choices against EVVM's contract-layer experience. Three observations:

1. **Per-environment policy lives at the contract layer, not the protocol.** EVVM instances configure KYC/AML gates, allowlists, and custom delegatecall guards through per-deployment contracts. Real adopters (banks, regulated DeFi, public-sector pilots) need this variance. EIP-8141 correctly does not try to support it natively, but the rationale should be explicit that policy-variance is contract-layer territory.
2. **Async-execution compatibility is achievable as a contract-level property.** The Monad / Base / Tempo concerns raised earlier in the thread (DanielVF, chunter) bind only protocol-level validation. EVVM-style validation runs *inside* the contract at execution time against post-state, so async execution and contract-level AA coexist by design. Not an argument against EIP-8141, but a reference point for chains where 8141 cannot ship.
3. **Two implementation choices in EIP-8141 differ from EVVM in production**:
    - Per-operation success in batches: EVVM's `batchPay` returns `bool[]` per operation; derek's atomic batching in EIP-8141 is all-or-nothing. The trade-off (independent dust transfers vs. approve-then-swap dependencies) hits early and may force the per-op case into separate transactions.
    - Reservation primitives: EVVM ships only non-authoritative reservations because authoritative locks expose DoS-by-lock-grabbing and bad-UI lockup vectors. Likely matters at protocol level too if keyed nonces ship.

Notably the post does not propose a competing alternative or push back on EIP-8141's direction. It's a load-bearing external production data point on the design choices EIP-8141 has already made. The atomic-batching observation lines up with derek's own May 5 reframing (post #147) about extending atomic batching beyond SENDER frames; the reservation-primitives observation is one to flag against PR #11598's keyed-nonce commit-on-payment semantics, since a single-use key is a narrow form of authoritative reservation.

References from the post:
- EVVM repo: https://github.com/EVVM-org/testnet-Contracts
- EVVM docs: https://www.evvm.info/docs/intro
- Signature constructor: https://www.evvm.dev

## Phase 11: Editorial Review and Layering Pattern (May 7 – May 10)

*Phase 11 is bracketed by lightclient's PR #11621 (frames cleanup, opened May 7) but its character is editorial. samwilsn weighs in on naming consistency and a substantive `FRAMEDATACOPY` revert-vs-zero-pad design question; forshtat extends the protocol-vs-mempool layering pattern (Phase 7's PR #11580, Phase 9's atomic-batching posts) to the `SSTORE`-in-`VERIFY` ban. The cleanup PR itself opens in this phase but lands in Phase 12 (May 11) along with EIP-8250 and the next wave of merges; see Phase 12 for its merge writeup. The trailing concerns of Phase 11 are the unanswered `SSTORE`-in-`VERIFY` layering question and the `FRAMEDATACOPY` revert-semantics design question.*

### Editor Review and Spec Coherence Questions (External)

*samwilsn — EthMagicians post #149, May 8*

Sam Wilson posted an editorial review of the spec text shortly after PR #11621 opened, focused on naming consistency and minor specification gaps rather than the structural redesign. Four concrete observations:

1. **Empty-target representation**: `frame.target is None` should likely be `frame.target = b""` for consistency with how empty byte strings are written elsewhere in the spec.
2. **`APPROVE_PAYMENT_AND_EXECUTION` naming**: the constant should be renamed to match evaluation order; today's name implies payment precedes execution scope but the scope evaluation reads the bits in the opposite order.
3. **Undefined "paymaster frame"**: the term appears in rationale text without a prior definition; either define it or rephrase as "the frame at `paymaster_frame_index`".
4. **Five opcode slots**: questions whether the five new opcodes (`APPROVE`, `VERIFY`, `FRAMEPARAM`, `TXPARAM`, `FRAMEDATACOPY`) all justify their permanent opcode-space cost, acknowledging that the alternatives (one combined opcode with selectors, system-contract precompiles) each carry their own ergonomic and gas-accounting downsides.

The most substantive design question is on `FRAMEDATACOPY` revert behavior: Wilson notes `CALLDATACOPY` does not revert on out-of-bounds reads (it zero-pads instead) and asks why `FRAMEDATACOPY` was chosen to revert. The trade-off is real. Reverting on out-of-bounds catches a class of integer-arithmetic bugs at execution time at the cost of forcing contracts to know exact frame-data sizes ahead of the call. Zero-padding lets contracts copy "up to N bytes" without precomputing the actual length but silently absorbs miscalculations. The question is not formally answered on the thread; in practice the per-frame `FRAMEDATA` regions are typed and well-known to the validating contract, so the revert semantic is closer to a typed-read assertion than to a memory primitive. This is one to track if PR #11488 (open since Apr 6) gets folded into the cleanup.

### Layering Pattern Extends to VERIFY Restrictions (External)

*alex-forshtat-tbk — EthMagicians post #150, May 10*

Forshtat returned to the protocol-vs-mempool layering thread (posts #146-147 in Phase 9 and PR #11580 in Phase 8) and asked whether the same pattern should extend to the `SSTORE` restriction on `VERIFY` frames: "do you think the same can be said about other things where VERIFY frames are treated differently, say, for example the SSTORE not being allowed on the protocol level vs. as a mempool rule?"

The current spec bans `SSTORE` inside `VERIFY` frames at the protocol level on the rationale that storage writes in validation make the mempool's "did this validate?" decision dependent on observed state, which breaks the cheap parallel-validation property the restrictive mempool tier relies on. Forshtat's question accepts the mempool problem but asks whether the restriction belongs in the protocol or in mempool policy. If it lives in mempool policy, an unrestricted execution-layer can still admit such transactions through a permissive tier or out-of-band inclusion (block builders, restrictive-pool extensions), preserving the mempool guarantees only where they are needed.

The pattern is the same one derek articulated for atomic batching in post #147 and lightclient encoded in PR #11580: protocol semantics stay maximally permissive, mempool policy carries the restrictions that the canonical pool needs for safety. The thread has not yet replied, but the question is structurally important enough to flag as the trailing concern of Phase 11. If subsequent PRs migrate the `SSTORE`-in-`VERIFY` ban into mempool policy, the spec text in `current-spec.md` and `mempool-strategy.md` will need a paired update.

---

## Phase 12: Cleanup, Keyed Nonces, and the Extended Feature Set Bundle (May 11)

*Phase 12 is a single-day cluster on May 11. Two Phase 9-11 design lines merge within minutes of each other: lightclient's PR #11621 (frames cleanup) and soispoke's PR #11598 (EIP-8250 Keyed Nonces). Hours later, Pedro Gomes opens PR #11643, an "extended feature set" bundle absorbing guarantors, keyed nonces, signer binding, and envelope expiry into EIP-8141 itself, an inverse of the requires-chain layering EIP-8250 just established. #11643 was later closed in favor of PR #11681 (see [Phase 14](#phase-14-extended-feature-set-supersession-may-16-may-18)); the open architectural question, compose-by-requires vs absorb-into-base, is what carries forward.*

### Frames Cleanup Refactor (Merged)

*lightclient — PR #11621, opened May 7, merged May 11*

The readability sweep that opened Phase 11 merged on May 11 after the bot's same-day "✅ All reviewers have approved" never required additional revisions. samwilsn's editorial review (post #149) was treated as follow-up rather than gating. The +185/-345 net -160-line diff is the largest spec-text refactor since PR #11521 (Apr 14).

The substantive changes that landed:

- **Restructured spec body** under `### Frame Transaction` with `#### Payload Encoding` and `#### Field Definitions` subsections. Field definitions are centralized into bulleted lists per object (outer payload, frame object) rather than scattered across prose.
- **Skipped-batch receipt status**: receipt status `0x3` introduced for frames skipped as part of an atomic batch (previously skipped frames had no distinct status code).
- **FRAMEPARAM operand order**: explicitly defined (was implicit and inconsistent across the rationale section).
- **P256 removed from default code**: the protocol-shipped default code now only ships ECDSA secp256k1 verification.
- **Default code on SENDER/DEFAULT**: default code no longer reverts unconditionally on `SENDER` and `DEFAULT` modes; this lets top-level value transfers to a default-code account succeed via a frame transaction, which the previous default code blocked.
- **Requires header**: `7623` (calldata gas pricing) and `7702` (delegation indicators) added, formalizing dependencies that were already implicit.
- **Abstract and Motivation rewritten**: leads with the structural "frames" concept and lists the practical wins (key rotation, simpler smart accounts via batching, decentralized fee payment) before the post-quantum off-ramp framing.

Two changes deserve continued attention now that they have landed:

1. **Removing P256 from default code** retracts the hardware-wallet / passkey bridge that was the headline EOA-support story since PR #11379 (Mar 10). The PR description does not justify the change; it is not yet clear whether this was a deliberate scope-narrowing (P256 belongs in EIP-7932 / a per-account extension rather than the protocol-shipped default code) or an unintended consequence of the cleanup.
2. **Default code accepting SENDER and DEFAULT frames** changes the semantics of native ETH transfer to a fresh EOA via a frame transaction: previously such a transfer reverted in default code; after the merge it succeeds. Small in implementation but visible to wallets, indexers, and explorers.

### EIP-8250: Keyed Nonces for Frame Transactions (Merged)

*soispoke, nerolation, lightclient, vbuterin — PR #11598, opened May 4, merged May 11*

The standalone Keyed Nonces EIP merged minutes after PR #11621. abcoathup's May 6 non-editor approval ("Looks good enough for a draft", with a small preference for *transaction pool* over *mempool*) sat for five days awaiting an editor signoff. lightclient (as EIP editor) approved on May 11 and auto-merge fired the same minute. The CI flag on the initial commit history (containing the unrelated `eip-FOCIL.md` parent inherited from #11597's branch) did not gate the merge.

The significance is governance-structural more than spec-textual. EIP-8250 is the first EIP whose `requires` header includes EIP-8141, making the EIP-8141 + EIP-8250 pair the first compose-by-requires AA stack in the EIP series. The mempool one-pending-per-sender rule lives in EIP-8141; the parallel-sequence primitive lives in EIP-8250; and a future keyed-aware mempool policy can compose them without re-litigating EIP-8141's payload schema. The pattern is the same protocol-vs-mempool layering that derek and forshtat articulated in posts #146-147 and #150, lifted from a per-PR convention to an EIP-series convention.

### Extended Feature Set Proposal (Opened, Later Superseded)

*pedrouid — PR #11643, opened May 11 (closed May 18 in favor of [PR #11681](#extended-feature-set-supersession-may-16-may-18))*

Eight hours after PR #11598 merged, Pedro Gomes opened PR #11643 with the inverse packaging of EIP-8250's requires-chain approach: fold guarantors, flexible nonces, signer binding, and envelope expiry into EIP-8141 itself rather than chain them as sibling EIPs. The diff was +843/-69 lines, the largest single-PR diff against EIP-8141 since the original submission.

The architectural position taken was that a bundled upgrade is more efficient than three or four sibling EIPs with overlapping system contracts. The envelope-expiry portion overlapped directly with PR #11662 (EXPIRY_VERIFIER frame), which merged May 14 and shipped protocol-level expiry as a verifier-frame contract rather than an outer-envelope field. With the expiry design question settled by #11662's merge, the envelope-expiry component of #11643 became redundant, and Pedro closed the PR on May 18 in favor of [PR #11681](#extended-feature-set-supersession-may-16-may-18), which retains the three remaining features without the envelope-expiry field. Phase 14 covers the supersession.

---

## Phase 13: Atomic-Batch Expansion and Expiry Verifier Frame (May 12 – May 14)

*Phase 13 follows the May 11 merges with two more spec changes that each resolve threads opened earlier. On May 12, derek's PR #11652 extends atomic batching from `SENDER`-only to any frame mode, encoding the protocol-vs-mempool layering pattern from posts #146-147 (Phase 9) and #150 (Phase 11) at the frame-mode level. On May 14, nerolation's PR #11662 adds an EXPIRY_VERIFIER frame, the first new frame shape since the original Jan 29 design and the first carve-out from the otherwise-uniform `VERIFY`-frame rules (sighash elision, `TIMESTAMP` ban, `APPROVE` requirement, validation-trace constraints). forshtat returns at the end of the phase (post #155) with an open question about `VERIFY`-frame aggregation methodology and whether the state-modification constraints should propagate to `APPROVE` semantics.*

### Atomic Batching Extended to All Frame Modes (Merged)

*derekchiang — PR #11652, opened and merged May 12*

derek's same-day PR extends atomic batching from `SENDER`-only to any frame mode, with the mempool validation-prefix carving out atomic-batch frames separately. The diff is +9/-10 lines. lightclient approved within 30 minutes ("LGTM"), auto-merge fired the same day. Credits to forshtat for the suggestion (per derek's PR description), reflecting forshtat's post #146 question from Phase 9.

The supporting thread discussion is in EthMagicians posts #151-154:

- **Post #151 (dror, May 11)**: clarifies that `VERIFY` frames are implicitly batched with all other frames in the sense that a `VERIFY` revert invalidates the entire transaction; this is the baseline invariant any atomic-batch extension must preserve.
- **Post #152 (derek, May 12)**: explains the `SSTORE`-in-`VERIFY` ban serves an aggregation invariant: removing `VERIFY` frames from a transaction should not change observable behavior, which lets block builders aggregate signature verification across transactions. Letting `VERIFY` frames participate in atomic batches breaks this invariant, since a later frame's revert could roll back the verification.
- **Post #153 (derek, May 12)**: notes no frame currently reverts a `VERIFY` frame's effects; admitting `VERIFY` into atomic batches would be the first construct doing so.
- **Post #154 (derek, May 12)**: announces the merge.

The architectural significance: the spec now encodes the protocol-vs-mempool layering pattern (PR #11580, forshtat's #146-147, #150) at the frame-mode level. Atomic batching is a protocol primitive applicable to all frame modes; the restrictive-tier mempool policy carves out validation-prefix atomic batches separately. This opens `DEFAULT`-frame batching for transaction-level post-op cleanup and (under permissive-tier propagation) `VERIFY`-frame batching for revert-protected validation sequences, without forcing those use cases out of the public mempool entirely.

### EXPIRY_VERIFIER Frame Added (Merged)

*nerolation (Toni Wahrstätter) — PR #11662, opened May 13, merged May 14*

Toni Wahrstätter opened PR #11662 on May 13 with the first new frame shape since the original Jan 29 design: an expiry verifier frame. The diff is +88/-33 lines. lightclient approved on May 14 ("This is great! Thanks Toni!", with a rocket reaction) and auto-merge fired the same day.

The design pins a single canonical address (`EXPIRY_VERIFIER = address(0x8141)`) whose runtime code is fixed at activation. A `VERIFY` frame whose `frame.target` equals this address is an expiry-verifier frame: `frame.data` is interpreted as an 8-byte big-endian unix-seconds deadline, and the runtime reverts unless `block.timestamp <= expiry_timestamp`. Constraints: `frame.flags == 0`, `frame.value == 0`, `len(frame.data) == 8`, at most one expiry-verifier frame per transaction.

The frame breaks three previously-uniform spec rules in narrow, controlled ways:

1. **Signature hash**: expiry-verifier `frame.data` is *not* elided from the signature hash. Every other `VERIFY` frame's `frame.data` is elided (per the Jan 29 day-0 fix in PR #11205). The deadline is a sender-authored commitment that must not be malleable in transit, so it must be covered by the signature.
2. **`TIMESTAMP` ban**: the `TIMESTAMP` opcode is banned during validation-prefix execution; expiry-verifier frames executing the canonical runtime get a single carve-out. Clients may optimize by skipping explicit EVM execution and performing the deadline check natively.
3. **`APPROVE` requirement**: `VERIFY` frames previously had to call `APPROVE` to be valid; PR #11662 relaxes this to "if the frame reverts, the transaction is invalid". An expiry-verifier frame succeeds without `APPROVE`. The mempool admission rule is narrowed accordingly: only `self_verify`, `only_verify`, and `pay` frames are required to call `APPROVE`.

Mempool admission for expiry-verifier frames is special-cased: nodes MUST drop a transaction whose expiry is less than the current view of `block.timestamp` at any point. Expiry-verifier frames are exempt from validation trace rules, storage-dependency tracking, and `MAX_VERIFY_GAS`. The validation-prefix shape-matching rules treat expiry-verifier frames as transparent (e.g. `[expiry_verify, self_verify]` is recognized as `[self_verify]`).

The "pinned target address whose runtime is fixed at activation" pattern (similar to `ENTRY_POINT`, EIP-4788, EIP-2935) becomes the second protocol-codepath inside EIP-8141 after the default code. Future system-frame designs (paymaster reservation, key delegation, etc.) likely follow the same pattern: a reserved address, a canonical runtime, and a narrow carve-out from the otherwise-uniform `VERIFY`-frame rules.

One open question that did not gate the merge: whether the canonical runtime reads `TIMESTAMP` (current draft, with the explicit ban-carve-out) or reads the block header directly. Toni flagged the question in the PR description; the submitted version reads `TIMESTAMP`, which is the simpler and more EVM-native choice but requires the explicit ban exception.

### Aggregation and APPROVE Questions Surface (External)

*alex-forshtat-tbk — EthMagicians post #155, May 14*

The day PR #11662 merged, forshtat thanked derek for the atomic-batching extension and asked for more detail on `VERIFY`-frame aggregation methodology, and whether the state-modification constraints (the `SSTORE`-in-`VERIFY` ban, derek's aggregation invariant in post #152) should affect `APPROVE`'s behavior. The question is unanswered as of this sync and is the trailing concern of Phase 12: derek's aggregation invariant ("removing `VERIFY` frames doesn't change behavior") implies anything `VERIFY` frames write to transaction state has to be carefully scoped, and `APPROVE` is the only protocol-defined `VERIFY` write. If aggregation pressure pushes for further restrictions, `APPROVE`'s semantics could need follow-up work.

**What carried into Phase 14**: Pedro's bundle did get reviewer engagement, but in the form of a self-issued rewrite: after PR #11662 settled the expiry design as a verifier-frame contract, #11643 was closed and replaced by PR #11681, which keeps guarantors, keyed nonces, and signer binding but drops the envelope-expiry field. The compose-by-requires vs absorb-into-base question remains open. Open into Phase 15: whether #11555 (guarantors) or #11580 (payer-before-sender) lands separately or folds into #11681; whether the `SSTORE`-in-`VERIFY` ban migrates to mempool policy as the trailing Phase 11 question asked; whether `APPROVE` semantics get follow-up restrictions from the aggregation thread (#155); and whether samwilsn's editorial-review items (post #149, especially the `FRAMEDATACOPY` revert-vs-zero-pad question) get a follow-up PR.

---

## Phase 14: Extended Feature Set Supersession (May 16 – May 18)

*Phase 14 captures the resolution of the open architectural question Phase 12 left dangling. Pedro's May 11 PR #11643 bundled four features into EIP-8141 itself, an inverse of the requires-chain layering EIP-8250 established. Within five days, PR #11662 (EXPIRY_VERIFIER frame, merged May 14) settled the envelope-expiry component independently, by shipping protocol-level expiry as a verifier-frame contract rather than an outer-envelope field. On May 16 Pedro opened a successor PR, #11681, that retains guarantors, keyed nonces, and signer binding but drops the now-redundant envelope-expiry field. On May 18 he closed #11643 in favor of #11681. The substantive architectural position carried forward (keyed nonces and guarantors should ship together inside EIP-8141 rather than as sibling EIPs) is unchanged; only the envelope-expiry component dropped out.*

### Extended Feature Set Successor Opened (Open)

*pedrouid — PR #11681, opened May 16*

Five days after PR #11662 shipped EXPIRY_VERIFIER as the protocol-level expiry mechanism, Pedro opened PR #11681 as the successor to #11643. The diff is +810/-74 lines across three files. The PR description retains the same architectural position as #11643 (keyed nonces and guarantors should ship together inside EIP-8141, sharing one system contract, rather than as a requires-chain of sibling EIPs), but drops the envelope-expiry field that PR #11662 made redundant.

The three features bundled in #11681:

1. **Guarantors**: adopted from PR #11555 (derekchiang) verbatim. New approval scope `APPROVE_GUARANTEE = 0x4`, a `compute_frame_sig_hash` helper, a `guarantor_approved` transaction-scoped flag, and a canonical-paymaster guarantor mode with a `bumpNonce` entry. The mempool tier that today rejects shared-state-reading sender validation can admit those transactions when a guarantor signature carries the risk.
2. **Keyed Nonces**: mirrors EIP-8250's `(nonce_key, nonce_seq)` semantics with one shape change. A single `uint64 signer` envelope field replaces the `(nonce_key, nonce_seq)` pair, so the same identifier indexes both the keyed nonce stream and the registered pubkey for signer binding. `signer == 0` aliases the legacy account nonce. The position taken: keyed nonces belong inside EIP-8141 because the upgrade path is more efficient when keyed nonces share the same system contract as guarantors and signer binding rather than living in a separate sibling EIP.
3. **Signer Binding**: a transaction-scoped `verified_signers` table populated by non-secp256k1 `VERIFY` frames that prove `(digest, address)` against a registered pubkey. `ECRECOVER` consults the table on the hit path; the miss path is byte-identical to upstream.

The envelope-expiry field present in #11643 is dropped; the `signer` envelope field is the only new outer-envelope addition. One new system contract (`AUTH_MANAGER` at a reserved address, EIP-4788 / EIP-2935 pattern) holds both the keyed nonce streams and the registered pubkey signers under one storage layout. Zero new opcodes, zero new precompiles, zero account-RLP changes.

The architectural tension with EIP-8250 is unchanged from #11643. EIP-8250's `requires` header established the compose-by-requires layering pattern at the EIP-series level; PR #11681 (and #11643 before it) take the absorb-into-base position that one bundled EIP with a shared system contract is a more efficient upgrade path than a chain of sibling EIPs with overlapping responsibilities. If #11681 lands, it supersedes EIP-8250 by absorption. The question of which packaging EIP-series convention wins is the open architectural question for Phase 15.

### PR #11643 Closed in Favor of #11681 (Closed)

*pedrouid — PR #11643, closed May 18*

Two days after opening PR #11681, Pedro closed #11643 with a one-line comment: "Closed in favor of #11681". Net spec impact of #11643 is zero; the substantive proposal carries forward in #11681 minus the envelope-expiry field. The supersession is a small but informative governance signal: in-flight PRs do rebase against the post-merge spec when external events (here, PR #11662) settle one of their components independently. The four-feature bundle becomes a three-feature bundle without re-litigating the bundle vs requires-chain question.

**What carried into Phase 15**: the architectural question Phase 14 framed (compose-by-requires vs absorb-into-base) carries forward with both sides still represented by open PRs. The next phase opens immediately with a second sibling EIP proposal that takes the opposite position from PR #11681's bundle.

---

## Phase 15: EIP-8266 Opens, Soispoke Review, and Trebor's Paymaster Questions (May 19 – May 21)

*Phase 15 opens one day after Phase 14 closes, with the architectural question Phase 14 framed (compose-by-requires vs absorb-into-base) getting a fresh data point from the opposite direction. PR #11681 (Phase 14) takes the absorb-into-base position, folding keyed nonces, guarantors, and signer binding into EIP-8141 itself. On May 19 nerolation and lightclient open PR #11692, a second sibling EIP requiring EIP-8141 ("Expiring Nonces for Frame Transactions"); soispoke (the EIP-8250 lead author) reviews it within hours and catches two correctness bugs, and on May 20 abcoathup assigns EIP number 8266. On May 21 trebor (a Kohaku privacy-focused paymaster researcher) posts two questions (#156, #157) reopening the feasibility of EIP-8141's Example 3 (ERC-20 paymaster) under the current restrictive-tier rules. Phase 16 picks up on May 22 when the two long-open structural threads merge.*

### Expiring Nonces Sibling EIP Opened (Open)

*nerolation (Toni Wahrstätter), lightclient — PR #11692, opened May 19*

PR #11692 adds a new Standards Track EIP (placeholder `eip-9999.md`, +161 lines) layering an "expiring-nonce" mode on EIP-8141. The proposal trades unbounded per-tx state growth for a fixed-capacity ring buffer, and reuses two existing primitives rather than introducing new envelope fields.

The mechanism in three parts:

1. **Sentinel-mode selection**: a transaction is in expiring-nonce mode if `tx.nonce == 2**64 - 1`. Reusing the existing `nonce` field keeps the payload schema unchanged and lets the canonical signature hash continue to commit to the mode marker through the existing field. Other nonce values follow EIP-8141 untouched.
2. **Ring-buffer state**: a `NONCE_RING` system contract (runtime `0x60006000fd`, revert-only) holds a fixed `RING_CAPACITY = 2**18` slot ring. Consumption happens atomically on the unique payment-approving `APPROVE`, scoped as an approval effect (journaled outside the current frame's revert journal and outside any `SENDER` atomic-batch snapshot). The zero-to-nonzero `SSTORE_SET` premium is intentionally omitted because the ring's leaf count is invariant in steady state: every fresh `slot_seen(h_new)` write is paired with a `slot_seen(h_old) = 0` clear within the same transition. A flat `EXPIRING_NONCE_GAS = 13000` covers the read/write set.
3. **Deadline enforcement**: the deadline is enforced by reusing PR #11662's `EXPIRY_VERIFIER` frame rather than a parallel envelope field. The 8-byte big-endian unix-seconds deadline is capped at `MAX_EXPIRY_SECS = 60`, keeping the sizing invariant `MAX_EXPIRY_SECS × peak_tps ≤ RING_CAPACITY` (the ring tolerates ~4369 sustained TPS before eviction races a live deadline). A `BufferFull` halt acts as defense-in-depth if the invariant ever breaks.

The architectural significance: PR #11692 stakes the opposite position from PR #11681 in the open Phase 14 question. PR #11681 argues that keyed nonces, guarantors, and signer binding belong inside EIP-8141 as a bundled upgrade. PR #11692 takes another candidate (expiring nonces, a nonce-mechanism alternative that overlaps with EIP-8250) and ships it as a second sibling EIP requiring EIP-8141, extending the compose-by-requires layering EIP-8250 established. The two open PRs now encode the same question from opposite ends.

The composition with EIP-8250 is explicitly addressed: if both ship, the expiring-nonce sentinel collapses into EIP-8250's keyed-nonce framing as a reserved `nonce_key == 2**256 - 1`, and `NONCE_RING`'s storage moves under a distinct slot prefix inside `NONCE_MANAGER`. This composition is non-normative but signals that the compose-by-requires camp expects sibling EIPs to compose with each other, not just with the base EIP.

Mempool implications are the visible policy break with EIP-8141's defaults: nodes MAY admit multiple pending expiring-nonce transactions per sender, reserving `TXPARAM(0x06)` against the payer's available balance for each, rather than enforcing EIP-8141's one-pending-frame-transaction-per-sender guidance. This is consistent with EIP-8250's treatment (parallel sequences per `(sender, nonce_key)`) and reinforces the layering pattern: the one-pending-per-sender rule is EIP-8141's, not a property of frame transactions in general, and sibling EIPs can relax it for their own scoped sequences.

The PR opened May 19 with CI initially flagging commit-graph errors.

### Soispoke Review and Same-Block Replay Fix (Review)

*soispoke, nerolation — PR #11692 inline review, May 19*

Within six hours of submission, soispoke (Thomas Thiery, the lead author of EIP-8250) left a five-comment inline review on the PR. The review caught two correctness issues, pushed back on the privacy framing, and raised a mempool-policy question that nerolation answered on the spot. The review is a direct test of the compose-by-requires pattern in practice: the author of the first sibling EIP reviewing the second sibling EIP, on the day it lands.

The substantive items:

1. **Same-block replay window**. The stateful-validity check `state[NONCE_RING].storage[slot_seen(h)] < now` is a strict inequality. soispoke observed that after a first inclusion stores `slot_seen(h) = now`, a second inclusion in the same block also passes the check (because `now < now` is false, but the EXPIRY_VERIFIER deadline check `block.timestamp <= d` still passes at `d == block.timestamp`). The fix soispoke proposed: `stored == 0 || stored < now` for stateful validity, and `oldDeadline >= now` (not `>=` with a tighter bound) for ring-buffer eviction. nerolation acknowledged ("Ah now I see it yeah, you're right! Will commit a fix").
2. **Pricing reframe**. soispoke argued the 13000 charge undersells `slot_seen(h)` writes plus first-time `slot_ring(idx)` writes, since each new transaction writes a fresh storage key. nerolation reframed the justification: "the 13,000 charge is justified by paired set+clear keeping trie leaves invariant, not by 'overwriting existing slots.'" The gas number stayed; the justification now points at the invariant rather than the slot-reuse intuition.
3. **Privacy framing walked back**. soispoke noted the replay key is `compute_sig_hash(tx)`, which only prevents same-hash replay and does not stop two different envelopes from spending the same private note. Privacy-pool spends still need a nullifier or keyed-nonce check; expiring nonces are "duplicate-hash replay protection, not single-use spend protection." nerolation agreed: "I later realized that it might not be useful for privacy at all (at least always when there is a nullifier involved). Otherwise, when the ring buffer expired, the transaction could be slightly changed (changing the hash) which would allow to double-spend the nullifier." The retracted use case is one of the originally-claimed motivations.
4. **Mempool one-pending-per-sender rule**. soispoke asked whether there should still be a bound on pending expiring-nonce transactions per sender. nerolation confirmed the rule is intentionally dropped for expiring-nonce mode because mass-invalidation is not a problem in this regime, but the "payer has enough funds for multiple pending txs" reservation rule stays in place. This is the visible policy break with EIP-8141's defaults and is consistent with EIP-8250's per-key relaxation, but goes one step further: no per-key bookkeeping, just payer-balance reservation.

The review pattern is itself informative. The author of EIP-8250 catches bugs in EIP-8266 and walks back its privacy claim, but does not push back on the architectural question of whether expiring nonces should be a sibling EIP at all. soispoke engages with EIP-8266 as a peer sibling, not a competitor, which is exactly what the compose-by-requires pattern presupposes: sibling EIPs share a review surface, expect to compose, and don't relitigate base-EIP-8141 framing during their own reviews.

### Number Assignment and Discussions Topic (Procedural)

*abcoathup — PR #11692 comment, May 20*

abcoathup (EIP editor associate) assigned EIP number **8266** on May 20 with the standard editor note: "Assigning next sequential EIP/ERC/RIP number. Numbers are assigned by editors & associates. Please also update the filename." The filename moves from the placeholder `EIPS/eip-9999.md` to `EIPS/eip-8266.md` and the additions count ticks 161 → 162. abcoathup also requested a separate discussions topic on EthMagicians, which nerolation created the same day at [ethereum-magicians.org/t/28575](https://ethereum-magicians.org/t/eip-8266-expiring-nonces-for-frame-transactions/28575) (1 post so far, the initial announcement).

The procedural step is small but the implication is not. EIP-8266 now sits in the formally-numbered Standards Track slot adjacent to EIP-8250, with its own discussion thread separated from the original Frame Transaction thread. The "frame-transaction thread carries everything" pattern is over; from Phase 15 onward, sibling-EIP discussion fragments across per-EIP threads, and the original 27617 thread becomes one of several venues for EIP-8141-adjacent design discussion.

### Trebor Reopens Example 3 (ERC-20 Paymaster) Feasibility

*trebor — EthMagicians posts #156-157, May 21*

trebor (researcher on Kohaku's privacy-focused paymaster designs) read through the spec and raised two related concerns. The first (#156) is concrete: Example 3 in the spec lists "EOA Paying Gas in ERC-20s" using a non-canonical paymaster, but the restrictive-tier rules forbid that paymaster from reading any storage outside `tx.sender`. A non-canonical paymaster cannot check that the user holds enough ERC-20 to repay it without reading the ERC-20 contract's storage. The canonical paymaster's fixed bytecode (ecrecover-only) does not cover ERC-20 either. trebor asks what mechanism Example 3 is supposed to use under the current rules.

The second (#157) is structural: trebor argues that complexity in the verification phase is not just dangerous, it is also useful. Some of Kohaku's alternative privacy-focused paymaster designs need flexibility that the current restrictive tier disallows; the spec should not scope the verification phase so tightly that only known use cases survive. Both posts are unanswered as of this sync.

The concrete answer to #156 lives in [Mempool Strategy → ERC-20 gas repayment: two paymaster patterns](/mempool-strategy#erc20-paymaster-patterns): the spec's Example 3 is the **permissionless ERC-20 paymaster (onchain)** variant, which is consensus-valid but does not propagate through the public restrictive mempool; it routes through the expansive tier, a private mempool, or direct-to-builder submission. The **live ERC-20 paymaster (offchain)** variant uses a signing service to avoid the external-storage read and does propagate through the restrictive public mempool (one pending tx per non-canonical paymaster). Whether Example 3 in `current-spec.md` should be relabeled to make the mempool tier explicit is an open editorial question; the spec text already carries a `Mempool note` paragraph below the example, but trebor's question shows it can still be missed.

The structural concern in #157 lines up with forshtat's protocol-vs-mempool layering thread (posts #146-147, #150) and the guarantors / payer-before-sender debate: if the restrictive-tier rules block a class of verification designs that the protocol could otherwise support, the right answer may be to push those restrictions out of the protocol and into mempool policy, where alternative tiers (expansive, private, direct-to-builder) can carry them. PR #11555 (guarantors) and PR #11580 (payer-before-sender) both attack the same problem for sender validation; trebor's concern extends it to paymaster validation.

**What changed because of this phase**: PR #11692 opened May 19 as the second sibling EIP requiring EIP-8141; soispoke's May 19 inline review caught the same-block replay strict-inequality bug, the pricing-justification slip, and the over-stated privacy framing, all folded into commits before the Phase 16 merge. EIP number 8266 assigned May 20 with its own EthMagicians discussion thread. trebor's May 21 posts #156-#157 reopened the Example 3 (ERC-20 paymaster) feasibility question and surfaced a structural concern about the restrictive tier blocking otherwise-valid paymaster verification designs.

---

## Phase 16: Signatures List and EIP-8266 Merged (May 22)

*Phase 16 is a one-day cluster on May 22. Two long-open structural threads close within sixteen hours of each other: PR #11481 (signatures list to outer tx, open since Apr 2) merges as the largest outer-envelope change since the original Jan 29 design, and PR #11692 / EIP-8266 itself merges as the second sibling EIP in the compose-by-requires stack. The compose-by-requires architectural pattern that EIP-8250 established now has two siblings backing it, raising the bar for PR #11681's absorb-into-base counter-proposal.*

### Signatures List in Outer Tx (Merged)

*lightclient — PR #11481, opened Apr 2, merged May 22*

The longest-open spec PR after PR #11272 (EIP-3607 carve-out, merged May 5) finally landed. PR #11481 had all-reviewer approval since Apr 2 but sat open for seven weeks across multiple subsequent spec churns; jochem-brouwer's May 22 editorial review ("Some small questions and comments!") closed the loop and the auto-merge fired minutes later. Net diff is +199/-61, the largest spec-text addition since PR #11521 (Apr 14 broad spec tightening).

The substantive change is a new `signatures` outer-envelope field, a list of signature objects each carrying the signature itself plus metadata (algorithm, message, signer). Signatures are verified before any frame executes; during execution, signer identity and signature validity are already established and contracts only need to check authority. The default code is updated to read from this list rather than from `frame.data`. The forward-compat hook is what justifies the schema change: a future block-level aggregated witness can replace the per-tx signature list entirely, eliding individual signatures from on-wire block serialization while preserving per-tx commitments. PQ signatures (Falcon ~666 bytes, Dilithium ~2,420 bytes) make this aggregation path strategically important.

derekchiang's Apr 9 concern about signature-index discovery (a contract cannot hardcode its index in the outer list because the list may be reordered, so the default code has to loop through entries to find a matching signer) is not separately addressed in the merged diff. The known-limitation status is the trade-off the merge accepts: ergonomics of index discovery in exchange for the aggregation forward-compat hook. The leading candidate for a follow-up PR is a deterministic signature-ordering rule (e.g. sort by signer address) so contracts can binary-search rather than linear-scan, but no such PR has opened yet.

Three subsequent forum comments (0xrcinus Apr 15, morph-dev Apr 16, 0xrcinus Apr 16) debated a default-code optimization that would skip the per-frame VERIFY when `tx.sender` had a matching sighash entry in the outer list. morph-dev rejected the shortcut citing key-rotation cases where the sender's signing key is not the smart-wallet's current validation key; an attacker who learned the old key could bypass the contract's rotation logic. The optimization did not make the merge.

### EIP-8266: Expiring Nonces for Frame Transactions (Merged)

*nerolation, lightclient — PR #11692, opened May 19, merged May 22*

Three days after opening (and after the soispoke review and number-assignment events captured above), PR #11692 lands as EIP-8266. The diff is +162/-0 new lines at `EIPS/eip-8266.md`; `requires` includes both EIP-8141 and EIP-8250. jochem-brouwer's editor signoff ("Editorial: LGTM! I'll leave some review comments on EthMagicians later") fired the auto-merge on May 22.

The soispoke review fixes (same-block replay strict-inequality, paired set+clear pricing justification, retracted privacy framing) were folded into the final commits before merge. The retracted "privacy" claim survives as a known scoping note rather than as a load-bearing motivation: expiring nonces are duplicate-hash replay protection, not single-use spend protection, and a privacy pool still needs nullifier or keyed-nonce checks on top.

One post-merge PR comment from kdenhartog (May 24) raised an ECDSA nonce-reuse-attack concern referencing pcaversaccio/ecdsa-nonce-reuse-attack; he edited the same comment shortly after to retract, having confirmed in the AA channel that the EIP does not change replay-protection semantics in the way his original concern assumed.

The architectural significance is that the compose-by-requires pattern now has two siblings (EIP-8250 and EIP-8266) rather than one. The absorb-into-base counter-proposal in PR #11681 (Pedro Gomes, open since May 16) competes against a baseline that already shows the requires-chain pattern working at scale. The Phase 14 question of whether PR #11681 lands by superseding EIP-8250 now has an additional dimension: would it also supersede EIP-8266, or would EIP-8266 stay as a sibling regardless of #11681's outcome?

**What changed because of this phase**: PR #11481 merged May 22 (signatures outer-envelope field, largest spec-text addition since PR #11521); PR #11692 / EIP-8266 merged May 22 as the second sibling EIP in the compose-by-requires stack.

**What to watch into Phase 17**: whether PR #11681 closes or merges now that the compose-by-requires baseline has expanded to two siblings; whether a follow-up PR addresses the signatures-list index-discovery weakness (deterministic ordering would let contracts binary-search rather than linear-scan); whether trebor's #156-#157 questions prompt either an Example 3 label clarification in `current-spec.md` or a serious push to migrate restrictive-tier paymaster rules into mempool policy; whether a third sibling EIP appears in the same window, which would settle the architectural question by empirical pressure; whether the previously-trailing concerns (forshtat's #155 aggregation question, samwilsn's #149 editorial items) get follow-up PRs.

---

## Phase 17: Third Sibling EIP Opens (May 25 – May 28)

*Phase 17 opens three days after Phase 16's May 22 merges, and the trailing Phase 16 question about whether a third sibling EIP would appear is answered with a yes. soispoke, vbuterin, and nerolation open PR #11726 introducing EIP-8272: Recent Roots for Frame Transactions, the third compose-by-requires sibling after EIP-8250 and EIP-8266. The empirical pressure against PR #11681's absorb-into-base packaging now spans three siblings, all merged or in active review, while #11681 itself remains open with no reviewer signoffs since May 16. Phase 16 also marks the first sibling EIP to introduce both a new top-level payload field and a new opcode, so the compose-by-requires pattern stops being a nonce-mechanism convention and becomes a general extension surface.*

### EIP-8272: Recent Roots for Frame Transactions (Open)

*soispoke (Thomas Thiery), vbuterin (Vitalik Buterin), nerolation (Toni Wahrstätter) — PR #11726, opened May 25*

PR #11726 introduces EIP-8272 as a 394-line sibling EIP requiring both EIP-7843 (consensus `slotNumber` field) and EIP-8141. The proposal addresses a recurring restrictive-tier gap: validation rules sometimes need to depend on recent application state (privacy-pool tree roots, wallet authorization roots, account validation roots), but the restrictive mempool tier forbids validation from reading arbitrary storage controlled by another account. The pre-EIP-8272 options were either (a) re-implement signed-snapshot logic per application with no shared verification surface or (b) stay locked out of the public mempool entirely. EIP-8272 makes the snapshot a protocol primitive.

The mechanism in three pieces:

1. **Outer-envelope field**: a new top-level `recent_root_references` list (max 16 entries), each a `(source_id, slot, root)` tuple. Source identifiers are `keccak256(source_address || salt)`, so a single writer can publish multiple independent root streams by varying the salt. References target slots strictly before `current_slot`, which clients MUST obtain from the EIP-7843 `slotNumber` field rather than deriving from `block.timestamp` (the discipline preventing clock-skew validity divergence).
2. **System contract**: `RECENT_ROOT_ADDRESS` accepts 64-byte calldata (`salt || root`) and stores the entry at `entries[S mod RECENT_ROOT_LENGTH]` keyed by `keccak256(RECENT_ROOT_STORAGE_DOMAIN || source_id || uint64_be(i))`. `RECENT_ROOT_LENGTH = 8192` with `RECENT_ROOT_USABLE_WINDOW = 8191`. Per-source storage is bounded; total state grows with the number of distinct writers, not with reference volume. The contract follows the now-established `EXPIRY_VERIFIER` / `NONCE_RING` / `NONCE_MANAGER` pattern: revert-only on direct calls outside its system role, no read operations exposed.
3. **New opcode**: `RECENTROOTREFLOAD = 0xB4` (gas 3) for validation-time read access to a verified reference's root; `TXPARAM(0x0D)` returns the reference count. This is the first sibling-EIP-introduced opcode in the frame-transaction stack and would be the fifth opcode in the `0xB*` range after `TXPARAM (0xB0)`, `FRAMEDATALOAD (0xB1)`, `FRAMEDATACOPY (0xB2)`, and `FRAMEPARAM (0xB3)`.

Validation semantics: recent root references are checked after the EIP-8141 nonce check and before frame execution, against the transaction pre-state. Competing blocks at the same slot validate against their own pre-states, so a reference is valid only in a block whose pre-state contains the referenced entry. Each `(source_id, slot)` has at most one referenceable root on the canonical chain; multiple writes in the same slot target the same storage key, and only the final write in canonical order is referenceable from slot `S + 1` onward. References are included in `compute_sig_hash(tx)` (not elided by the VERIFY-frame data-elision rule) and frame data MUST NOT mutate the reference set during execution.

abcoathup left a non-editor approving review on May 26 ("Looks good enough to merge as a draft"). The eth-bot reports 1 more reviewer needed (`@g11tech`, `@jochem-brouwer`, `@lightclient`, `@samwilsn`). The CI commit-graph errors from the initial submission were addressed in subsequent commits, matching the now-routine sibling-EIP submission pattern.

### Compose-by-Requires Pattern Consolidates (Architectural)

PR #11726 raises the compose-by-requires sibling count to three (EIP-8250 merged May 11, EIP-8266 merged May 22, EIP-8272 in active review). Two architectural observations follow.

First, the author overlap is no longer coincidental. soispoke and nerolation lead EIP-8250; nerolation and lightclient lead EIP-8266; soispoke, vbuterin, and nerolation lead EIP-8272. Three of the four EIP-8250 co-authors have shipped or are shipping at least two sibling EIPs in this stack. The compose-by-requires camp is not a one-off pattern but a deliberate layered design with a recurring author roster.

Second, EIP-8272 expands the sibling surface in ways the prior siblings did not. EIP-8250 reshaped an existing payload field (`nonce` → `(nonce_key, nonce_seq)`). EIP-8266 used a sentinel value in the existing `nonce` field, no schema change. EIP-8272 introduces a brand-new top-level field (`recent_root_references`), a brand-new opcode (`RECENTROOTREFLOAD`), and a brand-new non-EIP-8141 dependency (EIP-7843). The compose-by-requires pattern is now demonstrating that siblings can extend the transaction-payload schema, the opcode space, and the dependency graph, not just the storage layer of system contracts. This is the strongest empirical argument yet against PR #11681's absorb-into-base premise: absorbing the EIP-8250 + EIP-8266 + EIP-8272 stack into EIP-8141 would mean folding three independently-shipped feature surfaces, two of which (EIP-8266 expiring nonces, EIP-8272 recent roots) were not in scope for PR #11681's `AUTH_MANAGER` design when it was opened.

PR #11681 remains open with no reviewer signoffs since the May 16 submission. The next sync should check whether Pedro rebases against the EIP-8266 + EIP-8272 baseline, retracts the bundle in favor of joining the compose-by-requires camp, or closes #11681 outright.

**What to watch into Phase 18**: whether EIP-8272 gathers an editor signoff and merges within the same window as its predecessors (EIP-8250 took 7 days from submission to merge; EIP-8266 took 3); whether the `RECENTROOTREFLOAD` opcode triggers a samwilsn-style editorial review questioning whether the sibling EIPs should share an opcode-budget governance check; whether the EIP-7843 dependency exposes any ordering issues with the Hegotá fork sequence (EIP-7843 status TBD); whether PR #11681 reorients in response to the three-sibling baseline; and whether a fourth sibling EIP appears, which would push the question from "is this a pattern" to "what is the upper bound on the compose-by-requires stack."

---

## Phase 18: EIP-8250 Update and ERC-20 Paymaster Resolution (Jun 1 – Jun 4)

*Phase 18 covers the four-day window between the EIP-8272 submission (Phase 17 close) and EIP-8272's merge (Phase 19 open). Two threads close. EIP-8250 evolves in place via PR #11749 on Jun 1 (single keyed nonce generalized to a bounded key set), establishing that sibling EIPs are first-class evolving documents rather than frozen one-shot proposals. trebor's ERC-20 paymaster question from Phase 15 reaches resolution in posts #158-#159 (matt clarifies on Jun 3, trebor commits to a clarifying PR on Jun 4). Both events tie up earlier-phase loose ends without changing the core trajectory.*

### EIP-8250 Adds Nonce Key Sets (Merged)

*soispoke — PR #11749, opened and merged Jun 1*

PR #11749 generalizes EIP-8250 from a single keyed nonce `(nonce_key, nonce_seq)` to a bounded key set `(nonce_keys, nonce_seq)`. The diff is +64/-40, a single file. Auto-merged in 28 minutes with no public review thread.

The motivation lives in use cases the single-key shape forced into either chained transactions or off-chain coordination: multi-device wallets coordinating one session, relayer accounts arbitrating shared inclusion, session-key bundles needing to advance multiple sequences in lockstep within a single transaction. With key sets, all keys in the set advance their `nonce_seq` atomically as part of the payment-approving `APPROVE`; partial advancement is impossible. The single-key path remains expressible as a one-element set, preserving the EIP-8266 sentinel composition.

The architectural signal is that sibling EIPs are first-class evolving documents. EIP-8250 shipped a month ago, has already been depended on by EIP-8266 (`requires: 8141, 8250`), and is now extended in place rather than via a new chain-of-superseding EIP. The pattern that emerges: sibling EIPs evolve as their adopters surface needs, base spec stays load-bearing-but-stable, and the `requires` chain absorbs each evolution without re-litigating dependents. This is the answer to a quiet Phase 17 open question (whether sibling EIPs would stay frozen or evolve) and it answers in favor of in-place evolution.

### ERC-20 Paymaster Question Resolved (External)

*matt, trebor — EthMagicians posts #158-#159, Jun 3-4*

trebor's Phase 15 question (Example 3 in the spec, ERC-20 paymaster feasibility under restrictive-tier rules) gets a direct reply from matt (post #158, Jun 3): the sponsor cannot check the user's ERC-20 balance on-chain in Example 3's restrictive-tier framing. The sponsor must verify the balance off-chain before approving the payment. The risk profile is the same as today's non-AA sponsoring systems, where the sponsor fronts the cost of data and initial validation and is therefore expected to be sophisticated about who they sponsor.

trebor acknowledges in post #159 (Jun 4) and commits to a clarifying PR: Example 3 currently states "Frame 1 'Checks that the user has enough ERC-20 tokens'" which is misleading because the check requires a state read forbidden by restrictive-tier rules. The corrected framing is that Frame 1 should be a signature check by the canonical paymaster (off-chain authorization), not an on-chain balance read.

The resolution is small but important. It is the first time an external researcher's misreading of an EIP-8141 example traced to a documentation bug (rather than a spec gap or a reviewer disagreement) and produced a commitment to a fix. It also resolves the structural concern in post #157 by example: where a verification design genuinely needs flexibility the restrictive tier disallows, the answer is off-chain authorization with a canonical-paymaster signature check, not a relaxation of the restrictive-tier rules.

**What carried into Phase 19**: the EIP-8250 evolution-in-place precedent shows the compose-by-requires camp treats sibling EIPs as living documents, which becomes important context when EIP-8272 merges and EIP-8288 opens within hours of each other in Phase 19.

---

## Phase 19: EIP-8272 Merge and Fourth Sibling EIP (Jun 5 – Jun 8)

*Phase 19 is a one-day cluster on Jun 5 with three-day trailing review activity. EIP-8272 merges three days under EIP-8266's submission-to-merge window. Within an hour of the merge, vbuterin and Thomas Coratger open PR #11772 introducing a fourth sibling EIP: a frame mode for post-quantum signature and STARK aggregation. The compose-by-requires baseline grows from three to four siblings in a single hour, while PR #11681 (absorb-into-base) still sits without reviewer signoffs since May 16.*

### EIP-8272 Recent Roots Merged

*soispoke, vbuterin, nerolation — PR #11726, opened May 25, merged Jun 5*

EIP-8272 lands eleven days after submission, three days faster than the EIP-8266 cycle (and four days faster than EIP-8250's). The diff is +394/-0 lines at `EIPS/eip-8272.md`. abcoathup's May 26 non-editor approval ("Looks good enough to merge as a draft") sat for nine days awaiting editor signoff; lightclient signed off as EIP editor on Jun 5 and the auto-merge fired minutes later. No substantive review-thread changes between submission and merge; the design held through to merge unchanged.

The merge confirms three architectural claims Phase 17 made about EIP-8272 in advance. First, the "pinned target address whose runtime is fixed at activation" pattern (now used by `ENTRY_POINT`, `EXPIRY_VERIFIER`, `NONCE_RING`, `NONCE_MANAGER`, and now `RECENT_ROOT_ADDRESS`) is the dominant deployment idiom for sibling-EIP system contracts. Second, the `RECENTROOTREFLOAD = 0xB4` opcode lands as the fifth opcode in the `0xB*` range, and the compose-by-requires pattern is now demonstrated as a general extension surface (siblings can add opcodes, top-level fields, and external dependencies). Third, the `requires: 7843, 8141` header bakes in the first non-8141 sibling-EIP dependency, putting EIP-7843 on the upstream-watchlist for any Hegotá-fork ordering discussion.

### Fourth Sibling EIP: Frame Type for PQ Sig and STARK Aggregation (Open)

*vbuterin (Vitalik Buterin), Thomas Coratger — PR #11772, opened Jun 5*

Within an hour of EIP-8272 merging, vbuterin opened PR #11772 with a fourth sibling EIP: a new frame mode that aggregates post-quantum signatures and STARK proofs at the block level via a single recursive STARK. The PR is +508 lines, the largest sibling-EIP submission so far. EIP number 8288 is reserved through the EthMagicians thread title ([topic 28723](https://ethereum-magicians.org/t/eip-frame-type-for-quantum-resistant-signature-and-stark-aggregation/28723)) but not yet committed to the file (placeholder `eip-9999.md` pending editor assignment).

The mechanism in three pieces:

1. **`DEP_VERIFY_FRAME_MODE = 3`**: a new frame mode that declares a set of dependencies as `(scheme, data_hash, verification_key)` triples (96 bytes each, up to `MAX_DEPENDENCIES_PER_FRAME = 256` per frame; `MAX_SIGS_PER_TX = 16`, `MAX_STARKS_PER_TX = 1`). Dependencies are not executed; they are recorded for the block-level recursive proof. The two schemes are `LEANSPHINCS_SCHEME = 0x10` (hash-based signature, 3000 gas) and `LEANSTARK_SCHEME = 0x11` (STARK, 30000 gas), reusing Lean Ethereum tooling to maximize shared codebase.
2. **Block-header `recursive_stark`**: a new top-level header field carrying `[stark_proof, block_deps_hash]`. Block validity requires the single recursive STARK proves all declared dependencies; `block_deps_hash` commits to the concatenation of all per-block dependency triples. `AGGREGATED_VK` is the fixed verifying key for the recursive scheme. Mempool wrappers bundle transactions with their dependencies and witnesses, with recursive aggregation supported at the mempool layer for bandwidth savings. Explicitly FOCIL-compatible (EIP-7805).
3. **EVM introspection**: dependencies are visible during execution via existing EIP-8141 `FRAMEPARAM` / `FRAMEDATASIZE` / `FRAMEDATACOPY` opcodes, so contracts can iterate dependency frames and read declared triples without new opcodes. The data format is the concatenation of `scheme || data_hash || verification_key` triples.

The strategic significance is the largest single-sibling step yet. EIP-8288 is the first sibling to add a new frame mode (a structural extension to EIP-8141's mode enum, not just an opcode or system contract). It is also the first to introduce a new top-level block-header field, putting sibling-EIP changes into consensus header territory. The `requires` header (`2718, 2929, 2930, 7702, 8141`) is the longest sibling dependency chain yet, dropping the EIP-8250 dependency that EIP-8266 and EIP-8272 both carry, which signals that EIP-8288's frame mode operates independently of the nonce-mechanism layer.

The authoring shift is also notable. Vitalik Buterin authors directly with Thomas Coratger (new contributor from the Lean Ethereum tooling side), bringing the sibling-EIP author roster beyond the soispoke/nerolation/lightclient cluster that shipped the first three siblings. The compose-by-requires camp now spans Buterin himself plus four collaborators across four siblings, all merged or in active review within one month.

All reviewers approved via the auto-bot path. abcoathup left two clarifying comments on Jun 8. No editor signoff yet; the bot indicates the merge is gated on editor approval.

### Compose-by-Requires Stack at Four Siblings (Architectural)

The Phase 17 question about whether a fourth sibling EIP would appear "to push the question from 'is this a pattern' to 'what is the upper bound on the compose-by-requires stack'" is answered the same day. EIP-8288 makes the count four (EIP-8250 merged May 11, EIP-8266 merged May 22, EIP-8272 merged Jun 5, EIP-8288 opened Jun 5), with three merged and one in active review.

Two observations on the trajectory. First, sibling EIPs are demonstrably extending every dimension of EIP-8141: payload schema (EIP-8250 reshape, EIP-8272 new field, EIP-8288 implied via dep frames), opcode space (EIP-8272 `RECENTROOTREFLOAD`), frame-mode enum (EIP-8288 `DEP_VERIFY_FRAME_MODE`), block header (EIP-8288 `recursive_stark`), system-contract storage (every sibling), and dependency graph (EIP-8272 pulls in EIP-7843; EIP-8288 pulls in EIP-7702 and EIP-7805). There is no part of the EIP-8141 surface that the sibling stack has not extended in the last month. The pattern's upper bound, if any, is not yet visible.

Second, PR #11681 (Pedro Gomes's absorb-into-base bundle) still sits without any reviewer signoffs since May 16. The bundle proposed folding three features (guarantors, keyed nonces, signer binding) into EIP-8141 itself. In the three weeks since, the compose-by-requires stack has shipped a key-set generalization (EIP-8250 evolution), the EIP-8272 recent-roots layer, and opened the EIP-8288 PQ aggregation frame mode. Absorbing the equivalent surface into EIP-8141 now would mean folding four sibling EIPs' worth of design surface into one base-spec amendment, with all four authoring clusters needing to coordinate. The empirical argument against absorb-into-base is no longer theoretical; the surface area has grown beyond what one bundle PR can credibly carry.

**What to watch into Phase 20**: whether EIP-8288 gathers editor signoff and merges (vbuterin direct authorship and abcoathup's same-day non-editor approval suggest a fast path); whether the `DEP_VERIFY_FRAME_MODE = 3` addition triggers a samwilsn or jochem-brouwer editorial review on the frame-mode-enum governance question (sibling EIPs now claiming both opcodes and frame modes raises whether there should be a coordinated assignment policy); whether the Lean Ethereum tooling dependency (`LEANSPHINCS_SCHEME`, `LEANSTARK_SCHEME`) gets a separate EIP for the schemes themselves; whether PR #11681 retracts, rebases, or stays open with no further activity; whether trebor lands the promised Example 3 clarification PR; and whether a fifth sibling EIP appears in the next two weeks.

