# Arcium CLI & Deployment

## IMPORTANT: Always Use `arcium` CLI

**Never use `cargo build`, `anchor build`, `anchor deploy`, or `solana program deploy` directly.**

The `arcium` CLI wraps Anchor and adds MPC-specific steps (circuit compilation, MXE init, ARX node management).

## Docker Development Environment

All arcium commands run inside a Docker container with the full Arcium toolchain.

### Sample Dockerfile

```dockerfile
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

# System dependencies + Docker CLI (for arcium test MPC cluster)
RUN rm -rf /var/lib/apt/lists/* \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
        curl build-essential libssl-dev pkg-config libudev-dev \
        git ca-certificates unzip gnupg docker.io docker-compose-v2 \
    && rm -rf /var/lib/apt/lists/*

# Node.js 20.x (required for Arcium)
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y nodejs \
    && npm install -g yarn \
    && rm -rf /var/lib/apt/lists/*

# Rust 1.89.0 (match your rust-toolchain.toml)
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y \
    && . "$HOME/.cargo/env" \
    && rustup default 1.89.0 \
    && rustup component add rustfmt clippy

# Solana CLI 2.3.0 (required by Arcium 0.6.3)
RUN sh -c "$(curl -sSfL https://release.anza.xyz/v2.3.0/install)" \
    && . "$HOME/.bashrc"

ENV PATH="/root/.cargo/bin:/root/.local/share/solana/install/active_release/bin:${PATH}"

# Generate default Solana keypair
RUN solana-keygen new --no-bip39-passphrase --silent || true

# Anchor 0.32.1 via AVM
RUN cargo install --git https://github.com/coral-xyz/anchor avm --force \
    && avm install 0.32.1 \
    && avm use 0.32.1

# Arcium CLI 0.6.3 (direct binary download)
RUN curl "https://bin.arcium.com/download/arcium_x86_64_linux_0.6.3" -o /root/.cargo/bin/arcium \
    && chmod +x /root/.cargo/bin/arcium \
    && curl "https://bin.arcium.com/download/arcup_x86_64_linux_0.6.3" -o /root/.cargo/bin/arcup \
    && chmod +x /root/.cargo/bin/arcup

# Clean cargo cache to avoid version conflicts
RUN rm -rf /root/.cargo/registry/cache /root/.cargo/registry/src /root/.cargo/registry/index

# Verify
RUN rustc --version && cargo --version && solana --version \
    && anchor --version && arcium --version && node --version && yarn --version

WORKDIR /app
SHELL ["/bin/bash", "-c"]
```

### Running the Container

```bash
# Build image
docker build -t your-arcium-dev .

# Start container (mount project + Solana keys + Docker socket for MPC nodes)
docker run -d \
    --name your-dev \
    --ulimit nofile=1048576:1048576 \
    --ulimit nproc=65535:65535 \
    -v "$(pwd)":/app \
    -v "$HOME/.config/solana":/root/.config/solana \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -p 8899:8899 \
    -p 8900:8900 \
    your-arcium-dev \
    sleep infinity

# All commands via docker exec
docker exec your-dev bash -c "cd /app && arcium build"
```

**Key mounts:**
- `$(pwd):/app` — project source
- `$HOME/.config/solana:/root/.config/solana` — wallet keypair
- `/var/run/docker.sock` — required for `arcium test` to spin up ARX MPC nodes

## Commands

### Build
```bash
# Full build (circuits + program + IDL)
arcium build

# Skip key sync (when manually managing declare_id for devnet)
arcium build --skip-keys-sync
```

**What it does:**
1. Compiles Arcis circuits in `encrypted-ixs/` → generates `.arcis`, `.hash`, `.idarc` in `build/`
2. Runs `anchor build` for the Solana program
3. Runs `anchor keys sync` (updates `declare_id!` from keypair file)
4. Generates IDL

### Test (Localnet)
```bash
# Full build + test (recommended for first run or after struct changes)
arcium test

# Skip build (reuse existing binary + IDL)
arcium test --skip-build

# Skip local ARX nodes (manual node management)
arcium test --skip-local-arx-nodes
```

**What it does:**
1. Builds (unless `--skip-build`)
2. Generates genesis accounts in `artifacts/`
3. Starts `solana-test-validator` with Arcium program + accounts
4. Starts ARX Docker nodes
5. Waits for nodes to come online (~60s)
6. Runs `yarn run ts-mocha` tests
7. Cleans up nodes + validator

