# EIP-XXXX: Tempo-like Transactions

---

**Author**: Georgios Konstantopoulos (@gakonst, Paradigm/Reth)
**Status**: Pre-draft (gist) | **Category**: Core (Standards Track)
**Requires**: EIP-1559, EIP-2718, EIP-2930, EIP-7702, EIP-2, EIP-2929

## At a Glance

**What it is.** A pre-draft gist for a constrained-scope transaction type (`0x76`) that bundles a fixed set of wallet UX primitives into one transaction format: atomic batching, validity windows, gas sponsorship, parallelizable nonces, passkey signatures (P-256 and WebAuthn), and protocol-enforced access keys.

**Problem it solves.** Standardizes the specific features most wallets need today at the protocol level, where they can be statically reasoned about and mempool-validated without the design space of general account abstraction. No new opcodes, no arbitrary validation logic, no replacement of ERC-4337.

**Why an EIP-8141 reader should care.** This is the "what if you don't need frames?" extreme of the spectrum. If the position is "most users want five specific features and nothing else," Tempo-like is what that design looks like when fully fleshed out. It is the farthest-constrained counterpoint to EIP-8141's expressiveness.

## Overview

This proposal introduces a new EIP-2718 transaction type (`0x76`) that bundles a **constrained set of wallet UX primitives** into a single transaction format: atomic batching, validity windows, gas sponsorship, parallelizable nonces, passkey signatures (P-256 and WebAuthn), and protocol-enforced access keys.

The philosophy is explicit: **constrained scope over general framework**. The spec deliberately does not attempt to replace ERC-4337, provide arbitrary validation logic, or introduce new opcodes. Instead, it standardizes the specific features that most wallets need today (batching, sponsorship, passkeys, scheduled execution) at the protocol level where they can be statically reasoned about.

## Core Design

**Atomic Call Batching**: The `calls` field is a list of `[to, value, input]` tuples executed sequentially. If any call reverts, the entire transaction reverts. Unlike EIP-8141's frame architecture, there's no concept of modes or frame-level gas isolation; it's a flat list of calls, all-or-nothing.

**Validity Windows**: `valid_after` and `valid_before` fields provide time-bounded execution. A transaction is valid only when `block.timestamp` falls within the window. This enables scheduled transactions and automatic expiry without off-chain infrastructure.

**Gas Sponsorship**: An optional `fee_payer_signature` field. If present, the recovered fee payer address pays all gas. The fee payer signs a domain-separated hash (`0x78` prefix) covering the full transaction including the sender's address. Fee payer signatures must be secp256k1. The spec explicitly does not define ERC-20 fee payment; sponsors wanting token reimbursement must only co-sign transactions that include explicit compensating calls.

**2D Nonces**: `nonce_key` selects a nonce stream, `nonce` is the sequence within it. `nonce_key == 0` uses the standard protocol nonce. `nonce_key > 0` uses independent parallel streams. This enables concurrent pending transactions without blocking.

**Multiple Signature Schemes**: The `sender_signature` field supports four encodings, detected by length/prefix:

| Scheme | Detection | Notes |
|---|---|---|
| secp256k1 | 65 bytes exactly | Standard ECDSA, 0 extra gas |
| P-256 | First byte `0x01`, 130 bytes | Optional SHA-256 pre-hash, +5,000 gas |
| WebAuthn | First byte `0x02`, variable (max 2,049 bytes) | Full passkey flow with authenticatorData + clientDataJSON, +5,000 gas |
| Keychain wrapper | First byte `0x03` | Wraps inner signature with `user_address` for access key delegation |

**Address Derivation**: For P-256/WebAuthn, the signer address is `keccak256(pub_key_x || pub_key_y)[12:]`, which produces new addresses, not existing EOAs.

**EIP-7702 Interop**: An optional `authorization_list` field processed with standard EIP-7702 semantics. Authorizations are not reverted if call execution reverts (matching 7702 behavior).

**Access Keys**: Deferred to a companion EIP. The Keychain wrapper signature scheme provides the hook: an access key signs on behalf of a root account, validated against protocol-enforced rules (expiry, spending limits).

**Transaction Envelope**:
```
0x76 || rlp([
    chain_id,
    max_priority_fee_per_gas, max_fee_per_gas, gas_limit,
    calls, access_list,
    nonce_key, nonce,
    valid_before, valid_after,
    fee_payer_signature,
    authorization_list,
    sender_signature
])
```

**Per-Call Receipts**: Each call gets its own receipt entry with `success`, `gas_used`, and `logs`, enabling applications to identify which call in a batch failed.

## Mempool Strategy

