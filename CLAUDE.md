# EIP-8141 Research - Project Instructions

This repo tracks the evolution of EIP-8141 (Frame Transaction). It is a VitePress documentation site with research documents, a landing page, and supporting infrastructure. These instructions ensure any contributor (human or agent) can update and maintain the repo consistently.

---

## Repository Structure

```
├── CLAUDE.md                          # This file - project instructions
├── README.md                          # GitHub landing page with document index and synthesis
├── package.json                       # VitePress dev dependency
├── vercel.json                        # 308 redirects from old paths to current slugs (URL stability)
├── docs/
│   ├── index.md                       # VitePress home page (hero, audience routes, examples)
│   ├── current-spec.md                # Current spec overview (execution + mempool model)
│   ├── feedback-evolution.md          # Community feedback organized by chronological phase
│   ├── original-spec.md               # Original Jan 29 submission and what it lacked
│   ├── merged-changes.md              # Every PR (merged, closed, open) with rationale
│   ├── original-vs-latest.md          # Side-by-side diff of structural changes
│   ├── eoa-support.md                 # EOA support via default code, replacing EIP-7702 for common cases
│   ├── pq-roadmap.md                  # Seven-stage PQ roadmap from EIP-8141 foundation to private L1
│   ├── developer-tooling.md           # Bear/bull cases for wallet/app dev adoption, protocol defaults vs ERC fragmentation
│   ├── mempool-strategy.md            # Two-tier mempool architecture, VOPS extension, merkle escape hatch, no-relayers
│   ├── vops-compatibility.md          # VOPS compatibility (validation state, state growth, trilemma, witnesses)
│   ├── competing-standards.md         # Comparative analysis across all alternative proposals
│   ├── eip-8130.md                    # Alternative: Account Abstraction by Account Configuration (Coinbase/Base)
│   ├── eip-8175.md                    # Alternative: Composable Transaction (rakita)
│   ├── eip-8202.md                    # Alternative: Scheme-Agile Transactions (Giulio2002, benaadams)
│   ├── eip-8223.md                    # Complementary: Contract Payer Transaction (static sponsorship)
│   ├── eip-8224.md                    # Complementary: Counterfactual Transaction (shielded gas funding)
│   ├── eip-xxxx.md                    # Alternative: Tempo-like Transactions (gakonst, pre-draft gist)
│   ├── faq.md                         # Indexed Q&A (section.question format, e.g. 2.3)
│   ├── glossary.md                    # Comprehensive glossary of jargon used across the site, grouped by category
│   ├── appendix.md                    # Sources, PR timeline, contributors, external resources, competing standards
│   └── .vitepress/
│       ├── config.ts                  # Nav, sidebar, social links
│       └── theme/
│           ├── index.ts               # Theme entry, imports custom.css
│           ├── Layout.vue             # Wraps default layout, adds Footer
│           ├── Footer.vue             # 5-column footer (site title, Spec, Topics, Alternatives, Resources)
│           └── custom.css             # Global table styles
```

---

## Document Ordering and Categories

- Docs use slug filenames without numeric prefixes (e.g. `current-spec.md`, not `01-current-spec.md`)
- Order is defined manually in `config.ts` (nav + sidebar), `Footer.vue`, and `README.md`, not by filename
- Docs are grouped into four categories. The same categories apply across the header nav dropdowns, sidebar groups, footer columns, and README tables:

| Category | Purpose | Current docs |
|---|---|---|
| **Spec** | How EIP-8141 works and how it got here | Current Spec, Feedback Evolution, Original Spec, Merged Changes, Original vs Latest |
| **Topics** | Analytical deep-dives beyond the spec itself | EOA Support, PQ Roadmap, Developer Tooling, Mempool Strategy, VOPS Compatibility, Competing Standards |
| **Alternatives** | Per-proposal pages for each alternative/complementary EIP | EIP-8130, EIP-8175, EIP-8202, EIP-8223, EIP-8224, EIP-XXXX (Tempo-like) |
| **Resources** | Reference material, index, Q&A | FAQ, Glossary, Appendix |

- `appendix.md` is always **last** in the Resources group
- The FAQ (`faq.md`) uses indexed questions: sections are numbered (1–10), questions are `section.question` (e.g., 1.1, 2.3, 8.5)
- When adding a new document, decide which category it belongs to, then create `docs/<slug>.md` and update: `config.ts` (nav + sidebar), `Footer.vue`, and `README.md` in the matching category
- **Alternatives never appear in the top nav header.** The header is reserved for Home, Spec, Topics, FAQ, and Demo. Alternatives live in the sidebar (as a fourth group) and in the footer (as a column), but not in the nav. Reason: the top nav is a reader's primary path through the site, and leading with Alternatives would foreground competing proposals before EIP-8141 itself. When adding or renaming an alternative proposal page, update `config.ts` sidebar only — do **not** add it to `nav` — and update `Footer.vue` under the Alternatives column.

