# Hunt #1 — account_1: NFT Marketplace User Profile Reinitialization

## Summary

The `initialize_user_profile` instruction uses an `UncheckedAccount` for `user_profile` and performs a raw, unconditional write of account data with no check for prior initialization. Any signer can call this instruction against an already-initialized profile account belonging to another user and completely overwrite it — taking control of that profile and everything tied to it (NFT listing counts, sales history, active status).

## Vulnerability Class

Missing initialization-state check → Account Reinitialization Attack
(Related but distinct from a signer-check bypass — see Root Cause below for the important distinction.)

## Severity: Critical

**Why:** This results in complete, unauthorized account takeover with no privileged access required — just a valid signature from _any_ keypair, including the attacker's own. The impact directly affects protocol-tracked state (ownership, NFT counts, sales totals) with no additional preconditions beyond knowing the target account's public key, which is typically public/discoverable on-chain. Low exploit complexity + full state compromise = Critical.

## Root Cause

`user_profile` is declared as `UncheckedAccount`, which means Anchor performs **zero** automatic validation — no owner check, no discriminator check, no "is this already initialized" check. The instruction handler then does a manual raw byte write:

```rust
let user_profile = &ctx.accounts.user_profile;
let mut data = user_profile.data.borrow_mut();
let discriminator = <UserProfile as anchor_lang::Discriminator>::DISCRIMINATOR;
data[..8].copy_from_slice(&discriminator);
// ...serializes and writes profile struct unconditionally
```

There is no guard checking whether `data[..8]` already matches `UserProfile::DISCRIMINATOR`, or whether the account already holds a non-default `authority`. The function will happily stomp existing data every single time it's called, for anyone who signs.

**Important correction on attack mechanics:** this is _not_ a signer-check bypass. The attacker (Bob) signs the transaction with his own legitimate keypair as `authority`. Anchor's `Signer<'info>` constraint is satisfied honestly. The actual flaw is that the program has no concept of "this account belongs to someone else already" — it treats every call to `initialize_user_profile` as a fresh, valid initialization regardless of the account's actual prior state. This distinction matters because the fix isn't "add a signer check" (one already exists and works correctly) — it's "prevent re-initialization of an already-initialized account."

## Exploit Scenario

1. Alice creates the profile account on-chain (via `SystemProgram.createAccount`, owned by the program, pre-allocated space) and calls `initialize_user_profile("alice_the_artist")`, setting `authority = alice.pubkey()`.
2. Alice calls `list_nft`, incrementing her `nft_count` to 1 — this succeeds because `list_nft` correctly checks `profile.authority == ctx.accounts.authority.key()`.
3. Bob, an unrelated attacker, calls `initialize_user_profile("bob_the_hacker")` passing **Alice's existing profile account pubkey** as `user_profile`, and his own keypair as `authority`. No check stops him — the function blindly overwrites the account: `authority` becomes Bob's key, `username` becomes "bob_the_hacker", `nft_count` resets to 0.
4. Alice attempts to call `list_nft` on what she believes is still her profile — it now fails with `Unauthorized`, because `profile.authority` is now Bob's key, not hers.
5. Bob calls `list_nft` successfully — he now fully controls the account Alice originally created and had been using.

## Proof of Concept

See `account_1.ts` — full working PoC demonstrating:

- Alice initializes and lists an NFT (count = 1)
- Bob reinitializes the same account with his own authority (overwrites Alice's data, count resets to 0)
- Alice's subsequent `list_nft` call fails with `Unauthorized`
- Bob's subsequent `list_nft` call succeeds — full account takeover confirmed

## Fix

Replace `UncheckedAccount` with Anchor's `init` constraint plus PDA seeds derived from the authority:

```rust
#[account(
    init,
    payer = authority,
    space = 8 + 32 + (4 + 32) + 8 + 8 + 1,
    seeds = [b"profile", authority.key().as_ref()],
    bump
)]
pub user_profile: Account<'info, UserProfile>,
```

This closes the vulnerability through two independent, stacking protections:

1. **`init` constraint** — Anchor automatically verifies the account is not already initialized (checks discriminator/lamports state) before allowing creation, and errors (`already in use`) if it is.
2. **PDA seeds tied to `authority`** — the profile address is deterministically derived from the caller's own pubkey. Bob's authority key can only ever derive _Bob's own_ PDA — there is no way for his signature to target Alice's PDA address at all. This is why the fixed test correctly expects a `"A seeds constraint was violated"` error rather than an "already initialized" error: the seeds mismatch is caught before Anchor even reaches the init check.

## Key Learning

Always verify Anchor's `init` constraint (or an explicit manual "is this already initialized" check) is present on any account-creation instruction. An `UncheckedAccount` used for account initialization removes every one of Anchor's default guarantees — including the one guarantee most people assume is automatic: that you can't initialize the same account twice. Pair `init` with PDA seeds derived from the legitimate owner's key wherever a 1:1 user-to-account relationship should be structurally enforced, not just logically assumed.

## References

- Anchor `init` constraint docs: verifies account is not already in use before allowing creation
- General bug class: "Reinitialization Attack" — appears in Neodyme's and Sec3's public Solana security checklists as a canonical Anchor account-safety finding
