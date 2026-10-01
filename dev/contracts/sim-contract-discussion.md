# Contract v0.2 Discussion: Conflicts, Open Questions, and Workbook Flags

**Companion to:** `sim-contract.md` v0.2 (Merchandising Space Allocation → SKU Placements)
**Date:** 2026-09-29
**Replaces:** the v0.1 companion. Its C-n and Q-n numbers are retired; the numbering below is the only one the v0.2 contract cites.

Tags are as in the contract: `[TB x.y]` textbook, `[VS n]` the voice-session extract (the app-functionality file), `[DISC]` the merchandising-space discussion in project memory, `[SIM]` a decision made in the contract.

**How this file is organized**

| Part | Content | IDs |
|---|---|---|
| A | Where the textbook is at odds with the app-functionality file | T-1 … T-11 |
| B | Where the functionality file is at odds with itself, or with `[DISC]` | X-1 … X-10 |
| C | Questions for future functionality discussions, each with the contract's current default | Q-1 … Q-26 |
| D | Check of the extract's four unverified workbook claims | — |
| E | Cross-workstream flags (workbook Scheme 2 and textbook) | E-1 … E-16 |

---

## Part A — Textbook vs. app functionality

Each item says what the textbook says, what the functionality file says, and how the contract resolved it. Where the resolution needs your confirmation, a question number points to Part C.

### T-1. Slotting fees: upfront payment or ongoing rent

- **Textbook:** slotting fees are "upfront payments for placement" `[TB 2.4]`.
- **Functionality:** ongoing shelf space is effectively rented, priced by facings or linear feet, and renewed at reset `[VS 3]`.
- **Contract:** the ongoing-rent reading. `Agreement.fee_rate` is for a term, `term_fee(p)` scales with the placement's facings or linear feet (AGR-Q-1), and slotting windows start on reset dates (AGR-INV-2). An upfront one-time payment is `FeeBasis.FLAT`. → **Q-20**

### T-2. Is a category a business unit

- **Textbook:** each category is "a discrete business unit" with its own sales, margin, and space targets `[TB 2.1]`.
- **Functionality:** category and business unit are separate recursive hierarchies (division, banner, region, store, department on one side; department, category, subcategory, segment on the other) `[VS 6]`.
- **Contract:** two hierarchies. A `DEPARTMENT` business unit points at one root category (BU-INV-1, BU-INV-2). Category-level targets stay on `CategoryPolicy`. `[VS 6]` says this divergence belongs in the workbook's Methodology sheet (E-14).

### T-3. Flat department/category vs. arbitrary depth

- **Textbook:** department and category are two flat levels, and the department mix is expressed in shares of selling space `[TB 1.2, 2.1]`.
- **Functionality:** arbitrary depth for space, business unit, and category `[VS 6]`.
- **Contract:** arbitrary depth (HN-*). Policy is inherited from the nearest ancestor that defines it (CAT-Q-1). The textbook's department square-foot targets are not generated (T-9).

### T-4. Position "down to the inch" vs. placements with no position

- **Textbook:** a planogram fixes which SKU goes on which shelf, at what facing count, and in what position, down to the inch `[TB 2.2]`.
- **Functionality:** an incidental placement has no fixed position, and giving it one would model a precision that does not exist `[VS 3]`. Horizontal position carries almost no value `[VS 12]`.
- **Contract:** permanent and promotional placements carry `space`, `extent`, and `offset_in` (PLC-INV-5). Incidental placements carry none of them (INC-INV-4). Horizontal position is stored and checked for overlap (LS-INV-6) but never scored (POL-4).

### T-5. Facing count: the planned quantity, or a derived one

- **Textbook:** the planogram specifies "facing count" as a planned quantity `[TB 2.2]`, and space is optimized in facings `[TB 2.3]`.
- **Functionality:** store the allocated extent and the presentation plane, and derive facings `[VS 8]`.
- **Contract:** extent is stored, facings are derived (PLC-Q-1). The planner's target is still a facing count (POL-2, GEN-5), which is converted to extent for the placement.

### T-6. One planogram vs. versioned, frozen, and revised planograms

- **Textbook:** planograms enforce consistency across stores of a format, with local mods `[TB 2.2]`. Resets are full or partial `[TB 1.6]`. Assortment reviews happen at reset windows `[TB 2.6]`.
- **Functionality:** the planogram is versioned. A revision creates a new authoritative version mid-cycle. The version the crew executes is a frozen snapshot, and the one the category manager edits is a different object. A reset runs over a night or several, sometimes staggered across stores for weeks `[VS 4, 10]`.
- **Contract:** `Planogram.working` versus `PlanogramVersion` (6.5), `CYCLE` and `REVISION` versions, one pending version per scope (PGV-INV-4). Execution is instantaneous on the effective date in this version, which the functionality file says is not real. → **Q-19**

### T-7. Shelf tags: electronic or paper

- **Textbook:** chapter 8.4 describes electronic shelf labels as the price-tag technology `[TB 8.4]`.
- **Functionality:** paper tags, batch-printed and hung as part of the reset, are a deliberate sim assumption. Electronic labels would weaken the reason the batching exists `[VS 11]`.
- **Contract:** shelf tags are not modeled at all. The batching consequence is kept: change happens at resets and at released revisions (PLC-INV-14). The Methodology sheet should state the paper-tag assumption (E-14).

### T-8. Category captain and private label