Validation is purely cryptographic, with no EVM execution during signature verification. The mempool requirements are well-specified:

- Reject malformed signatures, expired transactions, and insufficient balances
- Defer future-valid transactions (`valid_after` in the future)
- Maintain per-root readiness across nonce streams independently
- RBF rules apply per `(root, nonce_key, nonce)` tuple
- Re-check fee payer balance on new head
- Apply anti-DoS policy for transactions with `authorization_list` (cross-account invalidation risk)

The bounded signature sizes (`MAX_WEBAUTHN_SIG_SIZE = 2,049 bytes`) and deterministic verification costs mean nodes can reason about validation cost statically.

## Key Differences from EIP-8141

| Aspect | EIP-XXXX (Tempo-like) | EIP-8141 |
|---|---|---|
| **Philosophy** | Constrained primitives for common UX needs | General-purpose programmable framework |
| **New opcodes** | None | 6 (`APPROVE`, `TXPARAM`, `FRAMEDATALOAD`, `FRAMEDATACOPY`, `FRAMEPARAM`, `SIGPARAM`) |
| **Tx type** | `0x76` | `0x06` |
| **Composition model** | Flat call list, all-or-nothing | Recursive frames with modes and per-frame gas |
| **Signature schemes** | Fixed set: secp256k1, P-256, WebAuthn | Arbitrary via account code + `APPROVE` |
| **Validation model** | Cryptographic only — fixed scheme detection | Programmable EVM in VERIFY frames |
| **Atomic batching** | Native: `calls` list, entire tx reverts on failure | Flags field bit 2 on consecutive SENDER frames |
| **Gas sponsorship** | `fee_payer_signature` field (secp256k1 only) | VERIFY frame for sponsor, canonical paymaster |
| **Validity windows** | Native: `valid_after` / `valid_before` | Not addressed |
| **2D nonces** | Native: `nonce_key` + `nonce` | Not addressed (single nonce) |
| **Passkeys/WebAuthn** | Native at transaction layer | Account code must implement |
| **Access keys** | Keychain wrapper + companion EIP | Not addressed |
| **EIP-7702 interop** | `authorization_list` field | No authorization list (PQ incompatible) |
| **Per-call receipts** | Native: each call gets `success`, `gas_used`, `logs` | Single receipt for entire tx |
| **Mempool complexity** | Low: deterministic crypto verification, bounded sizes | High: validation prefix, banned opcodes, gas caps |
| **PQ readiness** | Not addressed; P-256/WebAuthn are not PQ-safe | Native: arbitrary sig schemes in VERIFY frames |
| **Programmable validation** | No — fixed scheme set | Yes — arbitrary EVM logic |
| **Async execution** | Compatible (no EVM in validation) | Incompatible with async models |
| **Account creation** | Not addressed | DEFAULT frame to deployer contract |
| **EOA default behavior** | Requires 7702 for non-secp256k1 EOAs | Protocol-native default code for codeless accounts |

## Activity

- **[Pre-draft gist](https://gist.github.com/gakonst/00117aa2a1cd327f515bc08fb807102e)** by gakonst (Georgios Konstantopoulos, Paradigm/Reth), published as a design exploration outside the standard EIP channels
- No EIP number assigned, no PR to ethereum/EIPs, no EthMagicians thread yet
- Very early stage; referenced in community discussions but not yet submitted for formal review

## Strengths

- **Immediate UX wins in one tx type**: batching, sponsorship, passkeys (P-256/WebAuthn at L1), validity windows, 2D nonces, per-call receipts. No smart-contract wrappers, no relayers.
- **Simple mempool**: purely cryptographic validation, bounded signature sizes, deterministic cost, async-compatible.
- **EIP-7702 compatibility**: existing 7702 delegation works within the new tx format.

## Weaknesses

- **No programmable validation and no PQ strategy**: signature scheme set is fixed. Adding schemes requires a hard fork. P-256 and WebAuthn are both quantum-vulnerable. Complex policies (multisig, recovery, state-dependent rules) cannot be expressed.
- **Fee payer limited to secp256k1**: sponsors cannot use passkeys or other schemes.
- **No per-call gas isolation**: all calls share a single gas limit; one expensive call can starve subsequent calls.
- **P-256/WebAuthn create new addresses**: existing EOAs cannot use these schemes without migrating assets or 7702-delegating.
- **Access keys deferred to a companion EIP**: the Keychain wrapper is defined, but the actual access-key rules (expiry, spending limits, revocation) are not.
- **Pre-draft stage**: no EIP number, no community review.


---

*Continue with [Competing Standards](/competing-standards) for the comparative analysis, or return to the [Home page](/).*
