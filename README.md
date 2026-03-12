# SCAFFOLD-WATCH

> **A non-executing observer agent for Claude Code — watches your primary builder in real-time and fires surgical `/btw` interrupts before mistakes compound.**

**Author:** Insider747 | **Cluster:** Z | **Version:** 1.0.0
**Role:** Shadow / parallel observer — never leads, never executes

---

## What Is It?

SCAFFOLD-WATCH is a Claude Code skill that turns a second AI session into a **silent surveillance layer** running alongside your primary coding agent. It watches the primary's output, pattern-matches against four threat classes, applies an interrupt cost function, and only speaks when the math says it must.

It doesn't write code. It doesn't refactor. It doesn't praise. It catches what the primary won't.

---

## Architecture

```mermaid
graph TD
    A["🔨 Primary Builder Agent<br/>(writes code, runs tools)"] -->|"output stream"| B

    B["👁️ SCAFFOLD-WATCH<br/>(observer — silent by default)"]

    B -->|"URGENCY ≥ 4.0<br/>FIRE IMMEDIATELY"| C["⚡ /btw Interrupt<br/>(delivered to user)"]
    B -->|"URGENCY 2.0–3.9<br/>QUEUE"| D["⏳ Queued Interrupt<br/>(held for natural pause)"]
    B -->|"URGENCY < 2.0<br/>LOG ONLY"| E["📋 Session Log<br/>(appended to end report)"]

    C -->|"user sends /btw"| A
    D -->|"user sends /btw at pause"| A
    E -->|"end-of-session summary"| F["📊 Session Report"]

    style A fill:#1e3a5f,stroke:#4a9eff,color:#fff
    style B fill:#2d1b69,stroke:#a855f7,color:#fff
    style C fill:#7f1d1d,stroke:#ef4444,color:#fff
    style D fill:#713f12,stroke:#f59e0b,color:#fff
    style E fill:#14532d,stroke:#22c55e,color:#fff
    style F fill:#1e3a5f,stroke:#4a9eff,color:#fff
```

---

## The Four Watch Classes

```mermaid
graph LR
    SW["👁️ SCAFFOLD-WATCH"] --> A
    SW --> B
    SW --> C
    SW --> D

    A["🏗️ CLASS-A<br/>Architecture Conflict<br/>─────────────────<br/>Building something<br/>that already exists or<br/>contradicts a pattern"]
    B["🔐 CLASS-B<br/>Security Signal<br/>─────────────────<br/>Injection vectors,<br/>hardcoded secrets,<br/>weak crypto choices"]
    C["♻️ CLASS-C<br/>Redundancy / Thrash<br/>─────────────────<br/>Re-fetching data<br/>already in context,<br/>re-reading files"]
    D["📈 CLASS-D<br/>Scope Drift<br/>─────────────────<br/>Expanding beyond<br/>the task, adding<br/>unspecced features"]

    style SW fill:#2d1b69,stroke:#a855f7,color:#fff,font-size:14px
    style A fill:#1e3a5f,stroke:#4a9eff,color:#fff
    style B fill:#7f1d1d,stroke:#ef4444,color:#fff
    style C fill:#713f12,stroke:#f59e0b,color:#fff
    style D fill:#14532d,stroke:#22c55e,color:#fff
```

---

## Interrupt Cost Function

SCAFFOLD-WATCH never fires on instinct — every interrupt is gated by a cost function:

```mermaid
flowchart TD
    O["Primary agent output received"] --> E["Evaluate against CLASS A–D"]
    E --> R["Score Rework Cost<br/>1 = cosmetic · 3 = multi-file · 5 = restart"]
    R --> P["Estimate P(No Self-Correction)<br/>0.2 = likely catches it · 0.8 = committed to path"]
    P --> U["URGENCY = Rework Cost × P(No Self-Correction)"]

    U --> T1{"URGENCY ≥ 4.0?"}
    T1 -->|"YES"| FIRE["⚡ FIRE IMMEDIATELY<br/>Inject /btw now"]
    T1 -->|"NO"| T2{"URGENCY 2.0–3.9?"}
    T2 -->|"YES"| QUEUE["⏳ QUEUE<br/>Hold for next pause"]
    T2 -->|"NO"| LOG["📋 LOG ONLY<br/>Add to session summary"]

    FIRE --> CAP["Max 3 IMMEDIATE interrupts per session<br/>More than that degrades primary coherence"]

    style O fill:#1e293b,stroke:#64748b,color:#fff
    style E fill:#1e293b,stroke:#64748b,color:#fff
    style R fill:#1e293b,stroke:#64748b,color:#fff
    style P fill:#1e293b,stroke:#64748b,color:#fff
    style U fill:#2d1b69,stroke:#a855f7,color:#fff
    style T1 fill:#1e3a5f,stroke:#4a9eff,color:#fff
    style T2 fill:#1e3a5f,stroke:#4a9eff,color:#fff
    style FIRE fill:#7f1d1d,stroke:#ef4444,color:#fff
    style QUEUE fill:#713f12,stroke:#f59e0b,color:#fff
    style LOG fill:#14532d,stroke:#22c55e,color:#fff
    style CAP fill:#292524,stroke:#78716c,color:#ccc
```

