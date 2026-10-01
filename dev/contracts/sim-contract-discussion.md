# Contract v0.3 Discussion: Decisions, Conflicts, Open Questions, and Workbook Flags

**Companion to:** `sim-contract.md` v0.3 (Merchandising Space Allocation → SKU Placements)
**Date:** 2026-10-01
**Replaces:** the v0.2 companion. T-, X-, Q-, and E- numbers are kept from v0.2; new items are marked **(new)**. The v0.2 carry-forward part (CF-1 … CF-13) is resolved in Part 0 and no longer exists.

Tags are as in the contract: `[TB x.y]` textbook, `[VS n]` the app-functionality file, `[DISC]` the merchandising-space discussion, `[SIM]` a decision made in the contract.

| Part | Content | IDs |
|---|---|---|
| 0 | Decisions to check first: every choice this run made on its own | D-1 … D-32 |
| A | Where the textbook is at odds with the app-functionality file | T-1 … T-11 |
| B | Where the functionality file is at odds with itself, or with `[DISC]` | X-1 … X-10 |
| C | Questions for future functionality discussions, each with the contract's current default | Q-1 … Q-35 |
| D | Check of the extract's four unverified workbook claims (unchanged from v0.2) | — |
| E | Cross-workstream flags | E-1 … E-21 |

A note on Parts D and E: Scheme 2's model data are to come from a generator run at app initialization, not from the workbook. The workbook flags stand as written, and each applies equally to the generator: it must produce data that satisfies `Store.build`'s preconditions.

---

## Part 0 — Decisions to check first

Each item says what was decided, why, and what it would cost to reverse. A "yes" to every item needs no reply.

### Carry-forward items (CF-1 … CF-13 of the v0.2 companion)

**D-1. Gondola orientation: long sides face LEFT and RIGHT; end caps are FRONT and BACK. (new)** Adopted from CF-1. `[DISC]` names the four aisles from a viewer at the front of the store facing the rear, and in a grid layout `[TB 1.5]` gondolas run front to back, so the long faces border the left and right aisles and the ends border the front and back aisles. v0.2 had the reverse. Changed: 3.4 prose, GDL-INV-1 … GDL-INV-5, X-5, Q-8, Q-16. The Scheme 2 workbook uses v0.2's reading (E-17). Reversing this means swapping two pairs of names in five clauses.

**D-2. `generate_plan` has exactly one correct answer (GEN-12, `canonical_plan`). (new)** Adopted from CF-2 and required by the revised prompt ("deterministic allocation"). Restated over v0.2's model: pools, planes, extents, `room`, and `consumption`, instead of the retired draft's `assigned_spaces` and bin special case. The procedure tries target facings first, then fewer; within one facing count, leaves in shelf-level order, then leaf id; the first free gap that fits. GEN-1 … GEN-8, GEN-11, and GEN-13 stay as checkable properties; an independent simulation of the fixture reproduced the canonical plan exactly. Cost: a better allocation algorithm (optimization, look-ahead) is now a contract change, not an implementation choice. Supersedes Q-21. See Q-27 for the facings-versus-level order.

**D-3. One allocation order across all pools: descending score, ties by `sku_id`. (new)** The retired draft processed categories in key order, then SKUs by score within each. With v0.2's pools, SKUs of sibling categories can share a pool (Ready-to-Eat and Hot Cereal share Cereal's shelves in the fixture), so a per-category order would let a weak Ready-to-Eat SKU take space before a strong Hot Cereal one. A single score order treats a pool's competitors evenly and is the same for every pool. See Q-33.

