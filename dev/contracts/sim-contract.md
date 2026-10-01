# Supermarket Operations Simulation — Design-by-Contract Specification

**Contract version:** 0.3 (Merchandising Space Allocation → SKU Placements)
**Date:** 2026-10-01
**Supersedes:** v0.2 (2026-09-29). Part 11 is the change log from v0.2: every clause that was added, revised, or retired.
**Status:** Draft for review. Companion file: `sim-contract-discussion.md`. It opens with the decisions this revision made on its own (D-1 …), followed by textbook-vs-functionality conflicts, conflicts inside the functionality file, open questions, and cross-workstream flags.

**Sources this contract is derived from (and only these):**

| Tag | Source |
|---|---|
| `[TB x.y]` | *How a Modern Supermarket Works: Operations Textbook* (`modern-supermarket-operations.md`), chapter.section |
| `[VS n]` | *Voice Session Extract — Sim App Design Points* (`dev/contracts/voice-session-sim-design-extract.md`, Sept 27 2026), section n. This is the app-functionality file. |
| `[DISC]` | The merchandising-space design discussion recorded in project memory: sheet-per-space-type, no inheritance of attribute shape, a Protocol for tree behavior, the Gondola → Side → Bay → Shelf Run composition, the four aisles named from a viewer at the front of the store facing the rear, and the five fixture archetypes |
| `[DbC]` | Bertrand Meyer, *Object-Oriented Software Construction*, 2nd ed., and the Design by Contract method (assertions, invariants, command/query separation, subcontracting, creation procedures) |
| `[SIM]` | A simulation-only decision made in this contract, with no textbook or discussion grounding. Every `[SIM]` item that needs confirmation is listed in the companion file. |

The workbook is not a source for this contract. Workbook column names appear only in the companion file's cross-workstream flags. No implementation code was consulted.

---

## Part 0 — How to read and use this contract

### 0.1 Purpose, authority, and normativity

This document is the **sole specification** for the Python classes of the simulation's data model. A class, feature, or behavior not specified here is not part of the model. `[DbC]`

There are three kinds of content, with different force:

1. **Clauses are the sole normative specification.** A clause is a table row with an ID (for example `PLC-INV-5`). An implementation is correct when, and only when, it satisfies every clause for every call a correct client can make.
2. **The contract-by-example usages of Part 9** (worked usages, the query matrix, and the negative cases N-1 …) are **required acceptance tests**. The functionality file calls for them: "the example usages become the contract" `[VS 13, 14]`. The implementation must pass them. They are consistent with the clauses; if one ever disagrees with a clause, that is a defect in this contract to be raised, and the clause governs until it is fixed.
3. **The per-routine "Examples" blocks are illustrative.** They never add to, weaken, or contradict a clause. They are written so that they can be turned directly into pytest cases.

The document must stand on its own: a fresh session with no other context should be able to build the classes from it. `[VS 14]`

**Evolution rule (Open–Closed).** Later versions may add classes, features, and clauses. They may not weaken an existing postcondition or invariant, or strengthen an existing precondition, without a version bump that names the changed clause (Part 11 does this for v0.3). A clause ID is never reused, including the ID of a retired clause. `[DbC]`

### 0.2 Assertion vocabulary `[DbC]`

| Term | Meaning | Whose bug if violated |
|---|---|---|
| **require** (precondition) | What the client must make true before calling | The client (caller) |
| **ensure** (postcondition) | What the supplier guarantees on return, given the precondition held | The supplier (implementer) |
| **invariant** | What is true of every instance at every *stable time*: after creation and before and after every exported call. It may be false inside a routine of the class | The supplier |
| **review** | A requirement verified by reading the code or the data, not by a runtime assertion. It has no predicate | The supplier |
| `old` | The store snapshot taken on entry to a command (0.7) | — |
| `result` | The value a query or creation command returns | — |

Clauses are written as **Python boolean expressions** over the class's queries, so each one can be implemented as a runtime check. Every clause row carries its grounding tags in a third column. Appendix A indexes every clause by tag.

**Attached invariants.** An invariant marked **(A)** relates a record or node to the store that contains it. It holds from the moment the object is attached to a store (by `Store.build` or a store command) and is not evaluated on a detached object (0.8).

### 0.3 Design rules the implementation must follow

1. **Command–Query Separation.** A *query* returns information and has no observable side effect. A *command* changes state and returns nothing, except a creation command, which returns the new record's id (an accepted exception `[DbC]`). Every expression used in an assertion uses only queries. `[DbC]`
2. **Uniform Access.** A client can't tell whether a query is stored or computed. Derived facts (a placement's shelf level, its facing count, its business unit, whether it is vendor-funded) are queries and are **not** stored redundantly. `[DbC]` `[VS 8]`
3. **Preconditions are not defensive checks.** The supplier assumes the precondition. It does not quietly "handle" a violation with a fallback. When monitoring is on (0.4), a violation raises `PreconditionViolation`. `[DbC]`
4. **Expected outcomes are not exceptions.** Something the domain expects, like a SKU that doesn't fit in its category's space, is reported in a result (for example `PlacementPlan.unplaced`), never raised. Exceptions are reserved for contract violations. `[DbC]`
5. **Data shape vs. behavior.** Each space type is its own self-contained class with its own attributes. There is **no shared base class for attribute shape**. Shared *behavior* (tree navigation and rollup) is a **deferred class**, implemented in Python as a `typing.Protocol`. Every class that conforms to a deferred class inherits its contracts. `[DISC]` `[DbC]`
6. **Subcontracting.** A class conforming to a deferred class may only *weaken* inherited preconditions and *strengthen* inherited postconditions and invariants. `[DbC]`
7. **One hierarchy shape, implemented once.** The space, space-type, business-unit, and product-category hierarchies all conform to `HierarchyNode` (2.1). Their tree behavior is written once. `[VS 6]`
8. **Geometry and economics are separate layers.** No space class has a price, fee, or premium attribute. Money attaches to `Agreement`s, and an agreement attaches to a placement, never to a space. `[VS 3, 7]`
9. **Filters return sets.** Every filter query returns a `frozenset`. "And" is intersection, "or" is union, "but not" is difference. There is no query language. `[VS 13]`
10. **Intervals are half-open.** A date interval `[start, end)` contains `d` iff `start <= d and (end is None or d < end)`. `end is None` means open-ended. A placement replaced at a reset on date `R` ends at `R`, and its successor starts at `R`. `[VS 4]` `[SIM]`

### 0.4 Monitoring API (testability) `[DbC]` `[SIM]`

| ID | Clause | Grounding |
|---|---|---|
| MON-1 | **Modules.** The model is the module `supermarket_sim.model`: every class, enumeration, and module-level function of Parts 1–8. The monitoring runtime is the module `supermarket_sim.contracts`: the violation exceptions, `AssertionLevel`, `get_assertion_level`, `set_assertion_level`, `assertion_level`, `check_invariants`, and `predicate`. The clause predicates are the module `supermarket_sim.predicates`. | `[DbC]` `[SIM]` |
| MON-2 | **Violations carry the clause.** `ContractViolation(Exception)` has the attribute `clause_id: str`. Its three subclasses are `PreconditionViolation`, `PostconditionViolation`, and `InvariantViolation`. Every raised violation carries the ID of exactly one clause of this contract. | `[DbC]` |
| MON-3 | **Levels.** `AssertionLevel` has the values `NONE < REQUIRE < ENSURE < INVARIANT < ALL`, each checking everything the previous one does. `REQUIRE` checks preconditions. `ENSURE` adds postconditions. `INVARIANT` adds the invariants of the target object after every exported call and creation procedure, and the store invariants (ST-INV-*) after every store command. `ALL` adds a deep check: after every store command and `Store.build`, the invariants of every object the store contains (ST-INV-8). The default level is `ALL`. | `[DbC]` `[SIM]` |
| MON-4 | **Setting the level.** `set_assertion_level(level)` sets the process-wide level and `get_assertion_level()` returns it. `assertion_level(level)` is a context manager that sets `level` on entry and restores the previous level on exit, including exit by an exception. | `[DbC]` `[SIM]` |
| MON-5 | **Precondition order.** When several preconditions of a routine are false, the one reported is the **first false one in the routine's table order**. It is raised **before any state changes**. | `[DbC]` `[SIM]` |
| MON-6 | **NONE checks nothing.** At `NONE`, no clause is evaluated, not even on creation. `NONE` is used only to build counter-example objects in tests. Production and examples run at `ALL` unless they say otherwise. | `[DbC]` `[SIM]` |
| MON-7 | **`check_invariants(obj)`** evaluates the invariants of `obj`'s class, at any level: inherited invariants first (`HierarchyNode`, then `SpaceNode`, then `LeafSpace`, then `Rollup`), then the class's own, each in table order. It raises `InvariantViolation` for the first false one and returns `None` if all hold. (A)-marked invariants are skipped for a detached object. | `[DbC]` `[SIM]` |
| MON-8 | **Predicates.** Every `ensure` and `invariant` clause has a pure, callable predicate in `supermarket_sim.predicates`, named after its clause ID with `-` replaced by `_` (`PLC_INV_5`). `contracts.predicate(clause_id)` returns it. An invariant predicate has the signature `P(obj) -> bool`. A postcondition predicate has the signature `P(self, result, old, **args) -> bool`, where `self` is the object after the call, `old` is the snapshot taken on entry (0.7) for a store command and for `Store.generate_plan`, and `None` for any other query, and `args` are the routine's arguments by name. For a module function, `self` is `None`; for a creation procedure, `self` is the new object and `result` is `None`. `predicate(clause_id)` raises `KeyError` for a clause that has no predicate. The clauses of Part 1 that define enumeration features (ENUM-*) are `ensure` clauses of those features. `review` clauses have no predicate. | `[DbC]` `[SIM]` |
| MON-9 | **Predicates read only what their clause names.** A predicate reads only the queries its clause names, so a stub object that provides just those queries can stand in for a state the constructors cannot build. A predicate never raises a `ContractViolation` itself. | `[DbC]` `[SIM]` |
| MON-10 | **Atomic commands.** If a postcondition or invariant check fails at the end of a store command, the store is restored to `old` before `PostconditionViolation` or `InvariantViolation` is raised. | `[DbC]` |

### 0.5 Units, tolerances, and time `[TB 1.2, 2.3]` `[VS 8]` `[SIM]`

- Linear dimensions are in **inches** (`_in`). Feet appear only in reporting (`_ft`).
- Floor and deck area is in **square feet** (`_sqft`). Hook positions are counted in **hooks**.
- Facings are non-negative **integers**, always derived (PLC-Q-1).
- Money is in **dollars** as a `float`.
- Time enters the model only through explicit `on` / `as_of` date arguments. There is no simulation clock yet (Part 10).
- Floating-point comparisons use `EPS = 1e-6`. Footprint reconciliation uses `FOOTPRINT_TOL_SQFT = 1.0`. Both are module constants of `supermarket_sim.model`.

### 0.6 Identifiers `[SIM]`

| ID | Clause | Grounding |
|---|---|---|
| ID-1 | **One prefix, one class.** Each id prefix below belongs to exactly one class, and no prefix is a prefix of another: `STORE-` Store, `AREA-` Area, `GDL-` Gondola, `SID-` GondolaSide, `BAY-` Bay, `RUN-` ShelfRun, `PEG-` PegSection, `CLP-` ClipStrip, `SHR-` ShelvingRun, `DCR-` DoorCaseRun, `DOR-` Door, `SVC-` ServiceCase, `TIER-` Tier, `TBL-` ProduceTable, `BIN-` Bin, `FD-` FloorDisplay, `CAT-` CategoryNode, `BU-` BusinessUnit, `SKU-` SKU, `AGR-` Agreement, `PLC-` Placement, `INC-` IncidentalPlacement. A product is identified by its `upc`, which carries no prefix. A planogram version is identified by its integer `version_no`. | `[DISC]` `[SIM]` |
| ID-2 | **Created records.** Agreements, placements, and incidental placements have ids of the form `PREFIX` + a six-digit, zero-padded number (`PLC-000001`). The store assigns them: the next id of a prefix is `next_id_number(prefix)`, which starts at one more than the largest number among the records given to `Store.build` (or `1`) and increases by exactly one per record created. An id is never reused, not even after `cancel_future_placement` deletes its record. | `[SIM]` |
| ID-3 | **Version numbers** are `1, 2, 3, …` in release order (PGV-INV-3), and are never reused. | `[VS 10]` `[SIM]` |
| ID-4 | Within one store, ids are unique across all classes (ST-INV-3). Ids are given at creation and never change. | `[SIM]` |

### 0.7 Frame conditions: snapshots and `changed` `[DbC]` `[SIM]`

"Nothing else changes" is stated with one comparable value of the store's mutable state.

| ID | Clause | Grounding |
|---|---|---|
| FRM-1 | `Store.snapshot() -> StoreSnapshot` returns an immutable, hashable value with `==`. It holds every piece of mutable state, keyed as `(component, key)`: `("placement", id)` → all attributes of a committed placement; `("incidental", id)` → all attributes of an incidental placement; `("agreement", id)` → all attributes of an agreement; `("assignment", space_id)` → the id of the space node's `own_assignment`, for every node that has one; `("last_reset", category_id)` → `last_reset_date`, for every category node where it is not `None`; `("working", "")` → the working plan, if any; `("version", str(version_no))` → all features of a planogram version; `("counter", prefix)` → `next_id_number(prefix)` for `PLC`, `INC`, `AGR`. The structure of the four hierarchies, the products, the SKUs, and the reset calendar are static in this version (Q-14) and are not in the snapshot. | `[DbC]` `[SIM]` |
| FRM-2 | `changed(a, b) -> frozenset[tuple[str, str]]` (module function) returns the keys whose values differ between snapshots `a` and `b`, including keys present in only one of them. `changed(a, a) == frozenset()`. | `[DbC]` `[SIM]` |
| FRM-3 | A command's "nothing else changes" postcondition is written `changed(old, self.snapshot()) <= K` for a stated key set `K`. | `[DbC]` |

**Existence checks** used by preconditions are store queries (7.2): `has_space(id)`, `has_category(id)`, `has_business_unit(id)`, `has_sku(id)`, `has_agreement(id)`, `has_placement(id)` (committed or incidental), and `has_version(n)`.

### 0.8 Creation procedures and attachment `[DbC]`

Every class has an explicit creation procedure with its own clauses and examples.

1. **Bottom-up containers.** A container's creation procedure takes **finished, unattached children** and becomes their parent. A child's `parent` is set exactly once, by the container that takes it. Passing an attached child is a precondition violation. Because a node can only be given to a container created after it, no hierarchy can contain a cycle.
2. **The precondition of a creation procedure** is that the class's own invariants hold for the arguments, grouped into a few precondition rows per class.
3. **Records are detached at creation.** An `Agreement`, `Placement`, or `IncidentalPlacement` built by its creation procedure belongs to no store. `Store.build` attaches seed records. Store commands create and attach records in one atomic step. (A)-marked invariants hold from attachment on.
4. **One store per object.** An object is attached to at most one store, for its whole life.

### 0.9 How the Examples blocks are written

Every Examples block uses the shared fixture **EX-STORE-1** (0.10) and Python call syntax. A block runs in a test module that does `from supermarket_sim.model import *`, `from supermarket_sim.contracts import *`, and `from conftest import *` (which provides `P`, `Stub`, the date constants, `expect_violation`, `scope_all`, `s0_build_args`, `node`, and `_runs`). Names such as `s4` are the store in state S4, built fresh for each example by the fixture code. `D_0915` and the other date names are the fixture's constants. `P` is `supermarket_sim.predicates`. `expect_violation(kind, clause_id)` is the fixture's helper; it passes when the block raises `kind` carrying `clause_id`.

| Mark | Meaning |
|---|---|
| ✓ | Correct usage: every precondition holds. It names the start state, the postcondition(s) shown, and, for a command, the state reached |
| ✗ | Client counter-example: the call violates the named precondition and no other, and raises `PreconditionViolation` for it |
| ✗ supplier | A result an incorrect implementation might return. The named postcondition or invariant predicate rejects it |
| ✗ inv | An object state that violates the named invariant: either built at level `NONE`, so that `check_invariants` raises `InvariantViolation` for it, or a stub (MON-9) that the invariant's predicate rejects |

A routine with no precondition has no ✗ client example.

### 0.10 The shared example fixture EX-STORE-1 `[SIM]`

A small fictional store. It is a sequence of states, each reached from the previous one by the calls shown. The fixture is extended only by adding to it; existing values never change.

| State | Reached by | What it adds |
|---|---|---|
| **S0** | `Store.build(...)` (0.10.2) | The building and all fixtures, products, SKUs, the category and business-unit hierarchies, and the reset calendar. No assignments, agreements, placements, or versions |
| **S1** | five `assign_space` calls | Category space: two cereal shelves, the peg section, the cooler, and the produce table |
| **S2** | `add_agreement(SLOTTING, …)` | `AGR-000001`: Acme's slotting deal from the 2026-01-05 reset |
| **S3** | `generate_plan`, `with_agreement` ×2, `set_working_plan`, `release_working_plan` | Version 1, a `CYCLE` version effective 2026-01-05, `RELEASED` |
| **S4** | `execute_version(1, D_0105)` | `PLC-000001` … `PLC-000010`. Version 1 is `EXECUTED` |
| **S5** | `generate_plan`, `with_agreement`, `set_working_plan`, `release_working_plan`, `execute_version(2, D_0706)` | The 2026-07-06 reset. SKU-B's slotting is not renewed: `PLC-000004` ends and `PLC-000011` replaces it. Version 2 is `EXECUTED` |
| **S6** | `add_agreement(PROMOTIONAL_DISPLAY, …)`, `add_promotional_placement`, `add_incidental_placement` | `AGR-000002`, `PLC-000012` (SKU-A on floor display `FD-1`, Sept 2026), `INC-000001` (SKU-B overstock) |

#### 0.10.1 The fixture at a glance

**Space.** `STORE-1` (1200.0 sq ft) has three areas: `AREA-SF` sales floor (800.0), `AREA-BR` backroom (300.0), `AREA-CO` front end, type `CHECKOUT` (100.0).

| Node | Contents |
|---|---|
| `GDL-1` | Gondola, depth 48 in, footprint 32.0. Aisles: left `AISLE-1`, right `AISLE-2`, front `FRONT-AISLE`, back `BACK-AISLE`. Sides: `SID-1L` (`LEFT`, faces `AISLE-1`) with bays `BAY-1L1`, `BAY-1L2` (48 in each), peg section `PEG-1` (4 hooks, 6 deep), clip strip `CLP-1` (6 hooks, 3 deep, 2 in frontage); `SID-1F` (`FRONT`, the end cap, faces `FRONT-AISLE`) with bay `BAY-1F1` (48 in). Every bay has four runs `RUN-<bay>-T/E/M/B` (top, eye level, middle, bottom), each 48 wide, 18 deep, 14 clear |
| `SHR-1` | Wall shelf, footprint 3.0, one bay `BAY-W1` (36 in), runs `RUN-W1-T/E/M/B`, 36 wide, 12 deep, 12 clear |
| `DCR-1` | Reach-in cooler, footprint 10.0, one door `DOR-1` (30 in), runs `RUN-D1-T/E/M/B`, 30 wide, 20 deep, 12 clear |
| `SVC-1` | Deli case, 72 in long, footprint 24.0, tiers `TIER-1E` (eye level) and `TIER-1B` (bottom), 72 wide, 24 deep, 10 clear |
| `TBL-1` | Produce table, footprint 24.0, bins `BIN-1`, `BIN-2` of 8.0 sq ft |
| `FD-1` | Floor display on the sales floor, 24 × 36 in = 6.0 sq ft |
| `AREA-AI` | Aisles, 701.0 sq ft. 32 + 3 + 10 + 24 + 24 + 6 + 701 = 800 |
| `AREA-CO` | `FD-2`, the lane-1 impulse rack, 12 × 12 in = 1.0 sq ft, and `AREA-CL` checkout lanes (`CHECKOUT`, 99.0) |

**Categories** (root policies in full; children inherit; `allowed_space_types` of the dry-grocery root is `{GONDOLA, WALL_SHELF, PEG_SECTION, CLIP_STRIP}`):

| Node | Parent | Own policy | Notes |
|---|---|---|---|
| `CAT-DG` Dry Grocery | — | `ROUTINE`, `AMBIENT`, min 1 / max 6 facings, elasticity 0.5, 10.0 $/ft/wk, no captain, PL target 20 %, reset 6 months, `VERTICAL_BRAND`, no benchmark | root |
| `CAT-CER` Cereal | `CAT-DG` | `max_facings_per_sku = 12` | |
| `CAT-RTE` Ready-to-Eat | `CAT-CER` | — | leaf |
| `CAT-HOT` Hot Cereal | `CAT-CER` | — | leaf |
| `CAT-SPI` Spices | `CAT-DG` | — | leaf |
| `CAT-HBA` Health & Beauty | `CAT-DG` | `secured_case_required = True` (locked cases `[TB 6.4]`) | leaf, never given space |
| `CAT-DY` Dairy | — | as `CAT-DG`, but `REFRIGERATED`, allowed `{REACH_IN_COOLER}`, benchmark brand `"Dairyland"` | root |
| `CAT-YOG` Yogurt | `CAT-DY` | — | leaf |
| `CAT-PR` Produce | — | as `CAT-DG`, but allowed `{PRODUCE_TABLE}` | root |
| `CAT-APL` Apples & Pears | `CAT-PR` | — | leaf |

**Business units:** `BU-CO` (`COMPANY`) → `BU-ST` (`STORE`) → `BU-DG`, `BU-DY`, `BU-PR` (`DEPARTMENT`, root categories `CAT-DG`, `CAT-DY`, `CAT-PR`).

**Products and SKUs** (one SKU per product; price and cost in dollars, velocity in units/week; score = weekly gross margin, POL-1):

| SKU | Category | Brand / manufacturer | Form, W × H × D in | Price, cost, velocity | Score | Other |
|---|---|---|---|---|---|---|
| `SKU-A` | `CAT-RTE` | Acme / Acme Foods | `BOX` 8 × 8 × 6 | 4.00, 3.00, 30 | 30.0 | promo-eligible |
| `SKU-B` | `CAT-RTE` | Acme / Acme Foods | `BOX` 3 × 8 × 6 | 4.00, 3.00, 20 | 20.0 | promo-eligible |
| `SKU-C` | `CAT-RTE` | StoreBrand / StoreCo (private label) | `BOX` 4 × 8 × 6 | 3.00, 2.00, 10 | 10.0 | |
| `SKU-D` | `CAT-HOT` | Oatco / Oatco Mills | `BOX` 5 × 10 × 6 | 3.00, 2.50, 10 | 5.0 | `NEW_ITEM_TRIAL` |
| `SKU-E` | `CAT-RTE` | Acme / Acme Foods | `BOX` 4 × 16 × 6 | 5.00, 4.00, 5 | 5.0 | `DELIST_CANDIDATE`; too tall for 14 in |
| `SKU-F` | `CAT-RTE` | Acme / Acme Foods | `BOX` 4 × 8 × 6 | 4.00, 3.00, 8 | 8.0 | season `(9, 11)` |
| `SKU-G` | `CAT-HOT` | Oatco / Oatco Mills | `HANGING` 4 × 6 × 2 | 2.00, 1.50, 4 | 2.0 | |
| `SKU-H` | `CAT-HBA` | Bladeco / Bladeco Inc | `BOX` 6 × 10 × 4 | 5.00, 4.00, 6 | 6.0 | |
| `SKU-S1` | `CAT-SPI` | Spiceco / Spiceco Inc | `HANGING` 3 × 5 × 1 | 2.00, 1.00, 12 | 12.0 | |
| `SKU-S2` | `CAT-SPI` | Spiceco / Spiceco Inc | `HANGING` 3 × 5 × 1 | 2.00, 1.00, 6 | 6.0 | |
| `SKU-S3` | `CAT-SPI` | Spiceco / Spiceco Inc | `HANGING` 3 × 5 × 1 | 2.00, 1.00, 3 | 3.0 | |
| `SKU-Y1` | `CAT-YOG` | Dairyland / Dairyland Co | `BOX` 5 × 4 × 4, stackable, perishable | 3.00, 2.00, 40 | 40.0 | |
| `SKU-Y2` | `CAT-YOG` | StoreBrand / StoreCo (private label) | `BOX` 5 × 4 × 4, stackable, perishable | 2.50, 1.50, 10 | 10.0 | |
| `SKU-P1` | `CAT-APL` | Orchard / Orchard Growers | `LOOSE`, 0.5 sq ft per facing, 40 units per sq ft, perishable | 1.50, 1.00, 80 | 40.0 | |
| `SKU-P2` | `CAT-APL` | Orchard / Orchard Growers | `LOOSE`, 0.5 sq ft per facing, 30 units per sq ft, perishable | 1.75, 1.25, 40 | 20.0 | |

**Assignments (S1):** `RUN-1L1-E` and `RUN-1L1-B` → `CAT-CER`; `PEG-1` → `CAT-SPI`; `DCR-1` → `CAT-DY`; `TBL-1` → `CAT-PR`.

**Placements after S6** (window half-open; facings and units are derived):

| Id | SKU | Leaf | Type | Extent | Offset | Window | Agreement | Facings | Units |
|---|---|---|---|---|---|---|---|---|---|
| `PLC-000001` | P1 | `BIN-1` | PERMANENT | 8.0 sq ft | — | [01-05, —) | — | 16 | 320 |
| `PLC-000002` | Y1 | `RUN-D1-E` | PERMANENT | 30.0 in | 0.0 | [01-05, —) | — | 6 | 90 |
| `PLC-000003` | A | `RUN-1L1-E` | PERMANENT | 48.0 in | 0.0 | [01-05, —) | `AGR-000001` | 6 | 18 |
| `PLC-000004` | B | `RUN-1L1-B` | PERMANENT | 24.0 in | 0.0 | [01-05, 07-06) | `AGR-000001` | 8 | 24 |
| `PLC-000005` | P2 | `BIN-2` | PERMANENT | 8.0 sq ft | — | [01-05, —) | — | 16 | 240 |
| `PLC-000006` | S1 | `PEG-1` | PERMANENT | 3 hooks | — | [01-05, —) | — | 3 | 18 |
| `PLC-000007` | C | `RUN-1L1-B` | PERMANENT | 4.0 in | 24.0 | [01-05, —) | — | 1 | 3 |
| `PLC-000008` | Y2 | `RUN-D1-M` | PERMANENT | 5.0 in | 0.0 | [01-05, —) | — | 1 | 15 |
| `PLC-000009` | S2 | `PEG-1` | PERMANENT | 1 hook | — | [01-05, —) | — | 1 | 6 |
| `PLC-000010` | D | `RUN-1L1-B` | PERMANENT | 5.0 in | 28.0 | [01-05, —) | — | 1 | 3 |
| `PLC-000011` | B | `RUN-1L1-B` | PERMANENT | 24.0 in | 0.0 | [07-06, —) | — | 8 | 24 |
| `PLC-000012` | A | `FD-1` | PROMOTIONAL | 1.0 sq ft | — | [09-01, 09-29) | `AGR-000002` | 3 | 3 |
| `INC-000001` | B | (zone `SID-1L`) | INCIDENTAL | — | — | [09-10, —) | — | — | 24 held |

All dates are 2026. Unplaced by both plans: `SKU-F` (`OUT_OF_SEASON`), `SKU-H` (`NO_CATEGORY_SPACE`), `SKU-G` (`NO_SPACE_IN_PLANE`), `SKU-E` (`NO_DIMENSIONAL_FIT`), `SKU-S3` (`INSUFFICIENT_SPACE`). `SKU-A` is under target (6 facings of a target of 7; GEN-6).

#### 0.10.2 `conftest.py` (normative fixture code for Part 9; shipped with the tests)

