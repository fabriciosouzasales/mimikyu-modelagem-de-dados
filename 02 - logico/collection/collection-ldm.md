# MMKYU Collector — Logical Data Model (LDM)
## Domain: Collection

**Status:** In progress  
**Checkpoint:** LDM-01 through LDM-27 approved  
**Next decision:** LDM-28 — Removal of a member who still owns Inventory Items allocated to the Collection

---

## 1. Purpose

This document consolidates the approved logical modeling decisions for the **Collection** domain of MMKYU Collector.

It is the logical continuation of:

`01 - conceitual/collection/collection-concept-decisions.md`

Only the **current canonical decisions** are recorded. Intermediate proposals that were rejected or superseded are intentionally excluded.

### Current modeling status

- Conceptual model: **C-01 through C-37 — CLOSED**
- Logical model: **LDM-01 through LDM-27 — APPROVED**
- Physical model: **NOT STARTED**

---

# 2. Core Logical Principles

The model preserves the separation between:

- **Inventory Item:** physical card copy owned by a user.
- **Collection:** collecting objective to which a physical copy may be allocated.
- **Storage Container:** where the physical copy is stored.
- **Wishlist:** what the user wants to acquire.

> Owning, allocating, storing, completing and wishing are distinct concerns.

Every physical Inventory Item is based on exactly one **Card Variant**.

---

# 3. Approved Logical Decisions

## LDM-01 — Collection as Aggregate Root
`Collection` is a single aggregate root. Open curation, Card Set and Pokédex behaviors do not create independent root entities.

**Status:** APPROVED

## LDM-02 — Collection Ownership
Every Collection has exactly one explicit Owner through `owner_user_id`. Ownership is distinct from sharing/membership.

**Status:** APPROVED

## LDM-03 — Collection Member
Shared access is represented separately through `Collection Member`, relating Collection, User, permission profile and effective permissions. UX presets may simplify assignment, but effective permissions remain authoritative. The Owner is not simultaneously a normal Collection Member. Collection + User is unique. The complete permission matrix will be finalized after Collection Item/Inventory, Storage and Layout responsibilities are sufficiently modeled.

**Status:** APPROVED

## LDM-04 — Collection Mode and Reference
Collection has two primary modes:
- `OPEN_CURATION`
- `REFERENCE_BASED`

`STATIC/DYNAMIC` are characteristics of referenced universes, not Collection modes.

Rules:
- `OPEN_CURATION` → no Collection Reference.
- `REFERENCE_BASED` → exactly one Collection Reference.

**Status:** APPROVED

## LDM-05 — Adopted Scope for Dynamic References
Dynamic canonical references must not silently alter an existing Collection's objective when the canonical catalog evolves. The Collection explicitly persists the canonical positions it has adopted rather than relying only on version/count. The adopted scope is the authoritative denominator where applicable.

**Status:** APPROVED

## LDM-06 — Collection Reference and Explicit Subtypes
Reference-based Collections use a common `Collection Reference` with explicit subtypes:
- Collection Card Set Reference
- Collection Pokédex Reference

A loose polymorphic `reference_type + reference_id` structure is rejected. Each subtype uses a strong FK to its canonical entity.

**Status:** APPROVED

## LDM-07 — Reference Consolidation
A reference may be changed while the Collection has never received an Inventory Item. Current item count is insufficient because a Collection may have contained items and later become empty.

Collection persists `reference_locked_at`. On the first effective Inventory Item association, the reference is consolidated and `reference_locked_at` is set. In normal flow it never returns to `NULL`.

**Status:** APPROVED

## LDM-08 — Completion Policy
Mode, reference type and completion policy are independent.

Initial policies:
- `NONE`
- `STANDARD_SET`
- `MASTER_SET`
- `REFERENCE_POSITION`

Typical mapping:
- Open curation → `NONE`
- Card Set → `STANDARD_SET` or `MASTER_SET`
- Pokédex → `REFERENCE_POSITION`

Completion and progress remain derived.

**Status:** APPROVED

## LDM-09 — Lifecycle and Visibility
Independent dimensions:

Lifecycle:
- `ACTIVE`
- `ARCHIVED`

Visibility:
- `PRIVATE`
- `PUBLIC`

An archived Collection may remain public.

**Status:** APPROVED

## LDM-10 — Default Storage Container
Collection may define `default_storage_container_id`. It is an operational/UX default and does not mean every item must reside there. A Storage Container may be default for multiple Collections. Changing the default does not move existing Inventory Items.

**Status:** APPROVED

