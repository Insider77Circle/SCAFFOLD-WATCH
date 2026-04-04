# SCAFFOLD-WATCH — Quick Reference Card

> Fast lookup for signal evaluation, decision logic, and interrupt formatting.

---

## Signal Evaluation Matrix

| ID | Class | Pattern | Confidence Check | Rework Cost | Self-Correct Prob | Default Action |
|----|-------|---------|------------------|-------------|-------------------|----------------|
| **A1** | Architecture | Duplicate build | No prior codebase check | 3 | Low (0.2) | URGENCY ≥ 4.0 → FIRE |
| **A2** | Architecture | Pattern contradiction | No awareness shown | 4 | Low (0.3) | URGENCY ≥ 4.0 → FIRE |
| **A3** | Architecture | New build vs. modify | Existing file covers ≥70% | 3 | Med (0.5–0.7) | QUEUE |
| **B1** | Security | Credentials in code | Entropy > 4.0 AND in code | 2 | Low (0.1) | **AUTO-ESCALATE** |
| **B2** | Security | Injection vector | No validation AND dangerous op | 2–4 | Low (0.2) | **AUTO-ESCALATE** |
| **B3** | Security | Weak crypto | Algo in auth context | 1–2 | Med (0.5) | URGENCY ≥ 4.0 → FIRE |
| **C1** | Redundancy | Re-fetching in-context data | File/query already seen < 10 turns | 1 | Med (0.5–0.7) | LOG |
| **C2** | Redundancy | Rebuilding in-session work | Name in functions_built | 1–2 | Med (0.5) | LOG / QUEUE |
| **C3** | Redundancy | Circular problem-solving | Repetition ratio > 0.4 | 1–2 | Low (0.2) | LOG → QUEUE (2nd) |
| **D1** | Drift | Out-of-scope refactoring | Unrelated file ratio > 0.4 | 2 | Med (0.4–0.6) | LOG |
| **D2** | Drift | Feature expansion | ≥ 3 unspecced additions | 3 | Med (0.5–0.7) | QUEUE → FIRE (3rd) |
| **D3** | Drift | Research mid-build | ≥ 3 consecutive no-diff turns | 1 | Med (0.5) | LOG |

---

## Decision Tree

```
Is it CLASS-B (security)?
  YES → confidence > 0.85?
    YES → FIRE IMMEDIATELY (override budget)
    NO  → QUEUE
  NO → Calculate URGENCY = Rework Cost × P(No Self-Correction)
    URGENCY ≥ 4.0?
      YES → interrupt_budget > 0?
        YES → FIRE IMMEDIATELY, budget -= 1
        NO  → QUEUE (by urgency order)
      NO → URGENCY ≥ 2.0?
        YES → QUEUE (fire at next natural pause)
        NO  → LOG ONLY (session report)
```

---

## Urgency Calculation

```
URGENCY = Rework Cost × Self-Correct Probability

Self-Correct Probability values:
  Low  = 0.2   (primary will likely catch this itself)
  Med  = 0.5   (50/50)
  High = 0.8   (primary is committed, won't self-correct)

Example:
  Rework Cost = 3, Self-Correct = High (0.8)
  URGENCY = 3 × 0.8 = 2.4 → QUEUE
```

---

## Red Flag Phrases from Primary

Watch for these in PLAN segments (not code diffs):

**CLASS-A red flags:**
- "I'll build [X] from scratch"
- "Let me create a new [service / handler / middleware / layer]"
- "I don't see this functionality, so I'll add it"
- "I'll implement my own [X]"

**CLASS-B red flags:**
- "For now, I'll hardcode..."
- "The user sends the ID directly in the query"
- "MD5 should be fine for this"
- "I'll use `exec()` / `eval()` to run..."
- "I'll skip validation for now"

**CLASS-C red flags:**
- "Let me check that file again..."
- "I need to search for..." (something fetched < 10 turns ago)
- "Let me reconsider..." (second+ reconsideration of same decision)
- "I think I already did this..." (then does it again)

**CLASS-D red flags:**
- "While I'm here, I'll clean up..."
- "I should also add..."
- "I'll also include support for..."
- "Let me explore how [X] works before..."

---

## Interrupt Message Template

```
⚡ SCAFFOLD-WATCH INTERRUPT [CLASS-X | URGENCY: N.N]

/btw [action-oriented message, < 40 words, cite file/line if relevant]

REASON: [one sentence — internal logic only, do not send to primary]
HOLD UNTIL: [immediate / next pause / end of session]
```

### Example — CLASS-A

```
⚡ SCAFFOLD-WATCH INTERRUPT [CLASS-A | URGENCY: 4.0]

/btw Before you build the JWT handler — middleware/auth.js already implements this
at line 47. Wire to that instead, saves ~80 lines and keeps auth logic centralized.

REASON: Primary building duplicate auth logic, rework cost = 3, no prior code check.
HOLD UNTIL: immediate
```

### Example — CLASS-B

```
⚡ SCAFFOLD-WATCH INTERRUPT [CLASS-B | URGENCY: B1-ESCALATE]

/btw api_key is hardcoded in config.js line 12 — move to .env before this goes any further.

REASON: Credential exposed in source; entropy high, not a placeholder value.
HOLD UNTIL: immediate
```

### Example — CLASS-D

```
⚡ SCAFFOLD-WATCH INTERRUPT [CLASS-D | URGENCY: 2.1]

/btw You're refactoring the DB schema — that's outside the stated task. Finish the
auth endpoint first; flag the schema for a separate session.

REASON: 3 unspecced changes detected; scope drift pattern established.
HOLD UNTIL: next pause
```

---

## End-of-Session Report Template

```
## SCAFFOLD-WATCH SESSION REPORT

**Session turns observed:** N
**Interrupts fired:** N/3
**Interrupts queued (not sent):** N
**Signals logged (below threshold):** N

**Interrupts fired this session:**
  [Turn X] CLASS-A | URGENCY: 4.0 — [brief description]
  [Turn Y] CLASS-B | URGENCY: B1-ESCALATE — [brief description]

**Queued signals (not fired):**
  URGENCY N.N — [brief description]

**Pattern observations:**
  - [notable patterns, even if not interrupt-worthy]
  - [code smells, architectural notes below threshold]

**Recommendations for next session:**
  - [refactoring candidates]
  - [tech debt flagged]
  - [architectural improvements]

**Codebase health delta:**
  [Brief: Did this build improve or degrade code quality?]
  [Specific improvements and regressions]
```

---

## Session Rules (Hard Limits)

| Rule | Value |
|------|-------|
| Max IMMEDIATE interrupts per session | 3 |
| B1/B2 auto-fire threshold | confidence > 0.85 |
| URGENCY → IMMEDIATE | ≥ 4.0 |
| URGENCY → QUEUE | 2.0 – 3.9 |
| URGENCY → LOG | < 2.0 |
| Early-session grace period | Turns 1–3: no C or D signals |
| Self-correction window | Wait 1 full turn before firing |
| Interrupt cooldown | 2 turns between fires (except B1/B2) |
| Context window lookback | ~10 turns for redundancy checks |

---

*"Watch. Evaluate. Interrupt only when the cost demands it."*

**Cluster Z** — Insider747