```python
"""EX-STORE-1, the shared example fixture of sim-contract v0.3 (Part 0.10).

States S0..S6. Each fixture builds a fresh store and advances it with the
stated calls, so no test sees another test's changes.
"""
from contextlib import contextmanager
from datetime import date
from types import SimpleNamespace as Stub

import pytest

import supermarket_sim.predicates as P

from supermarket_sim.contracts import AssertionLevel, assertion_level
from supermarket_sim.model import (
    AgreementType, Area, AssortmentStatus, Bay, Bin, BlockingMode, BusinessUnit,
    CategoryNode, CategoryRole, ClipStrip, DateWindow, Door, DoorCaseRun,
    FeeBasis, FloorDisplay, Gondola, GondolaSide, NonBoxSpec, PegSection,
    ProduceTable, Product, ProductForm, ServiceCase, ShelfLevel, ShelfRun,
    ShelvingRun, SKU, SpaceType, SpaceUnit, Store, TemperatureZone, Tier,
    VersionKind,
)

T, L, F, Z = SpaceType, ShelfLevel, ProductForm, TemperatureZone

D_0102, D_0105 = date(2026, 1, 2), date(2026, 1, 5)
D_0701, D_0706 = date(2026, 7, 1), date(2026, 7, 6)
D_0901, D_0910, D_0915 = date(2026, 9, 1), date(2026, 9, 10), date(2026, 9, 15)
D_0929, D_1006, D_1015 = date(2026, 9, 29), date(2026, 10, 6), date(2026, 10, 15)
D_270104 = date(2027, 1, 4)
CALENDAR = (D_0105, D_0706, D_270104)
ALL_ROOTS = ("CAT-DG", "CAT-DY", "CAT-PR")


@contextmanager
def expect_violation(kind, clause_id):
    """Pass iff the block raises `kind` carrying `clause_id`."""
    with pytest.raises(kind) as info:
        yield info
    assert info.value.clause_id == clause_id, info.value.clause_id


def _runs(prefix, space_type, width, depth, clear):
    levels = (("T", L.TOP), ("E", L.EYE_LEVEL), ("M", L.MIDDLE), ("B", L.BOTTOM))
    return [ShelfRun(f"{prefix}-{c}", space_type, lvl, width, depth, clear)
            for c, lvl in levels]


def _areas():
    G = T.GONDOLA
    peg_1 = PegSection("PEG-1", hook_count=4, hook_depth_units=6,
                       location_description="Aisle 1, lower bay 2")
    clp_1 = ClipStrip("CLP-1", hook_count=6, hook_depth_units=3, frontage_in=2.0)
    sid_1l = GondolaSide(
        "SID-1L", "LEFT", "AISLE-1",
        bays=[Bay("BAY-1L1", G, 1, 48.0, _runs("RUN-1L1", G, 48.0, 18.0, 14.0)),
              Bay("BAY-1L2", G, 2, 48.0, _runs("RUN-1L2", G, 48.0, 18.0, 14.0))],
        pegs=[peg_1], clip_strips=[clp_1])
    sid_1f = GondolaSide(
        "SID-1F", "FRONT", "FRONT-AISLE",
        bays=[Bay("BAY-1F1", G, 1, 48.0, _runs("RUN-1F1", G, 48.0, 18.0, 14.0))])
    gdl_1 = Gondola("GDL-1", front_aisle="FRONT-AISLE", back_aisle="BACK-AISLE",
                    left_aisle="AISLE-1", right_aisle="AISLE-2", depth_in=48.0,
                    footprint_sqft=32.0, sides=[sid_1l, sid_1f])
    W = T.WALL_SHELF
    shr_1 = ShelvingRun("SHR-1", W, "Back wall", 3.0,
                        bays=[Bay("BAY-W1", W, 1, 36.0, _runs("RUN-W1", W, 36.0, 12.0, 12.0))])
    C = T.REACH_IN_COOLER
    dcr_1 = DoorCaseRun("DCR-1", C, "Dairy wall", 10.0,
                        doors=[Door("DOR-1", C, 1, 30.0, _runs("RUN-D1", C, 30.0, 20.0, 12.0))])
    svc_1 = ServiceCase("SVC-1", T.DELI_CASE, "Deli counter", 72.0, 24.0,
                        tiers=[Tier("TIER-1E", T.DELI_CASE, L.EYE_LEVEL, 72.0, 24.0, 10.0),
                               Tier("TIER-1B", T.DELI_CASE, L.BOTTOM, 72.0, 24.0, 10.0)])
    tbl_1 = ProduceTable("TBL-1", "Produce entrance", 24.0,
                         bins=[Bin("BIN-1", 1, 8.0), Bin("BIN-2", 2, 8.0)])
    fd_1 = FloorDisplay("FD-1", "Front racetrack", 24.0, 36.0)
    fd_2 = FloorDisplay("FD-2", "Lane 1 impulse rack", 12.0, 12.0)
    return [
        Area("AREA-SF", "Sales floor", T.SALES_FLOOR, 800.0,
             [gdl_1, shr_1, dcr_1, svc_1, tbl_1, fd_1,
              Area("AREA-AI", "Aisles", T.AISLE, 701.0, [])]),
        Area("AREA-BR", "Backroom", T.BACKROOM, 300.0, []),
        Area("AREA-CO", "Front end", T.CHECKOUT, 100.0,
             [fd_2, Area("AREA-CL", "Checkout lanes", T.CHECKOUT, 99.0, [])]),
    ]


DG_POLICY = dict(
    role=CategoryRole.ROUTINE, temperature_zone=Z.AMBIENT,
    allowed_space_types=frozenset({T.GONDOLA, T.WALL_SHELF, T.PEG_SECTION, T.CLIP_STRIP}),
    secured_case_required=False, min_facings_per_sku=1, max_facings_per_sku=6,
    space_elasticity=0.5, sales_per_linear_ft_per_week=10.0, category_captain=None,
    private_label_space_target_pct=20.0, reset_cycle_months=6,
    blocking_mode=BlockingMode.VERTICAL_BRAND, private_label_benchmark_brand=None)
DY_POLICY = {**DG_POLICY, "temperature_zone": Z.REFRIGERATED,
             "allowed_space_types": frozenset({T.REACH_IN_COOLER}),
             "private_label_benchmark_brand": "Dairyland"}
PR_POLICY = {**DG_POLICY, "allowed_space_types": frozenset({T.PRODUCE_TABLE})}


def _categories():
    cer = CategoryNode("CAT-CER", "Cereal", "category", {"max_facings_per_sku": 12},
                       [CategoryNode("CAT-RTE", "Ready-to-Eat", "subcategory", {}),
                        CategoryNode("CAT-HOT", "Hot Cereal", "subcategory", {})])
    dg = CategoryNode("CAT-DG", "Dry Grocery", "department", DG_POLICY,
                      [cer, CategoryNode("CAT-SPI", "Spices", "category", {}),
                       CategoryNode("CAT-HBA", "Health & Beauty", "category", {"secured_case_required": True})])
    dy = CategoryNode("CAT-DY", "Dairy", "department", DY_POLICY,
                      [CategoryNode("CAT-YOG", "Yogurt", "category", {})])
    pr = CategoryNode("CAT-PR", "Produce", "department", PR_POLICY,
                      [CategoryNode("CAT-APL", "Apples & Pears", "category", {})])
    return [dg, dy, pr]


def _business_units(cat):
    dept = [BusinessUnit(f"BU-{k}", n, "DEPARTMENT", root_category=cat[f"CAT-{k}"])
            for k, n in (("DG", "Dry Grocery"), ("DY", "Dairy"), ("PR", "Produce"))]
    return [BusinessUnit("BU-CO", "Example Co", "COMPANY",
                         children=[BusinessUnit("BU-ST", "Store 1", "STORE", children=dept)])]


#        sku    upc            name                brand        manufacturer      PL     perish form       W    H     D    upc_case stack
_PRODUCTS = [
    ("A",  "100000000001", "Crunch Family",   "Acme",       "Acme Foods",     False, False, F.BOX,     8.0, 8.0,  6.0, 12, False),
    ("B",  "100000000002", "Crunch Small",    "Acme",       "Acme Foods",     False, False, F.BOX,     3.0, 8.0,  6.0, 12, False),
    ("C",  "100000000003", "Crisp O's",       "StoreBrand", "StoreCo",        True,  False, F.BOX,     4.0, 8.0,  6.0, 12, False),
    ("D",  "100000000004", "Rolled Oats",     "Oatco",      "Oatco Mills",    False, False, F.BOX,     5.0, 10.0, 6.0, 12, False),
    ("E",  "100000000005", "Crunch Tall",     "Acme",       "Acme Foods",     False, False, F.BOX,     4.0, 16.0, 6.0, 12, False),
    ("F",  "100000000006", "Pumpkin O's",     "Acme",       "Acme Foods",     False, False, F.BOX,     4.0, 8.0,  6.0, 12, False),
    ("G",  "100000000007", "Oat Cup Peg",     "Oatco",      "Oatco Mills",    False, False, F.HANGING, 4.0, 6.0,  2.0, 24, False),
    ("H",  "100000000008", "Razor Cartridges", "Bladeco",   "Bladeco Inc",    False, False, F.BOX,     6.0, 10.0, 4.0, 8,  False),
    ("S1", "100000000011", "Cinnamon",        "Spiceco",    "Spiceco Inc",    False, False, F.HANGING, 3.0, 5.0,  1.0, 24, False),
    ("S2", "100000000012", "Paprika",         "Spiceco",    "Spiceco Inc",    False, False, F.HANGING, 3.0, 5.0,  1.0, 24, False),
    ("S3", "100000000013", "Saffron",         "Spiceco",    "Spiceco Inc",    False, False, F.HANGING, 3.0, 5.0,  1.0, 24, False),
    ("Y1", "100000000021", "Yogurt 6-pack",   "Dairyland",  "Dairyland Co",   False, True,  F.BOX,     5.0, 4.0,  4.0, 8,  True),
    ("Y2", "100000000022", "Yogurt 6-pack",   "StoreBrand", "StoreCo",        True,  True,  F.BOX,     5.0, 4.0,  4.0, 8,  True),
    ("P1", "100000000031", "Apples",          "Orchard",    "Orchard Growers", False, True, F.LOOSE,   3.0, 3.0,  3.0, 80, False),
    ("P2", "100000000032", "Pears",           "Orchard",    "Orchard Growers", False, True, F.LOOSE,   3.0, 3.0,  3.0, 60, False),
]
_SPECS = {"P1": NonBoxSpec({SpaceUnit.SQFT: 0.5}, {SpaceUnit.SQFT: 40.0}),
          "P2": NonBoxSpec({SpaceUnit.SQFT: 0.5}, {SpaceUnit.SQFT: 30.0})}

#        sku    category   price  cost  vel  status                            season   promo
_SKUS = [
    ("A",  "CAT-RTE", 4.00, 3.00, 30, AssortmentStatus.CORE,             None,    True),
    ("B",  "CAT-RTE", 4.00, 3.00, 20, AssortmentStatus.CORE,             None,    True),
    ("C",  "CAT-RTE", 3.00, 2.00, 10, AssortmentStatus.CORE,             None,    False),
    ("D",  "CAT-HOT", 3.00, 2.50, 10, AssortmentStatus.NEW_ITEM_TRIAL,   None,    False),
    ("E",  "CAT-RTE", 5.00, 4.00, 5,  AssortmentStatus.DELIST_CANDIDATE, None,    False),
    ("F",  "CAT-RTE", 4.00, 3.00, 8,  AssortmentStatus.CORE,             (9, 11), False),
    ("G",  "CAT-HOT", 2.00, 1.50, 4,  AssortmentStatus.CORE,             None,    False),
    ("H",  "CAT-HBA", 5.00, 4.00, 6,  AssortmentStatus.CORE,             None,    False),
    ("S1", "CAT-SPI", 2.00, 1.00, 12, AssortmentStatus.CORE,             None,    False),
    ("S2", "CAT-SPI", 2.00, 1.00, 6,  AssortmentStatus.CORE,             None,    False),
    ("S3", "CAT-SPI", 2.00, 1.00, 3,  AssortmentStatus.CORE,             None,    False),
    ("Y1", "CAT-YOG", 3.00, 2.00, 40, AssortmentStatus.CORE,             None,    False),
    ("Y2", "CAT-YOG", 2.50, 1.50, 10, AssortmentStatus.CORE,             None,    False),
    ("P1", "CAT-APL", 1.50, 1.00, 80, AssortmentStatus.CORE,             None,    False),
    ("P2", "CAT-APL", 1.75, 1.25, 40, AssortmentStatus.CORE,             None,    False),
]


def s0_build_args():
    """Fresh, unattached arguments for Store.build of S0 (Part 8 examples vary them)."""
    roots = _categories()
    cat = {n.id: n for r in roots for n in r.subtree()}
    products = {}
    for key, upc, name, brand, mfr, pl, per, form, w, h, d, upcase, stack in _PRODUCTS:
        products[key] = Product(upc, name, brand, mfr, pl, per, form, w, h, d,
                                units_per_case=upcase, stackable=stack, spec=_SPECS.get(key))
    skus = [SKU(f"SKU-{k}", products[k], cat[c], price, cost, vel,
                assortment_status=status, season=season, promo_display_eligible=promo)
            for k, c, price, cost, vel, status, season, promo in _SKUS]
    return dict(id="STORE-1", footprint_sqft=1200.0, areas=_areas(),
                products=list(products.values()), skus=skus, category_roots=roots,
                business_unit_roots=_business_units(cat), reset_calendar=CALENDAR)


def node(args, node_id):
    """Find a space node or SKU by id among unattached build arguments."""
    for s in args["skus"]:
        if s.sku_id == node_id:
            return s
    for a in args["areas"]:
        for n in a.subtree():
            if n.id == node_id:
                return n
    raise KeyError(node_id)


def build_s0():
    return Store.build(**s0_build_args())


def advance_s1(st):
    st.assign_space("RUN-1L1-E", "CAT-CER")
    st.assign_space("RUN-1L1-B", "CAT-CER")
    st.assign_space("PEG-1", "CAT-SPI")
    st.assign_space("DCR-1", "CAT-DY")
    st.assign_space("TBL-1", "CAT-PR")
    return st


def advance_s2(st):
    assert st.add_agreement(AgreementType.SLOTTING, "Acme Foods", FeeBasis.PER_LINEAR_FT,
                            2.0, DateWindow(D_0105, None)) == "AGR-000001"
    return st


def scope_all(st):
    return frozenset(st.category(c) for c in ALL_ROOTS)


def advance_s3(st):
    agr = st.agreements["AGR-000001"]
    plan = (st.generate_plan(scope_all(st), D_0105)
            .with_agreement(st.skus["SKU-A"], agr)
            .with_agreement(st.skus["SKU-B"], agr))
    st.set_working_plan(plan)
    assert st.release_working_plan(VersionKind.CYCLE, D_0102, D_0105) == 1
    return st


def advance_s4(st):
    st.execute_version(1, D_0105)
    return st


def advance_s5(st):
    plan = st.generate_plan(scope_all(st), D_0706).with_agreement(
        st.skus["SKU-A"], st.agreements["AGR-000001"])
    st.set_working_plan(plan)
    assert st.release_working_plan(VersionKind.CYCLE, D_0701, D_0706) == 2
    st.execute_version(2, D_0706)
    return st


def advance_s6(st):
    assert st.add_agreement(AgreementType.PROMOTIONAL_DISPLAY, "Acme Foods",
                            FeeBasis.PER_DISPLAY, 500.0,
                            DateWindow(D_0901, D_1006)) == "AGR-000002"
    assert st.add_promotional_placement("SKU-A", "FD-1", 1.0, None, D_0901, D_0929,
                                        "AGR-000002") == "PLC-000012"
    assert st.add_incidental_placement("SKU-B", 24, D_0910, None, "SID-1L",
                                       None) == "INC-000001"
    return st


_STEPS = (advance_s1, advance_s2, advance_s3, advance_s4, advance_s5, advance_s6)


def store_at(n):
    st = build_s0()
    for step in _STEPS[:n]:
        st = step(st)
    return st


@pytest.fixture(autouse=True)
def _level_all():
    with assertion_level(AssertionLevel.ALL):
        yield


@pytest.fixture
def s0(): return store_at(0)
@pytest.fixture
def s1(): return store_at(1)
@pytest.fixture
def s2(): return store_at(2)
@pytest.fixture
def s3(): return store_at(3)
@pytest.fixture
def s4(): return store_at(4)
@pytest.fixture
def s5(): return store_at(5)
@pytest.fixture
def s6(): return store_at(6)
```

Expected values used throughout: at S6 on `D_0915`, the store has `116.0` linear inches, `17.0` sq ft, and `4` hooks allocated; 11 committed placements live with 62 facings and 740 units of capacity. On `D_1015` (after the promotion), 10 placements, 59 facings, 737 units, `16.0` sq ft.

---
## Part 1 — Enumerations, value types, and module functions

| Enumeration | Values, in declaration order | Grounding |
|---|---|---|
| `SpaceUnit` | `LINEAR_IN`, `SQFT`, `HOOKS` | `[VS 7, 8]` |
| `PresentationPlane` | `FRONT`, `TOP`, `PEG` | `[VS 7, 8]` |
| `ShelfLevel` | `TOP`, `EYE_LEVEL`, `MIDDLE`, `BOTTOM` | `[DISC]` (4 levels); eye level `[TB 2.3]` |
| `TemperatureZone` | `AMBIENT`, `REFRIGERATED`, `FROZEN` | `[TB 1.3, 4.4]` |
| `ProductForm` | `BOX`, `SOFT_PACK`, `HANGING`, `LOOSE` | `[VS 8]`; values `[SIM]` |
| `PlacementType` | `PERMANENT`, `PROMOTIONAL`, `INCIDENTAL` | `[VS 3]` |
| `AgreementType` | `SLOTTING`, `PROMOTIONAL_DISPLAY`, `PAY_TO_STAY`, `OPPORTUNISTIC_BUY` | `[VS 3]` |
| `FeeBasis` | `PER_FACING`, `PER_LINEAR_FT`, `PER_DISPLAY`, `FLAT` | `[VS 3]`; `FLAT` `[SIM]` |
| `CategoryRole` | `DESTINATION`, `ROUTINE`, `OCCASIONAL_SEASONAL`, `CONVENIENCE` | `[TB 2.1]` |
| `AssortmentStatus` | `CORE`, `NEW_ITEM_TRIAL`, `DELIST_CANDIDATE`, `DELISTED` | `[TB 2.6]`; `DELISTED` `[SIM]` |
| `BlockingMode` | `VERTICAL_BRAND`, `HORIZONTAL_TIER` | `[VS 12]` |
| `VersionKind` | `CYCLE`, `REVISION` | `[VS 10]` |
| `VersionStatus` | `RELEASED`, `EXECUTED` | `[VS 10]` |
| `UnplacedReason` | `OUT_OF_SEASON`, `NO_CATEGORY_SPACE`, `NO_SPACE_IN_PLANE`, `NO_DIMENSIONAL_FIT`, `INSUFFICIENT_SPACE` | `[SIM]` |
| `SpaceType` | the nodes of the type tree (2.2) | `[VS 6]` |
| `AssertionLevel` | `NONE`, `REQUIRE`, `ENSURE`, `INVARIANT`, `ALL` (in `supermarket_sim.contracts`) | `[DbC]` |

| ID | Clause | Grounding |
|---|---|---|
| ENUM-1 | `plane_unit(plane)` is total: `FRONT → LINEAR_IN`, `TOP → SQFT`, `PEG → HOOKS`. Its inverse is `plane_of(unit)`. | `[VS 7, 8]` |
| ENUM-2 | `ShelfLevel.preference_rank`: `EYE_LEVEL` = 1, `MIDDLE` = 2, `TOP` = 3, `BOTTOM` = 4 (lower is better). Eye level and the level just below carry the premium; `TOP` before `BOTTOM` is `[SIM]` | `[VS 12]` `[TB 2.3]` `[SIM]` |
| ENUM-3 | `compatible(agreement_type, placement_type)` is true exactly for (`SLOTTING`, `PERMANENT`), (`PAY_TO_STAY`, `PERMANENT`), (`PROMOTIONAL_DISPLAY`, `PROMOTIONAL`), (`OPPORTUNISTIC_BUY`, `INCIDENTAL`). Placement type and agreement type are related but separate fields | `[VS 3]` |
| ENUM-4 | `permitted_forms(plane)`: `FRONT → {BOX, SOFT_PACK}`; `TOP → {BOX, SOFT_PACK, LOOSE}`; `PEG → {HANGING}`. `permitted_planes(product)` is PROD-Q-1 | `[SIM]` |
| ENUM-5 | `UnplacedReason.precedence` is the declaration order, 1 … 5. When several reasons are true for a SKU, the one with the lowest precedence is reported (GEN-4), so the reasons are mutually exclusive | `[SIM]` |
| ENUM-6 | `ShelfLevel.physical_order`: `TOP` = 1, `EYE_LEVEL` = 2, `MIDDLE` = 3, `BOTTOM` = 4, top to bottom | `[DISC]` `[SIM]` |

**Examples (ENUM-1 … ENUM-6)**

```python
# ✓ ENUM-1
assert plane_unit(PresentationPlane.TOP) is SpaceUnit.SQFT and plane_of(SpaceUnit.HOOKS) is PresentationPlane.PEG
# ✓ ENUM-2, ENUM-6: rank is by value, order is by position
assert ShelfLevel.EYE_LEVEL.preference_rank == 1 and ShelfLevel.TOP.preference_rank == 3
assert [l.physical_order for l in (ShelfLevel.TOP, ShelfLevel.BOTTOM)] == [1, 4]
# ✓ ENUM-3
assert compatible(AgreementType.SLOTTING, PlacementType.PERMANENT)
assert not compatible(AgreementType.SLOTTING, PlacementType.PROMOTIONAL)
# ✓ ENUM-4: a hanging product goes on a peg, never on a front-facing shelf
assert permitted_forms(PresentationPlane.PEG) == {ProductForm.HANGING}
# ✓ ENUM-5
assert UnplacedReason.OUT_OF_SEASON.precedence == 1 and UnplacedReason.INSUFFICIENT_SPACE.precedence == 5
```

### 1.1 `DateWindow` (value)

```
value DateWindow
creation  DateWindow(start: date, end: date | None)
queries   start, end; contains(d: date) -> bool; overlaps(other: DateWindow) -> bool
```

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| DW-NEW-PRE-1 | require | `end is None or start < end` — a window is never empty | `[VS 14]` `[SIM]` |
| DW-NEW-1 | ensure | `self.start == start and self.end == end` | `[DbC]` |
| DW-Q-1 | ensure | `contains(d)`: `result == (start <= d and (end is None or d < end))` (rule 10) | `[VS 4]` `[SIM]` |
| DW-Q-2 | ensure | `overlaps(o)`: `result == ((o.end is None or start < o.end) and (end is None or o.start < end))` — the two share at least one date | `[VS 14]` |
| DW-Q-3 | ensure | `window_covers(a, b)` (module function): `result == (a.start <= b.start and (a.end is None or (b.end is not None and b.end <= a.end)))` | `[VS 3]` |

**Examples (DateWindow)**

```python
w = DateWindow(D_0901, D_0929)
# ✓ DW-NEW-1, DW-Q-1: half-open
assert w.contains(D_0915) and not w.contains(D_0929)
# ✓ DW-Q-2: touching windows do not overlap
assert not w.overlaps(DateWindow(D_0929, None)) and w.overlaps(DateWindow(D_0910, D_0915))
# ✓ DW-Q-3
assert window_covers(DateWindow(D_0901, D_1006), w) and not window_covers(w, DateWindow(D_0901, None))
# ✗ DW-NEW-PRE-1: end precedes start
with expect_violation(PreconditionViolation, "DW-NEW-PRE-1"):
    DateWindow(D_0929, D_0901)
# ✗ supplier DW-Q-1: a closed interval that contains its end date
assert not P.DW_Q_1(w, True, None, d=D_0929)
```

### 1.2 `add_months` (module function) `[SIM]`

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| ADDM-PRE-1 | require | `n >= 0` | `[SIM]` |
| ADDM-1 | ensure | `(result.year * 12 + result.month) == (d.year * 12 + d.month + n)` and `result.day == min(d.day, days_in_month(result.year, result.month))` — a month-end date stays at month end | `[SIM]` |

**Examples (add_months)**

```python
# ✓ ADDM-1
assert add_months(D_0105, 6) == date(2026, 7, 5)
assert add_months(date(2026, 8, 31), 6) == date(2027, 2, 28)
# ✗ ADDM-PRE-1
with expect_violation(PreconditionViolation, "ADDM-PRE-1"):
    add_months(D_0105, -1)
# ✗ supplier ADDM-1: rolling an overflowing day into the next month
assert not P.ADDM_1(None, date(2027, 3, 3), None, d=date(2026, 8, 31), n=6)
```

---
## Part 2 — The shared hierarchy shape and rollups `[VS 6, 9]`

The model has **four** recursive hierarchies of arbitrary depth: physical **space**, **space type**, **business unit**, and product **category**. All four conform to `HierarchyNode`. The physical hierarchy says where things sit and nest. The type hierarchy says what kind of thing each one is and which attributes apply. The two are crossed. `[VS 6]`

### 2.1 Deferred class `HierarchyNode`

```
deferred class HierarchyNode
queries
    id: str                                   -- unique within its hierarchy
    parent: HierarchyNode | None              -- None for a root and for a node not yet given to a container
    children: tuple[HierarchyNode, ...]       -- ordered; fixed by the creation procedure
    is_leaf: bool
    path: tuple[str, ...]                     -- ids from the root down to self, inclusive
    depth: int                                -- len(path) - 1
    ancestors() -> tuple[HierarchyNode, ...]  -- parent, grandparent, ..., root
    root() -> HierarchyNode
    subtree() -> frozenset[HierarchyNode]     -- self and all descendants
    leaves() -> tuple[HierarchyNode, ...]     -- leaf descendants in child order; (self,) for a leaf
    is_within(other: HierarchyNode) -> bool   -- self is other or a descendant of other
```

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| HN-INV-1 | invariant | `self.is_leaf == (len(self.children) == 0)` | `[DbC]` |
| HN-INV-2 | invariant | `all(c.parent is self for c in self.children)` and `(self.parent is None or self in self.parent.children)` | `[DbC]` |
| HN-INV-3 | invariant | `self not in self.ancestors()` — acyclic. Bottom-up creation (0.8) makes this true by construction | `[VS 14]` |
| HN-INV-4 | invariant | ids are unique within the subtree of `self.root()` | `[DbC]` |
| HN-Q-1 | ensure | `path`: `result == tuple(a.id for a in reversed(self.ancestors())) + (self.id,)`; `depth == len(path) - 1` | `[DbC]` |
| HN-Q-2 | ensure | `ancestors()`: `result == () if self.parent is None else (self.parent,) + self.parent.ancestors()` | `[DbC]` |
| HN-Q-3 | ensure | `subtree()`: `result == frozenset({self}).union(*(c.subtree() for c in self.children))` | `[DbC]` |
| HN-Q-4 | ensure | `leaves()`: `result == (self,) if self.is_leaf else tuple(l for c in self.children for l in c.leaves())` | `[DbC]` |
| HN-Q-5 | ensure | `is_within(o)`: `result == (o.id in self.path and o.root() is self.root())` | `[DbC]` |
| HN-Q-6 | ensure | `root()`: `result is (self.ancestors()[-1] if self.ancestors() else self)` | `[DbC]` |
| HN-IMPL-1 | review | `subtree()` membership and "all nodes under X" are answered by an indexed lookup on `path` (path enumeration) or a closure table, not by a tree walk per call | `[VS 6]` |

The four hierarchies are **structurally static** after construction in this version: no command re-parents, adds, or removes a node. `[SIM]` (Q-14)

**Examples (HierarchyNode)**

```python
run = s0.space("RUN-1L1-E")
# ✓ HN-Q-1, HN-Q-6
assert run.path == ("STORE-1", "AREA-SF", "GDL-1", "SID-1L", "BAY-1L1", "RUN-1L1-E") and run.depth == 5
assert run.root() is s0 and s0.category("CAT-RTE").root().id == "CAT-DG"
# ✓ HN-Q-2
assert [a.id for a in run.ancestors()] == ["BAY-1L1", "SID-1L", "GDL-1", "AREA-SF", "STORE-1"]
# ✓ HN-Q-3: a bay and its four runs
assert {n.id for n in s0.space("BAY-1L1").subtree()} == {"BAY-1L1", "RUN-1L1-T", "RUN-1L1-E", "RUN-1L1-M", "RUN-1L1-B"}
# ✓ HN-Q-4: bays first, then pegs, then clip strips (creation order)
assert [l.id for l in s0.space("SID-1L").leaves()][-3:] == ["RUN-1L2-B", "PEG-1", "CLP-1"]
# ✓ HN-Q-5
assert run.is_within(s0.space("GDL-1")) and not run.is_within(s0.space("SID-1F"))
# ✗ supplier HN-Q-1: path listed leaf first
assert not P.HN_Q_1(run, ("RUN-1L1-E", "BAY-1L1", "SID-1L", "GDL-1", "AREA-SF", "STORE-1"), None)
# ✗ supplier HN-Q-4: leaves out of child order
assert not P.HN_Q_4(s0.space("BAY-1L1"), tuple(reversed(s0.space("BAY-1L1").leaves())), None)
# ✗ inv HN-INV-3: a stub that is its own grandparent (no constructor can build it)
a, b = Stub(id="X", children=()), Stub(id="Y", children=())
a.parent, b.parent = b, a
a.ancestors = lambda: (b, a)
assert not P.HN_INV_3(a)
# ✗ inv HN-INV-1: a stub that reports is_leaf while it has a child
assert not P.HN_INV_1(Stub(is_leaf=True, children=(run,)))
```

`Stub(**queries)` is a plain attribute holder (for example `types.SimpleNamespace`); MON-9 makes it a valid stand-in.

### 2.2 The space-type hierarchy `[VS 6]`

`SpaceType` is an enumeration whose members form a fixed tree; each member conforms to `HierarchyNode` (its `id` is its name). Every space node has exactly one `space_type`. `merchandisable` and `temperature_zone` are derived from it.

```
SPACE
├─ SALES_FLOOR                          (container area)
├─ MERCHANDISING
│   ├─ OPEN_SHELVING     ← GONDOLA, WALL_SHELF, BAKERY_SHELVING, FLORAL_SHELVING
│   ├─ DOOR_CASE         ← REACH_IN_COOLER, REACH_IN_FREEZER, FLORAL_COOLER
│   ├─ SERVICE_CASE      ← DELI_CASE, SERVICE_REFRIGERATED_CASE
│   ├─ PRODUCE_TABLE
│   ├─ FLOOR_DISPLAY
│   ├─ PEG_SECTION
│   └─ CLIP_STRIP
└─ NON_MERCHANDISING
    └─ BACKROOM, PREP_AREA, CHECKOUT, AISLE, RESTROOM, OFFICE, OTHER
```

`OPEN_SHELVING`, `DOOR_CASE`, `SERVICE_CASE`, `PRODUCE_TABLE`, and `FLOOR_DISPLAY` are the five physical archetypes of `[DISC]`. `PEG_SECTION` and `CLIP_STRIP` are two more `[VS 7]`.

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| TYP-1 | ensure | `merchandisable(t) == t.is_within(MERCHANDISING)`, total on `SpaceType` | `[VS 6]` |
| TYP-2 | ensure | `temperature_zone(t)` is total: `REACH_IN_COOLER`, `FLORAL_COOLER`, `DELI_CASE`, `SERVICE_REFRIGERATED_CASE` → `REFRIGERATED`; `REACH_IN_FREEZER` → `FROZEN`; every other type → `AMBIENT`. `PRODUCE_TABLE` = `AMBIENT` is `[SIM]` (Q-11) | `[TB 4.4]` `[SIM]` |
| TYP-3 | invariant | A checkout **impulse rack** is a `FLOOR_DISPLAY` whose parent is an `Area` of type `CHECKOUT`. `SALES_FLOOR` and `CHECKOUT` are the only non-merchandising types whose areas may contain merchandising fixtures (SN-INV-1, AREA-INV-3) | `[VS 6]` `[SIM]` |

**Examples (space types)**

```python
# ✓ TYP-1, TYP-2
assert merchandisable(SpaceType.GONDOLA) and not merchandisable(SpaceType.CHECKOUT)
assert temperature_zone(SpaceType.REACH_IN_COOLER) is TemperatureZone.REFRIGERATED
assert temperature_zone(SpaceType.PRODUCE_TABLE) is TemperatureZone.AMBIENT
assert SpaceType.WALL_SHELF.is_within(SpaceType.OPEN_SHELVING) and SpaceType.WALL_SHELF.parent is SpaceType.OPEN_SHELVING
# ✓ TYP-3: FD-2 is the impulse rack in the CHECKOUT area AREA-CO
assert s0.space("FD-2").parent.space_type is SpaceType.CHECKOUT
# ✗ supplier TYP-2: treating a deli case as ambient
assert not P.TYP_2(None, TemperatureZone.AMBIENT, None, t=SpaceType.DELI_CASE)
# ✗ inv TYP-3: a floor display under a BACKROOM area (stub)
assert not P.TYP_3(Stub(space_type=SpaceType.FLOOR_DISPLAY,
                        parent=Stub(space_type=SpaceType.BACKROOM)))
```

### 2.3 Deferred class `Rollup` and the measures `[VS 9, 14]`

Every non-leaf node of the space, category, and business-unit hierarchies aggregates its children. The same machinery serves all three ("the cube"): three views over the same leaf facts, which are the committed placements.

```
deferred class Rollup
queries
    capacity(u: SpaceUnit) -> float
    allocated_amount(u: SpaceUnit, on: date) -> float
    unallocated_amount(u: SpaceUnit, on: date) -> float
    facings(on: date) -> int
    units_capacity(on: date) -> int
    placement_count(on: date) -> int
    own(measure: str, *args) -> float          -- own_m of RU-1, e.g. own("facings", on)
    rollup_dates() -> frozenset[date]          -- the containing store's change_dates()
```

| Family | Measures | Space node `n` | Category node `n` | Business unit `n` |
|---|---|---|---|---|
| **Space measures** | `capacity(u)`, `allocated_amount(u, on)`, `unallocated_amount(u, on)` | over `n`'s allocatable leaves | over leaves assigned to any node in `n.subtree()` | sum over the unit's descendants; a `DEPARTMENT` unit counts its root category |
| **Placement measures** | `facings(on)`, `units_capacity(on)`, `placement_count(on)` | committed placements live on `on` in `n`'s leaves | committed placements live on `on` whose SKU's category is in `n.subtree()` | as for space measures |

Only committed placements (`Placement`, 5.2) count. Incidental placements (5.3) have no space commitment and are excluded. `[VS 3]`

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| RU-1 | invariant | **Additivity.** For every non-leaf node `n` and every additive measure `m` (`capacity(u)`, `allocated_amount(u, on)`, `facings(on)`, `units_capacity(on)`, `placement_count(on)`): `m(n) == own_m(n) + sum(m(c) for c in n.children)`. `own_m` is `0` for a non-leaf space node; for a category node it is the measure over the leaves assigned exactly to `n` (space family) or over placements of SKUs whose category is exactly `n` (placement family); for a business unit it is `0`, except a `DEPARTMENT` unit, whose own contribution is its root category's total. `own(m, *args)` returns `own_m(n)`. Checked at every date in `rollup_dates()` | `[VS 9, 14]` |
| RU-2 | ensure | `unallocated_amount(u, on) == capacity(u) - allocated_amount(u, on)` | `[VS 9]` |
| RU-3 | invariant | `0 <= allocated_amount(u, on) <= capacity(u) + EPS` for every `u`, at every date in `rollup_dates()` | `[VS 14]` |
| RU-4 | invariant | **No mixed units.** No rollup adds quantities of different `SpaceUnit`s; every space measure takes the unit as an argument. Cross-plane comparison uses the unit-free placement measures | `[VS 8]` |

**Reserved measures.** Revenue, gross margin, and sales per linear foot attach at the placement and must be additive under RU-1. They belong to the Sales section (Part 10). `[VS 9]`

**Examples (Rollup)** — values at S6

```python
gdl, sid = s6.space("GDL-1"), s6.space("SID-1L")
# ✓ RU-1: children sum to the parent
assert gdl.allocated_amount(SpaceUnit.LINEAR_IN, D_0915) == sum(
    c.allocated_amount(SpaceUnit.LINEAR_IN, D_0915) for c in gdl.children) == 81.0
assert s6.category("CAT-DG").facings(D_0915) == 23 == (
    s6.category("CAT-CER").facings(D_0915) + s6.category("CAT-SPI").facings(D_0915)
    + s6.category("CAT-HBA").facings(D_0915))                          # 19 + 4 + 0
# ✓ RU-2
assert sid.unallocated_amount(SpaceUnit.LINEAR_IN, D_0915) == 384.0 - 81.0
# ✓ RU-4: inches, square feet, and hooks stay apart
assert (s6.capacity(SpaceUnit.LINEAR_IN), s6.capacity(SpaceUnit.SQFT), s6.capacity(SpaceUnit.HOOKS)) == (984.0, 23.0, 10.0)
# ✗ supplier RU-2: unallocated reported as capacity
assert not P.RU_2(sid, 384.0, None, u=SpaceUnit.LINEAR_IN, on=D_0915)
# ✗ inv RU-1: a node whose children do not sum to it (stub)
assert not P.RU_1(Stub(children=(Stub(facings=lambda on: 2),), own=lambda m, *a: 0,
                       facings=lambda on: 3, rollup_dates=lambda: frozenset({D_0915})))
```

The stub provides only `facings`; the `RU_1` predicate checks each measure the node provides (MON-9).

---
## Part 3 — Physical space

### 3.1 Deferred class `SpaceNode` (conforms to `HierarchyNode` and `Rollup`) `[DISC]` `[VS 6]`

Every physical space class, `Area`, and `Store` conforms to `SpaceNode`. The physical hierarchy covers the **whole building footprint**, not only merchandising space, so unallocated merchandising space is a filter and the total still reconciles to the building. `[VS 6]`

```
deferred class SpaceNode
queries
    space_type: SpaceType
    merchandisable: bool                          -- == merchandisable(space_type)
    footprint_sqft: float | None                  -- Store, Area, and top-level fixtures; None for fixture parts
    temperature_zone: TemperatureZone             -- == temperature_zone(space_type)
    own_assignment: CategoryNode | None           -- assigned on this node itself (mutable, 7.3)
    assigned_category: CategoryNode | None        -- effective: nearest ancestor-or-self with an own_assignment
    allocatable_leaves() -> tuple[LeafSpace, ...]
    placements_on(on: date) -> frozenset[Placement]
```

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| SN-Q-1 | ensure | `allocatable_leaves()`: `result == tuple(l for l in self.leaves() if isinstance(l, LeafSpace))` | `[DISC]` |
| SN-Q-2 | ensure | `capacity(u)`: `result == sum(l.leaf_capacity for l in self.allocatable_leaves() if l.unit == u)` | `[DISC]` `[VS 9]` |
| SN-Q-3 | ensure | `allocated_amount(u, on)`: `result == sum(l.leaf_allocated(on) for l in self.allocatable_leaves() if l.unit == u)` | `[DISC]` `[VS 9]` |
| SN-Q-4 | ensure | `placements_on(on)`: `result == frozenset(p for l in self.allocatable_leaves() for p in l.placements if p.window.contains(on))` | `[VS 4]` |
| SN-Q-5 | ensure | `facings(on) == sum(p.facings for p in placements_on(on))`; `units_capacity(on) == sum(p.units_capacity for p in placements_on(on))`; `placement_count(on) == len(placements_on(on))` | `[VS 8, 9]` |
| SN-Q-6 | ensure | `assigned_category`: `result is self.own_assignment if self.own_assignment is not None else (None if self.parent is None else self.parent.assigned_category)` | `[SIM]` |
| SN-INV-1 | invariant | `not self.space_type.is_within(NON_MERCHANDISING) or self.space_type == CHECKOUT or len(self.allocatable_leaves()) == 0` — non-merchandising space holds no allocatable leaf; a `CHECKOUT` area may hold impulse racks (TYP-3) | `[VS 6]` `[SIM]` |
| SN-INV-2 | invariant | `self.own_assignment is None or (self.merchandisable and not any(a.own_assignment is not None for a in self.ancestors()) and not any(d.own_assignment is not None for d in self.subtree() if d is not self))` — at most one assignment on any root-to-leaf path | `[SIM]` |
| SN-INV-3 | invariant | `self.parent is None or isinstance(self.parent, (Area, Store)) or self.space_type == self.parent.space_type or (self.space_type in {PEG_SECTION, CLIP_STRIP} and isinstance(self.parent, (GondolaSide, ShelvingRun)))` — a fixture's parts share its type; only peg sections and clip strips hang off a face of another type | `[DISC]` `[VS 7]` |
| SN-INV-4 | invariant | `self.id` starts with the prefix of its class (ID-1) | `[DISC]` `[SIM]` |

RU-1 through RU-3 apply to `SpaceNode`. Each face of a gondola is its own `GondolaSide` node and is allocated separately: two faces, two aisles, possibly two categories. `[VS 7]`

**Examples (SpaceNode)** — at S6 unless stated

