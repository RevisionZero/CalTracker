# Food Data Layer — Specification

**Status:** Foundational / stable
**Scope:** The base persistence layer. Two tables, and the rules that govern them.

---

## 1. Motivation

This layer exists to be the part of the application that never needs to be rebuilt. Everything the user will ever see — daily totals, meals, weekly trends, macro splits, recipes, goal tracking, streaks — is a **view computed over these two tables**, never a parallel truth stored alongside them. If a fact can be derived, it is not stored. The tables hold the minimal set of mutually independent facts from which the entire application can be reconstructed, and nothing else.

The single most important property is that **a logged entry is self-contained and historically immutable by default**. A `FoodEntry` does not reference its nutrition; it *contains* it, snapshotted at the moment of logging. Nothing external — not a corrected food definition, not a database refresh, not a third-party API — can silently rewrite what the user recorded on a past day. This is the same principle that makes an invoice line item store the price it was sold at rather than joining to a live product table. It is the one place where deliberate denormalization is not a compromise but the entire point.

Correction remains possible, but it is always **explicit, scoped, and user-initiated**. When a food definition changes, the user decides whether history follows, and the version-cohort mechanism (§7) makes that decision precise rather than all-or-nothing. Every remaining design choice follows from removing degrees of freedom: the measurement basis is immutable, so nutrition can never be reinterpreted under a different scale; nutrition is anchored to that basis rather than to a serving, so serving definitions can be corrected without touching a calorie; identity is a UUID, so nothing depends on a name that might be edited. Each field that *cannot* drift is a category of bug that cannot exist.

---

## 2. Design principles

| Principle | Consequence in the schema |
|---|---|
| **Derive, don't store** | No daily totals, meal rows, or week rows. `servings` is computed, not persisted. |
| **Snapshot at log time** | `FoodEntry` duplicates `FoodType` nutrition columns. Intentional. |
| **Anchor to an immutable basis** | Nutrition is per `basisQty × basisUnit`, never per serving. `basisUnit` is fixed at creation. |
| **Identity is opaque** | UUID primary keys. Names are display data and may change freely. |
| **Deletion is an event** | Soft deletes via `deletedAt`. Rows are never physically removed. |
| **Mutation is explicit** | History changes only when the user asks, scoped to a version cohort. |

---

## 3. Terminology

- **Basis** — the immutable measurement anchor for a food's nutrition. Either 100 g, 100 ml, or 1 unit. All nutrition values are expressed *per basis*.
- **Basis unit** — `g`, `ml`, or `unit`. Fixed at creation and never changeable.
- **Unit-basis food** — a food measured in discrete countable items (bars, packets, capsules, slices). `basisUnit = 'unit'`, `basisQty = 1`.
- **Uniqueness-breaking change** — an edit to any field in `{calories, protein, carbs, fats, otherNutrition}`. These define a food's nutritional identity; changing any of them produces a materially different food and increments `versionNo`.
- **Snapshot** — the block of `FoodType` columns copied into a `FoodEntry` at creation time.
- **Version cohort** — the set of entries sharing a `(foodTypeId, versionNo)` pair. The unit of scoped retroactive update.
- **Custom entry** — a `FoodEntry` with no parent `FoodType`. Self-contained, never participates in propagation.

---

## 4. Table: `FoodType`

A reusable food definition. Mutable, versioned. Never referenced for nutrition at read time — only for creating and updating entries.

