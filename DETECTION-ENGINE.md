# SCAFFOLD-WATCH — Detection Engine

> **Version:** 1.1.0 | **Cluster Z** | **Author:** Insider747
>
> This document fully specifies the detection logic SCAFFOLD-WATCH uses to evaluate primary agent output.
> Every signal class, pattern signature, scoring function, and decision rule is defined here.

---

## Table of Contents

1. [Session State Model](#1-session-state-model)
2. [Ingestion Pipeline](#2-ingestion-pipeline)
3. [CLASS-A: Architecture Conflict](#3-class-a-architecture-conflict)
4. [CLASS-B: Security / Quality Signal](#4-class-b-security--quality-signal)
5. [CLASS-C: Redundancy / Context Thrash](#5-class-c-redundancy--context-thrash)
6. [CLASS-D: Scope Creep / Drift](#6-class-d-scope-creep--drift)
7. [Urgency Scoring & Dispatch](#7-urgency-scoring--dispatch)
8. [Interrupt Budget & Throttle Rules](#8-interrupt-budget--throttle-rules)
9. [End-of-Session Report](#9-end-of-session-report)
10. [Edge Cases & Override Rules](#10-edge-cases--override-rules)

---

## 1. Session State Model

SCAFFOLD-WATCH maintains the following state across the entire build session.
All state is accumulated from turn 0 and never reset mid-session.

```
SESSION STATE
├── turn_count              (int)   — total turns observed
├── interrupt_budget        (int)   — starts at 3, decrements on each IMMEDIATE fire
├── interrupts_fired        [list]  — CLASS, URGENCY, turn number of each fired interrupt
├── interrupts_queued       [list]  — same structure, for QUEUED signals
├── signals_logged          [list]  — all LOG ONLY signals, for end-of-session report
│
├── context_registry        {dict}  — files, functions, data already in primary's context
│   ├── files_read          [list of file paths seen in tool calls]
│   ├── functions_built     [list of function names created this session]
│   ├── data_fetched        [list of queries / fetch targets seen]
│   └── decisions_made      [list of architectural decisions observed]
│
├── scope_baseline          {dict}  — captured from first 1-2 turns
│   ├── original_task       (str)   — stated goal of the build
│   └── stated_constraints  [list]  — explicit constraints from the user
│
└── red_flags_seen          [list]  — raw red-flag phrase matches with turn + text
```

**Updating state:** After processing each turn of primary output, update all relevant
state fields before evaluating signals. State must be current before scoring begins.

---

## 2. Ingestion Pipeline

Every chunk of primary output goes through this pipeline in order:

```
PRIMARY OUTPUT (chunk)
        │
        ▼
[STEP 1] TOKENIZE
  — Split into: plan text, tool calls, code diffs, error output, todo updates
  — Tag each segment by type: PLAN | TOOL | DIFF | ERROR | TODO

        │
        ▼
[STEP 2] CONTEXT REGISTRY UPDATE
  — Extract any new files read → add to files_read
  — Extract any new functions/classes created → add to functions_built
  — Extract any data fetches (grep, search, API calls) → add to data_fetched
  — Capture stated architectural decisions → add to decisions_made

        │
        ▼
[STEP 3] RED FLAG SCAN
  — Run all 16 red-flag phrases against PLAN segments only (not code)
  — Log any matches to red_flags_seen with turn + phrase + context sentence

        │
        ▼
[STEP 4] CLASS EVALUATION (A → B → C → D)
  — Evaluate each class independently
  — Each class produces: { signal: bool, subclass: str, confidence: float,
                           rework_cost: int, self_correct_prob: float }

        │
        ▼
[STEP 5] URGENCY CALCULATION & DISPATCH
  — Calculate URGENCY for each active signal
  — Apply dispatch rules (FIRE / QUEUE / LOG)
  — Apply interrupt budget throttle
  — Emit interrupt or silence
```

---

## 3. CLASS-A: Architecture Conflict

**Definition:** The primary is about to build something that already exists, contradicts
an established pattern, or will require significant multi-file rework to integrate.

---

### A1 — Duplicate Build

**Detection trigger:** Primary signals intent to build functionality that already exists
in the codebase or was already built this session.

**Pattern signatures:**

| Signal | Lexical indicators in PLAN segments |
|--------|-------------------------------------|
| Intent to build from scratch | "I'll build", "I'll create", "I'll implement", "let me write", "I'll add a new" |
| Preceded by no codebase check | No prior Glob/Grep/Read tool call for similar functionality |
| Codebase already contains similar name | `functions_built` or file names overlap with what's being planned |

**Context validator:**
- CONFIRM: Did the primary search for existing implementations before announcing the build?
  - If YES → self-correction probability rises to Med (0.5); downgrade urgency
  - If NO → self-correction probability = Low (0.2); full urgency applies
- CONFIRM: Is the overlap substantial (>75% functional similarity) or superficial (same name, different purpose)?
  - Superficial overlap → downgrade to LOG ONLY

**Scoring:**
```
Rework Cost:
  Exact duplicate of in-session work    → 2  (can just point to it)
  Duplicate of existing codebase code   → 3  (multi-file integration needed)
  Contradicts established architecture  → 4  (architectural change required)

Self-Correct Probability:
  Primary checked existing code         → Med (0.5)
  Primary did not check                 → Low (0.2)
  Primary explicitly said "from scratch"→ Low (0.1)
```

**Special rule:** If primary is duplicating work it did *earlier this session*
(i.e., something in `functions_built`), urgency floor = 2.0 regardless of
self-correction probability. The primary should have this in context.

---

### A2 — Pattern Contradiction

**Detection trigger:** The primary is building something that directly contradicts
an established pattern, framework choice, or architectural decision.

**Pattern signatures:**

| Signal | Lexical indicators |
|--------|-------------------|
| New abstraction layer | "I'll create a new [service/layer/module/abstraction]" when one exists |
| Bypassing existing framework | "I'll use X directly" when a wrapper/ORM/middleware handles X |
| Inconsistent pattern | New code structure differs from existing similar structures in the project |

**Context validator:**
- CONFIRM: Is the contradiction intentional (refactor)? Check if `scope_baseline.original_task` mentions refactoring or replacing the existing pattern.
  - If YES → not a conflict, downgrade to LOG
- CONFIRM: Has the user or primary explicitly stated they're changing the pattern?
  - If YES → downgrade to LOG

**Scoring:**
```
Rework Cost:
  New layer that should wire to existing  → 3
  Bypass of established wrapper/ORM       → 4
  Full architectural contradiction        → 5

Self-Correct Probability:
  Primary likely aware of pattern         → Med (0.5)
  Primary shows no awareness              → Low (0.3)
```

---

### A3 — Modification Disguised as New Build

**Detection trigger:** The primary is creating a new file/function when the correct
action is to modify an existing one.

**Pattern signatures:**

| Signal | Indicators |
|--------|-----------|
| Creates new rather than extends | "I'll create a new version", "I'll add a separate handler for" |
| New file for narrow edge case | Small new file added for a case clearly handled by existing logic |
| Parallel implementation | Two similar implementations exist post-build |

**Context validator:**
- CONFIRM: Is the new code truly independent? Check if it duplicates control flow or data handling already present.
- CONFIRM: Is there an existing file that covers ≥70% of the same concern?

**Scoring:**
```
Rework Cost:         3  (refactoring and consolidation required later)
Self-Correct Prob:   Med (0.5–0.7)  — primary often catches this itself
```

**Note:** A3 rarely fires IMMEDIATELY unless rework cost is high. Default to QUEUE.

---

## 4. CLASS-B: Security / Quality Signal

**Definition:** The primary's approach introduces a known vulnerability class, weak
pattern, or quality debt that will be expensive or dangerous to fix post-build.

**CLASS-B is the highest-priority watch class. B1 and B2 skip the urgency formula
entirely — if confidence > 0.85, FIRE IMMEDIATELY regardless of budget.**

---

### B1 — Credential / Secret Exposure

**Detection trigger:** Credentials, API keys, tokens, or secrets are being written
directly into code, config files, or committed artifacts.

**Pattern signatures:**

| Signal | Indicators |
|--------|-----------|
| Hardcoded secret | "I'll hardcode", "for now I'll use the key directly", key/token/password appearing as string literal in code diff |
| Secret in non-.env file | Credential string in `.js`, `.py`, `.ts`, `.json` (non-secret-manager files) |
| Inline credential | `password = "..."`, `api_key = "..."`, `token = "..."` in DIFF segments |
| High entropy string | String literal with entropy > 4.0 bits/char and length > 16 chars in code |

**Context validator:**
- CONFIRM: Is the credential a placeholder/example value (e.g., `"your_api_key_here"`, `"REPLACE_ME"`)?
  - If YES → confidence drops to 0.3, downgrade to LOG
- CONFIRM: Is it in a test file explicitly for testing with mock values?
  - If YES → downgrade to LOG
- CONFIRM: Is it being written to `.env` or a secrets manager?
  - If YES → not a violation, do not fire

**Scoring:**
```
Confidence threshold for IMMEDIATE fire:  > 0.85
Rework Cost:   2  (key rotation + audit required)
Self-Correct:  Low (0.1)  — primary almost never self-corrects on this
```

**Interrupt text guidance:** Name the specific variable or line. "Credential in [file]
at [context] — move to .env before committing."

---

### B2 — Injection Vector

**Detection trigger:** User-controlled input flows into a dangerous operation without
validation or parameterization.

**Pattern signatures:**

| Signal | Indicators |
|--------|-----------|
| SQL string interpolation | f-string or string concat containing user variable flowing into SQL query |
| Unsanitized shell execution | `exec()`, `subprocess` with string concat, `os.system(f"...")`, `eval()` |
| Unsanitized HTML output | Template rendering user input without escaping |
| Missing input validation | "The user sends X directly", route handler receives param → uses immediately |

**Context validator:**
- CONFIRM: Is there a validation/sanitization step between input and dangerous operation?
  - If YES → not a violation
- CONFIRM: Is the input from a trusted internal source (not user-facing)?
  - If YES → downgrade confidence to 0.5, apply urgency formula
- CONFIRM: Is an ORM/parameterized query used (e.g., SQLAlchemy, knex, prepared statements)?
  - If YES → not a violation

**Scoring:**
```
Confidence threshold for IMMEDIATE fire:  > 0.85
Rework Cost:   2–4 depending on scope (2 = single endpoint, 4 = systemic pattern)
Self-Correct:  Low (0.2)
```

---

### B3 — Weak Cryptography

**Detection trigger:** The primary selects a cryptographic algorithm known to be
insufficient for the context.

**Pattern signatures:**

| Signal | Indicators |
|--------|-----------|
| MD5 or SHA1 for passwords | `md5`, `sha1`, `hashlib.md5`, `hashlib.sha1` in password/auth context |
| No salt on hash | `hash(password)` without salt parameter |
| ECB mode encryption | `AES.MODE_ECB`, `Cipher.ECB` |
| Weak random for security | `random.random()`, `Math.random()` used for tokens/sessions/passwords |
| Short key length | RSA < 2048, AES < 128 |

**Context validator:**
- CONFIRM: Is this for passwords/auth/tokens (high risk) vs. checksums/fingerprints (acceptable)?
  - Checksum use of MD5 → downgrade to LOG
- CONFIRM: Is the primary aware of the weakness and choosing it intentionally with comment?
  - If YES → downgrade, but still QUEUE if auth-context

**Scoring:**
```
Rework Cost:   1–2  (usually isolated, but pervasive if used in auth layer)
Self-Correct:  Med (0.5)  — primary sometimes catches this
Urgency formula applies (not auto-escalate like B1/B2)
```

---

## 5. CLASS-C: Redundancy / Context Thrash

**Definition:** The primary is repeating work it has already done or re-fetching data
that is already in its context window. This wastes tokens and signals context confusion.

**CLASS-C signals rarely justify IMMEDIATE interrupts. Default disposition: LOG or QUEUE.**

---

### C1 — Re-fetching In-Context Data

**Detection trigger:** The primary issues a tool call (Read, Grep, Glob, Bash) for
something it already retrieved within the last ~10 turns.

**Pattern signatures:**
- File path in a new Read call matches a path already in `context_registry.files_read`
- Grep/search query semantically identical to one already executed (same pattern, same path)
- API call or data fetch for the same resource already pulled this session

**Context validator:**
- CONFIRM: Was the prior fetch more than 10 turns ago? If so, re-fetching may be legitimate (context compression).
  - If > 10 turns → downgrade to LOG ONLY
- CONFIRM: Is the primary re-reading because the file was modified since last read?
  - If YES → not a redundancy, do not fire

**Scoring:**
```
Rework Cost:   1  (no code affected, just token waste)
Self-Correct:  Med (0.5–0.7)
Urgency:       Rarely exceeds 1.0 — almost always LOG ONLY
```

---

### C2 — Rebuilding In-Session Work

**Detection trigger:** The primary begins implementing a function, class, or component
it already created earlier in the current session.

**Pattern signatures:**
- Function/class name in PLAN or DIFF appears in `context_registry.functions_built`
- Primary states intent to "rewrite", "redo", or "replace" something it just built
- New DIFF nearly identical (>70% overlap) to a DIFF seen earlier this session

**Context validator:**
- CONFIRM: Is this an intentional iteration/fix? Primary explicitly says "I need to revise [X]"?
  - If YES → not a redundancy; the primary is aware
- CONFIRM: Is the rebuild substantially different (new interface, different approach)?
  - If YES → may be CLASS-A (pattern contradiction) instead; re-evaluate

**Scoring:**
```
Rework Cost:   1–2
Self-Correct:  Med (0.5)
Urgency:       Usually 1.0–2.0 → LOG or QUEUE
```

---

### C3 — Circular Problem-Solving

**Detection trigger:** The primary is revisiting a decision, approach, or file it
already resolved — not to fix it, but because it lost track of what it decided.

**Pattern signatures:**
- "Let me reconsider...", "Actually, let me rethink...", "I'm not sure about..."
- Primary proposes approach X, then re-evaluates X within same segment
- Same architectural question raised 2+ times across session with no new information
- Repetition ratio: ≥40% of current plan text rehashes prior plan text

**Context validator:**
- CONFIRM: Is the reconsideration driven by new information (error output, new requirement)?
  - If YES → legitimate; do not fire
- CONFIRM: Is this the second or third time this specific question has been re-raised?
  - Second time → QUEUE; Third time → FIRE (rework cost escalates to 2)

**Scoring:**
```
Rework Cost:   1 (first recurrence) → 2 (second recurrence)
Self-Correct:  Low (0.2) once circular pattern is established
Urgency:       0.5 first time → 1.6 second → fire threshold approached by third
```

---

## 6. CLASS-D: Scope Creep / Drift

**Definition:** The primary is expanding scope beyond the original task in ways that
will increase session length and token cost without improving the stated output.

---

### D1 — Out-of-Scope Refactoring

**Detection trigger:** The primary begins modifying or cleaning up code that is not
related to the current task objective.

**Pattern signatures:**
- "While I'm here, I'll clean up...", "I'll also refactor...", "Let me tidy this up"
- Diff touches files not referenced in `scope_baseline.original_task`
- Changes are structural/stylistic rather than functional

**Context validator:**
- CONFIRM: Is the refactor necessary for the primary task to function correctly?
  - If YES → not scope drift; it's a prerequisite
- CONFIRM: Did the user explicitly ask for cleanup alongside the main task?
  - If YES → not scope drift

**Scoring:**
```
Rework Cost:   2  (side-refactors often introduce regressions or need reverting)
Self-Correct:  Med (0.4–0.6)
Urgency:       Typically 0.8–1.2 → LOG ONLY unless unrelated_ratio > 0.6
```

---

### D2 — Feature Expansion

**Detection trigger:** The primary is adding features, options, or capabilities that
were not in the original task specification.

**Pattern signatures:**
- "I'll also add...", "I should also include...", "Let me add support for..."
- New functionality appears in DIFF or PLAN with no corresponding user requirement
- Feature counter: ≥3 unspecified additions in the current session

**Context validator:**
- CONFIRM: Are the additions genuinely unspecified, or reasonable implementation details?
  - Implementation details (error handling for the specified feature, required dependencies) → not scope drift
  - New user-visible features, new config options, new endpoints → scope drift
- CONFIRM: Did the user say "while you're at it" or similar?
  - If YES → not scope drift

**Scoring:**
```
Rework Cost:   3  (unspecced features often need rework or removal at review)
Self-Correct:  Med (0.5–0.7)
Urgency:       1.5–2.1 → QUEUE once threshold crossed
```

**Note:** Fire at first feature addition ≥ unspecced (QUEUE it). Fire IMMEDIATELY if
a third distinct unspecced feature is detected in the same session — at that point,
scope is clearly off-track (urgency = 3.0 minimum).

---

### D3 — Exploratory Research Mid-Build

**Detection trigger:** The primary has paused implementation to explore, investigate,
or research something that could be decided later or already known.

**Pattern signatures:**
- "Let me explore how...", "Let me research...", "I want to understand how X works before..."
- ≥3 consecutive Read/Grep/Search tool calls with no DIFF output between them
- Primary reads documentation, checks multiple options, without acting on information
- `research_turns` counter ≥ 3 (consecutive non-building turns)

**Context validator:**
- CONFIRM: Is this the start of the session (priming phase)? Research is expected in turns 1-3.
  - If in turns 1-3 → do not fire, this is normal context-gathering
- CONFIRM: Is the research directly unblocking a specific implementation decision?
  - If YES → downgrade to LOG; necessary exploration
- CONFIRM: Has the research been going on for ≥3 turns without producing a diff?
  - If YES → QUEUE a scope interrupt

**Scoring:**
```
Rework Cost:   1  (no code harm, just token/time cost)
Self-Correct:  Med (0.5)
Urgency:       0.5–1.5 → typically LOG, QUEUE if > 3 research turns
```

---

## 7. Urgency Scoring & Dispatch

### Core Formula

```
URGENCY = Rework Cost × Self-Correct Probability (inverted)

Where:
  Self-Correct Probability is P(primary does NOT self-correct)
  Low    = 0.2  (primary likely catches it)
  Med    = 0.5  (50/50)
  High   = 0.8  (primary is committed, won't catch it)
```

### Dispatch Thresholds

| URGENCY | Action | Condition |
|---------|--------|-----------|
| ≥ 4.0 | **FIRE IMMEDIATELY** | Interrupt budget > 0 |
| ≥ 4.0 | **QUEUE** | Interrupt budget = 0 |
| 2.0 – 3.9 | **QUEUE** | Hold for next natural pause |
| < 2.0 | **LOG ONLY** | Append to session report |

### CLASS-B Override

B1 and B2 signals with confidence > 0.85 bypass the urgency formula entirely:

```
IF class == B1 OR B2
  AND confidence > 0.85
  → FIRE IMMEDIATELY (budget consumed regardless)
```

This is the only case where budget can go negative. Security signals always fire.

### Natural Pause Detection

A "natural pause" is any of:
- Primary produces a complete diff and enters a new planning segment
- Primary explicitly states it's done with a sub-task ("That completes X")
- Primary awaits tool output or runs tests
- End of a response chunk

Queued interrupts fire at the next detected natural pause.

---

## 8. Interrupt Budget & Throttle Rules

```
budget = 3  (starts at session open)

On each IMMEDIATE fire:
  budget -= 1
  log to interrupts_fired

When budget == 0:
  All new URGENCY ≥ 4.0 signals → QUEUE (not dropped)
  B1/B2 security signals → FIRE anyway (budget goes to -1, then -2, etc.)
  Log a note: "Budget exhausted — N signals queued"

Queued signals fire in URGENCY order, not insertion order:
  Highest urgency → lowest urgency

At session end: all queued signals are included in session report.
```

**Rationale:** The 3-interrupt cap exists because each interrupt breaks the primary's
context. Beyond 3, the observer becomes the biggest source of degradation.
Security signals override this — a hardcoded credential is always worth the interruption.

---

## 9. End-of-Session Report

Produce this report when the primary's build task completes, signals it's done,
or the user explicitly requests the session summary.

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
  - [Notable patterns across the session, even if not interrupt-worthy]
  - [Code smells or architectural notes below threshold]

**Recommendations for next session:**
  - [Refactoring candidates flagged during this session]
  - [Tech debt introduced but not interrupt-worthy]
  - [Architectural improvements to consider]

**Codebase health delta:**
  [One paragraph: did this build improve or degrade code health?
   List specific improvements and regressions observed.]
```

---

## 10. Edge Cases & Override Rules

### Multi-signal turns
If a single turn of primary output triggers signals in more than one class,
evaluate each independently and fire only the highest-urgency signal.
Exception: if B1/B2 is one of the signals, fire B-class first regardless.

### Early-session grace period (turns 1–3)
Do not fire CLASS-C or CLASS-D signals in the first 3 turns.
Context-gathering and scoping activity is expected and legitimate.

### Explicit user override
If the user has explicitly told the primary to do something that would normally
trigger a signal (e.g., "rewrite the auth from scratch"), suppress that signal class
for that specific action. Log it as "user-directed: not signaled."

### Test file exception
CLASS-B signals (B1 hardcoded secrets, B2 injection) are suppressed if the
affected code is exclusively in test files (`test_*.py`, `*.test.ts`, `spec/`, etc.)
and the values are clearly mock/fixture values.

### Self-correction window
After a signal is detected, wait one full turn before firing.
If the primary self-corrects within that turn (catches the issue itself),
log it as "self-corrected" and do not fire. This prevents wasted interrupts.

### Interrupt cooldown
After firing an interrupt, do not fire another for at least 2 turns,
regardless of urgency. Stacked rapid interrupts degrade primary coherence.
Exception: B1/B2 security signals override the cooldown.

---

*"The detection engine exists to make silence meaningful. When SCAFFOLD-WATCH says nothing, it means nothing is wrong — not that it wasn't watching."*

**Cluster Z** — Insider747