```python
sid, run_b = s6.space("SID-1L"), s6.space("RUN-1L1-B")
# ✓ SN-Q-1: 8 runs, the peg section, and the clip strip
assert len(sid.allocatable_leaves()) == 10
# ✓ SN-Q-2, SN-Q-3
assert sid.capacity(SpaceUnit.LINEAR_IN) == 384.0 and sid.capacity(SpaceUnit.HOOKS) == 10.0
assert sid.allocated_amount(SpaceUnit.HOOKS, D_0915) == 4.0
# ✓ SN-Q-4: the shelf on 2026-09-15 vs. 2026-03-01 (PLC-000004 was there, PLC-000011 was not yet)
assert {p.placement_id for p in run_b.placements_on(D_0915)} == {"PLC-000007", "PLC-000010", "PLC-000011"}
assert {p.placement_id for p in run_b.placements_on(date(2026, 3, 1))} == {"PLC-000004", "PLC-000007", "PLC-000010"}
# ✓ SN-Q-5
assert (sid.facings(D_0915), sid.units_capacity(D_0915), sid.placement_count(D_0915)) == (20, 72, 6)
# ✓ SN-Q-6: inherited from the node that owns the assignment
assert s6.space("DOR-1").assigned_category.id == "CAT-DY" and s6.space("RUN-D1-B").own_assignment is None
assert s6.space("RUN-1L2-E").assigned_category is None
# ✗ supplier SN-Q-6: an assignment that does not flow down
assert not P.SN_Q_6(s6.space("RUN-D1-B"), None, None)
# ✗ inv SN-INV-1: a backroom area holding a shelf (stub)
assert not P.SN_INV_1(Stub(space_type=SpaceType.BACKROOM, allocatable_leaves=lambda: (run_b,)))
# ✗ inv SN-INV-2: two assignments on one path (stub of a run assigned under an assigned bay)
assert not P.SN_INV_2(Stub(own_assignment=s6.category("CAT-RTE"), merchandisable=True,
                           ancestors=lambda: (Stub(own_assignment=s6.category("CAT-CER")),),
                           subtree=lambda: frozenset()))
# ✗ inv SN-INV-3: a cooler run under a gondola bay (stub)
assert not P.SN_INV_3(Stub(space_type=SpaceType.REACH_IN_COOLER, parent=s6.space("BAY-1L1")))
```

`SID-1L` on `D_0915`: facings 6 (A) + 8 (B) + 1 (C) + 1 (D) + 3 (S1) + 1 (S2) = 20; units 18 + 24 + 3 + 3 + 18 + 6 = 72; six placements.

---

### 3.2 Deferred class `LeafSpace` (conforms to `SpaceNode`) — the allocatable unit `[DISC]` `[VS 7, 8]`

A leaf is the smallest unit a committed placement can occupy: `ShelfRun`, `Tier`, `Bin`, `FloorDisplay`, `PegSection`, or `ClipStrip`. The linear position of a placement on a leaf is a **segment** of the leaf, given by `offset_in`. A segment is not a node. `[VS 7]`

```
deferred class LeafSpace
queries
    presentation_plane: PresentationPlane         -- which surface the shopper sees; a property of the SPACE
    unit: SpaceUnit                               -- == plane_unit(presentation_plane)
    leaf_capacity: float                          -- in `unit`
    placements: tuple[Placement, ...]             -- every committed placement made here, incl. ended, ordered by (start_date, offset_in or 0, placement_id)
    secured: bool                                 -- locked/secured space [TB 6.4]
    shelf_level: ShelfLevel | None                -- FRONT-plane leaves only
    usable_depth_in: float | None
    clear_height_in: float | None
    hook_depth_units: int | None                  -- PEG plane only
    respace_without_reset: bool                   -- True where moving a placement takes seconds [VS 7]
    leaf_allocated(on: date) -> float
    peak_allocated(window: DateWindow, without: Placement | None = None) -> float
    free_intervals(window: DateWindow, without: Placement | None = None) -> tuple[tuple[float, float], ...]
    room(window: DateWindow, without: Placement | None = None) -> float
    can_hold(product: Product, extent: float, window: DateWindow) -> bool
    block_group() -> SpaceNode                    -- BLK-1
    bay_position() -> int                         -- BLK-2
```

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| LS-INV-1 | invariant | `self.is_leaf` (strengthens `HierarchyNode`) | `[DISC]` |
| LS-INV-2 | invariant | `self.leaf_capacity > 0` | `[DISC]` |
| LS-INV-3 | invariant | `self.unit == plane_unit(self.presentation_plane)` (ENUM-1) | `[VS 8]` |
| LS-INV-4 | invariant | `all(p.space is self for p in self.placements)` | `[DbC]` |
| LS-INV-5 | invariant | **No over-subscription.** For every `d` in `{p.start_date for p in self.placements}`: `self.leaf_allocated(d) <= self.leaf_capacity + EPS`. Allocation is piecewise constant and only rises at a start date, so these dates suffice | `[VS 14]` |
| LS-INV-6 | invariant | **No overlap in space and time.** For `LINEAR_IN` leaves: for every pair `p != q` in `placements` with `p.window.overlaps(q.window)`, `[p.offset_in, p.offset_in + p.extent)` and `[q.offset_in, q.offset_in + q.extent)` are disjoint | `[VS 14]` `[TB 2.2]` |
| LS-INV-7 | invariant | `self.temperature_zone == temperature_zone(self.space_type)` and `(self.assigned_category is None or self.temperature_zone == self.assigned_category.policy().temperature_zone)` | `[TB 1.3, 4.4]` |
| LS-INV-8 | invariant | `self.assigned_category is None or not self.assigned_category.policy().secured_case_required or self.secured` | `[TB 6.4]` |
| LS-INV-9 | invariant | `presentation_plane == FRONT ⇒ (usable_depth_in is not None and clear_height_in is not None)`; `presentation_plane == PEG ⇒ (hook_depth_units is not None and hook_depth_units >= 1 and usable_depth_in is None and clear_height_in is None)` | `[VS 7, 8]` |
| LS-INV-10 | invariant | `self.shelf_level is None or self.presentation_plane == FRONT` | `[DISC]` |
| LS-INV-11 | invariant | `self.presentation_plane != FRONT or self.shelf_level is not None` — every front-facing leaf has a shelf level, so every leaf of one plane is either ranked (`FRONT`) or unranked (`TOP`, `PEG`) | `[DISC]` `[SIM]` |
| LS-Q-1 | ensure | `leaf_allocated(on)`: `result == sum(p.extent for p in self.placements if p.window.contains(on))` | `[VS 8]` |
| LS-Q-2 | ensure | `peak_allocated(w, x)`: with `Q = [p for p in self.placements if p is not x]`, `result == max(sum(p.extent for p in Q if p.window.contains(d)) for d in {w.start} | {p.start_date for p in Q if w.contains(p.start_date)})` | `[VS 14]` |
| LS-PRE-1 | require | `free_intervals(w, x)`: `self.unit == LINEAR_IN` | `[VS 8]` |
| LS-Q-3 | ensure | `free_intervals(w, x)`: `result` is the ordered, maximal list of `(a, b)` gaps with `a < b` in `[0, leaf_capacity]` not covered by `[p.offset_in, p.offset_in + p.extent)` of any `p is not x` in `placements` with `p.window.overlaps(w)` | `[VS 8]` |
| LS-Q-4 | ensure | `room(w, x)`: `result == max((b - a for a, b in self.free_intervals(w, x)), default=0.0)` for a `LINEAR_IN` leaf, else `self.leaf_capacity - self.peak_allocated(w, x)`. For a `Bin` (BIN-INV-2): `self.leaf_capacity` if no `p is not x` overlaps `w`, else `0.0` | `[VS 8]` `[SIM]` |
| LS-PRE-2 | require | `can_hold(prod, x, w)`: `x > 0` | `[VS 8]` |
| LS-Q-5 | ensure | `can_hold(prod, x, w)`: `result == (fits_dimensionally(prod, self) and x <= self.room(w) + EPS)` | `[VS 8]` |

#### Fit and extent arithmetic `[VS 8]`

Product dimensions X (width), Y (height), Z (depth) are the consumer unit's **bounding box as merchandised** (4.1). The plane is a property of the space; the facing count is a property of the placement; the plane decides which product dimensions matter. `[VS 8]`

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| FIT-1 | ensure | `fits_dimensionally(prod, leaf)` (module function): `result == (leaf.presentation_plane in prod.permitted_planes() and` (`FRONT`: `prod.height_in <= leaf.clear_height_in and prod.depth_in <= leaf.usable_depth_in`; `TOP`: `(leaf.clear_height_in is None or prod.height_in <= leaf.clear_height_in) and (leaf.usable_depth_in is None or prod.depth_in <= leaf.usable_depth_in)`; `PEG`: `True`)`)` | `[VS 8]` |
| FIT-PRE-1 | require | `consumption(prod, f, leaf)`: `f >= 1` and `leaf.presentation_plane in prod.permitted_planes()` | `[VS 8]` |
| FIT-2 | ensure | `consumption(prod, f, leaf)` (module function): `result == leaf.leaf_capacity` if `leaf` is a `Bin`, else `f * prod.facing_extent(leaf.unit)` — the extent `f` facings consume | `[VS 8]` `[SIM]` |

**Examples (LeafSpace, fit)** — at S6

```python
run_b, peg, fd = s6.space("RUN-1L1-B"), s6.space("PEG-1"), s6.space("FD-1")
prod = {k: s6.skus[f"SKU-{k}"].product for k in ("A", "C", "E", "S1")}
open_from_915 = DateWindow(D_0915, None)
# ✓ LS-Q-1
assert run_b.leaf_allocated(D_0915) == 33.0 and peg.leaf_allocated(D_0915) == 4.0
# ✓ LS-Q-2
assert fd.peak_allocated(DateWindow(D_0901, D_1006)) == 1.0 and fd.peak_allocated(DateWindow(D_0929, None)) == 0.0
# ✓ LS-Q-3: PLC-000004 ended before the window, so it does not block
assert run_b.free_intervals(open_from_915) == ((33.0, 48.0),)
assert run_b.free_intervals(open_from_915, without=s6.placements["PLC-000010"]) == ((28.0, 48.0),)
# ✓ LS-Q-4
assert run_b.room(open_from_915) == 15.0 and peg.room(open_from_915) == 0.0
assert s6.space("BIN-1").room(open_from_915) == 0.0 and fd.room(open_from_915) == 6.0
# ✓ LS-Q-5
assert run_b.can_hold(prod["C"], 12.0, open_from_915) and not run_b.can_hold(prod["C"], 16.0, open_from_915)
# ✓ FIT-1
assert not fits_dimensionally(prod["E"], s6.space("RUN-1L1-E"))       # 16 in tall, 14 in clear
assert fits_dimensionally(prod["A"], fd) and not fits_dimensionally(prod["S1"], run_b)
# ✓ FIT-2
assert consumption(prod["A"], 6, s6.space("RUN-1L1-E")) == 48.0
assert consumption(s6.skus["SKU-P1"].product, 1, s6.space("BIN-1")) == 8.0
# ✗ LS-PRE-1: free intervals of a hook-counted leaf
with expect_violation(PreconditionViolation, "LS-PRE-1"):
    peg.free_intervals(open_from_915)
# ✗ LS-PRE-2
with expect_violation(PreconditionViolation, "LS-PRE-2"):
    run_b.can_hold(prod["C"], 0.0, open_from_915)
# ✗ FIT-PRE-1 (f >= 1)
with expect_violation(PreconditionViolation, "FIT-PRE-1"):
    consumption(prod["A"], 0, run_b)
# ✗ supplier LS-Q-3: a gap reported where PLC-000007 sits
assert not P.LS_Q_3(run_b, ((24.0, 28.0), (33.0, 48.0)), None, w=open_from_915, x=None)
# ✗ supplier LS-Q-4: total free length (4 + 15 = 19) instead of the largest gap, with PLC-000007 removed
assert not P.LS_Q_4(run_b, 19.0, None, w=open_from_915, x=s6.placements["PLC-000007"])   # true answer: 15.0
# ✗ inv LS-INV-6: two placements on one segment at once (stub leaf)
p1 = Stub(window=DateWindow(D_0105, None), offset_in=0.0, extent=12.0)
p2 = Stub(window=DateWindow(D_0901, None), offset_in=10.0, extent=6.0)
assert not P.LS_INV_6(Stub(unit=SpaceUnit.LINEAR_IN, placements=(p1, p2)))
# ✗ inv LS-INV-5: more allocated than the leaf holds (stub)
assert not P.LS_INV_5(Stub(leaf_capacity=6.0, placements=(p1,), leaf_allocated=lambda d: 7.0))
# ✗ inv LS-INV-11: a front-facing leaf with no shelf level (stub)
assert not P.LS_INV_11(Stub(presentation_plane=PresentationPlane.FRONT, shelf_level=None))
```

---

### 3.3 `Area` — non-fixture space, and footprint reconciliation `[VS 6]`

| Class | Own attributes | Children |
|---|---|---|
| `Area` | `name: str`, `footprint_sqft: float`, `space_type` (`SALES_FLOOR` or a `NON_MERCHANDISING` type) | `Area`s and top-level fixtures (`Gondola`, `ShelvingRun`, `DoorCaseRun`, `ServiceCase`, `ProduceTable`, `FloorDisplay`) |

`Area(id, name, space_type, footprint_sqft, children)`

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| AREA-NEW-PRE-1 | require | `id.startswith("AREA-")`, `space_type == SALES_FLOOR or space_type.is_within(NON_MERCHANDISING)`, and `footprint_sqft > 0` (AREA-INV-1) | `[VS 6]` |
| AREA-NEW-PRE-2 | require | every child is unattached (`parent is None`) and is an `Area` or a top-level fixture; a merchandising child only if `space_type in {SALES_FLOOR, CHECKOUT}` (AREA-INV-3) | `[DbC]` `[VS 6]` |
| AREA-NEW-PRE-3 | require | AREA-INV-2 holds for `footprint_sqft` and the children | `[VS 6]` |
| AREA-NEW-1 | ensure | attributes as given; `children == tuple(children)`; every child's `parent is self`; `parent is None`; `own_assignment is None` | `[DbC]` |
| AREA-INV-1 | invariant | `footprint_sqft > 0` | `[VS 6]` |
| AREA-INV-2 | invariant | **Reconciliation.** If `children` is non-empty and every child has a non-`None` footprint: `abs(self.footprint_sqft - sum(c.footprint_sqft for c in children)) <= FOOTPRINT_TOL_SQFT`. Circulation is an `Area` of type `AISLE`, so a floor accounts for all of its square feet (Q-17) | `[VS 6]` `[SIM]` |
| AREA-INV-3 | invariant | a merchandising child exists only if `space_type in {SALES_FLOOR, CHECKOUT}` (TYP-3) | `[VS 6]` `[SIM]` |

**Examples (Area)**

```python
# ✓ AREA-NEW-1, AREA-INV-2: 800 = 32 + 3 + 10 + 24 + 24 + 6 + 701
sf = s0.space("AREA-SF")
assert sum(c.footprint_sqft for c in sf.children) == 800.0 and all(c.parent is sf for c in sf.children)
# ✗ AREA-NEW-PRE-1: a merchandising type is not an area type
with expect_violation(PreconditionViolation, "AREA-NEW-PRE-1"):
    Area("AREA-X", "Bad", SpaceType.GONDOLA, 10.0, [])
# ✗ AREA-NEW-PRE-2: a floor display in the backroom
with expect_violation(PreconditionViolation, "AREA-NEW-PRE-2"):
    Area("AREA-X", "Bad", SpaceType.BACKROOM, 1.0, [FloorDisplay("FD-9", "x", 12.0, 12.0)])
# ✗ AREA-NEW-PRE-3: children add to 5.0, not 50.0
with expect_violation(PreconditionViolation, "AREA-NEW-PRE-3"):
    Area("AREA-X", "Bad", SpaceType.SALES_FLOOR, 50.0, [Area("AREA-Y", "Aisle", SpaceType.AISLE, 5.0, [])])
# ✗ AREA-NEW-PRE-2 (attached child): FD-1 already belongs to AREA-SF
with expect_violation(PreconditionViolation, "AREA-NEW-PRE-2"):
    Area("AREA-X", "Again", SpaceType.SALES_FLOOR, 6.0, [s0.space("FD-1")])
# ✗ inv AREA-INV-2 (built at NONE)
with assertion_level(AssertionLevel.NONE):
    bad = Area("AREA-X", "Bad", SpaceType.SALES_FLOOR, 50.0, [Area("AREA-Y", "A", SpaceType.AISLE, 5.0, [])])
with expect_violation(InvariantViolation, "AREA-INV-2"):
    check_invariants(bad)
```

---

### 3.4 Open shelving: Gondola, end caps, peg sections, clip strips `[DISC]` `[VS 7]`

A Gondola is bordered by four aisles, named from a viewer at the front of the store facing the rear: front, back, left, right `[DISC]`. In a grid layout the gondolas run front to back `[TB 1.5]`. So a gondola's two **long faces** are its `LEFT` and `RIGHT` sides, facing the left and right aisles, and its two **ends** are its `FRONT` and `BACK` sides, facing the front and back aisles. The ends are the **end caps**: faces of the same gondola, managed as a separate, more valuable unit `[VS 7]`. The added value is carried by agreements, not by the space (rule 8). (D-1)

| Class | Own attributes | Children |
|---|---|---|
| `Gondola` | `front_aisle, back_aisle, left_aisle, right_aisle: str`, `depth_in: float`, `footprint_sqft: float` | 1–4 `GondolaSide` |
| `GondolaSide` | `facing: Literal["LEFT", "RIGHT", "FRONT", "BACK"]`, `aisle_faced: str` | ≥ 1 `Bay`, then 0+ `PegSection`, then 0+ `ClipStrip` |
| `Bay` | `position: int` (1-based, left to right as seen from the aisle), `width_in: float` | 4 `ShelfRun` |
| `ShelfRun` (leaf) | `shelf_level: ShelfLevel`, `width_in, depth_in, clear_height_in: float`, `secured: bool` | — |
| `PegSection` (leaf) | `hook_count: int`, `hook_depth_units: int`, `location_description: str`, `secured: bool` | — |
| `ClipStrip` (leaf) | `hook_count: int`, `hook_depth_units: int`, `frontage_in: float`, `secured: bool` | — |

`Gondola`, `GondolaSide`: `space_type == GONDOLA`. `Bay`, `ShelfRun`: `space_type` is given at creation (a bay is also used by wall, bakery, and floral shelving, 3.5; a shelf run also by door cases). `PegSection`: `PEG_SECTION`. `ClipStrip`: `CLIP_STRIP`.

**`ShelfRun(id, space_type, shelf_level, width_in, depth_in, clear_height_in, secured=False)`**

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| RUN-NEW-PRE-1 | require | `id.startswith("RUN-")` and `space_type` is a leaf of the type tree within `OPEN_SHELVING` or `DOOR_CASE` | `[DISC]` |
| RUN-NEW-PRE-2 | require | `width_in > 0 and depth_in > 0 and clear_height_in > 0` (RUN-INV-2) | `[DISC]` `[SIM]` |
| RUN-NEW-1 | ensure | attributes as given; `parent is None`, `placements == ()`, `own_assignment is None` | `[DbC]` |
| RUN-INV-1 | invariant | `presentation_plane == FRONT and leaf_capacity == width_in and usable_depth_in == depth_in and hook_depth_units is None and respace_without_reset is False` | `[DISC]` `[VS 8]` |
| RUN-INV-2 | invariant | `width_in > 0 and depth_in > 0 and clear_height_in > 0` | `[DISC]` `[SIM]` |

**`Bay(id, space_type, position, width_in, runs)`**

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| BAY-NEW-PRE-1 | require | `id.startswith("BAY-")`, `space_type` is a leaf type within `OPEN_SHELVING`, `position >= 1`, and `width_in > 0` (BAY-INV-3) | `[DISC]` |
| BAY-NEW-PRE-2 | require | `runs` are four unattached `ShelfRun`s with `space_type` equal to `space_type` | `[DISC]` `[DbC]` |
| BAY-NEW-PRE-3 | require | `[r.shelf_level.physical_order for r in runs] == [1, 2, 3, 4]` and `all(r.width_in == width_in for r in runs)` (BAY-INV-1, BAY-INV-2, BAY-INV-4) | `[DISC]` |
| BAY-NEW-1 | ensure | `children == tuple(runs)` and every run's `parent is self` | `[DbC]` |
| BAY-INV-1 | invariant | `len(children) == 4 and {r.shelf_level for r in children} == set(ShelfLevel)` | `[DISC]` |
| BAY-INV-2 | invariant | `all(r.width_in == self.width_in for r in children)` — a shelf's length is fixed by its fixture | `[DISC]` |
| BAY-INV-3 | invariant | `position >= 1 and width_in > 0` | `[DISC]` `[SIM]` |
| BAY-INV-4 | invariant | `[r.shelf_level.physical_order for r in children] == [1, 2, 3, 4]` — top to bottom | `[DISC]` `[SIM]` |

**`PegSection(id, hook_count, hook_depth_units, location_description, secured=False)`** and **`ClipStrip(id, hook_count, hook_depth_units, frontage_in, secured=False)`**

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| PEG-NEW-PRE-1 | require | `id.startswith("PEG-")`, `hook_count >= 1`, `hook_depth_units >= 1` | `[VS 7]` |
| PEG-NEW-1 | ensure | attributes as given; `parent is None`, `placements == ()` | `[DbC]` |
| PEG-INV-1 | invariant | `presentation_plane == PEG and leaf_capacity == hook_count and shelf_level is None and respace_without_reset is True`. Only the section and its hook count are modeled, not each hook's coordinates | `[VS 7]` |
| PEG-INV-2 | invariant | `hook_count >= 1 and hook_depth_units >= 1` — depth is how many units deep a hook holds | `[VS 7]` |
| CLP-NEW-PRE-1 | require | `id.startswith("CLP-")`, `hook_count >= 1`, `hook_depth_units >= 1`, `frontage_in > 0` | `[VS 7]` |
| CLP-NEW-1 | ensure | attributes as given; `parent is None`, `placements == ()` | `[DbC]` |
| CLP-INV-1 | invariant | PEG-INV-1 and PEG-INV-2 hold for a `ClipStrip`. Its parent, once attached, is a `GondolaSide` | `[VS 7]` |
| CLP-INV-2 | invariant | `self.parent is None or 0 < frontage_in <= min(b.width_in for b in parent.children if isinstance(b, Bay))`. The strip uses a little of its face's frontage; that frontage is recorded but not deducted from any run's capacity (Q-15) | `[VS 7]` `[SIM]` |

**`GondolaSide(id, facing, aisle_faced, bays, pegs=(), clip_strips=())`**

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| SID-NEW-PRE-1 | require | `id.startswith("SID-")` and `facing in ("LEFT", "RIGHT", "FRONT", "BACK")` | `[DISC]` |
| SID-NEW-PRE-2 | require | `bays` is non-empty, its elements are unattached `Bay`s of type `GONDOLA` with positions `1 .. len(bays)` in the order given; `pegs` and `clip_strips` are unattached | `[DISC]` `[DbC]` |
| SID-NEW-PRE-3 | require | every clip strip satisfies CLP-INV-2 against `bays` | `[VS 7]` |
| SID-NEW-1 | ensure | `children == tuple(bays) + tuple(pegs) + tuple(clip_strips)`, each child's `parent is self` | `[DbC]` |
| SID-INV-1 | invariant | the positions of the `Bay` children are exactly `1 .. k`, in child order | `[DISC]` |
| SID-INV-2 | invariant | the side has at least one `Bay` | `[DISC]` |

**`Gondola(id, front_aisle, back_aisle, left_aisle, right_aisle, depth_in, footprint_sqft, sides)`**

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| GDL-NEW-PRE-1 | require | `id.startswith("GDL-")`, `depth_in > 0`, `footprint_sqft > 0` | `[DISC]` |
| GDL-NEW-PRE-2 | require | `sides` are unattached `GondolaSide`s satisfying GDL-INV-1 and GDL-INV-2 | `[DISC]` `[DbC]` |
| GDL-NEW-PRE-3 | require | GDL-INV-3, GDL-INV-4, and GDL-INV-5 hold for the arguments | `[DISC]` `[SIM]` |
| GDL-NEW-1 | ensure | `children == tuple(sides)`, each side's `parent is self` | `[DbC]` |
| GDL-INV-1 | invariant | `1 <= len(children) <= 4` and the sides' `facing` values are distinct | `[DISC]` `[VS 7]` |
| GDL-INV-2 | invariant | `all(s.aisle_faced == {"LEFT": left_aisle, "RIGHT": right_aisle, "FRONT": front_aisle, "BACK": back_aisle}[s.facing] for s in children)` | `[DISC]` |
| GDL-INV-3 | invariant | if both a `LEFT` and a `RIGHT` side exist, they have the same number of bays and the same bay widths, position by position (Q-8) | `[SIM]` |
| GDL-INV-4 | invariant | each end side (`FRONT`, `BACK`) has total bay width `<= depth_in` | `[DISC]` `[SIM]` |
| GDL-INV-5 | invariant | if a `LEFT` or `RIGHT` side exists: `abs(footprint_sqft - length_in * depth_in / 144) <= 0.01`, where `length_in` is the total bay width of the `LEFT` side (or of the `RIGHT` side if there is no `LEFT`) | `[DISC]` `[SIM]` |

Mounting-hardware compatibility (which hook fits which board) is not modeled. `[VS 7]`

**Examples (gondola family)**

```python
G = SpaceType.GONDOLA
runs = lambda p: _runs(p, G, 48.0, 18.0, 14.0)        # conftest helper
# ✓ RUN-NEW-1, RUN-INV-1
r = ShelfRun("RUN-X-E", G, ShelfLevel.EYE_LEVEL, 48.0, 18.0, 14.0)
assert r.parent is None and r.placements == () and r.leaf_capacity == 48.0 and r.unit is SpaceUnit.LINEAR_IN
# ✗ RUN-NEW-PRE-1: a shelf run cannot belong to a produce table
with expect_violation(PreconditionViolation, "RUN-NEW-PRE-1"):
    ShelfRun("RUN-X-E", SpaceType.PRODUCE_TABLE, ShelfLevel.EYE_LEVEL, 48.0, 18.0, 14.0)
# ✗ RUN-NEW-PRE-2
with expect_violation(PreconditionViolation, "RUN-NEW-PRE-2"):
    ShelfRun("RUN-X-E", G, ShelfLevel.EYE_LEVEL, 48.0, 18.0, 0.0)
# ✓ BAY-NEW-1
b = Bay("BAY-X", G, 1, 48.0, runs("RUN-X"))
assert [c.parent is b for c in b.children] == [True] * 4 and b.children[1].shelf_level is ShelfLevel.EYE_LEVEL
# ✗ BAY-NEW-PRE-1
with expect_violation(PreconditionViolation, "BAY-NEW-PRE-1"):
    Bay("BAY-X", G, 0, 48.0, runs("RUN-X"))
# ✗ BAY-NEW-PRE-2: runs of a different fixture type
with expect_violation(PreconditionViolation, "BAY-NEW-PRE-2"):
    Bay("BAY-X", G, 1, 48.0, _runs("RUN-X", SpaceType.WALL_SHELF, 48.0, 18.0, 14.0))
# ✗ BAY-NEW-PRE-3: bottom shelf listed first
with expect_violation(PreconditionViolation, "BAY-NEW-PRE-3"):
    Bay("BAY-X", G, 1, 48.0, list(reversed(runs("RUN-X"))))
# ✓ PEG-NEW-1, PEG-INV-1; CLP-NEW-1
pg = PegSection("PEG-X", 4, 6, "test")
assert pg.leaf_capacity == 4 and pg.unit is SpaceUnit.HOOKS and pg.respace_without_reset
assert ClipStrip("CLP-X", 6, 3, 2.0).presentation_plane is PresentationPlane.PEG
# ✗ PEG-NEW-PRE-1
with expect_violation(PreconditionViolation, "PEG-NEW-PRE-1"):
    PegSection("PEG-X", 0, 6, "test")
# ✗ CLP-NEW-PRE-1
with expect_violation(PreconditionViolation, "CLP-NEW-PRE-1"):
    ClipStrip("CLP-X", 6, 3, 0.0)
# ✓ SID-NEW-1: bays, then pegs, then clip strips
sd = GondolaSide("SID-X", "LEFT", "AISLE-9", bays=[b], pegs=[pg], clip_strips=[ClipStrip("CLP-X", 6, 3, 2.0)])
assert [c.id for c in sd.children] == ["BAY-X", "PEG-X", "CLP-X"]
# ✗ SID-NEW-PRE-1
with expect_violation(PreconditionViolation, "SID-NEW-PRE-1"):
    GondolaSide("SID-X", "UP", "AISLE-9", bays=[Bay("BAY-Y", G, 1, 48.0, runs("RUN-Y"))])
# ✗ SID-NEW-PRE-2: bay positions must start at 1
with expect_violation(PreconditionViolation, "SID-NEW-PRE-2"):
    GondolaSide("SID-X", "LEFT", "AISLE-9", bays=[Bay("BAY-Y", G, 2, 48.0, runs("RUN-Y"))])
# ✗ SID-NEW-PRE-3: a clip strip wider than the narrowest bay
with expect_violation(PreconditionViolation, "SID-NEW-PRE-3"):
    GondolaSide("SID-X", "LEFT", "AISLE-9", bays=[Bay("BAY-Y", G, 1, 48.0, runs("RUN-Y"))],
                clip_strips=[ClipStrip("CLP-Y", 6, 3, 60.0)])
# ✓ GDL-NEW-1, GDL-INV-2, GDL-INV-5 on the fixture: 96 × 48 / 144 = 32.0
g = s0.space("GDL-1")
assert [s.facing for s in g.children] == ["LEFT", "FRONT"] and g.children[0].aisle_faced == g.left_aisle == "AISLE-1"
# ✗ GDL-NEW-PRE-1
new_side = lambda: GondolaSide("SID-Z", "LEFT", "A", bays=[Bay("BAY-Z", G, 1, 48.0, runs("RUN-Z"))])
with expect_violation(PreconditionViolation, "GDL-NEW-PRE-1"):
    Gondola("GDL-Z", "F", "B", "A", "R", 0.0, 16.0, [new_side()])
# ✗ GDL-NEW-PRE-2: the side faces "A" but the gondola's left aisle is "L"
with expect_violation(PreconditionViolation, "GDL-NEW-PRE-2"):
    Gondola("GDL-Z", "F", "B", "L", "R", 48.0, 16.0, [new_side()])
# ✗ GDL-NEW-PRE-3: footprint 20.0 ≠ 48 × 48 / 144 = 16.0
with expect_violation(PreconditionViolation, "GDL-NEW-PRE-3"):
    Gondola("GDL-Z", "F", "B", "A", "R", 48.0, 20.0, [new_side()])
# ✗ inv GDL-INV-4: an end cap wider than the gondola is deep (built at NONE)
with assertion_level(AssertionLevel.NONE):
    bad = Gondola("GDL-Z", "F", "B", "A", "R", 24.0, 8.0,
                  [GondolaSide("SID-Z", "FRONT", "F", bays=[Bay("BAY-Z", G, 1, 48.0, runs("RUN-Z"))])])
with expect_violation(InvariantViolation, "GDL-INV-4"):
    check_invariants(bad)
# ✗ inv GDL-INV-3: long sides of different lengths (stub)
side = lambda f, n: Stub(facing=f, children=tuple(Stub(width_in=48.0, position=i + 1) for i in range(n)))
assert not P.GDL_INV_3(Stub(children=(side("LEFT", 2), side("RIGHT", 3))))
# ✗ inv BAY-INV-4: shelves out of physical order (stub)
assert not P.BAY_INV_4(Stub(children=tuple(Stub(shelf_level=l) for l in
                                          (ShelfLevel.EYE_LEVEL, ShelfLevel.TOP, ShelfLevel.MIDDLE, ShelfLevel.BOTTOM))))
```

### 3.5 Other open shelving, door cases, service cases, produce tables, floor displays `[DISC]`

**Non-gondola open shelving (wall, bakery, floral).** Same shape as a gondola face without the gondola layer: `ShelvingRun → Bay → ShelfRun`.

| Class | Own attributes | Children |
|---|---|---|
| `ShelvingRun` | `space_type ∈ {WALL_SHELF, BAKERY_SHELVING, FLORAL_SHELVING}`, `location_description: str`, `footprint_sqft: float` | ≥ 1 `Bay`, then 0+ `PegSection` |

`ShelvingRun(id, space_type, location_description, footprint_sqft, bays, pegs=())`

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| SHR-NEW-PRE-1 | require | `id.startswith("SHR-")`, `space_type in {WALL_SHELF, BAKERY_SHELVING, FLORAL_SHELVING}`, `footprint_sqft > 0` | `[DISC]` |
| SHR-NEW-PRE-2 | require | `bays` non-empty, unattached, of `space_type`, positions `1 .. len(bays)` in order; `pegs` unattached | `[DISC]` `[DbC]` |
| SHR-NEW-1 | ensure | `children == tuple(bays) + tuple(pegs)`, each child's `parent is self` | `[DbC]` |
| SHR-INV-1 | invariant | at least one `Bay`, with positions exactly `1 .. k` in child order | `[DISC]` |

**Door cases (reach-in cooler, reach-in freezer, floral cooler).**

| Class | Own attributes | Children |
|---|---|---|
| `DoorCaseRun` | `space_type ∈ {REACH_IN_COOLER, REACH_IN_FREEZER, FLORAL_COOLER}`, `location_description: str`, `footprint_sqft: float` | ≥ 1 `Door` |
| `Door` | `space_type` (as its run), `number: int`, `width_in: float` | 4 `ShelfRun` |

`Door(id, space_type, number, width_in, runs)` and `DoorCaseRun(id, space_type, location_description, footprint_sqft, doors)`

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| DOR-NEW-PRE-1 | require | `id.startswith("DOR-")`, `space_type` a leaf type within `DOOR_CASE`, `number >= 1`, `width_in > 0` | `[DISC]` |
| DOR-NEW-PRE-2 | require | `runs` are four unattached `ShelfRun`s of `space_type`, with `[r.shelf_level.physical_order for r in runs] == [1, 2, 3, 4]` and `all(r.width_in == width_in for r in runs)` | `[DISC]` `[DbC]` |
| DOR-NEW-1 | ensure | `children == tuple(runs)`, each run's `parent is self` | `[DbC]` |
| DOR-INV-1 | invariant | `len(children) == 4` and `{r.shelf_level for r in children} == set(ShelfLevel)` | `[DISC]` |
| DOR-INV-2 | invariant | `all(r.width_in == self.width_in for r in children)` | `[DISC]` |
| DOR-INV-3 | invariant | (on `DoorCaseRun`) door `number`s are exactly `1 .. n`, in child order | `[DISC]` |
| DOR-INV-4 | invariant | `[r.shelf_level.physical_order for r in children] == [1, 2, 3, 4]` | `[DISC]` `[SIM]` |
| DCR-NEW-PRE-1 | require | `id.startswith("DCR-")`, `space_type in {REACH_IN_COOLER, REACH_IN_FREEZER, FLORAL_COOLER}`, `footprint_sqft > 0` | `[DISC]` |
| DCR-NEW-PRE-2 | require | `doors` non-empty, unattached, of `space_type`, numbered `1 .. len(doors)` in order | `[DISC]` `[DbC]` |
| DCR-NEW-1 | ensure | `children == tuple(doors)`, each door's `parent is self` | `[DbC]` |
| DCR-INV-1 | invariant | a `DoorCaseRun` has at least one `Door` | `[DISC]` |