| Column | Type | Null? | Mutability | Description |
|---|---|---|---|---|
| `id` | TEXT (UUID) | NOT NULL | Immutable | Primary key. Client-generated UUID. |
| `name` | TEXT | NOT NULL | Mutable | User-assigned display name. Not an identity field. |
| `updatedAt` | INTEGER | NOT NULL | System | Unix time. Set at creation, refreshed on every write. |
| `deletedAt` | INTEGER | **Nullable** | System | Unix time. `NULL` means live. Non-null is a tombstone. |
| `basisUnit` | TEXT | NOT NULL | **Immutable** | One of `g`, `ml`, `unit`. Derived from user input at creation. |
| `basisQty` | REAL | NOT NULL | **Immutable** | `100` when `basisUnit ∈ {g, ml}`; `1` when `basisUnit = 'unit'`. Auto-assigned. |
| `unitLabel` | TEXT | Conditional | Mutable | Required iff `basisUnit = 'unit'` (e.g. `"bar"`, `"packet"`). Must be `NULL` otherwise. |
| `unitConvTo` | TEXT | Optional | Mutable | `g` or `ml`. The target unit for cross-expression. `NULL` unless `basisUnit = 'unit'`. |
| `unitConvAmt` | REAL | Optional | Mutable | Amount in `unitConvTo` units equal to one basis unit (e.g. one bar = 65 g). |
| `servingSize` | REAL | NOT NULL | Mutable | Expressed **in basis units**. Display convenience only; does not affect nutrition. |
| `versionNo` | INTEGER | NOT NULL | System | Starts at 1. Incremented **only** on uniqueness-breaking changes. |
| `calories` | REAL | NOT NULL | Mutable | Per `basisQty`. |
| `protein` | REAL | NOT NULL | Mutable | Per `basisQty`, grams. |
| `carbs` | REAL | NOT NULL | Mutable | Per `basisQty`, grams. |
| `fats` | REAL | NOT NULL | Mutable | Per `basisQty`, grams. |
| `otherNutrition` | TEXT (JSON) | NOT NULL | Mutable | Per `basisQty`. Sparse micronutrient map. Defaults to `{}`. |

**Notes**

- `unitConvTo` / `unitConvAmt` are **both-or-neither**. An amount without a target unit is meaningless.
- Conversion fields are optional even for unit-basis foods. Their absence only removes the ability to cross-express in mass/volume; all nutrition math still works.
- `basisQty` is formally redundant (fully determined by `basisUnit`) but stored explicitly to keep every calculation self-describing and to leave room for non-100 bases later. A CHECK constraint keeps it honest.

---

## 5. Table: `FoodEntry`

A single logged consumption event. Self-contained: carries everything needed to compute its own nutrition without joining to `FoodType`.

| Column | Type | Null? | Mutability | Description |
|---|---|---|---|---|
| `id` | TEXT (UUID) | NOT NULL | Immutable | Primary key. Client-generated UUID. |
| `name` | TEXT | NOT NULL | Mutable | Initialized from `FoodType.name` at creation, then independent. Preserves what the food was called when logged. |
| `foodTypeId` | TEXT (UUID) | Optional | Immutable | FK to `FoodType.id`. `NULL` for custom entries. |
| `loggedAt` | INTEGER | NOT NULL | Immutable | Unix time the record was created. A system fact. |
| `updatedAt` | INTEGER | NOT NULL | System | Unix time. Initialized equal to `loggedAt`, refreshed on every write. |
| `deletedAt` | INTEGER | **Nullable** | System | Unix time. `NULL` means live. |
| `logDate` | TEXT | NOT NULL | **Immutable** | `YYYY-MM-DD`. The day this food counts toward. User-assigned at creation; never editable afterward. |
| `versionNo` | INTEGER | Conditional | System | The `FoodType.versionNo` at snapshot time. `NULL` iff custom entry. Never modified by user edits. |
| `edited` | INTEGER (bool) | Conditional | System | `NULL` iff custom entry. `false` at creation. Set `true` on a uniqueness-breaking edit to this entry. Reset to `false` on re-pull. |

### Snapshot block

Copied from `FoodType` at creation. Mutable on the entry only where mutable on the parent.

| Column | Type | Mutability | Notes |
|---|---|---|---|
| `basisUnit` | TEXT | **Immutable** | Immutable on parent, therefore immutable here. |
| `basisQty` | REAL | **Immutable** | As above. |
| `unitLabel` | TEXT | Mutable | |
| `unitConvTo` | TEXT | Mutable | |
| `unitConvAmt` | REAL | Mutable | |
| `servingSize` | REAL | Mutable | In basis units. Editing does **not** set `edited`. |
| `calories` | REAL | Mutable | Editing **sets `edited = true`**. |
| `protein` | REAL | Mutable | Editing **sets `edited = true`**. |
| `carbs` | REAL | Mutable | Editing **sets `edited = true`**. |
| `fats` | REAL | Mutable | Editing **sets `edited = true`**. |
| `otherNutrition` | TEXT (JSON) | Mutable | Editing **sets `edited = true`**. |

### Quantity

