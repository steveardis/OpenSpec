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

## Classification Insight

Source: External conversation analyzing the extract/trace proposal in depth.

### The Core Problem

Not all extracted behavior should become requirements for a new system. Extracted specs describe what *exists*, but some of that behavior is:
- Platform workarounds being left behind
- Compensating logic for bugs being fixed
- Features being deliberately dropped

Without classification, the spec corpus becomes "spec spaghetti" — a web of behavioral requirements where nobody can tell which reflect genuine business need vs. artifacts of the old platform's limitations.

### Classification Taxonomy

Every extracted behavior gets tagged:

| Classification | Description | On Rewrite |
|---------------|-------------|------------|
| **BUSINESS** | Genuine domain rules ("orders over $10K require manager approval") | Hard requirement — must implement |
| **PLATFORM** | Technology workarounds ("flush entity manager for Hibernate 3.2 bug") | Evaluate — does the new platform need this? |
| **COMPENSATING** | Fixes for other bad code ("re-sort because upstream returns inconsistent order") | Evaluate — is the upstream being fixed? |

### The Key Insight

> "The right framing isn't 'don't spec everything' but rather 'spec everything, then classify each spec as business-essential vs. platform-contingent vs. compensating, and only carry forward the first category as hard requirements for the new system.'"

The classification step is what makes the round-trip code → spec → code produce genuinely better output rather than a polished replica of the old system's quirks.

### Command Taxonomy

The three commands serve distinct purposes but share analysis infrastructure:

| Command | Purpose | Output Orientation |
|---------|---------|-------------------|
| **explore** | "Here's how this works" | Comprehension |
| **extract** | "Here's what this promises" | Normative specs |
| **trace** | "Here's where it came from" | Provenance maps |

Extract and trace should share parsing/analysis infrastructure with explore, but their reasoning layers are fundamentally different. Extract's hard problem is classification; trace's hard problem is mapping stability across refactors.

---

## Storage Architecture: Extracts as a Staging Area

### The Parallel with Changes

Extracts follow the same pattern as changes — they are a **staging area** that feeds into canonical specs:

- **changes/** — proposed behavior (in-flight, authored by humans)
- **extracts/** — discovered behavior (from existing code, generated by AI)
- **specs/** — canonical specifications (source of truth)

Both require a deliberate act to promote into the spec tree.

### Proposed Structure

```
openspec/
├── changes/              # Proposed behavior (in-flight)
│   └── add-feature/
│       ├── proposal.md
│       └── specs/
│
├── extracts/             # Discovered behavior (from existing code)
│   └── order-approval/
│       ├── spec.md           # What we found
│       └── disposition.yaml  # carry-forward / drop / defer + rationale
│
├── specs/                # Canonical specs (source of truth)
│   └── order-processing/
│       └── spec.md
│
└── traces/               # Links: old code ↔ extracts ↔ specs
    └── OrderService.yaml
```

### Disposition as Decision Ledger

The trace map records what was found, how it was classified, and what was decided:

```yaml
# traces/OrderService.yaml
mapped:
  - lines: [42-58]
    extracted_spec: order-approval-threshold
    classification: business
    disposition: carry-forward
    new_spec: order-processing        # the spec it became

  - lines: [61-63]
    extracted_spec: hibernate-flush-workaround
    classification: platform
    disposition: drop
    rationale: "New system uses Prisma, not Hibernate"

  - lines: [70-85]
    extracted_spec: upstream-resort-compensation
    classification: compensating
    disposition: defer
    depends_on: "inventory-service-rewrite"
```

### Why Separation Matters

The original proposal stored extracts in `openspec/specs/` alongside authored specs. This exploration identified the risk: not all extracted behavior maps to new specs. Keeping extracts separate:

- Makes "what are we building" vs. "what did we find" impossible to confuse
- Preserves the audit trail for dropped/deferred behavior
- Allows bulk review of extracted specs before promotion
- Mirrors the changes/ pattern that already exists

### Positioning Shift

Extract + trace together turn OpenSpec from a greenfield development methodology into a **migration tool**. This significantly expands where spec-driven practices can be applied — especially for organizations with large legacy codebases.

---

## Open Design Questions (Remaining)

1. **Qualifying lines** — What counts? Import statements, type definitions, test files, config?
2. **Domain boundary determination** — AI-driven vs. user-guided vs. hybrid?
3. **Trace stability** — How do mappings survive refactors?
4. **Classification automation** — How much can AI classify vs. requiring human review?
5. **Classification location** — Inline in spec.md vs. separate disposition.yaml?
6. **Extract idempotency** — How does re-running extract interact with existing extracts?
7. **Promotion workflow** — What does "carry forward an extract into specs/" look like as a process?
