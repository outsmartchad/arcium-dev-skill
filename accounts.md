# Arcium Account Management

## Account Types Overview

### Standard Arcium Accounts

| Account | Purpose | Derivation |
|---------|---------|------------|
| MXE Account | Program's MXE metadata | `derive_mxe_pda!()` |
| Computation Definition | Circuit metadata + hash | `derive_comp_def_pda!(offset)` |
| Computation | Per-computation state | `derive_comp_pda!(offset, mxe)` |
| Cluster | MPC cluster info | `derive_cluster_pda!(mxe)` |
| Mempool | Pending computations | `derive_mempool_pda!(mxe)` |
| Executing Pool | Active computations | `derive_execpool_pda!(mxe)` |
| Sign PDA | Signing authority | `derive_sign_pda!()` |

### Global Accounts (Constants)

```rust
ARCIUM_FEE_POOL_ACCOUNT_ADDRESS  // Fee collection
ARCIUM_CLOCK_ACCOUNT_ADDRESS     // Epoch timing
```

## Computation Definition Offset

```rust
// Auto-derived from instruction name
const COMP_DEF_OFFSET: u32 = comp_def_offset("instruction_name");

// Derivation: sha256(instruction_name)[0..4] as u32 (little-endian)
```

## Storing Encrypted State

### Account Structure
```rust
#[account]
#[derive(InitSpace)]
pub struct VaultAccount {
    pub bump: u8,
    pub authority: Pubkey,
    pub token_mint: Pubkey,

    // Encrypted fields (raw ciphertext bytes, 32 bytes each)
    pub vault_state: [[u8; 32]; 3],  // 3 Enc<Mxe, u64>s

    // Nonce for MXE re-encryption (updated by every callback)
    pub nonce: u128,

    // Additional plain fields
    pub meta_nonce: u64,  // e.g., replay protection for meta-txs
}
```

### Space Calculation
```rust
// Use #[derive(InitSpace)] and let Anchor calculate, or manually:
const SPACE: usize = 8           // Discriminator
    + 1                          // bump
    + 32                         // authority
    + 32                         // token_mint
    + 32 * 3                     // vault_state (3 ciphertexts)
    + 16                         // nonce (u128)
    + 8;                         // meta_nonce (u64)
```

## Reading Encrypted Account Data in MPC

Use `.account()` in ArgBuilder to read encrypted state from on-chain accounts:

```rust
// In queue_computation call
let args = ArgBuilder::new()
    // First: nonce for the Enc<Mxe, T>
    .plaintext_u128(ctx.accounts.vault.nonce)
    // Then: account read
    .account(
        ctx.accounts.vault.key(),
        8 + 1 + 32 + 32,  // Offset: skip discriminator + bump + authority + token_mint
        32 * 3,            // Size: 3 ciphertexts × 32 bytes
    )
    .build();
```

### Memory Layout Example
```
Byte 0-7:     Anchor discriminator
Byte 8:       bump
Byte 9-40:    authority (Pubkey)
Byte 41-72:   token_mint (Pubkey)
Byte 73-104:  vault_state[0] (pending_deposits ciphertext)
Byte 105-136: vault_state[1] (total_liquidity ciphertext)
Byte 137-168: vault_state[2] (total_deposited ciphertext)
Byte 169-184: nonce (u128)
Byte 185-192: meta_nonce (u64)
```

**Offset for `.account()`: 73** (8 + 1 + 32 + 32)
**Size for `.account()`: 96** (32 × 3)

## Callback Account Requirements

### Pre-creation
Accounts must exist before callback executes:

```rust
// In queue instruction, use init_if_needed for idempotent creation
#[account(
    init_if_needed,
    payer = authority,
    space = 8 + VaultAccount::INIT_SPACE,
    seeds = [b"vault", token_mint.key().as_ref()],
    bump,
)]
pub vault: Account<'info, VaultAccount>,
```

### No Resizing
Allocate sufficient space upfront — callbacks cannot resize accounts.