**Service cases (deli case, service/self-service refrigerated case).** One fixed-length case with tiers and no bay layer.

| Class | Own attributes | Children |
|---|---|---|
| `ServiceCase` | `space_type ∈ {DELI_CASE, SERVICE_REFRIGERATED_CASE}`, `location_description: str`, `length_in: float`, `footprint_sqft: float` | 1–4 `Tier` |
| `Tier` (leaf) | `space_type` (as its case), `shelf_level: ShelfLevel`, `width_in, depth_in, clear_height_in: float`, `secured: bool` | — |

`Tier(id, space_type, shelf_level, width_in, depth_in, clear_height_in, secured=False)` and `ServiceCase(id, space_type, location_description, length_in, footprint_sqft, tiers)`

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| TIER-NEW-PRE-1 | require | `id.startswith("TIER-")` and `space_type in {DELI_CASE, SERVICE_REFRIGERATED_CASE}` | `[DISC]` |
| TIER-NEW-PRE-2 | require | `width_in > 0 and depth_in > 0 and clear_height_in > 0` (TIER-INV-2) | `[DISC]` `[SIM]` |
| TIER-NEW-1 | ensure | attributes as given; `parent is None`, `placements == ()` | `[DbC]` |
| TIER-INV-1 | invariant | `presentation_plane == FRONT and leaf_capacity == width_in and usable_depth_in == depth_in and respace_without_reset is False` | `[DISC]` |
| TIER-INV-2 | invariant | `width_in > 0 and depth_in > 0 and clear_height_in > 0` | `[DISC]` `[SIM]` |
| SVC-NEW-PRE-1 | require | `id.startswith("SVC-")`, `space_type in {DELI_CASE, SERVICE_REFRIGERATED_CASE}`, `length_in > 0`, `footprint_sqft > 0` | `[DISC]` |
| SVC-NEW-PRE-2 | require | `1 <= len(tiers) <= 4`, unattached, of `space_type`, satisfying SVC-INV-1 and SVC-INV-2 | `[DISC]` `[DbC]` |
| SVC-NEW-1 | ensure | `children == tuple(tiers)`, each tier's `parent is self` | `[DbC]` |
| SVC-INV-1 | invariant | `all(t.width_in == self.length_in for t in children)` and tier shelf levels are distinct | `[DISC]` |
| SVC-INV-2 | invariant | `[t.shelf_level.physical_order for t in children]` is strictly increasing — top to bottom | `[DISC]` `[SIM]` |

**Produce tables.** Footprint-based, with no shelf levels.

| Class | Own attributes | Children |
|---|---|---|
| `ProduceTable` | `location_description: str`, `footprint_sqft: float` | ≥ 1 `Bin` |
| `Bin` (leaf) | `position: int`, `footprint_sqft: float`, `secured: bool` | — |

`Bin(id, position, footprint_sqft, secured=False)` and `ProduceTable(id, location_description, footprint_sqft, bins)`

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| BIN-NEW-PRE-1 | require | `id.startswith("BIN-")`, `position >= 1`, `footprint_sqft > 0` | `[DISC]` |
| BIN-NEW-1 | ensure | attributes as given; `parent is None`, `placements == ()` | `[DbC]` |
| BIN-INV-1 | invariant | `presentation_plane == TOP and leaf_capacity == footprint_sqft and shelf_level is None and usable_depth_in is None and clear_height_in is None` — no shelf levels on produce tables | `[DISC]` |
| BIN-INV-2 | invariant | for every pair `p != q` in `placements`, `not p.window.overlaps(q.window)` — a bin holds one SKU at a time (Q-12) | `[SIM]` |
| TBL-NEW-PRE-1 | require | `id.startswith("TBL-")` and `footprint_sqft > 0` | `[DISC]` |
| TBL-NEW-PRE-2 | require | `bins` non-empty and unattached, positions `1 .. len(bins)` in order, and `sum(b.footprint_sqft for b in bins) <= footprint_sqft` | `[DISC]` `[DbC]` |
| TBL-NEW-1 | ensure | `children == tuple(bins)`, each bin's `parent is self` | `[DbC]` |
| TBL-INV-1 | invariant | `sum(b.footprint_sqft for b in children) <= self.footprint_sqft` | `[DISC]` |
| TBL-INV-2 | invariant | at least one `Bin`, with positions exactly `1 .. k` in child order | `[DISC]` `[SIM]` |

**Floor displays.** A flat footprint measured as an area, since horizontal decks allocate square feet. `[VS 8]` Both a leaf and a top-level fixture.

`FloorDisplay(id, location_description, capacity_linear_in, footprint_depth_in, max_stack_height_in=None, secured=False)`

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| FD-NEW-PRE-1 | require | `id.startswith("FD-")`, `capacity_linear_in > 0`, `footprint_depth_in > 0`, `max_stack_height_in is None or max_stack_height_in > 0` | `[DISC]` |
| FD-NEW-1 | ensure | attributes as given; `parent is None`, `placements == ()` | `[DbC]` |
| FD-INV-1 | invariant | `presentation_plane == TOP and leaf_capacity == capacity_linear_in * footprint_depth_in / 144 and shelf_level is None and usable_depth_in == footprint_depth_in and clear_height_in == max_stack_height_in` — flat, no tiers | `[DISC]` `[VS 8]` |
| FD-INV-2 | invariant | `all(p.placement_type == PROMOTIONAL for p in placements)` — floor displays carry promotional placements only (Q-10) | `[SIM]` |
| FD-INV-3 | invariant | `footprint_sqft == leaf_capacity` | `[DISC]` |

**Examples (other fixtures)**

```python
W, C, D = SpaceType.WALL_SHELF, SpaceType.REACH_IN_COOLER, SpaceType.DELI_CASE
# ✓ SHR-NEW-1
assert [c.id for c in s0.space("SHR-1").children] == ["BAY-W1"] and s0.space("RUN-W1-E").space_type is W
# ✗ SHR-NEW-PRE-1: a gondola type is not a shelving-run type
with expect_violation(PreconditionViolation, "SHR-NEW-PRE-1"):
    ShelvingRun("SHR-X", SpaceType.GONDOLA, "x", 3.0, bays=[Bay("BAY-X", W, 1, 36.0, _runs("RUN-X", W, 36.0, 12.0, 12.0))])
# ✗ SHR-NEW-PRE-2: no bays
with expect_violation(PreconditionViolation, "SHR-NEW-PRE-2"):
    ShelvingRun("SHR-X", W, "x", 3.0, bays=[])
# ✓ DOR-NEW-1, DCR-NEW-1; TYP-2 through the type
assert s0.space("DOR-1").children[1].id == "RUN-D1-E" and s0.space("RUN-D1-E").temperature_zone is TemperatureZone.REFRIGERATED
# ✗ DOR-NEW-PRE-1
with expect_violation(PreconditionViolation, "DOR-NEW-PRE-1"):
    Door("DOR-X", C, 0, 30.0, _runs("RUN-X", C, 30.0, 20.0, 12.0))
# ✗ DOR-NEW-PRE-2: runs 24 in wide on a 30 in door
with expect_violation(PreconditionViolation, "DOR-NEW-PRE-2"):
    Door("DOR-X", C, 1, 30.0, _runs("RUN-X", C, 24.0, 20.0, 12.0))
# ✗ DCR-NEW-PRE-1
with expect_violation(PreconditionViolation, "DCR-NEW-PRE-1"):
    DoorCaseRun("DCR-X", W, "x", 10.0, doors=[Door("DOR-X", C, 1, 30.0, _runs("RUN-X", C, 30.0, 20.0, 12.0))])
# ✗ DCR-NEW-PRE-2: doors numbered from 2
with expect_violation(PreconditionViolation, "DCR-NEW-PRE-2"):
    DoorCaseRun("DCR-X", C, "x", 10.0, doors=[Door("DOR-X", C, 2, 30.0, _runs("RUN-X", C, 30.0, 20.0, 12.0))])
# ✓ TIER-NEW-1, SVC-NEW-1
assert [t.shelf_level for t in s0.space("SVC-1").children] == [ShelfLevel.EYE_LEVEL, ShelfLevel.BOTTOM]
# ✗ TIER-NEW-PRE-1
with expect_violation(PreconditionViolation, "TIER-NEW-PRE-1"):
    Tier("TIER-X", C, ShelfLevel.TOP, 72.0, 24.0, 10.0)
# ✗ TIER-NEW-PRE-2
with expect_violation(PreconditionViolation, "TIER-NEW-PRE-2"):
    Tier("TIER-X", D, ShelfLevel.TOP, 72.0, 0.0, 10.0)
# ✗ SVC-NEW-PRE-1
with expect_violation(PreconditionViolation, "SVC-NEW-PRE-1"):
    ServiceCase("SVC-X", D, "x", 0.0, 24.0, tiers=[Tier("TIER-X", D, ShelfLevel.TOP, 72.0, 24.0, 10.0)])
# ✗ SVC-NEW-PRE-2: bottom tier listed above the top tier
with expect_violation(PreconditionViolation, "SVC-NEW-PRE-2"):
    ServiceCase("SVC-X", D, "x", 72.0, 24.0, tiers=[Tier("TIER-X", D, ShelfLevel.BOTTOM, 72.0, 24.0, 10.0),
                                                     Tier("TIER-Y", D, ShelfLevel.TOP, 72.0, 24.0, 10.0)])
# ✓ BIN-NEW-1, TBL-NEW-1
assert s0.space("BIN-2").position == 2 and s0.space("BIN-2").leaf_capacity == 8.0
# ✗ BIN-NEW-PRE-1
with expect_violation(PreconditionViolation, "BIN-NEW-PRE-1"):
    Bin("BIN-X", 1, 0.0)
# ✗ TBL-NEW-PRE-1
with expect_violation(PreconditionViolation, "TBL-NEW-PRE-1"):
    ProduceTable("TBL-X", "x", 0.0, bins=[Bin("BIN-X", 1, 8.0)])
# ✗ TBL-NEW-PRE-2: bins larger than the table
with expect_violation(PreconditionViolation, "TBL-NEW-PRE-2"):
    ProduceTable("TBL-X", "x", 10.0, bins=[Bin("BIN-X", 1, 8.0), Bin("BIN-Y", 2, 8.0)])
# ✓ FD-NEW-1, FD-INV-1: 24 × 36 / 144 = 6.0 sq ft
assert s0.space("FD-1").leaf_capacity == 6.0 == s0.space("FD-1").footprint_sqft
# ✗ FD-NEW-PRE-1
with expect_violation(PreconditionViolation, "FD-NEW-PRE-1"):
    FloorDisplay("FD-X", "x", 24.0, 36.0, max_stack_height_in=0.0)
# ✗ inv BIN-INV-2: two SKUs in one bin at once (stub)
assert not P.BIN_INV_2(Stub(placements=(Stub(window=DateWindow(D_0105, None)), Stub(window=DateWindow(D_0901, D_0929)))))
# ✗ inv FD-INV-2: a permanent placement on a floor display (stub)
assert not P.FD_INV_2(Stub(placements=(Stub(placement_type=PlacementType.PERMANENT),)))
# ✗ inv SVC-INV-2 (stub)
assert not P.SVC_INV_2(Stub(children=(Stub(shelf_level=ShelfLevel.BOTTOM), Stub(shelf_level=ShelfLevel.TOP))))
```

### 3.6 Blocking coordinates `[VS 12]`

The valuable vertical coordinate is the **shelf level**. The horizontal coordinate is (bay position, offset) and carries no allocation preference.

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| BLK-1 | ensure | `block_group()`: a `ShelfRun` under a `Bay` → the bay's parent (`GondolaSide` or `ShelvingRun`); a `ShelfRun` under a `Door` → the door's `DoorCaseRun`; a `Tier` → its `ServiceCase`; a `Bin` → its `ProduceTable`; `FloorDisplay`, `PegSection`, `ClipStrip` → `self` | `[VS 12]` |
| BLK-2 | ensure | `bay_position()`: a `ShelfRun` under a `Bay` → `Bay.position`; a `ShelfRun` under a `Door` → `Door.number`; every other leaf → `1` | `[VS 12]` |

**Examples (BLK-1, BLK-2)**

```python
# ✓ BLK-1, BLK-2
assert s0.space("RUN-1L2-E").block_group().id == "SID-1L" and s0.space("RUN-1L2-E").bay_position() == 2
assert s0.space("RUN-D1-M").block_group().id == "DCR-1" and s0.space("BIN-2").bay_position() == 1
assert s0.space("PEG-1").block_group().id == "PEG-1"
# ✗ supplier BLK-1: grouping a gondola run by its bay instead of its side
assert not P.BLK_1(s0.space("RUN-1L2-E"), s0.space("BAY-1L2"), None)
```

---
## Part 4 — Products, SKUs, categories, business units

### 4.1 `Product` — what the thing is `[VS 1, 8]`

A product is the manufactured item with its brand, size, formulation, and UPC. It owns the physical truth about the item, so its dimensions live here regardless of who stocks it or how it is displayed. `[VS 1, 8]`

```
class Product
creation  Product(upc, name, brand, manufacturer, private_label, perishable, form,
                  width_in, height_in, depth_in, units_per_case, stackable,
                  case_dims_in=None, spec=None)
queries
    upc, name, brand, manufacturer: str
    private_label, perishable, stackable: bool
    form: ProductForm
    width_in, height_in, depth_in: float              -- X, Y, Z: consumer-unit bounding box as merchandised
    case_dims_in: tuple[float, float, float] | None   -- case-pack (shipper) bounding box
    units_per_case: int
    spec: NonBoxSpec | None                           -- overrides for products that are not boxes
    skus: tuple[SKU, ...]                             -- derived (A); never stored on Product
    permitted_planes() -> frozenset[PresentationPlane]
    facing_extent(unit: SpaceUnit) -> float
    units_capacity_for(leaf: LeafSpace, extent: float) -> int

value NonBoxSpec
creation  NonBoxSpec(facing_extent, units_per_extent)
    facing_extent: Mapping[SpaceUnit, float]          -- extent one facing consumes, per unit
    units_per_extent: Mapping[SpaceUnit, float]       -- units held per inch / sq ft of extent
```

The bounding box is a box and nothing more: not the shape, not the graphics, not how the item sits in the case. Products that are not boxes (bagged goods that slump, hanging peg items, loose produce) take an override instead of a computed number. `[VS 8]`

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| NBS-NEW-PRE-1 | require | `len(facing_extent) >= 1`, `set(facing_extent) <= set(units_per_extent)`, and every value of both mappings `> 0` | `[VS 8]` `[SIM]` |
| NBS-NEW-1 | ensure | the mappings are stored as given (immutable copies) | `[DbC]` |
| PROD-NEW-PRE-1 | require | `upc != ""` and PROD-INV-1 holds for the arguments | `[VS 1, 8]` |
| PROD-NEW-PRE-2 | require | PROD-INV-2 holds for `form` and `spec` | `[VS 8]` `[SIM]` |
| PROD-NEW-1 | ensure | attributes as given | `[DbC]` |
| PROD-INV-1 | invariant | `width_in > 0 and height_in > 0 and depth_in > 0 and units_per_case >= 1`; if `case_dims_in is not None`, all three values are `> 0` | `[VS 8]` |
| PROD-INV-2 | invariant | `form in {BOX, HANGING} ⇒ spec is None`; `form == SOFT_PACK ⇒ spec is not None and LINEAR_IN in spec.facing_extent`; `form == LOOSE ⇒ spec is not None and SQFT in spec.facing_extent` | `[VS 8]` `[SIM]` |
| PROD-INV-3 | invariant (A) | `upc` is unique in the store, and `skus == tuple(s for s in store.skus.values() if s.product is self)` — the SKU references the product, not the reverse. One product may have several SKUs | `[VS 1]` |
| PROD-Q-1 | ensure | `permitted_planes()`: `BOX → {FRONT, TOP}`; `SOFT_PACK → {FRONT} ∪ ({TOP} if SQFT in spec.facing_extent else ∅)`; `HANGING → {PEG}`; `LOOSE → {TOP}` (Q-6) | `[SIM]` |
| PROD-PRE-1 | require | `facing_extent(u)`: `plane_of(u) in self.permitted_planes()` | `[VS 8]` |
| PROD-Q-2 | ensure | `facing_extent(u)`: `result > 0` and: `BOX`: `LINEAR_IN → width_in`, `SQFT → width_in * depth_in / 144`; `HANGING`: `HOOKS → 1.0`; `SOFT_PACK`, `LOOSE`: `spec.facing_extent[u]`. A front face uses X by Y and consumes X; a top face uses X by Z and consumes X·Z | `[VS 8]` |
| PROD-PRE-2 | require | `units_capacity_for(leaf, x)`: `fits_dimensionally(self, leaf)` and `x > 0` | `[VS 8]` |
| PROD-Q-3 | ensure | `units_capacity_for(leaf, x)`: let `f = floor(x / facing_extent(leaf.unit) + EPS)` and `stack = max(1, floor(leaf.clear_height_in / height_in + EPS))` if `stackable and leaf.clear_height_in is not None`, else `1`. `result ==` for `BOX` on `FRONT`: `f * floor(leaf.usable_depth_in / depth_in + EPS) * stack`; `BOX` on `TOP`: `f * stack`; `HANGING`: `f * leaf.hook_depth_units`; `SOFT_PACK`, `LOOSE`: `floor(x * spec.units_per_extent[leaf.unit] + EPS)`. Units held is a derived figure that rolls up across planes | `[VS 8]` |

**Examples (Product)**

```python
pa, py1, pp1, ps1 = (s0.skus[k].product for k in ("SKU-A", "SKU-Y1", "SKU-P1", "SKU-S1"))
# ✓ PROD-NEW-1, PROD-Q-1
assert pa.upc == "100000000001" and pa.permitted_planes() == {PresentationPlane.FRONT, PresentationPlane.TOP}
assert pp1.permitted_planes() == {PresentationPlane.TOP} and ps1.permitted_planes() == {PresentationPlane.PEG}
# ✓ PROD-Q-2: same box, two projections
assert pa.facing_extent(SpaceUnit.LINEAR_IN) == 8.0 and abs(pa.facing_extent(SpaceUnit.SQFT) - 48 / 144) < EPS
assert ps1.facing_extent(SpaceUnit.HOOKS) == 1.0 and pp1.facing_extent(SpaceUnit.SQFT) == 0.5
# ✓ PROD-Q-3: 6 facings × 3 deep; yogurt 6 × 5 deep × 3 high; apples 8 sq ft × 40
assert pa.units_capacity_for(s0.space("RUN-1L1-E"), 48.0) == 18
assert py1.units_capacity_for(s0.space("RUN-D1-E"), 30.0) == 90
assert pp1.units_capacity_for(s0.space("BIN-1"), 8.0) == 320
# ✓ NBS-NEW-1
assert pp1.spec.units_per_extent[SpaceUnit.SQFT] == 40.0
# ✗ NBS-NEW-PRE-1: a facing extent with no matching units-per-extent
with expect_violation(PreconditionViolation, "NBS-NEW-PRE-1"):
    NonBoxSpec({SpaceUnit.SQFT: 0.5}, {})
# ✗ PROD-NEW-PRE-1
with expect_violation(PreconditionViolation, "PROD-NEW-PRE-1"):
    Product("100000000099", "x", "b", "m", False, False, ProductForm.BOX, 0.0, 8.0, 6.0, 12, False)
# ✗ PROD-NEW-PRE-2: loose produce needs an override
with expect_violation(PreconditionViolation, "PROD-NEW-PRE-2"):
    Product("100000000099", "x", "b", "m", False, True, ProductForm.LOOSE, 3.0, 3.0, 3.0, 80, False)
# ✗ PROD-PRE-1: a box has no hook extent
with expect_violation(PreconditionViolation, "PROD-PRE-1"):
    pa.facing_extent(SpaceUnit.HOOKS)
# ✗ PROD-PRE-2: SKU-E's box does not fit under a 14 in shelf
with expect_violation(PreconditionViolation, "PROD-PRE-2"):
    s0.skus["SKU-E"].product.units_capacity_for(s0.space("RUN-1L1-E"), 4.0)
# ✗ supplier PROD-Q-3: ignoring depth on a front-facing shelf
assert not P.PROD_Q_3(pa, 6, None, leaf=s0.space("RUN-1L1-E"), x=48.0)
# ✗ inv PROD-INV-2: a box with an override (built at NONE)
with assertion_level(AssertionLevel.NONE):
    bad = Product("100000000099", "x", "b", "m", False, False, ProductForm.BOX, 4.0, 8.0, 6.0, 12, False,
                  spec=NonBoxSpec({SpaceUnit.LINEAR_IN: 4.0}, {SpaceUnit.LINEAR_IN: 1.0}))
with expect_violation(InvariantViolation, "PROD-INV-2"):
    check_invariants(bad)
```

---

### 4.2 `SKU` — how we stock, price, and shelve it here `[VS 1]` `[TB 2.2, 2.3]`

Product answers "what is this". SKU answers "how do we stock, price, and shelve it here". This version specifies only what allocation reads. Later sections add features but do not change these.

```
class SKU
creation  SKU(sku_id, product, category, retail_price, unit_cost, sales_velocity_units_per_week,
              assortment_status=CORE, season=None, promo_display_eligible=False)
queries
    sku_id: str
    product: Product
    category: CategoryNode                          -- a leaf of the category hierarchy
    retail_price, unit_cost: float                  [TB 7.1]
    sales_velocity_units_per_week: float            [TB 2.2, 2.3]
    assortment_status: AssortmentStatus             [TB 2.6]
    season: tuple[int, int] | None                  -- (first_month, last_month), inclusive, may wrap the year
    promo_display_eligible: bool
    -- delegated to product (never stored on the SKU)
    brand, manufacturer: str
    private_label, perishable: bool
    -- derived
    department: CategoryNode                        -- root of the category hierarchy above this SKU
    business_unit() -> BusinessUnit                 -- (A)
    unit_margin, gross_margin_pct, weekly_gross_margin: float
    in_season(on: date) -> bool                     [TB 2.1]
    placements: tuple[PlacementRecord, ...]         -- (A) all, including ended and future-dated
    placements_on(on: date) -> frozenset[PlacementRecord]
    home_placement(on: date) -> Placement | None    -- (A) HOME-1
```

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| SKU-NEW-PRE-1 | require | `sku_id.startswith("SKU-")`, `isinstance(category, CategoryNode)`, and `category.is_leaf` (SKU-INV-9) | `[SIM]` |
| SKU-NEW-PRE-2 | require | SKU-INV-1 holds for the arguments | `[TB 2.3]` |
| SKU-NEW-PRE-3 | require | SKU-INV-11 holds for `season` | `[TB 2.1]` `[SIM]` |
| SKU-NEW-1 | ensure | attributes as given; `placements == ()` until attached | `[DbC]` |
| SKU-INV-1 | invariant | `retail_price > 0 and unit_cost >= 0 and sales_velocity_units_per_week >= 0` | `[TB 2.3]` |
| SKU-INV-3 | invariant | `unit_margin == retail_price - unit_cost` and `gross_margin_pct == unit_margin / retail_price` | `[TB 2.3, 2.5]` |
| SKU-INV-4 | invariant | `weekly_gross_margin == sales_velocity_units_per_week * unit_margin` | `[TB 2.3]` |
| SKU-INV-6 | invariant | `department is self.category.root()` | `[VS 6]` |
| SKU-INV-8 | invariant (A) | `all(p.sku is self for p in placements)` | `[DbC]` |
| SKU-INV-9 | invariant | `self.category.is_leaf` — SKUs attach to leaf category nodes only (Q-3) | `[SIM]` |
| SKU-INV-10 | invariant | `brand == product.brand and manufacturer == product.manufacturer and private_label == product.private_label and perishable == product.perishable` | `[VS 1]` |
| SKU-INV-11 | invariant | `season is None or (1 <= season[0] <= 12 and 1 <= season[1] <= 12)` | `[TB 2.1]` `[SIM]` |
| SKU-Q-1 | ensure | `in_season(d)`: with `(first, last) = season`, `result == (season is None or (first <= d.month <= last if first <= last else (d.month >= first or d.month <= last)))` | `[TB 2.1]` `[SIM]` |
| SKU-Q-2 | ensure | `placements_on(d)`: `result == frozenset(p for p in placements if p.window.contains(d))` | `[VS 4]` |
| SKU-Q-3 | ensure | `home_placement(d)`: `result is store.home_placement(self, d)` (HOME-1) | `[VS 2]` |
| SKU-ZERO | invariant | **Zero placements is a legitimate state**, for a delisted or not-yet-reset item: `len(placements) >= 0`. One SKU may have any number of placements. Several placements signal something: a secondary display, an end cap, a seasonal off-shelf position | `[VS 2]` |

**Examples (SKU)**

```python
a, b, f = s6.skus["SKU-A"], s6.skus["SKU-B"], s6.skus["SKU-F"]
# ✓ SKU-NEW-1, SKU-INV-3, SKU-INV-4, SKU-INV-6, SKU-INV-10
assert (a.unit_margin, a.gross_margin_pct, a.weekly_gross_margin) == (1.0, 0.25, 30.0)
assert a.department.id == "CAT-DG" and a.brand == "Acme" and s6.skus["SKU-C"].private_label
# ✓ SKU-Q-1: Sept–Nov; a wrapping season (Nov–Feb) includes January
assert f.in_season(D_0915) and not f.in_season(D_0105)
assert SKU("SKU-X", a.product, a.category, 4.0, 3.0, 1.0, season=(11, 2)).in_season(D_0105)
# ✓ SKU-Q-2: a permanent placement and an incidental one
assert {p.placement_id for p in b.placements_on(D_0915)} == {"PLC-000011", "INC-000001"}
# ✓ SKU-Q-3
assert a.home_placement(D_0915).placement_id == "PLC-000003"
# ✓ SKU-ZERO
assert s6.skus["SKU-H"].placements == ()
# ✗ SKU-NEW-PRE-1: Cereal is not a leaf
with expect_violation(PreconditionViolation, "SKU-NEW-PRE-1"):
    SKU("SKU-X", a.product, s6.category("CAT-CER"), 4.0, 3.0, 1.0)
# ✗ SKU-NEW-PRE-2
with expect_violation(PreconditionViolation, "SKU-NEW-PRE-2"):
    SKU("SKU-X", a.product, a.category, 0.0, 3.0, 1.0)
# ✗ SKU-NEW-PRE-3
with expect_violation(PreconditionViolation, "SKU-NEW-PRE-3"):
    SKU("SKU-X", a.product, a.category, 4.0, 3.0, 1.0, season=(0, 3))
# ✗ supplier SKU-Q-1: a non-wrapping test applied to (11, 2)
assert not P.SKU_Q_1(Stub(season=(11, 2)), False, None, d=D_0105)
# ✗ inv SKU-INV-10: a SKU whose brand differs from its product's (stub)
assert not P.SKU_INV_10(Stub(product=a.product, brand="Other", manufacturer=a.manufacturer,
                             private_label=False, perishable=False))
```

---

### 4.3 `CategoryNode` — the product-category hierarchy and space policy `[VS 6]` `[TB 1.2, 2.1–2.5]`

The category hierarchy has arbitrary depth: department, category, subcategory, segment is the usual ladder, and some branches go deeper than others. `level_label` is a free label. `[VS 6]` Textbook 2.1 treats a category as "a discrete business unit"; this contract keeps the category and business-unit hierarchies separate (4.4, companion T-2).

```
class CategoryNode           (conforms to HierarchyNode and Rollup)
creation  CategoryNode(id, name, level_label, own_policy, children=(), last_reset_date=None)
queries
    id, name, level_label: str
    own_policy: Mapping[str, object]           -- fields this node defines itself (may be empty)
    last_reset_date: date | None               -- own state; set by execution of a CYCLE version
    policy() -> CategoryPolicy                 -- effective policy, resolved through ancestors
    skus: tuple[SKU, ...]                      -- (A) SKUs attached exactly to this node
    subtree_skus() -> frozenset[SKU]           -- (A)
    assigned_spaces() -> tuple[LeafSpace, ...] -- (A) leaves whose effective assigned_category is this node
    effective_last_reset_date() -> date | None
    next_reset_due(calendar: tuple[date, ...]) -> date | None
    private_label_space_share(on: date) -> float   -- (A)

value CategoryPolicy
    role: CategoryRole                                     [TB 2.1]
    temperature_zone: TemperatureZone                      [TB 1.3]
    allowed_space_types: frozenset[SpaceType]              -- a type allows its whole subtree of types
    secured_case_required: bool                            [TB 6.4]
    min_facings_per_sku, max_facings_per_sku: int          [TB 2.3]
    space_elasticity: float                                [TB 2.3]
    sales_per_linear_ft_per_week: float                    [TB 2.3]   (reporting input)
    category_captain: str | None                           [TB 2.4]
    private_label_space_target_pct: float                  [TB 2.5]
    reset_cycle_months: int                                [TB 2.6] [VS 4]
    blocking_mode: BlockingMode                            [VS 12]
    private_label_benchmark_brand: str | None              [VS 12]
```

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| CAT-NEW-PRE-1 | require | `id.startswith("CAT-")` and `set(own_policy) <= CategoryPolicy fields` | `[VS 6]` |
| CAT-NEW-PRE-2 | require | `children` are unattached `CategoryNode`s | `[DbC]` |
| CAT-NEW-PRE-3 | require | every field present in `own_policy` is valid on its own: facing bounds `>= 1`; `0 < space_elasticity < 1`; `0 <= private_label_space_target_pct <= 100`; `reset_cycle_months >= 1`; `allowed_space_types` non-empty and free of floor-display types (CAT-INV-10) | `[TB 2.3]` `[SIM]` |
| CAT-NEW-1 | ensure | attributes as given; `children == tuple(children)`, each child's `parent is self` | `[DbC]` |
| CAT-INV-1 | invariant | a root node's `own_policy` defines **every** `CategoryPolicy` field, so `policy()` is total (checked on roots given to `Store.build`) | `[VS 6]` `[SIM]` |
| CAT-Q-1 | ensure | `policy()`: for each field `f`, `result.f` is the value in `own_policy` of the nearest node in `(self,) + self.ancestors()` that defines `f` | `[VS 6]` `[SIM]` |
| CAT-INV-2 | invariant (A) | `1 <= policy().min_facings_per_sku <= policy().max_facings_per_sku` | `[TB 2.3]` |
| CAT-INV-3 | invariant (A) | `0 < policy().space_elasticity < 1` — incremental facings show diminishing returns | `[TB 2.3]` |
| CAT-INV-4 | invariant (A) | `0 <= policy().private_label_space_target_pct <= 100` | `[TB 2.5]` |
| CAT-INV-5 | invariant (A) | `policy().reset_cycle_months >= 1` (typical range 3–12) | `[VS 4]` `[TB 2.6]` |
| CAT-INV-6 | invariant (A) | `all(l.assigned_category is self and any(l.space_type.is_within(t) for t in policy().allowed_space_types) for l in assigned_spaces())` | `[TB 2.1]` `[SIM]` |
| CAT-INV-7 | invariant (A) | `all(s.category is self for s in skus)` and `(len(skus) == 0 or self.is_leaf)` | `[SIM]` |
| CAT-INV-9 | invariant (A) | `len(policy().allowed_space_types) >= 1` and every merchandising leaf type `u` within an allowed type has `temperature_zone(u) == policy().temperature_zone` — a refrigerated category can't be offered ambient fixtures | `[TB 4.4]` |
| CAT-INV-10 | invariant (A) | `all(not FLOOR_DISPLAY.is_within(t) and not t.is_within(FLOOR_DISPLAY) for t in policy().allowed_space_types)` — floor displays are never category space; they carry promotional placements only (FD-INV-2) | `[SIM]` |
| CAT-REV-1 | review | `policy().blocking_mode == HORIZONTAL_TIER` only for a category with a dominant private label or a clear tier structure | `[VS 12]` |
| CAT-Q-2 | ensure | `effective_last_reset_date()`: `last_reset_date` of the nearest node in `(self,) + self.ancestors()` with a non-`None` value, else `None` | `[VS 4]` |
| CAT-Q-3 | ensure | `next_reset_due(cal)`: `result is None if effective_last_reset_date() is None else min((d for d in cal if d >= add_months(effective_last_reset_date(), policy().reset_cycle_months)), default=None)` — resets happen on scheduled windows across the chain, not rolling per vendor | `[VS 4]` |
| CAT-Q-4 | ensure | `private_label_space_share(on)`: `result == 0.0 if T == 0 else 100 * PL / T`, where `T` is the total `extent` of `PERMANENT` placements live on `on` on `LINEAR_IN` leaves assigned within `self.subtree()`, and `PL` is the part of `T` for private-label SKUs. Reporting only | `[TB 2.5]` |
| CAT-Q-5 | ensure | `subtree_skus()`: `result == frozenset(s for n in self.subtree() for s in n.skus)` | `[VS 6]` |
| CAT-Q-6 | ensure | `assigned_spaces()`: `result == tuple(l for l in store.allocatable_leaves() if l.assigned_category is self)`, in the store's leaf order | `[SIM]` |

`CategoryNode` conforms to `Rollup` (Part 2). The private-label target `[TB 2.5]` and department space shares `[TB 1.2]` are **objectives, not postconditions** in this version: reported, not asserted (Q-14).

**Examples (CategoryNode)**

