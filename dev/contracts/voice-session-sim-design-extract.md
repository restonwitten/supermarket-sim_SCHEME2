# Voice Session Extract — Sim App Design Points

Source: voice conversation, Sept 27 2026 (on bike). Claude's responses only, filtered to those bearing on the simulation app / data model. Grouped by topic, in conversation order within each group. Responses cut off mid-sentence by interruption are marked *[truncated]*.

Excluded as not bearing on the sim: electronic shelf label costs and failure detection, dynamic/discriminatory pricing law, reminder setup, sign-offs.

**Unverified claims to check against the workbook:** statements below about what the workbook currently contains (one-to-one Product/SKU mapping, flat category columns, no business unit, no package dimensions) were made from memory without opening the file.

---

## 1. Product vs SKU

> Right, so the relationship runs product-first: Product Master is the source of truth for product-level facts, and SKU Master pulls them across by VLOOKUP on UPC. So SKU references product, not the other way round.
>
> Conceptually a product is the thing itself, the manufactured item with its brand, size, formulation, and UPC. A SKU is the retailer's unit of managing that thing in their own operation, with its own space, ordering, and pricing attributes. One product can in principle map to several SKUs if it's sold in different store contexts or pack configurations.

> So the practical upshot is that product answers "what is this," and SKU answers "how do we stock, price, and shelve it here."
>
> If yours is currently one-to-one, that's a modelling simplification rather than a real-world constraint — worth knowing before the sim starts doing anything with multi-pack or multi-location variants.

## 2. SKU Placement — cardinality and primary placement

> Yeah, that's exactly right — placement is one-to-many off SKU, and zero is a legitimate state for a delisted or not-yet-reset item.
>
> The contiguous case being a single placement is a nice clean rule, and it means multiple placements genuinely signal something: a secondary display, an endcap, a seasonal off-shelf position. That's actually useful information rather than just data duplication.
>
> The thing worth flagging is that if placement can be multiple, the sim needs a way to know which one is primary — the home shelf position the planogram owns and replenishment defaults to. Does your placement table carry a flag like that, or is it implied by which row came first?

> Right, so the placement row isn't just a physical fact, it's the commitment — what you agreed to give that vendor, where, and how much.
>
> That changes what primary means, though. Under that reading, primary isn't a flag you set, it falls out of which placement the planogram treats as the home position for replenishment. Worth keeping those two distinct.

## 3. Agreement types and placement types

> Yeah, that's the slotting fee logic — ongoing shelf space is effectively rented, priced by facings or linear feet, and renewed at reset.
>
> Secondary displays tend to be a separate, shorter deal though: a promotional endcap for a four-week window with its own fee, often tied to trade funding rather than the baseline space agreement. So two different contract types feeding the same placement table.

> Yeah, you've spotted something real. What you saw on top of the case sounds like overstock or an unplanogrammed stack rather than a negotiated display — the store just put product somewhere because the shelf couldn't hold it, or a DSD vendor dropped extra.
>
> So that's arguably a third category: not a contract at all, just an operational placement with no fee behind it. Which means your placement table might want a placement type that distinguishes permanent, promotional, and incidental.

> Exactly — no space commitment at all, just "we'll take the inventory and find room for it." Stockers put it wherever there's floor that week, so it moves around and isn't a stable location.
>
> That's a meaningfully different thing from a placement with coordinates. If the sim tries to give it a fixed position, it'll be modelling a precision that doesn't exist.

> Yeah, you've got it — there's a ladder of agreement types, and reset timing is just the clock the top tier runs on.
>
> Roughly: slotting agreements for permanent shelf space, tied to reset windows. Promotional display deals, typically four to eight weeks, priced per display per store. Then pay-to-stay or maintenance fees for holding existing space. And at the bottom, opportunistic buys where there's no space commitment at all — you took the deal on the product, not on the shelf.
>
> So in the model, placement type and agreement type are related but separate fields.

> Hmm, not quite — the space agreement doesn't usually commit to a number of shipments. It commits to space and time: so many facings or linear feet, for the reset period.
>
> Shipments are downstream of that. How much actually flows in depends on how fast it sells, driven by replenishment, not by a contracted quantity.

## 4. Reset cycle and effective dating