| Column | Type | Null? | Mutability | Description |
|---|---|---|---|---|
| `totalAmt` | REAL | NOT NULL | Mutable | Total consumed, **in basis units** (e.g. `200` for 200 g, `2` for 2 bars, `250` for 250 ml). Not a multiplier. |

`totalAmt` is the sole quantity field. Servings are derived (§9), never stored.

---

## 6. Invariants

Enforce at the database layer, not in application code — these are the constraints that leak the moment a second write path is added (import, sync, bulk edit).

**Both tables**

1. `basisUnit ∈ {'g', 'ml', 'unit'}`
2. `basisQty = 1` iff `basisUnit = 'unit'`; `basisQty = 100` iff `basisUnit ∈ {'g','ml'}`
3. `unitLabel IS NOT NULL` iff `basisUnit = 'unit'`
4. `(unitConvTo IS NULL) = (unitConvAmt IS NULL)` — both or neither
5. `basisUnit = 'unit'` OR `unitConvTo IS NULL` — no g→g conversions
6. `unitConvTo ∈ {'g','ml'}` when present
7. `servingSize > 0`; nutrition values `>= 0`

**`FoodEntry` only**

8. `foodTypeId`, `versionNo`, and `edited` are **null-together or non-null-together**
9. `totalAmt > 0`
10. `logDate` matches `^\d{4}-\d{2}-\d{2}$`
11. An entry's `basisUnit` and `basisQty` always equal its parent's (guaranteed by parent immutability, worth asserting in tests)

**Read-path rule**

12. Every user-facing query filters `deletedAt IS NULL`. `edited`'s three states are all load-bearing (`NULL` = custom, `false` = intact, `true` = diverged) — never treat `NULL` as falsy.

---

## 7. Entry state model

Every `FoodEntry` occupies exactly one of four states.

| State | Condition | Meaning |
|---|---|---|
| **CUSTOM** | `foodTypeId IS NULL` | Ad-hoc entry. No parent, no propagation, ever. |
| **SYNCED** | `versionNo = parent.versionNo` AND `edited = false` | Matches the current definition. |
| **STALE** | `versionNo < parent.versionNo` AND `edited = false` | Parent has moved on. Eligible for propagation. |
| **DIVERGED** | `edited = true` | Hand-adjusted. Excluded from propagation until re-pulled. |

### Transitions

```
                    create from FoodType
                              │
                              ▼
                          ┌────────┐
      re-pull ───────────►│ SYNCED │
         ▲                └───┬────┘
         │                    │ parent uniqueness-breaking edit
         │                    ▼
         │                ┌───────┐
         ├────────────────┤ STALE │
         │   propagate    └───┬───┘
         │                    │
         │      user edits nutrition (either state)
         │                    ▼
         │              ┌──────────┐
         └──────────────┤ DIVERGED │
             re-pull    └──────────┘
```

`CUSTOM` has no transitions — it is terminal by design.

---

## 8. Processes

### P1 — Create a `FoodType`

1. User supplies name, measurement basis, nutrition, serving size.
2. Derive `basisUnit` from the user's chosen measurement mode; auto-assign `basisQty` (100 or 1).
3. If `basisUnit = 'unit'`, require `unitLabel`. Optionally accept `unitConvTo` / `unitConvAmt`.
4. Set `versionNo = 1`, `updatedAt = now`, `deletedAt = NULL`.

> **`basisUnit` is chosen once and is unchangeable.** Changing it later is not an edit but a migration that invalidates every dependent snapshot. Surface this clearly in the creation UI. Guidance: use `unit` for manufactured discrete items (bars, packets, capsules); use `g`/`ml` for produce, bulk goods, meat, and liquids, where the "natural unit" varies between instances.

### P2 — Create a `FoodEntry` from a `FoodType`

1. Copy the entire snapshot block from the parent verbatim.
2. Copy `name` from the parent.
3. Set `foodTypeId = parent.id`, `versionNo = parent.versionNo`, `edited = false`.
4. Set `totalAmt` from user input, in basis units.
5. Set `logDate` (defaults to today in the device timezone; user may choose another date). **Immutable thereafter** — a mis-dated entry is corrected by deleting and re-creating.
6. Set `loggedAt = updatedAt = now`.

Resulting state: **SYNCED**.

### P3 — Create a custom `FoodEntry`

