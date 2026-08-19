---
layout: home
hero:
  name: EIP-8141
  text: Frame Transaction
  tagline: Native account abstraction and post-quantum readiness for Ethereum. One transaction type, multiple frames, programmable validation.
  actions:
    - theme: brand
      text: Read the Overview
      link: /current-spec
    - theme: alt
      text: Try the Demo
      link: https://demo.eip-8141.ethrex.xyz/
features:
  - title: Programmable Validation
    details: Accounts define validation with frame code, protocol signatures, and ARBITRARY witness bytes
  - title: Gas Sponsorship
    details: Third parties pay gas through sponsor VERIFY frames. No bundlers or relayers needed.
  - title: Atomic Batching
    details: Group DEFAULT/SENDER operations into atomic batches that succeed or revert together
  - title: Post-Quantum Ready
    details: No ECDSA dependency in the transaction format. Accounts choose their own scheme with aggregation.
  - title: EOA Compatible
    details: Built-in behavior for codeless accounts. EOAs get native AA without delegations or deployments.
  - title: Dual-Model Design
    details: Execution model allows arbitrary validation. Mempool model constrains it to propagatable shapes.
---

## What to read first?

Pick the route that matches why you're here.

- **Just want the overview** → [Current Spec](/current-spec) for the mechanism, then [FAQ](/faq) for common questions.
- **Building a wallet or app** → [Developer Tooling](/developer-tooling) for the adoption calculus, then [EOA Support](/eoa-support) for what existing accounts get without deploying anything.
- **Following the discussion** → [Feedback Evolution](/feedback-evolution) for the phase-by-phase debate, [Merged Changes](/merged-changes) for PR history, [Original vs Latest](/original-vs-latest) for the diff.
- **Comparing alternatives** → [Competing Standards](/competing-standards) for the comparative analysis across all alternative proposals.
- **FOCIL, VOPS, and statelessness** → [Mempool Strategy](/mempool-strategy) for the two-tier design and trilemma, then [VOPS Compatibility](/vops-compatibility) for the definitions.
- **Post-quantum readiness** → [PQ Roadmap](/pq-roadmap) for the seven-stage arc from EIP-8141 foundation to private L1 settlement.

## How does it work?

A frame transaction (`0x06`) consists of multiple **frames**, each with a mode that tells the protocol what the frame does:

| Mode | Name | Purpose |
|---|---|---|
| `DEFAULT` | Deployment | Deploy accounts, run post-operation hooks |
| `VERIFY` | Verification | Authenticate the sender, authorize payment. Read-only; approval-bearing validation frames call `APPROVE`. |
| `SENDER` | Execution | Execute the user's intended operations (calls, transfers, contract interactions) |

No bundler, no EntryPoint contract, no off-chain infrastructure. The protocol handles validation, gas payment, and execution natively through frames. SENDER frames execute with `msg.sender = tx.sender`, so existing contracts see the original account as the caller. Token approvals, NFT ownership, and all on-chain state work as-is.

EOAs benefit directly without EIP-7702. The protocol has built-in fallback behavior for codeless accounts: VERIFY frames use the secp256k1 signature at `tx.signatures[0]` and call `APPROVE` natively. P256/passkeys are supported as canonical low-`s` protocol-validated outer signatures, but account code is needed to authorize with them. Multi-call sequences come from the frame list (one SENDER frame per call) instead of a payload inside a single frame, with native ETH transfers via `frame.value`. No code is ever deployed to the EOA. With the atomic batch flag set on consecutive DEFAULT or SENDER frames, they become all-or-nothing, protecting users from partial execution; VERIFY cannot participate in or terminate a batch.

### Example A: Gasless Approve + Swap

A user approves and swaps tokens in a single atomic transaction. The sponsor pays all gas; the user pays nothing:

| Frame | Mode | Target | What it does |
|---|---|---|---|
| 0 | VERIFY | sender | Verify sender signature, approve execution |
| 1 | VERIFY | sponsor | Verify sponsor, approve payment |
| 2 | SENDER | ERC-20 | `approve(DEX, amount)` — allow DEX to spend tokens |
| 3 | SENDER | DEX | `swap(...)` — execute the swap |
| 4 | DEFAULT | sponsor | Post-op: refund overcharged gas fees to sponsor |

### Example B: Liquidity Rebalance & Pay Gas in USDC

An EOA at address A rebalances a Uniswap v4 liquidity position, paying for gas in USDC instead of ETH:

| Frame | Mode | Target | What it does |
|---|---|---|---|
| 0 | VERIFY | A | Protocol verifies ECDSA signature, approves execution |
| 1 | VERIFY | sponsor | Sponsor authorizes gas payment |
| 2 | SENDER | Position Manager | `decreaseLiquidity(...)` — remove from old range |
| 3 | SENDER | Position Manager | `collect(...)` — collect tokens |
| 4 | SENDER | USDC | `transfer(sponsor, fee)` — pay sponsor in USDC |
| 5 | SENDER | Position Manager | `increaseLiquidity(...)` — add to new range |
| 6 | DEFAULT | sponsor | Post-op: refund overcharged gas fees |

> In the current public-mempool shape, the sponsor checks authorization data and frame structure rather than the user's token balance. That propagates as a non-canonical paymaster, but the sponsor accepts frontrunning risk if the user drains USDC before inclusion. A trustless balance-checking sponsor remains consensus-valid, but it reads external token storage and routes through the expansive tier, a private mempool, or direct-to-builder. See [Mempool Strategy → ERC-20 gas repayment](/mempool-strategy#erc20-paymaster-patterns).

## What are the new opcodes?

EIP-8141 introduces seven new opcodes that give frame transactions their power:

| Opcode | Purpose |
|---|---|
| `APPROVE` | Terminates a VERIFY frame and sets transaction-scoped approval flags. Scope `0x1` approves payment, `0x2` approves execution, `0x3` approves both. |
| `TXPARAM` | Reads transaction parameters (sender, nonce, fees) from inside a frame. Replaces `ORIGIN` for introspection. |
| `FRAMEDATALOAD` | Loads 32 bytes from the current frame's `data` field. How account code reads frame calldata. |
| `FRAMEDATACOPY` | Copies frame data to memory. Bulk version of `FRAMEDATALOAD` for larger payloads. |
| `FRAMEPARAM` | Reads frame-level metadata (mode, flags, resolved target). Enables frame code to introspect its own execution context. |
| `SIGPARAM` | Reads signature-list metadata and `ARBITRARY` witness length. |
| `SIGDATACOPY` | Copies `ARBITRARY` witness bytes for custom verifiers. Its current `0xb5` assignment collides with EIP-8272. |

`APPROVE` is the central innovation: it's how account code tells the protocol "I've verified this transaction, proceed." The other opcodes give frame code access to transaction, frame, and signature context it needs to make that decision.