### URL stability and Vercel redirects

`vercel.json` at the repo root maintains 308 redirects from old paths to current slugs. The site is hosted on Vercel; redirects are server-side at the edge.

- **Destinations are extensionless** (e.g. `/current-spec`) since `cleanUrls: true` is set in `config.ts` and canonical URLs have no `.html`
- **Sources cover all variants**: each renamed doc has redirects for the old extensionless path, the old `.html` path, and the new-slug `.html` path. Example for `current-spec.md` (renamed from `01-current-spec.md`):
  - `/01-current-spec` → `/current-spec`
  - `/01-current-spec.html` → `/current-spec`
  - `/current-spec.html` → `/current-spec`
- **New docs** (no rename history) get a single `.html` redirect to handle visitors who type `.html`: `/<slug>.html` → `/<slug>`
- When renaming a doc or restructuring URLs, **add new redirects, do not remove existing ones**. External backlinks to old paths must keep working.

---

## Website Configuration

### VitePress (docs/.vitepress/)

- **config.ts**: Defines nav header and sidebar. Nav has: Home, Spec (dropdown), Topics (dropdown), FAQ, Demo (external link). **Alternatives is intentionally not in the nav** (see the Alternatives-never-in-nav rule in Document Ordering and Categories). Sidebar has four groups (Spec, Topics, Alternatives, Resources) with Appendix last in Resources. Keep these in sync when adding/removing docs.
- **Footer.vue**: 4-column grid (Spec, Topics, Competing Standards anchors, Resources with external links). Does NOT include FAQ or Appendix.
- **Layout.vue**: Wraps VitePress default layout, injects Footer via `#layout-bottom` slot.
- **custom.css**: Global rule `white-space: nowrap` on first column of all tables. Do not remove - prevents column text wrapping across all docs.
- **theme/index.ts**: Imports DefaultTheme, Layout, and custom.css. Keep imports here when adding new CSS.

### Landing Page (docs/index.md)

- Uses VitePress `layout: home` with hero, actions, and features in YAML frontmatter
- Hero buttons: "Read the Spec Overview" (brand), "Try the Demo" (alt)
- Below frontmatter: markdown sections with frame mode table, examples, opcode table
- Frame modes table lists DEFAULT first (it is mode 0 in the spec)
- Examples that include gas sponsorship should include a DEFAULT post-op frame for sponsor refund
- Feature card `details` strings in the YAML frontmatter must be 100 characters or fewer to keep the grid visually balanced

### Formatting Rules

- Tables: first column never wraps (enforced by CSS). Keep first-column text concise.
- Links between docs: use root-relative paths (`/current-spec`, not `./current-spec.md`)
- Links to external sites: use full URLs with `target="_blank"` in Vue templates
- No emojis in any file
- **Em dashes (`—`) are restricted to four use cases only**. Any other use is a violation and must be rewritten. The four allowed contexts:
  1. **Titles** with a subtitle or descriptor, including markdown link titles: `## Broad Spec Tightening — April 14, 2026`, `[Frame vs Tempo — Two clashing philosophies of native AA](...)`.
  2. **Dates** attached to a topic in a header or attribution line: `*benaadams — PR #11521, Apr 14*`.
  3. **Number ranges**: `posts #124-134` (note: number ranges use a hyphen `-`, not an em dash, but a date range like `(Mar 10 – Mar 25)` uses an en dash `–`; do not use em dashes for ranges at all).
  4. **Lists and table cells** to separate a topic from its description: `- EIP-8130 — Account Configuration` or `| Apr 14 | #11521 — Tighten spec |`. This includes "Read Next" list items (`- [Current Spec](/current-spec) — the mechanism behind the answers above`) and bold-labeled definition paragraphs that function as a definition list (`**VOPS** — short for *Validity-Only Partial Statelessness*. ...`).
- **Forbidden uses** (these are the common mistakes to avoid):
  - As parenthetical brackets in a sentence (`Stages 2–7 are future research — separate EIPs, separate forks — and therefore...`). Rewrite with parentheses, commas, or two sentences.
  - As a colon substitute in a sentence (`Not for frame transactions — they enter the mempool...`). Rewrite with a period, a semicolon, or a colon.
  - Inside FAQ answers, topic-doc prose, narrative phase summaries, or "Why this mattered" lines. These are all prose and must use commas, periods, semicolons, colons, or parentheses.
- **Test before using**: ask "is this a title, a date attached to a label, a range, or a list/table cell?" If no, do not use an em dash. When in doubt, rewrite without one.
- Keep answers in FAQ to 1-2 lines, always with question and answer on separate lines

---

## Research Data Sources

