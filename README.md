# Arcium Dev Skill

A Claude Code skill for building privacy-preserving Solana programs with [Arcium](https://arcium.com)'s Cerberus MPC protocol.

Battle-tested on the [Zodiac Liquidity](https://github.com/outsmartchad/zodiac-liquidity) project (16/16 tests passing on devnet + localnet).

## What's Included

| File | Covers |
|------|--------|
| `SKILL.md` | Main skill entry point and overview |
| `program-integration.md` | Anchor program structure, three-instruction pattern, callbacks |
| `arcis-circuits.md` | Arcis circuit syntax (`Enc<Shared,T>`, `Enc<Mxe,T>`, reveals) |
| `argbuilder-patterns.md` | All 5 ArgBuilder patterns for passing data to MPC |
| `accounts.md` | Account structs, byte offsets, encrypted state storage |
| `client.md` | TypeScript client — encryption, retry wrapper, MXE key fetch |
| `encryption-patterns.md` | Encryption types, nonce management, sealing |
| `cli-deployment.md` | `arcium` CLI build/test/deploy, Docker workflow |
| `troubleshooting.md` | Common errors and fixes from real development |
| `security.md` | Security checklist for MPC programs |
| `examples.md` | Pattern catalog (voting, auction, blackjack, etc.) |

## Usage

### As a Claude Code Skill

Copy the files into your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/arcium-dev
cp *.md ~/.claude/skills/arcium-dev/
```

Then invoke with `/arcium-dev` in Claude Code.

### As Reference

Browse the markdown files directly — they're self-contained documentation covering the full Arcium development workflow from circuit design to deployment.

## Tech Stack

- **Anchor** 0.32.1 — Solana program framework
- **Arcium** 0.6.3 — MPC privacy layer (Cerberus protocol)
- **@arcium-hq/client** 0.6.3 — TypeScript encryption + account helpers

## License

MIT
