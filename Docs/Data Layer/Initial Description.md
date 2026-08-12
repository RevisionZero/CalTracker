2 Base Tables: Food Type, Food Entry

All fields are mandatory unless otherwise specified.

Food Type Columns:
- id: UUID DB id, unique, immutable, primary key
- name: Mutable, user assigned
- updatedAt: UNIX time, automatically assigned at creation time
- deletedAt: UNIX time, null by default
- basisUnit: immutable, derived from user entry, only options are g, ml, or unit
- basisQty: auto assigned, immutable, 100 for g and ml, 1 for unit
- unitLabel: Mandatory if unit is not g or ml, Mutable, user assigned
- unitConvTo: Must be null if unit is g or ml, Optional, must exist if unitConvAmt exists, indicates unit conversion to either g or ml
- unitConvAmt: Must be null if unit is g or ml, Optional, must exist if unitConvTo exists, indicates unit conversion amount using unitConvTo specified g or ml
- servingSize: Mutable, user assigned, in basis unit
- versionNo: Immutable by user, auto assigned based on edits that break uniqueness(calories, protein, carbs, fats, otherNutrition)
- calories: Mutable, user assigned, specified per basisQty
- protein: Mutable, user assigned, specified per basisQty
- carbs: Mutable, user assigned, specified per basisQty
- fats: Mutable, user assigned, specified per basisQty
- otherNutrition: Mutable, user assigned, specified per basisQty, JSON object

Food Entry Columns:
- id: UUID DB id, unique, immutable, primary key
- name: Mutable, user assigned
- foodTypeId: Optional, foreign key to food type id
- loggedAt: UNIX time
- updatedAt: UNIX time, automatically copied from loggedAt at creation time(both are initialized as the same) 
- deletedAt: UNIX time, null by default
- logDate: Immutable by system or user, user assigned, type is TEXT as YYYY-MM-DD
- versionNo: Mandatory if entry is from food type, null if it’s a custom entry. Immutable by user, snapshotted from food type at creation time
- edited: If this is a custom entry, it’s null. If not, it’s automatically set to false.Indicates if any uniqueness breaking changes(detailed below) are made in the food entry instead of the food type. If the food entry is updated to match the latest version of the food type, this field is set to false again.

The following columns until and including otherNutrition are a snapshot from the food type entry, if present, and are mutable(only the ones originally mutable in the food type, so basis entries are not). However, if any uniqueness breaking changes are made(calories, protein, carbs, fats, otherNutrition), the “edited” field will be set to true to indicate that the entries have been edited.
- basisUnit
- basisQty
- unitLabel
- unitConvTo
- unitConvAmt
- servingSize
- calories
- protein
- carbs
- fats
- otherNutrition
- totalAmt: total amount in basis unit(eg. 200 g, 2 units, 250 ml)


Indexes worth having
- FoodEntry(logDate)                       -- hottest filter; every day/week view
- FoodEntry(foodTypeId, versionNo)         -- retroactive update + "entries of this food"
- FoodEntry(deletedAt)                     -- or partial index WHERE deletedAt IS NULL
- FoodType(name)                           -- search/autocomplete
- FoodEntry(name)                          -- ad-hoc entries in autocomplete