**D-4. Scarcity priority (GEN-13). (new)** Adopted from CF-3 under a new ID (v0.2's GEN-11 is home designation). Restated with `room(w, without=t)` and `consumption`. A higher scorer is never left out so that a lower scorer can stay.

**D-5. Unplaced reasons are exclusive, in a stated precedence (ENUM-5, GEN-4). (new)** Adopted from CF-4. Order: `OUT_OF_SEASON`, `NO_CATEGORY_SPACE`, `NO_SPACE_IN_PLANE`, `NO_DIMENSIONAL_FIT`, `INSUFFICIENT_SPACE`. The enum is declared in that order. The draft's `INCOMPATIBLE_ORIENTATION` is not reintroduced; v0.2's `NO_SPACE_IN_PLANE` covers it.

**D-6. Delist review: bottom decile by `floor(n / 10)`, ranked within the SKU's leaf category (PLAN-INV-6). (new)** Adopted from CF-5. `floor` means a category with fewer than 10 in-scope SKUs flags only explicit `DELIST_CANDIDATE`s; rounding up would flag the weakest SKU of every small category, including a category's only SKU. Ranked within the leaf category rather than the pool, because assortment reviews are run per category `[TB 2.6]`. See Q-34.

**D-7. A category's allowed space types must match its temperature zone (CAT-INV-9). (new)** Adopted from CF-6, restated over the type tree. So a refrigerated category can't list `DOOR_CASE` (which includes the frozen reach-in) and must list `REACH_IN_COOLER`. Consequence: ASN-PRE-4 is now implied by ASN-PRE-3 and has no independent counter-example.

**D-8. Floor displays are never category space (CAT-INV-10). (new)** Adopted from CF-7. Closes v0.2's gap where `assign_space` could give a promotional-only display to a category. A category may not allow `MERCHANDISING` as a whole either, since that includes floor displays.

**D-9. Physical order and positive dimensions of fixture parts. (new)** Adopted from CF-8: ENUM-6 (`physical_order`), BAY-INV-3, BAY-INV-4, DOR-INV-4 (new, same rule for doors), RUN-INV-2, SVC-INV-2, TIER-INV-2.

**D-10. `add_months` is defined (ADDM-PRE-1, ADDM-1). (new)** Adopted from CF-9. A month-end date stays at month end (31 Aug + 6 → 28 Feb).

**D-11. CF-10 (leaf order when a category mixes units) dropped. (new)** The case cannot arise: a pool is single-plane, every `FRONT` leaf has a shelf level (LS-INV-11, D-17), and no `TOP` or `PEG` leaf has one. POL-4's `level_preference(None) == 0` stays.

**D-12. CF-11 (season as a set of months) dropped as a representation; the question carried as Q-28. (new)** v0.2's `(first, last)` tuple, which may wrap the year, is kept; SKU-INV-11 (new) bounds its months. A set of months is more general, but nothing in the sources asks for a non-contiguous season.

**D-13. CF-12 (facings across bay boundaries) dropped as a clause; carried as Q-29. (new)** Already structural: a placement has one `space` leaf, and PLC-INV-5 keeps it inside the leaf.

**D-14. CF-13 (grain of `secured`) dropped as a clause; carried as Q-30. (new)** Already per leaf in v0.2. D-27 extends it to every leaf class.

### Defects found in v0.2

**D-15. SN-INV-1 corrected. (new)** v0.2 said non-merchandising space holds no allocatable leaf. Since `SALES_FLOOR`'s and the store's `merchandisable` is `False`, the clause was false for the store, the sales floor, and every checkout area holding an impulse rack. It now applies to `NON_MERCHANDISING` types other than `CHECKOUT`.

**D-16. GEN-6 corrected. (new)** v0.2 required no free block adjacent to an under-target placement. With whole facings, a gap smaller than one facing can always remain, so no plan could meet it. It now requires no adjacent gap of at least one facing.

**D-17. LS-INV-11: every front-facing leaf has a shelf level. (new)** v0.2 only said a non-front leaf has none. Needed for D-11 and for a total level order in pools.

**D-23. `target_facings` takes `as_of`. (new)** v0.2's POL-2 used `as_of` without receiving it. The signature is now `target_facings(sku, in_scope, as_of)`, with POL-PRE-1 requiring an in-season SKU of the scope; `+ EPS` added inside `floor` against float error.

**D-31. REL-PRE-3 also requires the new version's effective date to follow every overlapping version's. (new)** v0.2's precondition allowed a release that PGV-INV-4 then forbade.

### Testability requirements of the revised prompt

**D-18. One id prefix per class; created records numbered `PREFIX-nnnnnn`. (new)** v0.2 gave some classes several prefixes and some prefixes patterns (`…-DOR-`). New prefixes: `SHR-`, `DCR-`, `DOR-`, `SVC-`, `TIER-`, `TBL-`, `BIN-` (ID-1). Agreement, placement, and incidental ids are six-digit and sequential, never reused, continuing after the largest seed id (ID-2, BUILD-3, CAN-1). Workbook ids differ (E-18).

**D-19. Creation procedures are bottom-up; records are created detached. (new)** Every class has a constructor with its own preconditions, grouped so each has one counter-example. Containers take finished, unattached children, so no hierarchy can contain a cycle (HN-INV-3 holds by construction; N-1 now tests re-attachment). `ShelfRun`, `Bay`, `Door`, and `Tier` take `space_type` as an argument, because they serve several fixture types and a detached part has no parent to inherit from. Agreements, placements, and incidentals are detached until `Store.build` or a store command attaches them; invariants that need the store are marked (A).

**D-20. Monitoring API design (MON-1 … MON-10). (new)** Modules `supermarket_sim.model`, `supermarket_sim.contracts`, and `supermarket_sim.predicates`. Levels `NONE < REQUIRE < ENSURE < INVARIANT < ALL`, with `ALL` adding a deep check of every contained object after each store command. Postcondition predicates have the signature `P(self, result, old, **args)`; for a creation procedure `self` is the new object; for a module function `self` is `None`. A failed postcondition restores `old` before raising.

**D-21. Frame conditions use one snapshot and `changed(old, new)` (FRM-1 … FRM-3). (new)** The snapshot holds only mutable state; hierarchy structure, products, SKUs, and the calendar are static (Q-14) and left out. Every command's "nothing else changes" is now a `changed(…) <= K` clause.

**D-22. CAT-INV-8 retired; replaced by CAT-REV-1 (a review clause). (new)** It had no checkable expression, and MON-8 requires a predicate for every invariant.

**D-24. New placements of an executed version get consecutive ids in entry order (EXE-6). (new)** Needed so examples can name `PLC-000003` exactly.

**D-25. `HierarchyNode.root()` (HN-Q-6) replaces the undefined `path_root()`. (new)**

**D-26. The fixture EX-STORE-1 replaces v0.2's fixture `F`. (new)** Chosen so every class and every unplaced reason appears: a gondola with a long side, an end cap, a peg section, and a clip strip; a wall shelf; a reach-in cooler; a deli case; a produce table; two floor displays (one an impulse rack under `CHECKOUT`); a secured category (Health & Beauty) for ASN-PRE-5; and seven states, S0 … S6, covering two resets, a slotting renewal that lapses, a promotion, and overstock. All values are fictional.

**D-27. Every leaf class carries `secured` (default `False`). (new)** v0.2 listed it only for shelf runs, tiers, and floor displays while `LeafSpace.secured` applied to all leaves.

**D-28. `block_breaks` and `adjacency_breaks` are sorted. (new)** Needed for GEN-12's single result.

**D-29. BLK-4 kept as written although the fixture shows it flags a private label placed next to its benchmark in the same door. (new)** See Q-31.

**D-30. PLC-INV-12 also requires the agreement's vendor to be the SKU's manufacturer. (new)** AGR-INV-7 already required it from the agreement's side; stating it on the placement lets a detached placement be checked.

**D-32. Normativity is stated in 0.1. (new)** Clauses are the sole normative specification; Part 9 usages are required acceptance tests; Examples blocks are illustrative. Where Part 9 and a clause disagree, the clause governs and the disagreement is a defect to raise.

---

## Part A — Textbook vs. app functionality

Each item says what the textbook says, what the functionality file says, and how the contract resolves it.

### T-1. Slotting fees: upfront payment or ongoing rent

- **Textbook:** slotting fees are "upfront payments for placement" `[TB 2.4]`.
- **Functionality:** ongoing shelf space is effectively rented, priced by facings or linear feet, and renewed at reset `[VS 3]`.
- **Contract:** the ongoing-rent reading. `fee_rate` is for a term; `term_fee` scales with facings or linear feet (AGR-Q-1); slotting starts on reset dates (AGR-INV-2, AGC-PRE-2). An upfront payment is `FeeBasis.FLAT`. → **Q-20**

### T-2. Is a category a business unit

- **Textbook:** each category is "a discrete business unit" with its own sales, margin, and space targets `[TB 2.1]`.
- **Functionality:** category and business unit are separate recursive hierarchies `[VS 6]`.
- **Contract:** two hierarchies. A `DEPARTMENT` business unit points at one root category (BU-INV-1, BU-INV-2). Category targets stay on `CategoryPolicy`. Methodology note: E-14.

### T-3. Flat department/category vs. arbitrary depth

- **Textbook:** department and category are two flat levels `[TB 1.2, 2.1]`.
- **Functionality:** arbitrary depth for space, business unit, and category `[VS 6]`.
- **Contract:** arbitrary depth (HN-*). Policy is inherited from the nearest ancestor that defines it (CAT-Q-1).

### T-4. Position "down to the inch" vs. placements with no position

- **Textbook:** a planogram fixes shelf, facing count, and position, down to the inch `[TB 2.2]`.
- **Functionality:** an incidental placement has no fixed position `[VS 3]`; horizontal position carries almost no value `[VS 12]`.
- **Contract:** committed placements carry space, extent, and offset (PLC-INV-5); incidental placements carry none (INC-INV-4). Horizontal position is stored and checked for overlap (LS-INV-6) but never drives allocation (POL-4); the canonical procedure uses it only to pick the first free gap (GEN-12).

### T-5. Facing count: planned quantity or derived

- **Textbook:** the planogram specifies facing count `[TB 2.2]`; space is optimized in facings `[TB 2.3]`.
- **Functionality:** store allocated extent and presentation plane; derive facings `[VS 8]`.
- **Contract:** extent stored, facings derived (PLC-Q-1). The policy's target is still a facing count (POL-2), converted to extent by `consumption` (FIT-2).

### T-6. One planogram vs. versioned, frozen, and revised planograms

- **Textbook:** consistency across stores with local mods `[TB 2.2]`; full or partial resets `[TB 1.6]`; reviews at reset windows `[TB 2.6]`.
- **Functionality:** versions, frozen reset packs, revisions queued during a reset, resets that run over nights or weeks `[VS 4, 10]`.
- **Contract:** working plan vs. `PlanogramVersion` (6.5); one pending version per scope (PGV-INV-4, REL-PRE-3). Execution is instantaneous in this version. → **Q-19**

### T-7. Shelf tags: electronic or paper

- **Textbook:** electronic shelf labels `[TB 8.4]`.
- **Functionality:** paper tags batch-printed at reset, as a deliberate sim assumption `[VS 11]`.
- **Contract:** tags are not modeled. The batching consequence is kept: permanent placements start only at resets or released revisions (PLC-INV-14). Methodology note: E-14.

### T-8. Category captain and private label

- **Textbook:** a captain helps design the planogram in exchange for "a favored position" `[TB 2.4]`; private label has a space target `[TB 2.5]`.
- **Functionality:** private label sits immediately right of its benchmark; the retailer, not the captain, decides its placement `[VS 12]`.
- **Contract:** the captain has no effect (POL-5). Private-label adjacency is reported (BLK-4) and the target is reported, not asserted (CAT-Q-4). → **Q-7, Q-14, Q-22, Q-31**

### T-9. Department space targets and space shares

- **Textbook:** perishables use 30–40 % of *selling space* `[TB 1.2]`.
- **Functionality:** selling space is 60–70 % of the *floor* `[VS 6]`.
- **Contract:** two denominators, not a contradiction. `selling_share()` reports selling space over the building (ST-Q-6). Space is assigned to categories as input (ASN-*); no department target is generated. → **Q-2**

### T-10. Fixtures

- **Textbook:** no fixtures section; mentions locked cases `[TB 6.4]`, refrigerated display `[TB 4.4]`, and promotional visibility `[TB 7.3, 7.5]`.
- **Functionality:** gondola faces, end caps, peg sections, clip strips, floor displays, checkout racks `[VS 6, 7]`.
- **Contract:** the fixture structure comes from `[DISC]` and `[VS 7]` (E-15). End caps are the `FRONT` and `BACK` sides of a gondola (D-1). Checkout impulse racks are `FLOOR_DISPLAY`s under a `CHECKOUT` area (TYP-3). → **Q-16**

### T-11. Items carried over from v0.1, resolved

| Topic | Textbook | Functionality | Contract |
|---|---|---|---|
| Delisting | slow movers face review `[TB 2.3, 2.6]` | silent | the plan reports `delist_review` (PLAN-INV-6, D-6) but never delists. → **Q-18** |
| Cold chain | unbroken to the shelf `[TB 4.4]` | space types imply temperature | derived from type (TYP-2), enforced (LS-INV-7, PLC-INV-6, CAT-INV-9) |
| One store or chain | localized planograms `[TB 2.2, 7.2]` | one store | one store (BU-INV-3). → **Q-23** |
| Unit of space | sq ft for departments, linear feet for categories | linear vs. area by plane `[VS 8]` | per-unit measures, never mixed (RU-4) |

---

## Part B — Conflicts inside the functionality file (and against `[DISC]`)

| ID | Statement 1 | Statement 2 | Contract choice |
|---|---|---|---|
| X-1 | the presentation plane on the **placement** picks the dimensions `[VS 8]` | the plane is a property of the **space** node `[VS 8]` | Space. A mixed deck-and-riser case is two leaves and two placements |
| X-2 | the sim needs a primary flag `[VS 2]` | primary falls out of the planogram's home position `[VS 2]` | No flag. HOME-1 |
| X-3 | three parallel hierarchies `[VS 6]` | space type is its own hierarchy `[VS 6]` | Four hierarchies, one `HierarchyNode` |
| X-4 | gondola → face → shelf → segment `[VS 7]` | bay × shelf identifies the leaf `[VS 8]`; `[DISC]` has Gondola → Side → Bay → Shelf Run | `[DISC]`. A segment is an offset, not a node |
| X-5 | end caps are faces of the same gondola `[VS 7]` | gondola, endcap, cooler, floor display as sibling types `[VS 6]` | End caps are the `FRONT`/`BACK` sides of a gondola (D-1; v0.2 had `LEFT`/`RIGHT`). No `ENDCAP` type; a free-standing end display is a `FloorDisplay`. → **Q-8, Q-16, Q-32** |
| X-6 | a clip strip consumes a bit of the face's frontage `[VS 7]` | it is allocated and priced separately `[VS 7]` | Frontage recorded, not deducted (CLP-INV-2). → **Q-15, Q-35** |
| X-7 | peg placements may have a shorter life `[VS 7]` | permanent placements start at reset dates `[VS 4]` | A peg re-spacing is a released `REVISION` (PLC-INV-14); `respace_without_reset` marks where that is expected. → **Q-19** |
| X-8 | a reset is six to twelve months `[VS 4]` | three to twelve, by velocity; end open or set to the next reset `[VS 4]` | `reset_cycle_months >= 1` (CAT-INV-5); end dates set at the next execution (EXE-1) |
| X-9 | revenue attaches at the placement `[VS 9]` | a SKU may have several live placements, and sales are per SKU `[VS 2]` | Reserved (2.3). → **Q-9** |
| X-10 | slotting is rented space renewed at reset `[VS 3]` | pay-to-stay is a fee for holding space `[VS 3]` | Both are `PERMANENT` placements, differing by `AgreementType`. → **Q-20** |

---

## Part C — Questions for future functionality discussions

Each question gives the contract's current default in **bold**. A "yes" to every default needs no reply.

**Q-1. Authority of workbook versus contract.** **Default: the contract governs; the workbook (and the Scheme 2 generator) conform to it.**

**Q-2. Who assigns space to categories, and at what grain.** **Default: input data, at any merchandisable node, inherited downward; a SKU draws on the nearest ancestor category that owns space in its plane (POOL-2).**

**Q-3. SKU categories at leaves only.** **Default: yes (SKU-INV-9).**

**Q-4. Target-facings formula.** **Default: share proportional to `score ** (1 / (1 - e))` among the pool's in-season competitors, clamped to the category's facing bounds (POL-2).**

**Q-5. What performance means.** **Default: weekly gross-margin dollars (POL-1).** Switch to GMROI `[TB 2.3]` once inventory exists?

**Q-6. Permitted planes and forms.** **Default: `BOX` on FRONT or TOP; `SOFT_PACK` on FRONT (TOP with a per-square-foot override); `HANGING` on PEG; `LOOSE` on TOP.** Should bulky items be forced to the bottom shelf?

**Q-7. Category captain.** **Default: no effect (POL-5).**

**Q-8. Both long sides of a gondola have the same length (GDL-INV-3).** **Default: yes; the `LEFT` and `RIGHT` sides have the same bays.** Restated for D-1 (v0.2 said `FRONT` and `BACK`). Can a gondola lack one or both end caps? (The fixture's has only a `FRONT` one.)

**Q-9. Revenue across multiple placements.** **Default: reserved; when Sales arrives, a SKU's revenue goes to its `home_placement`.**

**Q-10. Floor displays hold promotional placements only (FD-INV-2).** **Default: yes.** CAT-INV-10 (D-8) now also keeps them out of category space.

**Q-11. Produce table temperature.** **Default: ambient (TYP-2).**

**Q-12. One SKU per bin at a time (BIN-INV-2).** **Default: yes; a bin is allocated whole.**

**Q-13. Hanging products currently on shelves.** **Default: a `HANGING` product goes only on `PEG` leaves.** The workbook has 425 hanging SKUs on shelving and displays (Part D).

**Q-14. Static hierarchies; objectives versus constraints.** **Default: no command changes any hierarchy; the private-label target and department mix are reported, not asserted.**

**Q-15. Clip strip frontage.** **Default: recorded, not deducted (CLP-INV-2).**

**Q-16. End caps and checkout racks.** **Default: end caps are the `FRONT`/`BACK` sides of a gondola (D-1); a checkout impulse rack is a `FLOOR_DISPLAY` under a `CHECKOUT` area.** Are there end caps not attached to a gondola?

**Q-17. Footprint tolerance.** **Default: 1.0 sq ft (AREA-INV-2).**

**Q-18. Assortment status.** **Default: `DELISTED` SKUs are out of scope; trials and candidates are allocated by score like `CORE`.**

**Q-19. Reset execution.** **Default: instantaneous on the effective date (EXE-1); a peg change is a `REVISION` like any other.**

**Q-20. Agreement semantics.** **Default: `fee_rate` covers the term; slotting and pay-to-stay differ only by type; `term_fee` is a query.** Where does trade funding `[TB 2.4]` go?

**Q-21. Contract strength versus algorithm freedom.** Superseded by D-2. Confirm D-2 or revert to property-only postconditions.

**Q-22. Blocking objectives.** **Default: breaks are reported, never required to be empty; vertical brand blocking by default.**

**Q-23. One store or a chain.** **Default: one store (BU-INV-3).**

**Q-24. DSD vendor-managed space.** **Default: ignored until Restocking and Receiving.**

**Q-25. Incidental placements.** **Default: `units_held` is a plain count; `zone_hint` is best effort; no capacity or cold-chain rule yet.**

**Q-26. Promotional placement without a permanent home.** **Default: allowed.**

**Q-27. Facings versus level priority. (new)** **Default: the canonical procedure keeps the most facings that fit before it seeks a better level (D-2).** In the fixture, SKU-A gets 6 facings on the eye-level shelf; had no 48 in eye-level gap existed, it would take 6 facings on the bottom shelf rather than fewer at eye level. `[TB 2.3]` rewards both without ranking them. Reverse the order?

**Q-28. Season granularity. (new)** **Default: a month range that may wrap the year (D-12).** Do you need weeks or dates (holidays), or a category-level season for `OCCASIONAL_SEASONAL` categories?

**Q-29. Facings across bay boundaries. (new)** **Default: no; a placement sits within one leaf (D-13).** In the fixture, SKU-A's target of 7 facings (56 in) can't be met on 48 in shelves for this reason. Allow a SKU to span an upright?

**Q-30. Grain of `secured`. (new)** **Default: per leaf (D-14, D-27).** Is secured space a whole fixture, a door, or a single shelf?

**Q-31. Private-label adjacency at bay granularity. (new)** **Default: BLK-4 as in v0.2, by bay or door position.** The fixture shows the consequence: the private-label yogurt sits on the shelf below the benchmark in the same door, and BLK-4 reports `DCR-1` as a break, because "immediately right" is measured in whole doors or bays. Should adjacency use the offset within a shelf, the shelf level, or both?

**Q-32. Gondolas that run side to side. (new)** **Default: not modeled; every gondola runs front to back (D-1).** Should a gondola along the front racetrack, whose long faces face front and back, be allowed?

**Q-33. Allocation order across categories that share a pool. (new)** **Default: one score order for all SKUs (D-3).** A pool shared by Ready-to-Eat and Hot Cereal goes to the higher scorers regardless of subcategory. Should each child category get a guaranteed minimum before score order applies?

**Q-34. Delist-review ranking basis. (new)** **Default: within the SKU's leaf category (D-6).** Rank within the pool (the space competitors) instead?

**Q-35. Where peg sections and clip strips sit on a face. (new)** **Default: they are children of the side and take no bay space.** In the fixture, `SID-1L`'s two bays fill its 96 in length and the peg section still exists beside them. Should a peg section replace a bay (take a bay position) or hang within one?

---

## Part D — The extract's four unverified workbook claims, checked

Unchanged from v0.2 (checked against `supermarket_operations_data_SCHEME2.xlsx`).

| # | Extract's claim | Result |
|---|---|---|
| 1 | Product and SKU are one-to-one `[VS 1]` | **Confirmed.** 5,990 rows each |
| 2 | Category fields are flat columns `[VS 9]` | **Confirmed.** No subcategory, segment, or parent-child structure |
| 3 | Business unit is not really there `[VS 9]` | **Confirmed.** `Department` is a text column only |
| 4 | No package dimensions `[VS 8]` | **Wrong.** `SKU Merchandising` has width, height, depth, orientation, and stackable; Product Master has none (E-3) |

Other findings (v0.2): placement types `Primary Shelf` 5,990, `Secondary/Impulse Display` 163, `Cross-Merchandised` 19; placement start dates follow first-listed dates, not resets; reset cycles of 20 and 24 months; no agreement, peg, clip-strip, end-cap, or version sheets; 414 hanging SKUs on gondola shelving and 11 on floor displays.

---

## Part E — Cross-workstream flags

None has been acted on.

**Workbook (Scheme 2) and generator: attributes the contract needs**

| ID | Gap | Contract clause |
|---|---|---|
| E-1 | Agreements (id, type, vendor, fee basis, rate, window) | 5.1 |
| E-2 | Placement extent and offset instead of facings and linear space | 5.2, PLC-Q-1 |
| E-3 | Package dimensions, form, non-box override, case dimensions on the product | 4.1 |
| E-4 | Peg sections, clip strips, end-cap sides, hook counts and depths | 3.4 |
| E-5 | `clear_height_in` and `secured` on every leaf (all leaf classes, D-27) | 3.2–3.5 |
| E-6 | Category → space-node assignments as data | ASN-*, SN-INV-2 |
| E-7 | Category hierarchy with policy fields per node; allowed space types must match temperature and exclude floor displays | 4.3, CAT-INV-9, CAT-INV-10 |
| E-8 | Business-unit hierarchy | 4.4 |
| E-9 | Planogram versions | 6.5 |
| E-10 | Reset calendar | ST-INV-6, PLC-INV-14 |
| E-11 | Whole-footprint `Area` rows that reconcile to the building | 3.3 |

**Workbook columns the contract makes derived or changes**

| ID | Item | Contract clause |
|---|---|---|
| E-12 | `Cross-Merchandised` has no counterpart among `PERMANENT`, `PROMOTIONAL`, `INCIDENTAL` | Part 1 |
| E-13 | Shelf level and fixture type on placements are derived; start dates must be reset or revision dates; 425 hanging SKUs need peg homes (Q-13) | PLC-Q-2, PLC-INV-14 |
| E-17 | **(new)** Gondola orientation. The `Gondola Sides` sheet gives each gondola `Front` and `Back` long sides, and the `Gondolas` sheet names its left and right aisles "(End)". The contract now has the reverse (D-1) | 3.4, GDL-INV-* |
| E-18 | **(new)** Id formats. The workbook uses `SID-00001` (five digits), `BAY-000001`, `RUN-`, `RIF-`, `RIC-`, `PRD-`, `DELI`, `BKY-`, and `FD-`. The contract has one prefix per class (ID-1) and six-digit numbers only for agreements, placements, and incidental placements (ID-2) | ID-1, ID-2 |
| E-19 | **(new)** Shelf runs, door runs, and tiers must be stored top to bottom, and bays and doors numbered 1 … n with no gaps | BAY-INV-4, DOR-INV-3, DOR-INV-4, SVC-INV-2, SID-INV-1 |

**Textbook and Methodology sheet**

| ID | Item |
|---|---|
| E-14 | Methodology & Sources: hierarchies of arbitrary depth (T-2, T-3); paper shelf tags (T-7); slotting as ongoing rent (T-1) |
| E-15 | A short fixtures section under textbook 2.2, or mark the fixture composition `[DISC]`/`[SIM]` in the Methodology sheet (T-10). If added, it should state the front-to-back gondola orientation of D-1 |
| E-16 | Textbook 2.4's "upfront payments" needs a note if the ongoing-rent reading stands (T-1) |
| E-20 | **(new)** The canonical allocation procedure (D-2) is a simulation construct. A short worked example under textbook 2.3 would give it a counterpart: high-velocity, high-margin items earn more facings and eye-level placement |
| E-21 | **(new)** Textbook 6.4 mentions locked cases for high-theft items. If the grain of `secured` changes (Q-30), the textbook sentence may need a matching note |