## LDM-11 — Audit Timestamps and Business Milestones
Technical audit:
- `created_at`
- `updated_at`

Business milestones:
- `started_at`
- `reference_locked_at`
- `archived_at`

`started_at` = first effective Inventory Item association and applies to open/reference-based Collections. `completed_at` is not persisted because completion is reversible.

**Status:** APPROVED

## LDM-12 — Collection Root Logical Skeleton

```text
Collection
├── id
├── game_id
├── owner_user_id
├── name
├── description
├── mode
├── completion_policy
├── lifecycle_status
├── visibility
├── default_storage_container_id
├── started_at
├── reference_locked_at
├── archived_at
├── created_at
├── created_by_user_id
├── updated_at
└── updated_by_user_id
```

`owner_user_id` differs from `created_by_user_id`; ownership transfer does not rewrite original authorship. `created_at/by + updated_at/by` is a candidate transversal standard, not yet automatically generalized.

**Status:** APPROVED

## LDM-13 — Collection Reference

```text
Collection Reference
├── id
├── collection_id
├── reference_kind
├── created_at
├── created_by_user_id
├── updated_at
└── updated_by_user_id
```

`collection_id` is unique. Initial kinds:
- `CARD_SET`
- `POKEDEX`

The concrete canonical identifier lives in the subtype. `reference_kind` is a discriminator, not a generic polymorphic reference. `game_id` is not duplicated.

**Status:** APPROVED

## LDM-14 — Collection Card Set Reference

```text
Collection Card Set Reference
├── collection_reference_id
└── card_set_id
```

`card_set_id` is a strong FK to canonical Card Set. The Card Set must belong to the same Game as Collection. Metadata is not duplicated. Completion policy stays on Collection. After reference consolidation, `card_set_id` is immutable in normal flow.

**Status:** APPROVED

## LDM-15 — Collection Pokédex Reference and Expandable Adopted Scope
A Pokédex Collection references canonical Pokédex via `pokedex_id`. Its effective objective is a separate Adopted Scope.

Initial scope may contain one generation, multiple generations, or the entire desired universe. Every Pokédex Collection is expandable through explicit user action.

Examples:
- Kanto → 151
- Kanto + Johto → 251
- Kanto + Johto + Hoenn → 386

Completion uses the currently adopted scope, never the whole current canonical Pokédex. In normal flow the Pokédex scope is monotonic: positions may be added but not removed.

**Status:** APPROVED

## LDM-16 — Pokédex Adopted Scope by Canonical Position
Scope references canonical `Pokédex Position`, not directly Pokémon.

Canonical dependency:

```text
Pokédex
└── Pokédex Position
    ├── id
    ├── pokemon_id
    └── position_number
```

Collection scope:

```text
Collection Pokédex Scope
├── collection_pokedex_reference_id
├── pokedex_position_id
├── adopted_at
└── adopted_by_user_id
```

Each position may be adopted at most once. Adoption metadata preserves when/by whom it entered the objective. Pokédex and Pokédex Position belong to their canonical domain, not Collection.

**Status:** APPROVED

## LDM-17 — Inventory Item Eligibility
Eligibility validates only the canonical universe; there is no arbitrary user-defined rule engine.

- Open Curation: no canonical-universe restriction.
- Card Set: Inventory Item's Card must belong to referenced Card Set.
- Pokédex: Card's principal Pokémon must correspond to a Pokédex Position in the Adopted Scope.

Language, rarity, variant and aesthetic preferences do not independently restrict eligibility unless a future explicit completion requirement uses them. Eligibility is derived, not stored as `is_eligible`. Eligibility and completion are independent.

**Status:** APPROVED

## LDM-18 — Card to Pokémon Relationship for Pokédex Eligibility
Every Card classified as Pokémon is associated with exactly one canonical Pokémon: the principal Pokémon in evidence. Incidental Pokémon in artwork do not generate eligibility.

```text
Card (category = POKEMON)
└── pokemon_id → exactly one canonical Pokemon
```

Eligibility path:

```text
Inventory Item
→ Card Variant
→ Card
→ pokemon_id
=
Pokédex Position
→ pokemon_id
```

The earlier N:N Card ↔ Pokémon hypothesis is superseded and must not be implemented.

**Status:** APPROVED

## LDM-19 — Inventory Item Always Originates from Card Variant
Every physical Inventory Item references exactly one Card Variant regardless of Collection type.

```text
Inventory Item
→ Card Variant
→ Card
```