Identical to P2 except `foodTypeId`, `versionNo`, and `edited` are all `NULL`, and the snapshot fields are supplied directly by the user. Resulting state: **CUSTOM**.

> Custom entries never participate in propagation. Surface this at creation so users don't expect later corrections to reach them.

### P4 — Edit a `FoodType` (non-uniqueness-breaking)

Applies to `name`, `servingSize`, `unitLabel`, `unitConvTo`, `unitConvAmt`.

1. Write the new values, update `updatedAt`.
2. **Do not** increment `versionNo`.
3. Existing entries are untouched and remain SYNCED.

Rationale: these fields carry no nutritional meaning. Bumping the version for a rename would prompt the user to make a retroactive decision about nutrition that did not change.

### P5 — Edit a `FoodType` (uniqueness-breaking)

Applies to `calories`, `protein`, `carbs`, `fats`, `otherNutrition`.

1. Write the new values, `versionNo += 1`, update `updatedAt`.
2. All SYNCED entries at the previous version silently become **STALE**.
3. Offer propagation (P6). If the user declines, history simply stands as recorded.

### P6 — Retroactive propagation

Triggered by explicit user choice, normally right after P5.

**Scope options**

| Option | Predicate |
|---|---|
| All prior entries | `foodTypeId = ? AND edited = 0 AND deletedAt IS NULL` |
| One version cohort | `foodTypeId = ? AND versionNo = ? AND edited = 0 AND deletedAt IS NULL` |
| From a date forward | above, plus `logDate >= ?` |
| Going forward only | no update performed |

**Execution** — single transaction:

```sql
UPDATE FoodEntry
SET calories = ?, protein = ?, carbs = ?, fats = ?, otherNutrition = ?,
    servingSize = ?, unitLabel = ?, unitConvTo = ?, unitConvAmt = ?,
    versionNo = ?, updatedAt = ?
WHERE foodTypeId = ?
  AND versionNo = ?          -- omit for "all prior"
  AND edited = 0
  AND deletedAt IS NULL;
```

No scale conversion or unit parsing is required, because `basisUnit` is immutable and both tables express nutrition per `basisQty`.

> **Propagation copies the full snapshot block, not only the nutrition fields.** A non-diverged entry is by definition tracking its parent, so it should track all of it. `name` is deliberately excluded — it preserves what the user called the food at log time.

> **This operation is irreversible.** Superseded values are not retained anywhere; the entries themselves were the only record of them. This is an accepted constraint: restoring old values is done by the user re-entering them, not by the app remembering. Because of this, the confirmation dialog must state the blast radius concretely — *"This will update 412 entries from 3 Jan to 12 Aug, including 47 at an older version."*

### P7 — Edit a `FoodEntry` snapshot directly

1. Write the new values, update `updatedAt`.
2. If any field in `{calories, protein, carbs, fats, otherNutrition}` changed → set `edited = true`.
3. If only `servingSize`, `unitLabel`, `unitConvTo`, or `unitConvAmt` changed → leave `edited` unchanged.
4. `versionNo` is never modified here. It permanently records the version this entry originally snapshotted.

Step 3 matters: marking an entry diverged for a cosmetic serving-label tweak would silently exclude it from all future nutrition corrections, with no signal to the user.

Resulting state: **DIVERGED** if step 2 fired.

### P8 — Re-pull an entry to the current `FoodType`

The escape hatch that returns a DIVERGED or STALE entry to SYNCED.

1. Overwrite the full snapshot block from the current parent.
2. Set `versionNo = parent.versionNo`, `edited = false`, `updatedAt = now`.
3. Leave `totalAmt`, `logDate`, `loggedAt`, and `name` untouched.

> Requires confirmation when the entry is DIVERGED — the user's hand-entered values are discarded and not recoverable.

### P9 — Soft-delete a `FoodType`

1. Set `deletedAt = now`. Exclude from search and autocomplete.
2. **Existing entries are unaffected** and continue to render and sum correctly — they are self-contained. This falls out of the snapshot design rather than requiring special handling.
3. Do not cascade. Do not hard-delete.

### P10 — Soft-delete a `FoodEntry`

Set `deletedAt = now`. Excluded from all totals and views. Retained for undo and for sync tombstoning.

---

## 9. Derived computations

Nothing below is stored.

