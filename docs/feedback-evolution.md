# How Feedback Evolved Over Time

---

The feedback on EIP-8141 arrived in distinct waves, each pushing the spec along a clear trajectory: **from expressive abstraction toward deployability, compatibility, and mempool safety**. The early drafts prioritized flexibility; the later drafts are about making that flexibility survivable for clients, wallets, and the p2p network.

This page tracks *why* the spec moved and *who* pushed back. The mechanics of each change (diffs, constants, opcode and system-contract internals) live in [Merged Changes](/merged-changes).

---

## Phase 1: Conceptual & Compatibility Scrutiny (Jan 29 – Feb 10)

*The thread tested whether the `APPROVE` mechanism and the frame abstraction held up under proxy-account, nested-call, and ecosystem-compatibility pressure. Core question: does this new opcode behave safely across the call-graph shapes real accounts use?*

### APPROVE Propagation Debate

*nlordell, frangio, fjl — EthMagicians posts #16-32*

The first major debate was how `APPROVE` interacts with nested call frames: does it auto-propagate through `RETURN`, can inner contracts accidentally approve, and how do proxy accounts (Safe) approve if `APPROVE` requires top-level invocation? The authors decided **against** auto-propagation, scoping `APPROVE` to the transaction with only `frame.target` able to call it. This was later relaxed (PR #11297) to permit `APPROVE` from nested calls so proxy-based accounts could adopt the spec.

### "Why Not Simpler Alternatives?"

*Helkomine — posts #6-14*

Helkomine argued command-oriented architectures like Uniswap's UniversalRouter could do the same job. matt (post #9) explained that frames exist for **protocol-level introspection**, letting the p2p layer reason about validity bounds. EIP-2938 had failed precisely because it lacked this structured introspection, making safe mempool relay rules impossible.

### EIP-3607 Compatibility

*thegaram33 — post #26, PR #11272*

Peter Garamvölgyi identified that EIP-3607 (rejecting transactions from senders with code) would block 8141's `SENDER` frames for smart accounts, and opened PR #11272 to carve frame transactions out. The PR stayed open well past this phase.

### APPROVE Security in Non-Account Contracts

*thegaram33 — posts #37-41*

thegaram33 worried any contract containing `APPROVE` could be used as a frame target, creating an alternative authorization path. fjl responded that `APPROVE` is just `RETURN` with extra semantics and only works when `frame.target == tx.sender`, which **reinforced** the decision to restrict `APPROVE` to `frame.target`.

**What changed**: `APPROVE` locked to transaction-scoped with a `frame.target` check, then relaxed via PR #11297 for proxy accounts. EIP-3607 compatibility opened as PR #11272.

---

## Phase 2: Adoption & UX Concerns (Feb – Mar 10)

*The thread pivoted from "does this work?" to "will anyone use it?" Monad's adoption data and Derek Chiang's commercial-AA experience reframed the question: if nearly all frame transactions will come from EOAs, the spec has to serve EOAs natively or lose to a simpler alternative.*

### EIP-7702 Adoption Data Battle

*DanielVF vs matt — posts #64-71*

DanielVF (Monad) presented data showing only ~0.28% of transactions were EIP-7702. matt countered the metric was wrong: most 7702 usage flows through entrypoint contracts and relayed actions, with BundleBear showing 4M+ operations/week. The disagreement surfaced a key insight: **measuring AA adoption requires counting operations, not raw transactions**.

### The Adoption Critique

*DanielVF — post #64*

DanielVF argued smart-contracts-as-asset-owners is the "adoption killer" (wallets are expensive and dangerous to build, users get vendor-locked, no interoperability); that EIP-1559 succeeded by being simple and standardized while 4337/7702 forced custom dangerous contracts; and that if 99%+ of frame txns are from EOAs, a simpler tx type would serve better. He noted 8141 didn't yet define the bytes a wallet sends to move 1 ETH. This directly drove the spec toward EOA support.

### Derek Chiang's Response and EOA Support

*derek — posts #60, #65*

Derek Chiang, with 3 years of commercial AA experience, largely agreed and proposed EOA support: EOAs get AA benefits (gas abstraction, sponsorship) without smart-contract risk, wallets integrate by supporting a tx type rather than auditing accounts, and advanced users keep the smart-account option. As he put it (post #62), it flips the default from "your EOA can't use AA unless you delegate" to "your EOA can use most AA by default." This led to PR #11379 (merged Mar 10).

**What changed**: EOA default code added via PR #11379 (Mar 10). The single biggest shift in the spec's trajectory, from "smart-account-assumed" to "EOA-first." Every downstream default-code decision followed from here.

---

## Phase 3: Operational Constraints & Mempool Safety (Mar 10 – Mar 25)

*With EOA support landed, attention shifted to making frame transactions safely propagatable and performant at the p2p layer. The async-execution critique, the atomic-batching debate, and the mempool policy dominated, all operational questions about whether clients and builders can ship this.*

### Async Execution Incompatibility

*DanielVF, pdobacz — posts #53, #79, #119-123*

The most structural criticism: frame transactions need EVM execution to determine inclusion validity, which is fundamentally incompatible with async-execution models (Monad, and potentially Ethereum via EIP-7886), where traditional transactions need only three state reads. Monad called frame txns "mutually incompatible" with their model and Base drafted EIP-8130 as an alternative. The authors argued it is manageable via FOCIL, the 100k validation-gas cap, and the bounded `VERIFY` structure.

### Atomic Batching Debate

*pedrouid, 0xrcinus, derekchiang, frangio — posts #72-89, PR #11395*

pedrouid argued SENDER frames should always be atomic; derekchiang explained why non-atomic is the better default (more general OR-logic; a paymaster using VERIFY needs the ERC-20 transfer not to revert because a later op reverts). frangio (post #73) named the ERC-20-for-gas case as the main non-atomic use. 0xrcinus proposed explicit group IDs; the final design used a bit-flag atomic-batch marker on consecutive SENDER frames.

### P256 Scope Creep

*shemnon, frangio — posts #78, #94-99*

Danno Ferrin flagged P256 in the default code as "significant scope creep," warning (post #99) that 8141 risked becoming "a kitchen sink EIP." frangio noted P256/passkey accounts can't rotate keys. The authors kept P256 but acknowledged it creates accounts that can't migrate to PQ schemes without further EIPs.

### PQ Signature Aggregation Path

*fjl — post #23*

fjl noted the VERIFY design deliberately enables future **signature aggregation**: because VERIFY frames can't change execution outcomes and their data is elided from the sighash, a builder could one day strip them and replace them with a single succinct validity proof. This is central to the long-term PQ strategy, since PQ signatures are large.

### Mempool Rules as Turning Point

*lightclient — post #112, PR #11415*

lightclient's mempool policy PR was a turning point (DanielVF: "a big big step forwards"). It introduced the canonical paymaster (verified by code match, replacing ERC-7562 reputation/staking), a validation prefix subject to mempool rules, a capped validation gas (`MAX_VERIFY_GAS = 100k`), and four recognized validation-prefix templates. Competing proposals (EIP-8175, EIP-8130, Tempo) emerged here and are tracked in [Competing Standards](/competing-standards).

**What changed**: atomic-batch flag (PR #11395) and comprehensive mempool policy (PR #11415) merged Mar 25, turning EIP-8141 into something clients could begin implementing.

---

## Phase 4: Forward-Compatibility Extensions & Open Gaps (Mar 26 – Apr 10)

*With the core spec landed, the thread moved into extension territory: forward-compat hooks for PQ aggregation and precompile verification, plus a steady stream of open gaps that needed decisions before the spec could be called complete.*

### VALUE in SENDER Frames

*rmeissner, DanielVF, frangio, 0xrcinus, derek, matt — posts #124-134*

rmeissner (Safe) identified that SENDER frames had no `value`, blocking native ETH transfers without custom execute methods. Strong consensus formed: DanielVF warned that without value, frames become "dumb message pipes"; rmeissner preferred a value field plus the atomic flag over a `DELEGATECALL`-precompile alternative, citing easier formal verification. matt (post #134) confirmed the authors now support value in frames given atomic batching exists, signaling the field would be added.

### Signature Aggregation Forward-Compatibility

*lightclient — PR #11481, Apr 2*

lightclient proposed a `signatures` field on the outer transaction for PQ forward-compatibility: signatures verified before frame execution so frames just check authority, with a future path to block-level aggregated witnesses eliding individual signatures. The most structurally significant open proposal, since it changes the transaction format itself.

### Precompile-Based Verification

*derekchiang — PR #11482, Apr 2*

derek proposed letting VERIFY frames target precompiles directly, extending default-code verification to contract accounts and enabling key rotation (the precompile reads a stored key commitment) and shared verification logic.

### EOA + EIP-7702 Delegation Compatibility

*DanielVF — posts #120, #122*

DanielVF flagged that an EOA with 7702-delegated code can't use signature-based authorization with frames, because the default-code path isn't invoked but the delegated contract may not implement `APPROVE`.

### Async Execution Compatibility (Continued)

*DanielVF, derek — posts #121-123*

The async thread continued; derek asked for resources (ethresearch, EIP-7886) to keep frame transactions compatible with a potential async-execution future.

### Spec Consistency Fixes

*node.cm, chiranjeev13 — posts #135-136, PR #11488*

node.cm noted VERIFY frames are implicitly capped at 2 per transaction (only two approval flags exist) and asked for it to be explicit. chiranjeev13 opened PR #11488 to add the static count check, fix stale APPROVE scope values, and allow any EOA as paymaster.

### EIP-3607 Compatibility Status Update

*lightclient — PR #11272, Apr 8*

lightclient's earlier review on the EIP-3607 carve-out PR was dismissed Apr 8 as the spec moved; the interaction stayed unresolved.

### Signature Index Discovery Problem

*derekchiang — PR #11481 comment, Apr 9*

derek raised that contracts using outer signatures can't know which list index is theirs, forcing the default code to loop the whole list. An ergonomic and gas weakness to address before finalizing.

### Frame Return Data Access

*jacopo-eth — post #137, Apr 10*

Jacopo proposed `FRAMERETURNDATASIZE`/`FRAMERETURNDATACOPY` opcodes so frame returndata can feed multi-step flows without wrapper contracts. No author response yet.

**What changed**: consensus built around a per-frame `value` field; PR #11481 (sig aggregation) and PR #11482 (precompile VERIFY) entered review; PR #11488 opened. No merges landed; this phase set up the pivot that followed.

---

## Phase 5: Sibling EIPs and Broad Spec Tightening (Apr 11 – Apr 15)

*Ben Adams produced three PRs in five days: two narrower sibling EIPs (EIP-8223 static sponsorship, EIP-8224 shielded gas funding) and a tightening PR that restructured the spec's internal consistency. The thread shifted from "proposals and debates" to "structural changes landing."*

### Contract Payer Transaction (EIP-8223)

*benaadams — PR #11509, Apr 11*

Ben Adams submitted EIP-8223, a narrow sponsored-transaction proposal charging gas to `tx.to` via a canonical payer registry, validated with one SLOAD and a balance check and no EVM execution (FOCIL/VOPS-compatible). The PR positions it as complementary to EIP-8141 and EIP-8175, covering the static-validation case while frame-based proposals handle the general case.

### Counterfactual Transaction (EIP-8224)

*benaadams — PR #11518, Apr 12*

A day later Ben Adams submitted EIP-8224, addressing the bootstrap problem EIP-8223 leaves: a fresh EOA with no ETH can't pay gas privately. EIP-8224 carries a ZK proof that the sender owns an unspent fee note in a canonical contract, validated with bounded crypto and fixed storage reads, no EVM execution. Intended composition: one-shot bootstrap via EIP-8224, then cheap sponsored transactions via EIP-8223, with EIP-8141 forming a layered AA stack.

### Validation Frame Ordering Within a Block

*fvictorio — post #138, Apr 13*

Franco Victorio asked whether validation frames execute first within a block (analogous to 4337's validation/execution split). A block-level scheduling question, unanswered in the thread.

### Broad Spec Tightening (Merged)

*benaadams — PR #11521, submitted Apr 13, merged Apr 14*

Ben Adams's tightening PR consolidated several open threads: splitting `mode`/`flags`, adding `FRAMEPARAM` (the fifth opcode), hardening the default secp256k1/P256 paths, reducing `MAX_FRAMES` from 1000 to 64, adding per-frame gas costs, locking deployment to EIP-7997, and strengthening VERIFY-malleability and DELEGATECALL+APPROVE warnings. lightclient and derekchiang approved; fjl questioned the `MAX_FRAMES` cut but benaadams argued it is easier to raise later after measurement. The broadest restructuring since the approval-bits PR #11401.

### Bytecodes in VOPS Proposal

*derekchiang — ethresear.ch post #12, Apr 15*

derek proposed adding contract bytecodes to the VOPS baseline (~10.55 GB, roughly doubling VOPS but well below full state), resolving the delegate-bytecode availability gap for AA-VOPS nodes without new opcodes or rent. Routed into [VOPS state growth](/vops-compatibility#state-growth-at-scale).

**What changed**: PR #11521 merged Apr 14 (mode/flags split, `FRAMEPARAM`, `MAX_FRAMES` = 64, per-frame gas, default-code hardening). EIP-8223 and EIP-8224 submitted as complementary siblings. Bytecodes-in-VOPS reframing surfaced.

---

## Phase 6: Value Field and Privacy/Delegation Threads (Apr 16 – Apr 19)

*The pending `value` consensus landed and Nero_eth's three-gates analysis began shaping the next wave around privacy-pool flows. Forum debate reopened the default-code-vs-7702 interaction. The Hegotá CFI PR opened here but merged Apr 30 (recorded in Phase 7).*

### Per-Frame Value (Merged)

*lightclient — PR #11534, submitted and merged Apr 16*

lightclient merged the long-requested `value` field two days after PR #11521, resolving the consensus across posts #124-134. The PR captures the reversal: the authors originally resisted frame-level value because user operations were expected to be account-handled, but with frames now aimed at a good out-of-the-box experience, native `value` became "a critical field for the SENDER frame." Non-zero value is restricted to SENDER frames so VERIFY stays `STATICCALL`-like. Auto-merged after all reviewers approved, no debate on the diff.

### Three Gates to Privacy

*Nero_eth — [ethresear.ch](https://ethresear.ch/t/frame-transactions-and-the-three-gates-to-privacy/24666), Apr 16*

Nero_eth framed privacy-pool inclusion as a three-gate problem: the public mempool (100k VERIFY cap rejects ~250k-gas Groth16), FOCIL enforcement, and VOPS/AA-VOPS node validation. The useful observation for EIP-8141: frames structurally remove relayer trust, because invalid or replayed proofs revert in VERIFY before gas is charged, so a sponsor can be paid from the withdrawn amount with zero trust. Five protocol changes proposed, with an acknowledged tradeoff that attesters absorb up to ~28% block gas worst-case. Routed into [Mempool Strategy → Privacy Pools](/mempool-strategy#privacy-pools-three-gates).

### Value Field Announced on Forum

*derek (post #139), DanielVF (post #140), Apr 17*

derek announced the value-field merge; DanielVF welcomed it and named two remaining priorities before the spec is production-ready: making the default signature path an explicit opt-in (to future-proof against sig-scheme changes), and making atomic batching practically usable.

### Default Code vs 7702 Delegation Interaction

*DanielVF, derek, alex-forshtat-tbk — posts #141-145, Apr 17-19*

DanielVF noted a 7702-delegated EOA can today choose per-transaction whether to invoke its delegation, but under 8141's "if there's code, use the code" rule the delegation always wins. He argued for restoring explicit opt-in via a flag byte in `frame.data`. forshtat observed the existing `signature_type` first byte already acts as an EOA flag and proposed extending it with a "use 7702 code" value. No PR yet.

**What changed**: per-frame `value` merged (PR #11534). EIP-8141 submitted to the Hegotá CFI list via PR #11537 (merged Apr 30, Phase 7). Nero_eth's three-gates analysis opened the privacy-pool discussion wave.

---

## Phase 7: Mempool Risk-Shifting and Svalbard Ratification (Apr 22 – Apr 27)

*A same-day pairing on Apr 22 (a small sighash fix, and the first of derekchiang's mempool-risk proposals) opens the phase; five days later the Svalbard interop ratifies the constraint set EIP-8141 has been converging toward. Net spec impact in Phase 7 itself: the sighash type-byte fix.*

### Transaction-Type Sighash Fix (Merged)

*derekchiang — PR #11544, submitted Apr 18, merged Apr 22*

A 1-line fix closing a cross-type signature replay weakness: `compute_sig_hash` now prefixes the `FRAME_TX_TYPE` byte before RLP, matching the EIP-2718 convention. Auto-merged within hours, no debate.

### Guarantors Proposal

*derekchiang — PR #11555, Apr 22*

derek opened an early proposal for a "guarantor" payer that covers gas even if sender validation fails. With a guarantor present, mempool nodes can skip sender-validation simulation, so the sender's VERIFY frame may read shared state (ERC-20 balances, environmental opcodes) and still propagate publicly; the guarantor absorbs the cost if validation would have failed. This opens a third path for ERC-20 gas repayment by moving the shared-state-read problem from a mempool-policy violation into an economic-risk problem.

Why this matters for statelessness: the restrictive tier bans shared-state reads during validation for VOPS compatibility. Guarantors route around that because the guarantor's commitment is itself a paymaster-like signature check inside every node's slice. The VOPS invariant is preserved; the economic risk shifts from the protocol to the guarantor. **What to watch**: whether guarantors gather author consensus, how third-party guarantor markets price the risk, and the interaction with VOPS/FOCIL.

### Svalbard Interop AA Breakout

*Hosts: Matt and Felix — [breakout notes](https://hackmd.io/@nixorokish/svalbard-aa-breakout), Apr 27*

A native-AA breakout (attended by Vitalik and Potuz alongside the wallet-side hosts) produced a constraint-and-goals framework that reads as external ratification of EIP-8141's design line rather than a new direction.

- **Primary goals**: alternative signature schemes (multi-sig, PQ, R1), protocol-level call batching, key management and recovery, post-transaction assertions, and **no relayers for core functionality**.
- **Stretch goals**: unified EOA/smart-account default treatment, ETH-denominated sponsorship (**ERC-20 explicitly deferred** to relayer paths), flexible nonces, and a cross-chain keystore wallet.
- **Hard constraints**: walk-away test, statelessness, ZK-EVM compatibility (hash-based PQ up front), public-mempool admissibility, FOCIL compatibility.
- **Open threads**: PQ starts with hash-based signatures (lattice needs work; Vitalik sketched a vector-math precompile); validation/execution separation gives bounded cost; Potuz raised L2 sequencer DoS via invalid txns; the canonical privacy-pool handler stays undefined (the gap Nero_eth named Apr 16); ERC-20 sponsorship deferred to relayer paths.

**Why this matters**: the primary-goal list maps cleanly onto features EIP-8141 already specifies or has open PRs for, and the hard constraints are the same ones driving the restrictive-tier mempool policy and the guarantors/payer-before-sender debates. Action items included specifying a canonical privacy-pool handler, continuing PQ aggregation, and evaluating competing EIPs against the framework.

---

## Phase 8: Payer Ordering, Default Code Cleanup, and Hegotá CFI (Apr 28 – Apr 30)

*The implementation tail of Phase 7's design debate. lightclient floats a simpler framing of guarantors (just let the payer approve before the sender); a default-code cleanup completes the multi-call transition; and the phase closes Apr 30 with the factory relaxation and Hegotá CFI both landing. The guarantors-vs-ordering choice carries into Phase 9.*

### Payer-Before-Sender Alternative to Guarantors

*lightclient — PRs #11575, #11579, #11580, Apr 28-29*

lightclient floated a simpler framing than derekchiang's guarantors (#11555): rather than a new role, just relax the ordering so the payer can `APPROVE_PAYMENT` before the sender, absorbing the same economic risk. PR #11575 auto-merged by mistake on Apr 28, was reverted by #11579 the next day, and reopened as draft #11580. Net spec impact: zero. The open question for Phase 9 is which framing (#11555 guarantors or #11580 ordering relaxation) gathers consensus.

### RLP Call Batch Removed from Default Code (Merged)

*lightclient — PR #11577, merged Apr 29*

A small cleanup completing the move of multi-call out of the default-code payload and into the frame list: default-code `SENDER` mode no longer decodes an RLP call batch, since atomic frame batching (#11395) and per-frame value (#11534) cover the use cases. Auto-merged, no debate.

### Deploy-Frame Factory Relaxation (Merged)

*derekchiang — PR #11567, opened Apr 24, merged Apr 30*

derek's second structural mempool proposal dropped the hard-coded EIP-7997 factory requirement for `deploy` frames, both as a `requires` entry and as the only valid target. Any contract can now be the factory provided the deploy frame still satisfies the stateless validation-trace rules, which the PR reifies directly as a carve-out. The actual safety property the restrictive tier needs is that deploy-frame outcome is independent of state outside `tx.sender`; EIP-7997 was a convenience dependency, not a safety one. lightclient approved with "SGTM"; auto-merged same day.

This is the broadest mempool-policy change since PR #11415 and the first to retract a `requires` entry. It also blurs the line between smart-account deployment and 7702-delegation installation, since both now flow through the same primitive. Open follow-ups: interaction with PR #11482's precompile-targeting VERIFY frames, and composition with the Phase 6 default-code-vs-7702 thread.

### Hegotá CFI Inclusion (Merged)

*dionysuzx — PR #11537, opened Apr 17, merged Apr 30*

The fork-meta PR waiting on a reviewer since Phase 6 merged after ralexstokes approved: EIP-8141 added to `Considered for Inclusion` in `eip-8081.md`, formalizing decisions captured at ACDE #233 and ACDC #177 two weeks earlier. Movement to PFI/SFI requires further client-readiness signals, not another spec PR.

---

## Phase 9: Keyed Nonces and Parallel Sequences (Apr 30 – May 5)

*A new design line opens: lifting EIP-8141's single linear sender nonce into a `(nonce_key, nonce_seq)` pair so one sender can run independent sequences in parallel. Motivating use cases are privacy protocols sharing one onchain sender, smart-wallet session keys, and relayer-style senders. The work arrives as a delta sketch (#11584) and a standalone Standards Track EIP (#11598).*

### 2D Nonces Sketch (Closed in Favor of Standalone EIP)

*nerolation — PR #11584, opened Apr 30, closed May 8*

Toni Wahrstätter opened a 28-line sketch replacing the single sender nonce with `(nonce_key, nonce_seq)`, per-key sequences running independently with `nonce_key = 0` as the legacy slot. He closed it May 8 ("Closing in favor of the EIP for now") once the standalone Keyed Nonces EIP (#11598) gathered the same idea with concrete `NONCE_MANAGER` semantics. The delta-against-8141 framing is abandoned.

### EIP-8250: Keyed Nonces for Frame Transactions (Opened in This Phase, Merged Phase 12)

*soispoke, nerolation, lightclient, vbuterin — PR #11598, opened May 4*

Four days after the sketch, the idea returned as standalone EIP-8250 (opened May 4, merged May 11; see Phase 12). The standalone framing keeps the parallel-sequence motivation and adds the implementation detail the sketch deferred: a `NONCE_MANAGER` system contract holding non-zero keys, a full `uint256` `nonce_key` so privacy protocols can derive keys from nullifiers, and a first-use gas surcharge. The load-bearing property: nonce consumption is journaled as a payment-approval effect outside the frame's revert journal and atomic-batch snapshots, so a single-use key is atomically spent the moment payment is approved. The security section flags that keyed transactions don't advance the legacy account nonce, so `CREATE` addresses can shift; applications must use `CREATE2` or authenticate the legacy nonce explicitly.

### Atomic Batching Scope: DEFAULT Frames and the VERIFY Carve-Out (Open)

*alex-forshtat-tbk and derek — posts #146, #147, May 5*

Forshtat asked why atomic batching is limited to SENDER frames, since DEFAULT frames may want post-transaction cleanup hooks. derek replied that VERIFY was excluded because a later frame could revert its outcome and break mempool validation, but saw no protocol reason to exclude DEFAULT, and proposed allowing *any* frame (including VERIFY) to batch at the protocol level with the restrictive mempool tier separately forbidding VERIFY-in-batch. The same protocol-vs-mempool layering PR #11580 used. It reframes the design space: atomic batching as a universal protocol primitive, with mempool policy carving out the validation-time hazards.

**What to watch**: which of #11584 and #11598 the authors converge on; whether the first-use surcharge survives review; how the spend-once-on-payment property composes with guarantors (#11555) and payer-before-sender (#11580); and whether atomic batching opens to DEFAULT and VERIFY frames at the protocol level.

---

## Phase 10: EIP-3607 Carve-Out and EVVM External Perspective (May 5 – May 7)

*Phase 10 closes a long-pending issue and brings an external production data point onto the thread. Thegaram's PR #11272 (open since Feb 6) finally lands the EIP-3607 carve-out; the same week, EVVM's co-founder contributes a contract-vs-protocol comparison.*

### EIP-3607 Carve-Out for Frame Transactions (Merged)

*Thegaram — PR #11272, opened Feb 6, merged May 5*

The longest-pending open spec PR landed. EIP-3607 forbids transactions whose sender has non-empty non-delegation code, which would block SENDER frames originating from contract accounts. The fix adds `3607` to `requires` and a "Transaction origination" subsection documenting the carve-out. Raised on day one ([post #26](https://ethereum-magicians.org/t/eip-8141-frame-transaction/27617/26)), it sat through the Phase 5-9 churn; lightclient re-approved May 5 after the diff was refreshed. Small (+7/-1) but clean: EIP-3607 becomes the first cross-EIP requirement EIP-8141 explicitly opts out of, stated in spec text rather than inferred.

### EVVM Production Perspective on Frame vs Contract-Layer AA (External)

*ariutokintumi — post #148, May 7*

German Abal (co-founder of EVVM, a contract-native AA framework deployed across ~200 instances since 2023) read the full thread and posted a four-paragraph comparison against EVVM's production experience. Three observations:

1. **Per-environment policy lives at the contract layer, not the protocol.** EVVM instances configure KYC/AML gates and custom guards per deployment; real adopters (banks, regulated DeFi) need this variance. EIP-8141 correctly doesn't support it natively, but the rationale should say so explicitly.
2. **Async-execution compatibility is achievable as a contract-level property.** EVVM-style validation runs inside the contract at execution time against post-state, so async execution and contract-level AA coexist. A reference point for chains where 8141 can't ship.
3. **Two EIP-8141 choices differ from EVVM in production**: per-operation success in batches (EVVM's `batchPay` returns `bool[]`; 8141's atomic batching is all-or-nothing), and reservation primitives (EVVM ships only non-authoritative reservations to avoid DoS-by-lock-grabbing, which likely matters at protocol level if keyed nonces ship).

The post proposes no competing alternative. It's a load-bearing external data point: the atomic-batching observation lines up with derek's May 5 reframing (post #147), and the reservation observation is one to flag against EIP-8250's commit-on-payment semantics, since a single-use key is a narrow authoritative reservation.

---

## Phase 11: Editorial Review and Layering Pattern (May 7 – May 10)

*Bracketed by lightclient's PR #11621 (frames cleanup, opened May 7) but editorial in character. samwilsn raises naming consistency and a `FRAMEDATACOPY` revert-vs-zero-pad design question; forshtat extends the protocol-vs-mempool layering pattern to the `SSTORE`-in-`VERIFY` ban. The cleanup PR itself lands in Phase 12.*

### Editor Review and Spec Coherence Questions (External)

*samwilsn — post #149, May 8*

Sam Wilson posted an editorial review focused on naming and minor gaps rather than structure: empty-target representation (`None` vs `b""`), an `APPROVE_PAYMENT_AND_EXECUTION` name that reads in the wrong scope-evaluation order, an undefined "paymaster frame" term, and whether all five new opcodes justify their permanent opcode-space cost.

The most substantive question is `FRAMEDATACOPY` revert behavior: `CALLDATACOPY` zero-pads on out-of-bounds reads, so why does `FRAMEDATACOPY` revert? Reverting catches integer-arithmetic bugs at the cost of forcing contracts to know exact frame-data sizes; zero-padding is more forgiving but silently absorbs miscalculations. Not formally answered, but in practice the per-frame data regions are typed and well-known, so the revert reads as a typed-read assertion. One to track if PR #11488 folds into the cleanup.

### Layering Pattern Extends to VERIFY Restrictions (External)

*alex-forshtat-tbk — post #150, May 10*

Forshtat returned to the protocol-vs-mempool layering thread (posts #146-147, PR #11580) and asked whether the `SSTORE`-in-`VERIFY` ban belongs in the protocol or in mempool policy. The current spec bans it at the protocol level because storage writes in validation make the mempool's "did this validate?" decision state-dependent, breaking cheap parallel validation. Forshtat accepts the mempool problem but argues that if the restriction lived in mempool policy, a permissive tier or direct-to-builder path could still admit such transactions where the guarantees aren't needed. The same pattern derek articulated for atomic batching (post #147) and lightclient encoded in PR #11580. Unanswered, and flagged as the trailing concern of Phase 11: if subsequent PRs migrate the ban into mempool policy, `current-spec.md` and `mempool-strategy.md` need a paired update.

---

## Phase 12: Cleanup, Keyed Nonces, and the Extended Feature Set Bundle (May 11)

*A single-day cluster on May 11. Two Phase 9-11 design lines merge within minutes (PR #11621 frames cleanup, PR #11598 EIP-8250 Keyed Nonces); hours later Pedro Gomes opens PR #11643, an "extended feature set" bundle absorbing four features into EIP-8141 itself, the inverse of EIP-8250's requires-chain layering. The compose-by-requires vs absorb-into-base question is what carries forward.*

### Frames Cleanup Refactor (Merged)

*lightclient — PR #11621, opened May 7, merged May 11*

The readability sweep that opened Phase 11 merged May 11; samwilsn's review (post #149) was treated as follow-up rather than gating. The net -160-line diff is the largest spec-text refactor since PR #11521. Beyond restructuring the spec body, two landed changes deserve continued attention:

1. **P256 removed from default code** retracts the hardware-wallet/passkey bridge that was the headline EOA-support story since PR #11379. The PR doesn't justify it, so it's unclear whether this is deliberate scope-narrowing (P256 belongs in EIP-7932 or a per-account extension) or an unintended cleanup consequence.
2. **Default code now accepts SENDER and DEFAULT frames** rather than reverting, so a native ETH transfer to a fresh EOA via a frame transaction now succeeds where it previously reverted. Small in implementation but visible to wallets, indexers, and explorers.

### EIP-8250: Keyed Nonces for Frame Transactions (Merged)

*soispoke, nerolation, lightclient, vbuterin — PR #11598, opened May 4, merged May 11*

EIP-8250 merged minutes after PR #11621; abcoathup's May 6 non-editor approval sat five days awaiting lightclient's editor signoff. The significance is governance-structural: EIP-8250 is the **first EIP whose `requires` header includes EIP-8141**, making the pair the first compose-by-requires AA stack in the EIP series. The mempool one-pending-per-sender rule lives in EIP-8141, the parallel-sequence primitive in EIP-8250, and a future keyed-aware policy can compose them without re-litigating EIP-8141's payload schema. The protocol-vs-mempool layering from posts #146-147 and #150, lifted to an EIP-series convention.

### Extended Feature Set Proposal (Opened, Later Superseded)

*pedrouid — PR #11643, opened May 11 (closed May 18 in favor of [PR #11681](#phase-14-extended-feature-set-supersession-may-16-may-18))*

Eight hours after #11598 merged, Pedro Gomes opened PR #11643 with the inverse packaging: fold guarantors, flexible nonces, signer binding, and envelope expiry into EIP-8141 itself rather than chain them as siblings, arguing a bundled upgrade is more efficient than several sibling EIPs with overlapping system contracts. The envelope-expiry portion overlapped PR #11662 (EXPIRY_VERIFIER), which merged May 14 and shipped expiry as a verifier-frame contract instead. With that settled, the envelope-expiry component became redundant and Pedro closed #11643 on May 18 in favor of PR #11681 (Phase 14).

---

## Phase 13: Atomic-Batch Expansion and Expiry Verifier Frame (May 12 – May 14)

*Two more spec changes, each resolving an earlier thread. derek's PR #11652 extends atomic batching from SENDER-only to any frame mode, encoding the protocol-vs-mempool layering pattern at the frame-mode level. nerolation's PR #11662 adds an EXPIRY_VERIFIER frame, the first new frame shape since the Jan 29 design and the first carve-out from the uniform VERIFY-frame rules.*

### Atomic Batching Extended to All Frame Modes (Merged)

*derekchiang — PR #11652, opened and merged May 12*

derek's same-day PR extends atomic batching to any frame mode, with the mempool validation-prefix carving out atomic-batch frames separately. lightclient approved within 30 minutes; credited to forshtat (post #146). The supporting thread (posts #151-154) established the invariant any extension must preserve: removing VERIFY frames from a transaction must not change observable behavior (so builders can aggregate signature verification), and letting VERIFY frames participate in atomic batches would break that, since a later frame's revert could roll back the verification. The spec now encodes the layering pattern at the frame-mode level: atomic batching is a universal protocol primitive, with the restrictive tier carving out validation-prefix batches.

### EXPIRY_VERIFIER Frame Added (Merged)

*nerolation — PR #11662, opened May 13, merged May 14*

The first new frame shape since the original design: a `VERIFY` frame targeting a pinned canonical address (`address(0x8141)`) whose `frame.data` is an 8-byte deadline, reverting unless `block.timestamp <= expiry`. lightclient approved enthusiastically; auto-merged same day. The frame breaks three previously-uniform rules in narrow ways:

1. **Signature hash**: the deadline is *not* elided (every other VERIFY frame's data is, per PR #11205), because it's a sender-authored commitment that must not be malleable in transit.
2. **`TIMESTAMP` ban**: banned in validation generally, carved out for this canonical runtime; clients may skip EVM execution and check natively.
3. **`APPROVE` requirement**: relaxed from "VERIFY must call APPROVE" to "if the frame reverts, the transaction is invalid," so an expiry frame succeeds without APPROVE.

The "pinned address with a runtime fixed at activation" pattern (after `ENTRY_POINT`, EIP-4788, EIP-2935) becomes the second protocol codepath inside EIP-8141 and the template later siblings follow. One open question that didn't gate merge: whether the runtime reads `TIMESTAMP` (current draft) or the block header directly.

### Aggregation and APPROVE Questions Surface (External)

*alex-forshtat-tbk — post #155, May 14*

The day #11662 merged, forshtat asked for more detail on VERIFY-frame aggregation methodology and whether the state-modification constraints (the `SSTORE` ban, derek's aggregation invariant in post #152) should affect `APPROVE`'s behavior. Unanswered, and the trailing concern: `APPROVE` is the only protocol-defined VERIFY write, so aggregation pressure could push for follow-up restrictions on its semantics.

**What carried into Phase 14**: Pedro's bundle got reviewer engagement as a self-issued rewrite (PR #11643 closed, replaced by #11681 minus the envelope-expiry field). The compose-by-requires vs absorb-into-base question remains open, alongside the `SSTORE`-in-`VERIFY` layering question (#150), the aggregation/APPROVE thread (#155), and samwilsn's editorial items (#149).

---

<span id="phase-14-extended-feature-set-supersession-may-16-may-18"></span>

## Phase 14: Extended Feature Set Supersession (May 16 – May 18)

*The resolution of the architectural question Phase 12 left dangling. PR #11662 (EXPIRY_VERIFIER, merged May 14) settled the envelope-expiry component of Pedro's bundle independently. On May 16 Pedro opened a successor, PR #11681, retaining guarantors, keyed nonces, and signer binding but dropping the now-redundant expiry field; on May 18 he closed #11643 in favor of it. The absorb-into-base position is unchanged.*

### Extended Feature Set Successor Opened (Open)

*pedrouid — PR #11681, opened May 16*

Pedro opened PR #11681 as the successor to #11643 (+810/-74), keeping the same architectural position (keyed nonces and guarantors should ship together inside EIP-8141, sharing one system contract, rather than as a requires-chain) but dropping the envelope-expiry field PR #11662 made redundant. The three bundled features:

1. **Guarantors**: adopted from PR #11555 verbatim. The mempool tier that today rejects shared-state-reading sender validation can admit those transactions when a guarantor signature carries the risk.
2. **Keyed Nonces**: mirrors EIP-8250's parallel-sequence semantics, but with a single `uint64 signer` envelope field that indexes both the keyed nonce stream and the registered pubkey for signer binding. Argued to belong inside EIP-8141 because the upgrade path is more efficient when these features share one system contract (`AUTH_MANAGER`).
3. **Signer Binding**: a transaction-scoped `verified_signers` table populated by non-secp256k1 VERIFY frames, consulted by `ECRECOVER` on the hit path with the miss path byte-identical to upstream.

The tension with EIP-8250 is unchanged: EIP-8250's `requires` header established compose-by-requires at the EIP-series level, while #11681 takes the absorb-into-base position that one bundled EIP is a more efficient upgrade path. If #11681 lands, it supersedes EIP-8250 by absorption. Which packaging convention wins is the open question for Phase 15.

### PR #11643 Closed in Favor of #11681 (Closed)

*pedrouid — PR #11643, closed May 18*

Pedro closed #11643 ("Closed in favor of #11681") two days after opening the successor. Net spec impact zero; the four-feature bundle becomes a three-feature bundle without re-litigating the bundle-vs-requires-chain question. A small governance signal: in-flight PRs do rebase against the post-merge spec when external events settle one of their components.

**What carried into Phase 15**: the compose-by-requires vs absorb-into-base question, with both sides represented by open PRs. The next phase opens with a second sibling EIP taking the opposite position from #11681's bundle.

---

## Phase 15: EIP-8266 Opens, Soispoke Review, and Trebor's Paymaster Questions (May 19 – May 21)

*The architectural question gets a fresh data point from the opposite direction. On May 19 nerolation and lightclient open PR #11692, a second sibling EIP requiring EIP-8141 (Expiring Nonces); soispoke reviews it within hours and catches two correctness bugs, and abcoathup assigns EIP-8266 on May 20. On May 21 trebor reopens the feasibility of EIP-8141's Example 3 (ERC-20 paymaster) under restrictive-tier rules.*

### Expiring Nonces Sibling EIP Opened (Open)

*nerolation, lightclient — PR #11692, opened May 19*

PR #11692 layers an expiring-nonce mode on EIP-8141, trading unbounded per-tx state growth for a fixed-capacity ring buffer and reusing existing primitives rather than adding envelope fields: a sentinel `nonce` value selects the mode (no schema change), a `NONCE_RING` system contract holds the ring, and the deadline reuses PR #11662's `EXPIRY_VERIFIER` frame. The architectural significance: #11692 stakes the **opposite position from PR #11681**, shipping a nonce-mechanism alternative as a second sibling EIP rather than folding it into the base. The two open PRs now encode the same question from opposite ends. The composition with EIP-8250 is explicitly addressed (the sentinel collapses into a reserved keyed-nonce value), signaling the compose-by-requires camp expects siblings to compose with each other, not just with the base.

### Soispoke Review and Same-Block Replay Fix (Review)

*soispoke, nerolation — PR #11692 inline review, May 19*

Within six hours, soispoke (lead author of EIP-8250) left a five-comment review, a direct test of compose-by-requires in practice: the author of the first sibling reviewing the second on the day it lands. The substantive items:

1. **Same-block replay window**: the stateful-validity check used a strict inequality that let a second same-block inclusion pass. nerolation acknowledged and committed a fix.
2. **Pricing reframe**: soispoke argued the flat charge underprices fresh storage writes; nerolation reframed the justification around paired set+clear keeping trie leaves invariant rather than slot-reuse. The number stayed.
3. **Privacy framing walked back**: the replay key is the sig hash, which only prevents same-hash replay, not double-spending the same private note. nerolation agreed expiring nonces are "duplicate-hash replay protection, not single-use spend protection." A retracted motivation.
4. **Mempool rule**: the one-pending-per-sender bound is intentionally dropped for this mode (mass-invalidation isn't a problem here), keeping only payer-balance reservation.

soispoke engages as a peer sibling, not a competitor, and doesn't relitigate whether expiring nonces should be a sibling EIP at all, exactly what compose-by-requires presupposes.

### Number Assignment and Discussions Topic (Procedural)

*abcoathup — PR #11692 comment, May 20*

abcoathup assigned EIP **8266** and requested a separate EthMagicians thread, which nerolation created at [topic 28575](https://ethereum-magicians.org/t/eip-8266-expiring-nonces-for-frame-transactions/28575). The "frame-transaction thread carries everything" pattern is over; from here, sibling-EIP discussion fragments across per-EIP threads.

### Trebor Reopens Example 3 (ERC-20 Paymaster) Feasibility

*trebor — posts #156-157, May 21*

trebor (Kohaku privacy-paymaster researcher) noted (#156) that Example 3's non-canonical ERC-20 paymaster can't check the user's ERC-20 balance without reading storage outside `tx.sender`, which the restrictive tier forbids, and the canonical paymaster's fixed bytecode doesn't cover ERC-20 either. He argued (#157) that verification-phase complexity is also *useful* and the spec shouldn't scope it so tightly only known use cases survive. The concrete answer lives in [Mempool Strategy → ERC-20 paymaster patterns](/mempool-strategy#erc20-paymaster-patterns): Example 3 is the permissionless onchain variant, which routes through non-public tiers, while the offchain variant uses a signing service and does propagate publicly. The structural concern (#157) lines up with forshtat's layering thread and the guarantors/payer-before-sender debate.

**What changed**: PR #11692 opened as the second sibling EIP; soispoke's review fixes folded in before the Phase 16 merge; EIP-8266 assigned with its own thread; trebor's #156-157 reopened the Example 3 paymaster question.

---

## Phase 16: Signatures List and EIP-8266 Merged (May 22)

*A one-day cluster. Two long-open structural threads close within sixteen hours: PR #11481 (signatures list, open since Apr 2) and PR #11692 / EIP-8266. The compose-by-requires pattern now has two siblings backing it, raising the bar for PR #11681's absorb-into-base counter-proposal.*

### Signatures List in Outer Tx (Merged)

*lightclient — PR #11481, opened Apr 2, merged May 22*

The longest-open spec PR after the EIP-3607 carve-out finally landed, with all-reviewer approval since Apr 2; jochem-brouwer's May 22 editorial review closed the loop. It adds a `signatures` outer-envelope field whose entries are verified before any frame executes, so contracts only check authority. The forward-compat hook justifies the schema change: a future block-level aggregated witness can replace the per-tx list, eliding individual signatures from on-wire serialization while preserving commitments, which matters because PQ signatures (Falcon ~666 B, Dilithium ~2,420 B) are large.

derekchiang's Apr 9 signature-index-discovery concern (a contract can't hardcode its index, so the default code must loop) is not addressed in the merged diff; that known limitation is the trade-off accepted for the aggregation hook. A deterministic signature-ordering rule is the leading follow-up candidate but no PR has opened. A separate forum thread (0xrcinus, morph-dev, Apr 15-16) debated a default-code optimization to skip per-frame VERIFY on a matching outer sighash; morph-dev rejected it citing key-rotation bypass risk, and it didn't make the merge.

### EIP-8266: Expiring Nonces for Frame Transactions (Merged)

*nerolation, lightclient — PR #11692, opened May 19, merged May 22*

Three days after opening (and after the soispoke review and number assignment), EIP-8266 merged on jochem-brouwer's editor signoff; `requires` includes both EIP-8141 and EIP-8250. The soispoke review fixes were folded in; the retracted privacy claim survives as a known scoping note. A post-merge ECDSA-nonce-reuse concern from kdenhartog (May 24) was raised and self-retracted after confirming the EIP doesn't change replay semantics.

The architectural significance: compose-by-requires now has two siblings, not one. PR #11681's absorb-into-base proposal competes against a baseline already showing the requires-chain pattern work at scale, and the Phase 14 supersession question now has a second dimension: would #11681 also supersede EIP-8266, or would EIP-8266 stay a sibling regardless?

**What to watch into Phase 17**: whether PR #11681 closes or merges against the expanded baseline; whether a follow-up addresses the signatures-list index-discovery weakness; whether trebor's #156-157 prompt an Example 3 clarification or a push to migrate restrictive-tier paymaster rules into mempool policy; whether a third sibling EIP appears, which would settle the architectural question by empirical pressure.

---

## Phase 17: Third Sibling EIP Opens (May 25 – May 28)

*The trailing Phase 16 question about a third sibling EIP is answered yes. soispoke, vbuterin, and nerolation open PR #11726 introducing EIP-8272 (Recent Roots), the third compose-by-requires sibling, and the first to add both a new top-level field and a new opcode. The empirical pressure against PR #11681's absorb-into-base packaging now spans three siblings while #11681 sits with no reviewer signoffs since May 16.*

### EIP-8272: Recent Roots for Frame Transactions (Open)

*soispoke, vbuterin, nerolation — PR #11726, opened May 25*

PR #11726 introduces EIP-8272 as a sibling requiring both EIP-7843 (consensus `slotNumber`) and EIP-8141. It addresses a recurring restrictive-tier gap: validation sometimes needs to depend on recent application state (privacy-pool tree roots, wallet authorization roots), but the tier forbids reading arbitrary storage controlled by another account. Before EIP-8272, applications either re-implemented signed-snapshot logic per-app or stayed locked out of the public mempool. EIP-8272 makes the snapshot a protocol primitive via a new `recent_root_references` outer field, a `RECENT_ROOT_ADDRESS` system contract following the established revert-only pattern, and a new `RECENTROOTREFLOAD` opcode for validation-time read access. References are committed in the sighash and may not mutate during execution; each `(source_id, slot)` has at most one canonical root.

abcoathup left a non-editor approval May 26; the bot reports one more reviewer needed.

### Compose-by-Requires Pattern Consolidates (Architectural)

PR #11726 raises the sibling count to three (EIP-8250 merged May 11, EIP-8266 merged May 22, EIP-8272 in review). Two observations:

First, the author overlap is no longer coincidental. soispoke and nerolation lead EIP-8250; nerolation and lightclient lead EIP-8266; soispoke, vbuterin, and nerolation lead EIP-8272. The compose-by-requires camp is a deliberate layered design with a recurring roster, not a one-off.

Second, EIP-8272 expands the sibling surface in ways prior siblings did not. EIP-8250 reshaped an existing field; EIP-8266 used a sentinel value with no schema change; EIP-8272 adds a brand-new top-level field, a brand-new opcode, and a brand-new non-8141 dependency (EIP-7843). The pattern now demonstrates that siblings can extend the payload schema, the opcode space, and the dependency graph, not just system-contract storage. This is the strongest empirical argument yet against PR #11681's absorb-into-base premise: absorbing the three-sibling stack would mean folding two surfaces (expiring nonces, recent roots) that weren't even in scope for #11681's `AUTH_MANAGER` design when it opened. PR #11681 remains open with no reviewer signoffs since May 16.

**What to watch into Phase 18**: whether EIP-8272 gathers an editor signoff and merges within its predecessors' window; whether `RECENTROOTREFLOAD` triggers an opcode-budget governance review; whether the EIP-7843 dependency exposes Hegotá fork-ordering issues; whether PR #11681 reorients; and whether a fourth sibling EIP appears, pushing the question from "is this a pattern" to "what is the upper bound."

---

## Phase 18: EIP-8250 Update and ERC-20 Paymaster Resolution (Jun 1 – Jun 4)

*The four-day window between EIP-8272's submission and its merge. Two threads close: EIP-8250 evolves in place via PR #11749 (single keyed nonce generalized to a bounded key set), establishing that sibling EIPs are first-class evolving documents; and trebor's ERC-20 paymaster question reaches resolution in posts #158-159.*

### EIP-8250 Adds Nonce Key Sets (Merged)

*soispoke — PR #11749, opened and merged Jun 1*

PR #11749 generalizes EIP-8250 from a single keyed nonce to a bounded key set, auto-merged in 28 minutes. The motivation is use cases the single-key shape forced into chained transactions or off-chain coordination: multi-device wallets, relayer accounts, and session-key bundles that need to advance multiple sequences in lockstep within one transaction. All keys advance atomically as part of the payment-approving `APPROVE`; the single-key path remains a one-element set, preserving the EIP-8266 composition. The architectural signal: sibling EIPs are first-class evolving documents. EIP-8250 shipped a month ago, is already depended on by EIP-8266, and is now extended in place rather than via a superseding EIP, answering a quiet Phase 17 question in favor of in-place evolution.

### ERC-20 Paymaster Question Resolved (External)

*matt, trebor — posts #158-159, Jun 3-4*

matt replied (#158) to trebor's Phase 15 question: in Example 3's restrictive-tier framing, the sponsor cannot check the user's ERC-20 balance on-chain and must verify off-chain before approving, the same risk profile as today's non-AA sponsoring systems. trebor acknowledged (#159) and committed to a clarifying PR: Example 3's "Frame 1 checks that the user has enough ERC-20 tokens" is misleading, since that check needs a forbidden state read; the correct framing is a signature check by the canonical paymaster (off-chain authorization), not an on-chain balance read. The first time an external researcher's misreading traced to a documentation bug rather than a spec gap, and it resolves the structural concern in #157 by example: where verification needs flexibility the restrictive tier disallows, the answer is off-chain authorization, not relaxing the tier.

### ERC-8286 Brings Modular Accounts to Frame Transactions (External)

*chiranjeev13 (node.cm) — ERC PR #1794, opened Jun 3*

chiranjeev13 opens ERC-8286, the first application-layer standard built on EIP-8141 and the first frame-transaction proposal to live in the ERCs repo rather than EIPs. It defines how ERC-7579 modular accounts (validator, executor, hook, and config modules) implement the validation flow: a validator module returns an approval mode the account applies via APPROVE inside a VERIFY frame. The significance is directional: the ERC-4337 and ERC-7579 modular-account ecosystem is starting to standardize on native AA as its base rather than treat it as a competitor, the adoption signal the developer-tooling bull case predicted. Still a draft, with one editor review outstanding and CI flagged, and no thread discussion yet beyond the Jun 3 announcement ([topic 28695](https://ethereum-magicians.org/t/erc-8286-modular-accounts-for-frame-transactions/28695)).

**What carried into Phase 19**: the EIP-8250 evolution-in-place precedent shows the compose-by-requires camp treats siblings as living documents, important context when EIP-8272 merges and EIP-8288 opens within hours of each other.

---

## Phase 19: EIP-8272 Merge and Fourth Sibling EIP (Jun 5 – Jun 8)

*A one-day cluster on Jun 5 with three-day trailing review. EIP-8272 merges, and within an hour vbuterin and Thomas Coratger open PR #11772 introducing a fourth sibling EIP: a frame mode for post-quantum signature and STARK aggregation. The compose-by-requires baseline grows from three to four siblings in a single hour, while PR #11681 still sits idle since May 16.*

### EIP-8272 Recent Roots Merged

*soispoke, vbuterin, nerolation — PR #11726, opened May 25, merged Jun 5*

EIP-8272 landed eleven days after submission (faster than EIP-8266 and EIP-8250) on lightclient's editor signoff; the design held unchanged from submission. The merge confirms three Phase 17 claims: the "pinned address, runtime fixed at activation" pattern is now the dominant deployment idiom for sibling system contracts (used by `ENTRY_POINT`, `EXPIRY_VERIFIER`, `NONCE_RING`, `NONCE_MANAGER`, and now `RECENT_ROOT_ADDRESS`); the `RECENTROOTREFLOAD` opcode lands, demonstrating siblings as a general extension surface; and the `requires: 7843, 8141` header bakes in the first non-8141 sibling dependency, putting EIP-7843 on the Hegotá fork-ordering watchlist.

### Fourth Sibling EIP: Frame Type for PQ Sig and STARK Aggregation (Open)

*vbuterin, Thomas Coratger — PR #11772, opened Jun 5*

Within an hour of EIP-8272 merging, vbuterin opened PR #11772 (later EIP-8288, reserved via [topic 28723](https://ethereum-magicians.org/t/eip-frame-type-for-quantum-resistant-signature-and-stark-aggregation/28723)) with a new frame mode that aggregates post-quantum signatures and STARK proofs at the block level via a single recursive STARK. The largest sibling submission so far (+508 lines). The mechanism: a `DEP_VERIFY_FRAME_MODE` declaring dependencies that are recorded rather than executed, a new block-header field carrying the recursive proof, and EVM introspection of dependencies through existing EIP-8141 opcodes. Explicitly FOCIL-compatible.

The strategic significance is the largest single-sibling step yet. EIP-8288 is the first sibling to add a new **frame mode** (a structural extension to the mode enum) and the first to introduce a new **block-header field**, putting sibling changes into consensus-header territory. Its `requires` chain is the longest yet and drops the EIP-8250 dependency the other siblings carry, signaling the frame mode operates independently of the nonce layer. The authoring also shifts: Vitalik Buterin authors directly with Thomas Coratger (new Lean Ethereum contributor), widening the roster beyond the soispoke/nerolation/lightclient cluster. abcoathup left two clarifying comments Jun 8; no editor signoff yet.

### Compose-by-Requires Stack at Four Siblings (Architectural)

The Phase 17 question about a fourth sibling is answered the same day. EIP-8288 makes the count four (three merged, one in review). Two observations:

First, siblings are extending every dimension of EIP-8141: payload schema, opcode space, frame-mode enum (new with EIP-8288), block header (new with EIP-8288), system-contract storage, and dependency graph. No part of the surface has gone unextended in a month; the pattern's upper bound isn't visible.

Second, PR #11681 still has no reviewer signoffs since May 16. Absorbing the equivalent surface into EIP-8141 now would mean folding four siblings' worth of design surface into one amendment, with all four authoring clusters coordinating. The empirical argument against absorb-into-base is no longer theoretical; the surface has grown beyond what one bundle PR can credibly carry.

**What to watch into Phase 20**: whether EIP-8288 gathers editor signoff and merges; whether the new frame mode triggers a frame-mode-enum governance review; whether the Lean Ethereum schemes get a separate EIP; whether PR #11681 retracts, rebases, or stalls; whether trebor lands the Example 3 clarification; and whether a fifth sibling EIP appears.

---

## Phase 20: EIP-8288 Thread Discussion and Hegotá PFI Status (Jun 9 – Jun 15)

*A low-tempo continuation window. No new merged 8141 PRs and no new siblings, but two design questions surface on the EIP-8288 thread, a Hegotá PFI batch excludes the entire AA stack, and PR #11681 passes its one-month-idle mark.*

### EIP-8288 Thread Raises Omission and Scheme Selection (External)

*pipavlo82, albert-garreta — [topic 28723](https://ethereum-magicians.org/t/eip-frame-type-for-quantum-resistant-signature-and-stark-aggregation/28723) posts #2-#4, Jun 6 – Jun 11*

The PQ frame mode's thread picks up two substantive questions. pipavlo82 (#2-3, Jun 6-7) raises omission accountability: the recursive aggregate proves validity of *included* dependencies but not exhaustiveness of *submitted* ones, so if a builder omits a valid dependency the frame mode produces no durable evidence of it. The draft leaves this to mempool/builder protocol. albert-garreta (#4, Jun 11) asks why the EIP is drafted around SPHINCS rather than the closer-to-standardization lattice schemes (Falcon, Dilithium). Neither changes the spec, but both are candidate motivations if EIP-8288 advances; the omission question echoes the FOCIL inclusion-list debate and would likely need EIP-7805 coordination. The authors have not responded as of this sync.

### Hegotá PFI Batch Lands Without Sibling EIPs (Governance)

*nixorokish — PR #11786, merged Jun 9*

PR #11786 promotes nine EIPs from "Considered" to "Proposed for Inclusion" in the Hegotá meta EIP, spanning EVM control flow, private transfers, and CL/EL sync. Conspicuously absent: EIP-8141 and all four siblings (EIP-8250, EIP-8266, EIP-8272, EIP-8288). EIP-8141 remains at CFI; the siblings aren't formally tracked in the meta EIP at all. The signal: ACD calls haven't promoted the AA stack toward Hegotá inclusion, consistent with the still-expanding merge tempo and the open architectural question. A second PFI batch the next day (PR #11793, Jun 10) proposes EIP-2488 and EIP-7645, also non-AA, so two consecutive batches promoted non-AA work while the AA cluster stayed at CFI.

### EIP-8130 Continues Finalization (External)

*chunter-cb — PRs #11782, #11785, #11791, #11794, Jun 9 – Jun 10*

EIP-8130 (Coinbase/Base, the principal alternative) continues its finalization burst with four PRs, including a `verifier` → `authenticator` rename, the second large naming refactor in two weeks (after `owners` → `actors`, PR #11764). EIP-8130 has shipped roughly twelve PRs in twelve days while EIP-8141's base spec sat stable; the two are converging on different governance trajectories (EIP-8130 toward a clean Hegotá submission, EIP-8141 expanding through siblings without yet seeking PFI).

### Helkomine Re-Asks the EIP-7825 Question (External)

*Helkomine — post #160, Jun 11*

Helkomine, who carried the simpler-alternatives debate in Phase 1, returns asking whether EIP-8141 removes EIP-7825's per-transaction gas cap. Since frame transactions are EIP-2718 typed transactions with no explicit carve-out, the answer is no, the limits are inherited. The question echoes the same broad-relevance pressure from Phase 1: whether the EIP unnecessarily constrains use cases. No author response yet.

### PR #11681 Passes the One-Month-Idle Mark (Spec)

*pedrouid — PR #11681, last activity May 18*

The Phase 19 watch item asked whether PR #11681 (absorb-into-base) would reorient or stay idle past the one-month mark (May 16 + 30 = Jun 15). As of Jun 15 it stayed idle: no activity since May 18, no reviewer signoffs. Over the same month the compose-by-requires stack added EIP-8272 and opened EIP-8288, so the surface the bundle would have to carry kept growing while the bundle stalled. The empirical signal favors compose-by-requires: siblings ship, the absorption amendment does not.

**What to watch into Phase 21**: whether the EIP-8288 thread questions get an author response or spec revision; whether EIP-8288 merges; whether the next Hegotá update promotes any AA-stack EIPs to PFI; whether the stalled PR #11681 is retracted or rebased; whether trebor's Example 3 clarification lands; and whether a fifth sibling EIP appears.

## Phase 21: Signatures-List Regression and Spec Cleanup (Jun 15 – Jun 30)

*The signatures-list refactor merged May 22 turns out to have dropped arbitrary signature bytes, breaking custom schemes like passkeys. The community surfaces it, matt acknowledges it and opens his own fix, and the first cleanup PR merges. EIP-2542 becomes the first EIP formally withdrawn as superseded by EIP-8141.*

### EIP-8288 Author Defends Hash-Based Minimalism (External)

*vbuterin, pipavlo82 — [topic 28723](https://ethereum-magicians.org/t/eip-frame-type-for-quantum-resistant-signature-and-stark-aggregation/28723) posts #5-6, Jun 15 – Jun 17*

vbuterin answers albert-garreta's Phase 20 SPHINCS-vs-lattice question (#5, Jun 15): the base layer should stay purely hash-based to minimize dependencies and infrastructure maintenance, accepting larger signatures as the cost. pipavlo82 accepts the rationale (#6, Jun 17) and proposes hash-only dependency receipts kept outside the core frame for future omission checks. Resolves the Phase 20 watch item without changing the spec; omission accountability stays deferred to FOCIL coordination.

### Signatures-List Refactor Breaks Arbitrary Signature Bytes (Spec)

*nlordell, DanielVF, matt — posts #161-164, Jun 15 – Jun 17*

The signatures-list outer field (PR #11481, merged May 22) draws its first serious scrutiny. nlordell (#161, Jun 15) argues custom schemes like passkeys are under-supported: a signature must commit to the frame-tx hash, but that hash now includes the signatures list, a circular dependency. DanielVF (#162) and nlordell (#163) note VERIFY frame data is elided from the sig hash for exactly this reason, but the elision was not carried over to the new list. matt (#164, Jun 17) acknowledges the merge dropped arbitrary signature bytes and commits to restoring it. The first confirmed regression from a merged 8141 PR, validating the cost of the fast merge tempo flagged in earlier phases.

### Two Cleanup PRs Open to Fix the Regression (Spec)

*nerolation — PR #11810; morph-dev — PR #11814, both Jun 17*

matt's commitment lands as two PRs the same day. PR #11810 (nerolation) restores a Behavior section accidentally deleted in an earlier merge; it merged June 26 with no debate. PR #11814 (morph-dev, new contributor) cleans up: raw `sig.signature` bytes are elided from every sig hash, `sig.signer` defaults to `tx.sender` when empty, and the public-mempool rule that EXPIRY_VERIFIER must be the first frame is made explicit. Rebased June 30 with a few TODOs still open; still awaiting an Author review.

### Lightclient Re-Adds Arbitrary Signature Data (Spec)

*lightclient — PR #11837, opened Jun 25*

The regression's own author ships the substantive fix. PR #11837 (+44/-19) re-adds arbitrary signature data to the outer signatures list so custom verifiers can sign over the canonical hash and insert the signature afterward, breaking the circular dependency nlordell flagged in post #161. lightclient's description concedes he had cut the feature from #11481 at the last minute after convincing himself it was unneeded. It overlaps #11814's elision approach; how the two compose is the open question.

### EIP-2542 Withdrawn as Superseded (External)

*forshtat — PR #11773, merged Jun 30*

EIP-2542 (2020, TXGASLIMIT/CALLGASLIMIT gas-introspection opcodes) is moved to Withdrawn with `withdrawal-reason: Superseded by EIP-8141`, since `TXPARAM`/`FRAMEPARAM` cover the use case. The first formal supersession of an older EIP by frame transactions, filed by an 8141 co-author.

### PR #11681 Stays Idle (Spec)

*pedrouid — PR #11681, last activity May 18*

No change: the absorb-into-base amendment remains idle while the compose-by-requires sibling stack sets the direction; the architectural question is unchanged from Phase 20.

**What to watch into Phase 22**: whether #11837 and #11814 merge, and which carries the signatures fix; whether EIP-8288 clears editorial review and merges; whether trebor's Example 3 clarification lands; whether the next Hegotá update promotes any AA-stack EIP to PFI; whether ERC-8286 advances past its CI failures toward editor review; and whether a fifth sibling EIP appears.

## Phase 22: Signature Fix Lands and EIP-8288 Hits Proof-Security Review (Jul 1 - Jul 9)

*The Phase 21 signatures-list regression is resolved in the spec through three July merges. The active uncertainty moves from "how do custom verifiers sign the canonical hash" to "how does the still-open PQ/STARK sibling prove recursive soundness." EIP-8130 also continues its finalization run, now under authenticator terminology.*

### Arbitrary Signature Data Returns (Spec)

*lightclient - PR #11837, merged Jul 6*

PR #11837 lands the custom-verifier fix lightclient previewed in Phase 21. The signatures list gains `ARBITRARY = 0x0`; `SECP256K1` becomes `0x1`, `P256` becomes `0x2`. `ARBITRARY` entries have empty signer metadata, no protocol cryptographic validation, and raw bytes accessible through the new `SIGPARAM (0xb4)` copy operation. Protocol-validated raw signature bytes remain hidden from EVM code, preserving future aggregation. The canonical hash now elides raw signature bytes for signatures with empty `msg`, so custom witness bytes can be inserted after signing without reintroducing VERIFY-frame data elision.

### Cleanup Locks Default Code and Public-Mempool Shape (Spec)

*lightclient, morph-dev - PR #11870 merged Jul 6; PR #11814 merged Jul 7*

PR #11870 clarifies that `APPROVE` and frame/signature introspection exceptional-halt outside frame transactions, and updates intrinsic gas to charge actual frame/signature data plus signature-validation cost. PR #11814 folds the repair into the execution and mempool text: default code now requires a `SECP256K1` signature at `tx.signatures[0]`; empty signer metadata defaults to `tx.sender`; public-mempool expiry verifier may only be first; validation-prefix flags must match `APPROVE` scope; intrinsic signature validation counts under `MAX_VERIFY_GAS`; no `VERIFY` frame may appear after the validation prefix. The ERC-20 sponsor example is also corrected: public sponsors check frame/signature metadata and accept sponsee frontrunning risk instead of checking token balance onchain.

### EIP-8288 Review Becomes a Proof-Security Review (Sibling)

*jochem-brouwer, b-wagn, soispoke, mmjahanara - PR #11772, still open*

EIP-8288 does not merge. jochem-brouwer requested editorial changes Jun 30, but the deeper July blocker is b-wagn's recursive-proof knowledge-soundness concern (Jul 3), with a suggested depth-counter public input. soispoke and mmjahanara continued LeanSTARK clarifications Jul 7-9. The Magicians thread remains quiet after Jun 17; the live review is now in GitHub.

### EIP-8130 Keeps Finalizing (External)

EIP-8130 lands PR #11847 on Jul 2 (actor-policy clarification and opaque metadata simplification) and PR #11903 on Jul 8 (renumbering `AA_TX_TYPE` to `0x79`, `AA_PAYER_TYPE` to `0x7A`, and adding `eth_call`/`eth_estimateGas` support for AA transaction fields). PR #11751, previously tracked as open, closed Jun 18. The alternative continues converging around "authenticator" rather than "verifier" language.

**What to watch into Phase 23**: whether EIP-8288 resolves the recursive-proof concern; whether #11482, #11555, #11580, or #11681 revive after the signature cleanup; whether ERC-8286 gets editor review; whether any AA-stack EIP reaches Hegotá PFI; and whether the `SIGPARAM (0xb4)` / EIP-8272 opcode-space interaction gets explicitly reconciled.

## Phase 23: Implementation Audit and Sibling Reconciliation (Jul 15 – Jul 20)

*Implementation work exposes underspecified details across the base EIP and its siblings. Fixes land quickly, while wallet and core-dev forums clarify the competitive and governance context without moving EIP-8141 beyond CFI.*

### Recent Roots: Envelope Field or Canonical Frame? (External)

*forshtat, soispoke — EIP-8272 thread posts #2-3, Jul 15 – Jul 17*

Forshtat asks whether recent roots need an envelope field and opcode when a canonical VERIFY frame could reuse the base encoding. Soispoke agrees the frame route could work, but notes the envelope enables pre-execution checks and direct indexing. No change follows.

### Wallet Developers Compare ERC-8286 and EIP-8130 (External)

*AllWalletDevs #40 — Jul 15*

[AllWalletDevs #40](https://ethereum-magicians.org/t/allwalletdevs-40-july-15-2026/28858) presents ERC-8286 beside EIP-8130. Chris Hunter says Base plans the narrower authenticator model for its September fork, making the deployment competition concrete.

### Hegotá Status Remains CFI (Governance)

*ACDE #241 — Jul 16*

The [ACDE #241 summary](https://ethereum-magicians.org/t/all-core-devs-execution-acde-241-july-16-2026/29001) calls frame transactions the CFI placeholder for native account abstraction. No proposal is promoted to PFI.

### Skipped Receipt Status Corrected (Merged)

*lightclient — PR #11953, merged Jul 17*

PR #11953 changes skipped atomic-batch status from `0x3` to the next available value, `0x2`. With approvals already satisfied, the consensus-visible constant correction auto-merges without debate.

### Base-Spec Implementation Audit Lands (Merged)

*lightclient, svlachakis — PRs #11954, #11937-#11939, #11941, merged Jul 17*

The audit restores codeless EOA sponsorship (#11954), pins signature and frame-data encoding, warns that execution approval covers all later sender frames, and applies the calldata floor. Review stays brief because each fix closes a concrete client or security ambiguity.

### Recent-Root Selector Corrected (Merged)

*AnkushinDaniil — PR #11930, merged Jul 18*

PR #11930 moves EIP-8272's reference-count selector to `0x0f`, avoiding EIP-8250's assignment. Soispoke approves the one-line constant correction without objection.

### Sibling Payloads and Mempool Rules Catch Up (Merged)

*AnkushinDaniil — PRs #11931, #11959-#11961, #11963, merged Jul 18*

The siblings regain inherited signature fields, hashing, and gas costs. For recent-root eviction, soispoke rejects a per-root pending cap because it would throttle privacy pools; reference/expiry indexing replaces it.

### Keyed-Nonce Pricing and Selector Corrected (Merged)

*AnkushinDaniil, soispoke — PRs #11958, #11966, merged Jul 19*

EIP-8250 starts pricing nonce data under EIP-7623, then moves its first-key selector to `0x10`. Review retains token pricing for consistency; nerolation approves the selector and current `payer` wording.

### Main Thread Returns to Operational Questions (External)

*novenrizkia856-ui — post #165, Jul 19*

Post #165 asks about shared-paymaster contention and `ORIGIN` compatibility with deployed `tx.origin` checks. No author response follows yet.

### Recent-Root Opcode Collision Resolved (Merged)

*soispoke — PR #11967, merged Jul 20*

PR #11967 moves `RECENTROOTREFLOAD` from `0xb4`, now occupied by `SIGPARAM`, to `0xb5`. Nerolation approves the final structural collision fix without proposing an alternative.

**What to watch into Phase 24**: approval/refund follow-ups (#11940, #11942, #11955-#11956, #11969, #11971), additive sibling rewrites, EIP-8288 proof security, and Hegotá status.

## Phase 24: Implementation Hardening and Fee Settlement (Jul 20 – Jul 28)

*Client implementation turns the audit queue into consensus rules. Review favors structural constraints and explicit formulas over special-case state preservation; no forum or ethresear.ch discussion changes the direction.*

### Arbitrary Signature Entries Get Priced (Merged)

*lightclient — PR #11976, submitted and merged Jul 20*

The proposed fixed signature-count cap gave way to 100 gas per `ARBITRARY` entry. lightclient preferred transaction gas as the bound, avoiding a second limit while closing the zero-cost decode path.

### Receipt Message Encoding Lands (Merged)

*svlachakis — PR #11942, submitted Jul 16, merged Jul 20*

Review keeps the wire shape in EIP-8141 until devp2p owns it, giving implementations one deterministic representation.

### VERIFY Frames Leave Atomic Batches (Merged)

*AnkushinDaniil, lightclient — PR #11955, submitted Jul 17, merged Jul 21*

Nethermind's snapshot implementation showed that payer state and escrow could diverge during batch rollback. Review abandoned special approval-survival semantics and excluded `VERIFY` from batches; mempool simulation now waits for the approving frame to succeed.

### Refund Accounting Becomes Transaction-Scoped (Merged)

*svlachakis — PR #11940, submitted Jul 16, merged Jul 21*

Review chose one EIP-3529 counter across frames. soispoke and AnkushinDaniil pushed for exact rollback, cap, and receipt wording, preserving gross frame observability even though frame totals need not match final transaction gas.

### P256 Becomes Canonical (Merged)

*svlachakis — PR #11984, submitted and merged Jul 21*

Because the P256 precompile accepts both signature forms, protocol validation now requires low-`s` and wallets must normalize. The choice removes outer-signature malleability without narrowing the underlying crypto primitive.

### Atomic-Batch Shape Becomes Static (Merged)

*lightclient — PR #11987, submitted and merged Jul 21*

The execution-time conclusion from #11955 becomes a validity rule: only DEFAULT and SENDER may carry the flag, and VERIFY cannot enter or terminate a batch. Review favored early rejection over another runtime corner case.

### Blob Support Replaces a Blobless Exception (Merged)

*svlachakis — PR #11985, submitted Jul 21, merged Jul 23*

The draft first forbade blobs, but review required frame transactions to support them fully. The result adopts EIP-7594 networking and includes blob gas in payer settlement, keeping the new transaction type aligned with Ethereum's data path.

### Protocol-Defined Prefixes Gain a Fast Path (Merged)

*svlachakis — PR #12001, submitted and merged Jul 23*

Clients may evaluate fully protocol-defined prefixes directly, but review preserves identical gas and dependency results. The optimization removes redundant EVM execution without weakening revalidation.

### APPROVE Pricing Is Simplified (Merged)

*AnkushinDaniil — PR #12003, submitted and merged Jul 23*

Review treats nonce, payer, and escrow bookkeeping as work already priced intrinsically. `APPROVE` therefore pays only memory expansion, avoiding a second charge for protocol effects.

### Fee Fields Gain Explicit Bounds (Merged)

*svlachakis — PR #12005, submitted and merged Jul 23*

All fee fields now use the typed-transaction `< 2**256` bound. The change removes cross-client arithmetic ambiguity before the settlement rewrite lands.

### Replacement, Logs, Registry, and Paymaster Work Opens

*svlachakis, AnkushinDaniil — PRs #12007-#12008, #12011-#12012, Jul 23 – Jul 24*

New drafts move attention to payer exposure and replacement, transaction-log aggregation, a governed signature-scheme registry, and a complete canonical-paymaster implementation.

### Complete Fee Settlement Lands (Merged)

*soispoke, lightclient — PR #11969, submitted Jul 18, merged Jul 28*

The final review resolves how the calldata floor, EIP-3529 refunds, blob fees, payer escrow, and receipts compose. Corrections during review fixed refund ordering and overflow checks; lightclient's final rewrite makes payer refunds explicit while retaining gross frame receipts.

**What to watch into Phase 25**: whether #12007, #12008, #12011, or #12012 gathers author consensus; whether additive sibling PRs #11968 and #11970 merge; and whether the base spec's new settlement rules expose further client mismatches.
