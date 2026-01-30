# TypeScript Client (@arcium-hq/client)

## Installation

```bash
npm install @arcium-hq/client
# or
yarn add @arcium-hq/client
```

## Core Imports

```typescript
import {
  // Encryption
  RescueCipher,
  x25519,
  getMXEPublicKey,
  deserializeLE,

  // Environment
  getArciumEnv,

  // Account addresses
  getMXEAccAddress,
  getMempoolAccAddress,
  getExecutingPoolAccAddress,
  getComputationAccAddress,
  getClusterAccAddress,
  getCompDefAccAddress,
  getCompDefAccOffset,
  getArciumAccountBaseSeed,
  getArciumProgramId,

  // Fee/Clock accounts
  getFeePoolAccAddress,
  getClockAccAddress,

  // Computation lifecycle
  awaitComputationFinalization,
} from "@arcium-hq/client";

import { randomBytes, createHash } from "crypto";
import * as anchor from "@coral-xyz/anchor";
```

## Full Test Structure (Real-World Pattern)

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { PublicKey, Keypair, SystemProgram } from "@solana/web3.js";
import { YourProgram } from "../target/types/your_program";
import { randomBytes, createHash } from "crypto";
import nacl from "tweetnacl";
import {
  awaitComputationFinalization,
  getArciumEnv,
  getCompDefAccOffset,
  getArciumAccountBaseSeed,
  getArciumProgramId,
  RescueCipher,
  deserializeLE,
  getMXEAccAddress,
  getMempoolAccAddress,
  getCompDefAccAddress,
  getExecutingPoolAccAddress,
  x25519,
  getComputationAccAddress,
  getMXEPublicKey,
  getClusterAccAddress,
  getFeePoolAccAddress,
  getClockAccAddress,
} from "@arcium-hq/client";
import * as fs from "fs";
import * as os from "os";
import { expect } from "chai";

describe("YourProgram", () => {
  anchor.setProvider(anchor.AnchorProvider.env());
  const program = anchor.workspace.YourProgram as Program<YourProgram>;
  const provider = anchor.getProvider() as anchor.AnchorProvider;

  // Shared test state
  let owner: Keypair;
  let mxePublicKey: Uint8Array;
  let cipher: RescueCipher;
  let encryptionKeys: { privateKey: Uint8Array; publicKey: Uint8Array };

  // Event listener helper
  type Event = anchor.IdlEvents<(typeof program)["idl"]>;
  const awaitEvent = async <E extends keyof Event>(
    eventName: E
  ): Promise<Event[E]> => {
    let listenerId: number;
    const event = await new Promise<Event[E]>((res) => {
      listenerId = program.addEventListener(eventName, (event) => {
        res(event);
      });
    });
    await program.removeEventListener(listenerId);
    return event;
  };

  const clusterAccount = getClusterAccAddress(getArciumEnv().arciumClusterOffset);

  before(async () => {
    owner = readKpJson(`${os.homedir()}/.config/solana/id.json`);

    // MXE key may not be available until after comp defs are initialized
    try {
      mxePublicKey = await getMXEPublicKeyWithRetry(provider, program.programId);
      encryptionKeys = deriveEncryptionKey(owner, "your-app-encryption-key-v1");
      const sharedSecret = x25519.getSharedSecret(encryptionKeys.privateKey, mxePublicKey);
      cipher = new RescueCipher(sharedSecret);
    } catch (e) {
      console.log("MXE key not available yet - will retry after comp def init");
    }
  });

  // ... tests ...
});
```

## Computation Retry Wrapper (Critical Pattern)

MPC computations can abort (e.g., node disagreement). Always use retry logic with a **new computation offset per attempt**:

```typescript
const MAX_COMPUTATION_RETRIES = 5;
const RETRY_DELAY_MS = 3000;

