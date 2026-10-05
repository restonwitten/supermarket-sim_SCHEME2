# Contract Review Chat v0.3_10042026
CRCT - Contract Review Chat Topic


## CRCT-3 - Leaf Identification in Tree Structures

### Reston:

Is a stored leaf attribute needed on hierarchical structures? Isn't the leaf status of a node indicated by the absence of elements in the attribute that points to child nodes?

### Claude:

No. A stored leaf flag is redundant state. Leaf status should be a derived query, not an attribute:

```python
@property
def is_leaf(self) -> bool:
    return len(self.children) == 0
```

In Meyer's terms, this is a query whose postcondition is `Result = children.is_empty`. Under the Uniform Access Principle, clients can't tell whether it's stored or computed, so nothing is lost by computing it. Storing it costs you something:

- You need a class invariant `is_leaf = children.is_empty`, and every `add_child` / `remove_child` must maintain it. That adds pre- and postcondition clutter and creates a way for the state to drift out of sync.
- It collides with the general hierarchy rule that undefined attributes are inherited from the parent. A stored `is_leaf` would be a stored attribute that must never inherit, so it becomes a special case.

One legitimate distinction still matters, because "has no children right now" isn't always the question being asked:

1. **Structural leaf**: no children at this moment. This is derived, as above. An empty Department or a newly created Gondola with no Bays yet is a structural leaf.
2. **Terminal by type or level**: the node *cannot* have children. Examples are a Shelf Run, a Produce Table Bin, a Floor Display, or a GPC Brick (level 4). This is a property of the node's type or hierarchy level, not a per-node stored flag.

The allocation logic cares about the second meaning, not the first. An empty Gondola is a structural leaf but must not receive placements.

The current contract (`dev/contracts/sim-contract.md`) already gets this mostly right:

- `is_leaf` is declared as a **query** on `HierarchyNode`. Principle 2, Uniform Access, says derived facts are not stored.
- Allocatability is kept separate from structural leaf status. `allocatable_leaves()` (SN-Q-1) filters on `isinstance(l, LeafSpace)`, not on `is_leaf`. The leaf classes (`ShelfRun`, `Tier`, `Bin`, `FloorDisplay`, `PegSection`, `ClipStrip`) take no children in their constructors, and LS-INV-1 requires them to be leaves. So "terminal by type" already exists as conformance to `LeafSpace`; an `is_terminal` attribute would be redundant.
- The other `is_leaf` uses mean structural leaf, which is correct: SKU-INV-9, CAT-INV-7, BU-INV-1, and RUN-, BAY- and DOR-NEW-PRE-1.
- The hierarchies are structurally static after construction (Q-14), so there is no maintenance burden either way.

The one change is to how the rule reads. HN-INV-1 states `is_leaf == (len(children) == 0)` as an invariant, which reads like a consistency constraint between two independent pieces of state and invites a stored field. As a postcondition, it is the definition of a derived query, matching how `path`, `depth` and `root()` are specified. The two forms mean the same thing.

Open: SKU-INV-9 needs to say whether "leaf category" means a structural leaf in the store's category tree or a GPC Brick. CRCT-2's adoption of GPC settles it as Brick.

**Recommended Changes to Contract Creation:**

- Prompt: specify derived facts such as `is_leaf` as query postconditions, not as invariants relating them to other state.
- Prompt: allocatability is expressed by conformance to the leaf-space class; no `is_terminal` attribute.
- Prompt: the leaf category that SKUs attach to is the GPC Brick (per CRCT-2).

**Effect on Contract Creation Products of Recommended Changes to Contract Creation:**

- HN-INV-1 becomes a postcondition (TN-Q-0 under CRCT-5), and the `is_leaf` interface line is annotated as derived.
- The HN-INV-1 stub counter-example is replaced by a supplier counter-example, e.g. a bay with four runs reporting `is_leaf == True`.
- MON-7 no longer lists the leaf rule among the tree-node invariants.
- SKU-INV-9 and CAT-INV-7 refer to GPC Bricks.

## CRCT-4 - Contract-to-Textbook Delta Checking

### Reston:

De-prioritize reconciling the contract or the implemented model against the supermarket operations textbook. The gap between this textbook should not be actively tracked, but only discovered by a specific request from me.

### Claude:

