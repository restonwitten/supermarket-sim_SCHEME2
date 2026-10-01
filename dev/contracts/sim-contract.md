# Supermarket Operations Simulation — Design-by-Contract Specification

**Contract version:** 0.2 (Merchandising Space Allocation → SKU Placements)
**Date:** 2026-09-29
**Supersedes:** v0.1 (same date), which was written without the app-functionality file. Part 11 lists every v0.1 clause that was retired or changed.
**Status:** Draft for review. Companion file: `sim-contract-discussion.md` (textbook-vs-functionality conflicts, conflicts inside the functionality discussion, and open questions).

**Sources this contract is derived from (and only these):**

| Tag | Source |
|---|---|
| `[TB x.y]` | *How a Modern Supermarket Works: Operations Textbook* (`modern-supermarket-operations.md`), chapter.section |
| `[VS n]` | *Voice Session Extract — Sim App Design Points* (`voice-session-sim-design-extract.md`, Sept 27 2026), section n. This is the app-functionality file. |
| `[DISC]` | The merchandising-space design discussion recorded in project memory (`areas/merchandising-space-model.md`): sheet-per-space-type, no inheritance of attribute shape, Protocol for tree behavior, the Gondola → Side → Bay → Shelf Run composition, and the five fixture archetypes |
| `[DbC]` | Bertrand Meyer, *Object-Oriented Software Construction*, 2nd ed., and the Design by Contract method (assertions, invariants, command/query separation, subcontracting) |
| `[SIM]` | A simulation-only decision made in this contract, with no textbook or discussion grounding. Every `[SIM]` item that needs your confirmation is listed in the companion file. |

**Workbook.** Workbook column names are used only in the companion file's cross-workstream flags. The workbook is not a source for this contract. The extract's four unverified workbook claims were checked separately (companion file, Part D).

---

## Part 0 — How to read and use this contract

### 0.1 Purpose and authority

This document is the **sole specification** for the Python classes of the simulation's data model. A class, feature, or behavior not specified here is not part of the model. An implementation is correct when, and only when, it satisfies every clause below for every call a correct client can make. `[DbC]`

The document must stand on its own: a fresh session with no other context should be able to build the classes from it. `[VS 14]`

The contract grows section by section. Part 10 lists the areas reserved for later versions. **Evolution rule (Open–Closed):** later versions may add classes, features, and clauses. They may not weaken an existing postcondition or invariant, or strengthen an existing precondition, without a version bump that names the broken clause. Existing clause IDs are never reused; that rule starts at v0.2 (Part 11.1). `[DbC]`

### 0.2 Assertion vocabulary `[DbC]`

| Term | Meaning | Whose bug if violated |
|---|---|---|
| **require** (precondition) | What the client must make true before calling | The client (caller) |
| **ensure** (postcondition) | What the supplier guarantees on return, given the precondition held | The supplier (implementer) |
| **invariant** | What is true of every instance at every *stable time*: after creation and before/after every exported call. It may be temporarily false inside a routine of the class | The supplier |
| `old(e)` | The value of expression `e` on entry to the routine | — |
| `result` | The value a query returns | — |

Clauses are written as **Python boolean expressions** over the class's queries, so each one can be implemented as a runtime check. `∀`/`∃` are written as `all(...)`/`any(...)`. Every clause has a stable ID (for example `PLC-INV-5`) for traceability from code, tests, and discussion.

### 0.3 Design rules the implementation must follow

1. **Command–Query Separation.** A *query* returns information and has no observable side effect. A *command* changes state and returns nothing, except where a command is specified to return a value. Every expression used in an assertion must use only queries. `[DbC]`
2. **Uniform Access.** A client can't tell whether a query is stored or computed. Derived facts (a placement's shelf level, its facing count, its business unit, whether it is vendor-funded) are specified as queries and must **not** be stored redundantly. `[DbC]` `[VS 8]`
3. **Preconditions are not defensive checks.** The supplier assumes the precondition. It does not quietly "handle" a violation with a fallback. When assertion monitoring is on (0.4), a violation raises `PreconditionViolation`. `[DbC]`
4. **Expected outcomes are not exceptions.** Something the domain expects, like a SKU that doesn't fit in its category's space, is reported in the result (for example `PlacementPlan.unplaced`). It is never raised as an exception. Exceptions are reserved for contract violations. `[DbC]`
5. **Data shape vs. behavior.** Each space type is its own self-contained class with its own attributes. There is **no shared base class for attribute shape**. Shared *behavior* (tree navigation and rollup) is a **deferred class**, implemented in Python as a `typing.Protocol`. Every effective class that conforms to a deferred class inherits all of its contracts. `[DISC]` `[DbC]`
6. **Subcontracting.** A class conforming to a deferred class may only *weaken* inherited preconditions and *strengthen* inherited postconditions and invariants. `[DbC]`
7. **One hierarchy shape, implemented once.** The space, space-type, business-unit, and product-category hierarchies all conform to the same deferred class `HierarchyNode` (2.1). Their tree behavior is written once. `[VS 6]`
8. **Geometry and economics are separate layers.** No space class has a price, fee, or premium attribute. Money attaches to `Agreement`s, and an agreement attaches to a placement, never to a space. `[VS 7]` `[VS 3]`
9. **Filters return sets.** Every filter query returns a `frozenset`. Combining conditions is set algebra: "and" is intersection, "or" is union, "but not" is difference. There is no query language. `[VS 13]`
10. **Intervals are half-open.** A date interval `[start, end)` contains `d` iff `start <= d and (end is None or d < end)`. `end is None` means open-ended. A placement replaced at a reset on date `R` ends at `R`, and its successor starts at `R`. `[VS 4]` `[SIM]` (half-open)

### 0.4 Assertion monitoring (implementation requirement) `[DbC]` `[SIM]`

The implementation must provide a runtime-selectable monitoring level, as in Eiffel: `NONE`, `REQUIRE`, `ENSURE` (includes REQUIRE), `INVARIANT` (includes ENSURE), `ALL`. Violations raise `PreconditionViolation`, `PostconditionViolation`, or `InvariantViolation`, each carrying the clause ID. The default level for development and tests is `ALL`.

### 0.5 Units and time `[TB 1.2, 2.3]` `[VS 8]` `[SIM]`

- Linear dimensions are in **inches** (`_in`). Feet appear only in reporting queries (`_ft`).
- Floor and deck area is in **square feet** (`_sqft`).
- Hook positions are counted in **hooks**.
- Facings are non-negative **integers**, and always derived (see PLC-INV-4).
- Money is in **dollars** as a `float`, rounded to cents only for display.
- Time enters the model only through explicit `on: date` / `as_of: date` arguments. There is no simulation clock yet (Part 10).
- Floating-point comparisons use a tolerance `EPS = 1e-6`. Footprint reconciliation uses `FOOTPRINT_TOL_SQFT = 1.0`.

---

## Part 1 — Enumerations and value types

| Enumeration | Values | Grounding |
|---|---|---|
| `SpaceUnit` | `LINEAR_IN`, `SQFT`, `HOOKS` | `[VS 8]` linear vs. area; `[VS 7]` hooks |
| `PresentationPlane` | `FRONT`, `TOP`, `PEG` | `[VS 8]`; `PEG` `[VS 7]` |
| `ShelfLevel` | `TOP`, `EYE_LEVEL`, `MIDDLE`, `BOTTOM` | `[DISC]` (4 levels); eye level `[TB 2.3]` |
| `TemperatureZone` | `AMBIENT`, `REFRIGERATED`, `FROZEN` | `[TB 1.3, 4.3, 4.4]` |
| `ProductForm` | `BOX`, `SOFT_PACK`, `HANGING`, `LOOSE` | `[VS 8]` (non-box products need overrides); values `[SIM]` |
| `PlacementType` | `PERMANENT`, `PROMOTIONAL`, `INCIDENTAL` | `[VS 3]` |
| `AgreementType` | `SLOTTING`, `PROMOTIONAL_DISPLAY`, `PAY_TO_STAY`, `OPPORTUNISTIC_BUY` | `[VS 3]` |
| `FeeBasis` | `PER_FACING`, `PER_LINEAR_FT`, `PER_DISPLAY`, `FLAT` | `[VS 3]` (priced by facings or linear feet; per display per store); `FLAT` `[SIM]` |
| `CategoryRole` | `DESTINATION`, `ROUTINE`, `OCCASIONAL_SEASONAL`, `CONVENIENCE` | `[TB 2.1]` |
| `AssortmentStatus` | `CORE`, `NEW_ITEM_TRIAL`, `DELIST_CANDIDATE`, `DELISTED` | `[TB 2.6]`; `DELISTED` `[SIM]` |
| `BlockingMode` | `VERTICAL_BRAND`, `HORIZONTAL_TIER` | `[VS 12]` |
| `VersionKind` | `CYCLE`, `REVISION` | `[VS 10]` |
| `VersionStatus` | `RELEASED`, `EXECUTED` | `[VS 10]` |
| `UnplacedReason` | `NO_CATEGORY_SPACE`, `NO_SPACE_IN_PLANE`, `NO_DIMENSIONAL_FIT`, `INSUFFICIENT_SPACE`, `OUT_OF_SEASON` | `[SIM]` |

**ENUM-1.** `plane_unit(plane)` is a total function: `FRONT → LINEAR_IN`, `TOP → SQFT`, `PEG → HOOKS`. `[VS 8]` `[VS 7]`

**ENUM-2.** `ShelfLevel.preference_rank`: `EYE_LEVEL`=1, `MIDDLE`=2, `TOP`=3, `BOTTOM`=4 (lower is better). Eye level and the level just below it carry the premium `[VS 12]`. `TOP` vs. `BOTTOM` order is `[SIM]`.

**ENUM-3.** `compatible(agreement_type, placement_type)` holds exactly for these pairs `[VS 3]`:

| Agreement type | Placement type |
|---|---|
| `SLOTTING` | `PERMANENT` |
| `PAY_TO_STAY` | `PERMANENT` |
| `PROMOTIONAL_DISPLAY` | `PROMOTIONAL` |
| `OPPORTUNISTIC_BUY` | `INCIDENTAL` |

Placement type and agreement type are related but **separate fields**. `[VS 3]`

**ENUM-4.** `permitted_planes(product)` is defined in 4.1 (PROD-Q-1). `permitted_forms(plane)`: `FRONT → {BOX, SOFT_PACK}`; `TOP → {BOX, SOFT_PACK, LOOSE}`; `PEG → {HANGING}`. `[SIM]` (Q-6)

**Value types.** `DateWindow(start: date, end: date | None)` with `contains(d)` and `overlaps(other)` per rule 10. `overlaps` is true iff the two intervals share at least one date.

---

## Part 2 — The shared hierarchy shape and rollups `[VS 6, 9]`

The model has **four** recursive hierarchies of arbitrary depth: physical **space**, **space type**, **business unit**, and product **category**. All four conform to `HierarchyNode`. The physical hierarchy says where things sit and nest. The type hierarchy says what kind of thing each one is and which attributes apply. The two are crossed. `[VS 6]`

### 2.1 Deferred class `HierarchyNode`

```
deferred class HierarchyNode
queries
    id: str                                   -- unique within its hierarchy
    parent: HierarchyNode | None              -- None only for a root
    children: tuple[HierarchyNode, ...]       -- ordered
    is_leaf: bool
    path: tuple[str, ...]                     -- ids from the root down to self, inclusive
    depth: int                                -- len(path) - 1
    ancestors() -> tuple[HierarchyNode, ...]  -- parent, grandparent, ..., root
    subtree() -> frozenset[HierarchyNode]     -- self and all descendants
    leaves() -> tuple[HierarchyNode, ...]     -- leaf descendants in child order; (self,) if leaf
    is_within(other: HierarchyNode) -> bool   -- self is other or a descendant of other
```

| ID | Clause |
|---|---|
| HN-INV-1 | `self.is_leaf == (len(self.children) == 0)` |
| HN-INV-2 | `all(c.parent is self for c in self.children)` and `self.parent is None or self in self.parent.children` |
| HN-INV-3 | `self not in self.ancestors()` — acyclic |
| HN-INV-4 | ids are unique within one hierarchy |
| HN-Q-1 | `path`: `result == tuple(a.id for a in reversed(self.ancestors())) + (self.id,)` |
| HN-Q-2 | `ancestors()`: `result == () if self.parent is None else (self.parent,) + self.parent.ancestors()` |
| HN-Q-3 | `subtree()`: `result == frozenset({self}).union(*(c.subtree() for c in self.children))` |
| HN-Q-4 | `leaves()`: `result == (self,) if self.is_leaf else tuple(l for c in self.children for l in c.leaves())` |
| HN-Q-5 | `is_within(o)`: `result == (o.id in self.path)` |
| HN-IMPL-1 | **Non-functional requirement (verified by review, not by a runtime assertion).** `subtree()` membership and "all nodes under X" must be answerable by an indexed lookup on `path` (path enumeration) or a closure table, not by a per-call tree walk. `[VS 6]` |

Hierarchies are **structurally static** after construction in this version: no command re-parents, adds, or removes a node. `[SIM]` (Q-14)

### 2.2 The space-type hierarchy `[VS 6]`

`SpaceType` is a node of a fixed tree. Every space node has exactly one `space_type`. `merchandisable` is derived from the type.

```
SPACE
├─ SALES_FLOOR                          (container; not itself MERCHANDISING)
├─ MERCHANDISING                        (merchandisable = True)
│   ├─ OPEN_SHELVING     ← GONDOLA, WALL_SHELF, BAKERY_SHELVING, FLORAL_SHELVING
│   ├─ DOOR_CASE         ← REACH_IN_COOLER, REACH_IN_FREEZER, FLORAL_COOLER
│   ├─ SERVICE_CASE      ← DELI_CASE, SERVICE_REFRIGERATED_CASE
│   ├─ PRODUCE_TABLE
│   ├─ FLOOR_DISPLAY
│   ├─ PEG_SECTION
│   └─ CLIP_STRIP
└─ NON_MERCHANDISING                    (merchandisable = False)
    └─ BACKROOM, PREP_AREA, CHECKOUT, AISLE, RESTROOM, OFFICE, OTHER
```

The three interior nodes under `MERCHANDISING` (`OPEN_SHELVING`, `DOOR_CASE`, `SERVICE_CASE`), plus `PRODUCE_TABLE` and `FLOOR_DISPLAY`, are the five physical archetypes from `[DISC]`. `PEG_SECTION` and `CLIP_STRIP` are two new archetypes. `[VS 7]`

