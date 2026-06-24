# Appendix

---

## Sources

- [EIP-8141 Spec](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-8141.md)
- [All Related PRs](https://github.com/ethereum/EIPs/pulls?q=is%3Apr+8141)
- [Ethereum Magicians Discussion](https://ethereum-magicians.org/t/frame-transaction/27617)

---

## Complete PR Timeline

### Merged

| Date | PR | Author | Description |
|---|---|---|---|
| Jan 29 | [#11202](https://github.com/ethereum/EIPs/pull/11202) | fjl | Original EIP submission |
| Jan 29 | [#11205](https://github.com/ethereum/EIPs/pull/11205) | fjl | Fix: elide VERIFY frame data from sig hash |
| Jan 29 | [#11209](https://github.com/ethereum/EIPs/pull/11209) | kevaundray | Fix: status field number in TXPARAM |
| Feb 10 | [#11297](https://github.com/ethereum/EIPs/pull/11297) | lightclient | Relax APPROVE to not require top-level frame |
| Feb 11 | [#11305](https://github.com/ethereum/EIPs/pull/11305) | lightclient | Fix typo |
| Mar 2 | [#11344](https://github.com/ethereum/EIPs/pull/11344) | derekchiang | Fix CALLER/ADDRESS bug, clarify reverts |
| Mar 10 | [#11355](https://github.com/ethereum/EIPs/pull/11355) | rakita | Add EIP-8175 Composable Transaction (related) |
| Mar 10 | [#11379](https://github.com/ethereum/EIPs/pull/11379) | derekchiang | Add EOA support (default code) |
| Mar 12 | [#11400](https://github.com/ethereum/EIPs/pull/11400) | fjl | Clean up opcodes: FRAMEDATALOAD/COPY |
| Mar 12 | [#11401](https://github.com/ethereum/EIPs/pull/11401) | fjl | Add approval bits to frame mode |
| Mar 13 | [#11402](https://github.com/ethereum/EIPs/pull/11402) | fjl | Fix bit indices (1-indexed) |
| Mar 13 | [#11406](https://github.com/ethereum/EIPs/pull/11406) | derekchiang | Add derekchiang as co-author |
| Mar 25 | [#11395](https://github.com/ethereum/EIPs/pull/11395) | derekchiang | Add atomic batching |
| Mar 25 | [#11415](https://github.com/ethereum/EIPs/pull/11415) | lightclient | Add mempool policy |
| Mar 26 | [#11448](https://github.com/ethereum/EIPs/pull/11448) | derekchiang | Update default code for approval bits |
| Apr 8 | [#11251](https://github.com/ethereum/EIPs/pull/11251) | BonyHanter83 | Add EIP-1559 to requires header |
| Apr 14 | [#11521](https://github.com/ethereum/EIPs/pull/11521) | benaadams | Tighten spec (mode/flags split, FRAMEPARAM, MAX_FRAMES=64, per-frame cost, default code hardening) |
| Apr 16 | [#11534](https://github.com/ethereum/EIPs/pull/11534) | lightclient | Add `value` field to frame (SENDER-only, TXPARAM(0x08), FRAMEPARAM(0x08)) |
| Apr 22 | [#11544](https://github.com/ethereum/EIPs/pull/11544) | derekchiang | Mix in FRAME_TX_TYPE to sighash (EIP-2718 cross-type replay fix) |
| Apr 28 | [#11575](https://github.com/ethereum/EIPs/pull/11575) | lightclient | Allow payer to approve before sender (auto-merged in error; reverted by #11579 same window, reopened as draft #11580) |
| Apr 29 | [#11579](https://github.com/ethereum/EIPs/pull/11579) | lightclient | Revert #11575 |
| Apr 29 | [#11577](https://github.com/ethereum/EIPs/pull/11577) | lightclient | Remove RLP call batch from default account (default-code `SENDER` mode now reverts) |
| Apr 30 | [#11567](https://github.com/ethereum/EIPs/pull/11567) | derekchiang | Relax mempool deploy-frame rule (drops EIP-7997 from requires; any stateless factory qualifies; CREATE/SETDELEGATE join CREATE2 in deploy-frame carve-out) |
| Apr 30 | [#11537](https://github.com/ethereum/EIPs/pull/11537) | dionysuzx | Add EIP-8141 to CFI in EIP-8081 Hegotá meta EIP (governance) |
| May 5 | [#11272](https://github.com/ethereum/EIPs/pull/11272) | Thegaram | Disable EIP-3607 origination check for frame transactions (adds 3607 to `requires` with explicit carve-out) |
| May 11 | [#11598](https://github.com/ethereum/EIPs/pull/11598) | soispoke, nerolation, lightclient, vbuterin | Add EIP-8250: Keyed Nonces for Frame Transactions (standalone EIP layering `(nonce_key, nonce_seq)` and a `NONCE_MANAGER` system contract on EIP-8141; first EIP whose `requires` header includes EIP-8141) |
| May 11 | [#11621](https://github.com/ethereum/EIPs/pull/11621) | lightclient | Frames cleanup (spec coherence refactor: skipped-batch receipt status, FRAMEPARAM operand order, P256 dropped from default code, default code accepts SENDER/DEFAULT, adds 7623+7702 to requires) |
| May 12 | [#11652](https://github.com/ethereum/EIPs/pull/11652) | derekchiang | Extend atomic batching from `SENDER`-only to any frame mode; restrictive mempool tier separately forbids the flag inside the validation prefix |
| May 14 | [#11662](https://github.com/ethereum/EIPs/pull/11662) | nerolation | Add EXPIRY_VERIFIER frame: canonical contract at `address(0x8141)` whose runtime enforces an 8-byte unix-seconds deadline; mempool drops expired txs; `TIMESTAMP` carve-out for canonical runtime |
| May 22 | [#11481](https://github.com/ethereum/EIPs/pull/11481) | lightclient | Add signatures list to outer tx (opened Apr 2); new `signatures` outer-envelope field carrying signature + algorithm + signer metadata, verified before frame execution; default code reads from this list. Forward-compat hook for PQ signature aggregation |
| May 22 | [#11692](https://github.com/ethereum/EIPs/pull/11692) | nerolation | Add EIP-8266: Expiring Nonces for Frame Transactions (sibling EIP whose `requires` includes EIP-8141 and EIP-8250). Second EIP in the compose-by-requires AA stack |
| Jun 1 | [#11749](https://github.com/ethereum/EIPs/pull/11749) | soispoke | Update EIP-8250: Add support for nonce key sets (generalizes single keyed nonce to a bounded key set; +64/-40). First post-merge revision to any sibling EIP |
| Jun 5 | [#11726](https://github.com/ethereum/EIPs/pull/11726) | soispoke, vbuterin, nerolation | Add EIP-8272: Recent Roots for Frame Transactions (sibling EIP, +394 lines). Third compose-by-requires sibling requiring both EIP-7843 and EIP-8141. New `recent_root_references` outer-envelope field, `RECENT_ROOT_ADDRESS` system contract with 8192-slot ring, new opcode `RECENTROOTREFLOAD (0xB4)` |

### Open

| Date | PR | Author | Description |
|---|---|---|---|
| Apr 2 | [#11482](https://github.com/ethereum/EIPs/pull/11482) | derekchiang | Allow precompiles for VERIFY frames (all reviewers approved) |
| Apr 22 | [#11555](https://github.com/ethereum/EIPs/pull/11555) | derekchiang | Add support for guarantors (payer covers gas even if sender validation fails) |
| Apr 29 | [#11580](https://github.com/ethereum/EIPs/pull/11580) | lightclient | Allow payer to approve before sender (draft; alternative to #11555 guarantors) |
| May 16 | [#11681](https://github.com/ethereum/EIPs/pull/11681) | pedrouid | Extend EIP-8141 with guarantors, keyed nonces, and signer binding via a `signer` envelope field and an `AUTH_MANAGER` system contract; +810/-74 lines. Successor to closed #11643 after PR #11662 settled the expiry design |
| Jun 5 | [#11772](https://github.com/ethereum/EIPs/pull/11772) | vbuterin, Thomas Coratger | Add EIP-8288 (proposed): Frame type for PQ sig and STARK aggregation (`eip-9999.md` placeholder, +508 lines). Fourth compose-by-requires sibling EIP. New `DEP_VERIFY_FRAME_MODE = 3` frame mode declaring `(scheme, data_hash, verification_key)` dependencies; block-level `recursive_stark` header field aggregates them into one recursive STARK proof. Lean Ethereum tooling: `LEANSPHINCS_SCHEME` and `LEANSTARK_SCHEME` |
| Jun 17 | [#11810](https://github.com/ethereum/EIPs/pull/11810) | nerolation | Restore the Behavior section accidentally deleted by the signatures-list merge (#11481); +23/-0, all reviewers approved, awaiting merge |
| Jun 17 | [#11814](https://github.com/ethereum/EIPs/pull/11814) | morph-dev | Spec cleanup: elide raw `sig.signature` bytes from all sig hashes (restores arbitrary-signature-byte support regressed by #11481, preserves aggregation compatibility), `sig.signer` defaults to `tx.sender`, EXPIRY_VERIFIER must be first frame; +103/-73, several TODOs remain |

### Related

| Date | PR | Author | Description |
|---|---|---|---|
| Apr 11 | [#11509](https://github.com/ethereum/EIPs/pull/11509) | benaadams | Add EIP-8223: Contract Payer Transaction (alternative/complementary sponsorship proposal) |
| Apr 12 | [#11518](https://github.com/ethereum/EIPs/pull/11518) | benaadams | Add EIP-8224: Counterfactual Transaction (shielded gas funding via ZK proofs) |
| Apr 25 | [#11571](https://github.com/ethereum/EIPs/pull/11571) | SirSpudlington | Update EIP-7932: refactor signature registry to be friendlier to EIP-8141 (rename `sigrecover` → `sigaddress`, add `sigverify`/`sigcosts` precompiles for AA use cases) |
| Jun 3 | [#1794](https://github.com/ethereum/ERCs/pull/1794) | chiranjeev13 | Add ERC-8286: Modular Accounts for Frame Transactions (ERCs repo). Defines how ERC-7579 modular accounts implement the EIP-8141 validation flow; first application-layer standard built on EIP-8141 |
| Jun 5 | [#11773](https://github.com/ethereum/EIPs/pull/11773) | forshtat | Move EIP-2542 to Withdrawn, superseded by EIP-8141 (first external AA proposal formally withdrawn in favor of frame transactions) |

### Closed (not merged)

| Date | PR | Author | Description | Reason |
|---|---|---|---|---|
| Feb 13 | [#11310](https://github.com/ethereum/EIPs/pull/11310) | marukai67 | Fix link to ERC-7562 | "It's not broken" — lightclient |
| Feb 14 | [#11314](https://github.com/ethereum/EIPs/pull/11314) | marukai67 | Fix link to EIP-2718 | "Not broken, thanks though" — lightclient |
| Feb 15 | [#11321](https://github.com/ethereum/EIPs/pull/11321) | marukai67 | Fix links | "They aren't broken" — lightclient |
| Feb 25 | [#11352](https://github.com/ethereum/EIPs/pull/11352) | lucemans | Accidental PR | Self-closed |
| Mar 13 | [#11404](https://github.com/ethereum/EIPs/pull/11404) | derekchiang | Simplify approval bits | Superseded by #11401 |
| Mar 14 | [#11408](https://github.com/ethereum/EIPs/pull/11408) | SirSpudlington | Migrate default code to EIP-7932 | Rejected: authors want to keep custom behavior |
| Apr 23 | [#11455](https://github.com/ethereum/EIPs/pull/11455) | SirSpudlington | Default code tweaks for EIP-7392 compatibility | Never gathered reviewer approvals; closed after ~4 weeks |
| May 4 | [#11597](https://github.com/ethereum/EIPs/pull/11597) | soispoke, nerolation, lightclient, vbuterin | Keyed Nonces for Frame Transactions (first attempt) | PR accidentally bundled an unrelated `eip-FOCIL.md` change; closed and resubmitted clean as #11598 the same day |
| May 8 | [#11584](https://github.com/ethereum/EIPs/pull/11584) | nerolation | Add 2D nonces (delta against EIP-8141) | Closed in favor of the standalone Keyed Nonces EIP (#11598); same author/concept moved to a Standards Track sibling |
| May 14 | [#11488](https://github.com/ethereum/EIPs/pull/11488) | chiranjeev13 | Fix spec inconsistencies (APPROVE scopes, VERIFY count) | Sat open since Apr 6 with no reviewer activity; closed after PR #11621 (May 11) absorbed the structurally compatible portions and the rest no longer applied |
| May 18 | [#11643](https://github.com/ethereum/EIPs/pull/11643) | pedrouid | Extended Feature Set: bundle guarantors + keyed nonces + signer binding + envelope expiry into EIP-8141 (+843/-69 lines) | Closed in favor of #11681 after PR #11662 (EXPIRY_VERIFIER, merged May 14) made the envelope-expiry component redundant |

## Key Contributors

| Person | Handle | Role |
|---|---|---|
| Vitalik Buterin | @vbuterin | Co-author of EIP-8141; co-author of EIP-8250 Keyed Nonces (PR #11598, merged May 11) and EIP-8272 Recent Roots (PR #11726, merged Jun 5); primary author with Thomas Coratger of EIP-8288 Frame type for PQ sig and STARK aggregation (PR #11772, opened Jun 5) |
| lightclient (Matt) | @lightclient | Co-author, primary spec maintainer, added per-frame `value` (PR #11534, merged Apr 16) |
| Felix Lange | @fjl | Co-author, original PR submitter, opcode design |
| Yoav Weiss | @yoavw | Co-author |
| Alex Forshtat | @forshtat | Co-author, ERC-7562/4337 expertise |
| Dror Tirosh | @drortirosh | Co-author |
| Shahaf Nacson | @shahafn | Co-author |
| Derek Chiang | @derekchiang | Co-author (added Mar 13), EOA support, batching, precompile VERIFY |
| Daniel Von Fange | @DanielVF | Key external reviewer (Monad), adoption/performance critique |
| 0xrcinus (Orca) | @0xrcinus | Active reviewer, mode simplification proposals |
| Francisco Giordano | @frangio | Active reviewer (OpenZeppelin), naming/semantics |
| nlordell | @nlordell | Early reviewer, APPROVE propagation analysis |
| Peter Garamvolgyi | @thegaram33 | Early reviewer; author of PR #11272 (EIP-3607 carve-out for frame transactions, merged May 5 after sitting open since Feb 6) |
| Danno Ferrin | @shemnon | Reviewer, scope creep concerns |
| jochem-brouwer | @jochem-brouwer | Detailed canonical paymaster review |
| Seungmin Jeon | @sm-stack | PoC implementation, atomic batch bit flag idea |
| rmeissner | @rmeissner | Safe team representative, value-in-frames advocate |
| Chiranjeev Mishra (node.cm) | @chiranjeev13 | Forum reviewer (VERIFY frame count observation) and author of the closed spec-consistency PR #11488; author of ERC-8286 Modular Accounts for Frame Transactions (ERC PR #1794, opened Jun 3), the first application-layer standard built on EIP-8141 |
| Ben Adams | @benaadams | Spec tightening (PR #11521, merged Apr 14), author of EIP-8223 (Contract Payer Transaction) and EIP-8224 (Counterfactual Transaction) |
| Jacopo | @jacopo-eth | Proposed FRAMERETURNDATASIZE/FRAMERETURNDATACOPY for multi-step flows |
| Franco Victorio | @fvictorio | Raised question about validation-frame execution ordering vs non-frame txs |
| dionysuzx | @dionysuzx | Hegotá meta-EIP maintainer, submitted PR #11537 moving EIP-8141 to CFI (merged Apr 30) |
| Nero_eth | Nero_eth | ethresear.ch analyst; "Three Gates to Privacy" post framing mempool/FOCIL/VOPS constraints on privacy-pool flows through frame transactions |
| Toni Wahrstätter | @nerolation | Author of PR #11584 (2D nonces, closed), co-author of EIP-8250 Keyed Nonces (PR #11598, merged May 11), author of PR #11662 (EXPIRY_VERIFIER frame, merged May 14), co-author with lightclient of EIP-8266 Expiring Nonces (PR #11692, merged May 22), and co-author with soispoke and vbuterin of EIP-8272 Recent Roots (PR #11726, merged Jun 5). Added to EIP-8141's `author` header in PR #11662 |
| Thomas Thiery | @soispoke | Lead author of EIP-8250 Keyed Nonces (PR #11598, merged May 11, extended Jun 1 by PR #11749 adding bounded key-set support) and EIP-8272 Recent Roots (PR #11726, merged Jun 5). Caught the same-block replay bug in PR #11692 (EIP-8266) during May 19 inline review, prompting nerolation's fixes |
| Pedro Gomes | @pedrouid | Author of PR #11681 (opened May 16), the successor to closed PR #11643 (Extended Feature Set, May 11 – May 18): proposes bundling guarantors, keyed nonces, and signer binding into EIP-8141 itself via a `signer` envelope field and an `AUTH_MANAGER` system contract, dropping the envelope-expiry component now that PR #11662 (EXPIRY_VERIFIER) ships protocol-level expiry as a verifier-frame contract |
| German Abal | @ariutokintumi | Co-founder/architect of EVVM (contract-native AA framework); contributed a production-perspective comparison on the magicians thread (post #148, May 7) on per-environment policy, async execution, batch granularity, and reservation primitives |
| Sam Wilson | @SamWilsn | EIP editor; spec-coherence review (post #149, May 8) on naming, empty-target representation, opcode-budget, and `FRAMEDATACOPY` revert semantics |
| trebor | @trebor | Privacy-focused paymaster researcher (Kohaku); raised the Example 3 (ERC-20 paymaster) feasibility question in posts #156-157 (May 21). After matt's clarification in post #158 (Jun 3), trebor committed in post #159 (Jun 4) to a PR clarifying Example 3 in the EIP (Frame 1 should be a signature check by the canonical paymaster, not an on-chain ERC-20 balance read) |
| Thomas Coratger | (not on GitHub directly in PR author list) | Co-author with vbuterin of EIP-8288 (PR #11772, opened Jun 5): Frame type for PQ sig and STARK aggregation. Lean Ethereum tooling contributor |
| Milos Stankovic | @morph-dev | Author of PR #11814 (spec cleanup and clarifications, opened Jun 17): elides raw signature bytes from all sig hashes to restore arbitrary-signature-byte support and preserve aggregation compatibility, defaults `sig.signer` to `tx.sender`, makes the EXPIRY_VERIFIER first-frame mempool rule explicit |

## External Resources

- [Live Demo](https://demo.eip-8141.ethrex.xyz/)
- [EIP-8141 Latest Spec](https://eips.ethereum.org/EIPS/eip-8141)
- [Ethereum Magicians Discussion](https://ethereum-magicians.org/t/frame-transaction/27617)
- [PoC Implementation by sm-stack](https://github.com/sm-stack/eip8141-poc)
- [PoC Writeup](https://hackmd.io/@TB5b8ghoQyChOtUKB0RsOg/B1PhyMK_be)
- [BundleBear EIP-7702 Metrics](https://www.bundlebear.com/eip7702-overview/ethereum)
- [Account Abstraction Link Tree (matt)](https://hackmd.io/@matt/aa-link-tree)
- [Biconomy: Native AA State-of-Art Q1/26](https://blog.biconomy.io/native-account-abstraction-state-of-art-and-pending-proposals-q1-26/)
- [Openfort: What EIP-8141 Means for Developers](https://www.openfort.io/blog/eip-8141-means-for-developers)
- [FOCIL + Native Account Abstraction](https://ethereum-magicians.org/t/focil-native-account-abstraction/27999)
- [AA-VOPS: A Pragmatic Path Towards Validity-Only Partial Statelessness](https://ethresear.ch/t/a-pragmatic-path-towards-validity-only-partial-statelessness-vops/22236#p-54075-vops-and-native-account-abstraction-aavops-9)
- [Frame vs Tempo — Two clashing philosophies of native AA](https://x.com/decentrek/status/2031013555898900838)
- [The case for Frame Transactions: Flexible Foundation with Powerful Defaults](https://x.com/decentrek/status/2036697881512701997)
- [The Evolution of Self-Custody](https://x.com/pedrouid/status/2031716092112929107)
- [Ethereum Wallet UX is changing](https://x.com/pedrouid/status/2042682070997033253)
- [Let us be brave and extend EIP-8141 benefits](https://x.com/pedrouid/status/2051354277520515316)
- [1 contract, 2 fields, 4 features](https://x.com/pedrouid/status/2054584429981659388)
- [Frame Transactions and the Three Gates to Privacy](https://ethresear.ch/t/frame-transactions-and-the-three-gates-to-privacy/24666)
- [Your Ethereum Wallet is About to Change Forever](https://dorisgxyz.substack.com/p/your-ethereum-wallet-is-about-to)
- [EIP-8141 Frame Transactions (HackMD)](https://hackmd.io/@dicethedev/HyhbyJA3bg)
- [Frame Transactions vs. SchemedTransactions](https://ethereum-magicians.org/t/frame-transactions-vs-schemedtransactions-for-post-quantum-ethereum/28056)
- [EIP-8266: Expiring Nonces for Frame Transactions](https://ethereum-magicians.org/t/eip-8266-expiring-nonces-for-frame-transactions/28575)
- [EIP-8272: Recent Roots for Frame Transactions](https://ethereum-magicians.org/t/eip-8272-recent-roots-for-frame-transactions/28621)
- [EIP-8288: Frame type for PQ sig and STARK aggregation](https://ethereum-magicians.org/t/eip-frame-type-for-quantum-resistant-signature-and-stark-aggregation/28723)
- [ERC-8286: Modular Accounts for Frame Transactions](https://ethereum-magicians.org/t/erc-8286-modular-accounts-for-frame-transactions/28695)
- [Svalbard AA Breakout Session Notes](https://hackmd.io/@nixorokish/svalbard-aa-breakout)

## Competing Standards

- [EIP-8130: AA by Account Configuration](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-8130.md) — [Magicians thread](https://ethereum-magicians.org/t/eip-8130-account-abstraction-by-account-configurations/25952)
- [EIP-8175: Composable Transaction](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-8175.md) — [Magicians thread](https://ethereum-magicians.org/t/eip-8175-composable-transaction/27850)
- [EIP-8202: Schemed Transaction](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-8202.md) — [Magicians thread](https://ethereum-magicians.org/t/eip-8202-schemed-transaction/28044)
- [EIP-8223: Contract Payer Transaction](https://github.com/ethereum/EIPs/pull/11509) — [Magicians thread](https://ethereum-magicians.org/t/eip-8223-contract-payer-transactions/28202)
- [EIP-8224: Counterfactual Transaction](https://github.com/ethereum/EIPs/pull/11518) — [Magicians thread](https://ethereum-magicians.org/t/eip-8224-counterfactual-transaction/28205)
- [Tempo-like Transaction (gakonst)](https://gist.github.com/gakonst/00117aa2a1cd327f515bc08fb807102e)