- **Textbook:** a captain helps design the planogram in exchange for shopper insight and "a favored position" `[TB 2.4]`. Private-label space has a target share `[TB 2.5]`.
- **Functionality:** private label sits immediately right of the brand it is benchmarked against. The captain is asked to place a direct competitor, so private-label placement is a retailer decision the captain works around `[VS 12]`.
- **Contract:** the captain has no effect on allocation (POL-5). Private label adjacency is reported (BLK-4), and the private-label target is reported, not asserted (CAT-Q-4). → **Q-7, Q-14, Q-22**

### T-9. Department space targets and space shares

- **Textbook:** square footage is split across departments by sales productivity and role. Perishables use 30–40% of *selling space* `[TB 1.2]`.
- **Functionality:** selling space is 60–70% of the *floor*. The hierarchy covers the whole footprint so that the total reconciles `[VS 6]`.
- **Contract:** two different denominators, not a contradiction, and the contract keeps them apart. `selling_share()` reports selling space over building footprint (ST-Q-6). Space is assigned to categories as input data (ASN-*), and no department-level target is generated. → **Q-2**

### T-10. Fixtures

- **Textbook:** no fixtures section. It mentions locked cases `[TB 6.4]`, temperature-controlled display `[TB 4.4]`, and secondary/promotional visibility `[TB 7.3, 7.5]`.
- **Functionality:** gondola faces, end caps, peg sections, clip strips, floor displays, and a checkout impulse rack as floor display or its own node `[VS 6, 7]`.
- **Contract:** the fixture structure is `[DISC]` and `[VS 7]` and is not in the textbook (E-15). Checkout impulse racks are `FLOOR_DISPLAY` under a `CHECKOUT` area (TYP-3). → **Q-16**

### T-11. Items carried over from v0.1, resolved

| Topic | Textbook | Functionality | Contract |
|---|---|---|---|
| Delisting | slow movers face review `[TB 2.3, 2.6]` | silent | plan reports `delist_review` but never delists (PLAN-INV-6). → **Q-18** |
| Cold chain | unbroken to the shelf `[TB 4.4]`; perishables on the perimeter `[TB 1.3]` | space types imply temperature | derived from type (TYP-2), enforced (LS-INV-7, PLC-INV-6) |
| One store or chain | localized planograms and price zones `[TB 2.2, 7.2]` | one store | one store (BU-INV-3). → **Q-23** |
| Unit of space | department in sq ft `[TB 1.2]`, category in linear feet `[TB 2.3]` | linear vs. area by plane `[VS 8]` | per-unit measures, never mixed (RU-4) |

---

## Part B — Conflicts inside the functionality file (and against `[DISC]`)

The functionality file is a set of voice responses, so it changes its mind as it goes. Each item shows the two statements and the choice the contract made.

| ID | Statement 1 | Statement 2 | Contract choice |
|---|---|---|---|
| X-1 | "Let the presentation plane on the **placement** pick which pair of dimensions matters" `[VS 8]` | "Presentation plane is a property of the **space** node, not the product" `[VS 8]` | Space. The placement inherits the plane from its leaf. A mixed case (a horizontal deck plus a vertical riser) is two leaves and two placements |
| X-2 | The sim needs a way to know which placement is primary. "Does your placement table carry a flag?" `[VS 2]` | Primary "isn't a flag you set, it falls out of" the planogram's home position `[VS 2]` | No flag. Derived by HOME-1 |
| X-3 | "Three parallel recursive hierarchies: space, business unit, and product category" `[VS 6]` | Space type is "its own hierarchy rather than a flat tag" `[VS 6]`; the DbC section then lists four | Four hierarchies, all conforming to one `HierarchyNode` |
| X-4 | "Gondola, then face, then shelf, then the linear segment within a shelf" `[VS 7]` | "Bay by shelf number identifies the leaf, the leaf carries a linear width" `[VS 8]`; `[DISC]` has Gondola → Side → Bay → ShelfRun | `[DISC]` structure. A segment is an offset within a leaf, not a node |
| X-5 | End caps are "faces of the same gondola" `[VS 7]` | Merchandising space subdivides into "gondola, endcap, cooler, floor display" as sibling types `[VS 6]` | End caps are `LEFT` and `RIGHT` `GondolaSide`s. There is no `ENDCAP` space type. A free-standing end display is a `FloorDisplay`. → **Q-8**, **Q-16** |
| X-6 | A clip strip "consumes a bit of that face's frontage" `[VS 7]` | It is "allocated and priced quite separately from the shelves behind it" `[VS 7]` | Frontage is recorded on the strip and not deducted from any shelf (CLP-INV-2). → **Q-15** |
| X-7 | Pegboard placements "might have a shorter effective life" because re-spacing takes seconds `[VS 7]` | Permanent placements start at the reset date `[VS 4]` | Reconciled by making a peg re-spacing a released `REVISION` version, which PLC-INV-14 allows. `respace_without_reset` marks the leaf types where this is expected. Whether a peg change should be free of the version process is open. → **Q-19** |
| X-8 | Reset means "the duration": six to twelve months `[VS 4]` | "Three to twelve months depending on category velocity" `[VS 4]`; and "leave the end open or set to the next reset" `[VS 4]` | `reset_cycle_months >= 1` (CAT-INV-5) with 3–12 as the documented range. An end date is set at execution of the next version (EXE-1), not predicted at creation |
| X-9 | Revenue "has to attach at the leaf, at the placement, so it rolls up cleanly" `[VS 9]` | A SKU may have several live placements. Sales are rung up per SKU, not per shelf position `[VS 2]` | Reserved measure (2.3). How SKU revenue is split across a SKU's placements is unspecified. → **Q-9** |
| X-10 | Slotting: "ongoing shelf space is rented, renewed at reset" `[VS 3]` | Pay-to-stay: "fees for holding existing space" `[VS 3]` | Both map to `PERMANENT` placements and differ only by `AgreementType`. The functionality file does not say how they differ in behavior. → **Q-20** |