```python
rte = s4.category("CAT-RTE")
# ✓ CAT-NEW-1, CAT-Q-1: max from Cereal, the rest from Dry Grocery
assert (rte.policy().max_facings_per_sku, rte.policy().min_facings_per_sku) == (12, 1)
assert s4.category("CAT-YOG").policy().private_label_benchmark_brand == "Dairyland"
# ✓ CAT-Q-2, CAT-Q-3: last reset 2026-01-05; due 6 months on, at the next scheduled window
assert rte.effective_last_reset_date() == D_0105 and rte.next_reset_due(CALENDAR) == D_0706
assert s5.category("CAT-RTE").next_reset_due(CALENDAR) is None    # 2027-01-06 is after the last window
# ✓ CAT-Q-4 at S6: StoreBrand has 4 of Cereal's 81 inches; 5 of Dairy's 35
assert abs(s6.category("CAT-CER").private_label_space_share(D_0915) - 400 / 81) < EPS
assert abs(s6.category("CAT-DY").private_label_space_share(D_0915) - 500 / 35) < EPS
# ✓ CAT-Q-5, CAT-Q-6
assert {s.sku_id for s in s1.category("CAT-CER").subtree_skus()} == {f"SKU-{k}" for k in "ABCDEFG"}
assert [l.id for l in s1.category("CAT-CER").assigned_spaces()] == ["RUN-1L1-E", "RUN-1L1-B"]
assert s1.category("CAT-YOG").assigned_spaces() == ()           # Dairy owns the cooler
# ✗ CAT-NEW-PRE-1
with expect_violation(PreconditionViolation, "CAT-NEW-PRE-1"):
    CategoryNode("CAT-X", "x", "segment", {"colour": "red"})
# ✗ CAT-NEW-PRE-2: CAT-RTE already has a parent
with expect_violation(PreconditionViolation, "CAT-NEW-PRE-2"):
    CategoryNode("CAT-X", "x", "category", {}, [s0.category("CAT-RTE")])
# ✗ CAT-NEW-PRE-3
with expect_violation(PreconditionViolation, "CAT-NEW-PRE-3"):
    CategoryNode("CAT-X", "x", "segment", {"space_elasticity": 1.5})
# ✗ supplier CAT-Q-1: taking the root's value where a nearer node overrides it
assert not P.CAT_Q_1(rte, Stub(max_facings_per_sku=6), None)
# ✗ inv CAT-INV-9: a refrigerated category allowed open gondolas (stub)
assert not P.CAT_INV_9(Stub(policy=lambda: Stub(temperature_zone=TemperatureZone.REFRIGERATED,
                                              allowed_space_types=frozenset({SpaceType.GONDOLA}))))
# ✗ inv CAT-INV-10: floor displays as category space (stub)
assert not P.CAT_INV_10(Stub(policy=lambda: Stub(allowed_space_types=frozenset({SpaceType.MERCHANDISING}))))
# ✗ inv CAT-INV-2 (stub)
assert not P.CAT_INV_2(Stub(policy=lambda: Stub(min_facings_per_sku=3, max_facings_per_sku=2)))
```

---

### 4.4 `BusinessUnit` — the business-unit hierarchy `[VS 6, 9]`

Real chains have division, banner, region, store, and department. The hierarchy has arbitrary depth, and `kind` is a free label.

```
class BusinessUnit           (conforms to HierarchyNode and Rollup)
creation  BusinessUnit(id, name, kind, root_category=None, children=())
queries
    id, name, kind: str                         -- kind e.g. "COMPANY", "DIVISION", "BANNER", "REGION", "STORE", "DEPARTMENT"
    root_category: CategoryNode | None          -- non-None only for kind == "DEPARTMENT"
```

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| BU-NEW-PRE-1 | require | `id.startswith("BU-")`, `kind != ""`, and BU-INV-1 holds for `kind`, `root_category`, and `children`; a given `root_category` is a root (`parent is None`) | `[VS 6]` |
| BU-NEW-PRE-2 | require | `children` are unattached `BusinessUnit`s | `[DbC]` |
| BU-NEW-1 | ensure | attributes as given; `children == tuple(children)`, each child's `parent is self` | `[DbC]` |
| BU-INV-1 | invariant | `(root_category is not None) == (kind == "DEPARTMENT")` and `kind == "DEPARTMENT" ⇒ is_leaf` | `[VS 6]` `[SIM]` |
| BU-INV-2 | invariant (A) | every root `CategoryNode` of the store is the `root_category` of exactly one `BusinessUnit` | `[VS 6]` `[SIM]` |
| BU-INV-3 | invariant (A) | exactly one unit has `kind == "STORE"`; it is `Store.business_unit`, and every `DEPARTMENT` unit is within it (single store, Q-23) | `[SIM]` |
| BU-Q-1 | ensure | `SKU.business_unit()`: `result.root_category is sku.department` | `[VS 6, 9]` |

A placement's business unit is derived (PLC-Q-2), never stored.

**Examples (BusinessUnit)**

```python
# ✓ BU-NEW-1, BU-Q-1
assert [c.id for c in s0.business_unit_node("BU-ST").children] == ["BU-DG", "BU-DY", "BU-PR"]
assert s0.skus["SKU-Y1"].business_unit().id == "BU-DY"
# ✓ Rollup over business units at S6 (RU-1)
assert s6.business_unit_node("BU-ST").facings(D_0915) == 62 and s6.business_unit_node("BU-DG").facings(D_0915) == 23
# ✗ BU-NEW-PRE-1: a department with no root category
with expect_violation(PreconditionViolation, "BU-NEW-PRE-1"):
    BusinessUnit("BU-X", "x", "DEPARTMENT")
# ✗ BU-NEW-PRE-2
with expect_violation(PreconditionViolation, "BU-NEW-PRE-2"):
    BusinessUnit("BU-X", "x", "REGION", children=[s0.business_unit_node("BU-ST")])
# ✗ supplier BU-Q-1: Yogurt's SKU reported under Dry Grocery
assert not P.BU_Q_1(s0.skus["SKU-Y1"], s0.business_unit_node("BU-DG"), None)
# ✗ inv BU-INV-1: a region that names a root category (stub)
assert not P.BU_INV_1(Stub(kind="REGION", root_category=s0.category("CAT-DG"), is_leaf=True))
```

---
## Part 5 — Agreements and placements

A placement is the **commitment**: what was agreed to give a vendor, where, and how much. It is not just a physical fact. `[VS 2]` Three things are kept apart throughout the model: the **planogram** as intended state (Part 6), the **placement** as agreed commitment (this part), and **observed shelf state** as what is actually there (reserved, Part 10). `[VS 10]`

### 5.1 `Agreement` — the ladder of deals `[VS 3]`

A separate class, not a column on the placement. The ladder: slotting agreements for permanent shelf space, tied to reset windows; promotional display deals of typically four to eight weeks, priced per display per store; pay-to-stay fees for holding existing space; and opportunistic buys with no space commitment at all. `[VS 3]` Slotting is ongoing rented space, priced by facings or linear feet and renewed at reset (companion T-1).

```
class Agreement
creation  Agreement(agreement_id, agreement_type, vendor, fee_basis, fee_rate, window)
queries
    agreement_id: str
    agreement_type: AgreementType
    vendor: str                              -- manufacturer name
    fee_basis: FeeBasis
    fee_rate: float                          -- dollars per basis unit, for the whole term
    window: DateWindow
    placements: tuple[PlacementRecord, ...]  -- (A) every placement that references this agreement
    term_fee(p: Placement) -> float
```

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| AGR-NEW-PRE-1 | require | `agreement_id` is `"AGR-"` plus six digits (ID-2) and `vendor != ""` | `[SIM]` |
| AGR-NEW-PRE-2 | require | AGR-INV-1, AGR-INV-3, AGR-INV-4, and AGR-INV-5 hold for the arguments | `[VS 3]` |
| AGR-NEW-1 | ensure | attributes as given; `placements == ()` until attached | `[DbC]` |
| AGR-INV-1 | invariant | `fee_rate >= 0` | `[VS 3]` |
| AGR-INV-2 | invariant (A) | `agreement_type == SLOTTING ⇒ window.start in store.reset_calendar` — slotting is tied to reset windows | `[VS 3, 4]` |
| AGR-INV-3 | invariant | `agreement_type == PROMOTIONAL_DISPLAY ⇒ window.end is not None and fee_basis in {PER_DISPLAY, FLAT}` | `[VS 3]` |
| AGR-INV-4 | invariant | `agreement_type == OPPORTUNISTIC_BUY ⇒ fee_rate == 0` — the deal is on the product, not the shelf | `[VS 3]` |
| AGR-INV-5 | invariant | `agreement_type in {SLOTTING, PAY_TO_STAY} ⇒ fee_basis in {PER_FACING, PER_LINEAR_FT, FLAT}` | `[VS 3]` |
| AGR-INV-6 | invariant (A) | `all(compatible(agreement_type, p.placement_type) and window_covers(window, p.window) for p in placements)` | `[VS 3]` |
| AGR-INV-7 | invariant (A) | `all(p.sku.manufacturer == vendor for p in placements)` | `[VS 3]` |
| AGR-PRE-1 | require | `term_fee(p)`: `p in self.placements`, `p.window.end is not None`, and `fee_basis != PER_LINEAR_FT or p.space.unit == LINEAR_IN` | `[VS 3]` |
| AGR-Q-1 | ensure | `term_fee(p)`: `result == fee_rate * q`, where `q` is `p.facings` for `PER_FACING`; `p.extent / 12` for `PER_LINEAR_FT`; `1` for `PER_DISPLAY` and `FLAT` | `[VS 3]` |

Space carries no price (rule 8). An end cap costs more only because the rate negotiated for a placement on it is higher.

**Examples (Agreement)** — at S6

```python
agr1, agr2 = s6.agreements["AGR-000001"], s6.agreements["AGR-000002"]
# ✓ AGR-NEW-1, AGR-INV-6: the slotting deal covers SKU-A's and SKU-B's permanent placements
assert {p.placement_id for p in agr1.placements} == {"PLC-000003", "PLC-000004"}
# ✓ AGR-Q-1: 24 in = 2 ft × $2.00; one display × $500
assert agr1.term_fee(s6.placements["PLC-000004"]) == 4.0
assert agr2.term_fee(s6.placements["PLC-000012"]) == 500.0
# ✗ AGR-NEW-PRE-1: not a six-digit id
with expect_violation(PreconditionViolation, "AGR-NEW-PRE-1"):
    Agreement("AGR-1", AgreementType.PAY_TO_STAY, "Acme Foods", FeeBasis.FLAT, 100.0, DateWindow(D_0105, None))
# ✗ AGR-NEW-PRE-2: an opportunistic buy with a fee
with expect_violation(PreconditionViolation, "AGR-NEW-PRE-2"):
    Agreement("AGR-000099", AgreementType.OPPORTUNISTIC_BUY, "Acme Foods", FeeBasis.FLAT, 10.0, DateWindow(D_0105, None))
# ✗ AGR-PRE-1: PLC-000003 is open-ended, so its term has no fee yet
with expect_violation(PreconditionViolation, "AGR-PRE-1"):
    agr1.term_fee(s6.placements["PLC-000003"])
# ✗ supplier AGR-Q-1: charging per inch instead of per foot
assert not P.AGR_Q_1(agr1, 48.0, None, p=s6.placements["PLC-000004"])
# ✗ inv AGR-INV-7: a deal with Acme that funds an Oatco placement (stub)
assert not P.AGR_INV_7(Stub(vendor="Acme Foods", placements=(s6.placements["PLC-000010"],)))
# ✗ inv AGR-INV-2: a slotting deal starting off the reset calendar (stub)
assert not P.AGR_INV_2(Stub(agreement_type=AgreementType.SLOTTING, window=DateWindow(D_0901, None),
                            store=Stub(reset_calendar=CALENDAR)))
```

### 5.2 `PlacementRecord` (deferred) and `Placement` (committed) `[VS 2, 3, 4, 8]`

```
deferred class PlacementRecord
queries
    placement_id: str
    sku: SKU
    placement_type: PlacementType
    start_date: date
    end_date: date | None                      -- exclusive; None = open
    window: DateWindow                         -- derived
    agreement: Agreement | None
    vendor_funded: bool                        -- derived
    business_unit: BusinessUnit                -- derived (A)
```

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| PREC-Q-1 | ensure | `window == DateWindow(start_date, end_date)` | `[VS 4]` |
| PREC-Q-2 | ensure | `vendor_funded == (agreement is not None and agreement.fee_rate > 0)` | `[VS 3]` |
| PREC-Q-3 | ensure | `business_unit is sku.business_unit()` — derived from the SKU, never entered | `[VS 9]` |

Both `Placement` and `IncidentalPlacement` (5.3) conform. "All placements live on a given date" returns both kinds.

```
class Placement                                -- conforms to PlacementRecord
creation  Placement(placement_id, sku, space, placement_type, extent, offset_in,
                    start_date, end_date=None, agreement=None)
queries
    space: LeafSpace
    placement_type: PlacementType              -- PERMANENT or PROMOTIONAL
    extent: float                              -- allocated extent, in space.unit
    offset_in: float | None                    -- from the leaf's left edge; LINEAR_IN leaves only
    -- derived (never stored)
    facings: int
    units_capacity: int
    shelf_level: ShelfLevel | None
    space_type: SpaceType
```

A placement stores the **allocated extent**, in the space's unit. Facings are derived from it and never stored. `[VS 8]` Facings and linear space assigned are one fact, not two.

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| PLC-NEW-PRE-1 | require | `placement_id` is `"PLC-"` plus six digits (ID-2); `sku` is a `SKU`, `space` a `LeafSpace`, `agreement` an `Agreement` or `None` | `[SIM]` |
| PLC-NEW-PRE-2 | require | PLC-INV-2, PLC-INV-8, and PLC-INV-9 hold for the arguments (type and window) | `[VS 3, 4]` |
| PLC-NEW-PRE-3 | require | PLC-INV-3, then PLC-INV-1, PLC-INV-4, and PLC-INV-5, hold for the arguments (physical fit; in that order, so facings are derived only for a placement that fits) | `[VS 8]` |
| PLC-NEW-PRE-4 | require | PLC-INV-6, PLC-INV-10, and PLC-INV-11 hold for the arguments (category policy) | `[TB 2.3, 4.4, 6.4]` |
| PLC-NEW-PRE-5 | require | PLC-INV-12 holds for the arguments (agreement) | `[VS 3]` |
| PLC-NEW-1 | ensure | attributes as given; the placement is detached: it is not yet in `space.placements` or `sku.placements` | `[DbC]` |
| PLC-Q-1 | ensure | `facings`: `result == floor(extent / sku.product.facing_extent(space.unit) + EPS)` | `[VS 8]` |
| PLC-Q-2 | ensure | `shelf_level == space.shelf_level` and `space_type == space.space_type` — derived from where the SKU sits, never entered | `[VS 8]` `[DbC]` |
| PLC-Q-3 | ensure | `units_capacity == sku.product.units_capacity_for(space, extent)` | `[VS 8]` |
| PLC-INV-1 | invariant | `extent > 0` and `facings >= 1` | `[VS 8]` |
| PLC-INV-2 | invariant | `placement_type in {PERMANENT, PROMOTIONAL}` | `[VS 3]` |
| PLC-INV-3 | invariant | `fits_dimensionally(sku.product, space)` (FIT-1) | `[VS 8]` |
| PLC-INV-4 | invariant | **Whole facings.** `sku.product.form != BOX or abs(extent - facings * sku.product.facing_extent(space.unit)) <= EPS`; and for a `Bin`, `extent == space.leaf_capacity` (a bin is allocated whole). A non-box product may have slack, because its extent is an override | `[VS 8]` `[SIM]` |
| PLC-INV-5 | invariant | `(offset_in is None) == (space.unit != LINEAR_IN)`; if linear, `0 <= offset_in and offset_in + extent <= space.leaf_capacity + EPS` | `[VS 8]` |
| PLC-INV-6 | invariant | `space.temperature_zone == sku.category.policy().temperature_zone` — the cold chain is unbroken on the shelf | `[TB 4.4]` |
| PLC-INV-7 | invariant (A) | `placement_type != PERMANENT or (space.assigned_category is not None and sku.category.is_within(space.assigned_category))` — permanent space belongs to the SKU's category or an ancestor of it | `[TB 2.1]` |
| PLC-INV-8 | invariant | `end_date is None or start_date < end_date` — an end that precedes its start is a violation | `[VS 14]` |
| PLC-INV-9 | invariant | `placement_type != PROMOTIONAL or (end_date is not None and sku.promo_display_eligible)` — promotional placements are time-boxed | `[VS 3]` |
| PLC-INV-10 | invariant | `placement_type != PERMANENT or sku.product.form == LOOSE or (policy.min_facings_per_sku <= facings <= policy.max_facings_per_sku)`, with `policy = sku.category.policy()` | `[TB 2.3]` |
| PLC-INV-11 | invariant | `not sku.category.policy().secured_case_required or space.secured` | `[TB 6.4]` |
| PLC-INV-12 | invariant | `agreement is None or (compatible(agreement.agreement_type, placement_type) and window_covers(agreement.window, window) and agreement.vendor == sku.manufacturer)` | `[VS 3]` |
| PLC-INV-13 | invariant (A) | `self in space.placements and self in sku.placements` | `[DbC]` |
| PLC-INV-14 | invariant (A) | `placement_type != PERMANENT or start_date in store.reset_calendar or start_date in {v.effective_date for v in store.planogram.versions if v.kind == REVISION}` — permanent placements start at a reset, or at a released mid-cycle revision | `[VS 4, 10]` |
| PLC-INV-IMM | invariant | **Immutable facts.** After creation, no attribute of a `Placement` changes except `end_date`, which may change only once, from `None` to a date `> start_date`, through `end_placement` or `execute_version`. Assigning any attribute directly raises `InvariantViolation("PLC-INV-IMM")`. A placement is deleted only by `cancel_future_placement` (7.3). So "what was on this shelf on this date" is one uniform query | `[VS 4]` |

No archive table exists: even years of history are tens of thousands of rows, all kept live. `[VS 4]` Placement type does not mark "primary"; the planogram's home position does (6.5). `[VS 2]`

**Examples (Placement)** — at S6

```python
p3, p12 = s6.placements["PLC-000003"], s6.placements["PLC-000012"]
a, y1 = s6.skus["SKU-A"], s6.skus["SKU-Y1"]
run_e = s6.space("RUN-1L1-E")
# ✓ PLC-Q-1, PLC-Q-2, PLC-Q-3, PREC-Q-1..3
assert (p3.facings, p3.units_capacity, p3.shelf_level) == (6, 18, ShelfLevel.EYE_LEVEL)
assert (p12.facings, p12.units_capacity, p12.shelf_level) == (3, 3, None)
assert p3.window == DateWindow(D_0105, None) and p3.business_unit.id == "BU-DG"
assert p12.vendor_funded and p3.vendor_funded and not s6.placements["PLC-000007"].vendor_funded
# ✓ PLC-NEW-1: a detached record (not yet on its leaf)
q = Placement("PLC-000099", a, s6.space("RUN-1L2-E"), PlacementType.PERMANENT, 8.0, 0.0, D_0706)
assert q not in s6.space("RUN-1L2-E").placements
# ✗ PLC-NEW-PRE-1
with expect_violation(PreconditionViolation, "PLC-NEW-PRE-1"):
    Placement("PLC-99", a, run_e, PlacementType.PERMANENT, 8.0, 0.0, D_0706)
# ✗ PLC-NEW-PRE-2: end precedes start
with expect_violation(PreconditionViolation, "PLC-NEW-PRE-2"):
    Placement("PLC-000099", a, run_e, PlacementType.PERMANENT, 8.0, 0.0, D_0929, D_0901)
# ✗ PLC-NEW-PRE-3: 10 in is not a whole number of 8 in facings
with expect_violation(PreconditionViolation, "PLC-NEW-PRE-3"):
    Placement("PLC-000099", a, run_e, PlacementType.PERMANENT, 10.0, 0.0, D_0706)
# ✗ PLC-NEW-PRE-4: refrigerated yogurt on an ambient gondola shelf
with expect_violation(PreconditionViolation, "PLC-NEW-PRE-4"):
    Placement("PLC-000099", y1, run_e, PlacementType.PERMANENT, 10.0, 0.0, D_0706)
# ✗ PLC-NEW-PRE-5: a promotional-display deal on a permanent placement
with expect_violation(PreconditionViolation, "PLC-NEW-PRE-5"):
    Placement("PLC-000099", a, run_e, PlacementType.PERMANENT, 8.0, 0.0, D_0901, D_0929,
              agreement=s6.agreements["AGR-000002"])
# ✗ inv PLC-INV-IMM: no attribute can be reassigned
with expect_violation(InvariantViolation, "PLC-INV-IMM"):
    p3.extent = 3.0
# ✗ supplier PLC-Q-1: rounding facings up
assert not P.PLC_Q_1(Stub(extent=20.0, sku=a, space=run_e), 3, None)
# ✗ inv PLC-INV-14: a permanent placement starting off-calendar (stub)
assert not P.PLC_INV_14(Stub(placement_type=PlacementType.PERMANENT, start_date=date(2026, 3, 1),
                             store=Stub(reset_calendar=CALENDAR, planogram=Stub(versions=()))))
# ✗ inv PLC-INV-7: permanent placement on space assigned to another category (stub)
assert not P.PLC_INV_7(Stub(placement_type=PlacementType.PERMANENT, sku=y1, space=run_e))
```

### 5.3 `IncidentalPlacement` — no space commitment `[VS 3]`

Overstock, or a DSD drop stacked wherever there was floor that week. It has no fixed position, so the model refuses to invent one: no leaf, no extent, no offset. `[VS 3]`

```
class IncidentalPlacement                      -- conforms to PlacementRecord
creation  IncidentalPlacement(placement_id, sku, units_held, start_date, end_date=None,
                              agreement=None, zone_hint=None)
queries
    placement_type: PlacementType              -- always INCIDENTAL
    units_held: int
    zone_hint: SpaceNode | None                -- coarse, best-effort, and may change
```

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| INC-NEW-PRE-1 | require | `placement_id` is `"INC-"` plus six digits (ID-2); `sku` is a `SKU`; `zone_hint` is `None` or a `SpaceNode` | `[SIM]` |
| INC-NEW-PRE-2 | require | INC-INV-1, INC-INV-2, and INC-INV-3 hold for the arguments | `[VS 3]` |
| INC-NEW-1 | ensure | attributes as given, `placement_type == INCIDENTAL`; detached | `[DbC]` |
| INC-INV-1 | invariant | `placement_type == INCIDENTAL and units_held >= 1` | `[VS 3]` |
| INC-INV-2 | invariant | `end_date is None or start_date < end_date` | `[VS 14]` |
| INC-INV-3 | invariant | `agreement is None or (agreement.agreement_type == OPPORTUNISTIC_BUY and window_covers(agreement.window, window))` | `[VS 3]` |
| INC-INV-4 | invariant | the class has no `space`, `extent`, `facings`, or `offset_in` feature. It contributes to no `Rollup` measure and to no `LeafSpace` allocation | `[VS 3]` |
| INC-INV-5 | invariant | `zone_hint` is the one mutable feature besides `end_date`, changed only by `move_incidental` (7.3). It is never used to compute a measure, only by the filter `placements_in_space` (FLT-3) | `[VS 3]` `[SIM]` |

Cold-chain rules for incidental placements are deferred to Restocking and Receiving (Part 10).

**Examples (IncidentalPlacement)**

```python
inc = s6.incidentals["INC-000001"]
# ✓ INC-NEW-1, INC-INV-4
assert (inc.units_held, inc.zone_hint.id, inc.placement_type) == (24, "SID-1L", PlacementType.INCIDENTAL)
assert not hasattr(inc, "extent") and not inc.vendor_funded
# ✗ INC-NEW-PRE-1: zone hint must be a space node
with expect_violation(PreconditionViolation, "INC-NEW-PRE-1"):
    IncidentalPlacement("INC-000099", s6.skus["SKU-B"], 12, D_0910, zone_hint="SID-1L")
# ✗ INC-NEW-PRE-2: zero units
with expect_violation(PreconditionViolation, "INC-NEW-PRE-2"):
    IncidentalPlacement("INC-000099", s6.skus["SKU-B"], 0, D_0910)
# ✗ inv INC-INV-3: a slotting deal on an incidental placement (stub)
assert not P.INC_INV_3(Stub(agreement=s6.agreements["AGR-000001"], window=DateWindow(D_0910, None)))
```

---
## Part 6 — Allocation: policy, plan, and planogram versions

### 6.1 Pools and planes

Space is assigned to category nodes (7.3), at any depth. A SKU draws on the space of its **pool**: the nearest category node above it that owns space in the SKU's plane.

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| POOL-1 | ensure | `primary_plane(sku)` (module function): `BOX`, `SOFT_PACK` → `FRONT`; `HANGING` → `PEG`; `LOOSE` → `TOP` (Q-6) | `[VS 8]` `[SIM]` |
| POOL-2 | ensure | `pool_node(sku, plane)`: the first node `n` in `(sku.category,) + sku.category.ancestors()` with `any(l.presentation_plane == plane for l in n.assigned_spaces())`, else `None` (Q-2) | `[SIM]` |
| POOL-3 | ensure | `pool_leaves(node, plane)`: `tuple(sorted((l for l in node.assigned_spaces() if l.presentation_plane == plane), key=lambda l: (policy.level_preference(l.shelf_level), l.id)))` — best shelf level first, then leaf id | `[VS 12]` `[SIM]` |
| POOL-4 | ensure | `pool_capacity(node, plane)`: `sum(l.leaf_capacity for l in pool_leaves(node, plane))`, in `plane_unit(plane)` | `[SIM]` |

Because a pool is single-plane and LS-INV-11 holds, every pool is either all ranked shelves (`FRONT`) or all unranked leaves (`TOP` bins, `PEG` sections and clip strips). A pool never mixes the two.

**Examples (pools)** — at S2

```python
sk = s2.skus
# ✓ POOL-1, POOL-2
assert primary_plane(sk["SKU-S1"]) is PresentationPlane.PEG and primary_plane(sk["SKU-P1"]) is PresentationPlane.TOP
assert pool_node(sk["SKU-A"], PresentationPlane.FRONT).id == "CAT-CER"        # shared with Hot Cereal
assert pool_node(sk["SKU-D"], PresentationPlane.FRONT).id == "CAT-CER"
assert pool_node(sk["SKU-G"], PresentationPlane.PEG) is None and pool_node(sk["SKU-H"], PresentationPlane.FRONT) is None
# ✓ POOL-3, POOL-4
assert [l.id for l in pool_leaves(s2.category("CAT-DY"), PresentationPlane.FRONT)] == ["RUN-D1-E", "RUN-D1-M", "RUN-D1-T", "RUN-D1-B"]
assert pool_capacity(s2.category("CAT-CER"), PresentationPlane.FRONT) == 96.0
assert pool_capacity(s2.category("CAT-SPI"), PresentationPlane.PEG) == 4.0
# ✗ supplier POOL-3: leaves in tree order (top shelf first)
assert not P.POOL_3(None, tuple(s2.space(i) for i in ("RUN-D1-T", "RUN-D1-E", "RUN-D1-M", "RUN-D1-B")), None,
                    node=s2.category("CAT-DY"), plane=PresentationPlane.FRONT)
```

### 6.2 `AllocationPolicy` — the scoring rules allocation obeys `[TB 2.3]`

The textbook's allocation inputs are sales velocity, gross margin, vendor category-captain input, private-label targets, and space elasticity with diminishing returns `[TB 2.3]`. The policy turns them into a deterministic target.

```
class AllocationPolicy
creation  AllocationPolicy()
queries
    score(sku: SKU) -> float
    target_facings(sku: SKU, in_scope: Sequence[SKU], as_of: date) -> int | None
    level_preference(level: ShelfLevel | None) -> int
    captain_weight: float
```

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| POL-NEW-1 | ensure | `captain_weight == 1.0` | `[DbC]` |
| POL-1 | ensure | `score(sku) == sku.weekly_gross_margin` — velocity × margin. GMROI is the alternative (Q-5) | `[TB 2.3]` |
| POL-PRE-1 | require | `target_facings(s, S, as_of)`: `s in S` and `s.in_season(as_of)` | `[SIM]` |
| POL-2 | ensure | `target_facings(s, S, as_of)`: `None` if `s.product.form == LOOSE` or `pool_node(s, primary_plane(s)) is None`. Otherwise let `p = primary_plane(s)`, `A = pool_node(s, p)`, `e = A.policy().space_elasticity`, `C = pool_capacity(A, p)`, `f = s.product.facing_extent(plane_unit(p))`, `competitors = [i for i in S if i.in_season(as_of) and primary_plane(i) == p and pool_node(i, p) is A]`, `w(i) = max(score(i), 0) ** (1 / (1 - e))`, and `share = w(s) / sum(w(i) for i in competitors)` (or `1 / len(competitors)` if every `w(i) == 0`). `result == clamp(floor(share * C / f + EPS), lo, hi)`, with `lo, hi = s.category.policy().min_facings_per_sku, s.category.policy().max_facings_per_sku`. This is the best split of space when each SKU's sales grow as space^e (Q-4) | `[TB 2.3]` `[SIM]` |
| POL-3 | ensure | `target_facings` is monotone in score within a pool: for `a`, `b` in the same pool and plane, `score(a) >= score(b) and a.product.facing_extent(u) <= b.product.facing_extent(u) ⇒ target_facings(a) >= target_facings(b)` | `[TB 2.3]` |
| POL-4 | ensure | `level_preference(l) == l.preference_rank` for a shelf level and `0` for `None`. **Horizontal position carries no preference**: bay position and offset never affect allocation order | `[VS 12]` |
| POL-5 | ensure | `captain_weight == 1.0` in this version: the captain's input does not affect allocation, and private-label placement is a retailer decision the captain works around (Q-7) | `[VS 12]` `[TB 2.4]` |

**Examples (AllocationPolicy)** — at S2, `S = s2.in_scope_skus(scope_all(s2), D_0105)`

```python
pol, sk = s2.policy, s2.skus
S = s2.in_scope_skus(scope_all(s2), D_0105)
# ✓ POL-NEW-1, POL-1
assert pol.captain_weight == 1.0 and pol.score(sk["SKU-A"]) == 30.0 and pol.score(sk["SKU-G"]) == 2.0
# ✓ POL-2: Cereal pool, weights 30², 20², 10², 5², 5² = 900, 400, 100, 25, 25 (SKU-F is out of season)
#   SKU-A: floor(900/1450 × 96 / 8) = 7; SKU-B: floor(400/1450 × 96 / 3) = 8; SKU-C: 1; SKU-D: 0 → min 1
assert [pol.target_facings(sk[k], S, D_0105) for k in ("SKU-A", "SKU-B", "SKU-C", "SKU-D")] == [7, 8, 1, 1]
assert pol.target_facings(sk["SKU-Y1"], S, D_0105) == 6        # 22 clamped to Dairy's max of 6
assert pol.target_facings(sk["SKU-S1"], S, D_0105) == 3        # floor(144/189 × 4)
assert pol.target_facings(sk["SKU-P1"], S, D_0105) is None and pol.target_facings(sk["SKU-G"], S, D_0105) is None
# ✓ POL-4
assert pol.level_preference(ShelfLevel.EYE_LEVEL) == 1 and pol.level_preference(None) == 0
# ✗ POL-PRE-1: SKU-F is out of season in January
with expect_violation(PreconditionViolation, "POL-PRE-1"):
    pol.target_facings(sk["SKU-F"], S, D_0105)
# ✗ supplier POL-2: ignoring the category's maximum
assert not P.POL_2(pol, 22, None, s=sk["SKU-Y1"], S=S, as_of=D_0105)
```

### 6.3 `PlacementPlan` — the result of allocation (a pure value)

```
value PlannedPlacement
creation  PlannedPlacement(sku, space, extent, offset_in, agreement=None)
    sku: SKU; space: LeafSpace; extent: float; offset_in: float | None; agreement: Agreement | None

class PlacementPlan
creation  PlacementPlan(scope, as_of, entries, home, unplaced, under_target, delist_review,
                        block_breaks, adjacency_breaks)      -- used by generate_plan and with_agreement
queries
    scope: frozenset[CategoryNode]
    as_of: date                                  -- the effective date the layout is planned for
    entries: tuple[PlannedPlacement, ...]        -- PERMANENT placements only, in allocation order
    home: Mapping[SKU, PlannedPlacement]
    unplaced: Mapping[SKU, UnplacedReason]
    under_target: tuple[SKU, ...]
    delist_review: tuple[SKU, ...]               -- reporting only [TB 2.3, 2.6]
    block_breaks: tuple[tuple[str, str], ...]    -- (block group id, brand), sorted
    adjacency_breaks: tuple[str, ...]            -- block group ids, sorted
    with_agreement(sku: SKU, agreement: Agreement) -> PlacementPlan
```

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| PP-NEW-PRE-1 | require | `extent > 0`; `(offset_in is None) == (space.unit != LINEAR_IN)`; `agreement` is `None` or an `Agreement` | `[VS 8]` |
| PP-NEW-1 | ensure | attributes as given | `[DbC]` |
| PLAN-NEW-PRE-1 | require | PLAN-INV-1, PLAN-INV-2, PLAN-INV-3, PLAN-INV-5, and PLAN-INV-7 hold for the arguments, and `len(scope) >= 1` | `[DbC]` |
| PLAN-NEW-1 | ensure | attributes as given (immutable copies) | `[DbC]` |
| PLAN-INV-1 | invariant | `{e.sku for e in entries}.isdisjoint(unplaced.keys())` | `[DbC]` |
| PLAN-INV-2 | invariant | every entry's `sku.category` is within some node of `scope` | `[DbC]` |
| PLAN-INV-3 | invariant | at most one entry per SKU | `[SIM]` |
| PLAN-INV-4 | invariant (A) | **Feasibility.** The entries, taken as `PERMANENT` placements with window `[as_of, None)`, with every existing `PERMANENT` placement of an in-scope SKU treated as ended at `as_of` (or removed, if it starts after `as_of`), satisfy LS-INV-5, LS-INV-6, BIN-INV-2, FD-INV-2, and PLC-INV-1 … PLC-INV-12 against the leaves they name | `[VS 14]` |
| PLAN-INV-5 | invariant | `set(under_target) <= {e.sku for e in entries}` | `[DbC]` |
| PLAN-INV-6 | invariant (A) | **Delist review.** `delist_review` is the in-scope SKUs, placed or not, whose `assortment_status == DELIST_CANDIDATE`, or whose 0-based rank among the in-scope SKUs of the same leaf category, by ascending `(score, sku_id)`, is `< floor(n / 10)`, where `n` is that category's in-scope count (the bottom decile). Ordered by `(sku.category.path, score, sku_id)`. A category with fewer than 10 in-scope SKUs flags only explicit candidates | `[TB 2.3, 2.6]` `[SIM]` |
| PLAN-INV-7 | invariant | `set(home) == {e.sku for e in entries}` and `all(home[s].sku is s and home[s] in entries for s in home)` | `[VS 2]` |
| PLAN-INV-8 | invariant (A) | `block_breaks` and `adjacency_breaks` equal the values BLK-3 and BLK-4 compute from `entries` — the plan reports honestly | `[VS 12]` |
| PLAN-PRE-1 | require | `with_agreement(s, a)`: `s` has an entry, `a.agreement_type in {SLOTTING, PAY_TO_STAY}`, `a.vendor == s.manufacturer`, and `window_covers(a.window, DateWindow(as_of, None))` | `[VS 3]` |
| PLAN-Q-1 | ensure | `with_agreement(s, a)`: `result` equals `self` except that `s`'s entry, and `home[s]`, carry `a`; `self` is unchanged | `[VS 3]` `[DbC]` |