Completion projection:
- `STANDARD_SET`: Card Variant → Card
- `MASTER_SET`: Card Variant
- `REFERENCE_POSITION`: Card Variant → Card → Pokemon → Pokédex Position

Requirement satisfaction is derived rather than persisted as a second source of truth.

**Status:** APPROVED

## LDM-20 — Completion Denominator
Requirements depend on `completion_policy`:

- `NONE`: no denominator.
- `STANDARD_SET`: each Card of referenced Card Set is a requirement.
- `MASTER_SET`: Card Variants explicitly selected in the Collection's Master Set Adopted Scope.
- `REFERENCE_POSITION`: each Pokédex Position in Adopted Scope.

Numerator = distinct requirements satisfied by at least one Inventory Item allocated to Collection. Duplicates do not create additional satisfied requirements. Counts/percentages may later be materialized for performance but are not canonical truth.

**Status:** APPROVED

## LDM-21 — Master Set Adopted Scope
A `MASTER_SET` Collection has a user-defined Master Set Adopted Scope.

```text
Collection Master Set Scope
├── collection_id
├── card_variant_id
├── adopted_at
└── adopted_by_user_id
```

The source of truth is explicit individual `card_variant_id` selection. Variant types/presets/bulk tools are UX mechanisms only.

The user may include/exclude special variants such as Jumbo, League, Tournament and others. Scope may expand or shrink. Removing a variant changes completion requirements only; it does not delete Inventory Items or remove them from Collection.

Full historical changes belong to Audit Log.

The canonical Card Variant list of a validated Card Set is stable after its initial validated catalog load in normal operation. Later corrections, if ever required, are exceptional catalog-governance events, not normal Collection evolution.

**Status:** APPROVED

## LDM-22 — Completion Policy Changes and Master Set Scope Redefinition
Card Set Collections may change:
- `STANDARD_SET ↔ MASTER_SET`

Changing policy does not modify/remove Inventory Items.

When switching to Master Set, the user explicitly validates the Master Set Adopted Scope. When switching to Standard Set, prior Master Set Scope may be preserved but is inactive. If returning to Master Set, it may be restored.

While remaining `MASTER_SET`, the user may redefine Scope at any time by including/excluding existing canonical Card Variants.

Distinction:
1. Completion policy change: `STANDARD_SET ↔ MASTER_SET`
2. Completion scope change: policy stays `MASTER_SET`, adopted variants change.

Denominator changes are conscious user changes to the collecting objective, not automatic catalog expansion.

**Status:** APPROVED

## LDM-23 — Single Canonical Identity for the Physical Item
The physical copy has one canonical Inventory identity. Association with Collection does not create a second physical identity.

```text
Inventory Item
├── id
├── owner_user_id
├── card_variant_id
├── collection_id (0..1)
└── storage_container_id (0..1)
```

An Inventory Item associated with Collection plays the contextual role previously described as a `Collection Item`.

It may exist without Collection, enter one, leave it, or move to another while retaining identity. It may belong to at most one Collection at a time. Collection allocation and physical Storage are independent.

**Status:** APPROVED

## LDM-24 — Inventory Item and Storage Container
Every Inventory Item must reference exactly one Card Variant. Storage is optional (`0..1`).

An item may temporarily have no defined location, supporting recent acquisitions, bulk imports, temporary reorganization or unknown location.

No Storage does not prevent Collection association or completion contribution. UX should favor contextual/default Storage assignment, especially One-Click and bulk workflows, without making Storage structurally mandatory.

**Status:** APPROVED

## LDM-25 — Inventory Item Ownership
Every Inventory Item has exactly one explicit Owner, independent of Collection ownership.

A shared Collection may contain items owned by different authorized members.

Transferring Collection ownership does not transfer Inventory Items. Item ownership transfer is independent. To associate another user's item, that Owner must have an authorized relationship with Collection.

Storage ownership remains a separate concern.

**Status:** APPROVED

## LDM-26 — Inventory Item Ownership Transfer
Ownership transfer preserves physical identity and does not automatically change:
- Card Variant
- Storage Container
- Collection association

If new Owner is already authorized in current Collection, the item may remain. If not, transfer must not silently create an invalid state or silently remove the item; incompatibility must be explicitly resolved before completion.

Physical ownership transfer and Collection reallocation are distinct.

**Status:** APPROVED

## LDM-27 — Operational Authority and Approval for Patrimonial Actions
Authority over an Inventory Item in a shared Collection has two dimensions.

### Collection operations
Collection permissions govern the item's role/organization inside Collection. Removing an item from Collection does not delete Inventory Item.

