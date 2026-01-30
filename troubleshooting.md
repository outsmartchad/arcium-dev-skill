# Arcium Troubleshooting Guide

Common errors encountered during real-world development and their fixes.

## Build & Deploy Errors

### `Unknown action 'undefined'`
**Cause:** Stale IDL. You changed an account struct (e.g., added a field) but used `--skip-build`.
**Fix:** Run `arcium test` (without `--skip-build`) to regenerate the IDL.

### `ArgBuilder account size mismatch with circuit`
**Cause:** ArgBuilder doesn't match the circuit's `.idarc` descriptor.
**Fix:**
1. Check `build/*.idarc` for the circuit's expected input layout
2. Verify `Shared` standalone params have **both** `.x25519_pubkey()` AND `.plaintext_u128(nonce)`
3. Verify `Enc<Mxe, T>` params have **both** `.plaintext_u128(nonce)` AND `.account()`
4. Verify argument order matches circuit parameter order exactly

### `Stack offset exceeded max offset of 4096`
**Cause:** Arcium client library has large stack frames. This is a warning, not a hard error.
**Fix:** Generally safe to ignore. The build still succeeds.

### `anchor keys sync` overwrites `declare_id!`
**Cause:** `arcium build`/`arcium test` run `anchor keys sync` automatically.
**Fix:** Ensure `target/deploy/your_program-keypair.json` matches your deployed program ID. Use `--skip-keys-sync` if managing keys manually.

## Test Errors

### `Failed to fetch MXE public key after N attempts`
**Cause:** ARX nodes haven't completed MXE keygen yet.
**Fix:**
- On first localnet run: Wait longer (increase retry count to 120+ with 2s delay)
- On subsequent runs: MXE key should be available quickly
- If persists: Restart container and clean ledger: `docker restart && rm -rf .anchor/test-ledger`

### `Error 6000: AbortedComputation`
**Cause:** MPC computation was aborted (ARX node disagreement, circuit error, or transient failure).
**Fix:** Retry with a **new computation offset**. Never reuse the same offset.

### `Cannot read properties of undefined (reading 'publicKey')`
**Cause:** `mxePublicKey` is `undefined` because keygen hasn't completed.
**Fix:** Ensure MXE key fetch has proper retry logic with sufficient attempts.

### `Cannot read properties of undefined (reading 'encrypt')`
**Cause:** `cipher` is `undefined` because `RescueCipher` creation failed (no MXE key).
**Fix:** Same as above — ensure MXE key is fetched before creating cipher.

### Tests stuck at "Creating test token mint..."
**Cause:** Multiple competing `arcium test` processes running on the same validator.
**Fix:** `docker restart your-dev` to kill all zombie processes, then run a single `arcium test`.

### Zombie processes piling up
**Cause:** Interrupted `arcium test` runs leave orphan mocha/validator/arcium processes.
**Fix:** `docker restart your-dev` then `rm -rf .anchor/test-ledger` before next run.

## Runtime Errors

### Computation never finalizes (hangs forever)
**Cause:** ARX nodes couldn't fetch circuit from URL, or circuit hash mismatch.
**Fix:**
1. Verify circuit URL is reachable: `curl -I <circuit_url>`
2. Verify hash matches: rebuild with `arcium build`
3. Re-upload `.arcis` to GitHub if hash changed
4. Re-initialize comp def on-chain

### `Account does not exist or has no data`
**Cause:** MXE account, computation account, or comp def account not initialized.
**Fix:**
- MXE: Deploy with `arcium deploy` (handles MXE init)
- Comp def: Call `init_*_comp_def` instruction
- Computation: Generated automatically by `queue_computation`

### Callback doesn't update state
**Cause:** Missing `CallbackAccount` entries or wrong `is_writable` flag.
**Fix:** Ensure all accounts that need updating are listed in the `CallbackAccount` array with `is_writable: true`.

## Circuit Errors

### Circuit too expensive (exceeds ACU limit)
**Cause:** Division or complex operations in MPC are extremely expensive.
**Fix:**
- Move division/ratio calculations off-chain
- Use plaintext inputs where privacy allows
- Keep circuits under ~500M ACUs
- Avoid comparisons where possible (bit decomposition is expensive)

### Circuit rename doesn't take effect
**Cause:** Old `.arcis` file still on GitHub, old comp def still on-chain.
**Fix:**
1. `arcium build` to regenerate
2. Upload new `.arcis` to GitHub
3. Deploy/init new comp def (new PDA from new name hash)
4. Update all program and test references

## Docker Network Issues

### ARX nodes can't fetch circuits from external URL
**Cause:** Some hosts (e.g., catbox.moe) are unreliable from Docker containers.
**Fix:** Use GitHub raw URLs (`raw.githubusercontent.com`) — these work reliably from Docker. If GitHub is also unreachable, host locally:
```bash
# In container
cd /app/build && python3 -m http.server 8080 --bind 0.0.0.0
# Use http://172.20.0.2:8080/<circuit>.arcis in program
```

## Checklist: Before Running Tests

1. [ ] Container is clean (no zombie processes): `docker restart`
2. [ ] Ledger is clean: `rm -rf .anchor/test-ledger`
3. [ ] IDL matches code: use `arcium test` (not `--skip-build`) after struct changes
4. [ ] Circuit `.arcis` files uploaded to GitHub if changed
5. [ ] `declare_id!` matches keypair file (or use `--skip-keys-sync`)
6. [ ] Single test runner (no parallel `arcium test` commands)