A `PlacementPlan` is immutable, and building one has no side effects. A plan is what a category manager edits before release; it is a different object from any released version (6.5). `[VS 10]`

**Brand blocking `[VS 12]`.** Adjacency is specified in the planogram and is often part of what the vendor negotiated.

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| BLK-3 | ensure | `block_breaks`: group the entries by `(pool A, group G = e.space.block_group())` and brand. For `VERTICAL_BRAND` (the default): let `P_b` be the set of `e.space.bay_position()` of brand `b`'s entries in `G`; `(G.id, b)` is a break iff some position `q` with `min(P_b) < q < max(P_b)`, `q ∉ P_b`, holds an entry of another brand. For `HORIZONTAL_TIER`: `(G.id, b)` is a break iff `b`'s entries in `G` span more than one shelf level. Sorted | `[VS 12]` |
| BLK-4 | ensure | `adjacency_breaks`: for a pool whose `policy().private_label_benchmark_brand` is `b*` (not `None`), a group `G` holding entries of both `b*` and private-label SKUs is a break iff `min(bay positions of private-label entries in G) != max(bay positions of b* entries in G) + 1` — private label sits immediately right of the brand it is benchmarked against (Q-31). Sorted | `[VS 12]` |

Breaks are reported, not required to be empty (Q-22).

**Examples (PlacementPlan)** — `plan = s2.generate_plan(scope_all(s2), D_0105)`

```python
plan, sk = s2.generate_plan(scope_all(s2), D_0105), s2.skus
# ✓ PLAN-INV-1 … PLAN-INV-8 hold for the fixture plan
assert len(plan.entries) == 10 and plan.home[sk["SKU-B"]].space.id == "RUN-1L1-B"
assert plan.delist_review == (sk["SKU-E"],)                 # explicit candidate; no category has 10 SKUs
assert plan.block_breaks == () and plan.adjacency_breaks == ("DCR-1",)   # same door, not the next one (Q-31)
# ✓ PLAN-Q-1
agr = s2.agreements["AGR-000001"]
p2 = plan.with_agreement(sk["SKU-A"], agr)
assert p2.home[sk["SKU-A"]].agreement is agr and plan.home[sk["SKU-A"]].agreement is None
# ✓ PP-NEW-1
assert PlannedPlacement(sk["SKU-C"], s2.space("RUN-1L1-B"), 4.0, 24.0).agreement is None
# ✗ PP-NEW-PRE-1: an offset on a bin
with expect_violation(PreconditionViolation, "PP-NEW-PRE-1"):
    PlannedPlacement(sk["SKU-P1"], s2.space("BIN-1"), 8.0, 0.0)
# ✗ PLAN-NEW-PRE-1: a SKU both placed and unplaced
with expect_violation(PreconditionViolation, "PLAN-NEW-PRE-1"):
    PlacementPlan(plan.scope, plan.as_of, plan.entries, plan.home,
                  {**plan.unplaced, sk["SKU-A"]: UnplacedReason.INSUFFICIENT_SPACE},
                  plan.under_target, plan.delist_review, plan.block_breaks, plan.adjacency_breaks)
# ✗ PLAN-PRE-1: SKU-S3 has no entry
with expect_violation(PreconditionViolation, "PLAN-PRE-1"):
    plan.with_agreement(sk["SKU-S3"], agr)
# ✗ supplier PLAN-Q-1: the agreement lands on the entry but not on home
bad = Stub(entries=p2.entries, home=plan.home)
assert not P.PLAN_Q_1(plan, bad, None, s=sk["SKU-A"], a=agr)
# ✗ inv PLAN-INV-6: SKU-E left out of delist review (stub over the real plan)
assert not P.PLAN_INV_6(Stub(**{k: getattr(plan, k) for k in ("scope", "as_of", "entries", "unplaced")}, delist_review=()))
```

### 6.4 Store operation `generate_plan` — **the allocation function** `[DISC]`

Allocation takes a set of available merchandising spaces and a set of SKUs and generates a set of SKU placements `[DISC]`. Here the spaces are the pools' assigned leaves, the SKUs are `Store.in_scope_skus(scope, as_of)`, and the output is a `PlacementPlan`. It is a **query**: calling it never changes the model. `[DbC]`

`Store.generate_plan(scope: frozenset[CategoryNode], as_of: date) -> PlacementPlan`

**The canonical procedure** `canonical_plan(store, scope, as_of)` (normative, via GEN-12). Let `S = store.in_scope_skus(scope, as_of)`, `w = DateWindow(as_of, None)`, and `T(s) = store.policy.target_facings(s, S, as_of)`.

1. **Working state.** Start from the store's committed placements, with every `PERMANENT` placement of a SKU in `S` treated as ending at `as_of` (and removed if it starts after `as_of`). Every other placement stays as it is. Leaf queries below (`room`, `free_intervals`) are evaluated in the working state over `w`.
2. **Allocation order.** Process the SKUs of `S` in ascending `(-score(s), s.sku_id)` order: highest score first, ties by id. This order is the same for every pool.
3. **Reasons.** For each `s`, with `p = primary_plane(s)`, report the first that applies (ENUM-5 order):
   1. `OUT_OF_SEASON` if `not s.in_season(as_of)`;
   2. `NO_CATEGORY_SPACE` if no node of `(s.category,) + s.category.ancestors()` has any `assigned_spaces()`;
   3. `NO_SPACE_IN_PLANE` if `pool_node(s, p) is None`;
   4. `NO_DIMENSIONAL_FIT` if `Fit = [l for l in pool_leaves(pool_node(s, p), p) if fits_dimensionally(s.product, l)]` is empty.
4. **Position.** Otherwise let `lo = s.category.policy().min_facings_per_sku`, and `top = 1` for a `LOOSE` SKU, else `T(s)`. For `f` from `top` down to `lo` (for `LOOSE`, just `f = 1`), and for each leaf `l` in `Fit` in order: let `c = consumption(s.product, f, l)`. If `l.room(w) >= c - EPS`, plan `s` on `l` with extent `c`, at the start of the **first** interval of `l.free_intervals(w)` whose length is at least `c` (offset `None` for a leaf that is not `LINEAR_IN`). Add it to the working state and stop searching for `s`.
5. **Scarcity.** If no position was found, report `INSUFFICIENT_SPACE`.
6. **Result.** `entries` are the planned placements in allocation order, each with `agreement = None`. `home[s]` is `s`'s entry. `under_target` is the non-`LOOSE` entries with fewer facings than `T(s)`, in entry order. `delist_review`, `block_breaks`, and `adjacency_breaks` follow PLAN-INV-6, BLK-3, and BLK-4.

Full target facings come before a better level: the best level is chosen among positions that hold the largest facing count that fits anywhere (Q-27). Each SKU is placed before every lower scorer, and free space only shrinks afterwards, so the canonical result satisfies GEN-1 … GEN-8, GEN-11, and GEN-13 by construction. Those stay as separate clauses because each is independently checkable and traces to a textbook rule. A case where `canonical_plan` breaks one of them is a defect in this contract, not a choice for the implementation.

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| GEN-PRE-1 | require | `len(scope) >= 1` and `all(self.has_category(n.id) and self.category(n.id) is n for n in scope)` | `[DbC]` |
| GEN-PRE-2 | require | the store is at a stable time: `check_invariants(self)` would pass (stated for emphasis) | `[DbC]` |

**ensure** — with `S`, `T`, and `E = result.entries` as above

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| GEN-1 | ensure | **Completeness.** `{e.sku for e in E} | set(result.unplaced) == set(S)` — every in-scope SKU is placed once or reported with a reason | `[TB 2.2]` |
| GEN-2 | ensure | **Seasonality.** For every `s in S`: `(not s.in_season(as_of)) == (result.unplaced.get(s) == OUT_OF_SEASON)` | `[TB 2.1]` |
| GEN-3 | ensure | **Right space and plane.** Every `e in E` has `e.space.presentation_plane == primary_plane(e.sku)` and `e.space.assigned_category is pool_node(e.sku, primary_plane(e.sku))`, and the plan is feasible (PLAN-INV-4) | `[DISC]` `[TB 2.1]` |
| GEN-4 | ensure | **Honest, exclusive reasons.** For every `s in S`, `result.unplaced.get(s)` is the first applicable reason of procedure step 3 (precedence ENUM-5) if one applies. If none applies, `s` is placed, or `result.unplaced[s] == INSUFFICIENT_SPACE` and in the planned state no leaf in `Fit` has `room(w) >= consumption(s.product, lo, l)` | `[SIM]` |
| GEN-5 | ensure | **Facing bounds.** For every non-`LOOSE` `e`: `lo <= e.facings <= T(e.sku)`, with `e.facings` derived per PLC-Q-1; the plan never gives a SKU more than its target. For a `LOOSE` SKU, `e.extent == e.space.leaf_capacity` | `[TB 2.3]` |
| GEN-6 | ensure | **No idle space while under target.** For every non-`LOOSE` `e` with `e.facings < T(e.sku)`: in the planned state, no free interval adjacent to `e`'s interval on `e.space` (`LINEAR_IN`), and no free amount on `e.space` (other units), is at least one facing's extent. And `under_target == tuple(e.sku for e in E if e.sku.product.form != LOOSE and e.facings < T(e.sku))` | `[TB 2.3]` |
| GEN-7 | ensure | **Facing monotonicity.** For `a, b in E` in the same pool with `score(a) > score(b)` and `a.sku.product.facing_extent(u) <= b.sku.product.facing_extent(u)`: `a.facings >= b.facings`, unless `a` is under target (GEN-6) | `[TB 2.3]` |
| GEN-8 | ensure | **Best-level priority.** For `a, b in E` in the same pool, both on shelf levels, with `score(a) > score(b)` and `level_preference(a.space.shelf_level) > level_preference(b.space.shelf_level)`: placing `a` at `(b.space, b.offset_in)` with `a.extent` and `b` at `(a.space, a.offset_in)` with `b.extent` would violate PLAN-INV-4. The better seller gets the better level whenever a feasible swap exists | `[TB 2.3]` |
| GEN-9 | ensure | **Determinism.** Equal model states and equal arguments give equal plans | `[SIM]` |
| GEN-10 | ensure | **Pure.** `changed(old, self.snapshot()) == frozenset()` | `[DbC]` |
| GEN-11 | ensure | **Home designation.** `result.home[s]` is `s`'s single entry | `[VS 2]` |
| GEN-12 | ensure | **Canonical result.** `result == canonical_plan(self, scope, as_of)`. An implementation may compute it any way it likes, provided the result is identical (D-2) | `[SIM]` |
| GEN-13 | ensure | **Scarcity priority.** If `result.unplaced[s] == INSUFFICIENT_SPACE`, then for every `t in E` in the same pool with `score(t.sku) < score(s)`: `s` could not take `t`'s place, i.e. `not (fits_dimensionally(s.product, t.space) and consumption(s.product, lo_s, t.space) <= t.space.room(w, without=t))` in the planned state, with `lo_s` the minimum facings of `s`'s category. A higher scorer is never left out so that a lower scorer can stay | `[TB 2.3]` `[SIM]` |

Blocking (BLK-3, BLK-4) and the private-label target are objectives that are reported, not asserted.

**Examples (generate_plan)** — from S2; the store is unchanged (S2 → S2)

```python
plan, sk = s2.generate_plan(scope_all(s2), D_0105), s2.skus
# ✓ GEN-12: the canonical entries, in allocation order (score 40, 40, 30, 20, 20, 12, 10, 10, 6, 5)
assert [(e.sku.sku_id, e.space.id, e.extent, e.offset_in) for e in plan.entries] == [
    ("SKU-P1", "BIN-1", 8.0, None), ("SKU-Y1", "RUN-D1-E", 30.0, 0.0),
    ("SKU-A", "RUN-1L1-E", 48.0, 0.0), ("SKU-B", "RUN-1L1-B", 24.0, 0.0),
    ("SKU-P2", "BIN-2", 8.0, None), ("SKU-S1", "PEG-1", 3.0, None),
    ("SKU-C", "RUN-1L1-B", 4.0, 24.0), ("SKU-Y2", "RUN-D1-M", 5.0, 0.0),
    ("SKU-S2", "PEG-1", 1.0, None), ("SKU-D", "RUN-1L1-B", 5.0, 28.0)]
# ✓ GEN-1, GEN-2, GEN-4: every reason appears once
assert {s.sku_id: r.name for s, r in plan.unplaced.items()} == {
    "SKU-F": "OUT_OF_SEASON", "SKU-H": "NO_CATEGORY_SPACE", "SKU-G": "NO_SPACE_IN_PLANE",
    "SKU-E": "NO_DIMENSIONAL_FIT", "SKU-S3": "INSUFFICIENT_SPACE"}
# ✓ GEN-5, GEN-6: SKU-A's target of 7 facings (56 in) fits no 48 in shelf, so it gets 6
assert plan.under_target == (sk["SKU-A"],)
# ✓ GEN-10
before = s2.snapshot(); s2.generate_plan(scope_all(s2), D_0105); assert s2.snapshot() == before
# ✓ GEN-9
assert s2.generate_plan(scope_all(s2), D_0105) == plan
# ✗ GEN-PRE-1
with expect_violation(PreconditionViolation, "GEN-PRE-1"):
    s2.generate_plan(frozenset(), D_0105)
# GEN-PRE-2 has no client counter-example: no sequence of correct calls leaves the store unstable.
# ✗ supplier GEN-5: SKU-B given 9 facings, one more than its target of 8
e9 = tuple(PlannedPlacement(e.sku, e.space, 27.0, 0.0) if e.sku is sk["SKU-B"] else e for e in plan.entries)
assert not P.GEN_5(s2, Stub(entries=e9, unplaced=plan.unplaced), s2.snapshot(), scope=scope_all(s2), as_of=D_0105)
# ✗ supplier GEN-13: SKU-S2 left out while the lower-scoring SKU-S3 takes its hook
swapped = tuple(PlannedPlacement(sk["SKU-S3"], e.space, 1.0, None) if e.sku is sk["SKU-S2"] else e
                for e in plan.entries)
unpl = {**{k: v for k, v in plan.unplaced.items() if k is not sk["SKU-S3"]},
        sk["SKU-S2"]: UnplacedReason.INSUFFICIENT_SPACE}
assert not P.GEN_13(s2, Stub(entries=swapped, unplaced=unpl), s2.snapshot(), scope=scope_all(s2), as_of=D_0105)
# ✗ supplier GEN-4: a hanging SKU reported as not fitting rather than as having no peg space
bad = {**plan.unplaced, sk["SKU-G"]: UnplacedReason.NO_DIMENSIONAL_FIT}
assert not P.GEN_4(s2, Stub(entries=plan.entries, unplaced=bad), s2.snapshot(), scope=scope_all(s2), as_of=D_0105)
# ✓ GEN-8 holds for the canonical plan
assert P.GEN_8(s2, plan, s2.snapshot(), scope=scope_all(s2), as_of=D_0105)
# ✗ supplier GEN-8: SKU-Y2 (score 10) at eye level and SKU-Y1 (score 40) on the middle shelf; the swap is feasible
flip = {"SKU-Y1": ("RUN-D1-M", 30.0), "SKU-Y2": ("RUN-D1-E", 5.0)}
e8 = tuple(PlannedPlacement(e.sku, s2.space(flip[e.sku.sku_id][0]), flip[e.sku.sku_id][1], 0.0)
           if e.sku.sku_id in flip else e for e in plan.entries)
assert not P.GEN_8(s2, Stub(entries=e8, unplaced=plan.unplaced), s2.snapshot(), scope=scope_all(s2), as_of=D_0105)
```

### 6.5 The planogram: versions, freezing, and drift `[VS 10]`

The planogram is versioned. A revision creates a new authoritative version mid-cycle; the shelf never quietly departs from the old one. The version the crew executes and the version the category manager is editing are **deliberately different objects**. Revisions made in the meantime queue up for the next release. `[VS 10]`

```
class PlanogramVersion                           -- immutable snapshot except status
creation  PlanogramVersion(version_no, kind, scope, cutoff_date, effective_date, entries, home,
                           status=RELEASED)      -- used by release_working_plan and Store.build
queries
    version_no: int
    kind: VersionKind                            -- CYCLE (on a reset date) or REVISION (mid-cycle)
    scope: frozenset[CategoryNode]
    cutoff_date: date                            -- the reset pack is the plan as of this date
    effective_date: date
    entries: tuple[PlannedPlacement, ...]
    home: Mapping[SKU, PlannedPlacement]
    status: VersionStatus                        -- RELEASED → EXECUTED, once

class Planogram
creation  Planogram(working=None, versions=())   -- used by Store.build
queries
    working: PlacementPlan | None                -- what the category manager is editing
    versions: tuple[PlanogramVersion, ...]       -- ordered by version_no
    current_version(category: CategoryNode, on: date) -> PlanogramVersion | None
```

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| PGV-NEW-PRE-1 | require | `version_no >= 1` and PGV-INV-2 (first part) and PGV-INV-5 hold for the arguments | `[VS 10]` |
| PGV-NEW-1 | ensure | attributes as given | `[DbC]` |
| PGV-INV-1 | invariant | after creation, `scope`, `entries`, `home`, `kind`, `cutoff_date`, `effective_date` never change; `status` changes once, `RELEASED → EXECUTED`, through `execute_version` | `[VS 10]` |
| PGV-INV-2 | invariant | `cutoff_date <= effective_date`, and (A) `kind == CYCLE ⇒ effective_date in store.reset_calendar` | `[VS 4, 10]` |
| PGV-INV-3 | invariant | (on `Planogram`) `[v.version_no for v in versions] == list(range(1, len(versions) + 1))` | `[VS 10]` |
| PGV-INV-4 | invariant | (on `Planogram`) **one pending version per scope.** For versions `v != w` whose scopes overlap (a node of one is within a node of the other): not both `RELEASED`; and `v.version_no < w.version_no ⇒ v.effective_date < w.effective_date` | `[VS 10]` |
| PGV-INV-5 | invariant | `all(any(e.sku.category.is_within(n) for n in scope) for e in entries)` and `set(home) == {e.sku for e in entries}` | `[VS 2, 10]` |
| PGM-NEW-PRE-1 | require | PGV-INV-3 and PGV-INV-4 hold for `versions` | `[VS 10]` |
| PGM-NEW-1 | ensure | attributes as given | `[DbC]` |
| PGV-Q-1 | ensure | `current_version(c, d)`: the version with the greatest `version_no` among those with `status == EXECUTED`, `effective_date <= d`, and some `n in scope` with `c.is_within(n)`; `None` if there is none. **Compliance is always measured against the current version** | `[VS 10]` |

**Home position `[VS 2]`.** "Primary" is not a flag on a placement. It falls out of which placement the planogram treats as the home position for replenishment.

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| HOME-1 | ensure | `Store.home_placement(sku, on)`: let `v = planogram.current_version(sku.category, on)` and `h = v.home.get(sku)` if `v` exists. `result` is the `PERMANENT` placement `p` of `sku` live on `on` with `(p.space, p.offset_in) == (h.space, h.offset_in)` if `h` exists and such a `p` exists, else `None` | `[VS 2]` |
| HOME-2 | invariant (A) | (on `Store`) for every SKU `s` and every date `d` in `change_dates()`: if `s` has a `PERMANENT` placement live on `d` created by executing the current version, `home_placement(s, d)` is not `None` | `[VS 2]` |

Observed shelf state is a third, separate artefact and is reserved (Part 10). No `Placement`, `PlanogramVersion`, or `Planogram` feature records observed state. `[VS 10]` `[SIM]`

**Examples (planogram)**

```python
rte = s5.category("CAT-RTE")
# ✓ PGV-NEW-1, PGV-INV-3
assert [(v.version_no, v.kind, v.status) for v in s5.planogram.versions] == [
    (1, VersionKind.CYCLE, VersionStatus.EXECUTED), (2, VersionKind.CYCLE, VersionStatus.EXECUTED)]
assert s3.planogram.versions[0].status is VersionStatus.RELEASED and s3.planogram.versions[0].cutoff_date == D_0102
# ✓ PGV-Q-1
assert s5.planogram.current_version(rte, date(2026, 3, 1)).version_no == 1
assert s5.planogram.current_version(rte, D_0915).version_no == 2
assert s3.planogram.current_version(rte, D_0105) is None              # released, not executed
# ✓ HOME-1: B's home moved from PLC-000004 to PLC-000011 at the July reset
assert s5.home_placement(s5.skus["SKU-B"], date(2026, 3, 1)).placement_id == "PLC-000004"
assert s5.home_placement(s5.skus["SKU-B"], D_0915).placement_id == "PLC-000011"
assert s6.home_placement(s6.skus["SKU-A"], D_0915).placement_id == "PLC-000003"   # not the promo display
# ✓ PGM-NEW-1
assert Planogram().working is None and Planogram().versions == ()
# ✗ PGV-NEW-PRE-1: cutoff after the effective date
v1 = s5.planogram.versions[0]
with expect_violation(PreconditionViolation, "PGV-NEW-PRE-1"):
    PlanogramVersion(9, VersionKind.REVISION, v1.scope, D_0706, D_0105, v1.entries, v1.home)
# ✗ PGM-NEW-PRE-1: version numbers must start at 1
with expect_violation(PreconditionViolation, "PGM-NEW-PRE-1"):
    Planogram(versions=(s5.planogram.versions[1],))
# ✗ supplier PGV-Q-1: returning a version that has not been executed
assert not P.PGV_Q_1(s3.planogram, s3.planogram.versions[0], None, c=s3.category("CAT-RTE"), d=D_0105)
# ✗ inv PGV-INV-4: two pending versions over Cereal (stub)
pend = lambda n, d: Stub(version_no=n, status=VersionStatus.RELEASED, effective_date=d,
                         scope=frozenset({s5.category("CAT-CER")}))
assert not P.PGV_INV_4(Stub(versions=(pend(1, D_0105), pend(2, D_0706))))
```

---
## Part 7 — `Store`: the root, the facade, filters, and commands

### 7.1 Features and invariants

```
class Store                    (conforms to SpaceNode; also aggregates the other hierarchies)
creation  Store.build(...)     -- Part 8
queries
    id: str                                    -- "STORE-" prefix
    footprint_sqft: float                      -- the building
    business_unit: BusinessUnit                -- the unit with kind == "STORE"
    reset_calendar: tuple[date, ...]           -- scheduled chain-wide reset windows, ascending
    products: Mapping[str, Product]            -- by upc
    skus: Mapping[str, SKU]
    category_roots: tuple[CategoryNode, ...]
    business_unit_roots: tuple[BusinessUnit, ...]
    placements: Mapping[str, Placement]        -- every committed placement ever made, incl. ended
    incidentals: Mapping[str, IncidentalPlacement]
    agreements: Mapping[str, Agreement]
    policy: AllocationPolicy
    planogram: Planogram
```

`Store.space_type == SPACE` (the root of the type tree) and `Store.parent is None`.

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| ST-INV-1 | invariant | `parent is None and space_type == SPACE`, and every child is an `Area` | `[VS 6]` |
| ST-INV-2 | invariant | AREA-INV-2 holds for the store: the building footprint reconciles to its children | `[VS 6]` |
| ST-INV-3 | invariant | every id in the store (space nodes, category nodes, business units, SKUs, agreements, placements, incidental placements) is unique across all classes and carries its class's prefix (ID-1, ID-2); `space(n.id) is n` for every space node `n`, and likewise for categories, business units, and SKUs | `[SIM]` |
| ST-INV-4 | invariant | **Referential integrity.** every placement's `space` is in `allocatable_leaves()` and its `sku is skus[sku.sku_id]`; every incidental's `sku` is in `skus` and its `zone_hint` is `None` or a node of this store; every referenced agreement is in `agreements`; every SKU's `product` is in `products` and its `category` is in the category hierarchy | `[DbC]` |
| ST-INV-5 | invariant | **The views agree.** `set(placements.values()) == {p for l in allocatable_leaves() for p in l.placements} == {p for s in skus.values() for p in s.placements if isinstance(p, Placement)}`, and `set(incidentals.values()) == {p for s in skus.values() for p in s.placements if isinstance(p, IncidentalPlacement)}` | `[DbC]` |
| ST-INV-6 | invariant | `reset_calendar` is strictly ascending | `[VS 4]` |
| ST-INV-7 | invariant | `l in l.assigned_category.assigned_spaces()` for every leaf with an assigned category | `[SIM]` |
| ST-INV-8 | invariant | every invariant of every contained object holds, including the (A) ones. The model is consistent at every stable time | `[DbC]` |
| ST-INV-9 | invariant | for each prefix `x` of `PLC`, `INC`, `AGR`: `next_id_number(x)` is greater than the number of every record of that prefix ever created in this store (ID-2) | `[SIM]` |

### 7.2 Queries and filters

**Lookups and reports**

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| ST-PRE-1 | require | `space(id)`, `category(id)`, `business_unit_node(id)`: `has_space(id)`, `has_category(id)`, `has_business_unit(id)` respectively | `[DbC]` |
| ST-Q-1 | ensure | `space(id)`, `category(id)`, `business_unit_node(id)`: `result.id == id` | `[DbC]` |
| ST-Q-11 | ensure | `has_space(id)`, `has_category(id)`, `has_business_unit(id)`, `has_sku(id)`, `has_agreement(id)`, `has_placement(id)`, `has_version(n)`: `result` is whether a node, SKU, agreement, committed or incidental placement, or version with that id or number exists in this store | `[DbC]` |
| ST-PRE-2 | require | `in_scope_skus(scope, as_of)`: every element of `scope` is a category node of this store | `[DbC]` |
| ST-Q-2 | ensure | `in_scope_skus(scope, as_of)`: `result == tuple(s for s in skus.values() if any(s.category.is_within(n) for n in scope) and s.assortment_status != DELISTED)`, ordered by `(s.category.path, -policy.score(s), s.sku_id)`. Out-of-season SKUs are included; the plan reports them as `OUT_OF_SEASON` | `[TB 2.6]` `[SIM]` |
| ST-Q-3 | ensure | `unassigned_leaves()`: `result == tuple(l for l in allocatable_leaves() if l.assigned_category is None)` | `[DISC]` |
| ST-Q-4 | ensure | `change_dates()`: `result == frozenset(p.start_date for p in placements.values()) | frozenset(p.end_date for p in placements.values() if p.end_date is not None)` | `[VS 4]` |
| ST-Q-5 | ensure | `spaces_of_type(t)`: `result == frozenset(n for n in subtree() if n.space_type.is_within(t))` — the type hierarchy crossed with the physical one | `[VS 6]` |
| ST-Q-6 | ensure | `selling_share()`: `result == sum(n.footprint_sqft for n in subtree() if n.merchandisable and not n.parent.merchandisable) / footprint_sqft`. A reporting figure only; selling space is roughly 60–70 % of a real floor | `[VS 6]` `[TB 1.2]` |
| ST-Q-7 | ensure | `all_placements()`: `result == frozenset(placements.values()) | frozenset(incidentals.values())` | `[VS 4]` |
| ST-Q-8 | ensure | `home_placement(sku, on)`: HOME-1 | `[VS 2]` |
| ST-PRE-3 | require | `next_id_number(x)`: `x in {"PLC", "INC", "AGR"}` | `[SIM]` |
| ST-Q-10 | ensure | `next_id_number(x)`: `result >= 1`, and the next record created with prefix `x` gets the id `f"{x}-{result:06d}"` (ID-2) | `[SIM]` |

`snapshot()` is FRM-1.

**Examples (Store queries)** — at S6 unless stated

```python
# ✓ ST-Q-1, ST-Q-11
assert s6.space("PEG-1").id == "PEG-1" and s6.has_placement("INC-000001") and not s6.has_version(3)
# ✓ ST-Q-2: Hot Cereal's path sorts before Ready-to-Eat's; within each, by descending score
assert [s.sku_id for s in s6.in_scope_skus(frozenset({s6.category("CAT-CER")}), D_0105)] == [
    "SKU-D", "SKU-G", "SKU-A", "SKU-B", "SKU-C", "SKU-F", "SKU-E"]
# ✓ ST-Q-3: 28 allocatable leaves, 9 assigned
assert len(s6.allocatable_leaves()) == 28 and len(s6.unassigned_leaves()) == 19
# ✓ ST-Q-4
assert s6.change_dates() == {D_0105, D_0706, D_0901, D_0929}
# ✓ ST-Q-5
assert (len(s6.spaces_of_type(SpaceType.GONDOLA)), len(s6.spaces_of_type(SpaceType.OPEN_SHELVING)),
        len(s6.spaces_of_type(SpaceType.MERCHANDISING))) == (18, 24, 40)
# ✓ ST-Q-6: fixtures total 100 of 1200 sq ft
assert abs(s6.selling_share() - 100 / 1200) < EPS
# ✓ ST-Q-7, ST-Q-10
assert len(s6.all_placements()) == 13
assert (s6.next_id_number("PLC"), s6.next_id_number("INC"), s6.next_id_number("AGR")) == (13, 2, 3)
# ✓ FRM-1, FRM-2
assert changed(s6.snapshot(), s6.snapshot()) == frozenset()
assert ("assignment", "TBL-1") in changed(s0.snapshot(), s1.snapshot())
# ✗ ST-PRE-1
with expect_violation(PreconditionViolation, "ST-PRE-1"):
    s6.space("RUN-9-E")
# ✗ ST-PRE-2: a node of another store's hierarchy
with expect_violation(PreconditionViolation, "ST-PRE-2"):
    s6.in_scope_skus(frozenset({s5.category("CAT-CER")}), D_0105)
# ✗ ST-PRE-3
with expect_violation(PreconditionViolation, "ST-PRE-3"):
    s6.next_id_number("SKU")
# ✗ supplier ST-Q-4: end dates left out
assert not P.ST_Q_4(s6, frozenset({D_0105, D_0706, D_0901}), None)
# ✗ inv ST-INV-5: a leaf listing a placement the store does not (stub)
assert not P.ST_INV_5(Stub(placements={}, incidentals={}, skus={},
                           allocatable_leaves=lambda: (Stub(placements=(s6.placements["PLC-000003"],)),)))
# ✗ inv ST-INV-6 (stub)
assert not P.ST_INV_6(Stub(reset_calendar=(D_0706, D_0105)))
```

**Placement filters** (rule 9: all return `frozenset[PlacementRecord]`, are pure, and are subsets of `all_placements()`). "And" is the intersection of two results, "or" the union. `[VS 13]`

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| FLT-PRE-1 | require | `placements_in_space(n)`, `placements_in_category(c)`, `placements_in_unit(u)`: the argument is a node of this store | `[DbC]` |
| FLT-1 | ensure | `placements_live_on(d)`: `result == frozenset(p for p in all_placements() if p.window.contains(d))` | `[VS 4, 13]` |
| FLT-2 | ensure | `placements_of_type(t)`: `result == frozenset(p for p in all_placements() if p.placement_type == t)` | `[VS 3, 13]` |
| FLT-3 | ensure | `placements_in_space(n)`: `result == frozenset(p for p in placements.values() if p.space.is_within(n)) | frozenset(p for p in incidentals.values() if p.zone_hint is not None and p.zone_hint.is_within(n))` | `[VS 6, 13]` |
| FLT-4 | ensure | `placements_in_category(c)`: `result == frozenset(p for p in all_placements() if p.sku.category.is_within(c))` | `[VS 6, 13]` |
| FLT-5 | ensure | `placements_in_unit(u)`: `result == frozenset(p for p in all_placements() if p.business_unit.is_within(u))` | `[VS 6, 13]` |
| FLT-6 | ensure | `placements_with_agreement_type(t)`: `result == frozenset(p for p in all_placements() if p.agreement is not None and p.agreement.agreement_type == t)` | `[VS 3, 13]` |
| FLT-7 | invariant | **Algebra.** For filter results `F`, `G`: `F & G`, `F | G`, `F - G` are plain set operations. The three `placements_of_type` results are pairwise disjoint and their union is `all_placements()`; `placements_in_category` over the category roots partitions `all_placements()`; `placements_in_unit` over the `DEPARTMENT` units partitions `all_placements()` | `[VS 13]` |

A string query language, if it ever earns its place, is a thin layer over these filters, not a replacement. `[VS 13]`

**Examples (filters)** — at S6

```python
ids = lambda F: {p.placement_id for p in F}
# ✓ FLT-1
assert len(s6.placements_live_on(D_0915)) == 12 and "PLC-000012" not in ids(s6.placements_live_on(D_1015))
# ✓ FLT-2, FLT-6
assert ids(s6.placements_of_type(PlacementType.PROMOTIONAL)) == {"PLC-000012"}
assert ids(s6.placements_with_agreement_type(AgreementType.SLOTTING)) == {"PLC-000003", "PLC-000004"}
# ✓ FLT-3: the peg section is inside SID-1L, and INC-000001 comes in by its zone hint
assert ids(s6.placements_in_space(s6.space("SID-1L"))) == {
    "PLC-000003", "PLC-000004", "PLC-000006", "PLC-000007", "PLC-000009", "PLC-000010", "PLC-000011", "INC-000001"}
# ✓ FLT-4, FLT-5
assert len(s6.placements_in_category(s6.category("CAT-CER"))) == 7
assert ids(s6.placements_in_unit(s6.business_unit_node("BU-DY"))) == {"PLC-000002", "PLC-000008"}
# ✓ FLT-7: "and" is intersection
cer = s6.placements_in_category(s6.category("CAT-CER"))
assert ids(cer & s6.placements_live_on(D_0915) & s6.placements_of_type(PlacementType.PERMANENT)) == {
    "PLC-000003", "PLC-000007", "PLC-000010", "PLC-000011"}
# ✗ FLT-PRE-1
with expect_violation(PreconditionViolation, "FLT-PRE-1"):
    s6.placements_in_space(s5.space("SID-1L"))
# ✗ supplier FLT-3: dropping the incidental placement found by zone hint
assert not P.FLT_3(s6, s6.placements_in_space(s6.space("SID-1L")) - {s6.incidentals["INC-000001"]}, None,
                   n=s6.space("SID-1L"))
```

### 7.3 Commands

All commands are **atomic** (MON-10). A command changes state and returns nothing, except a creation command, which returns the new record's id. Each command's frame condition (FRM-3) lists every key it may change.