| Input | Source | Purpose |
|---|---|---|
| Latest spec | `https://github.com/ethereum/EIPs/blob/master/EIPS/eip-8141.md` | Ground truth for current state |
| Closed/merged PRs | `gh pr list --repo ethereum/EIPs --search "8141" --state all` | Chronological record of spec changes |
| Open PRs | `gh pr list --repo ethereum/EIPs --search "8141" --state open` | Pending proposals and context |
| ERC-layer proposals | `gh pr list --repo ethereum/ERCs --search "8141" --state all` and `--search "frame transaction"` | Application-layer standards that build on or reference EIP-8141 (e.g. ERC-8286 modular accounts, ERC-8211 smart batching). The `ethereum/ERCs` repo is separate from `ethereum/EIPs` and is easy to miss |
| PR comments | `gh pr view --repo ethereum/EIPs <number> --comments` (or `--repo ethereum/ERCs`) | Design rationale not in the spec |
| EthMagicians thread | `https://ethereum-magicians.org/t/frame-transaction/27617` | Community feedback and debates |
| ethresear.ch | `https://ethresear.ch/t/frame-transactions-through-a-statelessness-lens/24538` | Statelessness and mempool concerns |

---

## Update Process

When updating the repo to capture new developments, follow this checklist in order:

### 1. Gather New Data

- **Fetch latest spec** from `master` branch - compare against `current-spec.md`
- **Search for new PRs** since the last documented PR (check `appendix.md` for the most recent)
- **Search the `ethereum/ERCs` repo too**, not just `ethereum/EIPs`. Run `gh pr list --repo ethereum/ERCs --search "8141" --state all` and `--search "frame transaction"`, then for any hit confirm it is a real reference (check the diff for an `8141` in the `requires:` header or a substantive mention, not a coincidental digit-run). Application-layer ERCs that build on EIP-8141 (ERC-8286, `requires: 7579, 8141`) or name it as a future transport (ERC-8211 Smart Batching) live here and are invisible to an EIPs-only search.
- **Read all comments** on new and updated PRs - PR comments often contain critical design rationale
- **Read new EthMagicians posts** beyond the last documented post number (check `appendix.md`)
- **Check competing proposals** for new PRs or discussion threads (EIP-8130, EIP-8175, EIP-8202)
- **Check ethresear.ch** for new posts in relevant threads
- **Classify every new merged PR by significance.** A PR is **significant** if any of the following holds: merged diff >100 lines, touches more than one spec area (frame model / default code / limits / deployment / security / mempool), adds or renames an opcode, changes a constant (e.g. `MAX_FRAMES`), or triggers a substantive review debate. Significant PRs require the mandatory fan-out in step 2; do not just append a one-line entry.

### 2. Update Documents

Each document has a specific scope. Update only the relevant ones:

| Document | When to update |
|---|---|
| `current-spec.md` | Any merged PR that changes spec text or constants. Pending and open proposals are tracked in `merged-changes.md` under `## Active/Open PRs`, not here. |
| `feedback-evolution.md` | New EthMagicians posts, new ethresear.ch posts, AND every significant merged PR (see criteria in step 1) — add a named entry in the current phase capturing the rationale and review debate, not just the diff. Audit phase length and thematic coherence per §2b; split, rename, or renumber phases when they drift. |
| `original-spec.md` | Rarely - only if new context about the original submission surfaces |
| `merged-changes.md` | Every new PR (merged, closed, opened) and every status change on existing open PRs. Significant merged PRs get their own dated section with per-area bullets and a review-discussion paragraph, not a one-liner. Document structure is enforced per §2c: chronological merged sections first, then `## Active/Open PRs`, then `## Rejected/Closed PRs`. The two status buckets always sit at the end. |
| `original-vs-latest.md` | Any merged PR that changes structural behavior, constants, opcode set, default-code rules, or deployment mechanism |
| `competing-standards.md` | New competing EIPs, new PRs on existing competitors, new comparison threads |
| `vops-compatibility.md` | New VOPS/statelessness developments, state growth data, witness cost changes |
| `faq.md` | New questions arise from community, or answers change due to spec updates. Also add or refresh a `section.question` Q&A whenever a Spec/Topics/Alternatives change alters a reader-facing answer (new sibling EIP, new constant, new adoption standard); see §2e. |
| `glossary.md` | Any change that introduces a new term, opcode, frame mode, constant, system contract, envelope field, sibling/related EIP, ERC, or named concept (e.g. compose-by-requires, guarantors), or renames/changes an existing one. The glossary indexes jargon from every other doc; audit it whenever Spec/Topics/Alternatives change (see §2e). |
| `mempool-strategy.md` | New mempool tier proposals, VOPS-extension changes, witness-cost analysis, relayer-substitute patterns |
| `developer-tooling.md` | New bear/bull arguments emerge from wallet/app developers, or protocol defaults expand |
| `eoa-support.md` | Default code spec changes (new sig schemes, scope rules), or new comparisons to EIP-7702 surfaces |
| `pq-roadmap.md` | New PQ precompile EIPs, ephemeral key research updates, encrypted mempool progress, secp256k1 revocability proposals, strawmap updates |
| `appendix.md` | Always — update PR timeline (Merged / Open / Related / Closed), contributor list, external resources, competing standards. Structure and link-formatting rules in §2d. Evergreen: no sync-snapshot counters in appendix link text. |

