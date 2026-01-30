# ArgBuilder Patterns (Battle-Tested)

The ArgBuilder constructs the encrypted argument payload for MPC computations. The order of arguments **must match** the circuit's `.idarc` descriptor exactly.

## Pattern Reference

| Arcis Type | ArgBuilder Pattern | Notes |
|------------|-------------------|-------|
| `Enc<Mxe, T>` | `.plaintext_u128(nonce)` + `.account(key, offset, size)` | Nonce from account, data read from chain |
| `Enc<Shared, T>` | `.x25519_pubkey(pubkey)` + `.plaintext_u128(nonce)` + `.encrypted_*()` | Client-encrypted input |
| `Shared` (standalone) | `.x25519_pubkey(pubkey)` + `.plaintext_u128(shared_nonce)` | **Both required!** |
| `Mxe` (standalone) | (no args needed) | Used in init circuits |
| `u64` (plaintext) | `.plaintext_u64(value)` | Public parameter |
| `u128` (plaintext) | `.plaintext_u128(value)` | Public parameter |

## CRITICAL: Shared Type Requires Nonce

The `Shared` type as a standalone parameter requires **BOTH** `.x25519_pubkey()` **AND** `.plaintext_u128(shared_nonce)`. Missing the nonce causes:
```
Error: ArgBuilder account size mismatch with circuit
```

This was discovered via the `.idarc` circuit descriptor. Always check `.idarc` files when debugging ArgBuilder mismatches.

## Real-World Examples

### 1. Init (Mxe param only)
```rust
// Circuit: pub fn init_vault(mxe: Mxe) -> Enc<Mxe, VaultState>
// Mxe requires no args in ArgBuilder
let args = ArgBuilder::new().build();
```

### 2. Client Input + MXE State (Deposit)
```rust
// Circuit: pub fn deposit(
//     deposit_input: Enc<Shared, DepositInput>,  // Client input
//     vault_state: Enc<Mxe, VaultState>,         // On-chain MXE state
//     user_position: Enc<Mxe, UserPosition>,     // On-chain MXE state
// ) -> (Enc<Mxe, VaultState>, Enc<Mxe, UserPosition>)

let args = ArgBuilder::new()
    // Enc<Shared, DepositInput>: pubkey + nonce + ciphertext
    .x25519_pubkey(encryption_pubkey)
    .plaintext_u128(amount_nonce)
    .encrypted_u64(encrypted_amount)
    // Enc<Mxe, VaultState>: nonce + account read
    .plaintext_u128(ctx.accounts.vault.nonce)
    .account(
        ctx.accounts.vault.key(),
        8 + 1 + 32 + 32,  // skip: discriminator + bump + authority + token_mint
        32 * 3,            // read: 3 encrypted u64s (pending, liquidity, deposited)
    )
    // Enc<Mxe, UserPosition>: nonce + account read
    .plaintext_u128(ctx.accounts.user_position.nonce)
    .account(
        ctx.accounts.user_position.key(),
        8 + 1 + 32 + 32,  // skip: discriminator + bump + owner + vault
        32 * 2,            // read: 2 encrypted u64s (deposited, lp_share)
    )
    .build();
```

### 3. MXE State + Plaintext (Record Liquidity)
```rust
// Circuit: pub fn record_liquidity(
//     liquidity_delta: u64,               // Plaintext
//     vault_state: Enc<Mxe, VaultState>,  // On-chain MXE state
// ) -> Enc<Mxe, VaultState>

let args = ArgBuilder::new()
    // u64 plaintext
    .plaintext_u64(liquidity_delta)
    // Enc<Mxe, VaultState>: nonce + account
    .plaintext_u128(ctx.accounts.vault.nonce)
    .account(
        ctx.accounts.vault.key(),
        8 + 1 + 32 + 32,
        32 * 3,
    )
    .build();
```

### 4. MXE State + Shared Output (Compute Withdrawal)
```rust
// Circuit: pub fn compute_withdrawal(
//     user_position: Enc<Mxe, UserPosition>,
//     user_pubkey: Shared,  // <-- standalone Shared
// ) -> Enc<Shared, u64>

let args = ArgBuilder::new()
    // Enc<Mxe, UserPosition>: nonce + account
    .plaintext_u128(ctx.accounts.user_position.nonce)
    .account(
        ctx.accounts.user_position.key(),
        8 + 1 + 32 + 32,
        32 * 2,
    )
    // Shared: pubkey + nonce (BOTH required!)
    .x25519_pubkey(encryption_pubkey)
    .plaintext_u128(shared_nonce)
    .build();
```

### 5. Multiple MXE States + Plaintext (Clear Position)
```rust
// Circuit: pub fn clear_position(
//     user_position: Enc<Mxe, UserPosition>,
//     withdraw_amount: u64,
//     vault_state: Enc<Mxe, VaultState>,
// ) -> (Enc<Mxe, UserPosition>, Enc<Mxe, VaultState>)

let args = ArgBuilder::new()
    // Enc<Mxe, UserPosition>
    .plaintext_u128(ctx.accounts.user_position.nonce)
    .account(ctx.accounts.user_position.key(), 8 + 1 + 32 + 32, 32 * 2)
    // u64 plaintext
    .plaintext_u64(withdraw_amount)
    // Enc<Mxe, VaultState>
    .plaintext_u128(ctx.accounts.vault.nonce)
    .account(ctx.accounts.vault.key(), 8 + 1 + 32 + 32, 32 * 3)
    .build();
```

## Account Byte Offset Calculation

The `.account(pubkey, offset, size)` reads raw bytes from on-chain account data.

```
Byte layout of a typical account:
  [0..8]     Anchor discriminator (8 bytes)
  [8]        bump (1 byte)
  [9..41]    authority/owner Pubkey (32 bytes)
  [41..73]   other Pubkey field (32 bytes)
  [73..169]  encrypted state (N * 32 bytes)
  [169..185] nonce (16 bytes, u128)
```

**Formula:** `offset = 8 (discriminator) + sum of all fields before encrypted state`

**Size:** `num_encrypted_fields * 32`

## Debugging ArgBuilder Mismatches

When you get `ArgBuilder account size mismatch with circuit`:

1. Check the `.idarc` file in `build/` directory — it describes exact input layout
2. Verify every circuit parameter has matching ArgBuilder calls
3. Verify order matches exactly (Arcis circuit param order = ArgBuilder order)
4. Remember `Shared` standalone needs **both** pubkey + nonce
5. Remember `Enc<Mxe, T>` needs **both** nonce + account read