**`assign_space(node_id, category_id)`** — assigns a space node, and so every leaf under it, to a category node (Q-2)

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| ASN-PRE-1 | require | `has_space(node_id)`, `has_category(category_id)`, and `space(node_id).merchandisable` | `[TB 1.2, 2.1]` |
| ASN-PRE-2 | require | `space(node_id).assigned_category is None` and no descendant of `space(node_id)` has an `own_assignment` (SN-INV-2) | `[SIM]` |
| ASN-PRE-3 | require | every allocatable leaf `l` under the node has `any(l.space_type.is_within(t) for t in category(category_id).policy().allowed_space_types)` | `[TB 2.1]` `[SIM]` |
| ASN-PRE-4 | require | every such leaf has `temperature_zone == category(category_id).policy().temperature_zone` (LS-INV-7). Implied by ASN-PRE-3 under CAT-INV-9; stated for emphasis | `[TB 4.4]` |
| ASN-PRE-5 | require | `not category(category_id).policy().secured_case_required or all(l.secured for l in those leaves)` (LS-INV-8) | `[TB 6.4]` |
| ASN-1 | ensure | `space(node_id).own_assignment is category(category_id)` | `[SIM]` |
| ASN-2 | ensure | `changed(old, self.snapshot()) <= {("assignment", node_id)}` | `[DbC]` |

```python
# ✓ S0 → (part of S1); ASN-1, ASN-2
old = s0.snapshot(); s0.assign_space("TBL-1", "CAT-PR")
assert s0.space("BIN-2").assigned_category.id == "CAT-PR" and changed(old, s0.snapshot()) == {("assignment", "TBL-1")}
# ✗ ASN-PRE-1: the backroom is not merchandising space
with expect_violation(PreconditionViolation, "ASN-PRE-1"):
    s1.assign_space("AREA-BR", "CAT-DG")
# ✗ ASN-PRE-2: a run under BAY-1L1 is already assigned
with expect_violation(PreconditionViolation, "ASN-PRE-2"):
    s1.assign_space("BAY-1L1", "CAT-CER")
# ✗ ASN-PRE-3: floor displays are never category space
with expect_violation(PreconditionViolation, "ASN-PRE-3"):
    s1.assign_space("FD-1", "CAT-CER")
# ASN-PRE-4 has no independent counter-example: ASN-PRE-3 fails first whenever it fails.
# ✗ ASN-PRE-5: Health & Beauty requires locked space; RUN-1L2-E is not secured
with expect_violation(PreconditionViolation, "ASN-PRE-5"):
    s1.assign_space("RUN-1L2-E", "CAT-HBA")
# ✗ supplier ASN-2: measured from S0, the state differs in five more assignments than the one requested
s1.assign_space("RUN-1L2-E", "CAT-CER")
assert not P.ASN_2(s1, None, s0.snapshot(), node_id="RUN-1L2-E", category_id="CAT-CER")
```

**`unassign_space(node_id, as_of)`**

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| UNA-PRE-1 | require | `has_space(node_id)` and `space(node_id).own_assignment is not None` | `[SIM]` |
| UNA-PRE-2 | require | every `PERMANENT` placement in the leaves under the node has `end_date is not None and end_date <= as_of` — clear permanent stock first, by a reset | `[VS 4]` `[SIM]` |
| UNA-1 | ensure | `space(node_id).own_assignment is None` | `[SIM]` |
| UNA-2 | ensure | `changed(old, self.snapshot()) <= {("assignment", node_id)}` | `[DbC]` |

```python
# ✓ S1 → S1 without the produce assignment
s1.unassign_space("TBL-1", D_0105); assert s1.space("BIN-1").assigned_category is None
# ✗ UNA-PRE-1: BIN-1 inherits its assignment; it has none of its own
with expect_violation(PreconditionViolation, "UNA-PRE-1"):
    s1.unassign_space("BIN-1", D_0105)
# ✗ UNA-PRE-2: apples and pears are still on the table
with expect_violation(PreconditionViolation, "UNA-PRE-2"):
    s4.unassign_space("TBL-1", D_0915)
```

**`set_working_plan(plan)`**

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| WRK-PRE-1 | require | `plan` equals `generate_plan(plan.scope, plan.as_of)` except for agreements added by `with_agreement` | `[VS 10]` `[SIM]` |
| WRK-1 | ensure | `planogram.working == plan` | `[VS 10]` |
| WRK-2 | ensure | `changed(old, self.snapshot()) <= {("working", "")}` | `[DbC]` |

```python
# ✓ (part of S2 → S3)
plan = s2.generate_plan(scope_all(s2), D_0105); s2.set_working_plan(plan); assert s2.planogram.working == plan
# ✗ WRK-PRE-1: a hand-edited plan whose entries are reordered
bad = PlacementPlan(plan.scope, plan.as_of, plan.entries[::-1], plan.home, plan.unplaced,
                    plan.under_target, plan.delist_review, plan.block_breaks, plan.adjacency_breaks)
with expect_violation(PreconditionViolation, "WRK-PRE-1"):
    s2.set_working_plan(bad)
```

**`release_working_plan(kind, cutoff_date, effective_date) -> int`** — freezes the working plan as a new version, the "reset pack" `[VS 10]`

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| REL-PRE-1 | require | `planogram.working is not None and planogram.working.as_of == effective_date and cutoff_date <= effective_date` | `[VS 10]` |
| REL-PRE-2 | require | `kind == CYCLE ⇒ effective_date in reset_calendar` (PGV-INV-2) | `[VS 4]` |
| REL-PRE-3 | require | no version whose scope overlaps the working plan's is still `RELEASED`, and every such version has `effective_date < effective_date` (PGV-INV-4). Revisions made meanwhile stay in `working` and queue for the next release | `[VS 10]` |
| REL-PRE-4 | require | the working plan is still current: it equals `generate_plan(working.scope, working.as_of)` except for agreements | `[VS 10]` `[SIM]` |
| REL-1 | ensure | `result == len(old.versions) + 1`; the new version's `scope`, `entries`, `home` equal the working plan's; `kind`, `cutoff_date`, `effective_date` as given; `status == RELEASED` | `[VS 10]` |
| REL-2 | ensure | `planogram.working == old.working` and `changed(old, self.snapshot()) <= {("version", str(result))}` | `[DbC]` |

`old.versions` and `old.working` read the snapshot's version and working entries.

```python
# ✓ S2 → S3 (REL-1, REL-2)
assert s3.planogram.versions[-1].version_no == 1 and s3.planogram.working is not None
# ✗ REL-PRE-1: nothing to release
with expect_violation(PreconditionViolation, "REL-PRE-1"):
    s2.release_working_plan(VersionKind.CYCLE, D_0102, D_0105)
# ✗ REL-PRE-2: a cycle version effective off the reset calendar
s2.set_working_plan(s2.generate_plan(scope_all(s2), date(2026, 3, 2)))
with expect_violation(PreconditionViolation, "REL-PRE-2"):
    s2.release_working_plan(VersionKind.CYCLE, date(2026, 3, 1), date(2026, 3, 2))
# ✗ REL-PRE-3: version 1 is still pending
s3.set_working_plan(s3.generate_plan(scope_all(s3), D_0706))
with expect_violation(PreconditionViolation, "REL-PRE-3"):
    s3.release_working_plan(VersionKind.CYCLE, D_0701, D_0706)
# ✗ REL-PRE-4: more space was assigned after the plan was set, so it is stale
s5.assign_space("RUN-1L2-E", "CAT-CER")
s5.set_working_plan(s5.generate_plan(scope_all(s5), D_270104))
s5.assign_space("RUN-1L2-B", "CAT-CER")
with expect_violation(PreconditionViolation, "REL-PRE-4"):
    s5.release_working_plan(VersionKind.CYCLE, date(2026, 12, 28), D_270104)
```

**`execute_version(version_no, on)`** — executes a reset. A full-store reset is a version whose scope is every root; a partial reset `[TB 1.6]` covers fewer nodes. `[VS 4, 10]`

Let `V = planogram.versions[version_no - 1]` and `Sc = {s for s in skus.values() if any(s.category.is_within(n) for n in V.scope)}`.

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| EXE-PRE-1 | require | `has_version(version_no)`, `V.status == RELEASED`, and `V.effective_date == on` | `[VS 10]` |
| EXE-PRE-2 | require | `V`'s entries are feasible in the current state (PLAN-INV-4 against the placements now in the model) | `[VS 14]` |
| EXE-PRE-3 | require | every non-`None` `e.agreement` in `V.entries` is in `agreements` | `[DbC]` |
| EXE-1 | ensure | **Replacement.** For every `s in Sc`, the `PERMANENT` placements of `s` live on `on` afterwards are exactly one per entry of `V` for `s`. An old live placement equal to an entry on `(sku, space, extent, offset_in, agreement)` survives unchanged (same `placement_id`, `start_date`, `end_date is None`). Every other old live `PERMANENT` placement of an `s in Sc` gets `end_date == on`. Each entry without a surviving match becomes a new `Placement` with `start_date == on`, `end_date is None`, `placement_type == PERMANENT` | `[VS 4, 10]` |
| EXE-2 | ensure | `changed(old, self.snapshot()) <= {("placement", i) for i in ids of PERMANENT placements of Sc, old or new} | {("version", str(version_no)), ("counter", "PLC")} | {("last_reset", n.id) for n in V.scope}` — promotional, incidental, out-of-scope, and ended placements are unchanged | `[DbC]` |
| EXE-3 | ensure | `V.kind == CYCLE ⇒ all(n.last_reset_date == on for n in V.scope)`; a `REVISION` leaves every `last_reset_date` unchanged | `[TB 2.6]` `[VS 4]` |
| EXE-4 | ensure | `V.status == EXECUTED`, and for every `s` with an entry in `V`, `home_placement(s, on)` is not `None` (HOME-2) | `[VS 2, 10]` |
| EXE-5 | ensure | **History preserved.** every placement that existed on entry exists afterwards with identical attributes, except that `end_date` may have changed from `None` to `on` (PLC-INV-IMM) | `[VS 4]` |
| EXE-6 | ensure | the new placements are created in `V.entries` order, with consecutive ids starting at `old` `next_id_number("PLC")` (ID-2) | `[SIM]` |

Executing a version is instantaneous on `on` in this version. A real reset runs over a night or several, sometimes staggered across stores for weeks `[VS 10]`; that is reserved (Part 10, Q-19).

```python
# ✓ S3 → S4: EXE-1, EXE-6: ten new placements in entry order
assert [s4.placements[f"PLC-{i:06d}"].sku.sku_id for i in range(1, 11)] == [
    "SKU-P1", "SKU-Y1", "SKU-A", "SKU-B", "SKU-P2", "SKU-S1", "SKU-C", "SKU-Y2", "SKU-S2", "SKU-D"]
# ✓ S4 → S5: EXE-1: SKU-A's entry matches PLC-000003 and survives; SKU-B's slotting was not renewed
assert s5.placements["PLC-000003"].end_date is None and s5.placements["PLC-000004"].end_date == D_0706
assert (s5.placements["PLC-000011"].sku.sku_id, s5.placements["PLC-000011"].start_date) == ("SKU-B", D_0706)
# ✓ EXE-3
assert s5.category("CAT-DG").last_reset_date == D_0706 and s5.category("CAT-RTE").last_reset_date is None
# ✗ EXE-PRE-1: executing on the wrong date
with expect_violation(PreconditionViolation, "EXE-PRE-1"):
    s3.execute_version(1, D_0706)
# ✗ EXE-PRE-2: a promotional placement now sits where SKU-A is planned
s3.add_promotional_placement("SKU-B", "RUN-1L1-E", 3.0, 0.0, D_0102, D_0929, None)
with expect_violation(PreconditionViolation, "EXE-PRE-2"):
    s3.execute_version(1, D_0105)
# EXE-PRE-3 has no client counter-example: agreements are never deleted, so no sequence of correct calls breaks it.
# ✗ supplier EXE-1: SKU-A's unchanged entry is ended and restarted under a new id
p3 = s5.placements["PLC-000003"]
same = dict(sku=p3.sku, space=p3.space, extent=p3.extent, offset_in=p3.offset_in,
            agreement=p3.agreement, placement_type=p3.placement_type)
ended = Stub(placement_id="PLC-000003", start_date=D_0105, end_date=D_0706, window=DateWindow(D_0105, D_0706), **same)
again = Stub(placement_id="PLC-000013", start_date=D_0706, end_date=None, window=DateWindow(D_0706, None), **same)
wrong = Stub(placements={**s5.placements, "PLC-000003": ended, "PLC-000013": again},
             skus=s5.skus, planogram=s5.planogram)
assert not P.EXE_1(wrong, None, s4.snapshot(), version_no=2, on=D_0706)   # old: the placements on entry, as at S4
```

**`add_agreement(agreement_type, vendor, fee_basis, fee_rate, window) -> str`**

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| AGC-PRE-1 | require | `vendor != ""` and AGR-INV-1, AGR-INV-3, AGR-INV-4, AGR-INV-5 hold for the arguments | `[VS 3]` |
| AGC-PRE-2 | require | `agreement_type != SLOTTING or window.start in reset_calendar` (AGR-INV-2) | `[VS 3, 4]` |
| AGC-1 | ensure | `result == f"AGR-{old_next:06d}"` with `old_next` the snapshot's `AGR` counter; `agreements[result]` has exactly these attributes | `[SIM]` |
| AGC-2 | ensure | `changed(old, self.snapshot()) == {("agreement", result), ("counter", "AGR")}` | `[DbC]` |

```python
# ✓ S1 → S2 (AGC-1, AGC-2) and S5 → part of S6
assert s2.agreements["AGR-000001"].fee_basis is FeeBasis.PER_LINEAR_FT and s2.next_id_number("AGR") == 2
# ✗ AGC-PRE-1: an opportunistic buy that charges for space
with expect_violation(PreconditionViolation, "AGC-PRE-1"):
    s1.add_agreement(AgreementType.OPPORTUNISTIC_BUY, "Acme Foods", FeeBasis.FLAT, 50.0, DateWindow(D_0105, None))
# ✗ AGC-PRE-2: slotting that does not start at a reset
with expect_violation(PreconditionViolation, "AGC-PRE-2"):
    s1.add_agreement(AgreementType.SLOTTING, "Acme Foods", FeeBasis.PER_FACING, 5.0, DateWindow(D_0901, None))
```

**`add_promotional_placement(sku_id, leaf_id, extent, offset_in, start_date, end_date, agreement_id) -> str`** — a time-boxed extra location, such as a promotional or vendor-funded display `[VS 3]`

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| PRM-PRE-1 | require | `has_sku(sku_id) and skus[sku_id].promo_display_eligible` | `[VS 3]` |
| PRM-PRE-2 | require | `has_space(leaf_id)`, `space(leaf_id)` is a `LeafSpace`, and `start_date < end_date` (PLC-INV-8, PLC-INV-9) | `[VS 3, 14]` |
| PRM-PRE-3 | require | for the arguments: `fits_dimensionally(sku.product, leaf)`, temperature (PLC-INV-6), secured (PLC-INV-11), whole facings (PLC-INV-4), and offset (PLC-INV-5) | `[VS 8]` `[TB 4.4, 6.4]` |
| PRM-PRE-4 | require | `leaf.can_hold(sku.product, extent, DateWindow(start_date, end_date))`; for a `LINEAR_IN` leaf, `[offset_in, offset_in + extent)` lies inside one interval of `leaf.free_intervals(DateWindow(start_date, end_date))`. So LS-INV-5, LS-INV-6, and BIN-INV-2 hold afterwards | `[VS 14]` |
| PRM-PRE-5 | require | `leaf.assigned_category is None or sku.category.is_within(leaf.assigned_category)` — promotional placements go on unassigned space (displays, end caps) or the SKU's own category space | `[SIM]` |
| PRM-PRE-6 | require | `agreement_id is None or (has_agreement(agreement_id) and agreements[agreement_id].agreement_type == PROMOTIONAL_DISPLAY and window_covers(agreements[agreement_id].window, DateWindow(start_date, end_date)) and agreements[agreement_id].vendor == sku.manufacturer)` | `[VS 3]` |
| PRM-1 | ensure | `result == f"PLC-{old_next:06d}"`; `placements[result]` has exactly these arguments, `placement_type == PROMOTIONAL` | `[VS 3]` |
| PRM-2 | ensure | `changed(old, self.snapshot()) == {("placement", result), ("counter", "PLC")}` | `[DbC]` |

A SKU needs no permanent placement first: promotional deals are a separate contract type from the baseline space agreement (Q-26). `[VS 3]`

```python
# ✓ S5 → part of S6 (PRM-1): three facings of SKU-A on the floor display
assert s6.placements["PLC-000012"].facings == 3 and s6.placements["PLC-000012"].vendor_funded
# ✗ PRM-PRE-1: SKU-C is not promo-eligible
with expect_violation(PreconditionViolation, "PRM-PRE-1"):
    s6.add_promotional_placement("SKU-C", "RUN-1F1-E", 4.0, 0.0, D_0901, D_0929, None)
# ✗ PRM-PRE-2: end precedes start
with expect_violation(PreconditionViolation, "PRM-PRE-2"):
    s6.add_promotional_placement("SKU-A", "RUN-1F1-E", 8.0, 0.0, D_0929, D_0901, None)
# ✗ PRM-PRE-3: 10 in on the end cap is not a whole number of 8 in facings
with expect_violation(PreconditionViolation, "PRM-PRE-3"):
    s6.add_promotional_placement("SKU-A", "RUN-1F1-E", 10.0, 0.0, D_0901, D_0929, None)
# ✗ PRM-PRE-4: 5.5 sq ft on FD-1 while PLC-000012 holds 1.0 of its 6.0
with expect_violation(PreconditionViolation, "PRM-PRE-4"):
    s6.add_promotional_placement("SKU-B", "FD-1", 5.5, None, D_0910, D_0929, None)
# ✗ PRM-PRE-5: a produce bin is Produce's space
with expect_violation(PreconditionViolation, "PRM-PRE-5"):
    s1.add_promotional_placement("SKU-B", "BIN-2", 8.0, None, D_0901, D_0929, None)
# ✗ PRM-PRE-6: a slotting deal cannot fund a promotion
with expect_violation(PreconditionViolation, "PRM-PRE-6"):
    s6.add_promotional_placement("SKU-A", "RUN-1F1-E", 8.0, 0.0, D_0901, D_0929, "AGR-000001")
```

**`add_incidental_placement(sku_id, units_held, start_date, end_date, zone_hint_id, agreement_id) -> str`**

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| INP-PRE-1 | require | `has_sku(sku_id)`; INC-INV-1 and INC-INV-2 hold for the arguments; `zone_hint_id is None or has_space(zone_hint_id)` | `[VS 3]` |
| INP-PRE-2 | require | `agreement_id is None or (has_agreement(agreement_id) and` INC-INV-3 holds for it`)` | `[VS 3]` |
| INP-1 | ensure | `result == f"INC-{old_next:06d}"`; `incidentals[result]` has exactly these arguments | `[VS 3]` |
| INP-2 | ensure | `changed(old, self.snapshot()) == {("incidental", result), ("counter", "INC")}` | `[DbC]` |

```python
# ✓ S5 → part of S6 (INP-1)
assert s6.incidentals["INC-000001"].units_held == 24
# ✗ INP-PRE-1
with expect_violation(PreconditionViolation, "INP-PRE-1"):
    s6.add_incidental_placement("SKU-B", 0, D_0910, None, None, None)
# ✗ INP-PRE-2: a slotting deal on overstock
with expect_violation(PreconditionViolation, "INP-PRE-2"):
    s6.add_incidental_placement("SKU-A", 12, D_0910, None, None, "AGR-000001")
```

**`move_incidental(placement_id, zone_hint_id)`**

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| MVI-PRE-1 | require | `placement_id in incidentals` and (`zone_hint_id is None or has_space(zone_hint_id)`) | `[VS 3]` |
| MVI-1 | ensure | `incidentals[placement_id].zone_hint` is the named node (or `None`) | `[VS 3]` |
| MVI-2 | ensure | `changed(old, self.snapshot()) <= {("incidental", placement_id)}` | `[DbC]` |

```python
# ✓ S6: the overstock moves to the backroom
s6.move_incidental("INC-000001", "AREA-BR")
assert s6.incidentals["INC-000001"].zone_hint.id == "AREA-BR"
assert "INC-000001" not in {p.placement_id for p in s6.placements_in_space(s6.space("SID-1L"))}
# ✗ MVI-PRE-1: a committed placement cannot be moved this way
with expect_violation(PreconditionViolation, "MVI-PRE-1"):
    s6.move_incidental("PLC-000003", "AREA-BR")
```

**`end_placement(placement_id, on)`**

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| END-PRE-1 | require | `has_placement(placement_id)`, its `end_date is None`, and `on > start_date` | `[VS 4, 14]` |
| END-1 | ensure | its `end_date == on` | `[VS 4]` |
| END-2 | ensure | `changed(old, self.snapshot()) <= {("placement", placement_id), ("incidental", placement_id)}` | `[DbC]` |

```python
# ✓ S6: the overstock is cleared on 2026-10-15
s6.end_placement("INC-000001", D_1015); assert s6.incidentals["INC-000001"].end_date == D_1015
# ✗ END-PRE-1: PLC-000004 has already ended
with expect_violation(PreconditionViolation, "END-PRE-1"):
    s6.end_placement("PLC-000004", D_0915)
```

**`cancel_future_placement(placement_id, as_of)`** — the only deletion; it removes a placement that has not started

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| CAN-PRE-1 | require | `has_placement(placement_id)` and its `start_date > as_of` | `[VS 4]` `[SIM]` |
| CAN-1 | ensure | it is gone from `placements` or `incidentals`, from its leaf, and from its SKU; `next_id_number` is unchanged (ID-2) | `[SIM]` |
| CAN-2 | ensure | `changed(old, self.snapshot()) <= {("placement", placement_id), ("incidental", placement_id)}` | `[DbC]` |

```python
# ✓ S6: a future incidental is cancelled; its id is not reused
assert s6.add_incidental_placement("SKU-B", 12, D_1015, None, None, None) == "INC-000002"
s6.cancel_future_placement("INC-000002", D_0915)
assert not s6.has_placement("INC-000002") and s6.next_id_number("INC") == 3
# ✗ CAN-PRE-1: PLC-000003 has already started
with expect_violation(PreconditionViolation, "CAN-PRE-1"):
    s6.cancel_future_placement("PLC-000003", D_0915)
```

---
## Part 8 — The store's creation procedure and the persistence boundary

### 8.1 `Store.build` `[DbC]` `[VS 14]`

The whole store is created by one creation procedure, built bottom-up (0.8) from finished, unattached parts:

`Store.build(id, footprint_sqft, areas, products, skus, category_roots, business_unit_roots, reset_calendar, assignments={}, agreements=(), placements=(), incidentals=(), versions=())`

`assignments` maps space node ids to category ids. Records (`agreements`, `placements`, `incidentals`, `versions`) are detached objects whose ids are given (ID-2). Invalid source data is a precondition violation of `build`, never silently repaired.

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| BUILD-PRE-1 | require | `id.startswith("STORE-")`, `footprint_sqft > 0`; `areas`, `category_roots`, and `business_unit_roots` are unattached roots of finished hierarchies, and no node, SKU, or record belongs to another store | `[DbC]` |
| BUILD-PRE-2 | require | ST-INV-3 holds for the arguments: every id unique across all classes and correctly prefixed | `[SIM]` |
| BUILD-PRE-3 | require | ST-INV-2 holds for `footprint_sqft` and `areas` | `[VS 6]` |
| BUILD-PRE-4 | require | `upc`s are unique; every SKU's `product` is in `products` and its `category` in the given category hierarchies; CAT-INV-1 holds for every root and CAT-INV-2 … CAT-INV-10 for every node; BU-INV-2 and BU-INV-3 hold | `[VS 1, 6]` |
| BUILD-PRE-5 | require | `reset_calendar` is strictly ascending (ST-INV-6) | `[VS 4]` |
| BUILD-PRE-6 | require | every assignment satisfies ASN-PRE-1 … ASN-PRE-5, and together they satisfy SN-INV-2 | `[SIM]` |
| BUILD-PRE-7 | require | every record is detached; and, once attached, every invariant of Parts 3–7 holds, including LS-INV-5, LS-INV-6, BIN-INV-2, FD-INV-2, AGR-INV-*, PLC-INV-*, INC-INV-*, PGV-INV-*, and HOME-2 | `[VS 14]` |
| BUILD-1 | ensure | the new store has the given attributes; every area's `parent` is the store; every record is attached (in its leaf, its SKU, and the store's mappings); `own_assignment` is set from `assignments`; `planogram.working is None` | `[DbC]` |
| BUILD-3 | ensure | `next_id_number(x)` is one more than the largest number among the given records of prefix `x`, or `1` if there are none (ID-2) | `[SIM]` |
| BUILD-2 | ensure | ST-INV-1 … ST-INV-9 hold for the new store | `[DbC]` |

**Examples (Store.build)** — `kw = s0_build_args()` gives fresh S0 arguments (conftest)

```python
# ✓ BUILD-1, BUILD-2 (this is S0)
st = Store.build(**s0_build_args())
assert st.space("AREA-SF").parent is st and st.planogram.working is None and st.next_id_number("PLC") == 1
# ✗ BUILD-PRE-1
with expect_violation(PreconditionViolation, "BUILD-PRE-1"):
    Store.build(**{**s0_build_args(), "id": "SHOP-1"})
# ✗ BUILD-PRE-2: two SKUs with one id
kw = s0_build_args()
dup = SKU("SKU-A", node(kw, "SKU-B").product, node(kw, "SKU-B").category, 4.0, 3.0, 1.0)
with expect_violation(PreconditionViolation, "BUILD-PRE-2"):
    Store.build(**{**kw, "skus": kw["skus"] + [dup]})
# ✗ BUILD-PRE-3: the building is 1300 sq ft but its areas add to 1200
with expect_violation(PreconditionViolation, "BUILD-PRE-3"):
    Store.build(**{**s0_build_args(), "footprint_sqft": 1300.0})
# ✗ BUILD-PRE-4: SKU-P2's product is missing
kw = s0_build_args()
with expect_violation(PreconditionViolation, "BUILD-PRE-4"):
    Store.build(**{**kw, "products": [p for p in kw["products"] if p.upc != "100000000032"]})
# ✗ BUILD-PRE-5
with expect_violation(PreconditionViolation, "BUILD-PRE-5"):
    Store.build(**{**s0_build_args(), "reset_calendar": (D_0706, D_0105)})
# ✗ BUILD-PRE-6: a floor display assigned as category space
with expect_violation(PreconditionViolation, "BUILD-PRE-6"):
    Store.build(**{**s0_build_args(), "assignments": {"FD-1": "CAT-CER"}})
# ✗ BUILD-PRE-7: a permanent placement starting off the reset calendar
kw = s0_build_args()
off = Placement("PLC-000001", node(kw, "SKU-A"), node(kw, "RUN-1L1-E"), PlacementType.PERMANENT, 48.0, 0.0, date(2026, 3, 1))
with expect_violation(PreconditionViolation, "BUILD-PRE-7"):
    Store.build(**{**kw, "assignments": {"RUN-1L1-E": "CAT-CER"}, "placements": [off]})
# ✓ BUILD-3: numbering continues after the seed records
kw = s0_build_args()
seed = Placement("PLC-000007", node(kw, "SKU-A"), node(kw, "RUN-1L1-E"), PlacementType.PERMANENT, 48.0, 0.0, D_0105)
args = {**kw, "assignments": {"RUN-1L1-E": "CAT-CER"}, "placements": [seed]}
assert Store.build(**args).next_id_number("PLC") == 8
# ✗ supplier BUILD-3: a counter that ignores the seed records
assert not P.BUILD_3(Stub(next_id_number=lambda x: 1), None, None, **args)
```

### 8.2 Persistence boundary `[VS 5]` `[SIM]`

The contract does not specify storage. It constrains the model so that any storage works.

| ID | Kind | Clause | Grounding |
|---|---|---|---|
| PERS-1 | review | Instance data is **seed data**: loaded once through `Store.build` to populate a real database, after which the simulation reads and writes there. The source layout (workbook, generator, or otherwise) is outside this contract | `[VS 5]` |
| PERS-2 | review | Every command in 7.3 is atomic, so each maps to one transaction. Whatever the model holds in memory is a cache over the system of record, written through | `[VS 5]` |
| PERS-3 | review | Ids are assigned by the model (ID-2), never taken from a storage row order, and are stable across save and load | `[VS 5]` `[SIM]` |
| PERS-4 | review | A rollup, filter, or subtree query may be implemented as a database query, provided its clauses (RU-*, FLT-*, HN-Q-*) and HN-IMPL-1 still hold | `[VS 6, 13]` |

---
## Part 9 — The contract by example `[VS 13, 14]`

The example usages in this part are **required acceptance tests** (0.1). They are client code the model must be able to run. If the model cannot express one cleanly, the model is wrong. Positive usages are specification, invariants are guard rails, and the matrix proves the hierarchies behave. `[VS 14]`

### 9.1 The reference fixture

All values below are computed against **EX-STORE-1** (0.10), shipped as the `conftest.py` of 0.10.2. Unless stated, `store = s6` and the dates are 2026.

### 9.2 Worked usages

```python
store = s6
# Space: where is the room?
sid, fd = store.space("SID-1L"), store.space("FD-1")
assert sid.capacity(SpaceUnit.LINEAR_IN) == 384.0
assert sid.allocated_amount(SpaceUnit.LINEAR_IN, D_0915) == 81.0            # 48 + 24 + 4 + 5
assert sid.unallocated_amount(SpaceUnit.LINEAR_IN, D_0915) == 303.0
assert fd.allocated_amount(SpaceUnit.SQFT, D_0915) == 1.0 and fd.unallocated_amount(SpaceUnit.SQFT, D_0915) == 5.0
assert fd.allocated_amount(SpaceUnit.SQFT, D_1015) == 0.0                   # the promotion has ended

# What was on this shelf on this date?
run = store.space("RUN-1L1-B")
assert {p.placement_id for p in run.placements_on(D_0915)} == {"PLC-000007", "PLC-000010", "PLC-000011"}
assert run.placements_on(date(2025, 12, 1)) == frozenset()

# Facings are derived, never entered
assert store.placements["PLC-000011"].facings == 8 and store.placements["PLC-000012"].facings == 3

# Home position falls out of the planogram, not a flag
assert store.home_placement(store.skus["SKU-A"], D_0915) is store.placements["PLC-000003"]
assert store.home_placement(store.skus["SKU-A"], date(2026, 1, 4)) is None

# Zero is legitimate
assert store.skus["SKU-H"].placements == ()

# Unit-honest rollups: inches, square feet, and hooks never add
assert (store.capacity(SpaceUnit.LINEAR_IN), store.capacity(SpaceUnit.SQFT), store.capacity(SpaceUnit.HOOKS)) == (984.0, 23.0, 10.0)
assert store.space("AREA-BR").capacity(SpaceUnit.LINEAR_IN) == 0.0          # non-merchandising space
assert store.footprint_sqft == 1200.0
assert (store.facings(D_0915), store.units_capacity(D_0915), store.placement_count(D_0915)) == (62, 740, 11)
```

### 9.3 The query matrix: each hierarchy × each query shape `[VS 13]`

| Query shape | Space hierarchy | Category hierarchy | Business-unit hierarchy |
|---|---|---|---|
| **Roll up** | `sid.allocated_amount(LINEAR_IN, D_0915) == 81.0` | `category("CAT-DG").facings(D_0915) == 23` | `business_unit_node("BU-CO").facings(D_0915) == 62` |
| **Drill down** | `[c.id for c in space("GDL-1").children] == ["SID-1L", "SID-1F"]`, and `space("GDL-1").allocated_amount(LINEAR_IN, D_0915) == sum(c.allocated_amount(LINEAR_IN, D_0915) for c in space("GDL-1").children)` | `[c.id for c in category("CAT-DG").children] == ["CAT-CER", "CAT-SPI", "CAT-HBA"]`, and `category("CAT-DG").facings(D_0915) == 19 + 4 + 0` | `[c.id for c in business_unit_node("BU-ST").children] == ["BU-DG", "BU-DY", "BU-PR"]`, and `business_unit_node("BU-ST").units_capacity(D_0915) == 740` |
| **Filter by type** | `len(spaces_of_type(GONDOLA)) == 18`; `len(spaces_of_type(MERCHANDISING)) == 40` | `ids(placements_in_category(category("CAT-CER")) & placements_of_type(PROMOTIONAL)) == {"PLC-000012"}` | `ids(placements_in_unit(business_unit_node("BU-DG")) & placements_with_agreement_type(SLOTTING)) == {"PLC-000003", "PLC-000004"}` |
| **Filter by date** | `ids(sid.placements_on(D_1015)) == {"PLC-000003", "PLC-000006", "PLC-000007", "PLC-000009", "PLC-000010", "PLC-000011"}` | `category("CAT-RTE").placement_count(D_1015) == 3` (incidentals excluded) | `business_unit_node("BU-DG").placement_count(D_1015) == 6` |
| **Allocated vs. unallocated** | `sid.unallocated_amount(LINEAR_IN, D_0915) == 303.0`; `fd.unallocated_amount(SQFT, D_0915) == 5.0` | `category("CAT-CER").unallocated_amount(LINEAR_IN, D_0915) == 15.0`; `category("CAT-CER").allocated_amount(SQFT, D_0915) == 0.0` (`FD-1` is not category space) | `business_unit_node("BU-ST").allocated_amount(LINEAR_IN, D_0915) == 116.0` and `business_unit_node("BU-ST").unallocated_amount(LINEAR_IN, D_0915) == 100.0` |

`ids(F)` is `{p.placement_id for p in F}`. A business unit's space measures cover only space assigned to its categories (216 linear inches here), unlike the store's own 984. The category column shows the two families of measure: the promotional placement on the unassigned floor display counts in `CAT-CER`'s `facings` (19) but not in its space measures.

**Set-algebra tests** (no query language; plain Python sets):

```python
store = s6
sid, fd = store.space("SID-1L"), store.space("FD-1")
ids = lambda F: {p.placement_id for p in F}
live_915 = store.placements_live_on(D_0915)
assert len(live_915) == 12                                                       # 11 committed + INC-000001
assert len(store.placements_live_on(date(2026, 9, 5))) == 11                     # INC-000001 not yet started
assert ids(store.placements_live_on(D_1015) - store.placements_of_type(PlacementType.INCIDENTAL)) == {
    f"PLC-{i:06d}" for i in (1, 2, 3, 5, 6, 7, 8, 9, 10, 11)}                    # but not = difference
assert ids(store.placements_in_space(sid) | store.placements_in_space(fd)) == {
    "PLC-000003", "PLC-000004", "PLC-000006", "PLC-000007", "PLC-000009", "PLC-000010", "PLC-000011",
    "PLC-000012", "INC-000001"}                                                  # or = union
assert ids(live_915 & store.placements_of_type(PlacementType.PERMANENT) & store.placements_in_space(sid)) == {
    "PLC-000003", "PLC-000006", "PLC-000007", "PLC-000009", "PLC-000010", "PLC-000011"}   # and = intersection
```