### Patrimonial/physical operations
Inventory Item Owner retains final authority over operations affecting ownership or existence of the physical asset.

Collection Owner may initiate a patrimonial operation over another member's item, but it is not executed immediately:

1. approval request is created;
2. Inventory Item Owner receives it in their inbox;
3. Owner approves or rejects;
4. no patrimonial state changes while pending;
5. approval executes the authorized operation;
6. rejection cancels it.

No self-approval is required when Collection Owner = Inventory Item Owner.

Examples include ownership transfer, deletion, and future operations materially affecting ownership/existence.

Future transversal dependency:
**Pending Action / Approval Request + User Inbox / Notification Center**

This should be platform-level, not Collection-specific.

**Status:** APPROVED

---

# 4. Canonical Relationship Summary

```text
Collection
├── Owner
├── Members
├── Default Storage Container
└── Collection Reference (0..1)
    ├── Card Set Reference
    │   └── Card Set
    └── Pokédex Reference
        ├── Pokédex
        └── Adopted Scope
            └── Pokédex Positions

Inventory Item
├── Owner
├── Card Variant
│   └── Card
│       ├── Card Set
│       └── Pokemon (when category = POKEMON)
├── Collection (0..1)
└── Storage Container (0..1)

MASTER_SET Collection
└── Master Set Adopted Scope
    └── selected Card Variants
```

---

# 5. Completion Model Summary

```text
NONE
→ no completion calculation

STANDARD_SET
→ denominator = Cards of referenced Card Set
→ Inventory Item → Card Variant → Card

MASTER_SET
→ denominator = Card Variants selected in Master Set Adopted Scope
→ Inventory Item → Card Variant

REFERENCE_POSITION / POKEDEX
→ denominator = Pokédex Positions in Adopted Scope
→ Inventory Item → Card Variant → Card → Pokemon → Pokédex Position
```

Completion is derived. Inventory quantity is not equivalent to completion progress.

---

# 6. Superseded / Rejected Logical Hypotheses

Do **not** implement:

1. Generic polymorphic `reference_type + reference_id`.
2. `STATIC/DYNAMIC` as Collection modes.
3. Reference lock derived only from current item count.
4. Persisted `is_eligible` as canonical truth.
5. Arbitrary user-defined eligibility rule engine.
6. Card ↔ Pokémon N:N for Pokédex eligibility.
7. Incidental artwork Pokémon satisfying Pokédex positions.
8. A second physical identity solely because an Inventory Item joins a Collection.
9. Every canonical Card Variant being automatically mandatory in every Master Set.
10. Automatic Master Set denominator changes caused by normal catalog expansion.
11. Structurally mandatory Storage for Inventory Item creation.
12. Automatic Inventory Item transfer when Collection ownership changes.
13. Unconditional patrimonial authority for Collection Owner over items owned by other members.

---

# 7. Dependencies Identified for Other Domains

## Canonical Catalog
- Card
- Card Variant
- Card Set
- Pokemon
- Pokédex
- Pokédex Position

Invariant: every Card classified as Pokémon identifies exactly one principal canonical Pokemon.

## Inventory
Inventory Item requires its own detailed model beyond the Collection-allocation decisions captured here.

## Storage
Storage ownership, sharing, physical organization and movement require their own model.

## Permissions
Complete permission matrix will be finalized after Collection, Inventory, Storage and Layout responsibilities are sufficiently defined.

## Audit
A transversal Audit Log should preserve meaningful changes without forcing every business relation into a temporal table.

## Approval / Messaging
A transversal Pending Action / Approval Request mechanism and User Inbox / Notification Center are required for multi-user operations requiring explicit approval.

---

# 8. Current Architectural Checkpoint

## Conceptual
**C-01 through C-37 — CLOSED**

Canonical document:
`01 - conceitual/collection/collection-concept-decisions.md`

## Logical
**LDM-01 through LDM-27 — APPROVED**

This document is the canonical logical checkpoint.

## Physical
**NOT STARTED**

---

# 9. Exact Point of Resumption

Next decision:

## LDM-28 — Removing a Collection Member Who Still Owns Inventory Items Allocated to the Collection

It must define what happens when a user is to be removed from Collection membership while one or more Inventory Items owned by that user remain allocated to Collection.

It must preserve:
- independent Inventory Item ownership;
- authorized Collection membership;
- no silent transfer of physical ownership;
- no silent invalid states;
- explicit approval for patrimonial operations when applicable.

> Resume from LDM-28 using LDM-01 through LDM-27 in this document as the approved logical baseline.
