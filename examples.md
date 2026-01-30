# Arcium Examples Reference

## Example Projects

From the [Arcium examples repository](https://github.com/arcium-hq/examples):

| Example | Pattern | Complexity |
|---------|---------|------------|
| coinflip | Stateless RNG | Beginner |
| rock_paper_scissors | Hidden moves | Beginner |
| voting | State accumulation | Intermediate |
| share_medical_records | Re-encryption | Intermediate |
| sealed_bid_auction | Encrypted comparison | Intermediate |
| blackjack | Bit-packing | Advanced |
| ed25519 | Distributed signing | Advanced |

---

## Pattern: Stateless Computation (Coinflip)

**Use when:** Single operation, no persistent state needed.

```rust
// Arcis circuit
#[instruction]
pub fn flip(input: Enc<Shared, UserChoice>) -> bool {
    let choice = input.to_arcis();
    let toss = ArcisRNG::bool();
    (choice.heads == toss).reveal()
}
```

**Characteristics:**
- No game state account
- Result returned directly
- Each call is independent

---

## Pattern: State Accumulation (Voting)

**Use when:** Aggregating encrypted values over time.

```rust
// State struct
pub struct VoteState {
    yes_count: u64,
    no_count: u64,
}

// Arcis circuit
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

    state.owner.from_arcis(s)
}

// Reveal only comparison
#[instruction]
pub fn reveal_winner(state: Enc<Mxe, VoteState>) -> bool {
    let s = state.to_arcis();
    (s.yes_count > s.no_count).reveal()
}
```

**Account storage:**
```rust
#[account]
pub struct PollAccount {
    pub vote_state: [[u8; 32]; 2],  // 2 encrypted u64s
    pub nonce: u128,
    pub authority: Pubkey,
}
```

**Key technique:** Reading encrypted account data with byte offsets.

---

## Pattern: Re-encryption / Sealing (Medical Records)

**Use when:** Transferring encrypted data between parties.

```rust
#[instruction]
pub fn share_with_doctor(
    doctor: Shared,                    // Doctor's public key
    patient_data: Enc<Shared, Record>,
) -> Enc<Shared, Record> {
    let data = patient_data.to_arcis();
    doctor.from_arcis(data)  // Re-encrypt for doctor
}
```

**Characteristics:**
- Data never decrypted publicly
- Only specified recipient can decrypt output
- Original owner loses access (unless also re-encrypted for them)

---

## Pattern: Encrypted Comparison (Sealed Auction)

**Use when:** Comparing encrypted values without revealing them.

```rust
pub struct AuctionState {
    highest_bid: u64,
    highest_bidder_lo: u128,
    highest_bidder_hi: u128,
    second_highest_bid: u64,
}

#[instruction]
pub fn place_bid(
    bid: Enc<Shared, BidInput>,
    state: Enc<Mxe, AuctionState>,
) -> Enc<Mxe, AuctionState> {
    let b = bid.to_arcis();
    let mut s = state.to_arcis();

    if b.amount > s.highest_bid {
        // New highest bid
        s.second_highest_bid = s.highest_bid;
        s.highest_bid = b.amount;
        s.highest_bidder_lo = b.bidder_lo;
        s.highest_bidder_hi = b.bidder_hi;
    } else if b.amount > s.second_highest_bid {
        // New second highest
        s.second_highest_bid = b.amount;
    }

    state.owner.from_arcis(s)
}
```

**Supports:**
- First-price auction (winner pays their bid)
- Vickrey auction (winner pays second-highest bid)

---

## Pattern: Bit-Packing Compression (Blackjack)

**Use when:** Large arrays of small values exceed transaction limits.

**Problem:** 52 cards × 32 bytes = 1,664 bytes (exceeds 1,232 byte limit)

**Solution:** Pack using base-64 encoding (6 bits per card):
- 52 cards × 6 bits = 312 bits
- Fits in 3 × u128 = 384 bits
- After encryption: 3 × 32 = 96 bytes (94% reduction)

```rust
const POWS_OF_64: [u128; 21] = [1, 64, 4096, ...];

fn pack_cards(cards: &[u8; 21]) -> u128 {
    let mut packed: u128 = 0;
    for i in 0..21 {
        packed += POWS_OF_64[i] * cards[i] as u128;
    }
    packed
}

fn unpack_cards(packed: u128) -> [u8; 21] {
    let mut cards = [0u8; 21];
    let mut temp = packed;
    for i in 0..21 {
        cards[i] = (temp % 64) as u8;
        temp /= 64;
    }
    cards
}
```

**Account structure:**
```rust
pub struct BlackjackGame {
    pub deck: [[u8; 32]; 3],      // 52 cards in 3 packed u128s
    pub player_hand: [u8; 32],    // Up to 21 cards in 1 u128
    pub dealer_hand: [u8; 32],
    pub deck_nonce: u128,
    pub player_nonce: u128,
    pub dealer_nonce: u128,
}
```

---

## Pattern: Distributed Signing (Ed25519)

**Use when:** Key should never exist in single location.

```rust
// Generate distributed key
#[instruction]
pub fn generate_keypair() -> ArcisEd25519PublicKey {
    let sk = SecretKey::new_rand();
    let vk = VerifyingKey::from_secret_key(&sk);
    vk.reveal()  // Only reveal public key
}

// Sign with MXE's collective key
#[instruction]
pub fn sign(message: [u8; 32]) -> ArcisEd25519Signature {
    MXESigningKey::sign(&message).reveal()
}

// Verify with hidden public key
#[instruction]
pub fn verify(
    vk: Enc<Shared, CompressedVerifyingKey>,
    message: [u8; 32],
    signature: [u8; 64],
    observer: Shared,
) -> Enc<Shared, bool> {
    let verifying_key = VerifyingKey::from_compressed(vk.to_arcis());
    let sig = ArcisEd25519Signature::from_bytes(signature);
    let valid = verifying_key.verify(&message, &sig);
    observer.from_arcis(valid)
}
```

---

## Choosing the Right Pattern

| Requirement | Pattern |
|-------------|---------|
| One-time random outcome | Stateless (Coinflip) |
| Accumulate over time | State accumulation (Voting) |
| Share data privately | Re-encryption (Medical) |
| Compare without revealing | Encrypted comparison (Auction) |
| Large data, size limits | Bit-packing (Blackjack) |
| Distributed key control | MXE signing (Ed25519) |
