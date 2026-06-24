# EIP-8141: Frame Transaction

Comprehensive analysis of [EIP-8141](https://eips.ethereum.org/EIPS/eip-8141) (Frame Transaction) - tracking its origin, evolution through community feedback, competing proposals, and open concerns.

## Documents

### Spec

| Document | Description |
|---|---|
| [Current Spec Overview](./docs/current-spec.md) | What EIP-8141 can do today - execution model, mempool model, transaction structure, opcodes, examples |
| [Feedback Evolution](./docs/feedback-evolution.md) | How community feedback shaped the spec across 21 phases |
| [Original Spec](./docs/original-spec.md) | The Jan 29 submission - original design and what it lacked |
| [Merged Changes](./docs/merged-changes.md) | Every PR (merged, rejected, open) in chronological order with rationale |
| [Original vs Latest](./docs/original-vs-latest.md) | Side-by-side comparison of structural changes and philosophical shifts |

### Topics

| Document | Description |
|---|---|
| [EOA Support](./docs/eoa-support.md) | How protocol-level default code replaces EIP-7702 delegation for common-case EOAs |
| [PQ Roadmap](./docs/pq-roadmap.md) | Seven-stage roadmap from EIP-8141 foundation to full post-quantum and private L1 settlement |
| [Mempool Strategy](./docs/mempool-strategy.md) | Two-tier mempool architecture, VOPS extension, merkle escape hatch, why no relayers are needed |
| [VOPS Compatibility](./docs/vops-compatibility.md) | How frame transactions interact with VOPS, state growth, FOCIL, and witnesses |
| [Developer Tooling](./docs/developer-tooling.md) | Bear and bull cases for wallet/app developer adoption, protocol defaults vs ERC fragmentation |
| [Competing Standards](./docs/competing-standards.md) | EIP-8130, EIP-8175, EIP-8202, EIP-8223, EIP-8224, Tempo - design, tradeoffs, comparative analysis |

### Resources

| Document | Description |
|---|---|
| [FAQ](./docs/faq.md) | Common questions with short answers and cross-references |
| [Glossary](./docs/glossary.md) | Comprehensive glossary of jargon used across the site, grouped by category |
| [Appendix](./docs/appendix.md) | PR timeline, key contributors, external resources |

## Updating

To sync the repo with the latest spec changes, PRs, discussions, and competing standards, paste this prompt into a Claude Code session inside the repo:

```
Run the Update Process in CLAUDE.md.
```

The agent will check master spec, new PRs since the last documented one, EthMagicians and ethresear.ch posts, competing standards activity, then update the relevant docs and refresh sync-snapshot dates per the methodology.

## Website

Browse the research at [eip8141.io](https://eip8141.io) or try the [live demo](https://demo.eip-8141.ethrex.xyz/).

## License

This project is licensed under the [MIT License](./LICENSE.md).