> Typically it's tied to the reset cycle rather than a calendar contract, so most center-store categories reset once or twice a year — six to twelve months is the usual life of a permanent placement.
>
> Fast-moving categories with heavy new-item flow, snacks and beverages especially, can reset quarterly. So call it three to twelve months depending on category velocity.

> Right — I've been using reset to mean the period between one planogram reset and the next, so the duration. Six to twelve months typically.
>
> But the actual start and end dates are fixed too: resets happen on scheduled windows across the chain, not rolling per vendor. So a deal runs from one reset date to the next.

> Exactly — the reset is a batching decision. You're amortising the labour of re-tagging, re-facing, and re-stocking across the whole store in one pass instead of paying it piecemeal.

> I'd give it effective start and end dates. Mainly because promotional placements genuinely have terms that don't line up with resets, and you want both kinds in one table without special-casing.
>
> For permanent placements you just set the start to the reset date and leave the end open or set to the next reset. That way the sim can ask "what was on this shelf on this date" uniformly, which you'll want for any historical analysis.

> Yeah, the volume's trivial — you're talking tens of thousands of placement rows even with years of history. That's nothing to hold in memory.
>
> So no need for an archive table. Just filter on effective dates for the live view and load the lot when you're doing historical work.

## 5. Persistence

> Yeah, write-through is the normal pattern — the database is the system of record, and whatever you hold in memory is a cache over it.
>
> For your sim, though, the workbook is already playing that role. So the question is whether you're persisting simulation state separately from the source data, or just re-reading the workbook each run.

> Right, so the workbook becomes seed data rather than the runtime store. You load it once to populate a real database, then the sim reads and writes there.

> Makes sense — good thinking-out-loud session. The useful residue is that placement wants effective dates and a placement type, and agreement type probably deserves to be its own thing rather than a column.

## 6. Recursive hierarchies (space, space type, business unit, product category)

> That's a reasonable instinct, not indecision. Arbitrary-depth hierarchies for both space and business unit are the right call — real chains have division, banner, region, store, department, subcategory, and the textbook's flat category framing is a simplification you're entitled to override.
>
> Worth flagging though: that's a textbook-versus-data divergence, so it should get noted in the methodology sheet rather than just living in the code.

> Yeah, category should be arbitrary-depth too — department, category, subcategory, segment is the standard grocery ladder, and some categories genuinely go deeper than others.
>
> Which means you've got three parallel recursive hierarchies: space, business unit, and product category. Same structural pattern each time, so it's worth implementing once as a shared shape rather than three times.

> Right, so you need aggregation queries that roll up or down either hierarchy and cross-reference against placements to find the gaps.
>
> That's an argument for the recursive structure being queryable rather than just nested objects in memory — you'll want something like a closure table or path enumeration so "all space under this node" is one query, not a tree walk.

> Yeah, that matters — selling space is maybe sixty to seventy percent of the floor. The rest is backroom, prep areas, checkout, aisles, restrooms, offices.
>
> So the space hierarchy should cover the whole footprint with a space type that says whether a node is merchandisable. Then "unallocated merchandising space" is a filter, and you can still audit that the total reconciles to the building.

> Yeah, and that's the case for space type being its own hierarchy rather than a flat tag — merchandising space subdivides into gondola, endcap, cooler, floor display, each with attributes the others don't have. Checkout space has utterly different ones.
>
> So the physical hierarchy says where things sit and nest; the type hierarchy says what kind of thing it is and which attributes apply. Two dimensions, crossed.

## 7. Merchandising space structure: gondola, face, endcap, peg, clip strip

> Exactly — the gondola's a physical object but the allocatable unit is the face, not the box. Two faces, two aisles, two different categories.
>
> So the merchandising space hierarchy goes gondola, then face, then shelf, then the linear segment within a shelf. Allocation happens at the bottom of that, while the gondola exists mainly to anchor the geometry.
>
> Endcaps are a nice edge case: they're faces of the same gondola but usually managed as a separate, more valuable unit.

> Right, and those command premium pricing in the space agreements — an endcap is typically the most valuable square footage in the centre store because it's in the traffic flow rather than requiring an aisle entry.
>
> So same structural node as any other face, but with quite different economics attached.

> Yeah, it falls out nicely — the geometry and the economics are separate layers over the same node.
>
> Which is also why the agreement sits on the placement rather than on the space itself. The space just is; what it costs depends on who's buying it and when.

