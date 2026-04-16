# Building the Radiant Ledger App with Claude AI

**A Retrospective on AI-Assisted Hardware Wallet Development**

This document shares lessons learned from building and validating the community Radiant Ledger app using Claude AI as a development partner. The goal of this guide is practical — if you're using Claude (or Cursor, or any AI coding assistant) to work on hardware wallet, consensus, or crypto protocol code, here's what worked.

---

## Table of Contents

1. [Context: What Was Built](#1-context-what-was-built)
2. [Why AI-Assisted Is a Good Fit for This Work](#2-why-ai-assisted-is-a-good-fit-for-this-work)
3. [Workflow Patterns That Worked](#3-workflow-patterns-that-worked)
4. [Patterns to Avoid](#4-patterns-to-avoid)
5. [Validation Strategy: Don't Trust the AI, Don't Trust Yourself](#5-validation-strategy-dont-trust-the-ai-dont-trust-yourself)
6. [Compound Knowledge](#6-compound-knowledge)
7. [Resources](#7-resources)

---

## 1. Context: What Was Built

Over approximately 48 hours of working sessions with Claude, we built:

- Fork of `LedgerHQ/app-bitcoin` adding a Radiant `COIN_KIND`
- C implementation of Radiant's `hashOutputHashes` preimage field (streaming FSM with opcode walker, ~500 lines across `lib-app-bitcoin`)
- Python port of `radiantjs` sighash reference (port of `lib/transaction/sighash.js` + `lib/script/script.js`)
- Test harnesses for device-vs-oracle signature validation
- Patched Electron Cash fork with SLIP-44 512 support and btchip vendoring
- 5 mainnet-confirmed transactions proving end-to-end signing: plain P2PKH, Glyph UTXO spending

Total mainnet cost of testing: ~4 RXD (fees + dust burned). No security incidents, no broken mainnet txs.

---

## 2. Why AI-Assisted Is a Good Fit for This Work

### What the AI Does Well

**Byte-level protocol work.** Translating "the preimage is `version || hashPrevouts || ...`" into working code is mostly mechanical — error-prone for a human (endianness bugs, off-by-ones in varint decoding, missing null terminators) but easy for an AI that can cross-check itself.

**Cross-reference two independent implementations.** When you have a JS reference (`radiantjs`) and a C++ consensus node (`radiant-node`), the AI can read both and flag discrepancies a human reader would miss.

**Running + interpreting long traces.** Device APDU debug output, compiled assembly, preimage byte dumps — all tedious to read manually. Claude parses them faster and catches anomalies.

**Repeating work at each step of a pipeline.** Commit a change. Re-sideload. Re-run the regression test. Re-run the oracle validation. Re-run the fixtures harness. Claude drives all of this in one go.

### What the AI Does Poorly Without Supervision

**Inventing plausible-looking numbers.** Several times Claude "computed" intermediate hashes manually, confidently reporting a result that turned out to be wrong. The fix is always the same: verify against actual tool output, not derivation in the model's head.

**Guessing instead of instrumenting.** When sig verification failed, the AI's instinct was to try more sighash variants (we tried 11). The breakthrough came from enabling APDU-level debug logging and comparing actual bytes. Instrumentation > speculation.

**Silently swallowed exceptions.** Claude didn't invent the `try: ... except Exception: pass` pattern in btchip — that was upstream code — but it did keep missing the hidden failure because the surface-level error was a different symptom. Always ask: "is there a swallowed exception somewhere upstream?"

### The Right Division of Labor

| Task | Who Leads | Why |
|------|-----------|-----|
| Writing new FSM code | AI | Mechanical translation from spec |
| Cross-checking against reference impl | AI | Reads two repos in parallel |
| Running build + sideload + test loops | AI | Repetitive, benefits from automation |
| Identifying what sighash variant matches a sig | AI + oracle | Needs independent computation |
| Deciding to broadcast to mainnet | Human | Irreversible, non-trivial consequences |
| Deciding what tests to run | Human | Depends on what you're trying to prove |
| Understanding privacy/consensus implications | Human | Requires judgment beyond the codebase |

---

## 3. Workflow Patterns That Worked

### Pattern: "Baseline-Isolation Test FIRST"

When something new fails, run the same infrastructure against a known-working baseline before debugging the new thing. This pattern caught 4 stacked bugs in ~5 minutes that had evaded 2 hours of "maybe the opcode walker is broken" speculation.

```
Test Glyph spend → fails
Test plain P2PKH spend via SAME harness → ALSO fails → test harness is broken, not the walker
```

### Pattern: Python Oracle as Ground Truth

Before writing C firmware code, write a Python implementation of the same algorithm. Validate that Python implementation against real mainnet data. THEN use it as the reference when debugging the firmware.

```
radiantjs (reference) → radiant_preimage_oracle.py (our port)
                         ↓
                        triple-validated (3 independent sources)
                         ↓
                        golden fixtures (5 mainnet txs, 18 sighashes)
                         ↓
                        device-vs-oracle comparison
```

This lets you answer "did the device compute the right hash?" with a crisp yes/no, not a handwavy "looks OK."

### Pattern: Compound Docs After Every Non-Trivial Fix

Every time you solve a non-trivial issue, spend 5 minutes writing a `docs/solutions/[category]/[slug].md` file that includes:

- Symptom (exact error)
- Investigation arc (what you tried that didn't work — saves future you hours)
- Root cause (specific file + line)
- Fix (the diff)
- Prevention (how to avoid repeating)

Your compound knowledge grows over time. AI sessions can read your prior solutions and avoid repeating the same debugging arc.

This guide grew out of exactly this practice. See [`radiant-ledger-app/docs/solutions/`](https://github.com/Zyrtnin-org/radiant-ledger-app/tree/main/docs/solutions) for the full set.

### Pattern: "Walking Skeleton" Before "Fully Polished"

Build the smallest end-to-end thing that signs a real transaction, even if it's ugly, rejected by mainnet, has zero UI polish, etc. Landing the walking skeleton reveals fundamental issues (like "the preimage is wrong") that pure-simulation testing won't catch.

Our walking skeleton: first Ledger-signed tx sent to mainnet, got script-execution-error back. That rejection — at 1 RXD of risk — taught us about `hashOutputHashes`. Without the walking skeleton, we'd have shipped v1.0 with broken signing and found out via angry users.

### Pattern: One Commit Per Working Unit

Don't wait to "get everything working" before committing. Commit after:

- A unit of logic is complete + tested
- The oracle re-validates
- About to attempt something risky

Small commits make bisection trivial. When an unrelated regression shows up 3 commits later, you know exactly which commit introduced it.

---

## 4. Patterns to Avoid

### Anti-Pattern: Asking "Does This Work?"

Don't ask an AI "is this right?" or "does this work?" Those questions invite hedging, speculation, or false confidence. Ask instead:

- "Run this against the oracle. Does it verify?"
- "Build this. Does it compile clean?"
- "Sign a known-value tx. Does the signature match the expected mainnet sig?"

Make the question binary. Outsource the verification to a tool that has ground truth.

### Anti-Pattern: Destructive Actions Without Confirmation

Hardware, mainnet broadcasts, GitHub pushes, `git reset --hard` — all irreversible. AI sessions sometimes try to move fast by combining these with other work. Always confirm before:

- Broadcasting a tx with non-trivial amounts or non-trivial output shapes
- Pushing to origin (especially force-pushing)
- Destructive git operations
- Uninstalling + reinstalling the app
- Changing production configuration

### Anti-Pattern: Trusting Computed Intermediate Values

If the AI reports "hashPrevouts = abc123...", always ask: "was this computed by running code, or derived in your head?" If derived in head, don't trust it. Re-run with actual tooling.

This burned us exactly once during this project — a sub-agent reported an oracle intermediate value that didn't match actual oracle output. If we'd trusted it, we'd have concluded the oracle was wrong and embarked on a bogus debugging arc.

### Anti-Pattern: Trying to Patch Symptoms

When a signature doesn't verify, the AI's instinct is to try alternative preimage variants. The Nth variant will sometimes work, but you've patched a symptom. The right pattern is:

1. Dump the actual bytes the device received (instrumentation, not speculation)
2. Compute the preimage from those bytes
3. Compare to oracle's preimage byte-by-byte
4. Find the divergent field

Eleven variants couldn't find what the device was computing. APDU-level byte capture found it in five minutes.

---

## 5. Validation Strategy: Don't Trust the AI, Don't Trust Yourself

For consensus-critical code (sighash, signing, serialization), always have at least TWO independent implementations compare against each other. For this project:

- **Implementation A**: our C firmware (on-device)
- **Implementation B**: our Python oracle (host)
- **Implementation C**: radiantjs (canonical reference, reviewable)
- **Implementation D**: FlipperHub's Node.js signing (independent implementer)

A passes → B matches A → C matches B → D matches C. The transitive chain eliminates single-point-of-failure bugs.

### Golden Vectors

Once you have a trusted implementation, capture its output for real-world inputs as **golden vectors**. We have 5 fixtures covering 18 sighashes. Any change to the C firmware, the oracle, or the host plugin must keep producing the same sighashes. If not, investigate — you've either found a bug or broken the spec.

### Mainnet-as-Test

The ultimate test isn't your unit tests. It's: "does Radiant mainnet consensus accept this tx?" Aim for at least one such test per feature before calling it "shipped." Budget the few RXD of fees — it's cheap insurance against entire feature classes of bugs.

---

## 6. Compound Knowledge

This project specifically prioritized turning implicit learning into explicit artifacts:

- **`INVESTIGATION.md`** in `radiant-ledger-app` — per-phase findings, SHA256s, commit hashes
- **`docs/solutions/`** — compound fix docs with cross-references
- **`docs/plans/`** — pre-work planning that drove implementation
- **`docs/brainstorms/`** — decisions + rationale
- **Python oracle + fixtures** — executable reference
- **This guide + `radiant-ledger-app` README** — public surface

The cost of keeping these up to date was maybe 20% of total engineering time. The benefit is that anyone (including future-me) can read them and get up to speed in hours instead of days.

Pattern: **every AI session produces a commit AND a note about what was learned.** Don't let hard-won insights exist only as vibes in the head of the developer.

---

## 7. Resources

### Primary Repos

- [`radiant-ledger-guide`](https://github.com/Zyrtnin-org/radiant-ledger-guide) — this guide
- [`app-radiant`](https://github.com/Zyrtnin-org/app-radiant) — the Ledger app (user-facing)
- [`lib-app-bitcoin`](https://github.com/Zyrtnin-org/lib-app-bitcoin) — C diff (submodule of above)
- [`Electron-Wallet@radiant-ledger-512`](https://github.com/Zyrtnin-org/Electron-Wallet/tree/radiant-ledger-512) — host wallet
- [`radiant-ledger-app`](https://github.com/Zyrtnin-org/radiant-ledger-app) — planning, oracle, fixtures, investigation

### Related Protocol Docs

- [`radiant-glyph-guide`](https://github.com/Zyrtnin-org/radiant-glyph-guide) — Glyph minting (AI-agent optimized)
- [`radiantjs`](https://github.com/RadiantBlockchain/radiantjs) — canonical JS reference
- [`radiant-node`](https://github.com/RadiantBlockchain/radiant-node) — C++ consensus node

### AI Context Priming

To bring an AI up to speed on this project quickly, paste (in order):

1. `radiant-ledger-guide/README.md` (this repo)
2. `radiant-ledger-app/README.md` (overview)
3. `radiant-ledger-app/INVESTIGATION.md` (per-phase narrative)
4. The specific `docs/solutions/` entry relevant to your current problem

For minting work, use [`radiant-glyph-guide`](https://github.com/Zyrtnin-org/radiant-glyph-guide) instead.

---

## Acknowledgments

Built with substantial AI assistance from Anthropic's Claude (various models across the session, primarily Opus 4.6 and Sonnet 4.6). The "compound engineering" workflow pattern — `/workflows:brainstorm`, `/workflows:plan`, `/workflows:work`, `/workflows:compound` — materially accelerated the arc from scoping to mainnet acceptance.

Radiant's protocol team ([`RadiantBlockchain`](https://github.com/RadiantBlockchain)) maintains the canonical consensus node and JS reference implementation. This Ledger app would not exist without their open source work.

Ledger's `app-bitcoin` fork architecture made this feasible — adding a new coin variant is well-documented and the build infrastructure is reproducible.
