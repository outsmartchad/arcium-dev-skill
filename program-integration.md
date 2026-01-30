# Arcium Program Integration (Anchor)

## Project Structure

```
programs/your_program/
├── Cargo.toml
└── src/
    └── lib.rs    # Anchor + Arcium program
```

## Cargo.toml

```toml
[package]
name = "your_program"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "lib"]
name = "your_program"

[features]
default = []
cpi = ["no-entrypoint"]
no-entrypoint = []
idl-build = ["anchor-lang/idl-build", "anchor-spl/idl-build", "arcium-anchor/idl-build"]

[dependencies]
anchor-lang = { version = "0.32.1", features = ["init-if-needed"] }
anchor-spl = "0.32.1"
arcium-client = { default-features = false, version = "=0.6.3" }
arcium-macros = "=0.6.3"
arcium-anchor = "=0.6.3"
```

## Basic Program Structure

```rust
use anchor_lang::prelude::*;
use arcium_anchor::prelude::*;
use arcium_client::idl::arcium::types::{CallbackAccount, CircuitSource, OffChainCircuitSource};
use arcium_macros::circuit_hash;

// Computation definition offsets (auto-derived from instruction name)
const COMP_DEF_OFFSET_MY_INSTRUCTION: u32 = comp_def_offset("my_instruction");

declare_id!("YourProgramId11111111111111111111111111111");

#[arcium_program]  // Use instead of #[program]
pub mod your_program {
    use super::*;

    // 1. Initialize computation definition (one-time, with offchain circuit)
    pub fn init_my_instruction_comp_def(
        ctx: Context<InitMyInstructionCompDef>
    ) -> Result<()> {
        init_comp_def(
            ctx.accounts,
            Some(CircuitSource::OffChain(OffChainCircuitSource {
                source: "https://raw.githubusercontent.com/user/circuits/main/my_instruction.arcis".to_string(),
                hash: circuit_hash!("my_instruction"),
            })),
            None,
        )?;
        Ok(())
    }

    // 2. Queue computation with encrypted args
    pub fn my_instruction(
        ctx: Context<MyInstruction>,
        computation_offset: u64,
        ciphertext: [u8; 32],
        pubkey: [u8; 32],
        nonce: u128,
    ) -> Result<()> {
        let args = ArgBuilder::new()
            .x25519_pubkey(pubkey)
            .plaintext_u128(nonce)
            .encrypted_u64(ciphertext)
            .build();

        queue_computation(
            ctx.accounts,
            computation_offset,
            args,
            None,
            vec![MyInstructionCallback::callback_ix(
                computation_offset,
                &ctx.accounts.mxe_account,
                &[
                    CallbackAccount {
                        pubkey: ctx.accounts.state_account.key(),
                        is_signer: false,
                        is_writable: true,
                    },
                ],
            )?],
            1,  // num_return_outputs
            0,  // reserved
        )?;
        Ok(())
    }

    // 3. Callback - receive MPC results
    #[arcium_callback(encrypted_ix = "my_instruction")]
    pub fn my_instruction_callback(
        ctx: Context<MyInstructionCallback>,
        output: SignedComputationOutputs<MyInstructionOutput>,
    ) -> Result<()> {
        let o = match output.verify_output(
            &ctx.accounts.cluster_account,
            &ctx.accounts.computation_account
        ) {
            Ok(MyInstructionOutput { field_0 }) => field_0,
            Err(_) => return Err(ErrorCode::AbortedComputation.into()),
        };

        // Update encrypted state
        ctx.accounts.state_account.encrypted_data = o.ciphertexts[0];
        ctx.accounts.state_account.nonce = o.nonce;

        Ok(())
    }
}
```

## Offchain Circuit Source (Required)

Every `init_comp_def` must specify the circuit source. Use `CircuitSource::OffChain`:

```rust
init_comp_def(
    ctx.accounts,
    Some(CircuitSource::OffChain(OffChainCircuitSource {
        source: "https://raw.githubusercontent.com/user/circuits/main/circuit_name.arcis".to_string(),
        hash: circuit_hash!("circuit_name"),  // Reads build/*.hash at compile time
    })),
    None,
)?;
```

**Do NOT pass `None` for the circuit source** — ARX nodes need to know where to fetch the circuit.

## Three-Instruction Pattern

### 1. Init Computation Definition

```rust
#[init_computation_definition_accounts("my_instruction", payer)]
#[derive(Accounts)]
pub struct InitMyInstructionCompDef<'info> {
    #[account(mut)]
    pub payer: Signer<'info>,
    #[account(mut, address = derive_mxe_pda!())]
    pub mxe_account: Box<Account<'info, MXEAccount>>,
    #[account(mut)]
    /// CHECK: Initialized by arcium program
    pub comp_def_account: UncheckedAccount<'info>,
    pub arcium_program: Program<'info, Arcium>,
    pub system_program: Program<'info, System>,
}
```

### 2. Queue Computation

```rust
#[queue_computation_accounts("my_instruction", payer)]
#[derive(Accounts)]
#[instruction(computation_offset: u64)]
pub struct MyInstruction<'info> {
    #[account(mut)]
    pub payer: Signer<'info>,

    #[account(
        init_if_needed, space = 9, payer = payer,
        seeds = [&SIGN_PDA_SEED], bump,
        address = derive_sign_pda!(),
    )]
    pub sign_pda_account: Account<'info, ArciumSignerAccount>,

    #[account(address = derive_mxe_pda!())]
    pub mxe_account: Account<'info, MXEAccount>,

    #[account(mut, address = derive_mempool_pda!(mxe_account, ErrorCode::ClusterNotSet))]
    pub mempool_account: UncheckedAccount<'info>,

    #[account(mut, address = derive_execpool_pda!(mxe_account, ErrorCode::ClusterNotSet))]
    pub executing_pool: UncheckedAccount<'info>,

    #[account(mut, address = derive_comp_pda!(computation_offset, mxe_account, ErrorCode::ClusterNotSet))]
    pub computation_account: UncheckedAccount<'info>,

    #[account(address = derive_comp_def_pda!(COMP_DEF_OFFSET_MY_INSTRUCTION))]
    pub comp_def_account: Account<'info, ComputationDefinitionAccount>,

    #[account(mut, address = derive_cluster_pda!(mxe_account, ErrorCode::ClusterNotSet))]
    pub cluster_account: Account<'info, Cluster>,

    #[account(mut, address = ARCIUM_FEE_POOL_ACCOUNT_ADDRESS)]
    pub pool_account: Account<'info, FeePool>,

    #[account(mut, address = ARCIUM_CLOCK_ACCOUNT_ADDRESS)]
    pub clock_account: Account<'info, ClockAccount>,

    pub system_program: Program<'info, System>,
    pub arcium_program: Program<'info, Arcium>,

    // Your custom accounts (state, user position, etc.)
    #[account(mut)]
    pub state_account: Account<'info, YourState>,
}
```

### 3. Callback

```rust
#[callback_accounts("my_instruction")]
#[derive(Accounts)]
pub struct MyInstructionCallback<'info> {
    pub arcium_program: Program<'info, Arcium>,

    #[account(address = derive_comp_def_pda!(COMP_DEF_OFFSET_MY_INSTRUCTION))]
    pub comp_def_account: Account<'info, ComputationDefinitionAccount>,

    #[account(address = derive_mxe_pda!())]
    pub mxe_account: Account<'info, MXEAccount>,

    /// CHECK: Verified by arcium program
    pub computation_account: UncheckedAccount<'info>,

    #[account(address = derive_cluster_pda!(mxe_account, ErrorCode::ClusterNotSet))]
    pub cluster_account: Account<'info, Cluster>,

    #[account(address = ::anchor_lang::solana_program::sysvar::instructions::ID)]
    /// CHECK: instructions sysvar
    pub instructions_sysvar: AccountInfo<'info>,

    // Custom accounts (must match CallbackAccount array order)
    #[account(mut)]
    pub state_account: Account<'info, YourState>,
}
```