### 2a. Mandatory Fan-Out for Significant Merged PRs

When a PR is classified as significant (step 1 criteria), a partial update is a bug. Work through this fan-out in order for each such PR before moving on:

1. **`merged-changes.md`** — add a dedicated section titled by theme and date (e.g. `## Broad Spec Tightening — April 14, 2026`). Include: author + merge date line, per-spec-area bullets (frame model, APPROVE/VERIFY, default code, limits, deployment, security, paymaster, whichever apply), a **Key review discussion** paragraph quoting or naming the reviewers and their objections/approvals, and a one-line significance note comparing it to past structural changes.
2. **`feedback-evolution.md`** — add a named entry inside the current phase's subsection for the PR (title it descriptively, e.g. `### Broad Spec Tightening (Merged)`). Cite `PR #<num>, submitted <date>, merged <date>`. Summarize the rationale and the review debate, not the diff. This doc is about *why* things moved, not *what* moved.
3. **`current-spec.md`** — update the spec body to match the new behavior. (Pending-proposals tracking lives in `merged-changes.md` under `## Active/Open PRs`; the corresponding move from Active/Open to a chronological merged section happens in step 1 above.)
4. **`original-vs-latest.md`** — refresh the Latest column for every structural change the PR introduced (new opcodes, renamed fields, changed constants, altered default-code rules).
5. **Topic docs** — for every area the PR touched, update the matching topic doc:
    - default code / EOA paymaster / signature schemes → `eoa-support.md`
    - mempool policy, VERIFY carve-outs, approval-scope rules affecting mempool → `mempool-strategy.md`
    - validation-state / witness / VOPS-relevant changes → `vops-compatibility.md`
    - PQ-relevant signature or precompile changes → `pq-roadmap.md`
    - wallet/app surface changes (new opcodes, default-code ergonomics) → `developer-tooling.md`
    - interactions with competing EIPs or new requires/supersedes → `competing-standards.md`
6. **`faq.md`** — add or update any question whose answer the PR changed (e.g. constants like `MAX_FRAMES`, opcode counts, default-code behavior).
7. **`appendix.md`** — always add the PR to the timeline with correct merge date and author.
8. **`README.md`** — update the "Last updated" date and any coverage counters (PR count, post count).

Skipping a step because "it's probably still accurate" is the failure mode this fan-out exists to prevent. If a step truly doesn't apply, confirm by reading the target doc before moving on.

### 2b. Managing Phases in `feedback-evolution.md`

`feedback-evolution.md` is organized into chronological + thematic phases. Each sync, audit whether the current trailing phase is still the right container for new entries, or whether the structure needs rebalancing.

**Phases are chronological AND thematic.** Each phase has a date range and a named theme. The theme is the character of what happened in that window (e.g. "Operational Constraints & Mempool Safety", "Sibling EIPs and Broad Spec Tightening"), not a PR number or an author. The date range is a sync-snapshot per §4; extend the end date when new entries land in the trailing phase.

**Hard 600-word cap per phase, enforced by tightening, not splitting.** Every phase must be 600 words or fewer (run the per-phase word-count check below). When a phase exceeds 600, the first remedy is always to **tighten the prose**, not to split into more phases. Splitting is a last resort reserved for genuine thematic pivots (see below); it is never an acceptable way to dodge the cap, because splitting keeps the same total words and only fragments the timeline. The mechanism for staying under 600:

- **This doc is "why it moved," not "what moved."** Mechanics (diffs, constants, opcode names, field renames, system-contract layouts, line counts) live in `merged-changes.md`. A phase entry states the rationale and the review debate, then cites the PR/post. If a sentence describes *what the code does* rather than *why it moved or who objected*, cut it and let `merged-changes.md` carry it.
- **Cite, don't restate.** Every claim still links a PR number, post number, or commit (the Traceability principle is non-negotiable), but the citation replaces the retelling. "PR #11726 merged Jun 5 (soispoke, vbuterin, nerolation)" plus one sentence of rationale beats a paragraph of mechanics.
- **One tight paragraph per sub-section.** Most `### ` sub-sections should be a single paragraph: what happened, why it mattered, who pushed back. Drop blockquotes unless the exact wording is load-bearing.
- **Compress resolved history harder than live work.** Older phases whose watch-items are all settled should be the tersest; the trailing phase may use more of its 600-word budget while events are still unfolding.