| ID | Clause |
|---|---|
| TYP-1 | `merchandisable(t) == t.is_within(MERCHANDISING)`, and it is a total function on `SpaceType` |
| TYP-2 | `temperature_zone(t)` is total: `REACH_IN_COOLER`, `FLORAL_COOLER`, `DELI_CASE`, `SERVICE_REFRIGERATED_CASE` → `REFRIGERATED`; `REACH_IN_FREEZER` → `FROZEN`; every other type → `AMBIENT`. `[TB 4.4]`; `PRODUCE_TABLE` = `AMBIENT` is `[SIM]` (Q-11) |
| TYP-3 | A checkout **impulse rack** is a `FLOOR_DISPLAY` whose parent is an `Area` of type `CHECKOUT`. The `CHECKOUT` area itself is non-merchandising. `[VS 6]` `[SIM]` (Q-16) |

### 2.3 Deferred class `Rollup` and the measures `[VS 9, 14]`

Every non-leaf node of the space, category, and business-unit hierarchies aggregates its children. The same machinery serves all three ("the cube"): three views over the same leaf facts, which are the placements.

```
deferred class Rollup
queries
    capacity(u: SpaceUnit) -> float
    allocated_amount(u: SpaceUnit, on: date) -> float
    unallocated_amount(u: SpaceUnit, on: date) -> float
    facings(on: date) -> int
    units_capacity(on: date) -> int
    placement_count(on: date) -> int
```

**Two families of measure.** The definitions differ by hierarchy, and they are additive in every hierarchy.

| Family | Measures | Space node `n` | Category node `n` | Business unit `n` |
|---|---|---|---|---|
| **Space measures** | `capacity(u)`, `allocated_amount(u, on)`, `unallocated_amount(u, on)` | over `n`'s allocatable leaves | over leaves assigned to any node in `n.subtree()` | sum over the unit's descendants (a `DEPARTMENT` unit counts its root category node) |
| **Placement measures** | `facings(on)`, `units_capacity(on)`, `placement_count(on)` | over committed placements live on `on` in `n`'s leaves | over committed placements live on `on` whose SKU's category is in `n.subtree()` | as for space measures |

Only committed placements (`Placement`, 5.2) count toward these measures. Incidental placements (5.3) have no space commitment and are excluded. `[VS 3]`

| ID | Clause |
|---|---|
| RU-1 | **Additivity.** For every non-leaf node `n` and every additive measure `m` (`capacity(u)`, `allocated_amount(u, on)`, `facings(on)`, `units_capacity(on)`, `placement_count(on)`): `m(n) == own_m(n) + sum(m(c) for c in n.children)`, where `own_m(n)` is the contribution attached to `n` itself. `own_m` is `0` for every space node that is not a leaf; for a category node it is the measure over the leaves assigned exactly to `n` (space family) or over placements of SKUs whose category is exactly `n` (placement family); for a business unit it is `0`, except a `DEPARTMENT` unit, whose own contribution is its root category's subtree total. A rollup asserts that children sum to the parent. `[VS 14]` |
| RU-2 | `unallocated_amount(u, on) == capacity(u) - allocated_amount(u, on)` |
| RU-3 | `0 <= allocated_amount(u, on) <= capacity(u)` for every `u` and every `on` |
| RU-4 | **No mixed units.** No rollup adds quantities of different `SpaceUnit`s. Every space measure takes the unit as an argument. Cross-plane comparison uses the placement measures (`facings`, `units_capacity`), which are unit-free. `[VS 8]` |

**Reserved measures.** Revenue, gross margin, and sales per linear foot attach at the placement and must be additive under RU-1. Their definitions belong to Section 6 (Part 10). `[VS 9]`

---

## Part 3 — Physical space

### 3.1 Deferred class `SpaceNode` (conforms to `HierarchyNode` and `Rollup`) `[DISC]` `[VS 6]`

Every physical space class, `Area`, and `Store` conforms to `SpaceNode`. The physical hierarchy covers the **whole building footprint**, not only merchandising space, so unallocated merchandising space is a filter and the total still reconciles to the building. `[VS 6]`

```
deferred class SpaceNode
queries
    space_type: SpaceType
    merchandisable: bool                          -- == TYP-1(space_type)
    footprint_sqft: float | None                  -- Store, Area, and top-level fixtures; None for fixture parts
    temperature_zone: TemperatureZone             -- == TYP-2(space_type)
    own_assignment: CategoryNode | None           -- assigned on this node itself
    assigned_category: CategoryNode | None        -- effective: nearest ancestor-or-self with an own_assignment
    allocatable_leaves() -> tuple[LeafSpace, ...]
    placements_on(on: date) -> frozenset[Placement]
```

| ID | Clause |
|---|---|
| SN-Q-1 | `allocatable_leaves()`: `result == tuple(l for l in self.leaves() if isinstance(l, LeafSpace))` |
| SN-Q-2 | `capacity(u)`: `result == sum(l.leaf_capacity for l in self.allocatable_leaves() if l.unit == u)` |
| SN-Q-3 | `allocated_amount(u, on)`: `result == sum(l.leaf_allocated(on) for l in self.allocatable_leaves() if l.unit == u)` |
| SN-Q-4 | `placements_on(on)`: `result == frozenset(p for l in self.allocatable_leaves() for p in l.placements if p.window.contains(on))` |
| SN-Q-5 | `facings(on) == sum(p.facings for p in placements_on(on))`; `units_capacity(on) == sum(p.units_capacity for p in placements_on(on))`; `placement_count(on) == len(placements_on(on))` |
| SN-Q-6 | `assigned_category`: `result is self.own_assignment if self.own_assignment is not None else (None if self.parent is None else self.parent.assigned_category)` |
| SN-INV-1 | `self.merchandisable or len(self.allocatable_leaves()) == 0` — non-merchandising space holds no allocatable leaf |
| SN-INV-2 | `self.own_assignment is None or (self.merchandisable and not any(a.own_assignment is not None for a in self.ancestors()) and not any(d.own_assignment is not None for d in self.subtree() if d is not self))` — at most one assignment on any root-to-leaf path |
| SN-INV-3 | `self.parent is None or self.parent.space_type is an Area type or self.space_type == self.parent.space_type or self.space_type in {PEG_SECTION, CLIP_STRIP}` — a fixture's parts share its type. Only peg sections and clip strips hang off a face of a different type `[VS 7]` |
| SN-INV-4 | node ids are globally unique across the space hierarchy, and each id carries its class's prefix `[DISC]` |

RU-1 through RU-3 apply to `SpaceNode`. Both faces of a fixture are separate `GondolaSide` nodes and are allocated separately: two faces, two aisles, possibly two categories. `[VS 7]`

---

### 3.2 Deferred class `LeafSpace` (conforms to `SpaceNode`) — the allocatable unit `[DISC]` `[VS 7, 8]`

A leaf is the smallest unit a committed placement can occupy: Shelf Run (open shelving, door case), Tier (service case), Bin (produce table), Floor Display, Peg Section, or Clip Strip. The linear position of a placement on a leaf is a **segment** of the leaf, described by `offset_in`. A segment is not a node. `[VS 7]`

```
deferred class LeafSpace
queries
    presentation_plane: PresentationPlane         -- which surface the shopper sees; a property of the SPACE
    unit: SpaceUnit                               -- == plane_unit(presentation_plane)
    leaf_capacity: float                          -- in `unit`
    placements: tuple[Placement, ...]             -- ALL committed placements ever made here, incl. ended, ordered by (start_date, offset_in)
    secured: bool                                 -- locked/secured case [TB 6.4]
    shelf_level: ShelfLevel | None                -- None for Bin, FloorDisplay, PegSection, ClipStrip
    usable_depth_in: float | None
    clear_height_in: float | None
    hook_depth_units: int | None                  -- PEG plane only
    respace_without_reset: bool                   -- True where moving a placement is seconds, not a reset [VS 7]
    leaf_allocated(on: date) -> float
    peak_allocated(window: DateWindow) -> float
    free_intervals(window: DateWindow) -> tuple[tuple[float, float], ...]   -- LINEAR_IN leaves only
    can_hold(product: Product, extent: float, window: DateWindow) -> bool
    block_group() -> SpaceNode                    -- see 3.6
    bay_position() -> int                         -- see 3.6
```

| ID | Clause |
|---|---|
| LS-INV-1 | `self.is_leaf` (strengthens `HierarchyNode`) |
| LS-INV-2 | `self.leaf_capacity > 0` |
| LS-INV-3 | `self.unit == plane_unit(self.presentation_plane)` (ENUM-1) |
| LS-INV-4 | `all(p.space is self for p in self.placements)` |
| LS-INV-5 | **No over-subscription.** For every date `d` in `Store.change_dates()`: `self.leaf_allocated(d) <= self.leaf_capacity + EPS`. Checking change dates suffices, because allocation is piecewise constant between them. `[VS 14]` |
| LS-INV-6 | **No overlap in space and time.** For `LINEAR_IN` leaves: for every pair `p != q` in `placements` with `p.window.overlaps(q.window)`, the intervals `[p.offset_in, p.offset_in + p.extent)` and `[q.offset_in, q.offset_in + q.extent)` are disjoint. Two placements may not overlap the same shelf segment on the same date. `[VS 14]` `[TB 2.2]` |
| LS-INV-7 | `self.temperature_zone == TYP-2(self.space_type)`; and `self.assigned_category is None or self.temperature_zone == self.assigned_category.policy().temperature_zone` `[TB 1.3, 4.4]` |
| LS-INV-8 | `self.assigned_category is None or not self.assigned_category.policy().secured_case_required or self.secured` `[TB 6.4]` |
| LS-INV-9 | `self.presentation_plane == FRONT ⇒ usable_depth_in is not None and clear_height_in is not None`; `self.presentation_plane == PEG ⇒ hook_depth_units is not None and hook_depth_units >= 1 and usable_depth_in is None and clear_height_in is None` |
| LS-INV-10 | `(self.shelf_level is None) or self.presentation_plane == FRONT` — only front-plane leaves carry a shelf level |
| LS-Q-1 | `leaf_allocated(on)`: `result == sum(p.extent for p in self.placements if p.window.contains(on))` |
| LS-Q-2 | `peak_allocated(w)`: `result == max(self.leaf_allocated(d) for d in {w.start} | {p.start_date for p in self.placements if w.contains(p.start_date)})` |
| LS-Q-3 | `free_intervals(w)`: require `self.unit == LINEAR_IN`. Ensure `result` is the ordered, maximal list of gaps in `[0, leaf_capacity]` not covered by the interval of any placement whose window overlaps `w`. |
| LS-Q-4 | `can_hold(prod, x, w)`: require `x > 0`. Ensure `result == (fits_dimensionally(prod, self) and x <= room)`, where `room` is `max((b - a for a, b in free_intervals(w)), default=0)` for `LINEAR_IN` leaves and `leaf_capacity - peak_allocated(w)` otherwise |

#### Fit and extent arithmetic `[VS 8]`

Product dimensions X (width), Y (height), Z (depth) are the consumer-unit **bounding box as merchandised** (4.1). The plane is a property of the space. The facing count is a property of the placement. The plane decides which product dimensions are relevant. `[VS 8]`

| ID | Clause |
|---|---|
| FIT-1 | `fits_dimensionally(prod, leaf) ==` `leaf.presentation_plane in prod.permitted_planes()` **and** (`FRONT`: `prod.height_in <= leaf.clear_height_in and prod.depth_in <= leaf.usable_depth_in`; `TOP`: `(leaf.clear_height_in is None or prod.height_in <= leaf.clear_height_in) and (leaf.usable_depth_in is None or prod.depth_in <= leaf.usable_depth_in)`; `PEG`: `True`) |
| FIT-2 | `facing_extent(prod, unit)` is defined by PROD-Q-2. A placement's `facings` is `floor(extent / facing_extent(prod, leaf.unit) + EPS)` (PLC-Q-1) |

---

### 3.3 `Area` — non-fixture space, and footprint reconciliation `[VS 6]`

| Class | Prefix | Own attributes | Children |
|---|---|---|---|
| `Area` | `AREA-` | `name: str`, `footprint_sqft: float`, `space_type` (`SALES_FLOOR` or a `NON_MERCHANDISING` type) | `Area`s and top-level fixtures |

| ID | Clause |
|---|---|
| AREA-INV-1 | `footprint_sqft > 0` |
| AREA-INV-2 | **Reconciliation.** If `children` is non-empty and every child has a non-`None` footprint: `abs(self.footprint_sqft - sum(c.footprint_sqft for c in children)) <= FOOTPRINT_TOL_SQFT`. Circulation is an `Area` of type `AISLE`, so a floor accounts for all of its square feet. `[VS 6]` `[SIM]` (Q-17) |
| AREA-INV-3 | Any merchandising fixture among `children` has an `Area` parent of type `SALES_FLOOR` or `CHECKOUT` (TYP-3) |

`Store` also satisfies AREA-INV-2 for the building footprint (7.1).

---

### 3.4 Open shelving: Gondola, End Caps, Peg Sections, Clip Strips `[DISC]` `[VS 7]`

A Gondola is bordered by four aisles, named from a viewer at the front of the store facing the rear: Front, Back, Left, Right. Its **faces** are `GondolaSide`s. `FRONT` and `BACK` faces run the length of the gondola. `LEFT` and `RIGHT` faces are the **end caps**: the same fixture, but managed as a separate, more valuable unit. The added value is carried by agreements, not by the space (rule 8). `[VS 7]`

| Class | Prefix | Own attributes | Children |
|---|---|---|---|
| `Gondola` | `GDL-` | `front_aisle, back_aisle, left_aisle, right_aisle: str`, `depth_in: float`, `footprint_sqft: float` | 1–4 `GondolaSide` |
| `GondolaSide` | `SID-` | `facing: Literal["FRONT","BACK","LEFT","RIGHT"]`, `aisle_faced: str` | ≥1 `Bay`; 0+ `PegSection`; 0+ `ClipStrip` |
| `Bay` | `BAY-` | `position: int` (1-based, left→right as viewed from the aisle), `width_in: float` | 4 `ShelfRun` |
| `ShelfRun` (leaf) | `RUN-` | `shelf_level: ShelfLevel`, `width_in, depth_in, clear_height_in: float`, `secured: bool` | — |
| `PegSection` (leaf) | `PEG-` | `hook_count: int`, `hook_depth_units: int`, `location_description: str` | — |
| `ClipStrip` (leaf) | `CLP-` | `hook_count: int`, `hook_depth_units: int`, `frontage_in: float` | — |

`Gondola.space_type == GONDOLA`, `PegSection.space_type == PEG_SECTION`, `ClipStrip.space_type == CLIP_STRIP` (allowed by SN-INV-3).