The textbook gap is tracked today because `create_contract_PROMPT.txt` (line 7) asks for a description of where the textbook is at odds with the app-functionality discussion. That request produces Part A of the discussion file (T-1 … T-11) and the textbook-edit rows of Part E (E-15, E-16, E-20, E-21). Removing it from the prompt stops the tracking at the source.

The textbook stays a grounding source. `[TB x.y]` tags cite where a clause comes from; they don't track the gap, so they remain.

This replaces the earlier plan to have a separate, independent process check the textbook-contract delta. The gap is now checked only on request.

**Recommended Changes to Contract Creation:**

- Prompt line 7: drop the request for textbook-vs-app-functionality conflicts, and keep the D-n, questions, "(new)" and carry-forward instructions.
- Prompt: don't report textbook gaps or flag textbook edits unless asked; keep citing the textbook as `[TB x.y]` grounding.

**Effect on Contract Creation Products of Recommended Changes to Contract Creation:**

- Discussion file: Part A (T-1 … T-11) and the textbook rows of Part E (E-15, E-16, E-20, E-21) are removed, and the part index is updated.
- Contract Status line: the companion-file description no longer mentions textbook conflicts.
- `[TB x.y]` tags remain.

## CRCT-5 - Terminology Changes - HierarchyNode to TreeNode

### Reston:

Change HierarchyNode to TreeNode? Discuss, then present "Recommended Changes to Contract" at the end.

### Claude:

Yes, rename it to `TreeNode`.

Why `TreeNode` fits better:

- **It names the constraint the class enforces.** The class has one parent per node (HN-INV-2), no cycles (HN-INV-3) and a single root (HN-Q-6), which makes it a rooted tree. "Hierarchy" also covers structures where a node has several parents (a DAG). Retail category schemes sometimes allow that, for example a product listed under both "Breakfast" and "Snacks". The contract forbids it, and `TreeNode` says so in the name.
- **The vocabulary already matches.** The class's queries (`root`, `leaves`, `subtree`, `depth`, `path`, `is_leaf`) are tree terms.
- **It separates shape from domain.** `TreeNode` is the structural behaviour shared by all four hierarchies. "Space hierarchy", "category hierarchy" and "business-unit hierarchy" stay the domain terms in prose. That fits the rule that data shape and behaviour are separate axes.

Cost: the name is the run's own choice, not the prompt's. It appears 18 times in the contract and 2 in the discussion file, and the `HN-` clause ID prefix appears 33 times in the contract and 3 in the discussion file. The prompt has to ask for the rename explicitly, or a rebuild may keep the old name. The clause IDs should change with it (to `TN-`), or the prefix will be orphaned from the class it names.

What doesn't change: the domain prose that says "hierarchy", the inheritance rule for hierarchies, and `SpaceNode` / `CategoryNode`.

**Recommended Changes to Contract Creation:**

- Prompt: name the deferred class for shared tree structure and navigation `TreeNode`, with clause-ID prefix `TN-`.
- Prompt: reserve "hierarchy" for the domain structures (space, space type, business unit, product category), and use "tree" for the structural class and its terms.

**Effect on Contract Creation Products of Recommended Changes to Contract Creation:**

- `HierarchyNode` becomes `TreeNode` throughout (18 in the contract, 2 in the discussion file).
- `HN-` clause IDs and `P.HN_*` predicate names become `TN-` and `P.TN_*`, including in MON-7 and Appendix A.
- Domain prose keeps "hierarchy"; `SpaceNode` and `CategoryNode` are unchanged.

## CRCT-6 - Product Category Reference - Product vs SKU

### Reston:

How does the contract describe the association between products and product categories?

It seems to me that the product category leaf should be referenced from the product. Why would we decide to put it on the SKU? All SKUs for a product would have the same product code. No?

### Claude:

**What the contract does now.** `Product` (4.1) has no product-category attribute. The association runs through `SKU` (4.2):