---

## Part C — Questions for future functionality discussions

Each question gives the contract's current default in **bold**. A "yes" to every default needs no reply.

**Q-1. Authority of workbook versus contract.** **Default: the contract governs, and the workbook is changed to conform (Part E), not the reverse.** The workbook currently disagrees with the contract in several places (Part D, E).

**Q-2. Who assigns space to categories, and at what grain.** **Default: input data, assigned at any merchandisable node and inherited downward (`assign_space`). A SKU draws on the nearest ancestor category that owns space in its plane (pool, 6.1).** Alternatives: assign at leaves only, or generate assignments from role and productivity `[TB 2.1, 2.3]`.

**Q-3. SKU categories at leaves only.** **Default: yes (SKU-INV-9).** A SKU is always in a leaf category node. The alternative allows a SKU on an interior node such as "Cereal" with no subcategory.

**Q-4. Target-facings formula.** **Default: space share proportional to `score ** (1 / (1 - e))` among competitors in the pool, then clamped to the category's min and max facings (POL-2).** Confirm, or name the rule you want, for example proportional to velocity only.

**Q-5. What performance means.** **Default: weekly gross-margin dollars (POL-1).** The textbook also names GMROI `[TB 2.3]`, which needs an inventory-investment figure per SKU. Switch to GMROI once inventory exists (Restocking)?

**Q-6. Permitted planes and forms.** **Default: `BOX` on FRONT or TOP; `SOFT_PACK` on FRONT (TOP if it has a per-square-foot override); `HANGING` on PEG only; `LOOSE` on TOP only (PROD-Q-1, ENUM-4).** Also: should bulky or heavy items be forced to the bottom shelf, and kids' items to lower levels? The textbook is silent.

**Q-7. Category captain.** **Default: no effect (POL-5).** Should a captain's brands get a score weight, first choice of levels, or authorship of a plan the retailer then checks for bias `[TB 2.4]`?

**Q-8. Both faces of a gondola have the same length (GDL-INV-3).** **Default: yes, the same bay count and widths.** Physical common sense, not from the functionality file. Can end faces (`LEFT`, `RIGHT`) also be missing on a gondola?

**Q-9. Revenue across multiple placements.** **Default: reserved (Part 10). When Sales arrives, a SKU's revenue is attributed to its `home_placement`.** Alternatives: split across live placements by facings or extent, or record sales per placement at the point of sale. Determines the definition of sales per linear foot.

**Q-10. Floor displays hold promotional placements only (FD-INV-2).** **Default: yes.** The workbook's floor display placements are all vendor-funded (Part D). Should a floor display ever host a permanent placement, for example a permanent bulk-display fixture?

**Q-11. Produce table temperature.** **Default: ambient (TYP-2).** Many produce tables are misted or refrigerated. Should temperature be an attribute per table rather than per type?

**Q-12. One SKU per bin at a time (BIN-INV-2).** **Default: yes, and a bin is allocated whole.** Alternatives: a SKU spans several bins (allowed today as several placements), or bins are shared by footprint.

**Q-13. Hanging products currently placed on shelves.** **Default: not allowed (FIT-1). A `HANGING` product can only be placed on a `PEG` leaf.** The workbook has 414 hanging SKUs placed on gondola shelving and 11 on floor displays (Part D). Do we create peg sections to house them, or treat hanging as a shelf-orientation variant?

**Q-14. Static hierarchies, and objectives versus constraints.** **Default: no command adds, moves, or removes a node in any of the four hierarchies. The private-label space target and department mix are reported objectives, not postconditions.** Should the private-label target be a hard postcondition when feasible?

**Q-15. Clip strip frontage.** **Default: recorded, not deducted from any shelf run's capacity (CLP-INV-2).** Deduct it from the run that the strip physically covers?

**Q-16. End caps and checkout racks.** **Default: end caps are `LEFT`/`RIGHT` gondola sides. A checkout impulse rack is a `FLOOR_DISPLAY` under a `CHECKOUT` area (TYP-3).** Are there end caps not attached to a gondola?

**Q-17. Footprint tolerance.** **Default: a parent's footprint must equal its children's sum within 1.0 sq ft (AREA-INV-2).** Real floor plans have unmeasured slivers. Larger tolerance, or an explicit "unaccounted" child?

**Q-18. Assortment status.** **Default: `DELISTED` SKUs are excluded from planning. `DELIST_CANDIDATE` and `NEW_ITEM_TRIAL` are allocated like `CORE` by score.** Should trials get guaranteed minimum or eye-level exposure? Should delist candidates be held to minimum facings?

**Q-19. Reset execution.** **Default: a version takes effect fully on its effective date (EXE-1). A peg or clip strip change is a `REVISION` version like any other.** The functionality file says real resets take a night or several, staggered across stores `[VS 10]`. When should the model represent a reset in progress (part-executed), and should a peg change bypass versioning?