| ID | Clause |
|---|---|
| GDL-INV-1 | `1 <= len(children) <= 4` and the sides' `facing` values are distinct |
| GDL-INV-2 | `all(s.aisle_faced == {"FRONT": front_aisle, "BACK": back_aisle, "LEFT": left_aisle, "RIGHT": right_aisle}[s.facing] for s in children)` |
| GDL-INV-3 | If both a `FRONT` and a `BACK` side exist, they have the same number of bays and the same bay widths, position by position. `[SIM]` physical assumption (Q-8) |
| GDL-INV-4 | Each end face's total bay width is `<= depth_in` |
| GDL-INV-5 | If a `FRONT` or `BACK` side exists, `abs(footprint_sqft - length_in * depth_in / 144) <= 0.01`, where `length_in` is the total bay width of the `FRONT` side (or of the `BACK` side if there is no `FRONT`) |
| SID-INV-1 | The positions of the `Bay` children are exactly `1..k` |
| SID-INV-2 | The side has at least one `Bay` |
| BAY-INV-1 | `len(children) == 4` and `{r.shelf_level for r in children} == set(ShelfLevel)` `[DISC]` |
| BAY-INV-2 | `all(r.width_in == self.width_in for r in children)` — a shelf's length is fixed by its fixture `[DISC]` |
| RUN-INV-1 | `presentation_plane == FRONT and leaf_capacity == width_in and usable_depth_in == depth_in and respace_without_reset is False` |
| PEG-INV-1 | `presentation_plane == PEG and leaf_capacity == hook_count and shelf_level is None and respace_without_reset is True`. Only the section and its hook count are modeled, not individual hook coordinates. `[VS 7]` |
| PEG-INV-2 | `hook_count >= 1 and hook_depth_units >= 1` — depth is how many units deep a hook holds `[VS 7]` |
| CLP-INV-1 | Same as PEG-INV-1 and PEG-INV-2, for a `ClipStrip`. Its parent is a `GondolaSide`. |
| CLP-INV-2 | `0 < frontage_in <= min(b.width_in for b in parent.children if isinstance(b, Bay))`. The strip consumes a little of its face's frontage. The frontage is recorded but not deducted from any run's capacity. `[VS 7]` `[SIM]` (Q-15) |

Mounting-hardware compatibility (which hook fits which board) is not modeled. `[VS 7]`

### 3.5 Other open shelving, door cases, service cases, produce tables, floor displays `[DISC]`

**Non-gondola open shelving (Wall Shelf, Bakery Shelving, Floral Shelving).** Same shape as a gondola face, without the gondola layer: `ShelvingRun → Bay → ShelfRun`. `Bay`, `ShelfRun`, and all their invariants are the classes above.

| Class | Prefix | Own attributes | Children |
|---|---|---|---|
| `ShelvingRun` | `WS-RUN-`, `BKY-RUN-`, `FLS-RUN-` | `location_description: str`, `footprint_sqft: float` | ≥1 `Bay`; 0+ `PegSection` |

`space_type ∈ {WALL_SHELF, BAKERY_SHELVING, FLORAL_SHELVING}`.

**Door cases (Reach-In Cooler, Reach-In Freezer, Floral Cooler).**

| Class | Prefix | Own attributes | Children |
|---|---|---|---|
| `DoorCaseRun` | `RIC-RUN-`, `RIF-RUN-`, `FLC-RUN-` | `location_description: str`, `footprint_sqft: float` | ≥1 `Door` |
| `Door` | `…-DOR-` | `number: int`, `width_in: float` | 4 `ShelfRun` |

| ID | Clause |
|---|---|
| DOR-INV-1 | `len(children) == 4` and `{r.shelf_level for r in children} == set(ShelfLevel)` |
| DOR-INV-2 | `all(r.width_in == self.width_in for r in children)` |
| DOR-INV-3 | Door `number` values within one `DoorCaseRun` are exactly `1..n` |
| DCR-INV-1 | A `DoorCaseRun` has at least one `Door` |

**Service cases (Deli Case, Service/Self-Service Refrigerated Case).** One fixed-length case with tiers and no bay layer.

| Class | Prefix | Own attributes | Children |
|---|---|---|---|
| `ServiceCase` | `DELI-CASE-`, `SVC-CASE-` | `location_description: str`, `length_in: float`, `footprint_sqft: float` | 1–4 `Tier` |
| `Tier` (leaf) | `…-TIER-` | `shelf_level: ShelfLevel`, `width_in, depth_in, clear_height_in: float`, `secured: bool` | — |

| ID | Clause |
|---|---|
| SVC-INV-1 | `all(t.width_in == self.length_in for t in children)` and tier shelf levels are distinct |
| TIER-INV-1 | `presentation_plane == FRONT and leaf_capacity == width_in and usable_depth_in == depth_in` |

**Produce tables.** Footprint-based, with no shelf levels.

| Class | Prefix | Own attributes | Children |
|---|---|---|---|
| `ProduceTable` | `PRD-TBL-` | `location_description: str`, `footprint_sqft: float` | ≥1 `Bin` |
| `Bin` (leaf) | `PRD-BIN-` | `position: int`, `footprint_sqft: float` | — |

| ID | Clause |
|---|---|
| TBL-INV-1 | `sum(b.footprint_sqft for b in children) <= self.footprint_sqft` |
| BIN-INV-1 | `presentation_plane == TOP and leaf_capacity == footprint_sqft and shelf_level is None and usable_depth_in is None and clear_height_in is None` — no shelf levels on produce tables `[DISC]` |
| BIN-INV-2 | For every pair `p != q` in `placements`, `not p.window.overlaps(q.window)` — a bin holds one SKU at a time `[SIM]` (Q-12) |

**Floor displays.** A flat footprint measured as an area, since horizontal decks allocate square feet. `[VS 8]` This resolves v0.1 Q-9.

| Class | Prefix | Own attributes | Children |
|---|---|---|---|
| `FloorDisplay` (leaf, also a top-level fixture) | `FD-` | `location_description: str`, `capacity_linear_in: float`, `footprint_depth_in: float`, `max_stack_height_in: float \| None`, `secured: bool` | — |

| ID | Clause |
|---|---|
| FD-INV-1 | `presentation_plane == TOP and leaf_capacity == capacity_linear_in * footprint_depth_in / 144 and shelf_level is None and usable_depth_in == footprint_depth_in and clear_height_in == max_stack_height_in` — flat, no tiers `[DISC]` |
| FD-INV-2 | `all(p.placement_type == PROMOTIONAL for p in placements)` — floor displays carry promotional placements only `[SIM]` (Q-10) |
| FD-INV-3 | The display's top-level `footprint_sqft` equals `leaf_capacity` |

### 3.6 Blocking coordinates `[VS 12]`

The valuable vertical coordinate is the **shelf level**. The horizontal coordinate is (bay position, offset) and carries no allocation preference.

| ID | Clause |
|---|---|
| BLK-1 | `block_group()`: a `ShelfRun` under a `GondolaSide` or `ShelvingRun` → that side or run; a door-case `ShelfRun` → its `DoorCaseRun`; a `Tier` → its `ServiceCase`; a `Bin` → its `ProduceTable`; `FloorDisplay`, `PegSection`, `ClipStrip` → self |
| BLK-2 | `bay_position()`: a `ShelfRun` → its `Bay.position`; a door-case `ShelfRun` → its `Door.number`; every other leaf → `1` |

---

## Part 4 — Products, SKUs, categories, business units

### 4.1 `Product` — what the thing is `[VS 1, 8]`

A product is the manufactured item with its brand, size, formulation, and UPC. It owns the physical truth about the item, so its dimensions live here regardless of who stocks it or how it is displayed. `[VS 1, 8]`

```
class Product
queries
    upc: str
    name: str
    brand: str
    manufacturer: str
    private_label: bool
    perishable: bool
    form: ProductForm
    width_in, height_in, depth_in: float              -- X, Y, Z: consumer-unit bounding box as merchandised
    case_dims_in: tuple[float, float, float] | None   -- case-pack (shipper) bounding box
    units_per_case: int
    stackable: bool
    spec: NonBoxSpec | None                           -- overrides for products that are not boxes
    -- derived
    skus: tuple[SKU, ...]                             -- never stored on Product
    permitted_planes() -> frozenset[PresentationPlane]
    facing_extent(unit: SpaceUnit) -> float
    units_capacity_for(leaf: LeafSpace, extent: float) -> int

value NonBoxSpec
    facing_extent: Mapping[SpaceUnit, float]          -- extent one facing consumes, per unit
    units_per_extent: Mapping[SpaceUnit, float]       -- units held per inch / sq ft of extent
```

The bounding box is a box and nothing more: not the shape, not the graphics, not how the item is oriented in the case. `[VS 8]` Products that are not boxes (bagged goods that slump, hanging peg items, loose produce) take an override instead of a computed number. `[VS 8]`

| ID | Clause |
|---|---|
| PROD-INV-1 | `width_in > 0 and height_in > 0 and depth_in > 0 and units_per_case >= 1`; if `case_dims_in` is not `None`, all three of its values are `> 0` |
| PROD-INV-2 | `form in {BOX, HANGING} ⇒ spec is None`. `form in {SOFT_PACK, LOOSE} ⇒ spec is not None`, with `LINEAR_IN in spec.facing_extent` for `SOFT_PACK`, `SQFT in spec.facing_extent` for `LOOSE`, every listed unit also present in `units_per_extent`, and every value `> 0` |
| PROD-INV-3 | `upc` is unique. `skus == tuple(s for s in store.skus.values() if s.product is self)` — the SKU references the product, not the reverse. One product may have several SKUs in principle `[VS 1]` |
| PROD-Q-1 | `permitted_planes()`: `BOX → {FRONT, TOP}`; `SOFT_PACK → {FRONT} ∪ ({TOP} if SQFT in spec.facing_extent else ∅)`; `HANGING → {PEG}`; `LOOSE → {TOP}` `[SIM]` (Q-6) |
| PROD-Q-2 | `facing_extent(u)`: require `plane_of(u) in permitted_planes()`. Ensure `result > 0` and: `BOX`: `LINEAR_IN → width_in`, `SQFT → width_in * depth_in / 144`. `HANGING`: `HOOKS → 1.0`. `SOFT_PACK` and `LOOSE`: `spec.facing_extent[u]`. Front face uses X by Y and consumes X. Top face uses X by Z and consumes X·Z. `[VS 8]` |
| PROD-Q-3 | `units_capacity_for(leaf, x)`: require `fits_dimensionally(self, leaf)` and `x > 0`. Let `f = floor(x / facing_extent(leaf.unit) + EPS)`, `stack = max(1, floor(leaf.clear_height_in / height_in + EPS))` if `stackable and leaf.clear_height_in is not None` else `1`. Ensure `result ==` for `BOX` on `FRONT`: `f * floor(leaf.usable_depth_in / depth_in + EPS) * stack`; for `BOX` on `TOP`: `f * stack`; for `HANGING`: `f * leaf.hook_depth_units`; for `SOFT_PACK`/`LOOSE`: `floor(x * spec.units_per_extent[leaf.unit])`. Units held is a derived figure that rolls up across planes. `[VS 8]` |

---

### 4.2 `SKU` — how we stock, price, and shelve it here `[VS 1]` `[TB 2.2, 2.3]`

Product answers "what is this". SKU answers "how do we stock, price, and shelve it here". This version specifies only what allocation reads. All features are **queries**. Later sections add features but do not change these.

```
class SKU
queries
    sku_id: str
    product: Product
    category: CategoryNode                          -- a leaf of the category hierarchy
    retail_price: float                             [TB 7.1]
    unit_cost: float
    sales_velocity_units_per_week: float            [TB 2.2, 2.3]
    assortment_status: AssortmentStatus             [TB 2.6]
    season: tuple[int, int] | None                  -- (first_month, last_month) inclusive, may wrap the year; None = year-round
    promo_display_eligible: bool
    -- delegated to product (Uniform Access; never stored on the SKU)
    brand, manufacturer: str
    private_label, perishable: bool
    -- derived
    department: CategoryNode                        -- root of the category hierarchy above this SKU
    business_unit() -> BusinessUnit
    unit_margin: float
    gross_margin_pct: float
    weekly_gross_margin: float
    in_season(on: date) -> bool                     [TB 2.1 occasional/seasonal]
    placements: tuple[PlacementRecord, ...]         -- all, including ended and future-dated
    placements_on(on: date) -> frozenset[PlacementRecord]
    home_placement(on: date) -> Placement | None    -- see HOME-1 (6.4)
```

| ID | Clause |
|---|---|
| SKU-INV-1 | `retail_price > 0 and unit_cost >= 0 and sales_velocity_units_per_week >= 0` |
| SKU-INV-3 | `unit_margin == retail_price - unit_cost` and `gross_margin_pct == unit_margin / retail_price` |
| SKU-INV-4 | `weekly_gross_margin == sales_velocity_units_per_week * unit_margin` |
| SKU-INV-6 | `department is self.category.path_root()` — the root node of the category hierarchy |
| SKU-INV-8 | `all(p.sku is self for p in placements)` |
| SKU-INV-9 | `self.category.is_leaf` — SKUs attach to leaf category nodes only `[SIM]` (Q-3) |
| SKU-INV-10 | `brand == product.brand and manufacturer == product.manufacturer and private_label == product.private_label and perishable == product.perishable` |
| SKU-Q-1 | `in_season(d)`: `result == (season is None or (first <= d.month <= last if first <= last else (d.month >= first or d.month <= last)))` |
| SKU-Q-2 | `placements_on(d)`: `result == frozenset(p for p in placements if p.window.contains(d))` |
| SKU-ZERO | **Zero placements is a legitimate state**, for a delisted or not-yet-reset item. No invariant requires a SKU to have a placement. One SKU may have any number of placements. Several placements signal something: a secondary display, an endcap, a seasonal off-shelf position. `[VS 2]` |

---

### 4.3 `CategoryNode` — the product-category hierarchy and space policy `[VS 6]` `[TB 1.2, 2.1–2.5]`

The category hierarchy is arbitrary-depth: department, category, subcategory, segment is the usual ladder, and some branches go deeper than others. `level_label` is a free label, not a level in the type system. `[VS 6]` Textbook 2.1 treats a category as a "discrete business unit". This contract separates the category hierarchy from the business-unit hierarchy (4.4). See companion T-2.