async function queueWithRetry(
  label: string,
  buildAndSend: (computationOffset: anchor.BN) => Promise<string>,
  provider: anchor.AnchorProvider,
  programId: PublicKey,
): Promise<string> {
  for (let attempt = 1; attempt <= MAX_COMPUTATION_RETRIES; attempt++) {
    // CRITICAL: New offset for each attempt (offsets are one-time-use)
    const computationOffset = new anchor.BN(randomBytes(8), "hex");
    try {
      console.log(`[${label}] Attempt ${attempt}/${MAX_COMPUTATION_RETRIES}`);
      const queueSig = await buildAndSend(computationOffset);

      const finalizeSig = await awaitComputationFinalization(
        provider,
        computationOffset,
        programId,
        "confirmed"
      );

      // Check if callback had an error (e.g., AbortedComputation = error 6000)
      const txResult = await provider.connection.getTransaction(finalizeSig, {
        commitment: "confirmed",
        maxSupportedTransactionVersion: 0,
      });

      if (txResult?.meta?.err) {
        console.log(`[${label}] Attempt ${attempt} ABORTED:`, txResult.meta.err);
        if (attempt < MAX_COMPUTATION_RETRIES) {
          await new Promise((r) => setTimeout(r, RETRY_DELAY_MS));
          continue;
        }
        throw new Error(`[${label}] All ${MAX_COMPUTATION_RETRIES} attempts aborted`);
      }

      return finalizeSig;
    } catch (err: any) {
      if (err.message?.includes("All") && err.message?.includes("aborted")) throw err;
      console.log(`[${label}] Attempt ${attempt} error:`, err.message);
      if (attempt < MAX_COMPUTATION_RETRIES) {
        await new Promise((r) => setTimeout(r, RETRY_DELAY_MS));
        continue;
      }
      throw err;
    }
  }
  throw new Error(`[${label}] Exhausted all retries`);
}
```

### Using the Retry Wrapper

```typescript
const finalizeSig = await queueWithRetry(
  "deposit",
  async (computationOffset) => {
    return program.methods
      .deposit(computationOffset, ciphertext, pubkey, nonce, amount)
      .accountsPartial({
        computationAccount: getComputationAccAddress(
          getArciumEnv().arciumClusterOffset,
          computationOffset  // Uses the fresh offset from retry wrapper
        ),
        // ... other accounts
      })
      .signers([owner])
      .rpc({ skipPreflight: true, commitment: "confirmed" });
  },
  provider,
  program.programId,
);
```

**Key insight:** The `computationOffset` is generated inside the retry loop, so each attempt gets a fresh PDA. The `buildAndSend` callback receives the new offset and must use it for both the instruction argument AND the `computationAccount` address derivation.

## MXE Public Key Fetch (With Retry)

MXE keygen happens asynchronously after comp defs are initialized. On localnet, ARX nodes need ~60s to come online. Use generous retry counts:

```typescript
async function getMXEPublicKeyWithRetry(
  provider: anchor.AnchorProvider,
  programId: PublicKey,
  maxRetries: number = 40,   // 40 retries for localnet (ARX node startup)
  retryDelayMs: number = 1000
): Promise<Uint8Array> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const mxePublicKey = await getMXEPublicKey(provider, programId);
      if (mxePublicKey) return mxePublicKey;
    } catch (error) {
      console.log(`MXE key attempt ${attempt}/${maxRetries} failed`);
    }
    if (attempt < maxRetries) {
      await new Promise((resolve) => setTimeout(resolve, retryDelayMs));
    }
  }
  throw new Error(`Failed to fetch MXE public key after ${maxRetries} attempts`);
}
```

**Retry counts:**
- Localnet first run: `maxRetries = 120, retryDelayMs = 2000` (ARX nodes booting)
- Localnet subsequent: `maxRetries = 40, retryDelayMs = 1000`
- Devnet: `maxRetries = 20, retryDelayMs = 500` (MXE already initialized)

## Deterministic Encryption Keys

Derive X25519 keypair deterministically from a Solana wallet (reproducible across sessions):

```typescript
import nacl from "tweetnacl";