### Test (Devnet)
```bash
arcium test --cluster devnet --skip-build
```

### Deploy (Devnet)
```bash
arcium deploy \
  --cluster-offset 456 \
  --recovery-set-size 4 \
  --keypair-path /root/.config/solana/id.json \
  --rpc-url devnet \
  --program-keypair /app/target/deploy/your_program-keypair.json \
  --program-name your_program
```

**What it does:**
1. Deploys program to Solana
2. Initializes MXE for the program
3. Sets up cluster association

## Offchain Circuit Storage

Circuits are uploaded to a separate repository (e.g., GitHub) rather than on-chain.

### Upload Workflow
```bash
# 1. Build generates .arcis files
arcium build

# 2. Copy to circuits repo
cp build/*.arcis /path/to/circuits-repo/

# 3. Push to GitHub
cd /path/to/circuits-repo && git add . && git commit -m "Update circuits" && git push
```

### Program References
```rust
init_comp_def(
    ctx.accounts,
    Some(CircuitSource::OffChain(OffChainCircuitSource {
        source: "https://raw.githubusercontent.com/user/circuits-repo/main/my_circuit.arcis".to_string(),
        hash: circuit_hash!("my_circuit"),  // Reads from build/*.hash
    })),
    None,
)?;
```

### ARX Node Behavior
- Nodes fetch `.arcis` from URL at computation time
- Verify circuit hash matches on-chain `circuit_hash!()`
- GitHub raw URLs work from both localnet Docker and devnet

## Key Sync Gotcha

`arcium build` and `arcium test` run `anchor keys sync`, which overwrites `declare_id!()` in `lib.rs` and `[programs.*]` in `Anchor.toml` from the keypair file.

**For devnet:** Ensure `target/deploy/your_program-keypair.json` matches your deployed program ID. If not, use `--skip-build` or `--skip-keys-sync`.

## Fresh Devnet Deploy Procedure

```bash
# 1. Generate new keypair
solana-keygen new -o target/deploy/your_program-keypair.json --force --no-bip39-passphrase
# Note the pubkey

# 2. Update declare_id! in lib.rs
# 3. Update [programs.devnet] in Anchor.toml
# 4. Set cluster = "devnet" in Anchor.toml

# 5. Build + Deploy + Init MXE
arcium build
arcium deploy \
  --cluster-offset 456 \
  --recovery-set-size 4 \
  --keypair-path /root/.config/solana/id.json \
  --rpc-url devnet \
  --program-keypair target/deploy/your_program-keypair.json \
  --program-name your_program

# 6. Test
arcium test --cluster devnet --skip-build
```

## Clean Localnet Test Procedure

```bash
# Ensure no zombie processes
docker restart your-dev

# Clean stale ledger
docker exec your-dev bash -c "rm -rf /app/.anchor/test-ledger"

# Full build + test
docker exec your-dev bash -c "cd /app && arcium test"
```

**Never use `--skip-build` after changing account structs** — the IDL must be regenerated. Stale IDL causes `Unknown action 'undefined'` errors.

## Diagnostic Commands

```bash
# Check mempool for pending computations
arcium mempool 456 --rpc-url devnet

# Check executing pool
arcium execpool 456 --rpc-url devnet

# Check computation details
arcium computation 456 <computation_offset>

# Check MXE info
arcium mxe-info --program-id <program_id> --rpc-url devnet
```

## Circuit Rename Breaking Change

Renaming a circuit function in `encrypted-ixs/src/lib.rs` requires:
1. `arcium build` to regenerate `.arcis`/`.hash` files
2. Upload new `.arcis` to GitHub circuits repo
3. Re-initialize comp def on-chain (new PDA, new offset)
4. Update all program references to new circuit name

## Callback Server vs. Circuit Storage

These are **different things**:

| Concept | Purpose | When Used |
|---------|---------|-----------|
| **Offchain Circuit Storage** | Host compiled `.arcis` files | Always (CircuitSource::OffChain) |
| **Callback Server** | Receive large MPC outputs | Only when outputs exceed ~1KB Solana tx limit |

Don't confuse them. Most projects only need offchain circuit storage.
