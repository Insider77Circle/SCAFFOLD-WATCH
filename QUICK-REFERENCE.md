# SCAFFOLD-WATCH Quick Reference Card

## Signal Evaluation Matrix

| Class | Pattern | Confidence Check | Rework Cost | Self-Correct Prob | Action |
|-------|---------|------------------|-------------|-------------------|--------|
| **A1** | Duplicate code exists | `similarity > 0.75` | 3 | Low (0.2-0.4) | IF urgent ≥ 4.0 |
| **A2** | Pattern violation | `contradiction > 0.75` | 4 | Low (0.3-0.5) | IF urgent ≥ 4.0 |
| **A3** | Could be modification | `modify_ratio < 0.3` | 3 | Med (0.5-0.7) | IF urgent ≥ 2.0 |
| **B1** | Credentials exposed | `entropy > 4.0 && in_code` | 2 | Low (0.1) | **ESCALATE** |
| **B2** | Injection vector | `no_validation && dangerous` | 2 | Low (0.2) | **ESCALATE** |
| **B3** | Weak crypto | `algo_strength < 0.5` | 1-2 | Med (0.5) | IF urgent ≥ 4.0 |
| **C1** | Re-fetching data | `in_context && identical` | 1 | Med (0.5-0.7) | LOG |
| **C2** | Rebuilding function | `similarity > 0.7` | 1 | Med (0.5) | LOG |
| **C3** | Circular problem-solving | `repetition > 0.4` | 2 | Low (0.2) | LOG/QUEUE |
| **D1** | Refactoring out-of-scope | `unrelated_ratio > 0.4` | 2 | Med (0.4-0.6) | LOG/QUEUE |
| **D2** | Feature expansion | `unspecced_features ≥ 3` | 3 | Med (0.5-0.7) | LOG/QUEUE |
| **D3** | Exploratory research | `research_turns ≥ 3` | 1 | Med (0.5) | LOG |

## Decision Tree

```
Is it CLASS-B (security)?
  YES → confidence > 0.85?
    YES → FIRE IMMEDIATELY
    NO → QUEUE
  NO → Calculate URGENCY = Cost × Self-Correct Prob
    URGENCY ≥ 4.0?
      YES → Interrupts fired < 3?
        YES → FIRE IMMEDIATELY, increment counter
        NO → QUEUE
      NO → URGENCY ≥ 2.0?
        YES → QUEUE
        NO → LOG ONLY
```

## Red Flag Phrases from Primary

Watch for these in primary's output:

**CLASS-A red flags:**
- "I'll build [X] from scratch"
- "Let me create a new [service/handler/middleware]"
- "I don't see this functionality, so I'll add it"
- (followed by: primary never checked existing code)

**CLASS-B red flags:**
- "For now, I'll hardcode..."
- "The user sends the ID directly in the query"
- "MD5 should be fine for this"
- "I'll use `exec()` to run..."

**CLASS-C red flags:**
- "Let me check that file again..."
- "I need to search for..."
- (something just fetched 2 turns ago)
- "Let me reconsider..."
- (same thing just decided)

**CLASS-D red flags:**
- "While I'm here, I'll clean up..."
- "I should also add..."
- "Let me explore how..."
- (no code, just reading)

## Interrupt Message Template

```
⚡ SCAFFOLD-WATCH INTERRUPT [CLASS-X | URGENCY: N.N]

/btw [action-oriented message, < 40 words, cite file/line if relevant]

REASON: [one sentence, internal logic, don't send to primary]
HOLD UNTIL: [immediate / next pause / end of session]
```

## End-of-Session Report Template

```
## SCAFFOLD-WATCH SESSION REPORT

**Interrupts fired:** N/3
**Interrupts queued (not sent):** N
**Pattern observations:**
  - [notable patterns, even if not interrupt-worthy]
  - [code smells, architectural notes]

**Recommendations for next session:**
  - [refactoring candidates]
  - [tech debt flagged]
  - [architectural improvements]

**Codebase health delta:**
  [Brief: Did this build improve or degrade code quality?]
  [Specific improvements and regressions]
```

## Key Metrics

- **Max Immediate Interrupts per Session**: 3
- **Urgency Immediate Threshold**: ≥ 4.0
- **Urgency Queue Threshold**: 2.0–3.9
- **Urgency Log Threshold**: < 2.0
- **Context Window Size**: ~10 turns
- **B-Class Auto-Escalate**: confidence > 0.85

## Session Rules

1. Track interrupts fired (max 3 immediate per session)
2. Queue signals if interrupt limit reached
3. Log all signals below urgency threshold
4. Fire B-class signals immediately if HIGH confidence
5. Never interrupt for style/formatting alone
6. Respect primary's momentum
7. One interrupt at a time