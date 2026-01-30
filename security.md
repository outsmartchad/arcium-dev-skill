# Arcium Security Checklist

## Core Principle

Arcium provides **computational privacy** through MPC. Security depends on:
1. At least 1 honest ARX node (Cerberus protocol)
2. Correct encryption/nonce usage
3. Proper account validation
4. Sound circuit design

---

## MPC Security Model

### Cerberus (Use for DeFi)
- Dishonest majority: tolerates N-1 malicious nodes
- Requires only 1 honest node
- MAC-based authentication
- Abort-on-cheat detection

### Manticore (NOT for DeFi)
- Honest-but-curious only
- Requires trusted dealer
- Use only for trusted operator scenarios

---

## Encryption Security

### Nonce Reuse (CRITICAL)
**Risk:** Reusing nonce with same key completely breaks encryption.

**Prevention:**
```rust
// Always store and update nonce
ctx.accounts.state.nonce = output.nonce;
```

```typescript
// Always generate fresh nonce
const nonce = randomBytes(16);
```

### Key Management
**Risk:** Shared secret exposure allows decryption of `Enc<Shared, T>`.

**Prevention:**
- Use ephemeral keys when possible
- Use `Enc<Mxe, T>` for protocol state users shouldn't decrypt
- Implement key rotation for long-lived applications

---

## Circuit Security

### Information Leakage via Branches
**Risk:** Both branches always execute, but selection can leak through output.

**Bad Pattern:**
```rust
// Leaks whether amount > threshold
if amount > threshold {
    transfer(amount)
} else {
    reject()
}
```

**Better Pattern:**
```rust
// Always perform same operations
let should_transfer = amount > threshold;
let transfer_amount = if should_transfer { amount } else { 0 };
// Always call transfer, but with 0 if rejected
```

### Timing Attacks
**Risk:** Variable execution time leaks information.

**Protection:** Arcis uses constant-time operations. Don't circumvent with:
- Early returns based on secrets
- Variable loop iterations

### Dynamic Indexing
**Risk:** O(n) complexity reveals array size, not index.

**Note:** This is safe - all positions checked regardless of actual index.

---

## Account Security

### Missing Owner Checks
**Risk:** Fake accounts with correct structure pass deserialization.

**Prevention:**
```rust
// Use typed accounts
pub state: Account<'info, GameState>,

// Or explicit check
#[account(owner = program_id)]
pub state: UncheckedAccount<'info>,
```

### Callback Account Manipulation
**Risk:** Attacker passes wrong accounts to callback.

**Prevention:**
```rust
// Validate PDA seeds
#[account(
    seeds = [b"game", game_id.to_le_bytes().as_ref()],
    bump = state.bump,
)]
pub state: Account<'info, GameState>,
```

### State Relationship Validation
**Risk:** Valid accounts but wrong relationships.

**Prevention:**
```rust
#[account(
    has_one = authority,
    constraint = state.game_id == expected_game_id,
)]
pub state: Account<'info, GameState>,
```

---

## Privacy Invariants

### What Arcium Protects
- Input values during computation
- Intermediate computation values
- Output values (if encrypted)

### What Arcium Does NOT Protect
- Transaction metadata (who called, when)
- Account existence and addresses
- Public parameters
- Revealed values

### Metadata Leakage
**Risk:** Timing, frequency, and parties reveal information.

**Mitigation:**
- Batch operations
- Use relayers/proxies
- Randomize timing where possible

---

## Common Vulnerabilities

### 1. Revealing Too Much
```rust
// BAD: Reveals exact balance
let balance = encrypted_balance.to_arcis();
balance.reveal()

// BETTER: Reveal only what's needed
let sufficient = balance >= required_amount;
sufficient.reveal()
```

### 2. Front-Running via Metadata
**Risk:** Seeing encrypted bid transaction allows front-running.

**Mitigation:**
- Commit-reveal schemes
- Randomized submission windows
- MEV protection

### 3. Replay Attacks
**Risk:** Reusing computation with same inputs.

**Prevention:**
- Include unique nonce/timestamp in inputs
- Track computation offsets

### 4. State Manipulation Between Computations
**Risk:** State changed between queue and callback.

**Prevention:**
- Validate state version/nonce in callback
- Use atomic state updates

---

## Checklist

### Circuit Design
- [ ] No unnecessary reveals
- [ ] Constant-time operations (no early exits on secrets)
- [ ] Bounded loops only
- [ ] No variable-length data structures

### Encryption
- [ ] Fresh nonce for every encryption
- [ ] Correct owner type (Shared vs Mxe)
- [ ] Key rotation strategy for long-lived apps

### Accounts
- [ ] Owner validation on all accounts
- [ ] PDA seed validation
- [ ] Relationship validation (has_one, constraints)
- [ ] Pre-created callback accounts with correct size

### Program
- [ ] Use Cerberus backend for DeFi
- [ ] Validate program IDs for CPIs
- [ ] Handle computation abort gracefully
- [ ] Update nonce after each computation

### Client
- [ ] Generate fresh nonces
- [ ] Secure key storage
- [ ] Handle MXE public key fetch failures
- [ ] Verify decrypted results

---

## Security Review Questions

1. What can an observer learn from transaction metadata alone?
2. What values are revealed vs. kept encrypted?
3. Can computation results be replayed?
4. Are all account relationships validated?
5. Is the nonce management correct?
6. Could timing or frequency patterns leak information?
7. What happens if MPC computation aborts?
