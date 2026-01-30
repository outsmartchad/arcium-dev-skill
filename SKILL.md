---
name: arcium-dev
description: End-to-end Arcium MPC development playbook (Jan 2026). Build privacy-preserving Solana programs using Arcium's Cerberus MPC protocol. Covers encrypted instructions (Arcis), three-instruction callback pattern, encryption/sealing, MXE configuration, offchain circuit storage, ArgBuilder patterns, and integration with Anchor programs. Battle-tested on the Zodiac Liquidity project (16/16 tests passing on devnet + localnet).
user-invocable: true
---

# Arcium Development Skill (MPC-first privacy)

## What this Skill is for
Use this Skill when the user asks for:
- Privacy-preserving Solana program development
- Encrypted computation / MPC circuits (Arcis)
- Confidential DeFi (private LP, sealed bids, hidden state)
- Arcium + Anchor program integration
- Encrypted state management on-chain
- Re-encryption / sealing for selective disclosure
- MXE (MPC eXecution Environment) setup
- Arcium CLI commands (build, deploy, test)

## Default stack decisions (opinionated)

1) **MPC Backend: Cerberus only for DeFi**
- Use Cerberus protocol (dishonest majority, N-1 malicious tolerated)
- Never use Manticore for financial applications (honest-but-curious only)
- Cerberus provides MAC-based authentication and abort-on-cheat

2) **Program Framework: Anchor + Arcium macros**
- Use `#[arcium_program]` instead of `#[program]`
- Use `arcium-anchor` crate for account macros
- Use `arcium-client` and `arcium-macros` for MPC integration

3) **Encrypted Instructions: Arcis framework**
- Write MPC circuits in `encrypted-ixs/` directory
- Use `#[encrypted]` module and `#[instruction]` function attributes
- Compile to MPC circuits via `arcium build`

4) **Three-Instruction Pattern (mandatory)**
Every MPC operation requires:
- `init_*_comp_def` — One-time computation definition setup (with offchain circuit URL + hash)
- `your_instruction` — Queue computation with encrypted args via ArgBuilder
- `your_instruction_callback` — Receive and verify MPC results

5) **Encryption: x25519 + Rescue cipher**
- Use `@arcium-hq/client` for TypeScript encryption
- Use `Enc<Shared, T>` for client-decryptable data
- Use `Enc<Mxe, T>` for protocol-only state
- Use `Mxe` as standalone param to get MXE owner for initializing state

6) **Circuit Storage: Offchain (GitHub)**
- Upload compiled `.arcis` files to a GitHub repo
- Use `CircuitSource::OffChain` with raw GitHub URLs
- `circuit_hash!()` macro reads from `build/*.hash` for verification
- ARX nodes fetch and verify circuits at computation time
- **NOT** the same as callback server (that's for large MPC outputs)

7) **Testing & Deployment: Arcium CLI**
- `arcium build` — Compile circuits + Anchor program
- `arcium test` — Full build + localnet validator + ARX nodes + tests
- `arcium deploy` — Program deploy + MXE initialization
- Version 0.6.3 with cluster offset 456
- All commands run inside Docker container

## Operating procedure (how to execute tasks)

### 1. Classify the task layer
- Arcis circuit layer (encrypted-ixs/)
- Anchor program layer (programs/)
- TypeScript client layer (tests/, client/)
- Deployment/MXE configuration

### 2. Pick the right patterns

| Use Case | Pattern | Example |
|----------|---------|---------|
| Single operation | Stateless | Coinflip (ArcisRNG) |
| Accumulating state | Enc<Mxe, T> storage | Voting counters, Vault deposits |
| Data sharing | Re-encryption/sealing | Medical records |
| Private comparison | Encrypted state update | Sealed auction |
| Large arrays | Pack<T> compression | Blackjack deck |
| Key management | MXESigningKey | Ed25519 distributed |
| Init encrypted state | Mxe param | init_vault(mxe: Mxe) -> Enc<Mxe, T> |
| User-readable state | Shared re-encrypt | get_position(state, user_pubkey: Shared) -> Enc<Shared, T> |

### 3. Implement with Arcium-specific correctness

Always be explicit about:
- Encryption ownership (`Shared` vs `Mxe`)
- Computation offset (random u64 per computation, **new offset per retry**)
- Callback account setup (pre-created, correct size, `CallbackAccount` array)
- Nonce management (unique per encryption, stored and updated in account)
- Byte offsets for encrypted account data reads via `.account()`
- **Shared type always needs both pubkey AND nonce** in ArgBuilder

### 4. Follow MPC constraints

**Arcis limitations:**
- No `Vec`, `String`, `HashMap` — use fixed arrays
- No `while`, `loop`, `break`, `match` — use bounded `for`
- No recursion, no early `return`
- Both `if/else` branches always execute (cost = sum)
- Dynamic indexing is O(n), not O(1)
- `.reveal()` and `.from_arcis()` cannot be inside conditionals
- Division is extremely expensive (~3B+ ACUs) — compute ratios off-chain

### 5. Add tests
- Write TypeScript tests using `@arcium-hq/client`
- Use `awaitComputationFinalization()` with retry wrapper
- Encrypt inputs with `RescueCipher` and x25519 shared secret
- Handle MXE keygen wait (40+ retries on first localnet run)
- Generate **new computation offset per retry attempt**
- Verify decrypted results match expected values

### 6. Deliverables expectations
When implementing Arcium features, provide:
- `encrypted-ixs/src/lib.rs` — Arcis circuit code
- `programs/*/src/lib.rs` — Anchor program with Arcium macros
- Test file with encryption/decryption flow + retry logic
- Arcium.toml configuration if needed
- Risk notes for privacy invariants

## Progressive disclosure (read when needed)
- Arcis circuits: [arcis-circuits.md](arcis-circuits.md)
- Program integration: [program-integration.md](program-integration.md)
- ArgBuilder patterns: [argbuilder-patterns.md](argbuilder-patterns.md)
- Encryption patterns: [encryption-patterns.md](encryption-patterns.md)
- Account management: [accounts.md](accounts.md)
- TypeScript client: [client.md](client.md)
- CLI & deployment: [cli-deployment.md](cli-deployment.md)
- Security checklist: [security.md](security.md)
- Examples reference: [examples.md](examples.md)
- Troubleshooting: [troubleshooting.md](troubleshooting.md)