```
class CategoryNode           (conforms to HierarchyNode and Rollup)
queries
    id: str                                    -- "CAT-" prefix
    name: str
    level_label: str
    own_policy: Mapping[str, object]           -- fields this node defines itself (may be empty)
    last_reset_date: date | None               -- own state; set by execution of a CYCLE version
    policy() -> CategoryPolicy                 -- effective policy, resolved through ancestors
    skus: tuple[SKU, ...]                      -- SKUs attached exactly to this node
    subtree_skus() -> frozenset[SKU]
    assigned_spaces() -> tuple[LeafSpace, ...] -- allocatable leaves whose effective assigned_category is this node
    effective_last_reset_date() -> date | None
    next_reset_due(calendar: tuple[date, ...]) -> date | None
    private_label_space_share(on: date) -> float

value CategoryPolicy
    role: CategoryRole                                     [TB 2.1]
    temperature_zone: TemperatureZone                      [TB 1.3]
    allowed_space_types: frozenset[SpaceType]              -- a type allows its whole subtree of types
    secured_case_required: bool                            [TB 6.4]
    min_facings_per_sku: int                               [TB 2.3]
    max_facings_per_sku: int                               [TB 2.3]
    space_elasticity: float                                [TB 2.3]
    sales_per_linear_ft_per_week: float                    [TB 2.3]   (reporting input)
    category_captain: str | None                           [TB 2.4]   manufacturer name or None
    private_label_space_target_pct: float                  [TB 2.5]
    reset_cycle_months: int                                [TB 2.6] [VS 4]
    blocking_mode: BlockingMode                            [VS 12]
    private_label_benchmark_brand: str | None              [VS 12]
```

| ID | Clause |
|---|---|
| CAT-INV-1 | A root node's `own_policy` defines **every** `CategoryPolicy` field, so `policy()` is total |
| CAT-Q-1 | `policy()`: for each field `f`, `result.f` is the value in `own_policy` of the nearest node in `(self,) + self.ancestors()` that defines `f` |
| CAT-INV-2 | `1 <= policy().min_facings_per_sku <= policy().max_facings_per_sku` |
| CAT-INV-3 | `0 < policy().space_elasticity < 1` — incremental facings show diminishing returns `[TB 2.3]` |
| CAT-INV-4 | `0 <= policy().private_label_space_target_pct <= 100` |
| CAT-INV-5 | `policy().reset_cycle_months >= 1` (typical range 3–12: quarterly for fast categories with heavy new-item flow, one or two resets a year in the center store) `[VS 4]` |
| CAT-INV-6 | `all(l.assigned_category is self and any(l.space_type.is_within(t) for t in policy().allowed_space_types) for l in assigned_spaces())` |
| CAT-INV-7 | `all(s.category is self for s in skus)` and `len(skus) == 0 or self.is_leaf` |
| CAT-INV-8 | `policy().blocking_mode == HORIZONTAL_TIER ⇒` the category is one with a dominant private label or a clear tier structure. This is data, not checkable, so it is a documentation rule only. `[VS 12]` |
| CAT-Q-2 | `effective_last_reset_date()`: `result` is `last_reset_date` of the nearest node in `(self,) + self.ancestors()` with a non-`None` value, else `None` |
| CAT-Q-3 | `next_reset_due(cal)`: `result is None if effective_last_reset_date() is None else min((d for d in cal if d >= add_months(effective_last_reset_date(), policy().reset_cycle_months)), default=None)` — resets happen on scheduled windows across the chain, not rolling per vendor `[VS 4]` |
| CAT-Q-4 | `private_label_space_share(on)`: `result == 0 if T == 0 else 100 * PL / T`, where `T` is the total `extent` of `PERMANENT` placements live on `on` on `LINEAR_IN` leaves that are assigned within `self.subtree()`, and `PL` is the part of `T` for private-label SKUs. Reporting query only. |

`CategoryNode` conforms to `Rollup` per Part 2. The private-label target `[TB 2.5]` and department space shares `[TB 1.2]` are **objectives, not contractual postconditions** in this version. They are reported, not asserted (Q-14).

---

### 4.4 `BusinessUnit` — the business-unit hierarchy `[VS 6, 9]`

Real chains have division, banner, region, store, and department. The hierarchy has arbitrary depth, and `kind` is a free label.

```
class BusinessUnit           (conforms to HierarchyNode and Rollup)
queries
    id: str                                    -- "BU-" prefix
    name: str
    kind: str                                  -- e.g. "COMPANY", "DIVISION", "BANNER", "REGION", "STORE", "DEPARTMENT"
    root_category: CategoryNode | None         -- non-None only for kind == "DEPARTMENT"
```

| ID | Clause |
|---|---|
| BU-INV-1 | `(root_category is not None) == (kind == "DEPARTMENT")` and `kind == "DEPARTMENT" ⇒ is_leaf` |
| BU-INV-2 | Every root `CategoryNode` is the `root_category` of exactly one `BusinessUnit` |
| BU-INV-3 | Exactly one unit has `kind == "STORE"`, it is `Store.business_unit`, and every `DEPARTMENT` unit is within it (single store, Q-23) |
| BU-Q-1 | `SKU.business_unit()`: `result.root_category is sku.department` |

A placement's business unit is derived (PLC-Q-2): `placement.sku.business_unit()`. It is never stored.

---

## Part 5 — Agreements and placements

A placement is the **commitment**: what was agreed to give a vendor, where, and how much. It is not just a physical fact. `[VS 2]` Three things are kept apart throughout the model: the **planogram** as intended state (Part 6), the **placement** as agreed commitment (this part), and **observed shelf state** as what is actually there (reserved, Part 10). `[VS 10]`

### 5.1 `Agreement` — the ladder of deals `[VS 3]`

A separate class, not a column on the placement. There is a ladder: slotting agreements for permanent shelf space tied to reset windows; promotional display deals of typically four to eight weeks, priced per display per store; pay-to-stay fees for holding existing space; and opportunistic buys with no space commitment at all. `[VS 3]` Slotting is ongoing rented space, priced by facings or linear feet and renewed at reset. It is not an upfront payment (companion T-1).

```
class Agreement
queries
    agreement_id: str                        -- "AGR-" prefix
    agreement_type: AgreementType
    vendor: str                              -- manufacturer name
    fee_basis: FeeBasis
    fee_rate: float                          -- dollars per basis unit, for the whole term
    window: DateWindow
    -- derived
    placements: tuple[PlacementRecord, ...]  -- every placement that references this agreement
    term_fee(p: Placement) -> float
```

| ID | Clause |
|---|---|
| AGR-INV-1 | `fee_rate >= 0` and `window.end is None or window.start < window.end` |
| AGR-INV-2 | `agreement_type == SLOTTING ⇒ window.start in store.reset_calendar` — slotting is tied to reset windows `[VS 3, 4]` |
| AGR-INV-3 | `agreement_type == PROMOTIONAL_DISPLAY ⇒ window.end is not None and fee_basis in {PER_DISPLAY, FLAT}` |
| AGR-INV-4 | `agreement_type == OPPORTUNISTIC_BUY ⇒ fee_rate == 0` — the deal is on the product, not the shelf `[VS 3]` |
| AGR-INV-5 | `agreement_type in {SLOTTING, PAY_TO_STAY} ⇒ fee_basis in {PER_FACING, PER_LINEAR_FT, FLAT}` |
| AGR-INV-6 | `all(compatible(agreement_type, p.placement_type) and window_covers(window, p.window) for p in placements)`, where `window_covers(a, b)` means `a.start <= b.start and (a.end is None or (b.end is not None and b.end <= a.end))` |
| AGR-INV-7 | `all(p.sku.manufacturer == vendor for p in placements)` |
| AGR-Q-1 | `term_fee(p)`: require `p in placements and p.window.end is not None`. Ensure `result == fee_rate * q`, where `q` is `p.facings` for `PER_FACING`; `p.extent / 12` for `PER_LINEAR_FT` (require `p.space.unit == LINEAR_IN`); `1` for `PER_DISPLAY` and `FLAT` |

Space carries no price (rule 8). An endcap costs more only because the agreement rate negotiated for a placement on it is higher.

### 5.2 `PlacementRecord` (deferred) and `Placement` (committed) `[VS 2, 3, 4, 8]`

```
deferred class PlacementRecord
queries
    placement_id: str
    sku: SKU
    placement_type: PlacementType
    window: DateWindow                         -- effective start and end (half-open)
    agreement: Agreement | None
    vendor_funded: bool                        -- derived: agreement is not None and agreement.fee_rate > 0
    business_unit: BusinessUnit                -- derived: sku.business_unit()
```

Both `Placement` and `IncidentalPlacement` (5.3) conform. "All placements live on a given date" returns both kinds.

```
class Placement                                -- conforms to PlacementRecord
queries
    placement_id: str                          -- "PLC-" prefix
    sku: SKU
    space: LeafSpace
    placement_type: PlacementType              -- PERMANENT or PROMOTIONAL
    extent: float                              -- allocated extent, in space.unit
    offset_in: float | None                    -- from the leaf's left edge; LINEAR_IN leaves only
    start_date: date
    end_date: date | None                      -- exclusive; None = open
    agreement: Agreement | None
    -- derived (Uniform Access; never stored)
    window: DateWindow
    facings: int
    units_capacity: int
    shelf_level: ShelfLevel | None
    space_type: SpaceType
```

A placement stores the **allocated extent**, in the space's unit. Facings are derived from it and never stored. `[VS 8]` The workbook-style pair (facings and linear space assigned) is one fact, not two.

| ID | Clause |
|---|---|
| PLC-Q-1 | `facings`: `result == floor(extent / sku.product.facing_extent(space.unit) + EPS)` |
| PLC-Q-2 | `window == DateWindow(start_date, end_date)`; `shelf_level == space.shelf_level`; `space_type == space.space_type`; `business_unit is sku.business_unit()` — derived from where the SKU sits, never entered separately |
| PLC-Q-3 | `units_capacity == sku.product.units_capacity_for(space, extent)` |
| PLC-INV-1 | `extent > 0` and `facings >= 1` |
| PLC-INV-2 | `placement_type in {PERMANENT, PROMOTIONAL}` |
| PLC-INV-3 | `fits_dimensionally(sku.product, space)` (FIT-1) |
| PLC-INV-4 | **Whole facings.** `sku.product.form != BOX or abs(extent - facings * sku.product.facing_extent(space.unit)) <= EPS`. A box product is allocated in whole facings. A non-box product may have slack, because its extent is an override. For a produce `Bin`, `extent == space.leaf_capacity` (a bin is allocated whole). |
| PLC-INV-5 | `(offset_in is None) == (space.unit != LINEAR_IN)`; if linear, `0 <= offset_in and offset_in + extent <= space.leaf_capacity + EPS` |
| PLC-INV-6 | `space.temperature_zone == sku.category.policy().temperature_zone` — the cold chain is unbroken on the shelf `[TB 4.4]` |
| PLC-INV-7 | `placement_type != PERMANENT or (space.assigned_category is not None and sku.category.is_within(space.assigned_category))` — permanent space belongs to the SKU's category or an ancestor of it `[TB 2.1]` |
| PLC-INV-8 | `end_date is None or start_date < end_date` — the interval is non-empty. An end that precedes its start is a violation. `[VS 14]` |
| PLC-INV-9 | `placement_type != PROMOTIONAL or (end_date is not None and sku.promo_display_eligible)` — promotional placements are time-boxed |
| PLC-INV-10 | `placement_type != PERMANENT or sku.product.form == LOOSE or (policy.min_facings_per_sku <= facings <= policy.max_facings_per_sku)`, where `policy = sku.category.policy()` |
| PLC-INV-11 | `not sku.category.policy().secured_case_required or space.secured` `[TB 6.4]` |
| PLC-INV-12 | `agreement is None or (compatible(agreement.agreement_type, placement_type) and window_covers(agreement.window, window))` (AGR-INV-6) |
| PLC-INV-13 | `self in space.placements and self in sku.placements` |
| PLC-INV-14 | `placement_type != PERMANENT or start_date in store.reset_calendar or start_date in {v.effective_date for v in store.planogram.versions if v.kind == REVISION}` — permanent placements start at a reset date, or at a released mid-cycle revision `[VS 4, 10]` |
| PLC-INV-IMM | **Immutable facts.** After creation, no attribute of a `Placement` changes except `end_date`, and `end_date` may change only once, from `None` to a date `> start_date`. A placement is never deleted, except by `cancel_future_placement` (7.3). This keeps the full history, so "what was on this shelf on this date" is one uniform query. `[VS 4]` |

No archive table exists. The volume is tens of thousands of rows even with years of history, and all of it stays live in the model. `[VS 4]` Placement type does not distinguish "primary". Which placement is the home position for replenishment is decided by the planogram (6.4). `[VS 2]`

### 5.3 `IncidentalPlacement` — no space commitment `[VS 3]`

Overstock, or a DSD drop stacked wherever there was floor that week. It has no fixed position, so the model refuses to invent one: no leaf, no extent, no offset. `[VS 3]`

```
class IncidentalPlacement                      -- conforms to PlacementRecord
queries
    placement_id: str                          -- "INC-" prefix
    sku: SKU
    placement_type: PlacementType              -- always INCIDENTAL
    units_held: int
    start_date: date
    end_date: date | None                      -- exclusive; None = open
    agreement: Agreement | None                -- None or OPPORTUNISTIC_BUY
    zone_hint: SpaceNode | None                -- coarse, best-effort, and may change
```

| ID | Clause |
|---|---|
| INC-INV-1 | `placement_type == INCIDENTAL and units_held >= 1` |
| INC-INV-2 | `end_date is None or start_date < end_date` |
| INC-INV-3 | `agreement is None or (agreement.agreement_type == OPPORTUNISTIC_BUY and window_covers(agreement.window, window))` |
| INC-INV-4 | The class has no `space`, `extent`, `facings`, or `offset_in` feature. It contributes to no `Rollup` measure (Part 2) and to no `LeafSpace` allocation. |
| INC-INV-5 | `zone_hint` is the one mutable feature, changed only by `move_incidental` (7.3). It is never used to compute any measure. It is used only by the space filter `placements_in_space` (7.2). |

Cold-chain rules for incidental placements are deferred to Restocking and Receiving (Part 10).

---

## Part 6 — Allocation: policy, plan, and planogram versions

### 6.1 Pools and planes

Space is assigned to category nodes (7.3), at any depth. A SKU draws on the space of its **pool**.

- `primary_plane(sku)`: `BOX` and `SOFT_PACK` → `FRONT`; `HANGING` → `PEG`; `LOOSE` → `TOP`. `[SIM]` (Q-6)
- `pool_node(sku, plane)`: the nearest node in `(sku.category,) + sku.category.ancestors()` whose `assigned_spaces()` contains a leaf with `presentation_plane == plane`, or `None`. `[SIM]` (Q-2)
- `pool_capacity(node, plane)`: `sum(l.leaf_capacity for l in node.assigned_spaces() if l.presentation_plane == plane)`, in `plane_unit(plane)`.

