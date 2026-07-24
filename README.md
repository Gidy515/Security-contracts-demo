# Solana Program Security Audits

A collection of Solana/Anchor program vulnerabilities I've identified and analyzed as part of my security research practice. Each entry includes a vulnerable version of the program, a fixed version, and a breakdown of the bug class, root cause, exploit path, and remediation.

The goal of this repo is to build pattern recognition across common Solana/Anchor vulnerability classes (missing signer checks, missing owner checks, PDA seed issues, account validation gaps, arithmetic errors, etc.) through hands-on analysis rather than passive reading.

## Structure

Each program has its own directory containing the vulnerable version, the fixed version, and a writeup.

```
/account1
  /vulnerable
  /fixed
  README.md
/account2
  /vulnerable
  /fixed
  README.md
/account3
  /vulnerable
  /fixed
  README.md
/account9
  /vulnerable
  /fixed
  README.md
```

## Programs Audited

| # | Program | Vulnerability Class | Severity | Status |
|---|---------|---------------------|----------|--------|
| 1 | [account1](./account1) | TBD | TBD | 🔲 In progress |
| 2 | [account2](./account2) | TBD | TBD | 🔲 In progress |
| 3 | [account3](./account3) | TBD | TBD | 🔲 In progress |
| 9 | [account9](./account9) | TBD | TBD | 🔲 In progress |

## Writeup Format

Each program's README follows this structure:

- **Summary** — one-line description of the bug
- **Vulnerability Class** — e.g. missing signer check, missing owner check, PDA seed collision, integer overflow/underflow, missing rent-exemption check, type confusion, reentrancy via CPI, oracle manipulation, access control
- **Severity** — Critical / High / Medium / Low, with reasoning
- **Root Cause** — what's actually wrong in the code, and why
- **Exploit Scenario** — how an attacker would realistically exploit this
- **Proof of Concept** — test/script demonstrating the exploit (where applicable)
- **Fix** — what changed between vulnerable and fixed versions, and why it resolves the issue
- **References** — related real-world exploits, Anchor docs, or Solana security best practices this maps to

## Why This Repo Exists

Part of a structured self-study path toward Solana security research and bug bounty hunting (Immunefi, Sherlock, Cantina). Building this alongside broader smart contract development work on Solana/Anchor.

## Disclaimer

All vulnerabilities documented here are either intentionally constructed for educational purposes or based on publicly disclosed issues. Nothing in this repo targets or discloses undisclosed vulnerabilities in live/production programs.