function deriveEncryptionKey(
  wallet: anchor.web3.Keypair,
  message: string
): { privateKey: Uint8Array; publicKey: Uint8Array } {
  const messageBytes = new TextEncoder().encode(message);
  const signature = nacl.sign.detached(messageBytes, wallet.secretKey);
  const privateKey = new Uint8Array(
    createHash("sha256").update(signature).digest()
  );
  const publicKey = x25519.getPublicKey(privateKey);
  return { privateKey, publicKey };
}
```

**Why deterministic?** Same wallet always produces the same encryption keys, so you can decrypt past data without storing X25519 keys separately.

## Account Address Helpers

```typescript
const arciumEnv = getArciumEnv();
const clusterOffset = arciumEnv.arciumClusterOffset;  // From Arcium.toml

// MXE account (per-program)
const mxeAccount = getMXEAccAddress(program.programId);

// Cluster accounts (shared across programs)
const clusterAccount = getClusterAccAddress(clusterOffset);
const mempoolAccount = getMempoolAccAddress(clusterOffset);
const executingPool = getExecutingPoolAccAddress(clusterOffset);

// Fee and clock (global constants)
const poolAccount = getFeePoolAccAddress();
const clockAccount = getClockAccAddress();

// Per-computation account (unique per queue call)
const computationOffset = new anchor.BN(randomBytes(8), "hex");
const computationAccount = getComputationAccAddress(clusterOffset, computationOffset);

// Computation definition (per-instruction, derived from name)
const compDefOffset = getCompDefAccOffset("my_instruction");
const compDefAccount = getCompDefAccAddress(
  program.programId,
  Buffer.from(compDefOffset).readUInt32LE()
);

// Sign PDA
const signPdaAccount = PublicKey.findProgramAddressSync(
  [Buffer.from("ArciumSignerAccount")],
  program.programId
)[0];
```

## Idempotent Comp Def Initialization

Check if comp def already exists before initializing (important for devnet re-runs):

```typescript
async function initCompDef(
  program: Program<YourProgram>,
  owner: Keypair,
  circuitName: string,
  methodName: string
): Promise<string> {
  const baseSeed = getArciumAccountBaseSeed("ComputationDefinitionAccount");
  const offset = getCompDefAccOffset(circuitName);

  const compDefPDA = PublicKey.findProgramAddressSync(
    [baseSeed, program.programId.toBuffer(), offset],
    getArciumProgramId()
  )[0];

  // Check if already initialized
  const accountInfo = await provider.connection.getAccountInfo(compDefPDA);
  if (accountInfo !== null && accountInfo.owner.equals(getArciumProgramId())) {
    console.log(`${circuitName} comp def already exists, skipping`);
    return "skipped";
  }

  // Initialize - program contains the offchain URL and hash
  // @ts-ignore - Dynamic method call
  const sig = await program.methods[methodName]()
    .accounts({
      compDefAccount: compDefPDA,
      payer: owner.publicKey,
      mxeAccount: getMXEAccAddress(program.programId),
    })
    .signers([owner])
    .rpc();

  return sig;
}
```

## Encryption Utilities

```typescript
// Encrypt a single value
const value = BigInt(1_000_000_000);
const nonce = randomBytes(16);
const ciphertext = cipher.encrypt([value], nonce);
// Pass to instruction: Array.from(ciphertext[0]) as number[]

// Encrypt multiple values
const values = [BigInt(10), BigInt(20), BigInt(30)];
const ciphertexts = cipher.encrypt(values, nonce);
// ciphertexts[0], ciphertexts[1], ciphertexts[2]

// Convert nonce to BN for instruction
const nonceBN = new anchor.BN(deserializeLE(nonce).toString());