### 6.2 `AllocationPolicy` — the scoring rules allocation obeys `[TB 2.3]`

The textbook's allocation inputs are sales velocity, gross margin, vendor category-captain input, private-label targets, and space elasticity with diminishing returns `[TB 2.3]`. The policy turns them into a deterministic target. Allocation (6.4) must respect the target through checkable postconditions. How it searches for positions is left to the implementation.

```
class AllocationPolicy
queries
    score(sku: SKU) -> float
    target_facings(sku: SKU, in_scope: Sequence[SKU]) -> int | None
    level_preference(level: ShelfLevel | None) -> int
    captain_weight: float           -- default 1.0 (no captain effect)
```

| ID | Clause |
|---|---|
| POL-1 | `score(sku) == sku.weekly_gross_margin` — velocity × margin `[TB 2.3]`. GMROI is the alternative (Q-5) |
| POL-2 | `target_facings(s, S)`: `None` if `s.product.form == LOOSE` or `pool_node(s, primary_plane(s)) is None`. Otherwise let `p = primary_plane(s)`, `A = pool_node(s, p)`, `e = A.policy().space_elasticity`, `C = pool_capacity(A, p)`, `f = s.product.facing_extent(plane_unit(p))`, `w_i = max(score(i), 0) ** (1 / (1 - e))` over `competitors = [i in S if i.in_season(as_of) and primary_plane(i) == p and pool_node(i, p) is A]`, and `share = w_s / sum(w_i)` (share `= 1 / len(competitors)` if all `w_i == 0`). Then `result == clamp(floor(share * C / f), lo, hi)` with `lo, hi = s.category.policy().min_facings_per_sku, .max_facings_per_sku`. This is the optimal space split when each SKU's sales grow as space^e `[TB 2.3]`. The formula is `[SIM]` (Q-4) |
| POL-3 | `target_facings` is monotone in score within a pool: for `a`, `b` with the same pool and plane, `score(a) >= score(b) and a.product.facing_extent(u) <= b.product.facing_extent(u) ⇒ target_facings(a) >= target_facings(b)` |
| POL-4 | `level_preference(l) == l.preference_rank` for a shelf level; `== 0` for `None` (unranked). **Horizontal position carries no preference**: bay position and offset never affect a plan's quality measure. It is much flatter than the vertical gradient. `[VS 12]` |
| POL-5 | `captain_weight == 1.0` in this version: the captain's input does not affect allocation, and private-label placement is a retailer decision the captain works around `[VS 12]` `[TB 2.4]` (Q-7) |

### 6.3 `PlacementPlan` — the result of allocation (a pure value)

```
value PlannedPlacement
    sku: SKU
    space: LeafSpace
    extent: float
    offset_in: float | None
    agreement: Agreement | None          -- always None as generated; see with_agreement

class PlacementPlan
queries
    scope: frozenset[CategoryNode]
    as_of: date                          -- the effective date the layout is planned for
    entries: tuple[PlannedPlacement, ...]       -- PERMANENT placements only
    home: Mapping[SKU, PlannedPlacement]
    unplaced: Mapping[SKU, UnplacedReason]
    under_target: tuple[SKU, ...]
    delist_review: tuple[SKU, ...]              -- reporting only [TB 2.3, 2.6]
    block_breaks: tuple[tuple[str, str], ...]   -- (block group id, brand) pairs whose block is broken
    adjacency_breaks: tuple[str, ...]           -- block group ids where private label is not next to its benchmark
    with_agreement(sku: SKU, agreement: Agreement) -> PlacementPlan
```

| ID | Clause |
|---|---|
| PLAN-INV-1 | `{e.sku for e in entries}.isdisjoint(unplaced.keys())` |
| PLAN-INV-2 | every entry's `sku.category` is within the subtree of some node in `scope` |
| PLAN-INV-3 | at most one entry per SKU (the generator makes one) |
| PLAN-INV-4 | **Feasibility.** The entries, taken as `PERMANENT` placements with window `[as_of, None)` and with every existing `PERMANENT` placement of an in-scope SKU treated as ended at `as_of`, satisfy LS-INV-5, LS-INV-6, BIN-INV-2, FD-INV-2, and PLC-INV-1 through PLC-INV-13 against the leaves they name |
| PLAN-INV-5 | `under_target ⊆ {e.sku for e in entries}` |
| PLAN-INV-6 | `delist_review` is the tuple of placed or unplaced in-scope SKUs whose `assortment_status == DELIST_CANDIDATE`, or whose score is in the bottom decile of their pool. Order: category path, then ascending score `[TB 2.3]`. The decile threshold is `[SIM]` |
| PLAN-INV-7 | `set(home) == {e.sku for e in entries}` and `all(home[s].sku is s and home[s] in entries for s in home)` |
| PLAN-INV-8 | `block_breaks` and `adjacency_breaks` equal the values computed by BLK-3 and BLK-4 from `entries` — the plan reports honestly |
| PLAN-INV-9 | `with_agreement(s, a)`: require `s` is placed and `a.agreement_type in {SLOTTING, PAY_TO_STAY}`, `a.vendor == s.manufacturer`. Ensure a new plan equal to `self` except `s`'s entry (and its `home` mapping) carries `a`. `self` is unchanged. |

A `PlacementPlan` is immutable. Building one has no side effects. The plan is what a category manager edits before release. It is a different object from any released version (6.5). `[VS 10]`

**Brand blocking `[VS 12]`.** Adjacency is specified in the planogram and is often part of what the vendor negotiated.

| ID | Clause |
|---|---|
| BLK-3 | `block_breaks`. Group the entries by `(pool A, group G = e.space.block_group())` and brand. For `VERTICAL_BRAND` (the default): let `P_b` be the set of `e.space.bay_position()` values of brand `b`'s entries in `G`. The pair `(G.id, b)` is in `block_breaks` iff some position `q` with `min(P_b) < q < max(P_b)` and `q ∉ P_b` holds an entry of a different brand. A vertical brand block gives every brand a share of good and poor shelves. For `HORIZONTAL_TIER`: `(G.id, b)` is in `block_breaks` iff `b`'s entries in `G` span more than one shelf level. |
| BLK-4 | `adjacency_breaks`. For a pool whose `policy().private_label_benchmark_brand` is `b*` (not `None`): a group `G` containing entries of both `b*` and private-label SKUs is in `adjacency_breaks` iff `min(bay positions of private-label entries in G) != max(bay positions of b* entries in G) + 1`. Private label sits immediately right of the brand it is benchmarked against. |

Breaks are reported, not required to be empty. Minimizing them is an objective (companion Q-22).

### 6.4 Store operation `generate_plan` — **the allocation function** `[DISC]`

Allocation takes a set of available merchandising spaces and a set of SKUs and generates a set of SKU placements `[DISC]`. Here the spaces are the `assigned_spaces()` of the pools, the SKUs are `Store.in_scope_skus(scope, as_of)`, and the output is a `PlacementPlan`. It is a **query**: calling it never changes the model. `[DbC]` (CQS)

`Store.generate_plan(scope: frozenset[CategoryNode], as_of: date) -> PlacementPlan`

**require**

| ID | Clause |
|---|---|
| GEN-PRE-1 | `len(scope) >= 1` and every element is a node of this store's category hierarchy |
| GEN-PRE-2 | the model satisfies ST-INV-* (implicit at a stable time; stated for emphasis) |

**ensure** — let `S = in_scope_skus(scope, as_of)`, `E = result.entries`, and `T(e) = policy.target_facings(e.sku, S)`

| ID | Clause |
|---|---|
| GEN-1 | **Completeness.** `{e.sku for e in E} ∪ result.unplaced.keys() == set(S)`. Every in-scope SKU is either placed once or reported with a reason |
| GEN-2 | **Seasonality.** For every `s in S`: `(not s.in_season(as_of)) == (result.unplaced.get(s) == OUT_OF_SEASON)` `[TB 2.1]` |
| GEN-3 | **Right space and plane.** Every `e in E` has `e.space.presentation_plane == primary_plane(e.sku)` and `e.space.assigned_category is pool_node(e.sku, primary_plane(e.sku))`, and the plan is feasible (PLAN-INV-4) |
| GEN-4 | **Honest reasons.** For every `s` in `unplaced` other than `OUT_OF_SEASON`, with `p = primary_plane(s)`: `NO_CATEGORY_SPACE ⇒` no node in `(s.category,) + s.category.ancestors()` has any `assigned_spaces()`. `NO_SPACE_IN_PLANE ⇒` some such node has assigned spaces but `pool_node(s, p) is None`. `NO_DIMENSIONAL_FIT ⇒` no leaf in `pool_node(s, p).assigned_spaces()` with plane `p` satisfies `fits_dimensionally(s.product, leaf)`. `INSUFFICIENT_SPACE ⇒` in the planned state no such fitting leaf has room for `s` at `s.category.policy().min_facings_per_sku` facings (a whole bin for `LOOSE`) |
| GEN-5 | **Facing bounds.** For every non-`LOOSE` `e`: `lo <= e.facings <= max(lo, T(e))`, where `lo = e.sku.category.policy().min_facings_per_sku`, and `e.facings` is derived from `e.extent` per PLC-Q-1. The plan never gives a SKU more than its target. For a `LOOSE` SKU, `e.extent == e.space.leaf_capacity` |
| GEN-6 | **No idle space while under target.** For every non-`LOOSE` `e` with `e.facings < T(e)`: in the planned state `e.space` has no free block adjacent to `e`'s interval (`LINEAR_IN`), or less than one facing's extent free (other units). And `under_target == tuple(e.sku for e in E if non-LOOSE and e.facings < T(e))` |
| GEN-7 | **Facing monotonicity.** For `a, b in E` in the same pool with `score(a) > score(b)` and `a.sku.product.facing_extent(u) <= b.sku.product.facing_extent(u)`: `a.facings >= b.facings`, unless GEN-6 blocked `a` from growing `[TB 2.3]` |
| GEN-8 | **Best-level priority.** For `a, b in E` in the same pool and group class (both on shelf levels) with `score(a) > score(b)` and `level_preference(a.space.shelf_level) > level_preference(b.space.shelf_level)` (`a` sits on the worse level): swapping `a` and `b` would violate PLAN-INV-4 or change a facing count. The better-performing SKU gets the better shelf level whenever a feasible swap exists `[TB 2.3]` |
| GEN-9 | **Determinism.** Equal model states and equal arguments give equal plans. Ties break by `sku_id`, then leaf id, then offset |
| GEN-10 | **Pure.** `self` is unchanged (CQS) |
| GEN-11 | **Home designation.** `result.home[s]` is, among `s`'s entries, the one with the best `level_preference`, ties broken by larger `extent`, then `(space.id, offset_in)` |

Plan quality beyond GEN-5 through GEN-8 is up to the implementation (Q-21). Blocking (BLK-3, BLK-4) and the private-label target are objectives that are reported, not asserted.

### 6.5 The planogram: versions, freezing, and drift `[VS 10]`

The planogram is versioned. A revision creates a new authoritative version mid-cycle. The shelf never quietly departs from the old one. `[VS 10]` The version the crew executes and the version the category manager is currently editing are **deliberately different objects**. Revisions made in the meantime queue up for the next release. `[VS 10]`

```
class PlanogramVersion                           -- immutable snapshot
queries
    version_no: int
    kind: VersionKind                            -- CYCLE (on a reset date) or REVISION (mid-cycle)
    scope: frozenset[CategoryNode]
    cutoff_date: date                            -- the reset pack is the plan as of this date
    effective_date: date
    entries: tuple[PlannedPlacement, ...]
    home: Mapping[SKU, PlannedPlacement]
    status: VersionStatus                        -- the one mutable feature: RELEASED → EXECUTED, once

class Planogram
queries
    working: PlacementPlan | None                -- what the category manager is editing
    versions: tuple[PlanogramVersion, ...]       -- ordered by version_no
    current_version(category: CategoryNode, on: date) -> PlanogramVersion | None
```

| ID | Clause |
|---|---|
| PGV-INV-1 | After release, `scope`, `entries`, `home`, `kind`, `cutoff_date`, `effective_date` never change. Only `status` changes, once, `RELEASED → EXECUTED`. |
| PGV-INV-2 | `cutoff_date <= effective_date`, and `kind == CYCLE ⇒ effective_date in store.reset_calendar` |
| PGV-INV-3 | `[v.version_no for v in versions] == list(range(1, len(versions) + 1))` |
| PGV-INV-4 | **One pending version per scope.** For any two versions `v != w` with `v.scope` and `w.scope` overlapping (some node of one is within the subtree of a node of the other): not both `RELEASED`. And if `v.version_no < w.version_no` then `v.effective_date < w.effective_date` |
| PGV-INV-5 | `all(e.sku.category.is_within(n) for e in entries for n in [some n in scope])` and `set(home) == {e.sku for e in entries}` |
| PGV-Q-1 | `current_version(c, d)`: `result` is the version with the greatest `version_no` among those with `status == EXECUTED`, `effective_date <= d`, and some `n in scope` with `c.is_within(n)`; `None` if there is none. **Compliance is always measured against the current version.** |

**Home position `[VS 2]`.** "Primary" is not a flag on a placement. It falls out of which placement the planogram treats as the home position for replenishment.

| ID | Clause |
|---|---|
| HOME-1 | `Store.home_placement(sku, on)`: let `v = planogram.current_version(sku.category, on)` and `h = v.home.get(sku)` if `v` exists. Ensure `result` is the `PERMANENT` placement `p` of `sku` live on `on` with `(p.space, p.offset_in) == (h.space, h.offset_in)` if `h` exists and such a `p` exists, else `None`. So `result is None or (result.placement_type == PERMANENT and result.sku is sku and result.window.contains(on))` |
| HOME-2 | If `sku` has any `PERMANENT` placement live on `on` that a version placed, `home_placement(sku, on)` is not `None`. It designates exactly one |

Observed shelf state is a third, separate artefact and is reserved (Part 10). No `Placement`, `PlanogramVersion`, or `Planogram` feature records observed state. `[VS 10]` `[SIM]`

---

## Part 7 — `Store`: the root, the facade, filters, and commands

### 7.1 Features and invariants