**Nutrient total for one entry** — for each of calories, protein, carbs, fats, and each key in `otherNutrition`:

```
nutrientTotal = (totalAmt / basisQty) × nutrientPerBasis
```

**Servings (display only)**

```
servings = totalAmt / servingSize
```

**Mass / volume equivalent** — unit-basis foods with conversion data:

```
equivalent = totalAmt × unitConvAmt        -- expressed in unitConvTo units
```

**Daily total**

```sql
SELECT
  SUM(totalAmt / basisQty * calories) AS kcal,
  SUM(totalAmt / basisQty * protein) AS protein,
  SUM(totalAmt / basisQty * carbs)   AS carbs,
  SUM(totalAmt / basisQty * fats)    AS fats
FROM FoodEntry
WHERE logDate = ?
  AND deletedAt IS NULL;
```

Mixed-basis days sum correctly without special handling: the basis unit never enters the arithmetic, only the ratio does.

**Weekly / range totals** — identical, with `logDate BETWEEN ? AND ?` and a `GROUP BY logDate`.

> Use `REAL` for all numeric columns. Integer division on `totalAmt / basisQty` would silently truncate every partial-basis entry to zero.

---

## 10. Indexes

```sql
CREATE INDEX idx_entry_logdate   ON FoodEntry(logDate)              WHERE deletedAt IS NULL;
CREATE INDEX idx_entry_type_ver  ON FoodEntry(foodTypeId, versionNo) WHERE deletedAt IS NULL;
CREATE INDEX idx_entry_name      ON FoodEntry(name)                  WHERE deletedAt IS NULL;
CREATE INDEX idx_type_name       ON FoodType(name)                   WHERE deletedAt IS NULL;
```

| Index | Serves |
|---|---|
| `FoodEntry(logDate)` | Hottest filter in the app — every day and week view. |
| `FoodEntry(foodTypeId, versionNo)` | Propagation cohort selection; "all entries of this food". |
| `FoodEntry(name)` | Autocomplete over ad-hoc entries, so custom entries are re-loggable without ever becoming a `FoodType`. |
| `FoodType(name)` | Primary search and autocomplete. |

Partial indexes (`WHERE deletedAt IS NULL`) keep tombstones out of the hot paths entirely, which removes the need for a separate index on `deletedAt`.

---

## 11. Deliberate exclusions

Recorded so their absence reads as a decision rather than an oversight.

| Excluded | Reasoning |
|---|---|
| Version-history table | Entries *are* the version history — any entry at v2 records what v2 contained. A separate table would duplicate that. |
| Undo log for propagation | Accepted constraint (§P6). Fewer degrees of freedom, less state. |
| `servings` column | Derivable from `totalAmt / servingSize`. |
| Stored daily/weekly totals | Derivable. Materializing them creates a second, drift-prone source of truth. |
| `brand`, `mealSlot` | Real features, but belong to layers above this one. `brand` is name disambiguation; `mealSlot` is a grouping annotation. Neither is part of what is nutritionally true about an entry. |
| `createdAt` | `loggedAt` covers entries; `updatedAt` covers types. |
| `source`, `externalId` | For third-party food database ingestion. Add when that layer is built. |
| Editable `logDate` | Accepted constraint. Mis-dated entries are deleted and re-created. |

---

## Appendix A — DDL (SQLite)