> Ha, yes — pegboard. Whole different allocation logic: you're selling hook positions, not linear inches, and the depth is how many units deep the peg holds.
>
> So peg space is a distinct space type with its own attributes, which is another argument for that type hierarchy earning its keep.

> Right — and it's modular in a way shelving isn't. You can re-space a pegboard section without a reset because moving a hook is seconds.
>
> Which quietly means peg placements might have a shorter effective life than shelf ones, since the cost of changing them is so low.

> Peg hook is the usual term, or just pegs — the board they mount into is pegboard, and a run of them across a gondola face is a peg section.
>
> And no, I don't think you need to model individual hook coordinates. Section plus hook count is enough for anything the sim will ask.

> Ah, got it — that's usually called a display strip or a clip strip if it's the flexible ribbon kind, and a shelf-mounted one with multiple prongs is often a merchandising strip or a J-hook rail.

> Yeah, I'd make the clip strip a space node in its own right, typed as clip strip, whose parent is the gondola face it hangs from. Its children are the hook positions, or just a hook count if you don't need that granularity.
>
> That keeps the hierarchy honest: it's physically attached to the face, it consumes a bit of that face's frontage, but it's allocated and priced quite separately from the shelves behind it.

> Ha, yeah, that's mounting-hardware compatibility — real in the physical world, irrelevant to the sim. *[truncated]*

## 8. Presentation plane, facings, and package dimensions

> Right, facings are exactly what gets renegotiated at reset — that's the main variable in the space agreement.
>
> And yeah, the horizontal case still has a facing concept, it's just measured across the width you see looking down rather than across a vertical shelf front. Same idea, different plane.

> Ahh, but it's not a bad edge case — it exposes that "facing" is really shorthand for "the product surface the shopper sees," and the axis changes with the display.
>
> The clean fix is to stop counting facings and store the allocated area or linear extent plus the presentation plane. Then facings become derived: allocated width divided by product width, along whichever axis the case presents. Your three-by-three yogurt is nine visible units on a horizontal plane, and the maths is the same as two facings on a vertical one.
>
> That also handles the mixed case, where a single item spans a horizontal lower deck and a vertical riser.

> Yeah, dimensions belong on the product — length, width, height of the consumer unit, plus case pack dimensions for the shipper.
>
> That's the physical truth about the thing, so it lives with the product regardless of which retailer stocks it or how they display it. Placement then just references those numbers to compute how many fit.

> It nearly is, though. The product's a box, and the placement picks which of its faces is presented — front, top, or occasionally side. Facing count is then just allocated extent divided by the relevant dimension.
>
> The only real complication is products that aren't boxes: bagged goods that slump, hanging peg items, loose produce. Those need an override rather than a computed number.

> I believe not — the workbook has shelf level and facings as categorical and count fields, but not the actual length, width, height of the package.
>
> Which is exactly the gap your facing-as-derived-value idea needs closed. Without package dimensions you can't compute facings from allocated extent; you can only store facings as a given number.
>
> So that's a Product Master column set to add: consumer unit dimensions, and ideally case pack dimensions too.

> Right — store the three dimensions on the product, and let the presentation plane on the placement pick which pair of them matters.
>
> Front face uses width by height, top face uses width by depth. Same box, different projection. That's the whole mechanism, and it collapses your yogurt edge case into ordinary arithmetic.

> Exactly — it's a bounding box, nothing more. Not the shape, not the graphics, not how it's oriented in the case.

> Right, if you define X as width, Y as height, Z as depth, then a vertical shelf facing consumes X of linear shelf run.
>
> And on a horizontal case viewed from above, the facing consumes X by Z of deck area — two dimensions instead of one, because you're allocating surface rather than a line.

> Yeah, that's the real difference — vertical shelving allocates linear feet, horizontal decks allocate square feet.
>
> So your space node needs to know which measure applies. Linear extent for shelves and pegs, area for open decks and floor displays. Otherwise the aggregation adds unlike quantities.

> Right, and that's clean: bay by shelf number identifies the leaf, the leaf carries a linear width, and allocation consumes linear inches off it.
>
> Unallocated is then just the leaf's width minus the sum of its live placements' extents, where extent is facings times package width.
>
> The term you're reaching for for vertical versus horizontal is presentation plane — a property of the space node, not the product. The shelf knows whether it presents a front face or a top deck, and that decides which package dimension the arithmetic uses.