### Custom Callback Accounts

```rust
// Pass accounts to callback that need MPC output written to them
vec![MyCallback::callback_ix(
    computation_offset,
    &ctx.accounts.mxe_account,
    &[
        CallbackAccount {
            pubkey: ctx.accounts.vault.key(),
            is_signer: false,
            is_writable: true,
        },
        CallbackAccount {
            pubkey: ctx.accounts.user_position.key(),
            is_signer: false,
            is_writable: true,
        },
    ]
)?]
```

### Callback Struct

```rust
#[callback_accounts("my_instruction")]
#[derive(Accounts)]
pub struct MyCallback<'info> {
    // Standard 6 accounts (auto-included by macro)
    pub arcium_program: Program<'info, Arcium>,
    #[account(address = derive_comp_def_pda!(COMP_DEF_OFFSET))]
    pub comp_def_account: Account<'info, ComputationDefinitionAccount>,
    #[account(address = derive_mxe_pda!())]
    pub mxe_account: Account<'info, MXEAccount>,
    pub computation_account: UncheckedAccount<'info>,
    #[account(address = derive_cluster_pda!(mxe_account, ErrorCode::ClusterNotSet))]
    pub cluster_account: Account<'info, Cluster>,
    #[account(address = ::anchor_lang::solana_program::sysvar::instructions::ID)]
    pub instructions_sysvar: AccountInfo<'info>,

    // Custom accounts (in order passed to callback_ix)
    #[account(mut)]
    pub vault: Account<'info, VaultAccount>,
    #[account(mut)]
    pub user_position: Account<'info, UserPositionAccount>,
}
```

## Updating Encrypted State in Callback

### Single Return Value
```rust
#[arcium_callback(encrypted_ix = "reveal_pending_deposits")]
pub fn reveal_callback(
    ctx: Context<RevealCallback>,
    output: SignedComputationOutputs<RevealOutput>,
) -> Result<()> {
    let o = match output.verify_output(
        &ctx.accounts.cluster_account,
        &ctx.accounts.computation_account
    ) {
        Ok(RevealOutput { field_0 }) => field_0,
        Err(_) => return Err(ErrorCode::AbortedComputation.into()),
    };

    // For plaintext reveal: access raw bytes
    emit!(RevealEvent { amount: o });

    Ok(())
}
```

### Tuple Return Value (Multiple Encrypted Outputs)
```rust
#[arcium_callback(encrypted_ix = "deposit")]
pub fn deposit_callback(
    ctx: Context<DepositCallback>,
    output: SignedComputationOutputs<DepositOutput>,
) -> Result<()> {
    let tuple = match output.verify_output(
        &ctx.accounts.cluster_account,
        &ctx.accounts.computation_account
    ) {
        Ok(DepositOutput { field_0 }) => field_0,
        Err(_) => return Err(ErrorCode::AbortedComputation.into()),
    };

    // First return value: vault state (3 ciphertexts)
    ctx.accounts.vault.vault_state[0] = tuple.field_0.ciphertexts[0];
    ctx.accounts.vault.vault_state[1] = tuple.field_0.ciphertexts[1];
    ctx.accounts.vault.vault_state[2] = tuple.field_0.ciphertexts[2];
    ctx.accounts.vault.nonce = tuple.field_0.nonce;

    // Second return value: user position (2 ciphertexts)
    ctx.accounts.user_position.position_state[0] = tuple.field_1.ciphertexts[0];
    ctx.accounts.user_position.position_state[1] = tuple.field_1.ciphertexts[1];
    ctx.accounts.user_position.nonce = tuple.field_1.nonce;

    Ok(())
}
```

## Account Validation

Always validate account relationships:

```rust
#[account(
    mut,
    seeds = [b"vault", vault.token_mint.as_ref()],
    bump = vault.bump,
    has_one = authority,
)]
pub vault: Account<'info, VaultAccount>,

#[account(mut)]
pub authority: Signer<'info>,
```