**Q-20. Agreement semantics.** **Default: `fee_rate` applies to the whole term. Slotting and pay-to-stay differ only by type and compatible basis (AGR-INV-2 to 5). No fee is charged by the model; `term_fee` is a query.** How do pay-to-stay and slotting differ in the sim? Is a fee ever charged per period, and where does trade funding `[TB 2.4]` (a rebate to the retailer) go in the model?

**Q-21. Contract strength versus algorithm freedom.** **Default: GEN-5 to GEN-8 constrain the result, and plan quality beyond them is up to the implementation.** Or should the search itself be specified, for example greedy fill by score within preferred levels?

**Q-22. Blocking objectives.** **Default: brand-block and private-label adjacency breaks are reported, never required to be empty (BLK-3, BLK-4). The default mode is vertical brand blocking; horizontal tier blocking is chosen per category.** Make breaks a plan objective, and who owns the choice of mode?

**Q-23. One store or a chain.** **Default: one store (BU-INV-3).** Will the sim need multiple stores or formats, with localized planograms `[TB 2.2]`?

**Q-24. DSD vendor-managed space.** **Default: ignored.** DSD vendors (bread, snacks, soda) often stock their own shelves `[TB 4.1]`. Flag vendor-maintained space when Restocking and Receiving are specified?

**Q-25. Incidental placements.** **Default: `units_held` is a plain count, and `zone_hint` is best effort. No shelf capacity is consumed and no cold-chain rule applies yet (5.3).** Should incidental stock count against a backroom or floor limit, and must a refrigerated SKU's incidental stock stay in a cold zone?

**Q-26. Promotional placement without a permanent home.** **Default: allowed (PRM-PRE-1 to 6). A SKU needs no permanent placement first.** The functionality file treats promotional deals as separate from the baseline space agreement `[VS 3]`. Should the sim insist on a permanent home first, so that promotional displays only ever add to a base?

---

## Part D — The extract's four unverified workbook claims, checked

The extract states it made four claims from memory, without opening the workbook. Checked against `supermarket_operations_data_SCHEME2.xlsx`, the active scheme.

| # | Extract's claim | Result |
|---|---|---|
| 1 | Product and SKU are one-to-one `[VS 1]` | **Confirmed.** Product Master is keyed by UPC and carries one `SKU (ref)` per row; SKU Master pulls product attributes by lookup on UPC; both have 5,990 rows |
| 2 | Category fields are flat columns, not true hierarchies `[VS 9]` | **Confirmed.** Category Space Allocation is one row per (Department, Category) with a single `Fixture Type` per category. There is no subcategory, segment, or parent-child structure |
| 3 | Business unit is "not really there at all" `[VS 9]` | **Confirmed.** No business-unit sheet or column. `Department` exists only as a text column |
| 4 | The workbook has shelf level and facings but not "the actual length, width, height of the package" `[VS 8]` | **Wrong.** `SKU Merchandising` carries `Package Width (in)`, `Package Height (in)`, `Package Depth (in)`, `Shelf Orientation` (Upright, Hanging, Lay-Down, Stacked), and `Stackable (Y/N)`. What is true is that Product Master carries **no** dimensions, so they sit on the SKU side, not the product side (E-3) |

**Other findings relevant to the contract (Scheme 2 `SKU Placements` and related sheets)**

- `Placement Type` values: `Primary Shelf` 5,990, `Secondary/Impulse Display` 163 (all vendor-funded), `Cross-Merchandised` 19.
- 6,009 of 6,172 placements have a null `End Date`. The 163 with an end date are the secondary displays.
- 180 SKUs have two placements and 1 SKU has three. Every other SKU has exactly one. No SKU has zero.
- `Shelf Level` includes `Floor` for the 163 floor-display placements. `Space ID (ref)` uses prefixes `RUN-`, `RIF-`, `RIC-`, `FD-`, `SVC-`, `PRD-`, `DELI`, `BKY-`.
- 414 `Hanging` SKUs are placed on `Gondola Shelving` and 11 on `Floor Display`. No peg fixture exists.
- Placement `Start Date` values follow the SKU's first-listed date, not reset dates (for example 2026-03-23 and 2023-02-17).
- `Reset Cycle (months)` in Category Space Allocation includes values such as 20 and 24, outside the 3–12 range in `[VS 4]`.
- No sheets exist for agreements, pegs, clip strips, end caps, or planogram versions.
- The fixture sheets are per type (Gondolas, Gondola Sides, Bays, Shelf Runs, Bakery Case *, Floral Shelving *, Wall Shelf *, Floor Displays, Reach-In *, Deli, Service, Produce). Bakery space is named "Bakery Case" in the workbook and `BAKERY_SHELVING` in the contract.

---

## Part E — Cross-workstream flags

These follow the project's drift rule: a change in one workstream can obligate another. None has been acted on.

**Simulation data workbook (Scheme 2), attributes the contract needs and the workbook lacks**