## Handling Tuple Returns in Callbacks

When a circuit returns a tuple like `(Enc<Mxe, A>, Enc<Mxe, B>)`, the output is wrapped:

```rust
#[arcium_callback(encrypted_ix = "deposit")]
pub fn deposit_callback(
    ctx: Context<DepositCallback>,
    output: SignedComputationOutputs<DepositOutput>,
) -> Result<()> {
    // Verify and unwrap tuple
    let tuple = match output.verify_output(
        &ctx.accounts.cluster_account,
        &ctx.accounts.computation_account
    ) {
        Ok(DepositOutput { field_0 }) => field_0,
        Err(_) => return Err(ErrorCode::AbortedComputation.into()),
    };

    // Access tuple elements:
    // tuple.field_0 = first return value (ciphertexts + nonce)
    // tuple.field_1 = second return value (ciphertexts + nonce)

    // Update vault state (3 encrypted u64s)
    ctx.accounts.vault.vault_state[0] = tuple.field_0.ciphertexts[0];
    ctx.accounts.vault.vault_state[1] = tuple.field_0.ciphertexts[1];
    ctx.accounts.vault.vault_state[2] = tuple.field_0.ciphertexts[2];
    ctx.accounts.vault.nonce = tuple.field_0.nonce;

    // Update user position (2 encrypted u64s)
    ctx.accounts.user_position.position_state[0] = tuple.field_1.ciphertexts[0];
    ctx.accounts.user_position.position_state[1] = tuple.field_1.ciphertexts[1];
    ctx.accounts.user_position.nonce = tuple.field_1.nonce;

    Ok(())
}
```

**Key pattern:** `OutputType { field_0 }` → `field_0.field_0` and `field_0.field_1` for tuple elements. Each element has `.ciphertexts[N]` (array of [u8; 32]) and `.nonce` (u128).

## Custom Callback Accounts

```rust
vec![DepositCallback::callback_ix(
    computation_offset,
    &ctx.accounts.mxe_account,
    &[
        CallbackAccount {
            pubkey: ctx.accounts.vault.key(),
            is_signer: false,
            is_writable: true,  // Must be true to update state
        },
        CallbackAccount {
            pubkey: ctx.accounts.user_position.key(),
            is_signer: false,
            is_writable: true,
        },
    ]
)?]
```

**Requirements:**
- Accounts must exist before callback executes
- Cannot create or resize accounts in callback
- Order must match callback struct (after the 6 standard Arcium accounts)
- `is_writable: true` required for any account you update in callback

## Account Patterns

### Encrypted State Account
```rust
#[account]
#[derive(InitSpace)]
pub struct VaultAccount {
    pub bump: u8,
    pub authority: Pubkey,
    pub token_mint: Pubkey,
    /// Encrypted state: [field1, field2, field3] as [u8; 32] each
    pub vault_state: [[u8; 32]; 3],
    pub nonce: u128,
}
```

### init_if_needed for Idempotent Creation
```rust
#[account(
    init_if_needed,
    payer = authority,
    space = 8 + VaultAccount::INIT_SPACE,
    seeds = [b"vault", token_mint.key().as_ref()],
    bump,
)]
pub vault: Account<'info, VaultAccount>,
```

## Error Handling

```rust
#[error_code]
pub enum ErrorCode {
    #[msg("The computation was aborted")]
    AbortedComputation,
    #[msg("Cluster not set")]
    ClusterNotSet,
    #[msg("Unauthorized")]
    Unauthorized,
}
```

## Events

```rust
#[event]
pub struct DepositEvent {
    pub vault: Pubkey,
    pub user: Pubkey,
    pub computation_offset: u64,
}

#[event]
pub struct RevealEvent {
    pub vault: Pubkey,
    pub revealed_amount: u64,
}
```