**When to split a phase (last resort).** Only split when a phase has crossed a genuine chronological pivot that changes the character of the thread (a major spec merge, a sibling EIP appearing, a fork-inclusion milestone, a load-bearing external analysis) AND tightening to 600 words would force unrelated themes into one container. If tightening alone gets the phase under 600 while keeping a coherent theme, do not split.

**How to split.** Pick boundaries at genuine pivots, not arbitrary dates. "The week the sibling EIPs appeared" or "after the value-field merge" are pivots; "first week of April" is not. Name each new phase by its theme. Both resulting phases must independently obey the 600-word cap. Preserve entry order; tighten as you regroup. Keep numbering contiguous: if you delete or absorb a phase, renumber the rest so phases run 1-2-3-4-…, never 1-2-3-5-6.

**Sub-section order within a phase.** Sub-sections (`### ` headings) inside a phase must be in chronological order by **primary event date**, earliest first. The primary event date is the merge date for merged PRs, the open date for open PRs, the post date for forum discussions (use the earliest post if the entry spans multiple), and the publication date for external analyses. If two sub-sections share a primary date, use the finer-grained timestamp (merge time precedes open time on the same day) or, if that is still ambiguous, place the spec-change entry before the discussion entry. When moving an entry between phases because its primary date has shifted, preserve this ordering in the destination phase. Do not re-order thematically once a phase is populated.

**When to delete a phase.** If a phase's content is fully covered elsewhere in the repo (e.g. competing-standard bullets belong in `competing-standards.md` and per-EIP pages, not a timeline stub) and it adds no chronological signal, delete it. Collapse to a one-line pointer at the end of the adjacent phase rather than keeping an empty shell. Then renumber.

**How to run the word-count check.** From the repo root:

```
awk '/^## Phase/ { if (name != "") print name ": " wc; name=$0; wc=0; next } /^## / { if (name != "") print name ": " wc; name=""; wc=0; next } name != "" { wc += NF } END { if (name != "") print name ": " wc }' docs/feedback-evolution.md
```

Any phase over 600 words must be tightened back under the cap before committing (split only per the last-resort rule above).

### 2c. Structure of `merged-changes.md`

`merged-changes.md` is primarily **chronological** for merged PRs, but the two status buckets are always pinned to the end. Required top-level structure:

1. **Chronological merged sections**, in ascending date order. Each section is titled `## <Theme> — <Month> <Day>, <Year>` (e.g. `## Broad Spec Tightening — April 14, 2026`) and groups one or more related merges. New merges go into a new dated section, or into the most-recent section if they extend its theme.
2. **`## Active/Open PRs`** — a single section listing every currently-open PR with author, rationale, proposed change, and status line. The heading itself carries no date; immediately under the heading, put a one-line italic sync-snapshot note (e.g. `*As of April 20, 2026.*`). Refresh that note each sync per §4.
3. **`## Rejected/Closed PRs`** — a single section, always last. Lists closed-without-merge PRs with the rejection reason (quoted from the reviewer when possible).

**Placement rule, stated as a hard invariant**: the last two `## `-level sections in the file must be `## Active/Open PRs` followed by `## Rejected/Closed PRs`, in that order. No merged chronological section ever appears after them.

**When a PR changes status this sync**:
- Newly merged → move its entry from `## Active/Open PRs` into a chronological section (new or existing) at the correct date position. Do not leave a stub in the Open list.
- Newly opened → add to `## Active/Open PRs`.
- Newly closed without merge → move from `## Active/Open PRs` to `## Rejected/Closed PRs` with a one-line rejection reason.

**Verification**: the `## `-level headings in order must read as *N chronological sections (ascending date)* + `## Active/Open PRs` + `## Rejected/Closed PRs`. Grep `^## ` to confirm before committing.

### 2d. Structure of `appendix.md`

`appendix.md` is the index, not a place to restate content. Keep it terse and non-duplicative.

**Section set and order** (all present, in this order):
1. `## Sources` — authoritative upstream sources (spec, PR list, discussion thread).
2. `## Complete PR Timeline` — four subsections: `### Merged`, `### Open`, `### Related`, `### Closed (not merged)`.
3. `## Key Contributors` — table.
4. `## External Resources` — link list.
5. `## Competing Standards` — link list.

Do not add ad-hoc sections. If something looks like it wants a new section, ask whether it actually belongs in a topic doc or an existing list.

**External Resources formatting rule**: every entry is **bare-title-link only**, in the form `- [Exact Title](URL)`. No descriptions, no author/date suffixes after the link, no parenthetical counters in the link text (e.g. "(140 posts)"). The reason to include a link is that the title alone is enough for a reader to decide whether to follow it. If the title is not self-explanatory, the source belongs in a topic doc where the context lives, not in the appendix index.

