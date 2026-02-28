# MIS Architecture Report: Card Manufacturing System
## From First Principles to CQRS-Ready — A Complete Guide

*Prepared for: Printing/Card Manufacturing MIS*
*Date: February 2026*

---

## TABLE OF CONTENTS

1. [The Core Problem, Stated Precisely](#1-the-core-problem-stated-precisely)
2. [First Principles: What Is This System Actually Doing?](#2-first-principles-what-is-this-system-actually-doing)
3. [Domain Modeling Before Everything](#3-domain-modeling-before-everything)
4. [The Fan-Out / Fan-In Problem](#4-the-fan-out--fan-in-problem)
5. [Event-Driven Thinking Without Infrastructure](#5-event-driven-thinking-without-infrastructure)
6. [CQRS: Origins, Theory, and What It Actually Solves](#6-cqrs-origins-theory-and-what-it-actually-solves)
7. [The Progression Path: CRUD → CQRS-Ready → CQRS → Event Sourcing](#7-the-progression-path-crud--cqrs-ready--cqrs--event-sourcing)
8. [When to Introduce CQRS (Specific Signals)](#8-when-to-introduce-cqrs-specific-signals)
9. [How to Build CQRS-Ready Without CQRS](#9-how-to-build-cqrs-ready-without-cqrs)
10. [Projections: The Mathematics of Read Models](#10-projections-the-mathematics-of-read-models)
11. [Process Manager vs Saga: Which One You Need](#11-process-manager-vs-saga-which-one-you-need)
12. [Manufacturing Genealogy and Serialization: Industry Precedent](#12-manufacturing-genealogy-and-serialization-industry-precedent)
13. [Event Modeling: Designing the System Before Writing Code](#13-event-modeling-designing-the-system-before-writing-code)
14. [Capacity Planning: The Data Model](#14-capacity-planning-the-data-model)
15. [Build Order: First Principles to Production](#15-build-order-first-principles-to-production)
16. [The AI Layer: Why This Foundation Matters](#16-the-ai-layer-why-this-foundation-matters)
17. [Common Mistakes to Avoid](#17-common-mistakes-to-avoid)
18. [Summary: The Master Sequence](#18-summary-the-master-sequence)

---

## 1. The Core Problem, Stated Precisely

Your manufacturing process has a property that breaks most off-the-shelf systems:
**a single tracked object transforms into N individually-tracked objects, each of which must later reconverge into an exact, validated set.**

Most MIS/ERP systems track at the **order** or **batch** level. They answer "how many cards were completed?" They cannot answer "where is card #7 of job J-1044 right now, what operations has it been through, does it have a serial number, and has it been matched to its complement cards for packing?"

This is not a software feature gap. It is a **data modeling gap**. The systems don't have the concept of an individually-tracked sub-unit that:
- Derives its identity from a parent (the sheet)
- Has its own lifecycle independent of that parent after the cut
- Must later prove membership in a specific set

The semiconductor industry solved this first — a wafer becomes hundreds of individual dies, each serialized and tracked individually through packaging and test. The pharmaceutical industry solved it second — a batch of ingredients becomes thousands of individually-tracked pills. You are solving the same class of problem for trading cards.

This report is about how to model this precisely, build the foundation correctly, and evolve the architecture to support it at scale.

---

## 2. First Principles: What Is This System Actually Doing?

Before touching code, define the system's job in mathematical terms.

### The System Tracks State Changes of Physical Objects Over Time

That's it. Everything else is derived from that.

A physical object (sheet, card) has:
- **Identity** — it can be uniquely distinguished from all other objects
- **State** — a set of properties at a given moment
- **History** — an ordered sequence of state transitions
- **Lineage** — its relationship to the objects it came from

A production operation is a **function** that takes one or more objects, applies work, and returns:
- The same objects in a new state, OR
- A set of new objects (fan-out), OR
- A single new object (fan-in / assembly)

This is the mathematical core. Your software is recording the **application of these functions over time**, with enough fidelity to answer any question about any object at any point.

### The Business's Real Questions

Your system must answer two classes of questions:

**Forward-looking (capacity planning):**
- "How much work is queued at station X?"
- "Can we commit job J-1044 for delivery on date D?"
- "Which jobs are at risk of missing their deadline?"

**Backward-looking (tracking and traceability):**
- "Where is this specific card right now?"
- "Has every card in set S passed QA?"
- "Show me the complete history of this card from sheet to pack"

These two question types have fundamentally different data shapes. This observation is the seed of CQRS — but we don't need to act on it yet.

---

## 3. Domain Modeling Before Everything

The most important work you can do is **model the domain purely, without any persistence, framework, or infrastructure concern**. This is not an intermediate step. This is the foundation that determines everything else's quality.

### The Ubiquitous Language

Before writing a single class, establish the vocabulary. Every term should mean exactly one thing:

| Term | Definition |
|------|-----------|
| **ProductionOrder** | A commitment to produce a specific quantity of a specific product for a specific customer |
| **Job** | The work unit created to fulfill a ProductionOrder (one Order may spawn multiple Jobs) |
| **Sheet** | A physical substrate (paper) that enters the press. The unit of pre-cut work. |
| **Cut** | The operation that destroys a Sheet as a tracked entity and creates N CardUnits |
| **CardUnit** | An individual card. Born at Cut, dies when Packed. Has its own lifecycle. |
| **Operation** | A discrete unit of work performed on a Sheet or CardUnit at a Station |
| **Station** | A physical location where an operation type is performed (e.g., FOIL-1, QA-BENCH-3) |
| **Embellishment** | Any decorative operation: foil stamping, UV coating, die cutting, etc. |
| **Memorabilia** | A physical insert placed into a card (autograph, relic, etc.) |
| **Set** | The N CardUnits that originated from the same Sheet and must be reunited for packing |
| **Assembly** | The gathering of all N cards in a Set, validated as complete before packing |
| **Pack** | A physical shipment container (box, case, pallet) |
| **QA** | Quality inspection of a CardUnit. Result is pass or fail with reason. |

Getting this language right means your code reads like the business. When a developer reads `sheet.cut(n_up=18)` and gets back `List[CardUnit]`, they understand the domain without needing comments.

### Value Objects vs Entities

This distinction from Domain-Driven Design is fundamental:

**Entities** have identity. Two entities with identical properties are still different things if they have different IDs. A CardUnit is an entity — card #7 and card #8 from the same sheet are different things even if they look identical.

**Value Objects** have no identity — they are defined entirely by their values. Two value objects with the same values are interchangeable. An `Embellishment` specification (type: "foil", color: "gold", die: "wave") is a value object — it doesn't need its own ID.

For your system:
- **Entities**: ProductionOrder, Job, Sheet, CardUnit, Pack, Station
- **Value Objects**: EmbellishmentSpec, QAResult, SequentialNumber, Address, Dimensions

```python
# Entity — identity-based equality
@dataclass
class CardUnit:
    card_unit_id: CardUnitId
    sheet_id: SheetId
    position: int           # position on sheet (1-18)
    job_id: JobId
    status: CardUnitStatus
    # ... operations history stored separately as events

    def __eq__(self, other):
        return isinstance(other, CardUnit) and self.card_unit_id == other.card_unit_id

# Value Object — value-based equality
@dataclass(frozen=True)
class EmbellishmentSpec:
    embellishment_type: str    # "foil" | "uv" | "die_cut" | "glue"
    color: str | None
    die_name: str | None
    position: str | None       # "front" | "back"
    # No ID. Immutable. Two identical specs are the same thing.
```

### Aggregate Roots and Invariants

An **aggregate** is a cluster of objects that must change together to maintain consistency. The **aggregate root** controls all access.

The critical design question for your system: **Is the Sheet the aggregate root, or is each CardUnit its own aggregate?**

The answer has major implications for how you model the cut operation and how you handle concurrent updates.

**Option A: Sheet as root, CardUnits as children**
```
Sheet (aggregate root)
  └── List[CardUnit]
```
This makes the cut operation clean (it's a method on Sheet that spawns children), and invariants like "all cards come from a valid sheet" are easy to enforce. But it means every operation on a single CardUnit requires loading the whole Sheet aggregate, and concurrent operations on different cards from the same sheet can conflict.

**Option B: CardUnit as its own aggregate after cut**
```
Sheet (aggregate root, pre-cut)
    ↓ cut event
CardUnit (new aggregate root, post-cut)
```
This is the right answer for your domain. After the cut, CardUnits need to be independently updateable — one card is at QA while another is at the foil stamp station. They can't be locked to each other. Sheet remains the aggregate for pre-cut operations. At the cut, it fires a domain event that spawns N new CardUnit aggregates.

**The invariant that bridges them**: a `Set` or `Assembly` aggregate is responsible for enforcing that all CardUnits are present and accounted for before packing. This aggregate holds the set membership rules and validates completeness.

```python
@dataclass
class Assembly:
    assembly_id: AssemblyId
    job_id: JobId
    sheet_id: SheetId          # which sheet these cards came from
    expected_count: int         # 18 (or N)
    gathered_card_ids: List[CardUnitId]
    status: AssemblyStatus     # "in_progress" | "complete" | "error"

    def gather(self, card_unit: CardUnit) -> 'Assembly':
        """Pure function. Returns new Assembly state. Raises if wrong card."""
        if card_unit.sheet_id != self.sheet_id:
            raise WrongCardForAssembly(card_unit.card_unit_id, self.assembly_id)
        if card_unit.card_unit_id in self.gathered_card_ids:
            raise DuplicateCardInAssembly(card_unit.card_unit_id)
        new_gathered = self.gathered_card_ids + [card_unit.card_unit_id]
        new_status = AssemblyStatus.COMPLETE if len(new_gathered) == self.expected_count else AssemblyStatus.IN_PROGRESS
        return Assembly(
            assembly_id=self.assembly_id,
            job_id=self.job_id,
            sheet_id=self.sheet_id,
            expected_count=self.expected_count,
            gathered_card_ids=new_gathered,
            status=new_status
        )
```

Notice: pure function, no I/O, no database, returns new state. This is the right model.

---

## 4. The Fan-Out / Fan-In Problem

This is the architectural heart of your system. Most resources don't address it directly, so let's be precise.

### Fan-Out: One Object Becomes Many

The cut operation is a **destructive transformation** — the Sheet as a tracked entity ceases to exist, and N CardUnits come into existence. This is not just an update; it is a lifecycle event that creates new aggregate roots.

```
Before Cut:           After Cut:
Sheet S-001           CardUnit S-001-01
                      CardUnit S-001-02
                      CardUnit S-001-03
                      ...
                      CardUnit S-001-18
```

Key design decisions:
1. **Sheet S-001 is not deleted** — it is archived. Its history is preserved. The cut is an event on the Sheet.
2. **CardUnit IDs encode lineage** — `S-001-07` tells you immediately it is position 7 from Sheet S-001. This is deterministic identity, not a UUID.
3. **All 18 CardUnits inherit Sheet metadata at creation** — job ID, product spec, embellishment specs, customer — so each CardUnit is self-describing without joins.

### Fan-In: Many Objects Must Become One Set

The assembly operation is a **convergence** — N separate CardUnit lifecycles must merge into a single, validated Set. This is where the zero-error requirement lives.

The Assembly aggregate is responsible for:
1. Knowing exactly which cards belong (from the Sheet's lineage)
2. Tracking which have arrived
3. Refusing to accept cards that don't belong
4. Refusing to complete until all expected cards are present
5. Detecting and alerting on cards that are absent past a timeout

This is fundamentally a **Process Manager** (see section 11) — it tracks the state of a multi-actor, long-running process and enforces completion rules.

### The Tricky Part: What Happens When a Card Is Lost or Damaged?

Your system must have a policy for:
- A card fails QA and cannot be repaired → the set is incomplete
- A card is lost → the set is incomplete
- A replacement card is produced → it must be linked to the original sheet lineage

This is a domain decision, not a software decision. But your data model must support it. The right approach is:
- A CardUnit can be `VOIDED` (with reason) — it exits the set
- A `ReplacementCardUnit` can be created with explicit reference to the original it replaces
- The Assembly tracks both expected and actual composition
- Packing requires explicit manager authorization when a set is assembled with a replacement

Document this as a domain decision before you model it. Don't let the software make this choice implicitly.

---

## 5. Event-Driven Thinking Without Infrastructure

Before introducing any event bus, message queue, or CQRS infrastructure, you should think in events. This is the modeling discipline — not the technology.

### What Is a Domain Event?

A domain event is a **fact about something that happened in the past** in your domain. It is:
- Immutable — it happened; you can't un-happen it
- Named in past tense — `SheetCut`, `CardUnitEnteredStation`, `QAPassed`, `AssemblyCompleted`
- Meaningful to the business — a domain expert would say "yes, that's something that matters"
- Rich — it carries all the context needed to understand what happened without lookups

```python
@dataclass(frozen=True)
class SheetCut:
    """Fired when a sheet is cut into individual card units."""
    occurred_at: datetime
    sheet_id: SheetId
    job_id: JobId
    n_up: int                     # how many cards were cut
    operator_id: OperatorId
    station_id: StationId
    # All 18 CardUnit IDs determined at this moment
    card_unit_ids: tuple[CardUnitId, ...]

@dataclass(frozen=True)
class CardUnitEnteredStation:
    occurred_at: datetime
    card_unit_id: CardUnitId
    station_id: StationId
    operator_id: OperatorId
    # Inferred from sequence: previous station exit
    # No "exited" event needed — next "entered" implies exit from previous

@dataclass(frozen=True)
class QAResultRecorded:
    occurred_at: datetime
    card_unit_id: CardUnitId
    result: QAOutcome            # PASS | FAIL
    inspector_id: OperatorId
    failure_reason: str | None   # None if PASS
    failure_codes: tuple[str, ...] # [] if PASS
```

### Events Are Not Log Messages

A common mistake: treating events as structured logs ("INFO: card scanned at station"). Events are **domain facts** that could trigger other behavior, update read models, or be replayed to reconstruct state. They carry business meaning, not technical meaning.

A log message: `"Card S-001-07 scanned at QA-BENCH-3 by operator OP-42 at 14:32:07"`
A domain event: `QAResultRecorded(card_unit_id="S-001-07", result=PASS, inspector_id="OP-42", occurred_at=...)`

The log is for debugging. The event is for the domain.

### Raising Events From Pure Functions

Your domain functions should **return** events, not emit them to a bus. The adapter layer does the emission. This keeps the core pure.

```python
# In core — pure function returns events
def cut_sheet(sheet: Sheet, n_up: int, operator_id: OperatorId, station_id: StationId, at: datetime) -> tuple[Sheet, list[DomainEvent]]:
    if sheet.status != SheetStatus.READY_TO_CUT:
        raise SheetNotReadyForCut(sheet.sheet_id, sheet.status)

    card_unit_ids = [CardUnitId.for_position(sheet.sheet_id, i) for i in range(1, n_up + 1)]
    new_sheet = dataclasses.replace(sheet, status=SheetStatus.CUT)

    events = [
        SheetCut(
            occurred_at=at,
            sheet_id=sheet.sheet_id,
            job_id=sheet.job_id,
            n_up=n_up,
            operator_id=operator_id,
            station_id=station_id,
            card_unit_ids=tuple(card_unit_ids)
        )
    ] + [
        CardUnitCreated(
            occurred_at=at,
            card_unit_id=cuid,
            sheet_id=sheet.sheet_id,
            job_id=sheet.job_id,
            position=i + 1,
        )
        for i, cuid in enumerate(card_unit_ids)
    ]

    return new_sheet, events

# In adapter layer — emits events
sheet, events = cut_sheet(sheet, n_up=18, operator_id=op, station_id=station, at=now())
for event in events:
    event_store.append(event)
    # optionally: message_bus.publish(event)
```

---

## 6. CQRS: Origins, Theory, and What It Actually Solves

### The Intellectual Lineage

CQRS traces back to **Bertrand Meyer's Command-Query Separation (CQS)** principle (1988), which states:

> *Every method should either be a command that performs an action, or a query that returns data to the caller, but not both.*

Meyer applied this at the method level. Greg Young extended it to the object and system level in 2010. The insight: if a command and a query can't share the same method, they may not need to share the same model either.

Greg Young's original definition:

> *"CQRS uses the same definition of commands and queries that Meyer used and maintains the viewpoint that they should be pure. The fundamental difference is that, in CQRS, objects are split into two objects, one containing the commands, one containing the queries."*

Udi Dahan added the business context:

> *"CQRS addresses two driving forces: collaboration (multiple actors modifying shared data) and staleness (data shown to users becomes outdated). The pattern separates command processing from query handling to handle these realities more effectively."*

Martin Fowler's warning (critically important):

> *"Be very cautious about using CQRS. CQRS adds risky complexity to your system. For most systems, sharing a model is easier. I've seen CQRS implementation cause significant drag on productivity in real projects."*

### What CQRS Actually Solves

CQRS solves a specific problem: **your write model and read model want to be different shapes**.

When you write a command, you care about:
- Consistency (did this state transition violate any invariants?)
- Atomicity (did the full change succeed or fail cleanly?)
- Business rules (is this command valid given the current state?)

When you read data, you care about:
- Query performance (fast!)
- View shape (flat, denormalized, ready to display)
- Aggregated data (totals, counts, progress percentages)

These are fundamentally different concerns. A normalized, invariant-protecting domain model is terrible at fast reads. A denormalized read table is terrible at enforcing business rules.

CQRS says: **use separate models for each**. The write model handles commands and produces events. Events update the read model. The read model answers queries.

### The Consistency Question

The most important practical question with CQRS is: **how long can the read model lag behind the write model?**

**Synchronous (same transaction)**: Update the read model in the same database transaction as the event. Zero lag. This is the minimum viable CQRS and is correct for most applications.

**Asynchronous (eventual consistency)**: Update the read model after the transaction commits, via a queue or event stream. Lag exists — could be milliseconds or seconds. Requires your UI to handle "your action was recorded, results will appear shortly." Adds significant complexity. Only worth it at scale.

For a manufacturing tracking system with dozens of concurrent operators (not thousands of concurrent users), **synchronous CQRS is sufficient and correct**. Do not add eventual consistency until you have a proven performance problem.

---

## 7. The Progression Path: CRUD → CQRS-Ready → CQRS → Event Sourcing

This is the most important section for your build strategy.

### Stage 0: Stateless Tools (Where You Are Now)

Your current `tool-hub-try5` is here. Pure functions, no persistence. Excellent foundation.

### Stage 1: CQRS-Ready (Where to Go Next)

You add state, but you structure it so CQRS can be introduced without rewriting anything.

Characteristics:
- **Append-only event log** — every state change is recorded as an event row
- **Pure reducer functions** in core that project events → current state
- **Read models** updated synchronously in the same transaction
- **Single database** (Postgres is fine — you don't need separate read/write stores yet)
- **No message bus** — events flow within the request/response cycle

What you're building in Stage 1 looks like CQRS but uses the same database for reads and writes. The separation is **conceptual and code-level**, not infrastructure-level. This is the minimum viable approach and it carries you very far.

### Stage 2: CQRS (If and When You Need It)

You separate read and write databases. Events are published to a bus or queue. Read models are updated asynchronously.

**Only do this if you have proven**:
- Read query performance is inadequate on the shared database
- You need to scale reads independently of writes
- Your team understands eventual consistency and has built UI to handle it
- You have monitoring to detect when read models lag unacceptably

### Stage 3: Full Event Sourcing (Probably Never)

The event log **is** the source of truth. You never update entity state — you replay events from scratch to reconstruct current state. Read models are always projections.

**Only do this if you need**:
- Complete temporal auditability (replay any moment in history)
- The ability to add new read models from historical events
- Regulatory compliance requiring immutable audit trail

For a card manufacturing shop, Stage 1 (CQRS-Ready) will serve you for years. Stage 2 only if you grow to hundreds of concurrent operators and measurable DB bottlenecks. Stage 3 almost certainly never.

---

## 8. When to Introduce CQRS (Specific Signals)

The signals that tell you it's time to split read and write stores:

### Signal 1: Query Shape Impedance
Your domain model requires complex joins, aggregations, or transformations for every read query. You're spending more time transforming data for display than processing commands. Example: to show the dashboard, you join 6 tables and aggregate 3 datasets on every page load.

### Signal 2: Read/Write Contention
Dashboards and reporting queries are taking table locks that delay write operations. Workers scanning barcodes are waiting because someone ran a report.

### Signal 3: Divergent Scaling Needs
You have 10 operators doing writes but 500 managers refreshing dashboards. Or the inverse: massive batch import jobs with few concurrent readers.

### Signal 4: Read Model Complexity Exceeds Write Model
The queries your business needs are so complex that they require materialized, pre-computed views that the write model cannot efficiently produce.

### Signal 5: Multiple Consumer Shapes
Different departments need completely different views of the same underlying events: operations wants real-time station load, finance wants job cost rollups, shipping wants pack manifests. Maintaining these as separate read models from a shared event log is cleaner than trying to query-engineer everything from a normalized schema.

**Warning signals that you don't need CQRS:**
- "We read more than we write" — this is not sufficient justification
- "CQRS is modern best practice" — not a technical reason
- "We want to scale eventually" — scale to what? How many orders?
- "Event sourcing sounds interesting" — solve real problems, not hypothetical ones

---

## 9. How to Build CQRS-Ready Without CQRS

This is the practical section. Here is exactly how to structure the code so that introducing full CQRS later requires no rewrites — only additions.

### The Event Log Table (The Foundation of Everything)

```sql
CREATE TABLE production_events (
    -- Global ordering (safe for concurrent writers)
    sequence_num     BIGSERIAL PRIMARY KEY,

    -- Per-aggregate ordering (for optimistic concurrency)
    aggregate_id     TEXT NOT NULL,
    aggregate_type   TEXT NOT NULL,       -- 'sheet' | 'card_unit' | 'assembly' | 'job'
    aggregate_version INTEGER NOT NULL,    -- monotonically increasing per aggregate

    -- The event
    event_type       TEXT NOT NULL,       -- 'SheetCut' | 'CardUnitEnteredStation' | ...
    event_payload    JSONB NOT NULL,      -- the full event, serialized

    -- Context
    occurred_at      TIMESTAMPTZ NOT NULL,
    recorded_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    operator_id      TEXT,
    station_id       TEXT,
    correlation_id   TEXT,               -- links events from the same user action

    -- Enforce no duplicate versions per aggregate (optimistic concurrency)
    UNIQUE (aggregate_id, aggregate_type, aggregate_version)
);

CREATE INDEX ON production_events (aggregate_id, aggregate_type);
CREATE INDEX ON production_events (event_type);
CREATE INDEX ON production_events (occurred_at);
```

This table never gets UPDATE or DELETE. Ever. It is append-only.

### Read Model Tables (Materialized Projections)

These are updated synchronously when events are appended. In Stage 1, they live in the same database:

```sql
-- Current state of each card unit
CREATE TABLE rm_card_units (
    card_unit_id        TEXT PRIMARY KEY,
    sheet_id            TEXT NOT NULL,
    job_id              TEXT NOT NULL,
    position            INTEGER NOT NULL,    -- 1..N on sheet
    current_station_id  TEXT,
    status              TEXT NOT NULL,       -- 'in_progress' | 'qa_passed' | 'qa_failed' | 'assembled' | 'packed'
    qa_result           TEXT,               -- 'pass' | 'fail' | null
    qa_failure_reason   TEXT,
    sequential_number   TEXT,               -- if serialized
    last_event_seq      BIGINT,             -- which event last updated this
    updated_at          TIMESTAMPTZ
);

-- Station current load
CREATE TABLE rm_station_load (
    station_id          TEXT PRIMARY KEY,
    card_unit_count     INTEGER NOT NULL DEFAULT 0,
    sheet_count         INTEGER NOT NULL DEFAULT 0,
    last_event_seq      BIGINT,
    updated_at          TIMESTAMPTZ
);

-- Job progress
CREATE TABLE rm_job_progress (
    job_id              TEXT PRIMARY KEY,
    total_sheets        INTEGER,
    total_cards         INTEGER,
    cards_at_qa         INTEGER DEFAULT 0,
    cards_qa_passed     INTEGER DEFAULT 0,
    cards_qa_failed     INTEGER DEFAULT 0,
    cards_assembled     INTEGER DEFAULT 0,
    cards_packed        INTEGER DEFAULT 0,
    status              TEXT,
    last_event_seq      BIGINT,
    updated_at          TIMESTAMPTZ
);

-- Assembly (gathering) state
CREATE TABLE rm_assemblies (
    assembly_id         TEXT PRIMARY KEY,
    sheet_id            TEXT NOT NULL,
    job_id              TEXT NOT NULL,
    expected_count      INTEGER NOT NULL,
    gathered_count      INTEGER NOT NULL DEFAULT 0,
    missing_positions   INTEGER[],
    status              TEXT NOT NULL,    -- 'pending' | 'in_progress' | 'complete' | 'error'
    last_event_seq      BIGINT,
    updated_at          TIMESTAMPTZ
);
```

### The Pure Reducer Functions in Core

These live in `core/` with no I/O:

```python
# core/projections/card_unit_projection.py

@dataclass
class CardUnitReadModel:
    card_unit_id: str
    sheet_id: str
    job_id: str
    position: int
    current_station_id: str | None
    status: str
    qa_result: str | None
    qa_failure_reason: str | None
    sequential_number: str | None
    last_event_seq: int

def apply_event(state: CardUnitReadModel | None, event: dict) -> CardUnitReadModel | None:
    """
    Pure function. Given current read model state and an event dict,
    return new read model state.
    This is a left-fold: fold(events) -> current_state.
    """
    event_type = event["event_type"]
    payload = event["event_payload"]
    seq = event["sequence_num"]

    match event_type:
        case "CardUnitCreated":
            return CardUnitReadModel(
                card_unit_id=payload["card_unit_id"],
                sheet_id=payload["sheet_id"],
                job_id=payload["job_id"],
                position=payload["position"],
                current_station_id=None,
                status="created",
                qa_result=None,
                qa_failure_reason=None,
                sequential_number=None,
                last_event_seq=seq,
            )
        case "CardUnitEnteredStation":
            if state is None: return None
            return dataclasses.replace(state,
                current_station_id=payload["station_id"],
                last_event_seq=seq,
            )
        case "QAResultRecorded":
            if state is None: return None
            return dataclasses.replace(state,
                status="qa_passed" if payload["result"] == "PASS" else "qa_failed",
                qa_result=payload["result"],
                qa_failure_reason=payload.get("failure_reason"),
                last_event_seq=seq,
            )
        case _:
            return state  # event not relevant to this projection
```

**This is the mathematical core of CQRS.** Each read model is a left-fold over the event stream:

```
current_state = fold(apply_event, initial_state, events)
```

This is functionally equivalent to:
```python
state = None
for event in events:
    state = apply_event(state, event)
return state
```

The power: if you need a new read model, write a new reducer. **You never change the event log.** You replay historical events through the new reducer and you have your new view — historically complete, back to day one.

### The Adapter: Synchronous Projection Update

In the adapter layer, when you write an event, you immediately update the read model in the same transaction:

```python
# adapters/event_store/postgres.py

def append_event(conn, event: dict, aggregate_version: int) -> dict:
    """
    Append event to log and update read models — all in one transaction.
    This is synchronous CQRS. No eventual consistency. No message bus.
    """
    with conn.transaction():
        # 1. Append to event log
        row = conn.execute("""
            INSERT INTO production_events
            (aggregate_id, aggregate_type, aggregate_version, event_type, event_payload, occurred_at, operator_id, station_id)
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
            RETURNING sequence_num, recorded_at
        """, [
            event["aggregate_id"],
            event["aggregate_type"],
            aggregate_version,
            event["event_type"],
            json.dumps(event["payload"]),
            event["occurred_at"],
            event.get("operator_id"),
            event.get("station_id"),
        ]).fetchone()

        event_with_seq = {**event, "sequence_num": row["sequence_num"]}

        # 2. Update all relevant read models (pure functions decide new state)
        _update_card_unit_projection(conn, event_with_seq)
        _update_station_load_projection(conn, event_with_seq)
        _update_job_progress_projection(conn, event_with_seq)
        _update_assembly_projection(conn, event_with_seq)

    return event_with_seq
```

When you're ready to go to full CQRS, you:
1. Remove the synchronous projection updates from this function
2. Add event publishing after commit
3. Run projection updaters as separate services consuming the event stream

**Zero rewrites to core.** The reducer functions stay identical.

---

## 10. Projections: The Mathematics of Read Models

A projection is a **left-fold** (functional programming term) over an ordered sequence of events:

```
fold :: (state → event → state) → initial_state → [event] → state
```

This is mathematically equivalent to the reduce/accumulate pattern:

```python
from functools import reduce

def build_state(events: list[Event], initial_state: State) -> State:
    return reduce(apply_event, events, initial_state)
```

### Why This Matters

1. **Correctness**: Pure functions are trivially testable. Give a sequence of events, assert the resulting state. No database setup needed.

2. **Rebuildability**: If you have a bug in a projection, fix the reducer and replay. The event log is the ground truth.

3. **New Views for Free**: Six months from now when you need "show me all cards from Job J-1044 that failed QA at FOIL-2 during the last week", you write a new reducer. You don't need new data — it was already captured.

4. **Time Travel**: Project only events before timestamp T to see the state of the system at time T. This is how you answer "what did we think we knew at 2pm on Tuesday?"

### Live vs Materialized Projections

**Live projection**: rebuild on every query by replaying events. Simple, always correct, slow for large event streams.

**Materialized projection**: maintain a pre-built read model table, update it when events come in. Fast queries, requires keeping the table in sync.

The recommended strategy (from EventStoreDB/Kurrent's documentation):
> *"Start with the live projection strategy — rebuild on demand. Then, if necessary, switch to the full-blown implementation with a separate service."*

For your system: materialize the critical dashboards (station load, job progress, assembly status). Keep less-critical views as live projections.

### Snapshot for Long-Running Aggregates

If a Sheet has been through 50 events before being cut, you don't want to replay all 50 to get its current state every time you process a command. A **snapshot** is a cached state at a known version:

```python
@dataclass
class SheetSnapshot:
    sheet_id: str
    at_version: int         # which event version this snapshot represents
    state: dict             # serialized Sheet state
    created_at: datetime
```

When loading a Sheet: load the latest snapshot, then replay only events after that snapshot's version. For small systems, you won't need snapshots for a while — aggregate streams rarely exceed 100-200 events.

---

## 11. Process Manager vs Saga: Which One You Need

These two patterns are often confused. The distinction matters for your fan-in problem.

### Saga (Choreography)
A Saga has **no centralized state**. Each step publishes an event, and other components react. No coordinator knows the full picture.

```
CardUnit QA Passes → QAPassed event → Assembly hears it → updates its own state
```

Simple, decoupled, but hard to debug. If something breaks mid-process, there's no single place to look. Hard to enforce "all 18 cards must be gathered before packing" because nobody owns that invariant.

### Process Manager (Orchestration)
A Process Manager **is a stateful coordinator**. It knows where the process is, listens for events, and issues commands to advance it.

```python
@dataclass
class AssemblyProcessManager:
    """
    Coordinates the gathering of N CardUnits into a complete Set.
    Knows the expected membership. Tracks who has arrived.
    Enforces the invariant: all members present before completion.
    """
    assembly_id: AssemblyId
    job_id: JobId
    sheet_id: SheetId
    expected_positions: frozenset[int]     # {1, 2, 3, ... 18}
    gathered_positions: frozenset[int]     # which have arrived
    status: ProcessStatus

    def on_card_unit_scanned_for_assembly(self, event: CardScannedForAssembly) -> list[Command]:
        """React to event. Return commands to issue."""
        # Validate: does this card belong here?
        if event.sheet_id != self.sheet_id:
            return [RejectCard(card_id=event.card_unit_id, reason="wrong_sheet")]

        # Validate: is this a duplicate?
        if event.position in self.gathered_positions:
            return [RejectCard(card_id=event.card_unit_id, reason="duplicate")]

        new_gathered = self.gathered_positions | {event.position}

        # Are we complete?
        if new_gathered == self.expected_positions:
            return [CompleteAssembly(assembly_id=self.assembly_id)]

        # Missing positions
        missing = self.expected_positions - new_gathered
        return [UpdateAssemblyProgress(assembly_id=self.assembly_id, missing=missing)]
```

**For your system: use a Process Manager for assembly.** The zero-error requirement means you need centralized state that can enforce the completeness invariant. A saga would let cards slip through the cracks.

The Process Manager's state is persisted to its own table and updated via its own events. It is a first-class entity in your domain.

---

## 12. Manufacturing Genealogy and Serialization: Industry Precedent

Your problem has been solved at scale in semiconductor manufacturing. Wafer-to-die is the canonical example:

**Wafer → Dies (1:300+ transformation)**
- Each wafer gets a unique ID at creation (scribed or laser-marked)
- At dicing (cut), each die receives a coordinate-based ID: `(Wafer-ID, X, Y)` — deterministic, encodes lineage
- Dies go through individual bonding, packaging, and test operations
- Genealogy is maintained as a tree: wafer → die → packaged unit → tested unit
- Any field failure can be traced back to the exact wafer, lot, process run, and operator

**What they got right that you should copy:**

1. **Deterministic IDs that encode lineage** — you can read a die ID and know its wafer without a database lookup
2. **The cut as a first-class event** — it's recorded with context (which machine, which operator, at what time)
3. **Parent inherits nothing forward** — the wafer's status is "diced" but the dies have their own independent status
4. **Genealogy stored as relationships, not copies** — the die knows its wafer ID; you look up the wafer separately
5. **Zero-defect traceability** — every operation, every operator, every timestamp

**For your system:**
```
Sheet ID:      J1044-S003           (Job 1044, Sheet 3)
CardUnit IDs:  J1044-S003-01        (position 1)
               J1044-S003-02        (position 2)
               ...
               J1044-S003-18        (position 18)
```

The ID is a string, but it is not opaque — it carries meaning. A new developer can look at `J1044-S003-07` and understand exactly what it is.

---

## 13. Event Modeling: Designing the System Before Writing Code

Event Modeling (Adam Dymitruk, 2018) is a methodology for designing information systems using events as the primary building block. It is designed to eliminate rework by producing a blueprint that all stakeholders (developers, domain experts, UX designers) can read.

### The Four Ingredients

1. **Events** (orange) — facts about what happened: `SheetCut`, `CardUnitEnteredStation`, `QAPassed`
2. **Commands** (blue) — user intents that trigger events: `CutSheet`, `ScanCardAtStation`, `RecordQAResult`
3. **Read Models / Views** (green) — what users see that informs their next command: "Current Station Load", "Assembly Progress"
4. **Automations** (purple) — system-triggered commands: "when all 18 cards assemble, automatically start packing workflow"

### The Event Modeling Timeline for Your System

Reading left to right:

```
[View: Job Queue]
      ↓ operator reviews queue
[Command: StartJob]
      ↓
[Event: JobStarted]
      ↓ creates
[Event: SheetsCreated]
      ↓
[View: Press Queue / Sheet List]
      ↓ pressman selects next sheet
[Command: CheckSheetIntoPressStation]
      ↓
[Event: SheetEnteredPressStation]
      ↓
[Command: RecordPrintComplete]
      ↓
[Event: PrintCompleted]
      ↓
[Event: SheetExitedPressStation]
      ↓
... (foil stamping, embellishment operations) ...
      ↓
[View: Cutting Queue]
      ↓ operator selects sheet
[Command: CutSheet]
      ↓
[Event: SheetCut]
      ↓
[Events: CardUnit01Created, CardUnit02Created, ... CardUnit18Created]
      ↓
[View: Individual Card Queues — 18 parallel lanes]
      ↓
... (individual operations, QA per card) ...
      ↓
[View: Assembly Station — shows which cards in set have arrived]
      ↓ gather operator scans each card
[Command: ScanCardForAssembly]
      ↓
[Event: CardScannedForAssembly]
      ↓
[Automation: when all 18 scanned → AssemblyComplete]
      ↓
[Event: AssemblyCompleted]
      ↓
[Command: PackSet]
      ↓
[Event: SetPacked]
```

**This blueprint is your specification.** Before writing a single class, you can walk a domain expert through this and verify it matches reality. Every box that doesn't make sense is a domain question to resolve before coding.

### The Mapping to CQRS

Event modeling naturally produces CQRS:
- **Commands** = write side inputs
- **Events** = write side outputs (stored in event log)
- **Read Models / Views** = read side (projections)
- **Automations** = event handlers that issue new commands

When you draw out the event model, you're implicitly designing your CQRS architecture. The only thing missing is the infrastructure to run it.

---

## 14. Capacity Planning: The Data Model

Capacity planning requires knowing: **how much work can a station do, and how much work is queued?**

### Core Concepts

**Station** — a physical resource with a capacity rate:
```python
@dataclass
class Station:
    station_id: StationId
    name: str
    station_type: StationType          # PRESS | FOIL_STAMP | QA | CUTTING | ASSEMBLY
    capacity_unit: CapacityUnit        # SHEETS_PER_HOUR | CARDS_PER_HOUR
    rated_capacity: Decimal            # e.g., 500 (cards/hour)
    setup_time_minutes: int            # changeover time between jobs
    operating_hours: OperatingSchedule # when this station is staffed
```

**Operation** — a unit of work for a specific entity type:
```python
@dataclass(frozen=True)
class OperationSpec:
    operation_type: str
    station_type: StationType
    standard_time: Decimal      # expected time per unit (minutes)
    # Used for scheduling and capacity planning
```

**Queue** — derived from events. You don't store the queue — you project it:
```python
def project_station_queue(events: list[Event], station_id: StationId) -> StationQueue:
    """
    Project current queue for a station from events.
    An entity is 'in queue' if it last entered this station and hasn't left.
    """
    ...
```

### Capacity Planning Algorithm

The simplest correct approach:

1. For each Job, enumerate all required Operations (from product spec)
2. For each Operation, look up the standard time at the required Station
3. Build a timeline: earliest start = max(predecessor operations complete, station available)
4. Station available = station's current queue + in-progress work / station capacity
5. Commit date = latest operation end time + pack + ship duration

This is a **directed acyclic graph (DAG)** scheduling problem. Your product spec defines the DAG structure (what operations must happen in what order). Your event data gives you current position in the DAG.

```python
@dataclass
class OperationNode:
    operation_id: str
    operation_type: str
    station_type: StationType
    required_for_entity_type: str    # "sheet" or "card_unit"
    predecessors: list[str]          # operation_ids that must complete first
    standard_time_minutes: Decimal

@dataclass
class ProductSpec:
    product_type: str
    operation_dag: list[OperationNode]
    n_up: int                        # cards per sheet
```

Start simple: estimate only. Build the data structures that let you later add constraint-based scheduling. Don't try to build SAP-level scheduling on day one.

---

## 15. Build Order: First Principles to Production

This is the master plan. Each phase is self-contained and useful. No phase requires the next phase to deliver value.

---

### Phase 0: Domain Language and Identity (Pure Core, No DB)
**What:** Define the ubiquitous language as a DECISIONS.md. Build identity primitives as pure functions. Write tests.

**Deliverables:**
- `DECISIONS.md` — every domain concept defined, every invariant stated
- `core/identity/` — `SheetId`, `CardUnitId`, `JobId` with deterministic generation
- `core/domain/models.py` — `Sheet`, `CardUnit`, `Job`, `Assembly`, `Station` as pure dataclasses
- `core/domain/events.py` — all domain events as frozen dataclasses
- `core/domain/commands.py` — all commands as frozen dataclasses
- `core/domain/state_machines.py` — valid transitions for Sheet and CardUnit
- Tests for all identity, invariant, and state transition logic

**No database. No adapters. No HTTP. Just math.**

This phase is the most important. Get the vocabulary right here and everything else follows. Get it wrong and you'll be refactoring across your entire codebase.

---

### Phase 1: Domain Logic (Pure Core, No DB)
**What:** Implement the business rules as pure functions. The full domain logic for every operation.

**Deliverables:**
- `core/production/sheet_operations.py` — `create_sheet()`, `cut_sheet()`, `enter_station()`, `exit_station()`
- `core/production/card_unit_operations.py` — `create_card_unit()`, `record_qa()`, `apply_sequential_number()`, `place_memorabilia()`
- `core/production/assembly.py` — `scan_card_for_assembly()`, `validate_assembly_complete()`
- `core/projections/` — all reducer functions (pure folds)
- `core/capacity/` — station capacity calculations, queue length projection
- Full test coverage for every function

**All functions have signature: `(state, command) -> (new_state, list[events])` or `(state, event) -> new_state`**

---

### Phase 2: Event Store Adapter (First Persistence)
**What:** Postgres adapter. Append-only event log + synchronous read model updates.

**Deliverables:**
- `adapters/event_store/` — Postgres event log writer
- `adapters/projections/` — synchronous read model updaters (call core reducers, write to DB)
- Database schema (migrations)
- Integration tests with real Postgres

---

### Phase 3: Scan API (First HTTP)
**What:** Minimal HTTP API for barcode scanning. Workers scan once; system infers transitions.

**Deliverables:**
- `adapters/flask/scan_api/` — single endpoint `POST /scan`
- Payload: `{barcode, station_id, operator_id, timestamp}`
- System determines: is this a sheet? A card? What state transition does this trigger?
- Returns: current entity state + any validation errors
- No UI. JSON only. Designed to be called by a simple barcode scanner app.

---

### Phase 4: Read API (Queries)
**What:** HTTP endpoints for all read model queries. No writes.

**Deliverables:**
- `GET /jobs/{id}/progress` — job completion status
- `GET /stations/{id}/queue` — current queue at station
- `GET /card-units/{id}` — current state + history of a card
- `GET /assemblies/{id}` — which cards present, which missing
- `GET /capacity` — current system-wide load

**These are trivial because the read models are already maintained.**

---

### Phase 5: Job and Order Management
**What:** Create and manage production orders and jobs. Connect to existing quoting tools.

**Deliverables:**
- `POST /jobs` — create a new job (spawns sheets, defines operation plan)
- `GET /jobs` — list all active jobs
- Integration with existing `order_quote` and `trading_card_sheet_normalizer` tools
- Product specification model (defines the operation DAG for each product type)

---

### Phase 6: Capacity Planning
**What:** Forward-looking scheduling from current state.

**Deliverables:**
- `core/scheduling/` — DAG-based schedule computation
- `GET /capacity/forecast` — predicted load at each station for next N days
- `GET /jobs/{id}/estimated-completion` — derived from current state + remaining operations
- Alert rules: jobs at risk of missing commit date

---

### Phase 7: AI Layer
**What:** Build on top of the structured event log.

**Deliverables:**
- Anomaly detection: flag operations that take 2x standard time
- Predictive QA: identify patterns that correlate with failures
- Smart scheduling suggestions
- Eventually: vision-based scan (camera recognizes barcode automatically)

The AI layer is easy to build when you have a clean, structured event log. You are building the training data in Phases 0-6.

---

## 16. The AI Layer: Why This Foundation Matters

Every event in your log is a structured data point:

```json
{
  "event_type": "QAResultRecorded",
  "occurred_at": "2026-03-15T14:32:07Z",
  "card_unit_id": "J1044-S003-07",
  "sheet_id": "J1044-S003",
  "job_id": "J1044",
  "inspector_id": "OP-42",
  "station_id": "QA-BENCH-3",
  "result": "FAIL",
  "failure_reason": "foil_adhesion",
  "failure_codes": ["F-201"]
}
```

After six months of operation, you have tens of thousands of events like this. You can answer:
- "Which operators have the highest QA pass rates?"
- "Does job type X correlate with higher foil adhesion failures?"
- "At what time of day do QA failures spike?"
- "Which product specs produce the most rework?"

An LLM agent can query this structured event log and produce insights that would take weeks to surface from a traditional system. But only if the data is structured. Unstructured logs, spreadsheets, or ad-hoc database schemas produce noise.

**Build the event log correctly now. The AI reads it later.**

---

## 17. Common Mistakes to Avoid

### Mistake 1: Modeling State Instead of Events
Wrong: `UPDATE card_units SET status = 'qa_passed' WHERE id = 'J1044-S003-07'`
Right: `INSERT INTO production_events (event_type='QAResultRecorded', ...) → update read model`

Once you update in place, you lose the history. You can't answer "when did this card pass QA?" or "how many QA attempts did it take?"

### Mistake 2: Making IDs Opaque
Wrong: `card_unit_id = uuid4()` — tells you nothing about the card
Right: `card_unit_id = "J1044-S003-07"` — tells you job, sheet, and position at a glance

Opaque IDs force database lookups to understand what something is. Semantic IDs make the system self-describing.

### Mistake 3: The God Aggregate
Wrong: One `ProductionOrder` aggregate that contains sheets, cards, operations, QA results, everything.
Right: Small, focused aggregates. `Sheet` knows about sheet-level operations. `CardUnit` knows about card-level operations. `Assembly` coordinates gathering.

God aggregates create massive contention — you can't update a single card without locking the entire order.

### Mistake 4: Introducing Eventual Consistency Too Early
Eventual consistency is a performance optimization with a significant UX and operational cost. Don't add it until you have proven you need it. A synchronous, single-database CQRS can handle thousands of operations per day without breaking a sweat.

### Mistake 5: Reporting As Events
Wrong: emitting events like `DashboardViewed` or `ReportGenerated` — these are queries, not domain facts.
Right: events are state changes in the domain. Only things that happened to physical objects or business entities are events.

### Mistake 6: Skipping the Ubiquitous Language
If your code uses "item", "thing", "record", or "object" where the business says "card unit", "sheet", and "assembly", you've already introduced translation costs. Every time a developer reads a variable named `item`, they have to translate. Name things what the business calls them.

### Mistake 7: Building UI Before Domain
You will be tempted to build a nice interface to see your progress. Don't. The UI is a trap — it shapes the data model to what looks good on screen rather than what is correct in the domain. Build the domain model first. Build the API second. Build the UI last.

### Mistake 8: Using CQRS as Top-Level Architecture
Udi Dahan: *"CQRS is not a top-level architecture. CQRS is something that happens at a much lower level."* Apply it within a bounded context (e.g., production tracking), not across your entire system. Your quoting tool doesn't need CQRS. Your label generator doesn't need CQRS. Your production tracking module does.

---

## 18. Summary: The Master Sequence

```
PHASE 0: Language and Identity
  → DECISIONS.md (domain vocabulary, invariants, non-goals)
  → Identity primitives (deterministic IDs with lineage)
  → Domain model (pure dataclasses)
  → Domain events (frozen dataclasses, past tense)
  → State machines (valid transitions)
  → No persistence. No HTTP. 100% testable.

PHASE 1: Domain Logic
  → Sheet operations (create, station entry/exit, cut → N cards)
  → CardUnit operations (create, individual ops, QA, sequential numbers)
  → Assembly (scan, validate, complete)
  → Projections/reducers (pure left-fold functions)
  → Capacity calculations
  → No persistence. No HTTP. 100% testable.

PHASE 2: Event Store
  → Postgres append-only event log
  → Synchronous read model updates (same transaction)
  → This is "CQRS-Ready" — full CQRS requires only adding a message bus
  → Integration tests with real DB

PHASE 3: Scan API
  → POST /scan → command → events → read model updated
  → Zero-friction for workers

PHASE 4: Read API
  → GET endpoints for all dashboards
  → Trivially implemented from read models

PHASE 5: Job Management
  → Order intake → job creation → sheet spawning
  → Integration with existing tools

PHASE 6: Capacity Planning
  → Scheduling, forecasting, deadline risk

PHASE 7: AI Layer
  → Anomaly detection, prediction, optimization
  → Easy because the event log is clean structured data

CQRS UPGRADE (when signals appear — not before):
  → Separate read database
  → Event publishing after transaction commit
  → Async projection updaters
  → Zero core rewrites — only adapter changes
```

---

## Sources

- [CQRS Documents by Greg Young (2010)](https://cqrs.wordpress.com/wp-content/uploads/2010/11/cqrs_documents.pdf)
- [Martin Fowler on CQRS](https://www.martinfowler.com/bliki/CQRS.html)
- [Clarified CQRS — Udi Dahan](https://udidahan.com/2009/12/09/clarified-cqrs/)
- [Udi & Greg Reach CQRS Agreement](https://udidahan.com/2012/02/10/udi-greg-reach-cqrs-agreement/)
- [CQRS without Event Sourcing — Medium](https://medium.com/@mbue/some-thoughts-on-using-cqrs-without-event-sourcing-938b878166a2)
- [CQRS without Event Sourcing, but with Event Log — Medium](https://medium.com/@marco_be/cqrs-without-event-sourcing-but-most-likely-with-an-event-log-94fb5a5b5b84)
- [Event Storage in Postgres — DEV Community](https://dev.to/kspeakman/event-storage-in-postgres-4dk2)
- [Live Projections for Read Models — Kurrent/EventStoreDB](https://www.kurrent.io/blog/live-projections-for-read-models-with-event-sourcing-and-cqrs)
- [Event Modeling — eventmodeling.org](https://eventmodeling.org/)
- [Interview with Adam Dymitruk on Event Modeling — InfoQ](https://www.infoq.com/news/2020/09/adameventmodeling/)
- [Architecture Patterns with Python (Cosmic Python) — Domain Modeling](https://www.cosmicpython.com/book/chapter_01_domain_model.html)
- [Architecture Patterns with Python — Events and Message Bus](https://www.cosmicpython.com/book/chapter_08_events_and_message_bus.html)
- [Saga and Process Manager — event-driven.io](https://event-driven.io/en/saga_process_manager_distributed_transactions/)
- [Semiconductor Wafer to Die Serialization — Intraratio RunCard MES](https://blog.intraratio.com/die-serialization)
- [Genealogy and Serialization in Digital Manufacturing — iBase-t](https://www.ibaset.com/how-genealogy-and-serialization-is-eased-in-digital-manufacturing/)
- [CQRS Pattern — Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [Busting CQRS Myths — Jimmy Bogard](https://lostechies.com/jimmybogard/2012/08/22/busting-some-cqrs-myths/)
- [1 Year of Event Sourcing and CQRS — ITNEXT](https://itnext.io/1-year-of-event-sourcing-and-cqrs-fb9033ccd1c6)
- [Achieving Consistency in CQRS with Linear Event Store](https://squirrel.pl/blog/2015/09/14/achieving-consistency-in-cqrs-with-linear-event-store/)
- [Hexagonal Architecture: Persistence-Ignorant Domain Models — Medium](https://medium.com/@john200Ok/domain-models-that-are-100-ignorant-of-persistence-and-orm-unaware-d8f7a8253c7b)
