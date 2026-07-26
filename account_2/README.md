# Hunt #2 — account_2: NFT Marketplace Unauthorized Authority Transfer

## Summary

The `update_profile_authority` instruction's `authority` account is declared as `UncheckedAccount` instead of `Signer`. Although the account context includes `has_one = authority` on `user_profile`, that constraint only verifies pubkey equality — it does not require a signature. Anyone who knows a profile's current authority pubkey (public information) can call `update_profile_authority` and reassign ownership to any key they choose, without ever possessing the original owner's private key.

## Vulnerability Class

Missing signer check on a privileged authorization-transfer instruction.

## Severity: Critical

**Why:** This is a direct authentication bypass on the single most sensitive instruction in the program — the one that reassigns ownership itself. No knowledge beyond a public key is required, exploit complexity is trivial (one instruction call), and impact is complete, irreversible account takeover. There is no scenario where this finding would be rated below Critical.

## Root Cause

```rust
#[derive(Accounts)]
pub struct UpdateProfileAuthority<'info> {
    #[account(mut, has_one = authority)]
    pub user_profile: Account<'info, UserProfile>,
    /// CHECK: New authority to set
    pub new_authority: UncheckedAccount<'info>,
    /// CHECK: Current authority account
    pub authority: UncheckedAccount<'info>,
}
```

`has_one = authority` compiles to a pubkey equality check — approximately `require_keys_eq!(user_profile.authority, ctx.accounts.authority.key())`. It is entirely independent of the Rust type used for `authority`. The signature requirement Solana/Anchor programs rely on comes from a completely separate mechanism: the `Signer<'info>` wrapper type, which checks `is_signer` on the account during Anchor's account validation phase, before the instruction body executes.

Because `authority` here is `UncheckedAccount`, Anchor never checks whether that account actually signed the transaction. The instruction only verifies the _pubkey supplied_ matches `user_profile.authority` — it never verifies that the real owner of that pubkey authorized the call.

**Key lesson embedded in this bug:** `has_one` and `Signer` answer two different questions. `has_one` answers "is this the correct account." `Signer` answers "did the actual owner of this account authorize this transaction." A privileged instruction needs both. Seeing `has_one = authority` in a constraint list can create a false sense of security if the paired account type isn't checked — it's easy to visually associate the word "authority" in `has_one` with an authorization guarantee that isn't actually there.

## Exploit Scenario

1. Alice initializes her profile PDA and calls `list_nft`, establishing legitimate ownership and state (`nft_count = 1`).
2. Bob — who only needs Alice's public key (visible on-chain, not secret) — calls `update_profile_authority`, passing `userProfile: aliceProfilePda`, `newAuthority: bob.publicKey`, `authority: alice.publicKey`. Critically, **Bob never includes Alice as a transaction signer** — only the provider/fee-payer wallet signs.
3. The transaction succeeds. `has_one = authority` passes because the pubkey supplied (`alice.publicKey`) does match `user_profile.authority` — but since `authority` was never required to sign, nothing checks that Alice actually approved this.
4. `user_profile.authority` is now Bob's key. Alice's subsequent `list_nft` call fails with `Unauthorized`. Bob's `list_nft` call succeeds.

## Proof of Concept

See `account_2.ts`. Sequence: Alice initializes profile → lists NFT (nft_count = 1) → Bob calls `update_profile_authority` without Alice as a signer → transaction succeeds and `user_profile.authority` becomes Bob's key → Alice's follow-up `list_nft` call fails with `Unauthorized` → Bob's `list_nft` call succeeds, confirming full takeover.

## Fix

```rust
#[derive(Accounts)]
pub struct UpdateProfileAuthority<'info> {
    #[account(mut, has_one = authority)]
    pub user_profile: Account<'info, UserProfile>,
    /// CHECK: New authority to set
    pub new_authority: UncheckedAccount<'info>,
    pub authority: Signer<'info>, // FIXED: now requires signature from current authority
}
```

Changing `authority` from `UncheckedAccount` to `Signer` adds the missing half of the authorization check: Anchor now rejects the transaction outright (during validation, before the instruction body runs) if the account in the `authority` slot did not actually sign. Combined with the existing `has_one = authority` pubkey check, the instruction now correctly requires both "correct account" and "actual signature from that account's owner" — which together constitute real authorization.

## Proof of Fix

See `account_2_fixed.ts`. The identical unauthorized attempt (Bob calling `update_profile_authority` with Alice's pubkey but without her signature) now fails during transaction validation. Alice's ownership is confirmed unchanged afterward, and she retains full functional access (`list_nft` still succeeds for her). The test then demonstrates the _legitimate_ transfer path — Alice actually signs `update_profile_authority` herself — succeeding normally, proving the fix blocks the unauthorized path without breaking intended functionality.

## Key Learning

`has_one` constraints verify pubkey equality only — they are not a substitute for `Signer<'info>` and do not imply a signature requirement. Any instruction that transfers privilege, ownership, or authority must have the relevant account typed as `Signer<'info>`, not `UncheckedAccount` or plain `AccountInfo`, regardless of what other constraints (like `has_one`) are also present. When reviewing any privileged instruction, explicitly check the _type_ of every account referenced in an authorization-relevant constraint — don't assume a constraint's presence implies the strongest possible check; verify what it actually compiles down to.

## References

- Anchor `Signer<'info>` type: enforces `is_signer` validation during account deserialization
- Anchor `has_one` constraint: performs field-level pubkey equality only, independent of signer status
- Related bug class: "Missing Signer Check" — a canonical entry in Neodyme's and Sec3's public Solana security checklists, frequently paired with over-trusted `has_one`/ownership-adjacent constraints