| ID | Gap | Contract clause |
|---|---|---|
| E-1 | Agreements sheet (id, type, vendor, fee basis, rate, window). `Vendor Funded (Y/N)` becomes derived | 5.1, PLC-Q-2 |
| E-2 | Placement `Extent` and `Offset` in place of the pair `Facings` and `Linear Space Assigned`. `Facings` becomes derived | 5.2, PLC-Q-1 |
| E-3 | Package dimensions, form, `NonBoxSpec`, and case dimensions on **Product Master** (today on SKU Merchandising) | 4.1 |
| E-4 | Peg sections, clip strips, and end-cap sides (`LEFT`, `RIGHT`) as space sheets. Hook count and hook depth | 3.4 |
| E-5 | `clear_height_in` and `secured` on each shelf run and tier | 3.2, 3.5 |
| E-6 | Category → space-node assignment as explicit data. Today implied by location text and category name | ASN-*, SN-INV-2 |
| E-7 | Category hierarchy sheet (arbitrary depth) with `CategoryPolicy` fields per node. Category Space Allocation `Fixture Type` becomes `allowed_space_types` | 4.3 |
| E-8 | Business-unit hierarchy sheet | 4.4 |
| E-9 | Planogram versions sheet (kind, scope, cutoff, effective date, status, entries, home) | 6.5 |
| E-10 | Reset calendar (chain-wide dates) | ST-INV-6, PLC-INV-14 |
| E-11 | Whole-footprint space hierarchy: `Area` rows (sales floor, backroom, checkout, aisles) that reconcile to the building | 3.3 |

**Workbook columns the contract makes derived or changes**

| ID | Item | Contract clause |
|---|---|---|
| E-12 | `Cross-Merchandised` placement type has no counterpart. Contract types are `PERMANENT`, `PROMOTIONAL`, `INCIDENTAL`. The 19 cross-merchandised rows need a mapping to one of the three (raise with Q-26) | Part 1 |
| E-13 | `Floor` as a shelf level and `Fixture Type` / `Location Description` on placements should be derived from `Space ID`, not entered. Placement `Start Date`s need to be reset dates or revision dates. 414 + 11 hanging SKUs need peg homes (Q-13) | PLC-Q-2, PLC-INV-14 |

**Textbook and Methodology sheet**

| ID | Item |
|---|---|
| E-14 | Record in the Methodology & Sources sheet: category, space, and business unit as arbitrary-depth hierarchies (T-2, T-3); paper shelf tags (T-7); slotting as ongoing rent (T-1) |
| E-15 | Add a short fixtures section under textbook 2.2, or mark the fixture composition `[SIM]`/`[DISC]` in the Methodology sheet (T-10) |
| E-16 | Textbook 2.4 says slotting fees are "upfront payments". If the ongoing-rent reading is adopted (T-1), the textbook sentence needs a revision or a note |

---

## Part F — Carry-forward from a superseded draft

A separate v0.2 draft of this contract and its discussion (written without the app-functionality file, with per-routine examples) has been retired. The entries below are the domain clauses and open questions from that draft that this contract does not already cover. Each entry is self-contained, because the draft itself is no longer kept. "A" below means that draft; its clause IDs are its own and are **not** IDs of this contract.

The next run of the contract-creation prompt resolves every entry: it adopts it as a clause (logged as a D-n decision) or drops it (logged with the reason), and omits this part from the new discussion file.

### CF-1. Gondola side orientation

**Source in A:** A 2.4.1 (prose, GDL-INV-1, GDL-INV-2, SID-INV-2)

**Clause(s), verbatim:**

A Gondola is bordered by four aisles, named from a viewer at the front of the store facing the rear. In a grid layout `[TB 1.5]` its two long faces, the Gondola Sides, face the **left and right** aisles; its ends face the front and back aisles (C-6, Q-25). A Side is made of one or more Bays. A Bay has one Shelf Run per shelf level.

| ID | Clause |
|---|---|
| GDL-INV-1 | `1 <= len(children) <= 2` and the sides' `facing` values are distinct |
| GDL-INV-2 | `all(s.aisle_faced == (left_aisle if s.facing == "LEFT" else right_aisle) for s in children)` |
| SID-INV-2 | `facing in ("LEFT", "RIGHT")` |

**Behind it in A's discussion, verbatim:**

#### C-6. Which fixtures exist, and which way a gondola faces

- **Textbook:** Mentions locked cases for high-theft items `[TB 6.4]`, refrigerated and frozen display `[TB 4.4]`, and newer uses of space such as online-order staging and pickup lockers `[TB 1.6]`. It describes promotional visibility `[TB 7.3, 7.5]` and grid aisles `[TB 1.5]`.
- **Discussion:**
  - It names End Cap and checkout/impulse rack as space types, but neither is among the five physical archetypes.
  - It says a gondola is bordered by Front, Back, Left, and Right aisles, as seen from the front of the store facing the rear.
- **Conflict (new):** In a grid layout the gondolas run front to back, so their long faces (the sides that hold bays) face the left and right aisles, and the ends (where end caps go) face the front and back. The 0.1 draft had it the other way round.
- **Contract:** Sides face LEFT and RIGHT. Only the five archetypes are modeled, and `secured` is a Boolean on each leaf. → **Q-16, Q-19, Q-25**

**D-2. Gondola sides face the LEFT and RIGHT aisles (was FRONT and BACK in 0.1). (new)** See C-6 and Q-25.

**Q-25. Gondola orientation. (new)** **Default: sides face LEFT and RIGHT; ends face FRONT and BACK (D-2).** Is that how you meant the four-aisle description? Should a gondola that runs side to side (for example, along the front racetrack) be allowed, with sides facing FRONT and BACK?