**Competing Standards formatting rule**: one line per alternative proposal, linking the canonical source. For EIP-numbered proposals, link the GitHub EIP file and append the Magicians thread if one exists: `- [EIP-NNNN: Title](github-url) — [Magicians thread](forum-url)`. For non-EIP proposals (gists, HackMD drafts, private forks), link the authoritative source directly and identify the author in the title (e.g. `Tempo-like Transaction (gakonst)`). Every alternative mentioned anywhere in `competing-standards.md` or the per-EIP pages must have a corresponding link here.

**Competing Standards is proposals, not commentary.** Comparison threads, analyses, and "X vs Y" posts go in `## External Resources`, never in `## Competing Standards`. The test: does the link *define* an alternative proposal, or does it *discuss* alternatives? Only the former belongs in Competing Standards.

**De-duplication rule**: a URL appears in at most one section of `appendix.md`. If the EthMagicians thread is already in `## External Resources`, it does not also get a `## EthMagicians Discussion` section with post counters. If a link belongs in both External Resources and a narrative topic doc, the appendix gets the bare link and the topic doc gets the context.

**Post-count and sync-snapshot tracking**: do not embed sync-snapshot counters (post count, PR count, "as of Apr X") inside appendix link text. The global sync date lives in memory and in `README.md`; `appendix.md` is evergreen.

### 2e. Keeping the reference docs (`appendix.md`, `glossary.md`, `faq.md`) in sync

The Appendix (sources index), Glossary (concepts index), and FAQ (common questions) are **downstream of every other section**. They do not generate new facts; they index what the Spec, Topics, and Alternatives docs already say. Because of that they drift silently: a sync can add a sibling EIP, a new opcode, a system contract, or an external source to the primary docs and leave the indexes stale. Treat updating them as a required closing step of every sync, not an optional one.

After editing any Spec, Topics, or Alternatives doc, propagate as follows:

- **Appendix** — for every PR, contributor, external link, or competing/complementary proposal newly referenced in another doc, confirm it has a corresponding entry (PR timeline / Key Contributors / External Resources / Competing Standards). The appendix is the union of all sources cited anywhere on the site; a source that appears in a topic doc but not the appendix is a bug. Structure per §2d.
- **Glossary** — for every new term a change introduces (opcode, frame mode, constant, system contract, envelope field, sibling/related EIP, ERC, or named concept such as compose-by-requires or guarantors), add a glossary entry in the correct category, alphabetical within the category. A term that appears in bold or code across a doc but is undefined in the glossary is a gap. When a merged PR changes a constant or renames a term, update the existing entry rather than leaving it stale.
- **FAQ** — when a change answers, or changes the answer to, a question a reader would plausibly ask, add or update a `section.question`-indexed Q&A (1-2 lines, question and answer on separate lines). New sibling EIPs, new default-code behavior, new adoption standards, and changed constants are the common triggers.

The significant-PR fan-out (§2a) routes structural merges into the relevant Spec and Topics docs; this step ensures the same change also lands in the three indexes. A sync that updates a primary doc without checking these three is incomplete.

### 3. Update Infrastructure

- **README.md**: Update "Last updated" date and coverage numbers. Update document table if docs added.
- **config.ts**: Update nav/sidebar if docs added or renamed
- **Footer.vue**: Update if docs added to Spec or Topics column, or new external links added to Resources

### 4. Verify Consistency

