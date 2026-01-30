# Arcis Circuits (encrypted-ixs)

## Purpose
Arcis is Arcium's Rust-based framework for writing MPC circuits that run on encrypted data. Code compiles to fixed circuit structures before any data flows through.

## Project Structure

```
encrypted-ixs/
├── Cargo.toml
└── src/
    └── lib.rs    # All #[instruction] functions
```

## Cargo.toml

```toml
[package]
name = "encrypted-ixs"
version = "0.1.0"
edition = "2021"

[dependencies]
arcis = "=0.6.3"
blake3 = "=1.8.2"
```

## Basic Structure

```rust
use arcis::*;

#[encrypted]
mod circuits {
    use arcis::*;

    // Input struct
    pub struct MyInput {
        value: u64,
        flag: bool,
    }

    // MPC instruction
    #[instruction]
    pub fn process(input: Enc<Shared, MyInput>) -> Enc<Shared, u64> {
        let data = input.to_arcis();      // Decrypt to secret shares
        let result = data.value * 2;
        input.owner.from_arcis(result)    // Re-encrypt for client
    }
}
```

## Encryption Types

| Type | Who Can Decrypt | Use Case |
|------|-----------------|----------|
| `Enc<Shared, T>` | Client + MXE | User inputs/outputs |
| `Enc<Mxe, T>` | MXE only | Protocol state |
| Plaintext | Everyone (ARX nodes) | Public parameters |

## Core Operations

```rust
// Decrypt encrypted input to secret shares
let value = encrypted_input.to_arcis();

// Re-encrypt result for specific owner
let result = owner.from_arcis(computed_value);

// Reveal to plaintext (visible to all)
let public = secret_value.reveal();

// Get MXE owner for protocol state
let mxe_encrypted = Mxe::get().from_arcis(state);
```

## Supported Types

**Integers:** `u8`, `u16`, `u32`, `u64`, `u128`, `i8`-`i128`
**Floats:** `f32`, `f64` (fixed-point internally)
**Composite:** Arrays `[T; N]`, tuples `(A, B)`, structs
**Special:** `ArcisX25519Pubkey`, `BaseField`, `Pack<T>`

**NOT supported:** `Vec`, `String`, `HashMap`, enums, `Option`, `Result`

## Control Flow

```rust
// OK: if/else (both branches execute, cost = sum)
let result = if condition { a } else { b };

// OK: bounded for loops
for i in 0..10 {
    process(arr[i]);
}

// NOT OK: while, loop, break, continue, match, early return
```

## Randomness (ArcisRNG)

```rust
let coin = ArcisRNG::bool();
let num = ArcisRNG::gen_integer_from_width(64);
let uniform = ArcisRNG::gen_uniform::<u32>();
ArcisRNG::shuffle(&mut deck);
```

## Cryptographic Primitives

```rust
// Hashing
let hash = SHA3_256::new().digest(&data);

// Ed25519 signatures
let sk = SecretKey::new_rand();
let vk = VerifyingKey::from_secret_key(&sk);
let valid = vk.verify(&message, &signature);

// MXE cluster signing
let sig = MXESigningKey::sign(&message).reveal();
```

## Data Packing (compression)

For large arrays of small values:

```rust
// Pack [u8; 256] into ~10 field elements (26x compression)
let packed = Pack::new(large_array);
let unpacked: [u8; 256] = packed.unpack();
```

## Cost Hierarchy

| Operation | Cost |
|-----------|------|
| Add, subtract | Cheap |
| Multiply (constant) | Cheap |
| Multiply (secret) | Moderate |
| Comparisons | Expensive (bit decomposition) |
| Division/modulo | Very expensive |
| Dynamic indexing | O(n) |
| Sort | O(n·log²(n)·bit_size) |

## Common Patterns

### Accumulate encrypted state
```rust
#[instruction]
pub fn vote(
    user_vote: Enc<Shared, bool>,
    state: Enc<Mxe, VoteState>,
) -> Enc<Mxe, VoteState> {
    let vote = user_vote.to_arcis();
    let mut s = state.to_arcis();
    if vote { s.yes += 1; } else { s.no += 1; }
    state.owner.from_arcis(s)
}
```

### Re-encryption (sealing)
```rust
#[instruction]
pub fn share_data(
    receiver: Shared,
    data: Enc<Shared, Secret>,
) -> Enc<Shared, Secret> {
    let d = data.to_arcis();
    receiver.from_arcis(d)  // Re-encrypt for receiver
}
```

### Encrypted comparison
```rust
#[instruction]
pub fn place_bid(
    bid: Enc<Shared, u64>,
    state: Enc<Mxe, AuctionState>,
) -> Enc<Mxe, AuctionState> {
    let b = bid.to_arcis();
    let mut s = state.to_arcis();
    if b > s.highest {
        s.second = s.highest;
        s.highest = b;
    }
    state.owner.from_arcis(s)
}
```

## Debugging

```rust
// Development only (no effect on circuit)
println!("value = {}", x);
debug_assert!(x > 0);
```

## Testing Limitations

`#[instruction]` functions require full MPC runtime. Unit test only:
- Helper functions
- `#[arcis_circuit]` builtins
- Pure logic extracted to separate functions

Use TypeScript + `arcium test` for integration tests.