**Apparent conflict with this contract:** The baseline (3.4) has the opposite geometry, grounded in `[VS 7]`: `FRONT`/`BACK` faces run the length of the gondola, and `LEFT`/`RIGHT` faces are the end caps. Adopting A's reading would also change GDL-INV-1..5, X-5, and Q-8/Q-16 in the baseline.

### CF-2. Canonical (deterministic) allocation procedure

**Source in A:** A 2.10.4 (GEN-14, `canonical_plan`, and the paragraph that follows it)

**Clause(s), verbatim:**

| ID | Clause |
|---|---|
| GEN-14 | **Canonical result.** `result == canonical_plan(self, categories, as_of)`, the procedure below. An implementation may compute it any way it likes, provided the result is identical `[SIM]` (D-1) |

**`canonical_plan(store, categories, as_of)`** (normative, via GEN-14):

1. Start the working state as the current model with every PRIMARY placement of a SKU whose category is in `categories` removed.
2. For each category `c` in `categories`, in ascending `c.key` order:
   1. Order `c.assigned_spaces` by `(policy.level_preference(l.shelf_level), l.id)` into `Lc`.
   2. For each `s` of `c` in `S` order (descending score, then `sku_id`):
      1. If `not s.in_season(as_of)`: `OUT_OF_SEASON`. Else, if `Lc` is empty: `NO_CATEGORY_SPACE`. Else, if no leaf of `Lc` accepts `s.orientation`: `INCOMPATIBLE_ORIENTATION`. Else, let `Fit = [l for l in Lc if fits_dimensionally(s, l)]`; if `Fit` is empty: `NO_DIMENSIONAL_FIT`.
      2. Otherwise, for `f` from `T(s)` down to `c.min_facings_per_sku`, and for each `l` in `Fit` in order: let `g = 1` if `l` is a `Bin` else `f`. If `l` is a `Bin` with no placement in the working state, or `l` is linear and has a free interval of length ≥ `consumption(s, g, l)`, then plan `s` on `l` with `g` facings, at the start of the **first** such interval (offset `None` for a bin). Add it to the working state and stop searching for `s`.
      3. If no position was found: `INSUFFICIENT_SPACE`.
3. `under_target` and `delist_review` follow GEN-6 and GEN-12. Placements and `unplaced` follow `S` order (GEN-13).

Full target facings take priority over a better level; the best level is chosen among positions that fit that facing count (Q-28). The canonical plan satisfies GEN-1..8 and GEN-11 by construction: each SKU is placed before every lower scorer in its category, and free space only shrinks afterward. They stay as separate clauses because each is independently checkable and traces to a textbook rule. If a case is ever found where `canonical_plan` breaks one of them, that is a defect in this contract to raise, not a choice for the implementation.

**Behind it in A's discussion, verbatim:**

**D-1. `generate_plan` now has exactly one correct answer (GEN-14, `canonical_plan`). (new)**

- **Why:** You asked for examples that show the state after each call and can be turned directly into pytest cases. Under 0.1, many different plans satisfied every postcondition. For EX-STORE-1, for example, `SKU-C3` could legally go on the top, eye-level, or bottom shelf at several offsets. So no example could assert what `generate_plan` returns, and no fixture state after a reset could be fixed.
- **What changed:** A greedy, deterministic procedure is now normative:
  - categories in key order;
  - within a category, SKUs by descending score;
  - try target facings first, then fewer;
  - leaves in shelf-level preference order, then leaf ID;
  - leftmost free gap.

  GEN-5..8 and GEN-11 remain as checkable properties that the procedure satisfies by construction. An independent check of the fixture found no case where it breaks one.
- **Cost:** This reverses 0.1's default for Q-21 ("the contract constrains results, not the algorithm"). Better allocation algorithms, such as optimization or look-ahead, would now be contract changes, not implementation choices.
- **Alternative:** keep the algorithm open and make examples assert only properties (GEN-1..13), with fixture states after a reset produced by whatever the implementation returns. The examples would then be weaker: no exact placements, and later examples couldn't reference `PLC-000004` being `SKU-C3` at offset 32.

**Q-21. Contract strength vs. algorithm freedom.** Superseded by D-1. Confirm D-1 or revert.

**Q-28. Facings vs. level priority. (new)** **Default: `canonical_plan` keeps target facings before it seeks a better level.** A top SKU gets its full target on a worse shelf rather than fewer facings at eye level. `[TB 2.3]` rewards both without ranking them. Reverse the order?

Related A cross-workstream flag (Part C, item 15):

15. **(new)** The allocation procedure (`canonical_plan`) is a simulation construct. If D-1 stands, a short worked example under Ch. 2.3 would give the textbook a concrete counterpart for "high-velocity, high-margin items earn more facings and eye-level placement."

**Apparent conflict with this contract:** The baseline (6.2, Q-21) leaves the search to the implementation and constrains only properties (GEN-5..8, GEN-9 tie-breaks). The revised creation prompt now requires a single canonical result, so a procedure is needed; A's is written over A's model (each category's `assigned_spaces`, `Bin` special case, `T(s)` from A's POL-2) and would have to be restated over the baseline's pools, planes, extents, and `home`.

### CF-3. Scarcity priority

**Source in A:** A 2.10.4 (GEN-11)

**Clause(s), verbatim:**