After updates, check:
- All internal links between docs are valid (root-relative paths match filenames)
- PR numbers, post numbers, and dates are consistent across documents
- The appendix PR timeline matches what's described in `merged-changes.md`
- FAQ cross-references to `vops-compatibility.md` and `mempool-strategy.md` use correct anchor slugs
- **Significant-PR fan-out audit.** For every PR classified as significant this sync, grep the PR number across `docs/` and confirm it appears in at least: `merged-changes.md` (dedicated section), `feedback-evolution.md` (named entry), `current-spec.md` (spec text reflects it), `original-vs-latest.md` (Latest column), and `appendix.md` (timeline). Any doc listed in the fan-out for that PR's touched areas must also reference it. If a PR is missing from any of these, the sync is incomplete: do not commit.
- **Phase balance audit.** Run the per-phase word-count check from §2b on `feedback-evolution.md`. Every phase must be ≤600 words; tighten any phase over the cap per §2b before committing (split only as the last resort §2b describes). Confirm phase numbers are contiguous (no gaps from deletions/renames).
- **Sub-section chronology audit.** For every phase in `feedback-evolution.md`, read the `### ` headings in order and confirm each sub-section's primary event date (merge date, open date, post date, publication date, per §2b) is greater than or equal to the previous. If a sub-section is out of order, move it into the correct chronological position before committing. A sub-section whose primary date falls outside the phase's date range belongs in a different phase; relocate it there.
- **`merged-changes.md` structure audit.** Grep `^## ` on the file. The last two headings must be `## Active/Open PRs` (no date in the title; italic `*As of <date>.*` sits under it) and `## Rejected/Closed PRs`, in that order. Every chronological section above them must be in ascending date order. If either invariant is violated, fix before committing.
- **`appendix.md` structure audit.** Confirm per §2d: sections appear in the fixed order (Sources, Complete PR Timeline, Key Contributors, External Resources, Competing Standards); no ad-hoc section duplicates content that already lives in External Resources; every External Resources entry is a bare-title link with no description or counter; every competing proposal mentioned in `competing-standards.md` or a per-EIP page has a link under Competing Standards; no comparison threads or analyses are listed under Competing Standards.
- **Em-dash audit.** Grep `—` across `docs/` and classify every hit against the four allowed contexts in Formatting Rules (titles, dates attached to a label, ranges, list/table topic-description separators). Any em dash used as a parenthetical separator or colon substitute inside a sentence is a violation and must be rewritten with commas, a period, a semicolon, a colon, or parentheses. This applies to FAQ answers, topic-doc prose, phase summaries, and "Why this mattered" lines equally.
- **Topic-doc TL;DR audit.** For every file in the Topics category (`eoa-support.md`, `pq-roadmap.md`, `developer-tooling.md`, `mempool-strategy.md`, `vops-compatibility.md`, `competing-standards.md`), grep `^## ` and confirm the first match is `## TL;DR`. Any other section appearing before the TL;DR is a violation per Topic-doc structure; move it below the TL;DR before committing.
- **Reference-doc completeness audit (§2e).** Confirm the three index docs absorbed this sync's changes. (1) **Glossary**: for every new opcode, constant, system-contract name, frame mode, sibling/related EIP, ERC, and named concept introduced this sync, grep `docs/glossary.md` and confirm an entry exists; confirm no existing entry contradicts a merged change (e.g. a changed constant or governance status). (2) **Appendix**: confirm every PR, contributor, external URL, and competing/complementary proposal newly cited in any doc appears in the appendix (the `appendix.md` structure audit above covers placement). (3) **FAQ**: confirm any reader-facing answer changed this sync is reflected. A primary-doc edit without the matching index update is an incomplete sync.

**Distinguish event-timestamp dates from sync-snapshot dates:**

| Date type | Examples | Action |
|---|---|---|
| **Event timestamp** | "PR #11481 merged Apr 2", "post #137 (Apr 10)", "derekchiang's comment (Apr 9)" | Never change. These are real events. |
| **Sync-snapshot date** | "Latest (Apr 8)" header in `original-vs-latest.md`, "Pending Proposals (as of Apr 13)" in `current-spec.md`, the italic `*As of Apr 13.*` line under `## Active/Open PRs` in `merged-changes.md`, phase-range headers like "Phase 5 (Mar 26 – Apr 13)" in `feedback-evolution.md` | Refresh on every sync to reflect what is actually current as of the new sync date. |

A common stale-date trap: phase-range headers in `feedback-evolution.md` get extended when the latest sync adds entries to that phase, but the header end-date isn't updated.

---

## Writing Principles

- **Traceability**: Every claim links to a specific PR number, EthMagicians post number, or commit
- **Direct quotes**: Use author quotes to capture rationale - paraphrasing loses nuance
- **Rejected PRs matter**: Closed PRs reveal the design space the authors explored and chose against
- **Comments over descriptions**: PR review threads often contain more insight than the PR description
- **Chronological + thematic**: `merged-changes.md` is chronological; `feedback-evolution.md` groups by thematic phase
- **What's absent**: Document what the spec does NOT have - gaps explain future direction
- **FAQ brevity**: Answers are 1-2 lines max. Link to docs or external sources for detail.

### Topic-doc structure

Topic docs (the Topics category) have a **2,500-word maximum**. If a topic doc exceeds this limit, tighten the prose before adding new content.

Topic docs follow a fixed shape, enforced in this order:

1. **`## TL;DR` is always the first `##`-level section, with no other section above it.** This is a hard invariant: the only allowed content between the `#` page title and `## TL;DR` is the initial `---` horizontal rule. No framing paragraphs, no "About this page" blocks, no "Competing vs Complementary" distinctions, no glossary entries. If a framing section feels necessary, it goes *after* the TL;DR, not before it.
2. **Numbered or named sections** in the middle. These carry the analysis, tables, and position statements.
3. **Summary at the end** that mirrors the TL;DR with any additional nuance.

Why the invariant: the TL;DR is the reader's landing contract. If the first screen is a framing preamble instead of the one-paragraph takeaway, readers who bounce at the first section get nothing useful. A reader who reads only the TL;DR should still come away with the doc's thesis.