### 9.4 Allocation usage: two resets

```python
st = s2
plan = st.generate_plan(scope_all(st), D_0105)            # a query: the store is unchanged
assert len(plan.entries) == 10 and len(plan.unplaced) == 5
agr = st.agreements["AGR-000001"]
st.set_working_plan(plan.with_agreement(st.skus["SKU-A"], agr).with_agreement(st.skus["SKU-B"], agr))
v = st.release_working_plan(VersionKind.CYCLE, D_0102, D_0105)            # S3
st.execute_version(v, D_0105)                                              # S4
assert st.home_placement(st.skus["SKU-A"], D_0105).placement_id == "PLC-000003"
plan2 = st.generate_plan(scope_all(st), D_0706).with_agreement(st.skus["SKU-A"], agr)
st.set_working_plan(plan2)
st.execute_version(st.release_working_plan(VersionKind.CYCLE, D_0701, D_0706), D_0706)   # S5
assert st.placements["PLC-000003"].end_date is None                        # unchanged entry survives
assert st.placements["PLC-000004"].end_date == D_0706                      # slotting not renewed: replaced
assert st.home_placement(st.skus["SKU-B"], D_0706).placement_id == "PLC-000011"
```

### 9.5 The negative cases: things that must raise `[VS 14]`

A model that cannot be made to fail correctly usually cannot be trusted when it succeeds. Each row is a required test. "Raises" means the named exception carries the named clause ID at level `ALL`.

| # | Action on EX-STORE-1 | Raises | Clause |
|---|---|---|---|
| N-1 | `Bay("BAY-X", GONDOLA, 1, 48.0, list(s0.space("BAY-1L1").children))` — attaching nodes that already have a parent, the only way a cycle could form | `PreconditionViolation` | BAY-NEW-PRE-2 (HN-INV-3) |
| N-2 | `s6.add_promotional_placement("SKU-A", "RUN-1F1-E", 8.0, 0.0, D_0929, D_0901, None)` — end precedes start | `PreconditionViolation` | PRM-PRE-2 (PLC-INV-8) |
| N-3 | `s6.add_promotional_placement("SKU-B", "FD-1", 5.5, None, D_0910, D_0929, None)` — 1.0 + 5.5 > 6.0 sq ft | `PreconditionViolation` | PRM-PRE-4 (LS-INV-5) |
| N-4 | `s6.add_promotional_placement("SKU-B", "RUN-1L1-E", 6.0, 10.0, D_0910, D_0929, None)` — overlaps `PLC-000003`'s segment `[0, 48)` | `PreconditionViolation` | PRM-PRE-4 (LS-INV-6) |
| N-5 | `Store.build` with two `PERMANENT` placements on `RUN-1L1-E` at offsets 0 and 4, extent 8, both from 2026-01-05 | `PreconditionViolation` | BUILD-PRE-7 (LS-INV-6) |
| N-6 | `P.RU_1` on a stub whose children do not sum to it (2.3) | predicate returns `False` | RU-1 |
| N-7 | `Placement("PLC-000099", s6.skus["SKU-S1"], s6.space("RUN-1L2-M"), PERMANENT, 3.0, 0.0, D_0706)` — a hanging product on a front-facing shelf | `PreconditionViolation` | PLC-NEW-PRE-3 (PLC-INV-3) |
| N-8 | `s1.assign_space("DCR-1", "CAT-DG")` — refrigerated doors to an ambient category | `PreconditionViolation` | ASN-PRE-3 (CAT-INV-9, LS-INV-7) |
| N-9 | `s1.assign_space("BAY-1L1", "CAT-RTE")` while `RUN-1L1-E` is assigned | `PreconditionViolation` | ASN-PRE-2 (SN-INV-2) |
| N-10 | `s3.execute_version(1, D_0706)` | `PreconditionViolation` | EXE-PRE-1 |
| N-11 | `s6.placements["PLC-000003"].extent = 3.0` | `InvariantViolation` | PLC-INV-IMM |
| N-12 | `s6.end_placement("PLC-000004", D_0915)` — already ended | `PreconditionViolation` | END-PRE-1 |
| N-13 | `Store.build` with a `PERMANENT` placement starting 2026-03-01 (8.1 example) | `PreconditionViolation` | BUILD-PRE-7 (PLC-INV-14) |
| N-14 | `release_working_plan(CYCLE, …, effective_date=2026-03-02)` (7.3 example) | `PreconditionViolation` | REL-PRE-2 (PGV-INV-2) |
| N-15 | at S3, set a new working plan and release it while version 1 is `RELEASED` | `PreconditionViolation` | REL-PRE-3 (PGV-INV-4) |
| N-16 | `Store.build` with two placements on `BIN-1` whose windows overlap | `PreconditionViolation` | BUILD-PRE-7 (BIN-INV-2) |
| N-17 | `Area("AREA-X", "Bad", SALES_FLOOR, 50.0, [Area("AREA-Y", "Aisle", AISLE, 5.0, [])])` | `PreconditionViolation` | AREA-NEW-PRE-3 (AREA-INV-2) |
| N-18 | `s2.generate_plan(frozenset(), D_0105)` | `PreconditionViolation` | GEN-PRE-1 |
| N-19 | `s1.assign_space("FD-1", "CAT-CER")` — a floor display as category space | `PreconditionViolation` | ASN-PRE-3 (CAT-INV-10) |
| N-20 | `s1.add_agreement(SLOTTING, "Acme Foods", PER_FACING, 5.0, DateWindow(D_0901, None))` | `PreconditionViolation` | AGC-PRE-2 (AGR-INV-2) |
| N-21 | with `assertion_level(NONE)` active, a call that would violate a precondition does not raise; on leaving the block, the previous level is back | no exception; `get_assertion_level() is ALL` afterwards | MON-4, MON-6 |

Runtime-only checks (cycle detection, date overlap, over-subscription) need the running model. The implementation is delivered with them as tests, alongside the matrix in 9.3. `[VS 14]`

**Examples (monitoring API)**

```python
from supermarket_sim.contracts import (AssertionLevel, assertion_level, get_assertion_level,
                                       set_assertion_level, check_invariants, predicate)
# ✓ MON-3, MON-4: the context manager restores the level, even after an exception
assert get_assertion_level() is AssertionLevel.ALL
try:
    with assertion_level(AssertionLevel.REQUIRE):
        raise RuntimeError
except RuntimeError:
    pass
assert get_assertion_level() is AssertionLevel.ALL
# ✓ MON-5: two preconditions false; the first in table order is reported, and nothing changed
old = s6.snapshot()
with expect_violation(PreconditionViolation, "PRM-PRE-1"):
    s6.add_promotional_placement("SKU-C", "RUN-1F1-E", 8.0, 0.0, D_0929, D_0901, None)   # also breaks PRM-PRE-2
assert s6.snapshot() == old
# ✓ MON-6: at NONE nothing is checked (used only to build counter-examples)
with assertion_level(AssertionLevel.NONE):
    DateWindow(D_0929, D_0901)
# ✓ MON-7: a sound object passes
assert check_invariants(s6.placements["PLC-000003"]) is None
# ✓ MON-8: predicates by clause id
assert predicate("PLC-Q-1") is P.PLC_Q_1 and P.PLC_Q_1(s6.placements["PLC-000003"], 6, None)
# ✗ MON-8: no predicate exists for a review clause
with pytest.raises(KeyError):
    predicate("HN-IMPL-1")
```

---
## Part 10 — Reserved areas (not yet specified)

Nothing in this part is part of the model yet. Each entry lists what the area will need from Parts 1–9, so that later versions extend the contract instead of reopening it (0.1). Every listed dependency is a feature this version already exposes.

| Area | Will add | Depends on (already specified) |
|---|---|---|
| **Simulation clock** | A `Clock` with a current date and an `advance(to)` command. Replaces the explicit `on` / `as_of` arguments as the default; they stay as overrides | rule 10, `Store.change_dates()` (ST-Q-4), `reset_calendar` |
| **Staggered reset execution** | Execution of a `RELEASED` version over several nights and stores, replacing the instantaneous `execute_version` with a multi-step process (Q-19) | PGV-*, EXE-1 (the end state it must reach), REL-PRE-3 (queued revisions) |
| **Observed shelf state and compliance** | A third artefact: what is actually on the shelf per leaf and offset, and a compliance measure against `current_version` — execution drift versus intended revision `[VS 10]` | PGV-Q-1, `Placement` and `LeafSpace` geometry; the rule that no placement or version records observed state (6.5) |
| **Product restocking** | Shelf inventory per placement (units on shelf against `units_capacity`), replenishment triggers, and work orders from backroom to shelf | PLC-Q-3, HOME-1 (replenishment defaults to the home position), `respace_without_reset`, `IncidentalPlacement.units_held` |
| **Product reordering** | Reorder points and order quantities, driven by velocity and inventory position `[TB 3.2–3.4]` | `SKU.sales_velocity_units_per_week`, `Product.units_per_case`, `Product.case_dims_in`, `SKU.assortment_status` |
| **Product purchasing from vendors** | Purchase orders, vendor terms, and cost. Agreements attach here too: `OPPORTUNISTIC_BUY` is a deal on the product, not the shelf | `Agreement` (5.1), `SKU.unit_cost`, `Product.manufacturer` |
| **Product receiving** | Receipt against purchase orders, cold-chain checks at the dock, stock into backroom or onto the shelf; DSD vendor-managed space (Q-24) `[TB 4.1–4.4]` | `TemperatureZone`, `NON_MERCHANDISING` backroom and prep space (2.2), incidental cold-chain rules (5.3), `Product.case_dims_in` |
| **Sales to store customers** | Sales transactions per SKU per day, drawing down shelf inventory. Revenue, gross margin, and sales per linear foot become live measures | reserved measures (2.3, additive under RU-1), `SKU.retail_price`, `SKU.unit_margin`, `Placement` as the leaf fact revenue attaches to `[VS 9]` (Q-9) |
| **Assortment changes** | Commands that add, trial, and delist SKUs at reset windows `[TB 2.6]` (Q-18) | `AssortmentStatus`, PLAN-INV-6, SKU-ZERO |
| **Hierarchy maintenance** | Commands to re-parent, add, and remove nodes in the four hierarchies (Q-14) | HN-*, HN-IMPL-1, bottom-up creation (0.8), FRM-1 (the snapshot will then include structure) |
| **Multi-store and localized planograms** | More than one `STORE` unit; planogram mods per store `[TB 2.2]` (Q-23) | BU-INV-3 (exactly one store today), the business-unit hierarchy |

**Constraints on the extension.** These follow from the design rules and bind every later section:

1. New state that changes over time is recorded with effective dates, as a placement is, so that "what was true on date d" stays one uniform query. `[VS 4]`
2. New measures are additive and declared as `Rollup` measures, with their unit stated (RU-4).
3. New space attributes go on the space class of the type they belong to, not on a shared base class (rule 5).
4. Money attaches to agreements and transactions, never to space (rule 8).
5. Every new command is atomic, has a frame condition (FRM-3), a predicate for every postcondition (MON-8), and a negative case in Part 9.
6. Every new class has a creation procedure (0.8) and an Examples block on EX-STORE-1, extended only by adding states.

---

## Part 11 — Change log, v0.2 → v0.3

### 11.1 What drove the revision

1. The revised contract-creation prompt adds testability requirements: a monitoring API (0.4), explicit creation procedures (0.8), frame conditions (0.7), exclusive outcome reasons (ENUM-5), id rules (0.6), a deterministic allocation (GEN-12), per-routine Examples blocks on one fixture (0.10), and a grounding index (Appendix A).
2. The companion file's carry-forward part (CF-1 … CF-13, items from a retired draft) is resolved: each item is adopted or dropped, logged as a decision D-n in the companion file.
3. Defects found while writing the examples (D-15, D-16, D-17, D-23).

Clause IDs are stable from v0.2. No ID is reused. Clauses whose text changed keep their ID; the evolution rule (0.1) is satisfied by naming each changed clause below.

### 11.2 Revised clauses

| Clause | Change | Strength | Reason |
|---|---|---|---|
| GDL-INV-1, GDL-INV-2, GDL-INV-3, GDL-INV-4, GDL-INV-5 | Long faces are `LEFT`/`RIGHT`; end caps are `FRONT`/`BACK` (was the reverse) | changed meaning | CF-1, D-1 |
| SID-INV-2, SID-INV-1 | wording only (positions in child order) | same | 0.8 |
| GEN-4 | Rewritten as one exclusive ladder in ENUM-5 order, adding `OUT_OF_SEASON` first | strengthened postcondition | CF-4, D-5 |
| GEN-5 | upper bound `T(e)` (was `max(lo, T(e))`; equal, since POL-2 clamps) | same | simplification |
| GEN-6 | "no free block adjacent" became "no adjacent free interval of at least one facing" | weakened postcondition | D-16: the v0.2 wording could not be met by any plan with whole facings |
| GEN-7 | "unless GEN-6 blocked `a`" became "unless `a` is under target" | same | precision |
| GEN-8 | the swap is defined (each keeps its extent at the other's position) | same | precision |
| GEN-9 | tie-breaks moved into GEN-12's procedure | weakened (moved) | D-2 |
| GEN-10 | stated with `changed` | same | FRM-3 |
| GEN-11 | one entry per SKU, so home is that entry | same | PLAN-INV-3 |
| GEN-PRE-1 | adds existence of every scope node in this store | strengthened precondition | 0.7 |
| PLAN-INV-4 | in-scope permanents starting after `as_of` are removed, not ended | precision | D-2 |
| PLAN-INV-6 | precise bottom-decile rule, `floor(n / 10)`, per leaf category, with an order | precision (was `[SIM]` and unstated) | CF-5, D-6 |
| POL-2 | `target_facings` takes `as_of`; `+ EPS` inside `floor` | signature change | D-23 |
| POL-4 | "never affect a plan's quality measure" became "never affect allocation order" | same | D-2 |
| SN-INV-1 | `CHECKOUT` areas may hold impulse racks; containers no longer violate it | weakened invariant | D-15: v0.2's clause was false for the store, the sales floor, and every checkout area |
| SN-INV-3 | parent test by class rather than "an Area type" | same | 0.8 |
| SN-INV-4 | prefix per class only; global uniqueness moved to ST-INV-3 | same | ID-1 |
| LS-INV-5 | checked at the leaf's own placement start dates, not the store's change dates | same (equivalent) | 0.8: a detached leaf has no store |
| LS-Q-2, LS-Q-3 | take an optional `without` placement | extended | GEN-13 |
| LS-Q-4 | now `room(w, without)`; the old `can_hold` is LS-Q-5 | renumbered within the class | GEN-13 |
| FIT-2 | defines `consumption`, with a require (FIT-PRE-1) | changed | D-2 |
| PROD-Q-2, PROD-Q-3 | requires split out as PROD-PRE-1, PROD-PRE-2 | same | MON-5 |
| AGR-INV-1 | the window check moved to `DateWindow` (DW-NEW-PRE-1) | same | 1.1 |
| AGR-Q-1 | require split out as AGR-PRE-1 | same | MON-5 |
| PLC-INV-4 | the bin rule became part of the clause | same | — |
| PLC-INV-12 | adds `agreement.vendor == sku.manufacturer` | strengthened invariant | AGR-INV-7 at the placement |
| PLC-INV-IMM | direct assignment raises `InvariantViolation("PLC-INV-IMM")` | precision | N-11 |
| PLC-Q-2 | `window` and `business_unit` moved to PREC-Q-1, PREC-Q-3 | same | 5.2 |
| HOME-2 | kind `invariant` on `Store`, checked at change dates | precision | MON-8 |
| PGV-INV-5 | rewritten as a valid expression | same | v0.2 syntax error |
| CAT-INV-6, CAT-INV-7, CAT-INV-2 … CAT-INV-5 | marked (A) | same | 0.8 |
| RU-1, RU-3 | checked over `rollup_dates()`; `own(m)` exposed | same | MON-9 |
| HN-Q-5 | also requires the same root | strengthened postcondition | 0.8: two detached trees |
| ST-INV-1 | every child of the store is an `Area` | strengthened | 3.3 |
| ST-INV-3, ST-INV-4 | cover every class and the SKU's product and category | strengthened | ID-1 |
| ST-Q-1 | requires split out as ST-PRE-1 | same | MON-5 |
| ST-Q-2 | require split out as ST-PRE-2 | same | MON-5 |
| ASN-PRE-1 | adds existence of the node and the category | strengthened precondition | 0.7 |
| ASN-2, UNA-1, WRK-1, REL-2, AGC-1, PRM-2, INP-1, MVI-1, END-1, CAN-1 | "nothing else changes" moved to frame clauses using `changed` (new IDs UNA-2, WRK-2, AGC-2, INP-2, MVI-2, END-2, CAN-2) | same | FRM-3 |
| REL-PRE-3 | adds the effective-date order of PGV-INV-4 | strengthened precondition | PGV-INV-4 |
| EXE-2 | stated with `changed` | same | FRM-3 |
| AGC-PRE-1 | adds `vendor != ""` | strengthened precondition | AGR-NEW-PRE-1 |
| PRM-PRE-1, PRM-PRE-2, PRM-PRE-6 | add existence checks | strengthened precondition | 0.7 |
| INP-PRE-1 | the agreement part moved to INP-PRE-2 | same | MON-5 |
| END-PRE-1, CAN-PRE-1 | existence via `has_placement` | same | 0.7 |
| BUILD-PRE-1 | split into BUILD-PRE-1 … BUILD-PRE-7 | same overall | one counter-example per precondition |
| BUILD-1 | split into BUILD-1, BUILD-2, BUILD-3 | same overall | ID-2 |
| ENUM-1 … ENUM-4 | moved into a clause table; ENUM-4's planes part refers to PROD-Q-1 | same | grounding index |
| TYP-3 | kind `invariant`; names the two container types | same | SN-INV-1 |
| CLP-INV-2 | holds once attached | same | 0.8 |
| FD-INV-3 | wording | same | — |
| AREA-INV-3 | `CHECKOUT` allowed explicitly | same | TYP-3 |

### 11.3 Retired clauses

| v0.2 | Replaced by | Reason |
|---|---|---|
| CAT-INV-8 | CAT-REV-1 | It was a documentation rule with no checkable expression; MON-8 requires every invariant to have a predicate |
| PLAN-INV-9 | PLAN-PRE-1, PLAN-Q-1 | It specified a routine (`with_agreement`), not an invariant |

### 11.4 New clauses

| Area | Clauses |
|---|---|
| Monitoring, ids, frames | MON-1 … MON-10, ID-1 … ID-4, FRM-1 … FRM-3 |
| Value types and functions | ENUM-5, ENUM-6, DW-NEW-PRE-1, DW-NEW-1, DW-Q-1 … DW-Q-3, ADDM-PRE-1, ADDM-1 |
| Hierarchy | HN-Q-6 |
| Space | LS-INV-11, LS-PRE-1, LS-PRE-2, LS-Q-5, FIT-PRE-1, BAY-INV-3, BAY-INV-4, RUN-INV-2, DOR-INV-4, SVC-INV-2, TIER-INV-2, SHR-INV-1, TBL-INV-2 |
| Creation procedures | `*-NEW-PRE-*` and `*-NEW-*` for AREA, RUN, BAY, PEG, CLP, SID, GDL, SHR, DOR, DCR, TIER, SVC, BIN, TBL, FD, NBS, PROD, SKU, CAT, BU, AGR, PLC, INC, PP, PLAN, PGV, PGM, POL |
| Products, SKUs, categories | PROD-PRE-1, PROD-PRE-2, SKU-INV-11, SKU-Q-3, CAT-INV-9, CAT-INV-10, CAT-REV-1, CAT-Q-5, CAT-Q-6 |
| Placements | PREC-Q-1 … PREC-Q-3, AGR-PRE-1 |
| Allocation | POOL-1 … POOL-4, POL-PRE-1, PLAN-PRE-1, PLAN-Q-1, GEN-12, GEN-13 |
| Store | ST-INV-9, ST-PRE-1 … ST-PRE-3, ST-Q-10, ST-Q-11, FLT-PRE-1, AGC-PRE-2, INP-PRE-2, EXE-6, UNA-2, WRK-2, AGC-2, INP-2, MVI-2, END-2, CAN-2, BUILD-PRE-2 … BUILD-PRE-7, BUILD-2, BUILD-3 |
| Acceptance tests | N-19 … N-21; N-1 … N-18 restated on EX-STORE-1 |

### 11.5 Other changes

- **Fixture.** v0.2's fixture `F` is replaced by EX-STORE-1 (0.10), a sequence of states with shipped `conftest.py`. All Part 9 values are recomputed.
- **Id prefixes.** One prefix per class (ID-1): `SHR-` replaces `WS-RUN-`, `BKY-RUN-`, `FLS-RUN-`; `DCR-` replaces `RIC-RUN-`, `RIF-RUN-`, `FLC-RUN-`; `DOR-` replaces `…-DOR-`; `SVC-` replaces `DELI-CASE-`, `SVC-CASE-`; `TIER-` replaces `…-TIER-`; `TBL-` and `BIN-` replace `PRD-TBL-` and `PRD-BIN-`. Created records use six-digit numbers (D-18).
- **Creation.** `Store.build` no longer takes loose nodes; it takes finished hierarchies built bottom-up (0.8, D-19). `ShelfRun`, `Bay`, `Door`, and `Tier` take their `space_type` at creation.
- **Normativity** is stated explicitly (0.1): clauses are the sole normative specification; Part 9 usages are required acceptance tests; Examples blocks are illustrative.
- **Assertion monitoring** (v0.2's 0.4) is replaced by the clause table MON-1 … MON-10.
- **Leaves** all carry `secured` (D-27).
- **Enumeration** `UnplacedReason` is declared in precedence order (ENUM-5); `NO_SPACE_IN_PLANE` stays, and `INCOMPATIBLE_ORIENTATION` from the retired draft is not reintroduced.

---
## Appendix A — Grounding index

Every clause of this contract, indexed by its grounding tags. A clause with several tags appears under each. Generated from the Grounding column of every clause table.

### `[TB x.y]` — textbook

| Section | Clauses |
|---|---|
| 1.2 | ASN-PRE-1, ST-Q-6 |
| 1.3 | LS-INV-7 |
| 2.1 | ASN-PRE-1, ASN-PRE-3, CAT-INV-6, GEN-2, GEN-3, PLC-INV-7, SKU-INV-11, SKU-NEW-PRE-3, SKU-Q-1 |
| 2.2 | GEN-1, LS-INV-6 |
| 2.3 | CAT-INV-2, CAT-INV-3, CAT-NEW-PRE-3, ENUM-2, GEN-5, GEN-6, GEN-7, GEN-8, GEN-13, PLAN-INV-6, PLC-INV-10, PLC-NEW-PRE-4, POL-1, POL-2, POL-3, SKU-INV-1, SKU-INV-3, SKU-INV-4, SKU-NEW-PRE-2 |
| 2.4 | POL-5 |
| 2.5 | CAT-INV-4, CAT-Q-4, SKU-INV-3 |
| 2.6 | CAT-INV-5, EXE-3, PLAN-INV-6, ST-Q-2 |
| 4.4 | ASN-PRE-4, CAT-INV-9, LS-INV-7, PLC-INV-6, PLC-NEW-PRE-4, PRM-PRE-3, TYP-2 |
| 6.4 | ASN-PRE-5, LS-INV-8, PLC-INV-11, PLC-NEW-PRE-4, PRM-PRE-3 |

### `[VS n]` — app-functionality file

| Section | Clauses |
|---|---|
| 1 | BUILD-PRE-4, PROD-INV-3, PROD-NEW-PRE-1, SKU-INV-10 |
| 2 | EXE-4, GEN-11, HOME-1, HOME-2, PGV-INV-5, PLAN-INV-7, SKU-Q-3, SKU-ZERO, ST-Q-8 |
| 3 | AGC-PRE-1, AGC-PRE-2, AGR-INV-1, AGR-INV-2, AGR-INV-3, AGR-INV-4, AGR-INV-5, AGR-INV-6, AGR-INV-7, AGR-NEW-PRE-2, AGR-PRE-1, AGR-Q-1, DW-Q-3, ENUM-3, FLT-2, FLT-6, INC-INV-1, INC-INV-3, INC-INV-4, INC-INV-5, INC-NEW-PRE-2, INP-1, INP-PRE-1, INP-PRE-2, MVI-1, MVI-PRE-1, PLAN-PRE-1, PLAN-Q-1, PLC-INV-2, PLC-INV-9, PLC-INV-12, PLC-NEW-PRE-2, PLC-NEW-PRE-5, PREC-Q-2, PRM-1, PRM-PRE-1, PRM-PRE-2, PRM-PRE-6 |
| 4 | AGC-PRE-2, AGR-INV-2, BUILD-PRE-5, CAN-PRE-1, CAT-INV-5, CAT-Q-2, CAT-Q-3, DW-Q-1, END-1, END-PRE-1, EXE-1, EXE-3, EXE-5, FLT-1, PGV-INV-2, PLC-INV-14, PLC-INV-IMM, PLC-NEW-PRE-2, PREC-Q-1, REL-PRE-2, SKU-Q-2, SN-Q-4, ST-INV-6, ST-Q-4, ST-Q-7, UNA-PRE-2 |
| 5 | PERS-1, PERS-2, PERS-3 |
| 6 | AREA-INV-1, AREA-INV-2, AREA-INV-3, AREA-NEW-PRE-1, AREA-NEW-PRE-2, AREA-NEW-PRE-3, BU-INV-1, BU-INV-2, BU-NEW-PRE-1, BU-Q-1, BUILD-PRE-3, BUILD-PRE-4, CAT-INV-1, CAT-NEW-PRE-1, CAT-Q-1, CAT-Q-5, FLT-3, FLT-4, FLT-5, HN-IMPL-1, PERS-4, SKU-INV-6, SN-INV-1, ST-INV-1, ST-INV-2, ST-Q-5, ST-Q-6, TYP-1, TYP-3 |
| 7 | CLP-INV-1, CLP-INV-2, CLP-NEW-PRE-1, ENUM-1, GDL-INV-1, LS-INV-9, PEG-INV-1, PEG-INV-2, PEG-NEW-PRE-1, SID-NEW-PRE-3, SN-INV-3 |
| 8 | ENUM-1, FD-INV-1, FIT-1, FIT-2, FIT-PRE-1, LS-INV-3, LS-INV-9, LS-PRE-1, LS-PRE-2, LS-Q-1, LS-Q-3, LS-Q-4, LS-Q-5, NBS-NEW-PRE-1, PLC-INV-1, PLC-INV-3, PLC-INV-4, PLC-INV-5, PLC-NEW-PRE-3, PLC-Q-1, PLC-Q-2, PLC-Q-3, POOL-1, PP-NEW-PRE-1, PRM-PRE-3, PROD-INV-1, PROD-INV-2, PROD-NEW-PRE-1, PROD-NEW-PRE-2, PROD-PRE-1, PROD-PRE-2, PROD-Q-2, PROD-Q-3, RU-4, RUN-INV-1, SN-Q-5 |
| 9 | BU-Q-1, PREC-Q-3, RU-1, RU-2, SN-Q-2, SN-Q-3, SN-Q-5 |
| 10 | EXE-1, EXE-4, EXE-PRE-1, ID-3, PGM-NEW-PRE-1, PGV-INV-1, PGV-INV-2, PGV-INV-3, PGV-INV-4, PGV-INV-5, PGV-NEW-PRE-1, PGV-Q-1, PLC-INV-14, REL-1, REL-PRE-1, REL-PRE-3, REL-PRE-4, WRK-1, WRK-PRE-1 |
| 12 | BLK-1, BLK-2, BLK-3, BLK-4, CAT-REV-1, ENUM-2, PLAN-INV-8, POL-4, POL-5, POOL-3 |
| 13 | FLT-1, FLT-2, FLT-3, FLT-4, FLT-5, FLT-6, FLT-7, PERS-4 |
| 14 | BUILD-PRE-7, DW-NEW-PRE-1, DW-Q-2, END-PRE-1, EXE-PRE-2, HN-INV-3, INC-INV-2, LS-INV-5, LS-INV-6, LS-Q-2, PLAN-INV-4, PLC-INV-8, PRM-PRE-2, PRM-PRE-4, RU-1, RU-3 |

### `[DISC]` — merchandising-space design discussion

BAY-INV-1, BAY-INV-2, BAY-INV-3, BAY-INV-4, BAY-NEW-PRE-1, BAY-NEW-PRE-2, BAY-NEW-PRE-3, BIN-INV-1, BIN-NEW-PRE-1, DCR-INV-1, DCR-NEW-PRE-1, DCR-NEW-PRE-2, DOR-INV-1, DOR-INV-2, DOR-INV-3, DOR-INV-4, DOR-NEW-PRE-1, DOR-NEW-PRE-2, ENUM-6, FD-INV-1, FD-INV-3, FD-NEW-PRE-1, GDL-INV-1, GDL-INV-2, GDL-INV-4, GDL-INV-5, GDL-NEW-PRE-1, GDL-NEW-PRE-2, GDL-NEW-PRE-3, GEN-3, ID-1, LS-INV-1, LS-INV-2, LS-INV-10, LS-INV-11, RUN-INV-1, RUN-INV-2, RUN-NEW-PRE-1, RUN-NEW-PRE-2, SHR-INV-1, SHR-NEW-PRE-1, SHR-NEW-PRE-2, SID-INV-1, SID-INV-2, SID-NEW-PRE-1, SID-NEW-PRE-2, SN-INV-3, SN-INV-4, SN-Q-1, SN-Q-2, SN-Q-3, ST-Q-3, SVC-INV-1, SVC-INV-2, SVC-NEW-PRE-1, SVC-NEW-PRE-2, TBL-INV-1, TBL-INV-2, TBL-NEW-PRE-1, TBL-NEW-PRE-2, TIER-INV-1, TIER-INV-2, TIER-NEW-PRE-1, TIER-NEW-PRE-2

### `[SIM]` — simulation-only decisions

ADDM-1, ADDM-PRE-1, AGC-1, AGR-NEW-PRE-1, AREA-INV-2, AREA-INV-3, ASN-1, ASN-PRE-2, ASN-PRE-3, BAY-INV-3, BAY-INV-4, BIN-INV-2, BU-INV-1, BU-INV-2, BU-INV-3, BUILD-3, BUILD-PRE-2, BUILD-PRE-6, CAN-1, CAN-PRE-1, CAT-INV-1, CAT-INV-6, CAT-INV-7, CAT-INV-10, CAT-NEW-PRE-3, CAT-Q-1, CAT-Q-6, CLP-INV-2, DOR-INV-4, DW-NEW-PRE-1, DW-Q-1, ENUM-2, ENUM-4, ENUM-5, ENUM-6, EXE-6, FD-INV-2, FIT-2, FRM-1, FRM-2, GDL-INV-3, GDL-INV-4, GDL-INV-5, GDL-NEW-PRE-3, GEN-4, GEN-9, GEN-12, GEN-13, ID-1, ID-2, ID-3, ID-4, INC-INV-5, INC-NEW-PRE-1, LS-INV-11, LS-Q-4, MON-1, MON-3, MON-4, MON-5, MON-6, MON-7, MON-8, MON-9, NBS-NEW-PRE-1, PERS-3, PLAN-INV-3, PLAN-INV-6, PLC-INV-4, PLC-NEW-PRE-1, POL-2, POL-PRE-1, POOL-1, POOL-2, POOL-3, POOL-4, PRM-PRE-5, PROD-INV-2, PROD-NEW-PRE-2, PROD-Q-1, REL-PRE-4, RUN-INV-2, RUN-NEW-PRE-2, SKU-INV-9, SKU-INV-11, SKU-NEW-PRE-1, SKU-NEW-PRE-3, SKU-Q-1, SN-INV-1, SN-INV-2, SN-INV-4, SN-Q-6, ST-INV-3, ST-INV-7, ST-INV-9, ST-PRE-3, ST-Q-2, ST-Q-10, SVC-INV-2, TBL-INV-2, TIER-INV-2, TIER-NEW-PRE-2, TYP-2, TYP-3, UNA-1, UNA-PRE-1, UNA-PRE-2, WRK-PRE-1

### `[DbC]` — Design by Contract method

AGC-2, AGR-NEW-1, AREA-NEW-1, AREA-NEW-PRE-2, ASN-2, BAY-NEW-1, BAY-NEW-PRE-2, BIN-NEW-1, BU-NEW-1, BU-NEW-PRE-2, BUILD-1, BUILD-2, BUILD-PRE-1, CAN-2, CAT-NEW-1, CAT-NEW-PRE-2, CLP-NEW-1, DCR-NEW-1, DCR-NEW-PRE-2, DOR-NEW-1, DOR-NEW-PRE-2, DW-NEW-1, END-2, EXE-2, EXE-PRE-3, FD-NEW-1, FLT-PRE-1, FRM-1, FRM-2, FRM-3, GDL-NEW-1, GDL-NEW-PRE-2, GEN-10, GEN-PRE-1, GEN-PRE-2, HN-INV-1, HN-INV-2, HN-INV-4, HN-Q-1, HN-Q-2, HN-Q-3, HN-Q-4, HN-Q-5, HN-Q-6, INC-NEW-1, INP-2, LS-INV-4, MON-1, MON-2, MON-3, MON-4, MON-5, MON-6, MON-7, MON-8, MON-9, MON-10, MVI-2, NBS-NEW-1, PEG-NEW-1, PGM-NEW-1, PGV-NEW-1, PLAN-INV-1, PLAN-INV-2, PLAN-INV-5, PLAN-NEW-1, PLAN-NEW-PRE-1, PLAN-Q-1, PLC-INV-13, PLC-NEW-1, PLC-Q-2, POL-NEW-1, PP-NEW-1, PRM-2, PROD-NEW-1, REL-2, RUN-NEW-1, SHR-NEW-1, SHR-NEW-PRE-2, SID-NEW-1, SID-NEW-PRE-2, SKU-INV-8, SKU-NEW-1, ST-INV-4, ST-INV-5, ST-INV-8, ST-PRE-1, ST-PRE-2, ST-Q-1, ST-Q-11, SVC-NEW-1, SVC-NEW-PRE-2, TBL-NEW-1, TBL-NEW-PRE-2, TIER-NEW-1, UNA-2, WRK-2