// Decrypt
const decrypted = cipher.decrypt([ct0, ct1, ct2], nonceBytes);
// decrypted[0], decrypted[1], decrypted[2] as BigInt
```

## Common Patterns

### Queue Computation with Encrypted Input
```typescript
const depositAmount = BigInt(1_000_000_000);
const nonce = randomBytes(16);
const ciphertext = cipher.encrypt([depositAmount], nonce);

await program.methods
  .deposit(
    computationOffset,
    Array.from(ciphertext[0]) as number[],
    Array.from(encryptionKeys.publicKey) as number[],
    new anchor.BN(deserializeLE(nonce).toString()),
    new anchor.BN(depositAmount.toString())  // plaintext amount for token transfer
  )
  .accountsPartial({
    computationAccount: getComputationAccAddress(
      arciumEnv.arciumClusterOffset,
      computationOffset
    ),
    clusterAccount,
    mxeAccount: getMXEAccAddress(program.programId),
    mempoolAccount: getMempoolAccAddress(arciumEnv.arciumClusterOffset),
    executingPool: getExecutingPoolAccAddress(arciumEnv.arciumClusterOffset),
    compDefAccount: getCompDefAccAddress(
      program.programId,
      Buffer.from(getCompDefAccOffset("deposit")).readUInt32LE()
    ),
    poolAccount: getFeePoolAccAddress(),
    clockAccount: getClockAccAddress(),
    // ... your custom accounts
  })
  .signers([owner])
  .rpc({ skipPreflight: true, commitment: "confirmed" });
```

### Queue Computation with Shared Nonce (for Shared/Enc<Shared,T> outputs)
```typescript
// When circuit returns Enc<Shared, T>, pass both pubkey AND shared nonce
const sharedNonce = randomBytes(16);

await program.methods
  .withdraw(
    computationOffset,
    Array.from(encryptionKeys.publicKey) as number[],
    new anchor.BN(deserializeLE(sharedNonce).toString())
  )
  .accountsPartial({ /* ... */ })
  .rpc({ commitment: "confirmed" });
```

### MXE-Only Computation (No Client Encryption)
```typescript
// For Enc<Mxe, T> state reads — just pass nonce from on-chain account
const nonce = randomBytes(16);

await program.methods
  .revealPendingDeposits(computationOffset)
  .accountsPartial({ /* ... */ })
  .rpc({ skipPreflight: true, commitment: "confirmed" });
```

## Helper: Read Keypair

```typescript
function readKpJson(path: string): anchor.web3.Keypair {
  const file = fs.readFileSync(path);
  return anchor.web3.Keypair.fromSecretKey(
    new Uint8Array(JSON.parse(file.toString()))
  );
}
```

## PDA Derivation Examples

```typescript
// Vault PDA
const [vaultPda] = PublicKey.findProgramAddressSync(
  [Buffer.from("vault"), tokenMint.toBuffer()],
  program.programId
);

// User position PDA
const [userPositionPda] = PublicKey.findProgramAddressSync(
  [Buffer.from("position"), vaultPda.toBuffer(), owner.publicKey.toBuffer()],
  program.programId
);

// Relay PDA (for multi-relay patterns)
function deriveRelayPda(vaultPda: PublicKey, relayIndex: number, programId: PublicKey): PublicKey {
  const [pda] = PublicKey.findProgramAddressSync(
    [Buffer.from("zodiac_relay"), vaultPda.toBuffer(), Buffer.from([relayIndex])],
    programId
  );
  return pda;
}
```

## File Logging (Optional)

Useful for debugging MPC computations:

```typescript
const LOG_FILE = "test-run.log";
const logStream = fs.createWriteStream(LOG_FILE, { flags: "w" });
const origLog = console.log;
console.log = (...args: any[]) => {
  const line = args.map((a) =>
    typeof a === "object" ? JSON.stringify(a, null, 2) : String(a)
  ).join(" ");
  origLog(...args);
  logStream.write(line + "\n");
};

// At end of tests:
after(() => logStream.end());
```