When a topic doc presents opposing positions (e.g. Bear Case / Bull Case in `developer-tooling.md`), label them explicitly and include a one-line *Position* statement before the argument. Each position should have its own source link.

**Verification**: for every file in the Topics category, the first `##`-level heading must be `## TL;DR`. Grep `^## ` on the file and confirm the first match is `## TL;DR`.

### Source attribution

Attribution rules vary by document, set deliberately:

| Document | Attribution policy |
|---|---|
| `vops-compatibility.md` | **Never attribute** to named individuals. Present tradeoffs, not people. |
| `developer-tooling.md` | **Always attribute** Bear and Bull positions via a `[source](URL)` link next to the position header. |
| All other docs | Cite PR numbers, post numbers, commits per the **Traceability** principle. |

### Counterpoint convention

When a new framework doc (e.g. `mempool-strategy.md`) addresses existing concerns:

1. Update the relevant section in `vops-compatibility.md` with the new resolution, linking to the framework doc.
2. Update that topic's row in the Status table to reflect the resolution.

This keeps the concerns doc self-contained while routing readers to the framework when they want depth.

---

## Style Rules

- Be pragmatic and direct. No filler, no trailing summaries.
- Don't add features, refactor code, or clean up code that wasn't part of the ask
- No docstrings, comments, or type annotations on code you didn't touch
- Keep commit messages concise (1-2 lines)
- Don't create new files unless explicitly needed - prefer editing existing ones

---

## Common Tasks

### Adding a new document

1. Decide the category (Spec, Topics, or Resources)
2. Create `docs/<slug>.md`
3. Add to `config.ts` nav (inside the matching dropdown, Spec or Topics) and sidebar (inside the matching group)
4. Add to `Footer.vue` under the matching column (Spec or Topics); Resources are sidebar-only
5. Add to `README.md` under the matching category table
6. Update this file's repository structure section

### Adding a new competing standard

1. Create a dedicated per-EIP page at `docs/eip-<N>.md` (or `docs/eip-xxxx.md` for pre-draft/gist proposals without an EIP number). Open it with the standard three-move pattern: **At a Glance** (what it is, problem it solves, why an EIP-8141 reader should care), then Overview, Core Design, Mempool Strategy, Key Differences from EIP-8141, Activity, Strengths, Weaknesses. Close with a pointer back to `competing-standards.md`.
2. Update `config.ts` sidebar under the Alternatives group and add the page to `Footer.vue` under the Alternatives column. Do **not** add Alternatives entries to `config.ts` nav; Alternatives never appears in the top nav (see Document Ordering and Categories).
3. Update the comparative analysis in `competing-standards.md` (spectrum diagram, PQ table, Mempool table, Adoption table) to include the new proposal. Do **not** paste the full per-EIP content into `competing-standards.md`; that doc is the comparative hub, the per-EIP page is the canonical source.
4. Add a link in `appendix.md` under `## Competing Standards` following the link-formatting rule in §2d (canonical source URL, Magicians thread if one exists, author tag for non-EIP proposals).
5. Add the proposal's author to `## Key Contributors` in `appendix.md` if they are not already listed.
6. If the new proposal reframes or resolves concerns on other topic docs (`vops-compatibility.md`, `mempool-strategy.md`, `developer-tooling.md`), follow the [Counterpoint convention](#counterpoint-convention).
7. Update `README.md` Alternatives table if the README mirrors sidebar structure.

### Adding a new VOPS/statelessness topic

1. Add section in `vops-compatibility.md` following existing pattern
2. Add row to the Status table at the top of the doc
3. Add cross-reference in `faq.md` if a relevant question exists (use anchor slug format)
4. Do not attribute to named individuals

### Adding a new mempool open question

1. Add subsection under "Open Questions" in `mempool-strategy.md`
2. Add cross-reference in `faq.md` if relevant

### Adding a new topic doc that addresses existing concerns

When the new doc proposes a framework, resolution, or counterargument that touches existing pending concerns:

1. Write the topic doc following the [Topic-doc structure](#topic-doc-structure) conventions
2. Audit `vops-compatibility.md` and `mempool-strategy.md` for topics the doc resolves or reframes
3. For each affected concern, follow the [Counterpoint convention](#counterpoint-convention): add a `**Counterpoint**:` paragraph linking to the relevant section, and update that concern's row in the Summary table
4. Cross-link from related FAQ entries to the new doc's relevant sections
5. If the new doc reframes content already in `current-spec.md` (e.g. mempool policy), open the relevant section with a short paragraph pointing to the new framework

### Updating PR status

1. Update `merged-changes.md` - move PR between Open/Merged/Closed sections as needed, add review comment summaries
2. Update `appendix.md` PR timeline table
3. If a merged PR changes the spec: update `current-spec.md` and `original-vs-latest.md`
