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