---

## Cluster Z Integration

SCAFFOLD-WATCH is designed to compose with other Cluster Z agents:

```mermaid
graph TD
    USER["👤 User"] -->|"activate"| SW
    SW["👁️ SCAFFOLD-WATCH"] -->|"CLASS-B escalation"| FS["🛡️ FRAUD-SPOT<br/>Adversarial Analysis"]
    SW -->|"drift signal"| MA["🧠 Meta-Architect<br/>Framework Alignment"]
    SW -->|"raw interrupt text"| PF["✒️ PromptForge<br/>Message Sharpening"]

    FS -->|"confirmed risk"| INT["⚡ /btw Interrupt"]
    MA -->|"framework check"| INT
    PF -->|"sharpened message"| INT

    INT -->|"user fires /btw"| PB["🔨 Primary Builder"]

    style USER fill:#1e293b,stroke:#64748b,color:#fff
    style SW fill:#2d1b69,stroke:#a855f7,color:#fff
    style FS fill:#7f1d1d,stroke:#ef4444,color:#fff
    style MA fill:#1e3a5f,stroke:#4a9eff,color:#fff
    style PF fill:#14532d,stroke:#22c55e,color:#fff
    style INT fill:#713f12,stroke:#f59e0b,color:#fff
    style PB fill:#1e3a5f,stroke:#4a9eff,color:#fff
```

---

## Interrupt Message Format

Every interrupt SCAFFOLD-WATCH fires follows this exact structure:

```
⚡ SCAFFOLD-WATCH INTERRUPT [CLASS-X | URGENCY: N.N]

/btw [message — under 40 words, action-oriented]

REASON: [one sentence internal justification — not sent to primary]
HOLD UNTIL: [immediate / next pause / end of session]
```

**Example:**
```
⚡ SCAFFOLD-WATCH INTERRUPT [CLASS-A | URGENCY: 4.0]

/btw Before you build the JWT handler — middleware/auth.js already implements this
at line 47. Wire to that instead, saves ~80 lines and keeps auth logic centralized.

REASON: Primary is about to duplicate existing auth logic, rework cost = 3, self-correction probability = low.
HOLD UNTIL: immediate
```

---

## Installation

### 1. Copy the skill file

Place [`SKILL.md`](.claude/skills/scaffold-watch/SKILL.md) at:

```
~/.claude/skills/scaffold-watch/SKILL.md
```

Or for project-scoped use:

```
<your-project>/.claude/skills/scaffold-watch/SKILL.md
```

### 2. Copy the identity doc

Place [`SCAFFOLD-WATCH.md`](SCAFFOLD-WATCH.md) at:

```
~/.claude/skills/scaffold-watch.md
```

### 3. Activate

Open a **second** Claude Code session alongside your primary builder and run:

```
SCAFFOLD-WATCH: observe and monitor.
```

Then paste the primary agent's output into this session as the build progresses.

---

## Usage Modes

### Manual (two sessions)
1. Run your primary builder in Session 1
2. Copy-paste its output into Session 2 (SCAFFOLD-WATCH)
3. SCAFFOLD-WATCH tells you when and what to `/btw` into Session 1

### Agent Teams (automated)
```
Spawn a SCAFFOLD-WATCH teammate to observe this session and recommend /btw interrupts.
```

### Priming for large codebases
Before entering watch mode, give SCAFFOLD-WATCH context:
```
Here is the current codebase structure and key patterns: [summary]. Now enter watch mode.
```

---

## Personality

SCAFFOLD-WATCH is deliberately minimal:

- **Terse.** Clinical. High signal.
- One sentence of justification, maximum.
- No alternatives, no fixes, no implementation.
- No praise. No commentary on the primary's progress.
- Speaks only when the cost function demands it.

---

## End-of-Session Report

At the end of every build session, SCAFFOLD-WATCH produces:

```
## SCAFFOLD-WATCH SESSION REPORT

Interrupts fired: N
Interrupts queued (not sent): N
Pattern observations:
  - [patterns noticed but not interrupt-worthy]

Recommendations for next session:
  - [architectural suggestions, refactoring candidates, tech debt flagged]

Codebase health delta: [brief assessment]
```

---

## Hard Rules

| Rule | Why |
|------|-----|
| Never write code | Convert fix urges into `/btw` recommendations |
| Never interrupt for style | Naming/formatting isn't your domain |
| Respect the primary's momentum | A bad interrupt costs more than the problem |
| One recommendation at a time | Stacked interrupts degrade primary coherence |
| Max 3 IMMEDIATE interrupts per session | Beyond that you're the problem, not the fix |

---

## Files

```
SCAFFOLD-WATCH/
├── README.md                          ← you are here
├── SCAFFOLD-WATCH.md                  ← agent identity doc
└── .claude/
    └── skills/
        └── scaffold-watch/
            └── SKILL.md               ← full skill definition
```

---

*"Watch. Evaluate. Interrupt only when the cost demands it."*

**Cluster Z** — Insider747
