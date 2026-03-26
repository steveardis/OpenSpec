# Exploration: Extract & Trace Commands

**Source:** [Discussion #739](https://github.com/Fission-AI/OpenSpec/discussions/739)
**Date:** 2026-03-26
**Status:** Exploring

---

## Summary of Discussion #739

Two proposed commands for code-to-specification traceability:

### Extract Command (`/opsx:extract`)
- Analyzes entire repositories to autonomously generate OpenSpec specifications
- AI determines domains, boundaries, and decomposition
- Produces standard `openspec/specs/<domain>/spec.md` files
- Re-runnable: subsequent runs read existing specs and only produce new specs for behavior not already described

### Trace Command (`/opsx:trace`)
- Maps production code lines to existing specifications
- Generates traceability maps in `openspec/specs/<domain>/traces/<target>.yaml`
- Identifies unmapped code as gap reports
- The trace map must balance to zero per target — every qualifying line is either mapped to a spec or listed in the unmapped section

### Use Cases
- Understanding inherited codebases
- Proving compliance and audit coverage
- Retroactively adopting OpenSpec governance
- Detecting code drift outside the OpenSpec workflow
- Supporting modernization efforts with multiple implementation targets

### Community Input
- DivineDominion asked about distinguishing extract from trace, particularly regarding drift detection

---

## Exploration Notes

### Core Concept

```
  EXTRACT: Code ──────────────▶ Specs
  "What does this code do?"

  TRACE:   Code ◀─────────────▶ Specs
  "Which spec covers this code?"
```

### Open Questions

#### 1. What counts as a "qualifying line"?
The trace map must balance to zero — every qualifying line is mapped or listed unmapped. But what qualifies? Import statements? Type definitions? Test files? Configuration? The boundary definition matters for signal vs noise.

#### 2. Extract granularity — who decides domain boundaries?
The proposal says "AI determines domains, boundaries, and decomposition." That's significant autonomy. Domain decomposition is often the hardest architectural question for mature codebases.

**Option A: AI determines domains** — Fully autonomous, fast but potentially misaligned with team's mental model. Risk of refactoring specs after the fact.

**Option B: User guides domains** — Interactive/config-driven, slower but aligned with team's mental model (e.g. "auth/, payments/, notifications/").

#### 3. Trace stability over time
Line-level tracing is inherently fragile. A simple refactor (extract method, rename, reorder) could invalidate huge chunks of the trace map even if behavior hasn't changed. How does this interact with the change workflow? Does every refactor require re-running trace?

#### 4. Relationship between Extract and the existing change workflow
Today, OpenSpec specs are *authored artifacts* — humans (with AI help) write them as part of proposing changes. Extracted specs are *derived artifacts* — reverse-engineered from code. Are they first-class citizens? Can they be modified by the normal change workflow? Or are they "read-only" until a human adopts them?

#### 5. Drift detection (from DivineDominion's comment)
Extract detects "code that doesn't match any spec." Trace detects "specs that don't match any code." Together they form a drift detection system:

```
┌──────────────┐                    ┌──────────────┐
│   Specs      │                    │    Code      │
│              │   ┌────────────┐   │              │
│  spec A ─────┼──▶│   Trace    │◀──┼── file1.ts  │
│  spec B ─────┼──▶│   Map      │◀──┼── file2.ts  │
│  spec C ─────┼──▶│            │   │  file3.ts ──┼──▶ UNMAPPED (drift!)
│              │   └────────────┘   │              │
└──────────────┘                    └──────────────┘
```

### User Journeys

| Journey | Today (OpenSpec) | With Extract + Trace |
|---------|-----------------|---------------------|
| Greenfield | Propose → Spec → Build → Archive | Same |
| Brownfield adoption | Manual spec writing | Extract → Trace → Govern |
| Ongoing governance | Change workflow | Change workflow + Trace validation |
| Compliance/audit | Specs exist as docs | Specs + provable code coverage |

### Scoping Question
Is this one change or two? Extract and Trace feel like they could be independent features with independent value.

### The "re-runnable" claim
Extract being idempotent is crucial but tricky. "Only produce new specs for behavior not already described" requires comparing generated specs against existing ones — needs further exploration of what this means in practice.

---

## Pending: Classification Discussion
A shared Claude conversation about classification was referenced but could not be fetched programmatically. Content to be added when available.