- SKU → Product: `SKU.product`. The reference goes one way only. PROD-INV-3 makes `Product.skus` derived, and one product may have several SKUs.
- SKU → Product Category: `SKU.category: CategoryNode`, which must be a leaf (SKU-NEW-PRE-1, SKU-INV-9; leaf-only is discussion default Q-3, grounded `[SIM]`).
- `CategoryNode.skus` holds the SKUs attached exactly to a node and is non-empty only on leaves (CAT-INV-7). `subtree_skus()` unions them over the subtree (CAT-Q-5).
- `SKU.department` is `category.root()` (SKU-INV-6).
- ST-INV-4 and BUILD-PRE-4 require each SKU's category to be in the store's hierarchies.
- Through `sku.category.policy()`, the SKU gets its facing bounds and other policy fields (PLC-INV-10, POL-2, GEN-4, GEN-13).

**This is wrong in the contract, not just a matter of wording.** Product Category is GS1 GPC (CRCT-2), and GPC classifies the product: the brick is a fact about the GTIN/UPC, like brand and dimensions. Every SKU of a product has the same brick, so a SKU-level reference is redundant. It also permits an inconsistent state, two SKUs of one product under different product categories, which no clause forbids.

**Why the contract put it on the SKU.** Neither the discussion file nor `create_contract_PROMPT.txt` records a rationale. The implicit logic is the contract's own split: Product answers "what is this", SKU answers "how we stock, price, and shelve it here". The generator read the hierarchy as a retailer merchandising hierarchy rather than a product classification. That was plausible because the hierarchy carries space policy (facings, elasticity, allowed space types, reset cycle).

That distinction is real in retail, since a retailer's merchandise hierarchy can diverge from GPC. Under the adopted design, though, the retailer-specific layer is the department → Product Category set mapping (CRCT-1, CRCT-2), not a per-SKU assignment. So nothing justifies the SKU-level link.

**Assumption.** No two SKUs of one product need different product-category policy. In a single-store sim this holds. It would matter only if SKUs later represent the same product across stores or formats, and then the difference belongs to the business unit, not the product category.

**Interactions with earlier items:**

- CRCT-2 ("SKUs attach to Bricks") and CRCT-3 ("the leaf category that SKUs attach to is the GPC Brick") are superseded on attachment: Products attach to Bricks, and SKUs inherit through their product.
- CRCT-1's `SKU.business_unit()` rule (the department whose set contains an ancestor-or-self of the SKU's product category) still holds, read through `sku.product.product_category`. CRCT-1's removal of `SKU.department` / SKU-INV-6 still stands.
- CRCT-2's frozen-as-attribute risk is unaffected in placement. If a mapping entry becomes "brick plus attribute value", the attribute is product-level as well.
- CRCT-5: the class name `CategoryNode` is left as CRCT-5 decided. The attribute is named `product_category` per the terminology rule.

**Recommended Changes to Contract Creation:**

- Prompt: `Product` references its Product Category (a GPC Brick, leaf only) as `product_category`. Product Category is a product-level classification fact, not a per-SKU merchandising choice.
- Prompt: `SKU` stores no product-category reference. `SKU.product_category` is a query delegated to `product`, like brand and manufacturer.
- Prompt: the product-category node's derived membership is its products. SKU sets under a node are derived through those products.
- Prompt: name the attribute `product_category`, never bare `category`.

**Effect on Contract Creation Products of Recommended Changes to Contract Creation:**

- 4.1 `Product`: creation gains `product_category`, with a new precondition and invariant that it is a Brick/leaf. PROD-INV-3 is unchanged.
- 4.2 `SKU`: `category` is removed from creation and stored queries, and `product_category` is added under "delegated to product". SKU-NEW-PRE-1 loses its category clause. SKU-INV-9 moves to `Product`. SKU-INV-10 adds `product_category == product.product_category`.
- 4.3 `CategoryNode`: `skus` becomes `products` (derived). CAT-INV-7 is restated over products. CAT-Q-5 `subtree_skus()` is derived via products.
- `s.category.policy()` becomes `s.product_category.policy()` in PLC-INV-10, POL-2, GEN-4, GEN-13, and D-6.
- ST-INV-4 and BUILD-PRE-4: the product's `product_category`, not the SKU's, must be in the hierarchy.
- Fixture EX-STORE-1: the `_PRODUCTS` rows carry the product category, and the SKU rows drop it. SKU examples (`SKU("SKU-X", a.product, a.category, ...)`) lose the category argument. The non-leaf counter-example moves to a `Product` creation.
- Discussion file: Q-3 is restated as Product-level and leaf-only, and a decision records the move with its single-store assumption.