> Yes, and that's the distinction worth locking in. Presentation plane is a property of the space — which surface of the fixture the shopper sees. Facing is a property of the placement — how many units wide the product sits along that surface.
>
> So the plane tells you which of the product's three dimensions is the relevant one, and the facing count times that dimension gives the extent consumed. Two different objects, cleanly separated.

> Right, so summing extent across a SKU's live placements gives its total allocated space — but in mixed units if some placements are linear and others area.
>
> Which means the roll-up needs either a common measure, or to report the two separately. Probably cleanest to carry both and let the query decide.

> Right, and in that case the SKU's total is genuinely two quantities — square inches on the deck plus linear inches across the uprights. They don't add.
>
> What does roll up cleanly is the derived figures: total facings, total units held, and revenue. So for cross-plane comparison you'd aggregate on those rather than on raw extent.

## 9. Aggregation / roll-ups ("the cube")

> Yes — and that's the key requirement. Every non-leaf node needs to aggregate its children: total linear feet, allocated versus unallocated, revenue, margin, sales per foot.
>
> Which means revenue has to attach at the leaf, at the placement, so it rolls up cleanly. And it means the same aggregation machinery serves the space tree, the category tree, and the business unit tree — three views over the same leaf facts.
>
> That's your cube, essentially.

> Partly — the workbook has category fields and shelf assignment, but as flat columns rather than true hierarchies. And business unit isn't really there at all.
>
> That's why what you've been sketching is a genuine restructure rather than an addition.

## 10. Planogram versioning and drift

> Your instinct to doubt it is right. The planogram is authored ahead of the reset and executed on reset day, so that's the moment shelf and plan agree. But drift starts immediately.
>
> Two sources. Off-cycle changes: an item gets discontinued, a recall lands, a vendor pushes a new item mid-cycle, and the category manager issues a revision rather than waiting. Those are real plan changes, versioned, and the shelf is meant to follow.

> Yeah, exactly — rare per item, constant across thirty thousand of them.
>
> The second is execution drift, where the plan stays put and the shelf wanders off it. Stockers spread facings to cover a hole, product creeps into a neighbour's space, an out-of-stock gets backfilled with whatever's adjacent.
>
> That's why compliance auditing exists — periodic checks, increasingly shelf cameras, measuring actual against plan. And it means your model wants three things kept apart: the planogram as intended state, the placement as agreed commitment, and observed shelf state as what's actually there.

> Yes, and that's the right framing — every divergence is a prediction failure somewhere, even the excusable ones.
>
> But worth separating two kinds. Execution drift is pure waste: nobody wanted it, it costs sales and compliance, and driving it to zero is unambiguously good.
>
> Plan revisions are different. A recall or a discontinuation is the world changing, not the plan being wrong. Driving those to zero would mean freezing the plan against reality, which is worse. What you actually want there is the revision loop to be fast and cheap, not absent.
>
> So the economic target isn't sameness everywhere — it's zero unintended divergence, and low-latency intended divergence.

> Right — and that's the cleanest way to hold it: the planogram is versioned, and a revision creates a new authoritative version mid-cycle rather than the shelf quietly departing from the old one.
>
> So compliance is always measured against the current version. The plan can change; what's forbidden is the shelf changing without it.

> Yes, and the software handles that precisely by freezing a version for execution. The reset pack the crew works from is a snapshot — the planogram as of a cutoff date, printed or on a handheld, with the shelf tags batch-generated against that same version.
>
> Which matters because a reset isn't instantaneous. It runs over a night or several, sometimes staggered across stores for weeks. If the plan were live during execution, two crews could be building different stores from different plans.
>
> So the version the crew executes and the version the category manager is currently editing are deliberately different objects. Revisions made during the window queue up for the next release rather than landing mid-build.

> Right, generating it is nearly free now. Which shifts the whole constraint.
>
> The cost was never really the drawing — it's the physical labour of moving product and the cognitive load of a crew executing a plan that keeps shifting under them. That hasn't got cheaper.

## 11. Shelf tags — sim assumption

> Right — paper tags, batch-printed and hung as part of the reset, with off-cycle reprints possible but exceptional.
>
> That's a cleaner model anyway, and it keeps the reset as the meaningful unit of change. Electronic labels would mostly just weaken the reason the batching exists, which isn't the question you're trying to answer.

## 12. Shelf position value, brand blocking, private label adjacency