```
class Store                    (conforms to SpaceNode; also aggregates the other hierarchies)
queries
    id: str                                    -- "STORE-" prefix
    footprint_sqft: float                      -- the building
    business_unit: BusinessUnit                -- the unit with kind == "STORE"
    reset_calendar: tuple[date, ...]           -- scheduled chain-wide reset windows, ascending
    products: Mapping[str, Product]
    skus: Mapping[str, SKU]
    category_roots: tuple[CategoryNode, ...]
    business_unit_roots: tuple[BusinessUnit, ...]
    placements: Mapping[str, Placement]        -- ALL committed placements ever made, incl. ended
    incidentals: Mapping[str, IncidentalPlacement]
    agreements: Mapping[str, Agreement]
    policy: AllocationPolicy
    planogram: Planogram
```

`Store.space_type == SPACE` (the root of the type tree), and `Store.parent is None`.

| ID | Clause |
|---|---|
| ST-INV-1 | `parent is None and space_type == SPACE` |
| ST-INV-2 | AREA-INV-2 holds for the store: the building footprint reconciles to its children `[VS 6]` |
| ST-INV-3 | ids are unique per class and carry their prefix; `space(n.id) is n` for every space node `n`, and likewise for categories and business units |
| ST-INV-4 | **Referential integrity.** `all(p.space in self.allocatable_leaves() and p.sku is self.skus[p.sku.sku_id] for p in placements.values())`; the same for `incidentals` (SKU) and for each agreement referenced by a placement |
| ST-INV-5 | **The views agree.** `set(placements.values()) == {p for l in self.allocatable_leaves() for p in l.placements} == {p for s in skus.values() for p in s.placements if isinstance(p, Placement)}`, and `set(incidentals.values()) == {p for s in skus.values() for p in s.placements if isinstance(p, IncidentalPlacement)}` |
| ST-INV-6 | `reset_calendar` is strictly ascending |
| ST-INV-7 | `l in l.assigned_category.assigned_spaces()` for every leaf with an assigned category (SN-INV-2 makes the assignment single-valued) |
| ST-INV-8 | All `HierarchyNode`, `SpaceNode`, `LeafSpace`, `Placement`, `Agreement`, `CategoryNode`, `BusinessUnit`, `PlanogramVersion` invariants hold for every contained object. The model is consistent at every stable time. |

### 7.2 Queries and filters

**Lookups and reports**

| ID | Feature | require | ensure |
|---|---|---|---|
| ST-Q-1 | `space(id)`, `category(id)`, `business_unit_node(id)` | the id belongs to a node of that hierarchy | `result.id == id` |
| ST-Q-2 | `in_scope_skus(scope, as_of)` | every element of `scope` is a category node of this store | `result == tuple(s for s in skus.values() if any(s.category.is_within(n) for n in scope) and s.assortment_status != DELISTED)`, ordered by `(category.path, -policy.score(s), sku_id)`. Out-of-season SKUs are included and are reported by the plan as `OUT_OF_SEASON` |
| ST-Q-3 | `unassigned_leaves()` | — | `result == tuple(l for l in allocatable_leaves() if l.assigned_category is None)` |
| ST-Q-4 | `change_dates()` | — | `result == frozenset(p.start_date for p in placements.values()) | frozenset(p.end_date for p in placements.values() if p.end_date is not None)` |
| ST-Q-5 | `spaces_of_type(t)` | — | `result == frozenset(n for n in subtree() if n.space_type.is_within(t))` — the type hierarchy crossed with the physical one `[VS 6]` |
| ST-Q-6 | `selling_share()` | — | `result == sum(n.footprint_sqft for n in subtree() if n.footprint_sqft is not None and n.merchandisable and (n.parent is None or not n.parent.merchandisable)) / footprint_sqft`. A reporting figure only: selling space is roughly 60–70% of a real floor `[VS 6]` |
| ST-Q-7 | `all_placements()` | — | `result == frozenset(placements.values()) | frozenset(incidentals.values())` |
| ST-Q-8 | `home_placement(sku, on)` | — | HOME-1 |

**Placement filters** (rule 9: all return `frozenset[PlacementRecord]`, are pure, and are subsets of `all_placements()`). The "and" of two conditions is the intersection of their results, and "or" is the union. `[VS 13]`

| ID | Filter | ensure |
|---|---|---|
| FLT-1 | `placements_live_on(d)` | `result == frozenset(p for p in all_placements() if p.window.contains(d))` |
| FLT-2 | `placements_of_type(t)` | `result == frozenset(p for p in all_placements() if p.placement_type == t)` |
| FLT-3 | `placements_in_space(n)` | `result == frozenset(p for p in placements.values() if p.space.is_within(n)) | frozenset(p for p in incidentals.values() if p.zone_hint is not None and p.zone_hint.is_within(n))` |
| FLT-4 | `placements_in_category(c)` | `result == frozenset(p for p in all_placements() if p.sku.category.is_within(c))` |
| FLT-5 | `placements_in_unit(u)` | `result == frozenset(p for p in all_placements() if p.business_unit.is_within(u))` |
| FLT-6 | `placements_with_agreement_type(t)` | `result == frozenset(p for p in all_placements() if p.agreement is not None and p.agreement.agreement_type == t)` |
| FLT-7 | **Algebra.** For any filter results `F`, `G`: `F & G`, `F | G`, `F - G` are plain set operations. Also: the three `placements_of_type` results are pairwise disjoint and their union is `all_placements()`; and `placements_in_category` over the roots of the category hierarchy partitions `all_placements()`; and `placements_in_unit` over the `DEPARTMENT` units partitions `all_placements()`. |

If a string query language ever earns its place, it is a thin layer over these same filters, not a replacement. `[VS 13]`

### 7.3 Commands

All commands are **atomic**: if any postcondition fails, the model equals `old(self)`. Commands change state and return nothing, except where marked as a creation command that returns the new id (an accepted CQS exception `[DbC]`).

**`assign_space(node_id, category_id)`** — assigns a space node, and so all leaves under it, to a category node `[TB 1.2, 2.1]` `[SIM]` (Q-2)

| ID | Kind | Clause |
|---|---|---|
| ASN-PRE-1 | require | `space(node_id).merchandisable` |
| ASN-PRE-2 | require | `space(node_id).assigned_category is None` and no descendant of `space(node_id)` has an `own_assignment` (SN-INV-2) |
| ASN-PRE-3 | require | every allocatable leaf `l` under the node satisfies `any(l.space_type.is_within(t) for t in category(category_id).policy().allowed_space_types)` |
| ASN-PRE-4 | require | every such leaf has `temperature_zone == category(category_id).policy().temperature_zone` (LS-INV-7) |
| ASN-PRE-5 | require | `not category(category_id).policy().secured_case_required or all(l.secured for l in those leaves)` (LS-INV-8) |
| ASN-1 | ensure | `space(node_id).own_assignment is category(category_id)` |
| ASN-2 | ensure | nothing else changes |

**`unassign_space(node_id, as_of)`**

| ID | Kind | Clause |
|---|---|---|
| UNA-PRE-1 | require | `space(node_id).own_assignment is not None` |
| UNA-PRE-2 | require | every `PERMANENT` placement in the leaves under the node has `end_date is not None and end_date <= as_of` — clear permanent stock first, by a reset |
| UNA-1 | ensure | `space(node_id).own_assignment is None`; nothing else changes |

**`set_working_plan(plan)`**

| ID | Kind | Clause |
|---|---|---|
| WRK-PRE-1 | require | `plan` equals `generate_plan(plan.scope, plan.as_of)` except for agreements added by `with_agreement` |
| WRK-1 | ensure | `planogram.working == plan`; nothing else changes |

**`release_working_plan(kind, cutoff_date, effective_date) -> int`** — freezes the working plan as a new version, the "reset pack" `[VS 10]`

| ID | Kind | Clause |
|---|---|---|
| REL-PRE-1 | require | `planogram.working is not None and planogram.working.as_of == effective_date and cutoff_date <= effective_date` |
| REL-PRE-2 | require | `kind == CYCLE ⇒ effective_date in reset_calendar` (PGV-INV-2) |
| REL-PRE-3 | require | no version with overlapping scope is still `RELEASED` (PGV-INV-4). Revisions made meanwhile stay in `working` and queue for the next release. |
| REL-PRE-4 | require | the working plan is still current: it equals `generate_plan(working.scope, working.as_of)` except for agreements |
| REL-1 | ensure | `result == len(old(planogram.versions)) + 1`; the new version's `scope`, `entries`, `home` equal the working plan's; `kind`, `cutoff_date`, `effective_date` are as given; `status == RELEASED` |
| REL-2 | ensure | `planogram.working == old(planogram.working)` and nothing else changes |

**`execute_version(version_no, on)`** — executes a reset. A full-store reset is a version whose scope is all roots. A partial reset `[TB 1.6]` is a version over fewer nodes. `[VS 4, 10]`

Let `V = planogram.versions[version_no - 1]` and `Sc = {s for s in skus.values() if any(s.category.is_within(n) for n in V.scope)}`.

| ID | Kind | Clause |
|---|---|---|
| EXE-PRE-1 | require | `V.status == RELEASED and V.effective_date == on` |
| EXE-PRE-2 | require | `V`'s entries are feasible in the current state (PLAN-INV-4 evaluated against the placements now in the model) |
| EXE-PRE-3 | require | every non-`None` `e.agreement` in `V.entries` is in `agreements` |
| EXE-1 | ensure | **Replacement.** For every `s in Sc`, the `PERMANENT` placements of `s` live on `on` after the call are exactly one placement per entry of `V` for `s`. An old live placement equal to an entry on `(sku, space, extent, offset_in, agreement)` survives unchanged (same `placement_id`, `start_date`, `end_date is None`). Every other old live `PERMANENT` placement of an `s in Sc` has `end_date == on`. Each entry without a surviving match becomes a new `Placement` with a new unique id, `start_date == on`, `end_date is None`, `placement_type == PERMANENT`. |
| EXE-2 | ensure | every placement not covered by EXE-1 is unchanged: promotional, incidental, out-of-scope, and all ended ones |
| EXE-3 | ensure | `V.kind == CYCLE ⇒ all(n.last_reset_date == on for n in V.scope)`; a `REVISION` leaves every `last_reset_date` unchanged |
| EXE-4 | ensure | `V.status == EXECUTED`, and for every `s` placed by `V`, `home_placement(s, on)` is not `None` (HOME-2) |
| EXE-5 | ensure | **History preserved.** Every placement that existed at entry exists afterwards with identical attributes, except that `end_date` may have changed from `None` to `on` (PLC-INV-IMM) |

Executing a version is instantaneous on `on` in this version. A real reset runs over a night or several, sometimes staggered across stores for weeks `[VS 10]`. That is reserved (Part 10, Q-19).

**`add_agreement(agreement_type, vendor, fee_basis, fee_rate, window) -> str`**

| ID | Kind | Clause |
|---|---|---|
| AGC-PRE-1 | require | AGR-INV-1 through AGR-INV-5 hold for the arguments |
| AGC-1 | ensure | `result in agreements` with exactly these attributes; `len(agreements) == old(len(agreements)) + 1`; nothing else changes |

**`add_promotional_placement(sku_id, leaf_id, extent, offset_in, start_date, end_date, agreement_id) -> str`** — a time-boxed extra location, such as a promotional or vendor-funded display `[VS 3]`

| ID | Kind | Clause |
|---|---|---|
| PRM-PRE-1 | require | `sku_id in skus and skus[sku_id].promo_display_eligible` |
| PRM-PRE-2 | require | `space(leaf_id)` is a `LeafSpace` and `start_date < end_date` (PLC-INV-8, PLC-INV-9: time-boxed) |
| PRM-PRE-3 | require | `fits_dimensionally(sku.product, leaf)`, temperature match (PLC-INV-6), secured match (PLC-INV-11), and PLC-INV-4 and PLC-INV-5 for the arguments |
| PRM-PRE-4 | require | `leaf.can_hold(sku.product, extent, DateWindow(start_date, end_date))`; and for `LINEAR_IN` leaves, `[offset_in, offset_in + extent)` lies inside one interval of `leaf.free_intervals(DateWindow(start_date, end_date))`. So LS-INV-5 and LS-INV-6 will hold afterwards. |
| PRM-PRE-5 | require | `leaf.assigned_category is None or sku.category.is_within(leaf.assigned_category)` — promotional placements go on unassigned space (displays, end caps) or the SKU's own category space |
| PRM-PRE-6 | require | `agreement_id is None or (agreements[agreement_id].agreement_type == PROMOTIONAL_DISPLAY and window_covers(agreements[agreement_id].window, DateWindow(start_date, end_date)) and agreements[agreement_id].vendor == sku.manufacturer)` |
| PRM-1 | ensure | `result in placements` and `placements[result]` has exactly these arguments, with `placement_type == PROMOTIONAL` |
| PRM-2 | ensure | `len(placements) == old(len(placements)) + 1`, and nothing else changes |

A SKU needs no permanent placement first: promotional deals are a separate contract type from the baseline space agreement. `[VS 3]` `[SIM]` (Q-26)

**`add_incidental_placement(sku_id, units_held, start_date, end_date, zone_hint_id, agreement_id) -> str`**

| ID | Kind | Clause |
|---|---|---|
| INP-PRE-1 | require | `sku_id in skus`; INC-INV-1 through INC-INV-3 hold for the arguments; `zone_hint_id is None or` it names a space node |
| INP-1 | ensure | `result in incidentals` with exactly these arguments; `len(incidentals) == old(len(incidentals)) + 1`; nothing else changes |

**`move_incidental(placement_id, zone_hint_id)`**

| ID | Kind | Clause |
|---|---|---|
| MVI-PRE-1 | require | `placement_id in incidentals` and `zone_hint_id is None` or names a space node |
| MVI-1 | ensure | `incidentals[placement_id].zone_hint` is the named node (or `None`); nothing else changes |

**`end_placement(placement_id, on)`**

| ID | Kind | Clause |
|---|---|---|
| END-PRE-1 | require | `placement_id` names a placement or incidental placement, its `end_date is None`, and `on > start_date` |
| END-1 | ensure | its `end_date == on`; nothing else changes |

**`cancel_future_placement(placement_id, as_of)`** — the only deletion; it removes a placement that has not yet started

| ID | Kind | Clause |
|---|---|---|
| CAN-PRE-1 | require | `placement_id` names a placement or incidental placement with `start_date > as_of` |
| CAN-1 | ensure | it is gone from `placements` or `incidentals`, from its leaf and its SKU, and nothing else changes |

---

## Part 8 — Construction and the persistence boundary

### 8.1 Construction `[DbC]` `[VS 14]`

