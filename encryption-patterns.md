# Encryption Patterns

## Encryption Types Overview

| Type | Decryption | Storage | Use Case |
|------|------------|---------|----------|
| `Enc<Shared, T>` | Client + MXE | Events, returns | User data |
| `Enc<Mxe, T>` | MXE only | Account state | Protocol data |
| Plaintext | Everyone | Anywhere | Public params |

## Client-Side Encryption (TypeScript)

```typescript
import {
  RescueCipher,
  x25519,
  getMXEPublicKey,
  deserializeLE,
} from "@arcium-hq/client";
import { randomBytes } from "crypto";

// 1. Generate client keypair
const privateKey = x25519.utils.randomSecretKey();
const publicKey = x25519.getPublicKey(privateKey);

// 2. Get MXE public key
const mxePublicKey = await getMXEPublicKey(provider, program.programId);

// 3. Derive shared secret
const sharedSecret = x25519.getSharedSecret(privateKey, mxePublicKey);

// 4. Create cipher
const cipher = new RescueCipher(sharedSecret);

// 5. Encrypt values
const nonce = randomBytes(16);
const plaintext = [BigInt(value1), BigInt(value2)];
const ciphertext = cipher.encrypt(plaintext, nonce);

// 6. Submit to program
await program.methods
  .myInstruction(
    computationOffset,
    Array.from(ciphertext[0]),
    Array.from(ciphertext[1]),
    Array.from(publicKey),
    new anchor.BN(deserializeLE(nonce).toString())
  )
  .rpc();
```

## Client-Side Decryption

```typescript
// After receiving callback event
const event = await awaitEvent("resultEvent");
const decrypted = cipher.decrypt(
  [event.data],
  event.nonce
);
console.log("Result:", decrypted[0]);
```

## Sealing (Re-encryption)

Transfer data from one key to another without exposing plaintext:

### Arcis Circuit
```rust
#[instruction]
pub fn share_with_doctor(
    doctor_pubkey: Shared,           // Doctor's public key
    patient_data: Enc<Shared, MedicalRecord>,
) -> Enc<Shared, MedicalRecord> {
    let data = patient_data.to_arcis();
    doctor_pubkey.from_arcis(data)   // Re-encrypt for doctor
}
```

### Use Cases
- Compliance (auditor access)
- Data marketplaces
- Credential sharing
- Multi-party workflows

## MXE-Only State (Protocol State)

For state that users shouldn't decrypt directly:

### Arcis Circuit
```rust
pub struct VoteState {
    yes_count: u64,
    no_count: u64,
}

#[instruction]
pub fn vote(
    user_vote: Enc<Shared, bool>,
    state: Enc<Mxe, VoteState>,
) -> Enc<Mxe, VoteState> {
    let vote = user_vote.to_arcis();
    let mut s = state.to_arcis();

    if vote {
        s.yes_count += 1;
    } else {
        s.no_count += 1;
    }

    // Re-encrypt for MXE (not client)
    state.owner.from_arcis(s)
}
```

### Storing MXE State On-Chain
```rust
#[account]
pub struct PollAccount {
    pub bump: u8,
    pub vote_state: [[u8; 32]; 2],  // 2 encrypted u64s
    pub nonce: u128,
    pub authority: Pubkey,
}
```

## Revealing Results

Only reveal aggregate/final results, not individual data:

```rust
#[instruction]
pub fn reveal_winner(
    state: Enc<Mxe, VoteState>,
) -> bool {
    let s = state.to_arcis();
    // Only reveal comparison, not actual counts
    (s.yes_count > s.no_count).reveal()
}
```

## Nonce Management

**Critical:** Never reuse nonces with same key!

```rust
// In callback, update stored nonce
ctx.accounts.state_account.nonce = output.nonce;
```

```typescript
// Client: always generate fresh nonce
const nonce = randomBytes(16);
```

## Encryption Size Reference

| Type | Encrypted Size |
|------|----------------|
| `u8` | 32 bytes |
| `u16` | 32 bytes |
| `u32` | 32 bytes |
| `u64` | 32 bytes |
| `u128` | 32 bytes |
| `bool` | 32 bytes |
| `[u8; N]` | N × 32 bytes |
| Struct with K fields | K × 32 bytes |

## Pack<T> for Compression

Compress large arrays of small values:

```rust
// 52 cards as [u8; 52] would be 52 × 32 = 1664 bytes
// Packed into 3 u128s = 3 × 32 = 96 bytes (94% savings)

pub struct PackedDeck {
    cards: [u128; 3],  // 21 + 21 + 10 cards in base-64
}
```

## Security Considerations

1. **Shared encryption key exposure**: Anyone with the shared secret can decrypt `Enc<Shared, T>`
2. **Nonce reuse**: Compromises security - always use fresh nonces
3. **Side channels**: Arcis uses constant-time operations to prevent timing attacks
4. **MXE state**: Use `Enc<Mxe, T>` for data users shouldn't see directly
