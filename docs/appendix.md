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
| Apr 29 | [#11577](https://github.com/ethereum/EIPs/pull/11577) | lightclient | Remove RLP call batch from default account (temporarily made default-code `SENDER` mode revert, later relaxed by #11621) |
| Apr 30 | [#11567](https://github.com/ethereum/EIPs/pull/11567) | derekchiang | Relax mempool deploy-frame rule (drops EIP-7997 from requires; any stateless factory qualifies; CREATE/SETDELEGATE join CREATE2 in deploy-frame carve-out) |
| Apr 30 | [#11537](https://github.com/ethereum/EIPs/pull/11537) | dionysuzx | Add EIP-8141 to CFI in EIP-8081 Hegotá meta EIP (governance) |
| May 5 | [#11272](https://github.com/ethereum/EIPs/pull/11272) | Thegaram | Disable EIP-3607 origination check for frame transactions (adds 3607 to `requires` with explicit carve-out) |
| May 11 | [#11598](https://github.com/ethereum/EIPs/pull/11598) | soispoke, nerolation, lightclient, vbuterin | Add EIP-8250: Keyed Nonces for Frame Transactions (standalone EIP layering `(nonce_key, nonce_seq)` and a `NONCE_MANAGER` system contract on EIP-8141; first EIP whose `requires` header includes EIP-8141) |
| May 11 | [#11621](https://github.com/ethereum/EIPs/pull/11621) | lightclient | Frames cleanup (spec coherence refactor: skipped-batch receipt status, FRAMEPARAM operand order, P256 dropped from default code, default code accepts SENDER/DEFAULT, adds 7623+7702 to requires) |
| May 12 | [#11652](https://github.com/ethereum/EIPs/pull/11652) | derekchiang | Extend atomic batching from `SENDER`-only to any frame mode; restrictive mempool tier separately forbids the flag inside the validation prefix |
| May 14 | [#11662](https://github.com/ethereum/EIPs/pull/11662) | nerolation | Add EXPIRY_VERIFIER frame: canonical contract at `address(0x8141)` whose runtime enforces an 8-byte unix-seconds deadline; mempool drops expired txs; `TIMESTAMP` carve-out for canonical runtime |
| May 22 | [#11481](https://github.com/ethereum/EIPs/pull/11481) | lightclient | Add signatures list to outer tx (opened Apr 2); new `signatures` outer-envelope field carrying signature, scheme/algorithm, signer, and message metadata, verified before frame execution; default code reads from this list. Forward-compat hook for PQ signature aggregation |
| May 22 | [#11692](https://github.com/ethereum/EIPs/pull/11692) | nerolation | Add EIP-8266: Expiring Nonces for Frame Transactions (sibling EIP whose `requires` includes EIP-8141 and EIP-8250). Second EIP in the compose-by-requires AA stack |
| Jun 1 | [#11749](https://github.com/ethereum/EIPs/pull/11749) | soispoke | Update EIP-8250: Add support for nonce key sets (generalizes single keyed nonce to a bounded key set; +64/-40). First post-merge revision to any sibling EIP |
| Jun 5 | [#11726](https://github.com/ethereum/EIPs/pull/11726) | soispoke, vbuterin, nerolation | Add EIP-8272: Recent Roots for Frame Transactions (sibling EIP, +394 lines). Third compose-by-requires sibling requiring both EIP-7843 and EIP-8141. New `recent_root_references` outer-envelope field, `RECENT_ROOT_ADDRESS` system contract with 8192-slot ring, and `RECENTROOTREFLOAD` opcode (moved from `0xb4` to `0xb5` by #11967) |
| Jun 26 | [#11810](https://github.com/ethereum/EIPs/pull/11810) | nerolation | Restore the `APPROVE` Behavior section accidentally deleted by the signatures-list merge (#11481); +23/-0, no semantic change beyond restoration |
| Jul 6 | [#11837](https://github.com/ethereum/EIPs/pull/11837) | lightclient | Add `ARBITRARY` signature scheme, `SIGPARAM (0xb4)`, and raw signature-byte elision for empty-`msg` signatures so custom verifiers can sign the canonical hash |
| Jul 6 | [#11870](https://github.com/ethereum/EIPs/pull/11870) | lightclient | Clarify exceptional halts outside frame transactions and charge intrinsic gas for actual frame/signature data plus signature-validation cost |
| Jul 7 | [#11814](https://github.com/ethereum/EIPs/pull/11814) | morph-dev | Spec cleanup: default code uses `tx.signatures[0]`, signer defaults to `tx.sender`, public-mempool expiry frames must be first, mempool flags must match approval scope, no VERIFY after validation prefix |
| Jul 17 | [#11953](https://github.com/ethereum/EIPs/pull/11953) | lightclient | Correct skipped atomic-batch receipt status from `0x3` to `0x2` |
| Jul 17 | [#11954](https://github.com/ethereum/EIPs/pull/11954) | lightclient | Restore codeless EOA sponsorship: index `0` for execution scope, index `1` for payment-only default VERIFY |
| Jul 17 | [#11937](https://github.com/ethereum/EIPs/pull/11937) | svlachakis | Pin secp256k1 recovery id to `0/1` and require canonical `r`/low-`s` |
| Jul 17 | [#11938](https://github.com/ethereum/EIPs/pull/11938) | svlachakis | Specify stack operand order for `FRAMEDATALOAD` and `FRAMEDATACOPY` |
| Jul 17 | [#11939](https://github.com/ethereum/EIPs/pull/11939) | svlachakis | Warn that execution approval authorizes every later SENDER frame |
| Jul 17 | [#11941](https://github.com/ethereum/EIPs/pull/11941) | svlachakis | Apply the EIP-7623 calldata floor to frame and signature data |
| Jul 18 | [#11960](https://github.com/ethereum/EIPs/pull/11960) | AnkushinDaniil | Align EIP-8250 `NONCE_MANAGER` activation wording with EIP-8272 |
| Jul 18 | [#11930](https://github.com/ethereum/EIPs/pull/11930) | AnkushinDaniil | Fix EIP-8272 recent-root count selector to non-conflicting `0x0f` |
| Jul 18 | [#11931](https://github.com/ethereum/EIPs/pull/11931) | AnkushinDaniil | Restore EIP-8141 signature data and verification cost in EIP-8272 gas limit |
| Jul 18 | [#11959](https://github.com/ethereum/EIPs/pull/11959) | AnkushinDaniil | Restore signatures and current sighash semantics in EIP-8272 |
| Jul 18 | [#11961](https://github.com/ethereum/EIPs/pull/11961) | AnkushinDaniil | Harden EIP-8272 public-mempool expiry and reorg eviction |
| Jul 18 | [#11963](https://github.com/ethereum/EIPs/pull/11963) | AnkushinDaniil | Restore signatures and correct sighash description in EIP-8250 |
| Jul 19 | [#11958](https://github.com/ethereum/EIPs/pull/11958) | AnkushinDaniil | Price EIP-8250 nonce keys and sequence as EIP-7623 transaction data |
| Jul 19 | [#11966](https://github.com/ethereum/EIPs/pull/11966) | soispoke | Move EIP-8250's first nonce-key selector to `0x10` and align approval wording |
| Jul 20 | [#11967](https://github.com/ethereum/EIPs/pull/11967) | soispoke | Move EIP-8272 `RECENTROOTREFLOAD` from colliding `0xb4` to `0xb5` |
| Jul 20 | [#11976](https://github.com/ethereum/EIPs/pull/11976) | lightclient | Price each `ARBITRARY` signature entry at 100 gas |
| Jul 20 | [#11942](https://github.com/ethereum/EIPs/pull/11942) | svlachakis | Define frame receipt encoding for the fork's `Receipts` message |
| Jul 21 | [#11955](https://github.com/ethereum/EIPs/pull/11955) | AnkushinDaniil | Exclude VERIFY frames from atomic batches and wait for the payer-setting frame to finish |
| Jul 21 | [#11940](https://github.com/ethereum/EIPs/pull/11940) | svlachakis | Define transaction-level EIP-3529 refund accounting and gross frame receipt gas |
| Jul 21 | [#11984](https://github.com/ethereum/EIPs/pull/11984) | svlachakis | Require canonical low-`s` P256 signatures and add `SECP256R1N` |
| Jul 21 | [#11987](https://github.com/ethereum/EIPs/pull/11987) | lightclient | Restrict atomic flags to DEFAULT/SENDER and statically exclude VERIFY |
| Jul 23 | [#11985](https://github.com/ethereum/EIPs/pull/11985) | svlachakis | Add full blob and EIP-7594 pooled-transaction support |
| Jul 23 | [#12001](https://github.com/ethereum/EIPs/pull/12001) | svlachakis | Allow equivalent direct evaluation of protocol-defined validation prefixes |
| Jul 23 | [#12003](https://github.com/ethereum/EIPs/pull/12003) | AnkushinDaniil | Set APPROVE base gas to zero, charging only memory expansion |
| Jul 23 | [#12005](https://github.com/ethereum/EIPs/pull/12005) | svlachakis | Bound execution and blob fee fields below `2**256` |
| Jul 28 | [#11969](https://github.com/ethereum/EIPs/pull/11969) | soispoke | Complete gas, refund, payer, receipt, and blob-fee settlement |
| Jul 28 | [#12007](https://github.com/ethereum/EIPs/pull/12007) | svlachakis | Define public-mempool replacement, per-payer exposure, revalidation, and eviction |
| Jul 28 | [#12008](https://github.com/ethereum/EIPs/pull/12008) | svlachakis | Define transaction logs and atomic-batch log discard semantics |
| Jul 29 | [#11968](https://github.com/ethereum/EIPs/pull/11968) | soispoke | Express EIP-8250 as additive changes to EIP-8141 |
| Jul 29 | [#11970](https://github.com/ethereum/EIPs/pull/11970) | soispoke | Express EIP-8272 as additive changes to EIP-8141 |
| Jul 30 | [#12042](https://github.com/ethereum/EIPs/pull/12042) | Marchhill | Align SIGPARAM copy operand order with EVM copy opcodes |
| Aug 3 | [#12067](https://github.com/ethereum/EIPs/pull/12067) | AnkushinDaniil | Pin EIP-8250 `NONCE_MANAGER` to `0x...8250` |
| Aug 3 | [#12068](https://github.com/ethereum/EIPs/pull/12068) | AnkushinDaniil | Pin EIP-8272 `RECENT_ROOT_ADDRESS` to `0x...8272` |
| Aug 11 | [#12066](https://github.com/ethereum/EIPs/pull/12066) | AnkushinDaniil | Ban `SLOTNUM` during validation-prefix execution |
| Aug 11 | [#12121](https://github.com/ethereum/EIPs/pull/12121) | AnkushinDaniil | Link first references to cited proposals |
| Aug 13 | [#12062](https://github.com/ethereum/EIPs/pull/12062) | lightclient | Add explicit execution/state gas budgets to every frame |
| Aug 14 | [#12026](https://github.com/ethereum/EIPs/pull/12026) | AnkushinDaniil | Clarify signature-validation access-list behavior after gas repricing |
| Aug 14 | [#12061](https://github.com/ethereum/EIPs/pull/12061) | AnkushinDaniil | Clarify that receipts have no transaction-level status |
| Aug 14 | [#12109](https://github.com/ethereum/EIPs/pull/12109) | Marchhill | Statically disallow approval scope on atomic-batch frames |
| Aug 17 | [#12113](https://github.com/ethereum/EIPs/pull/12113) | Marchhill | Pin initial warm addresses and storage-key state |
| Aug 18 | [#12187](https://github.com/ethereum/EIPs/pull/12187) | lightclient | Add `SIGDATACOPY (0xb5)` and make SIGPARAM stack arity static |
| Aug 18 | [#12167](https://github.com/ethereum/EIPs/pull/12167) | Marchhill | Relax deterministic validation opcodes and align deploy SSTORE rules |

### Open

| Date | PR | Author | Description |
|---|---|---|---|
| Jun 5 | [#11772](https://github.com/ethereum/EIPs/pull/11772) | vbuterin, Thomas Coratger | Add EIP-8288: Frame type for PQ sig and STARK aggregation (`eip-8288.md`, +508 lines). Fourth compose-by-requires sibling EIP. Still open with requested changes and proof-security review around recursive STARK soundness |
| Jul 30 | [#12039](https://github.com/ethereum/EIPs/pull/12039) | soispoke | Define EIP-8250 keyed-nonce mempool concurrency |
| Jul 30 | [#12041](https://github.com/ethereum/EIPs/pull/12041) | Marchhill | Specify assembled canonical-paymaster reference bytecode |
| Jul 31 | [#12047](https://github.com/ethereum/EIPs/pull/12047) | shemnon | Add EIP-7819 trailing delegation data with an EIP-8141 ML-DSA example |
| Aug 3 | [#12075](https://github.com/ethereum/EIPs/pull/12075) | Marchhill | Add EIP-8361 Transaction Validity Proofs |
| Aug 3 | [#12077](https://github.com/ethereum/EIPs/pull/12077) | Marchhill | Move EIP-7793 to Draft with `TXINDEX` and an EIP-8141 guard frame |
| Aug 5 | [#12110](https://github.com/ethereum/EIPs/pull/12110) | soispoke | Add EIP-8369 VOPS Profiles for FOCIL Eligibility |
| Aug 7 | [#12131](https://github.com/ethereum/EIPs/pull/12131) | AnkushinDaniil | Specify EIP-8272 recent-root runtime code and deployment |
| Aug 11 | [#12139](https://github.com/ethereum/EIPs/pull/12139) | nerolation | Define EIP-8077 source and nonce for frame transactions |
| Aug 13 | [#12157](https://github.com/ethereum/EIPs/pull/12157) | Marchhill | Dispatch an active precompile targeted by a frame |
| Aug 13 | [#12160](https://github.com/ethereum/EIPs/pull/12160) | AnkushinDaniil | Enumerate validation-prefix dependencies for revalidation |
| Aug 13 | [#12162](https://github.com/ethereum/EIPs/pull/12162) | AnkushinDaniil | Relax the pending cap for validation-stable accounts |
| Aug 19 | [#12198](https://github.com/ethereum/EIPs/pull/12198) | nerolation | Make expiry a distinct frame mode |
| Aug 19 | [#12203](https://github.com/ethereum/EIPs/pull/12203) | Marchhill | Preserve expiry-verifier account nonce during activation |

### Related

| Date | PR | Author | Description |
|---|---|---|---|
| Mar 30 | [#1638](https://github.com/ethereum/ERCs/pull/1638) | oxshaman | Add ERC-8211: Smart Batching (ERCs repo). Transport-agnostic composable batch encoding; names EIP-8141 SENDER frames as a forward-compatibility execution path, does not require it |
| Apr 2 | [#11480](https://github.com/ethereum/EIPs/pull/11480) | zacksfF | Add EIP-8215: Hash-Committed Account. Open complementary PQ address-derivation proposal: new accounts derive from a Merkle root of spending conditions, while EIP-8141 upgrades validation for existing accounts |
| Apr 11 | [#11509](https://github.com/ethereum/EIPs/pull/11509) | benaadams | Add EIP-8223: Contract Payer Transaction (alternative/complementary sponsorship proposal) |
| Apr 12 | [#11518](https://github.com/ethereum/EIPs/pull/11518) | benaadams | Add EIP-8224: Counterfactual Transaction (shielded gas funding via ZK proofs) |
| Apr 22 | [#11438](https://github.com/ethereum/EIPs/pull/11438) | Giulio2002 | Add EIP-8202: Scheme-Agile Transactions (alternative AA proposal; PQ signatures on L1 without general AA) |
| Apr 25 | [#11571](https://github.com/ethereum/EIPs/pull/11571) | SirSpudlington | Update EIP-7932: refactor signature registry to be friendlier to EIP-8141 (rename `sigrecover` → `sigaddress`, add `sigverify`/`sigcosts` precompiles for AA use cases) |
| Jun 30 | [#11773](https://github.com/ethereum/EIPs/pull/11773) | forshtat | Move EIP-2542 to Withdrawn, superseded by EIP-8141 (merged Jun 30; first EIP formally withdrawn in favor of frame transactions) |
| Jul 15 | [#1883](https://github.com/ethereum/ERCs/pull/1883) | chunter-cb | Add draft ERC-8340 Transaction Metadata Encoding, an application-layer CBOR layout for EIP-8130's opaque metadata field |
| Jul 28 | [#1794](https://github.com/ethereum/ERCs/pull/1794) | chiranjeev13 | Merge ERC-8286: Modular Accounts for Frame Transactions, the first application-layer standard requiring EIP-8141 |
| Aug 11 | [#12135](https://github.com/ethereum/EIPs/pull/12135) | chunter-cb | Align EIP-8130 with the Keystore contract, add validity windows, and slim the core spec |
| Aug 12 | [#12148](https://github.com/ethereum/EIPs/pull/12148) | chunter-cb | Reconcile EIP-8130 self-actor revocation, authenticator allowlist, and expired grants |
| Aug 14 | [#12173](https://github.com/ethereum/EIPs/pull/12173) | chunter-cb | Right-align EIP-8130 address-derived actor IDs |

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
| Jul 16 | [#11932](https://github.com/ethereum/EIPs/pull/11932) | AnkushinDaniil | Bound the signatures list | Closed without comment; same proposal continues as #11935 |
| Jul 17 | [#11957](https://github.com/ethereum/EIPs/pull/11957) | AnkushinDaniil | Exclude keyed-nonce first-use gas from `MAX_VERIFY_GAS` | Self-withdrawn because it weakened keyed-read amplification protection |
| Jul 18 | [#11964](https://github.com/ethereum/EIPs/pull/11964) | soispoke | Add EIP-8272 `source_id` test vector | Closed because fixed 20-byte + 32-byte encoding was already unambiguous |
| Jul 19 | [#11972](https://github.com/ethereum/EIPs/pull/11972) | soispoke | Clarify NONCE_MANAGER activation | Draft self-closed without comment |
| Jul 19 | [#11973](https://github.com/ethereum/EIPs/pull/11973) | soispoke | Clarify EIP-8272 sighash and predeploy address | Draft self-closed without comment |
| Jul 20 | [#11935](https://github.com/ethereum/EIPs/pull/11935) | svlachakis | Bound the signatures list | Replaced by per-entry gas pricing in merged #11976 |
| Jul 21 | [#11956](https://github.com/ethereum/EIPs/pull/11956) | AnkushinDaniil | Batch sponsor repayment with user execution | Superseded when #11955 excluded VERIFY from atomic batches |
| Jul 24 | [#12004](https://github.com/ethereum/EIPs/pull/12004) | AnkushinDaniil | Add a canonical P256 pseudo-account | P256 is a crypto operation, not a complete account interface |
| Jul 24 | [#12010](https://github.com/ethereum/EIPs/pull/12010) | AnkushinDaniil | Stabilize canonical-paymaster identity | EIP-7702 recognition was flawed; pinned code hash continues in #12012 |
| Jul 27 | [#11971](https://github.com/ethereum/EIPs/pull/11971) | soispoke | Clarify decoding, signing, and activation | P256 portion landed in #11984; remaining combined patch was not justified |
| Jul 29 | [#12011](https://github.com/ethereum/EIPs/pull/12011) | AnkushinDaniil | Formalize protocol signature-scheme registry | Explicit registry rejected; future fixed-cost schemes should use separate specifications |
| Jul 30 | [#12012](https://github.com/ethereum/EIPs/pull/12012) | svlachakis | Canonical-paymaster Solidity draft | Work moved to assembled-runtime PR #12041 |
| Aug 6 | [#12086](https://github.com/ethereum/EIPs/pull/12086) | Marchhill | Frame-aware FOCIL validity | Superseded by EIP-8369 PR #12110 after review found validation work insufficiently bounded |
| Aug 13 | [#12161](https://github.com/ethereum/EIPs/pull/12161) | AnkushinDaniil | Recognize a canonical passkey wallet | Overlapped rejected P256-account directions; general admission work continues in #12160/#12162 |
| Aug 14 | [#11580](https://github.com/ethereum/EIPs/pull/11580) | lightclient | Allow payer to approve before sender | Complexity no longer justified |
| Aug 14 | [#11482](https://github.com/ethereum/EIPs/pull/11482) | derekchiang | Allow precompiles for VERIFY | Stale against current spec; invited to reopen rebased |
| Aug 14 | [#11555](https://github.com/ethereum/EIPs/pull/11555) | derekchiang | Add guarantors | Stale against current spec; invited to reopen rebased |
| Aug 14 | [#11681](https://github.com/ethereum/EIPs/pull/11681) | pedrouid | Bundle guarantors, flexible nonces, and signer binding | Stale against the current sibling and envelope model; invited to reopen rebased |
| Aug 14 | [#12091](https://github.com/ethereum/EIPs/pull/12091) | AnkushinDaniil | Specify block inclusion gating and payer solvency | Resolved by PR #12062 block-level gas accounting |
| Aug 14 | [#12168](https://github.com/ethereum/EIPs/pull/12168) | Marchhill | Restrict SIGPARAM signer introspection | Left to wallet validation logic |
| Aug 16 | [#12155](https://github.com/ethereum/EIPs/pull/12155) | AnkushinDaniil | Forbid approval scope inside atomic batches | Superseded by merged PR #12109 |
| Aug 17 | [#12175](https://github.com/ethereum/EIPs/pull/12175) | leekt | Frame Paymaster Web Service draft | Closed without review |
| Aug 17 | [#12176](https://github.com/ethereum/EIPs/pull/12176) | leekt | Frame transaction gas-estimation RPC draft | Closed without review |

## Key Contributors

| Person | Handle | Role |
|---|---|---|
| Vitalik Buterin | @vbuterin | Co-author of EIP-8141; co-author of EIP-8250 Keyed Nonces (PR #11598, merged May 11) and EIP-8272 Recent Roots (PR #11726, merged Jun 5); primary author with Thomas Coratger of EIP-8288 Frame type for PQ sig and STARK aggregation (PR #11772, opened Jun 5) |
| lightclient (Matt) | @lightclient | Co-author and primary spec maintainer; added per-frame `value` (#11534), repaired custom-verifier signature bytes (#11837), completed two-dimensional frame gas (#12062), and split `SIGDATACOPY` from `SIGPARAM` (#12187) |
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
| jochem-brouwer | @jochem-brouwer | Detailed canonical paymaster review; EIP editor who requested changes on EIP-8288 on Jun 30 |
| Seungmin Jeon | @sm-stack | PoC implementation, atomic batch bit flag idea |
| rmeissner | @rmeissner | Safe team representative, value-in-frames advocate |
| Chiranjeev Mishra (node.cm) | @chiranjeev13 | Forum reviewer and author of ERC-8286 Modular Accounts for Frame Transactions, merged Jul 28 as the first application-layer standard requiring EIP-8141 |
| Ben Adams | @benaadams | Spec tightening (PR #11521, merged Apr 14), author of EIP-8223 (Contract Payer Transaction) and EIP-8224 (Counterfactual Transaction) |
| Jacopo | @jacopo-eth | Proposed FRAMERETURNDATASIZE/FRAMERETURNDATACOPY for multi-step flows |
| Franco Victorio | @fvictorio | Raised question about validation-frame execution ordering vs non-frame txs |
| dionysuzx | @dionysuzx | Hegotá meta-EIP maintainer, submitted PR #11537 moving EIP-8141 to CFI (merged Apr 30) |
| Nero_eth | Nero_eth | ethresear.ch analyst; "Three Gates to Privacy" post framing mempool/FOCIL/VOPS constraints on privacy-pool flows through frame transactions |
| Toni Wahrstätter | @nerolation | Author of PR #11584 (2D nonces, closed), co-author of EIP-8250 Keyed Nonces (PR #11598, merged May 11), author of PR #11662 (EXPIRY_VERIFIER frame, merged May 14), co-author with lightclient of EIP-8266 Expiring Nonces (PR #11692, merged May 22), and co-author with soispoke and vbuterin of EIP-8272 Recent Roots (PR #11726, merged Jun 5). Added to EIP-8141's `author` header in PR #11662 |
| Thomas Thiery | @soispoke | Lead author of EIP-8250 Keyed Nonces (#11598, extended by #11749) and EIP-8272 Recent Roots (#11726); caught EIP-8266's same-block replay bug and completed EIP-8141 fee settlement in #11969 |
| Pedro Gomes | @pedrouid | Author of the bundled guarantor, keyed-nonce, and signer-binding proposals #11643/#11681; #11681 closed stale on Aug 14 after the sibling stack and base envelope moved on |
| German Abal | @ariutokintumi | Co-founder/architect of EVVM (contract-native AA framework); contributed a production-perspective comparison on the magicians thread (post #148, May 7) on per-environment policy, async execution, batch granularity, and reservation primitives |
| Sam Wilson | @SamWilsn | EIP editor; spec-coherence review (post #149, May 8) on naming, empty-target representation, opcode-budget, and `FRAMEDATACOPY` revert semantics |
| trebor | @trebor | Privacy-focused paymaster researcher (Kohaku); raised the Example 3 (ERC-20 paymaster) feasibility question in posts #156-157 (May 21). After matt's clarification in post #158 (Jun 3), trebor committed in post #159 (Jun 4) to a PR clarifying Example 3 in the EIP (Frame 1 should be a signature check by the canonical paymaster, not an on-chain ERC-20 balance read) |
| Thomas Coratger | (not on GitHub directly in PR author list) | Co-author with vbuterin of EIP-8288 (PR #11772, opened Jun 5): Frame type for PQ sig and STARK aggregation. Lean Ethereum tooling contributor |
| Milos Stankovic | @morph-dev | Author of PR #11814 (spec cleanup and clarifications, merged Jul 7): default-code index-0 signature rule later refined by #11954, public-mempool expiry-first rule, signer defaulting to `tx.sender`, validation-prefix flag matching, and ERC-20 sponsor clarification |
| Stavros Vlachakis | @svlachakis | EIP-8141 co-author added by #12062; authored July signature, refund, receipt, blob, fee-bound, direct-evaluation, replacement, log, and paymaster work |
| Daniil Ankushin | @AnkushinDaniil | Implementation-focused contributor; authored sibling corrections, VERIFY batch safety, APPROVE pricing, SLOTNUM validation safety, and current dependency/revalidation proposals |
| Marchhill | @Marchhill | Implementation-focused contributor; authored atomic approval safety #12109, warm-set pinning #12113, validation-opcode cleanup #12167, and open canonical-paymaster and precompile-dispatch work |
| Mislav Javor | @oxshaman | Lead author of ERC-8211 Smart Batching (ERC PR #1638); the proposal names EIP-8141 SENDER frames as a forward-compatibility execution transport, without taking it as a dependency |
| Giulio Rebuffo | @Giulio2002 | Author of EIP-8202 Scheme-Agile Transactions (PR #11438, merged Apr 22) and its Falcon-512 support; raised the "smart wallet tax" critique of EIP-8141 in the Frame-vs-SchemedTransactions thread |
| Dragan Rakita | @rakita | Author of EIP-8175 Composable Transaction (PR #11355, merged Mar 10), the flat-composition alternative to frame-based AA |
| Chris Hunter | @chunter-cb | Author of EIP-8130, EIP-8141's principal alternative; aligned it with the canonical Keystore, validity windows, audit behavior, and right-aligned actor IDs in August |
| nixorokish | @nixorokish | Hegotá meta-EIP PFI maintainer (PR #11786, Jun 9); host of the Svalbard native-AA breakout session |
| pipavlo82 | pipavlo82 | EIP-8288 reviewer; raised omission accountability (posts #2-3, #6) and engaged vbuterin's hash-based-minimalism rationale on the PQ frame thread |
| albert-garreta | albert_g | EIP-8288 reviewer; asked the SPHINCS-vs-lattice scheme-selection question (post #4) that vbuterin answered |
| b-wagn | @b-wagn | EIP-8288 reviewer; raised the recursive-proof knowledge-soundness concern on Jul 3 and suggested a depth-counter public input |
| Zakaria Saif | @zacksfF | Author of EIP-8215 Hash-Committed Account (PR #11480), a complementary PQ address-derivation proposal that positions itself alongside EIP-8141 |
| brettf | @brettf | Raised the arbitrary-signature padding and payer-grief question in Magicians post #167 |
| edg-l | @edg-l | Reported ethrex implementation ambiguities in EIP-8369's mempool, FOCIL, and expiry rules |
| Helkomine | @Helkomine | Reviewed whether EIP-8141's execution cap should apply per frame or per transaction in PR #12062 |
| ilitteri | @ilitteri | Reviewed EIP-8272 runtime edge cases and identified the missing claimed-index transport in EIP-8369 |

## External Resources

- [Live Demo](https://demo.eip-8141.ethrex.xyz/)
- [EIP-8141 Latest Spec](https://eips.ethereum.org/EIPS/eip-8141)
- [EIP-3529: Reduction in refunds](https://eips.ethereum.org/EIPS/eip-3529)
- [EIP-2780: Reduce intrinsic transaction gas and introduce gas charge for transaction value transfer](https://eips.ethereum.org/EIPS/eip-2780)
- [EIP-7708: ETH transfers emit a log](https://eips.ethereum.org/EIPS/eip-7708)
- [EIP-7825: Transaction Gas Limit Cap](https://eips.ethereum.org/EIPS/eip-7825)
- [EIP-8037: State Gas Accounting](https://eips.ethereum.org/EIPS/eip-8037)
- [EIP-7594: PeerDAS - Peer Data Availability Sampling](https://eips.ethereum.org/EIPS/eip-7594)
- [PoC Implementation by sm-stack](https://github.com/sm-stack/eip8141-poc)
- [PoC Writeup](https://hackmd.io/@TB5b8ghoQyChOtUKB0RsOg/B1PhyMK_be)
- [BundleBear EIP-7702 Metrics](https://www.bundlebear.com/eip7702-overview/ethereum)
- [Account Abstraction Link Tree (matt)](https://hackmd.io/@matt/aa-link-tree)
- [Biconomy: Native AA State-of-Art Q1/26](https://blog.biconomy.io/native-account-abstraction-state-of-art-and-pending-proposals-q1-26/)
- [Openfort: What EIP-8141 Means for Developers](https://www.openfort.io/blog/eip-8141-means-for-developers)
- [FOCIL + Native Account Abstraction](https://ethereum-magicians.org/t/focil-native-account-abstraction/27999)
- [AA-VOPS: A Pragmatic Path Towards Validity-Only Partial Statelessness](https://ethresear.ch/t/a-pragmatic-path-towards-validity-only-partial-statelessness-vops/22236#p-54075-vops-and-native-account-abstraction-aavops-9)
- [Frame Transactions Through a Statelessness Lens](https://ethresear.ch/t/frame-transactions-through-a-statelessness-lens/24538)
- [Frame vs Tempo — Two clashing philosophies of native AA](https://x.com/decentrek/status/2031013555898900838)
- [EIP-8141 is Too Unopinionated (jxom)](https://x.com/_jxom/status/2043135281604464905)
- [The case for Frame Transactions: Flexible Foundation with Powerful Defaults](https://x.com/decentrek/status/2036697881512701997)
- [The Evolution of Self-Custody](https://x.com/pedrouid/status/2031716092112929107)
- [Ethereum Wallet UX is changing](https://x.com/pedrouid/status/2042682070997033253)
- [Let us be brave and extend EIP-8141 benefits](https://x.com/pedrouid/status/2051354277520515316)
- [1 contract, 2 fields, 4 features](https://x.com/pedrouid/status/2054584429981659388)
- [Frame Transactions and the Three Gates to Privacy](https://ethresear.ch/t/frame-transactions-and-the-three-gates-to-privacy/24666)
- [Achieving Quantum Safety through Ephemeral Key Pairs and Account Abstraction](https://ethresear.ch/t/achieving-quantum-safety-through-ephemeral-key-pairs-and-account-abstraction/24273)
- [Ethereum L1 Strawmap](https://strawmap.org/)
- [Post-Quantum Ethereum](https://pq.ethereum.org/)
- [Google: Cryptography Migration Timeline](https://blog.google/innovation-and-ai/technology/safety-security/cryptography-migration-timeline/)
- [Your Ethereum Wallet is About to Change Forever](https://dorisgxyz.substack.com/p/your-ethereum-wallet-is-about-to)
- [EIP-8141 Frame Transactions (HackMD)](https://hackmd.io/@dicethedev/HyhbyJA3bg)
- [Frame Transactions vs. SchemedTransactions](https://ethereum-magicians.org/t/frame-transactions-vs-schemedtransactions-for-post-quantum-ethereum/28056)
- [EIP-8266: Expiring Nonces for Frame Transactions](https://ethereum-magicians.org/t/eip-8266-expiring-nonces-for-frame-transactions/28575)
- [EIP-8272: Recent Roots for Frame Transactions](https://ethereum-magicians.org/t/eip-8272-recent-roots-for-frame-transactions/28621)
- [EIP-8288: Frame type for PQ sig and STARK aggregation](https://ethereum-magicians.org/t/eip-frame-type-for-quantum-resistant-signature-and-stark-aggregation/28723)
- [ERC-8286: Modular Accounts for Frame Transactions](https://ethereum-magicians.org/t/erc-8286-modular-accounts-for-frame-transactions/28695)
- [ERC-8211: Smart Batching](https://ethereum-magicians.org/t/erc-8211-smart-batching/28135)
- [Svalbard AA Breakout Session Notes](https://hackmd.io/@nixorokish/svalbard-aa-breakout)
- [AllWalletDevs #40, July 15, 2026](https://ethereum-magicians.org/t/allwalletdevs-40-july-15-2026/28858)
- [All Core Devs - Execution (ACDE) #241, July 16, 2026](https://ethereum-magicians.org/t/all-core-devs-execution-acde-241-july-16-2026/29001)

## Competing Standards

- [EIP-8130: AA by Account Configuration](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-8130.md) — [Magicians thread](https://ethereum-magicians.org/t/eip-8130-account-abstraction-by-account-configurations/25952)
- [EIP-8175: Composable Transaction](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-8175.md) — [Magicians thread](https://ethereum-magicians.org/t/eip-8175-composable-transaction/27850)
- [EIP-8202: Schemed Transaction](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-8202.md) — [Magicians thread](https://ethereum-magicians.org/t/eip-8202-schemed-transaction/28044)
- [EIP-8223: Contract Payer Transaction](https://github.com/benaadams/EIPs/blob/899fd04868d2be4d13a484e892c78abee4bb1e12/EIPS/eip-8223.md) — [Magicians thread](https://ethereum-magicians.org/t/eip-8223-contract-payer-transactions/28202)
- [EIP-8224: Counterfactual Transaction](https://github.com/benaadams/EIPs/blob/7f91aade4410a071bf37694fd43e6540bbf8aef6/EIPS/eip-8224.md) — [Magicians thread](https://ethereum-magicians.org/t/eip-8224-counterfactual-transaction/28205)
- [EIP-8215: Hash-Committed Account](https://github.com/zacksfF/EIPs/blob/cffc3ba51e00009fcd50402be030bd99572f4df6/EIPS/eip-8215.md) — [Magicians thread](https://ethereum-magicians.org/t/eip-8215-hash-committed-account-hca/28094)
- [Tempo-like Transaction (gakonst)](https://gist.github.com/gakonst/00117aa2a1cd327f515bc08fb807102e)