Every class above is created only through a constructor whose precondition is the conjunction of the class's own invariants over its arguments. The whole `Store` is built by one creation procedure, `Store.build(...)`, which takes the products, SKUs, all four hierarchies, agreements, placements, incidental placements, the reset calendar, and any previously executed planogram versions.

| ID | Clause |
|---|---|
| BUILD-PRE-1 | The arguments satisfy every invariant listed in Parts 2 through 7 (ST-INV-1 through ST-INV-8 and every class invariant they include), evaluated over the arguments **before any object is exposed**. This includes hierarchy cycles (HN-INV-3), overlapping placements (LS-INV-6), over-subscribed leaves (LS-INV-5), and placements whose end precedes their start (PLC-INV-8). Invalid source data is a precondition violation of `build`, never silently repaired. |
| BUILD-1 | ensure ST-INV-1 through ST-INV-8 |

### 8.2 Persistence boundary `[VS 5]` `[SIM]`

The contract does not specify storage. It constrains the model so that any storage works.

| ID | Clause |
|---|---|
| PERS-1 | Instance data is **seed data**. It is loaded once (through `Store.build`) to populate a real database, after which the simulation reads and writes there. The source layout (workbook or otherwise) is outside this contract. |
| PERS-2 | Every command in 7.3 is atomic, so each maps to one transaction. Whatever the model holds in memory is a cache over the system of record, written through. |
| PERS-3 | Ids are assigned by the model (prefix plus a monotone number), never taken from a storage row order. Ids are stable across save and load. |
| PERS-4 | A rollup, filter, or subtree query may be implemented as a database query, provided its clause (RU-*, FLT-*, HN-Q-*) and HN-IMPL-1 still hold. |

---

## Part 9 — The contract by example `[VS 13, 14]`

The example usages are part of the specification. They are client code the model must be able to run. If the model cannot express one cleanly, the model is wrong. Positive usages are specification, invariants are guard rails, and the matrix proves the hierarchies behave. `[VS 14]`

### 9.1 The reference fixture `F`

Every expected value below is computed against this small fictional store. The implementation must ship it as a test fixture.

| Element | Definition |
|---|---|
| Store | `STORE-1`, footprint 1000.0 sq ft, reset calendar `[2026-01-05, 2026-07-06]`. Children: `AREA-SF` (`SALES_FLOOR`, 700.0), `AREA-BR` (`BACKROOM`, 200.0), `AREA-CO` (`CHECKOUT`, 100.0) |
| `AREA-SF` children | `GDL-1` (footprint 32.0), `FD-1` (footprint 6.0), `AREA-SF-AISLE` (`AISLE`, 662.0). 32 + 6 + 662 = 700 |
| `GDL-1` | `depth_in = 48`, one side `SID-1F` (`FRONT`), bays `BAY-1` and `BAY-2` of width 48 in (length 96 in; 96 × 48 / 144 = 32.0). Each bay has 4 runs, all `width_in = 48`, `depth_in = 18`, `clear_height_in = 14`. Run ids: `RUN-1-{T,E,M,B}` in `BAY-1`, `RUN-2-{T,E,M,B}` in `BAY-2` (top, eye, middle, bottom) |
| `FD-1` | `capacity_linear_in = 24`, `footprint_depth_in = 36`, `max_stack_height_in = None`, so `leaf_capacity = 24 × 36 / 144 = 6.0` sq ft |
| Category hierarchy | `CAT-DG` "Dry Grocery" (root; defines the full policy: `AMBIENT`, allowed types `{OPEN_SHELVING, FLOOR_DISPLAY}`, min 1, max 6 facings, elasticity 0.5, reset cycle 6 months, `VERTICAL_BRAND`) → `CAT-CER` "Cereal" → `CAT-RTE` "Ready-to-Eat" (leaf) |
| Business units | `BU-CO` (`COMPANY`) → `BU-ST` (`STORE`) → `BU-DG` (`DEPARTMENT`, `root_category = CAT-DG`) |
| Products | `PRD-A`: brand "Acme", manufacturer "Acme Foods", `BOX`, 4 × 8 × 6 in. `PRD-B`: same brand and manufacturer, 3 × 8 × 6 in. `PRD-C`: brand "StoreBrand", manufacturer "StoreCo", private label, 4 × 8 × 6 in. None stackable |
| SKUs | `SKU-A`, `SKU-B`, `SKU-C` on `PRD-A/B/C`, all in `CAT-RTE`. `SKU-A` and `SKU-B` are `promo_display_eligible` |
| Assignment | `assign_space("SID-1F", "CAT-CER")`. The 8 runs are assigned by inheritance |
| Agreements | `AGR-1` `PROMOTIONAL_DISPLAY`, vendor "Acme Foods", `PER_DISPLAY`, rate 500.0, window `[2026-08-25, 2026-10-06)`. `AGR-2` `SLOTTING`, vendor "Acme Foods", `PER_LINEAR_FT`, rate 2.0, window `[2026-01-05, None)` |
| Placements | `PLC-1`: `SKU-A` on `RUN-1-E`, extent 12.0 (3 facings × 4 in), offset 0, `PERMANENT`, start 2026-01-05, agreement `AGR-2`. `PLC-2`: `SKU-B` on `RUN-1-E`, extent 9.0 (3 facings × 3 in), offset 12.0, `PERMANENT`, start 2026-01-05, `AGR-2`. `PLC-3`: `SKU-C` on `RUN-2-E`, extent 16.0 (4 facings), offset 0, `PERMANENT`, start 2026-01-05, no agreement. `PLC-4`: `SKU-A` on `FD-1`, extent 1.0 sq ft (6 facings × 4·6/144), `PROMOTIONAL`, window `[2026-09-01, 2026-09-29)`, `AGR-1` |
| Incidental | `INC-1`: `SKU-B`, `units_held = 24`, start 2026-09-10, open-ended, `zone_hint = SID-1F`, no agreement |
| Planogram | Version 1: `CYCLE`, scope `{CAT-CER}`, effective 2026-01-05, cutoff 2026-01-02, entries = `PLC-1..3`'s data, each SKU's own entry as home, `status = EXECUTED` |

Derived: `PLC-1` has 3 facings and `units_capacity` 9 (3 facings × depth count `floor(18/6) = 3`). `PLC-2`: 3 facings, 9. `PLC-3`: 4 facings, 12. `PLC-4`: 6 facings, 6 (top plane, no stacking). Let `d0915 = 2026-09-15` and `d1015 = 2026-10-15`.

### 9.2 Worked usages

```python
store = fixtures.reference_store()

# Space: where is the room?
sid = store.space("SID-1F")
assert sid.capacity(LINEAR_IN) == 384.0
assert sid.allocated_amount(LINEAR_IN, d0915) == 37.0          # 12 + 9 + 16
assert sid.unallocated_amount(LINEAR_IN, d0915) == 347.0
fd = store.space("FD-1")
assert fd.allocated_amount(SQFT, d0915) == 1.0
assert fd.unallocated_amount(SQFT, d0915) == 5.0
assert fd.allocated_amount(SQFT, d1015) == 0.0                 # promotion has ended

# What was on this shelf on this date?
run = store.space("RUN-1-E")
assert {p.placement_id for p in run.placements_on(d0915)} == {"PLC-1", "PLC-2"}
assert run.placements_on(date(2025, 12, 1)) == frozenset()

# Facings are derived, never entered
assert store.placements["PLC-1"].facings == 3
assert store.placements["PLC-4"].facings == 6

# Home position falls out of the planogram, not a flag
assert store.home_placement(store.skus["SKU-A"], d0915) is store.placements["PLC-1"]
assert store.home_placement(store.skus["SKU-A"], date(2026, 1, 4)) is None

# Zero is legitimate: a SKU with no placements
assert store.skus["SKU-NEW"].placements == ()                  # in a fixture variant

# Unit-honest rollups: inches and square feet never add
assert store.capacity(LINEAR_IN) == 384.0 and store.capacity(SQFT) == 6.0 and store.capacity(HOOKS) == 0.0
assert store.space("AREA-BR").capacity(LINEAR_IN) == 0.0       # non-merchandising space
assert store.footprint_sqft == 1000.0
```

### 9.3 The query matrix: each hierarchy × each query shape `[VS 13]`

| Query shape | Space hierarchy | Category hierarchy | Business-unit hierarchy |
|---|---|---|---|
| **Roll up** | `sid.allocated_amount(LINEAR_IN, d0915) == 37.0` | `category("CAT-DG").facings(d0915) == 16` (3 + 3 + 4 + 6) | `business_unit_node("BU-CO").facings(d0915) == 16` |
| **Drill down** | `[c.id for c in store.space("GDL-1").children] == ["SID-1F"]`, and RU-1: `gdl.allocated_amount(LINEAR_IN, d0915) == sum(c.allocated_amount(LINEAR_IN, d0915) for c in gdl.children)` | `[c.id for c in category("CAT-DG").children] == ["CAT-CER"]`, and `category("CAT-CER").facings(d0915) == sum(c.facings(d0915) for c in category("CAT-CER").children)` | `[c.id for c in business_unit_node("BU-ST").children] == ["BU-DG"]`, and `BU-ST.units_capacity(d0915) == 36` (9 + 9 + 12 + 6) |
| **Filter by type** | `len(store.spaces_of_type(OPEN_SHELVING)) == 12` (gondola, side, 2 bays, 8 runs); `len(store.spaces_of_type(MERCHANDISING)) == 13` | `store.placements_in_category(category("CAT-CER")) & store.placements_of_type(PROMOTIONAL) == {PLC-4}` | `store.placements_in_unit(BU-DG) & store.placements_with_agreement_type(SLOTTING) == {PLC-1, PLC-2}` |
| **Filter by date** | `{p.placement_id for p in sid.placements_on(d1015)} == {"PLC-1", "PLC-2", "PLC-3"}` | `category("CAT-RTE").placement_count(d1015) == 3` (incidentals excluded) | `business_unit_node("BU-DG").placement_count(d1015) == 3` |
| **Allocated vs. unallocated** | `sid.unallocated_amount(LINEAR_IN, d0915) == 347.0`; `fd.unallocated_amount(SQFT, d0915) == 5.0` | `category("CAT-CER").unallocated_amount(LINEAR_IN, d0915) == 347.0`; `category("CAT-CER").allocated_amount(SQFT, d0915) == 0.0` (`FD-1` is not assigned to it) | `BU-ST.allocated_amount(LINEAR_IN, d0915) == 37.0` and `BU-ST.unallocated_amount(LINEAR_IN, d0915) == 347.0` |

Category `CAT-DG` and `CAT-CER` `capacity(LINEAR_IN)` are both `384.0`; `CAT-RTE` is `0.0` (no leaf is assigned exactly to it). The last cell of the category column shows the two families of measure: the promotional placement on the unassigned floor display counts in `facings` but not in `CAT-CER`'s space measures.

**Set-algebra tests** (no query language; plain Python sets):

```python
live_915 = store.placements_live_on(d0915)
assert live_915 == {PLC_1, PLC_2, PLC_3, PLC_4, INC_1}
assert store.placements_live_on(d1015) == {PLC_1, PLC_2, PLC_3, INC_1}          # PLC-4 ended
assert store.placements_live_on(date(2026, 9, 5)) == {PLC_1, PLC_2, PLC_3, PLC_4}   # INC-1 not started
assert live_915 & store.placements_of_type(PERMANENT) == {PLC_1, PLC_2, PLC_3}  # and = intersection
assert store.placements_in_space(sid) | store.placements_in_space(fd) == store.all_placements()  # or = union
assert store.placements_live_on(d1015) - store.placements_of_type(INCIDENTAL) == {PLC_1, PLC_2, PLC_3}
assert store.placements_in_space(sid) == {PLC_1, PLC_2, PLC_3, INC_1}           # INC-1 via zone_hint
assert store.placements_of_type(PROMOTIONAL) == {PLC_4}
assert store.placements["PLC-4"].vendor_funded and not store.placements["PLC-3"].vendor_funded
```

### 9.4 Allocation usage

```python
scope = frozenset({store.category("CAT-CER")})
plan = store.generate_plan(scope, date(2026, 7, 6))          # a query: the store is unchanged
assert plan.unplaced == {}                                    # all three SKUs fit
assert {e.sku.sku_id for e in plan.entries} == {"SKU-A", "SKU-B", "SKU-C"}
store.set_working_plan(plan)
v = store.release_working_plan(CYCLE, cutoff_date=date(2026, 7, 1), effective_date=date(2026, 7, 6))
store.execute_version(v, on=date(2026, 7, 6))                 # all old PERMANENT placements end 2026-07-06,
                                                              # unchanged entries keep their placement ids
assert store.home_placement(store.skus["SKU-A"], date(2026, 7, 6)) is not None
```

### 9.5 The negative cases: things that must raise `[VS 14]`

A model that cannot be made to fail correctly usually cannot be trusted when it succeeds. Each row is a required test. "Raises" means the named exception carries the named clause ID at monitoring level `ALL`.

| # | Action on fixture `F` (or a variant) | Raises | Clause |
|---|---|---|---|
| N-1 | Build a store where `SID-1F.parent` is `BAY-1` (a cycle) | `PreconditionViolation` | BUILD-PRE-1 / HN-INV-3 |
| N-2 | `add_promotional_placement("SKU-A", "FD-1", 1.0, None, 2026-10-10, 2026-10-01, None)` — end precedes start | `PreconditionViolation` | PRM-PRE-2 / PLC-INV-8 |
| N-3 | Promote `SKU-B` on `FD-1` with extent 5.5 sq ft over a window overlapping `PLC-4` (1.0 + 5.5 > 6.0) — allocated extent exceeds capacity | `PreconditionViolation` | PRM-PRE-4 / LS-INV-5 |
| N-4 | Promote `SKU-B` on `RUN-1-E`, offset 10.0, extent 6.0, over any window — overlaps `PLC-1`'s segment `[0, 12)` on the same dates | `PreconditionViolation` | PRM-PRE-4 / LS-INV-6 |
| N-5 | Build a store with two overlapping placements on one run whose windows intersect | `PreconditionViolation` | BUILD-PRE-1 / LS-INV-6 |
| N-6 | Break one child's `allocated_amount` (test double) so the children no longer sum to `GDL-1` | `PostconditionViolation` | RU-1 |
| N-7 | Promote a `HANGING` product on `RUN-1-M` (a front-plane shelf) | `PreconditionViolation` | PRM-PRE-3 / PLC-INV-3 |
| N-8 | `assign_space("SID-1F", <a FROZEN category>)` | `PreconditionViolation` | ASN-PRE-4 / LS-INV-7 |
| N-9 | `assign_space("BAY-1", "CAT-RTE")` while `SID-1F` is assigned | `PreconditionViolation` | ASN-PRE-2 / SN-INV-2 |
| N-10 | `execute_version(v, on=<any date other than v.effective_date>)` | `PreconditionViolation` | EXE-PRE-1 |
| N-11 | Set `placements["PLC-1"].extent = 3.0` | `InvariantViolation` (or a frozen-instance error) | PLC-INV-IMM |
| N-12 | `end_placement("PLC-1", on)` twice | `PreconditionViolation` | END-PRE-1 |
| N-13 | Build a `PERMANENT` placement with start 2026-03-01, which is not in `reset_calendar` and no revision version exists | `PreconditionViolation` | BUILD-PRE-1 / PLC-INV-14 |
| N-14 | `release_working_plan(CYCLE, …, effective_date=<not in reset_calendar>)` | `PreconditionViolation` | REL-PRE-2 / PGV-INV-2 |
| N-15 | Release a second version over `CAT-CER` while the first is still `RELEASED` | `PreconditionViolation` | REL-PRE-3 / PGV-INV-4 |
| N-16 | Build a produce `Bin` with two placements whose windows overlap | `PreconditionViolation` | BUILD-PRE-1 / BIN-INV-2 |
| N-17 | Build an `Area` whose children's footprints differ from its own by more than the tolerance | `PreconditionViolation` | BUILD-PRE-1 / AREA-INV-2 |
| N-18 | `generate_plan(frozenset(), d)` | `PreconditionViolation` | GEN-PRE-1 |