| ID | Clause |
|---|---|
| GEN-11 | **Scarcity priority.** If `result.unplaced[s] == INSUFFICIENT_SPACE`, then for every placed `t` of the same category with `score(t.sku) < score(s)`: `s` can't take `t`'s place, i.e. `not (fits_dimensionally(s, t.space) and consumption(s, m, t.space) <= largest_block(t.space, without=t))` with `m` = min facings (1 for a bin). A higher scorer is never left out so a lower scorer can stay |

**Behind it in A's discussion, verbatim:**

No A discussion item; A's change list, item 6: 6. **New clauses:** GEN-11 (a higher scorer is never left unplaced in favor of a lower scorer), GEN-12/13 (delist review and ordering, moved from plan invariants), CAT-INV-8/9 (a category's allowed fixture types match its temperature zone), CAT-INV-10 (floor displays are never category space, which closes a gap where allocation could put a primary placement on a display), and ENUM-4 (physical shelf order).

**Apparent conflict with this contract:** None in substance; the baseline has no equivalent. ID clash: the baseline's GEN-11 is home designation, so adopting this needs a new ID (IDs are never reused).

### CF-4. Unplaced-reason precedence

**Source in A:** A Part 1 (ENUM-5) and 2.10.4 (GEN-4)

**Clause(s), verbatim:**

| ID | Clause |
|---|---|
| ENUM-5 | `UnplacedReason.precedence` (query): the declaration order above, 1..5. When several reasons are true for a SKU, the one with the lowest precedence is reported (GEN-4) `[SIM]` |
| GEN-4 | **Reasons, by precedence (ENUM-5).** For in-season `s ∈ S` of category `c`: if `c.assigned_spaces == ()` then `NO_CATEGORY_SPACE`; else if no assigned leaf accepts `s.orientation` then `INCOMPATIBLE_ORIENTATION`; else if `not any(fits_dimensionally(s, l) for l in c.assigned_spaces)` then `NO_DIMENSIONAL_FIT`. In each of those cases `result.unplaced[s]` is exactly that reason. Otherwise `s` is placed or `result.unplaced[s] == INSUFFICIENT_SPACE`. `INSUFFICIENT_SPACE` further ensures that no leaf in `c.assigned_spaces` can take `s` at `c.min_facings_per_sku` (1 for a bin) in the planned state |

**Behind it in A's discussion, verbatim:**

No A discussion item; A's change list, item 5: 5. **Unplaced-reason precedence** (ENUM-5, GEN-4). 0.1 allowed two reasons to be true at once, for example a hanging SKU is both `INCOMPATIBLE_ORIENTATION` and `NO_DIMENSIONAL_FIT`.

**Apparent conflict with this contract:** The baseline's GEN-4 states each reason as an implication, so two reasons can hold at once. Its `UnplacedReason` values differ (`NO_SPACE_IN_PLANE` replaces `INCOMPATIBLE_ORIENTATION`), so a precedence order would be restated over the baseline's five values.

### CF-5. Delist-review threshold and order

**Source in A:** A 2.10.4 (GEN-12)

**Clause(s), verbatim:**

| ID | Clause |
|---|---|
| GEN-12 | **Delist review.** `result.delist_review` is the SKUs of `S` (placed or not) whose `assortment_status == DELIST_CANDIDATE`, or whose 0-based rank in their category's in-scope SKUs by ascending `(score, sku_id)` is `< floor(n / 10)` (bottom decile; `n` = the category's in-scope count). It is ordered by `(category.key, score, sku_id)` `[TB 2.3, 2.6]`. The decile rule is `[SIM]` (Q-26) |

**Behind it in A's discussion, verbatim:**

**Q-26. Delist-review threshold. (new)** **Default: the bottom decile uses `floor(n/10)`, so a category with fewer than 10 in-scope SKUs flags only explicit DELIST_CANDIDATEs.** 0.1's wording, which rounded up, would flag the weakest SKU of every small category, including a category's only SKU.

**Apparent conflict with this contract:** The baseline's PLAN-INV-6 says "bottom decile of their pool" with the threshold left `[SIM]` and unstated, and ranks within the pool rather than the category. A's `floor(n / 10)` rule and ordering would fill that gap.

### CF-6. Category temperature zone matches its allowed fixture types

**Source in A:** A 2.6 (CAT-INV-8, CAT-INV-9)

**Clause(s), verbatim:**

| ID | Clause |
|---|---|
| CAT-INV-8 | `len(allowed_fixture_types) >= 1` |
| CAT-INV-9 | `all(ft.temperature_zone == temperature_zone for ft in allowed_fixture_types)` — a refrigerated category can't be allowed ambient fixtures `[TB 4.4]` |

**Behind it in A's discussion, verbatim:**

#### C-11. Temperature and the cold chain

- **Textbook:** An unbroken cold chain to the shelf is a safety and shrink requirement `[TB 4.4]`. Perishables sit on the perimeter `[TB 1.3]`.
- **Discussion:** The archetypes imply temperature (reach-in cooler, reach-in freezer) but never state it.
- **Contract:** Temperature is derived from fixture type (ENUM-2). A category's allowed fixture types must match its temperature (CAT-INV-9), so a refrigerated category can't even be offered ambient shelving. → **Q-11**

**Apparent conflict with this contract:** The baseline checks temperature only per assignment (ASN-PRE-4, LS-INV-7). It has no policy-level invariant, so a refrigerated category can be configured with ambient `allowed_space_types`. Would be restated over space types and TYP-2.

### CF-7. Floor displays are never category space

**Source in A:** A 2.6 (CAT-INV-10)

**Clause(s), verbatim:**