```sql
CREATE TABLE FoodType (
  id             TEXT    PRIMARY KEY NOT NULL,
  name           TEXT    NOT NULL,
  updatedAt      INTEGER NOT NULL,
  deletedAt      INTEGER,
  basisUnit      TEXT    NOT NULL,
  basisQty       REAL    NOT NULL,
  unitLabel      TEXT,
  unitConvTo     TEXT,
  unitConvAmt    REAL,
  servingSize    REAL    NOT NULL,
  versionNo      INTEGER NOT NULL DEFAULT 1,
  calories       REAL    NOT NULL,
  protein        REAL    NOT NULL,
  carbs          REAL    NOT NULL,
  fats           REAL    NOT NULL,
  otherNutrition TEXT    NOT NULL DEFAULT '{}',

  CHECK (basisUnit IN ('g','ml','unit')),
  CHECK ((basisUnit = 'unit' AND basisQty = 1)
      OR (basisUnit IN ('g','ml') AND basisQty = 100)),
  CHECK ((basisUnit = 'unit') = (unitLabel IS NOT NULL)),
  CHECK ((unitConvTo IS NULL) = (unitConvAmt IS NULL)),
  CHECK (basisUnit = 'unit' OR unitConvTo IS NULL),
  CHECK (unitConvTo IS NULL OR unitConvTo IN ('g','ml')),
  CHECK (unitConvAmt IS NULL OR unitConvAmt > 0),
  CHECK (servingSize > 0),
  CHECK (versionNo >= 1),
  CHECK (calories >= 0 AND protein >= 0 AND carbs >= 0 AND fats >= 0)
);

CREATE TABLE FoodEntry (
  id             TEXT    PRIMARY KEY NOT NULL,
  name           TEXT    NOT NULL,
  foodTypeId     TEXT    REFERENCES FoodType(id),
  loggedAt       INTEGER NOT NULL,
  updatedAt      INTEGER NOT NULL,
  deletedAt      INTEGER,
  logDate        TEXT    NOT NULL,
  versionNo      INTEGER,
  edited         INTEGER,

  basisUnit      TEXT    NOT NULL,
  basisQty       REAL    NOT NULL,
  unitLabel      TEXT,
  unitConvTo     TEXT,
  unitConvAmt    REAL,
  servingSize    REAL    NOT NULL,
  calories       REAL    NOT NULL,
  protein        REAL    NOT NULL,
  carbs          REAL    NOT NULL,
  fats           REAL    NOT NULL,
  otherNutrition TEXT    NOT NULL DEFAULT '{}',
  totalAmt       REAL    NOT NULL,

  CHECK (basisUnit IN ('g','ml','unit')),
  CHECK ((basisUnit = 'unit' AND basisQty = 1)
      OR (basisUnit IN ('g','ml') AND basisQty = 100)),
  CHECK ((basisUnit = 'unit') = (unitLabel IS NOT NULL)),
  CHECK ((unitConvTo IS NULL) = (unitConvAmt IS NULL)),
  CHECK (basisUnit = 'unit' OR unitConvTo IS NULL),
  CHECK (unitConvTo IS NULL OR unitConvTo IN ('g','ml')),
  CHECK (unitConvAmt IS NULL OR unitConvAmt > 0),
  CHECK (servingSize > 0),
  CHECK (totalAmt > 0),
  CHECK (calories >= 0 AND protein >= 0 AND carbs >= 0 AND fats >= 0),
  CHECK (edited IS NULL OR edited IN (0,1)),
  CHECK (logDate GLOB '[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]'),
  CHECK (
    (foodTypeId IS NULL     AND versionNo IS NULL     AND edited IS NULL)
 OR (foodTypeId IS NOT NULL AND versionNo IS NOT NULL AND edited IS NOT NULL)
  )
);
```

---

## Appendix B — Worked examples

**Bulk food (gram basis)**

```
FoodType: name="Rolled Oats", basisUnit='g', basisQty=100, unitLabel=NULL,
          servingSize=40, versionNo=1, calories=389, protein=16.9, carbs=66.3, fats=6.9

FoodEntry: totalAmt=60, versionNo=1, edited=false
  calories → (60 / 100) × 389 = 233.4 kcal
  servings → 60 / 40 = 1.5
```

**Discrete food (unit basis)**

```
FoodType: name="Ice Cream Bar", basisUnit='unit', basisQty=1, unitLabel="bar",
          unitConvTo='g', unitConvAmt=65, servingSize=1, versionNo=1, calories=240

FoodEntry: totalAmt=2, versionNo=1, edited=false
  calories   → (2 / 1) × 240 = 480 kcal
  servings   → 2 / 1 = 2
  equivalent → 2 × 65 = 130 g
```

**Version divergence**

```
t0  FoodType v1, calories=240.  Entry A logged → versionNo=1, edited=false   [SYNCED]
t1  Recipe reformulated: calories → 260, versionNo=2.  Entry A now           [STALE]
    Entry B logged → versionNo=2, edited=false                               [SYNCED]
t2  User hand-corrects Entry B to 255 kcal → edited=true                     [DIVERGED]
t3  Propagation over "all prior entries":
      Entry A → updated to 260, versionNo=2                                  [SYNCED]
      Entry B → skipped (edited = 1)                                         [DIVERGED]
```