Runtime-only checks (cycle detection, date overlap, over-subscription) need the running model. The implementation is delivered with them as tests, alongside the matrix in 9.3. `[VS 14]`

---

## Part 10 — Reserved areas (not yet specified)

Nothing in this part is part of the model yet. Each entry lists what the area will need from Parts 2–9, so that later versions extend the contract instead of reopening it (evolution rule, 0.1). Every listed dependency is a feature this version already exposes.

| Area | Will add | Depends on (already specified) |
|---|---|---|
| **Simulation clock** | A `Clock` with a current date and a `advance(to)` command. Replaces the explicit `on:` / `as_of:` arguments as the default, which stay as overrides. | Half-open windows (rule 10), `Store.change_dates()` (ST-Q-4), `reset_calendar` |
| **Staggered reset execution** | Execution of a `RELEASED` version over several nights, and across several stores. Replaces the instantaneous `execute_version` of 7.3 with a multi-step process. Q-19. | `PlanogramVersion` freeze and status (PGV-*), EXE-1 (the end state it must reach), REL-PRE-3 (queued revisions) |
| **Observed shelf state and compliance** | A third artefact: what is actually on the shelf per leaf and offset, plus a compliance measure against `current_version`. Execution drift versus intended revision, per `[VS 10]`. | `PlanogramVersion`, `Planogram.current_version` (PGV-Q-1), `Placement` and `LeafSpace` geometry; the invariant that no placement or version records observed state (6.5) |
| **Product restocking** | Shelf inventory per placement (units on shelf against `units_capacity`), replenishment triggers, and work orders from backroom to shelf. Each placement's `units_capacity` is the shelf-side ceiling. | `Placement.units_capacity` (PLC-Q-3), `home_placement` (HOME-1: replenishment defaults to the home position), `respace_without_reset`, `IncidentalPlacement.units_held` |
| **Product reordering** | Reorder points and order quantities, driven by velocity and inventory position. | `SKU.sales_velocity_units_per_week`, `Product.units_per_case`, `Product.case_dims_in`, `SKU.assortment_status` |
| **Product purchasing from vendors** | Purchase orders, vendor terms, and cost. Agreements attach here as well: `OPPORTUNISTIC_BUY` (a deal on the product, not the shelf). | `Agreement` (5.1), `SKU.unit_cost`, `Product.manufacturer` |
| **Product receiving** | Receipt against purchase orders, cold-chain checks at the dock, stock into backroom or onto the shelf. DSD vendor-managed space (Q-24). | `TemperatureZone`, `NON_MERCHANDISING` backroom and prep space (2.2), `IncidentalPlacement` cold-chain rules (deferred, 5.3), `Product.case_dims_in` |
| **Sales to store customers** | Sales transactions per SKU per day, drawing down shelf inventory. Revenue, gross margin, and sales per linear foot become live measures. | Reserved measures in 2.3 (must be additive under RU-1), `SKU.retail_price`, `SKU.unit_margin`, `Placement` as the leaf fact that revenue attaches to `[VS 9]` |
| **Assortment changes** | Commands that add, trial, and delist SKUs at reset windows `[TB 2.6]`. Q-18. | `AssortmentStatus`, `PlacementPlan.delist_review`, `SKU-ZERO` |
| **Hierarchy maintenance** | Commands to re-parent, add, and remove nodes in the four hierarchies. Q-14. | `HierarchyNode` (HN-*), HN-IMPL-1, the rule that a node with placements cannot vanish |
| **Multi-store and localized planograms** | More than one `STORE` business unit; planogram mods per store `[TB 2.2]`. Q-23. | BU-INV-3 (exactly one store today), `BusinessUnit` hierarchy |

**Constraints on the extension.** These follow from the design rules and bind every later section:

1. New state that changes over time is recorded with effective dates, as a placement is, so that "what was true on date d" stays one uniform query. `[VS 4]`
2. New measures must be additive and must be declared as `Rollup` measures, with unit or unit-free semantics stated (RU-4).
3. New space attributes go on the space class of the type they belong to, not on a shared base class (rule 5).
4. Money attaches to agreements and transactions, never to space (rule 8).
5. Every new command is atomic (7.3) and has a negative case in the style of 9.5.

---

## Part 11 — Change log, v0.1 → v0.2

### 11.1 ID stability

v0.1 was a same-day draft, written before the app-functionality file was available. **Clause IDs are stable from v0.2 on** (0.1). Several v0.1 IDs are reused in v0.2 with a different meaning, listed below. Only the v0.1 contract and its v0.1 companion cite v0.1 IDs, and both are superseded, so v0.1 is withdrawn rather than a prior release.

### 11.2 Retired or changed v0.1 clauses

| v0.1 | v0.2 | Change | Reason |
|---|---|---|---|
| `PlacementType` `PRIMARY`, `SECONDARY` | `PERMANENT`, `PROMOTIONAL`, `INCIDENTAL` (Part 1) | Renamed and a third value added | `[VS 3]`. "Primary" is now derived (HOME-1), not a type |
| SKU-INV-7 (at most one PRIMARY; `primary_placement`) | SKU-ZERO; HOME-1, HOME-2 | Removed | `[VS 2]`: one-to-many, zero legitimate, home position falls out of the planogram |
| PLC-INV-8 (primary is open-ended) | PLC-INV-14 | Reversed | `[VS 4]`: permanent placements carry an effective end date, set at the next reset |
| PLC-INV-9 (`start <= end`) | PLC-INV-8 (`start < end`) | Strengthened | Half-open windows (rule 10) |
| `Placement.facings` (stored), `space_consumed` | `Placement.extent` (stored), `facings` (derived; PLC-Q-1) | Inverted | `[VS 8]` |
| PLC-INV-2 (`space_consumed == consumption(...)`) | PLC-Q-1, PLC-INV-4 | Replaced | As above; whole facings for boxes |
| PLC-INV-3 (shelf level and fixture type entered, then checked) | PLC-Q-2 (derived queries) | Made derived | Uniform Access; `[VS 8]` |
| PLC-INV-7 (primary space belongs to the SKU's category exactly) | PLC-INV-7 (category or an ancestor) | Weakened precondition | Assignment is now at any node (ASN-*, Q-2) |
| PLC-INV-10 (secondary needs eligibility) | PLC-INV-9 | Extended | Promotional placements are also time-boxed |
| PLC-INV-11 (min/max facings; bin facings == 1) | PLC-INV-10, PLC-INV-4 | Reworded | Bins are allocated whole; `LOOSE` products are exempt |
| Placements removed on `apply_plan` | PLC-INV-IMM, EXE-1, EXE-5 | Removed as an operation | `[VS 4]`: full history is kept, no archive table. Resolves v0.1 Q-15 |
| LS-INV-3, LS-INV-4 (leaf allocated, current-state only) | LS-Q-1, LS-INV-5 (checked at every change date) | Time-aware | `[VS 4]`, `[VS 14]` |
| LS-INV-7 (interval overlap, current-state) | LS-INV-6 (overlap in space **and** time) | Strengthened | `[VS 14]` |
| LS-INV-5, LS-INV-8, LS-INV-9 (temperature, category temperature, secured) | LS-INV-7, LS-INV-8 | Renumbered | Type tree (TYP-2) replaces the fixture-type property |
| SKU package dimensions (SKU-INV-2, SKU-INV-5) | PROD-INV-1, PROD-Q-2, `Product.width_in`, `height_in`, `depth_in` | Moved to Product | `[VS 1, 8]` |
| `ShelfOrientation` (`UPRIGHT`, `LAY_DOWN`, `STACKED`, `HANGING`) | `ProductForm`, `PresentationPlane`, `Product.stackable` | Dropped | Plane is a property of the space `[VS 8]`; the product presents its front or top face |
| `UnplacedReason.INCOMPATIBLE_ORIENTATION` | `NO_SPACE_IN_PLANE`, `NO_DIMENSIONAL_FIT` | Replaced | Hanging products are now placeable on `PEG` planes. **Reverses v0.1 Q-13's default** |
| ENUM-1 (fixture archetype), ENUM-2 (fixture type → temperature) | Part 2.2 (type tree), TYP-2 | Replaced | Space-type tree `[VS 6]` |
| ENUM-3 (shelf level rank) | ENUM-2 | Renumbered. v0.2's ENUM-3 is now the agreement/placement compatibility table | — |
| SN-INV-1 … SN-INV-5 (v0.1 physical tree) | HN-INV-1..3 (tree shape), RU-3 (bounds), SN-INV-3 (fixture parts share a type) | Split between `HierarchyNode` and `SpaceNode` | `[VS 6]` |
| `FloorDisplay.capacity_linear_in` as the leaf capacity | FD-INV-1 (`leaf_capacity` in sq ft) | Changed | `[VS 8]`. Resolves v0.1 Q-9 |
| `SpaceRollup` (assignment relation, C-4) | `CategoryNode` conforming to `HierarchyNode` and `Rollup` (4.3, Part 2) | Replaced | `[VS 6, 9]`: category is a true hierarchy; rollups are one machinery |
| Category flat attributes (`space_elasticity`, `min/max facings`, …) | `CategoryPolicy`, inherited through ancestors (CAT-Q-1) | Restructured | `[VS 6]` |
| `assign_space` at leaf grain only (v0.1 Q-2) | `assign_space` at any merchandisable node, inherited by descendants (ASN-*, SN-INV-2) | Broadened | Default of v0.1 Q-2 reversed |
| `apply_plan` (APP-1..4) | `set_working_plan`, `release_working_plan`, `execute_version` (WRK, REL, EXE) | Split into three commands | `[VS 10]`: the version executed and the version edited are different objects |
| `remove_placement` (RMV-*) | `end_placement`, `cancel_future_placement` | Replaced | History is kept (PLC-INV-IMM) |
| `add_secondary_placement` (SEC-PRE-1..5, SEC-1, SEC-2) | `add_promotional_placement` (PRM-PRE-1..6, PRM-1, PRM-2) | Renamed and reworked | `[VS 3]`. SEC-PRE-2 (needs a primary placement first) is dropped (Q-26). Agreement replaces the `vendor_funded` flag |
| `expire_placements` (EXP-1, EXP-2) | Removed | Ended placements stay in the model as history (PLC-INV-IMM) | `[VS 4]` |
| ST-Q-1 … ST-Q-4 | ST-Q-1 … ST-Q-8, FLT-1 … FLT-7 | Extended | `[VS 13]` |
| GEN-PRE-2, GEN-5 … GEN-8 | Same IDs, restated over pools and planes | Reworded | Pool node replaces the SKU's exact category (6.1) |

### 11.3 New in v0.2 (no v0.1 counterpart)

`HierarchyNode` and its four instances (Part 2); space-type tree (2.2); `Rollup` and additive measures (2.3); `Area` and footprint reconciliation (3.3); Gondola sides, end caps, `PegSection`, `ClipStrip` (3.4); presentation plane and derived unit; `Product` (4.1) and the Product/SKU split; `BusinessUnit` (4.4); `Agreement` (5.1); `IncidentalPlacement` (5.3); `PlanogramVersion`, `Planogram`, HOME-* (6.5); brand blocking BLK-3, BLK-4; `PlacementPlan.block_breaks`, `adjacency_breaks`, `home`; filters FLT-1..7; `Store.build` and BUILD-* (Part 8); the reference fixture, query matrix, and negative cases (Part 9); reserved areas (Part 10).

### 11.4 Grounding appendix: functionality-file section → clauses

| `[VS n]` | Topic | Clauses |
|---|---|---|
| 1 | Product vs SKU | 4.1, 4.2, PROD-INV-3, SKU-INV-10 |
| 2 | Cardinality; home position | SKU-ZERO, HOME-1, HOME-2, PLC-INV-IMM |
| 3 | Agreement and placement types | ENUM-3, 5.1, 5.3, AGR-*, INC-*, rule 8 |
| 4 | Effective dating; resets; no archive | Rule 10, PLC-INV-8, PLC-INV-14, PLC-INV-IMM, CAT-INV-5, CAT-Q-3, EXE-* |
| 5 | Persistence | PERS-1..4 |
| 6 | Four hierarchies; footprint | Part 2, AREA-INV-*, ST-Q-5, ST-Q-6, 4.3, 4.4, HN-IMPL-1 |
| 7 | Gondola faces, end caps, peg, clip strip | 3.4, PEG-INV-*, CLP-INV-*, rule 8 |
| 8 | Plane, facings, dimensions, units | ENUM-1, 3.2 fit, PROD-Q-*, PLC-Q-1, RU-4 |
| 9 | Rollups (the cube) | 2.3, RU-1..4, 9.3 |
| 10 | Planogram versions, freeze, drift | 6.5, PGV-*, REL-*, EXE-*; observed state reserved (Part 10) |
| 11 | Paper shelf tags | Not modeled (no clause needed); see companion T-7 |
| 12 | Shelf level value; blocking; private label | ENUM-2, POL-4, GEN-8, 3.6, BLK-3, BLK-4 |
| 13 | Queries as set operations | Rule 9, 7.2 filters, FLT-7, 9.3 |
| 14 | DbC frame; negative cases | Part 0, BUILD-*, 9.5 |