| ID | Clause |
|---|---|
| CAT-INV-10 | `FixtureType.FLOOR_DISPLAY not in allowed_fixture_types` — floor displays are never category (primary) space; they carry secondary placements only (FD-INV-2) `[SIM]` (Q-16) |

**Behind it in A's discussion, verbatim:**

**Q-16. End caps and checkout racks.** **Default: not modeled. Floor displays are never category space (CAT-INV-10, new).** Should an end cap be a child of a Gondola facing the Front or Back aisle? Is a checkout rack a floor display?

**Apparent conflict with this contract:** The baseline makes floor displays promotional-only (FD-INV-2) but does not stop `FLOOR_DISPLAY` from appearing in a category's `allowed_space_types`, so `assign_space` can assign a display to a category. A's clause closes that.

### CF-8. Physical order and positive dimensions of fixture parts

**Source in A:** A Part 1 (ENUM-4) and 2.4 (BAY-INV-3, BAY-INV-4, RUN-INV-2, SVC-INV-3, TIER-INV-2)

**Clause(s), verbatim:**

| ID | Clause |
|---|---|
| ENUM-4 | `ShelfLevel.physical_order` (query): TOP = 1, EYE_LEVEL = 2, MIDDLE = 3, BOTTOM = 4, top to bottom `[SIM]` |
| BAY-INV-3 | `position >= 1 and width_in > 0` |
| BAY-INV-4 | `[r.shelf_level.physical_order for r in children] == [1, 2, 3, 4]` — top to bottom |
| RUN-INV-2 | `depth_in > 0 and clear_height_in > 0` |
| SVC-INV-3 | tier shelf levels are distinct and `[t.shelf_level.physical_order for t in children]` is strictly increasing |
| TIER-INV-2 | `depth_in > 0 and clear_height_in > 0` |

**Behind it in A's discussion, verbatim:**

No A discussion item. Related A cross-workstream flag (Part C, item 10):

10. **(new)** Door numbers, bay positions, and tier levels in physical order (BAY-INV-4, SVC-INV-3).

**Apparent conflict with this contract:** The baseline requires each shelf level once per bay or door (BAY-INV-1, DOR-INV-1) and distinct tier levels (SVC-INV-1), but no top-to-bottom order and no positive width, depth, or clear height.

### CF-9. `add_months` definition

**Source in A:** A 2.6 (`add_months`, ADDM-PRE-1, ADDM-1)

**Clause(s), verbatim:**

| ID | Kind | Clause |
|---|---|---|
| ADDM-PRE-1 | require | `n >= 0` |
| ADDM-1 | ensure | `(result.year * 12 + result.month) == (d.year * 12 + d.month + n)` and `result.day == min(d.day, days_in_month(result.year, result.month))` |

**Behind it in A's discussion, verbatim:**

No A discussion item.

**Apparent conflict with this contract:** The baseline's CAT-Q-3 calls `add_months` but never defines it, including the month-end case.

### CF-10. Leaf order when a category mixes units

**Source in A:** A discussion only

**Clause(s), verbatim:**

A's POL-4 (the baseline's POL-4 has the same `None` rule):

| ID | Kind | Clause |
|---|---|---|
| POL-4 | ensure | `level_preference(l) == l.preference_rank` for a shelf level, and `0` for `None` (unranked leaves sort first) |

**Behind it in A's discussion, verbatim:**

**Q-31. Leaf order when a category mixes units. (new)** **Default: unranked leaves (bins, displays) sort before ranked shelves (`level_preference(None) == 0`).** It only matters if a category is assigned both bins and shelves. Is that possible, for example packaged salad on a shelf next to loose produce?

**Apparent conflict with this contract:** None apparent; the baseline gives no leaf ordering at all. Matters only if a canonical procedure (CF-2) is adopted.

### CF-11. Season granularity

**Source in A:** A discussion only

**Clause(s), verbatim:**

A's representation: | SKU-INV-9 | `season_months is None or (len(season_months) >= 1 and season_months <= set(range(1, 13)))` |

| SKU-Q-1 | `in_season(on)`: `result == (season_months is None or on.month in season_months)` |

**Behind it in A's discussion, verbatim:**

**Q-29. Season representation. (new)** **Default: `season_months`, a set of months, or None for year-round.** Do you need finer windows (weeks or specific dates, e.g. holidays), or a category-level season for OCCASIONAL_SEASONAL categories?

**Apparent conflict with this contract:** The baseline represents a season as a `(first_month, last_month)` tuple that may wrap the year, not a set of months. The question (finer windows, or a category-level season) is open in both.

### CF-12. Facings across bay boundaries

**Source in A:** A discussion only

**Clause(s), verbatim:**

(no clause)

**Behind it in A's discussion, verbatim:**

**Q-17. Facings across bay boundaries.** **Default: no; a placement sits within one leaf.** Allow a SKU to span an upright?

**Apparent conflict with this contract:** None apparent; the baseline also places each placement in one leaf, but does not ask the question.

### CF-13. Grain of the `secured` attribute

**Source in A:** A discussion only

**Clause(s), verbatim:**

(no clause)

**Behind it in A's discussion, verbatim:**

**Q-19. Secured/locked space.** **Default: `secured` is set per leaf.** Is secured a whole fixture, a door, or a single shelf?

**Apparent conflict with this contract:** None apparent; the baseline also sets `secured` per leaf, but does not ask the question.