> Yeah, brand block — the vendor's items sit contiguous so the block reads as one presence on the shelf.
>
> Which is another thing the placement model should be able to express: adjacency isn't accidental, it's specified in the planogram, and often part of what the vendor negotiated.

> Right, and the vertical dimension is where the real value sits — eye level and the level just below carry a substantial premium over top and bottom shelves.
>
> Horizontally it's much flatter. There's a modest effect from travel direction, since most shoppers work one way down an aisle, but nothing like the vertical gradient. So among competitors all sitting at eye level, left-to-right position isn't doing much.
>
> Which means the valuable coordinate for your model is shelf number, not bay position.

> Yeah, that's the usual arrangement — vertical brand blocking, where each brand owns a horizontal span and runs top to bottom within it.
>
> So every brand gets a share of the good shelves and a share of the poor ones, which is part of why it's the standard. It's equitable enough that competing vendors accept it.
>
> The alternative, horizontal blocking, puts a brand along one whole shelf level — which would hand eye level entirely to one player. That's why you mostly see it only where a category has a dominant private label or a clear tier structure.

> Close, and the logic is deliberate. Private label sits immediately adjacent to the brand it's benchmarked against, usually just to the right of the leader, because the comparison is the whole point — same look, visibly lower price, decision made at the shelf.
>
> The tension you've spotted is real, though. The category captain builds the planogram and is being asked to place a direct competitor to itself in the best available adjacency. That's exactly why retailers keep independent judgement over the captain's recommendation, and why private label placement is typically a retailer decision the captain has to work around rather than something they propose.

## 13. Querying and testing approach

> Yeah, worked query examples — unallocated merchandising space under a department, revenue rolled up a category subtree, all placements live on a given date.
>
> Those double as a spec: if the model can't express them cleanly, the model's wrong.

> Yeah, a full matrix — each hierarchy crossed with each query shape: roll-up, drill-down, filter by type, filter by date, allocated versus unallocated.
>
> Which is a proper test suite as much as documentation. Good way to prove the shared recursive shape actually holds across all three.

> Ahh, so a little query language over the object model — parse a string, apply it to the in-memory objects.
>
> I'd push back gently: you're describing SQL, and if the data's heading to a database anyway, you get that for free. A hand-rolled parser is real work to build and a real thing to maintain.

> You can — set operations work fine on Python objects. Intersect two filter results, union them, difference them.
>
> So testing unions and intersections doesn't require a query language. It requires the filters to return sets, which is just a design choice you can make up front.

> Right — express the test cases as plain Python set operations over filter results. No parser, no grammar, and the tests stay readable.
>
> If a string language ever earns its place, it's a thin layer on top of that same machinery rather than a replacement for it.

> Careful — it's the other way round. An "and" of conditions is an intersection, since both must hold; an "or" is a union.

## 14. Design by contract — the next deliverable

> Ah, so the example usages become the contract — you write the client code you want to be able to write, and the model has to satisfy it.
>
> That's a good discipline, and it inverts the usual order nicely: the queries you sketched earlier stop being tests of the model and become the specification of it.
>
> Design by contract proper adds pre- and postconditions on the methods themselves, which would also be worth having on the aggregation functions — a rollup should assert its children sum to the parent.

> Right — the negative cases are as much a part of the contract as the positive ones. Cycles in the hierarchy, a placement whose end date precedes its start, allocated extent exceeding the parent node's capacity, two placements overlapping the same shelf segment on the same date.
>
> Each of those wants an explicit invariant that raises rather than quietly producing nonsense. And they make excellent tests, since a model that can't be made to fail correctly usually can't be trusted when it succeeds.

> So that's the frame: positive usages as specification, invariants as guard rails, and the query matrix proving the hierarchies behave.

> Understood — two artefacts, in that order. First the contract document: the example usages, the invariants, the query matrix, written as the specification. Then the implementation against it, plus the runtime tests for the things only checkable when it's actually running, like cycle detection and date overlaps.
>
> The handoff being to myself is the honest part — the contract document has to stand on its own well enough that a fresh session could build from it without this conversation.

> Yeah, and that's a good reason to do it this way — you'll see the standard form rather than reinventing it. Preconditions, postconditions, invariants, with the example usages sitting on top.
>
> It's a well-worn pattern from Eiffel originally, so there's a settled shape to borrow rather than a design problem to solve.
