# ARPG Loot Lab

A browser-based loot prototype inspired by ARPG progression systems. The project is a single-file simulator built in [index.html](index.html), with a design brief and behavior notes captured here.

## Project status

The app is a focused, robust MVP slice that includes:

- random loot generation without reusing the same seed on every run
- separate quantity and rarity controls
- area-based and difficulty-based drop context
- item-level progression tied to area level
- equipment generation with affix rolls and visible metadata, guaranteed to include at least one
  prefix and one suffix (3 total affixes) on every Rare item
- a Path of Exile-style bagpack inventory (12×5 grid per tab, up to 24 scrollable tabs) with
  emoji-based slot icons, sorting (type → slot → rarity), and per-rarity bulk delete, plus run
  history, export, and inspection
- generated loot is listed directly beneath the inventory in its own scrollable panel, so a run
  with many drops doesn't grow the page
- a Wiki tab covering areas, base gear, modifiers, currency, rarity distribution, the monster loot
  table, and the item/affix/special-item design summaries, keeping the main Loot Simulator page
  focused on simulation config and inventory
- a hover compare tooltip that shows tiered modifiers and comparison details

The simulator intentionally starts idle. Nothing is generated until the user clicks the controls.

## Core architecture

### Stack

- plain HTML, CSS, and JavaScript
- no framework
- single-page browser app with tabbed views
- no backend or database
- browser localStorage for recent run history only

### Design goal

This project follows the same broad ARPG design philosophy as PoE without copying proprietary assets or code. It aims to provide a flexible, data-driven loot engine for original content.

## Gameplay systems in scope

### Loot categories

- equipment
- currency
- potion
- consumable
- miscellaneous
- special

### Rarity model

The app supports five item rarities, colored to match Path of Exile's convention:

- Normal — white
- Magic — blue
- Rare — yellow
- Unique — orange
- God — red

Rules:

- Normal items have no random explicit modifiers.
- Magic items can roll a limited number of prefix/suffix affixes.
- Rare items can roll more affixes with configurable limits, and are guaranteed at least one
  prefix and one suffix (3 total affixes) — an item with only one prefix and one suffix and
  nothing else is a Magic item, never a Rare one.
- Unique and God items use predefined templates rather than generic rare rolls.

### Area and monster progression

- area level is the primary progression anchor
- monster level is derived from the active area level unless a specific monster has an override
- difficulty and area modifiers influence quantity and rarity scaling
- item drops are leveled against the current selected context

### Affix library

Affixes are fully data-driven and live in [affixes.json](affixes.json). `index.html` loads them at
runtime with `fetch("affixes.json")` (see `loadAffixLibrary()`) instead of keeping an embedded
copy — this means **the JSON file is the single, unduplicated source of truth**, and editing it is
immediately reflected on next reload with nothing to keep in sync.

Because browsers block `fetch()` of local files opened via a raw `file://` URL (a CORS security
restriction), this app must be served over HTTP to load its data, for example:

```
python -m http.server 8080
# or
npx serve .
```

then open the printed `http://localhost:...` address (VS Code's "Live Server" extension also
works). If the fetch fails — e.g. because the page was opened directly as `file://` — a banner at
the top of the page explains why and shows these same commands; the Simulate/Generate Drop buttons
are disabled until the data loads successfully. `AFFIX_DEFS` in `index.html` is a thin
normalizer/loader over this data; it never hardcodes affix names or values.

The library follows real ARPG (Path of Exile-inspired) itemization conventions rather than invented flavor names:

- **28 stat groups**, each with **6 tiers**: life, mana, fire/cold/lightning/chaos resistance, strength/dexterity/intelligence, armour, evasion, energy shield, physical weapon damage, attack speed, cast speed, critical strike chance, critical strike multiplier, spell damage, stun threshold, flat/percent fire damage, flat/percent cold damage, flat/percent lightning damage, life regeneration, mana regeneration, and item rarity.
- **Correct tier direction**: Tier 1 is the strongest roll and requires the highest item level; the highest tier number is the weakest roll and only requires item level 1 — so a level-1 item always has at least one legal (weak) tier available for every eligible affix, and the strongest rolls stay locked behind higher item levels.
- **Mana is a prefix** (matching PoE convention), not a suffix — both the Life and Mana groups can also roll on off-hand items.
- Plain, functional naming (e.g. "Fire Resistance", "Critical Strike Multiplier") instead of invented prefixes like "Aegis" or "Swift".

### Currency library

Currency is treated as a first-class category, not generic junk loot.

| Currency | Role | Effect |
| --- | --- | --- |
| Gold Pile | low-tier economy | basic vendor and trade value |
| Silver Cache | mid-tier economy | small crafting and upgrade value |
| Boss Cache | endgame progression | stronger crafting and elite upgrade currency |
| Chaos Shard | high volatility | reroll and rebalancing currency |
| Divine Sigil | premium refinement | high-end affix refinement and optimization |

Consumables and specialty items are separated from true currency so the economy is easier to reason about and extend.

## Item generation rules

### Base items

Each base item includes:

- slot
- item type
- minimum level
- archetype
- tags
- base stats
- implicit modifiers
- requirements
- affix group allowances

`BASE_ITEMS` spans 40 entries across every slot (weapon subtypes: sword, axe, bow, wand, staff;
offhand; helm; chest; gloves; boots; ring; amulet; belt), each with low, mid, and high-level tiers
so equipment progression is visible from item level 1 up into the 80s.

### Affix system

Affixes are data-driven and include:

- prefix or suffix type
- slot compatibility
- supported groups and tags
- exclusive-group restrictions
- weighted tier ranges based on item level
- display text for final stats

### Modifier visibility

The UI exposes modifiers in two layers:

1. item cards show a short summary
2. the detail panel lists all modifiers in a readable inspection view

This keeps the simulator readable while still feeling close to a PoE-style item inspection flow.

When a run produces no loot, the simulator renders an explicit empty-state card instead of leaving the output area blank.

## Browser behavior

### Default state

- quantity default: 0
- rarity default: 0
- default monster: a normal mob, not a boss
- no simulation runs fire on page load
- the user must click the controls to generate loot or a single drop

### Simulation flow

- select a monster, area, and difficulty
- review the area level and current roll scale
- click Simulate or Generate Drop
- a single low-volume run may legitimately produce 0 items
- inspect the items, inventory, history, and export output

### Item inspection

- item cards show tiered modifier lines directly on the card
- hovering a loot or inventory item opens a compact Path of Exile-style tooltip: item name, base
  type/slot line, implicit modifiers, then explicit prefixes and suffixes, each tagged with its
  tier (`T1`–`T6`)
- holding **Alt** while hovering swaps each explicit modifier's rolled value for its tier's full
  `(min-max)` range, so you can see how strong a roll is relative to what that tier could produce
- rarity text and tooltip accents use Path of Exile's color convention: Normal is white, Magic is
  blue, Rare is yellow, Unique is orange, and God-tier is red
- the detail panel still provides the full inspection view for selected items

## Current implementation notes

The app in [index.html](index.html) already demonstrates:

- weighted rarity selection
- item scoring and grouping
- inventory retention
- run history persistence
- equipment and stat detail rendering
- a Wiki tab for area, affix, and currency reference data
- a dedicated currency library section
- a data-driven affix library ([affixes.json](affixes.json)) fetched at runtime, with correct tier/item-level scaling and no hardcoded per-slot special cases

## Roadmap

Near-term work continues from this MVP. See [Design specification & roadmap](#design-specification--roadmap) below for the full, itemized backlog with status markers; the short list here is just the current focus:

- deeper crafting and currency relationship rules
- more advanced filtering and trading flows
- richer gear archetypes and affix families
- endgame special item variants
- stronger simulation analytics and export data

## Notes for contributors

- keep the app browser-only unless a backend is required
- prefer data-driven configuration over hardcoded item logic
- preserve defensive coding and avoid stale DOM references
- keep the default load state idle to avoid unnecessary generation
- serve the folder over local HTTP (e.g. `python -m http.server` or `npx serve .`) when testing —
  opening `index.html` directly as `file://` blocks the `affixes.json` fetch
- do not reintroduce an embedded copy of `affixes.json` inside `index.html`; it must stay the
  single source of truth, loaded via `loadAffixLibrary()`

---

This project is intentionally built as a compact, original ARPG loot lab and not as a direct clone of any proprietary game.

## Design specification & roadmap

Everything below is the full original design brief for the target feature set. It intentionally
describes a much larger scope than the current MVP — it is the project's long-term backlog, not
a description of what exists today. Each section is tagged with a status so it's easy to tell
what's already working in [index.html](index.html) versus what's still planned.

**Status legend:** ✅ Implemented &nbsp;·&nbsp; 🟡 Partially implemented &nbsp;·&nbsp; ⬜ Planned / not started

### Example: base item → generated item

```
Base Item:      Iron Helmet

Generated Item: Rare Iron Helmet
Item Level:     84

+92 Maximum Life
+37% Fire Resistance
+31% Cold Resistance
+18 Armour
```

A `BaseItem` contains data such as: `id`, `name`, `slot`, `itemType`, `tags`, `baseStats`,
`implicitModifiers`, `requirements`, `minimumLevel`, `allowedAffixGroups`.

Depending on item type, base items support stats such as Armour, Evasion, Energy Shield,
Physical Damage, Elemental Damage, Attack Speed, Critical Chance, and defence combinations
(Armour+Evasion, Armour+Energy Shield, Evasion+Energy Shield).

---

### 12. Item instance — 🟡 partially implemented

A generated equipment item should be an independent instance:

```
ItemInstance
├── instanceId
├── baseId
├── itemLevel
├── rarity
├── quality
├── implicitModifiers
├── explicitModifiers
├── craftedModifiers
├── tags
└── metadata
```

Every generated equipment item should have a unique instance ID. Generated items today already
carry a unique id, rarity, item level, tags, and explicit modifiers; `quality` and a distinct
`craftedModifiers` list (separate from rolled explicit modifiers) don't exist yet because there is
no crafting system (see [§20](#20-crafting-system--not-implemented)).

### 13. Affix system — ✅ implemented

Every affix is data-driven ([affixes.json](affixes.json)) with: `id`, `name`, `kind` (prefix/suffix),
`group`, `exclusiveGroup`, `slots`, `tags`, `statKey`, `display`, and a `tiers` array
(`tier`, `minLevel`, `minValue`, `maxValue`, `weight`). Affixes are not hardcoded — adding a new
one only requires a new entry in the JSON file.

### 14. Prefixes and suffixes — ✅ implemented

Every explicit modifier identifies as `prefix` or `suffix` (`kind`). Affixes are filtered by item
slot, item level, affix `kind`, already-used affix ids, tags, exclusive groups (`exclusiveGroup`),
and base-item `allowedAffixGroups`. Maximum prefix/suffix counts per rarity are enforced via
`CONFIG.rarityLimits` (Magic: 1 prefix/1 suffix, Rare: up to 3/3).

### 15. Affix weighting — ✅ implemented

Affix and tier selection both use `weightedChoice`/`weightedKeyChoice` over the remaining
candidates' weights after filtering — never a uniform random array index.

### 16. Affix tiers — ✅ implemented (fixed)

Each affix has 6 tiers, each with `minLevel`, `weight`, `minValue`, `maxValue`. **Tier 1 requires
the highest item level and rolls the strongest values; the highest tier number only requires item
level 1 and rolls the weakest values** — so low item-level gear always has a legal (weak) option,
and the strongest rolls stay gated behind higher item levels, matching "higher item levels unlock
stronger tiers."

### 17. Modifier library — 🟡 partially implemented

Currently implemented in [affixes.json](affixes.json):

- **Defence:** Armour, Evasion Rating, maximum Energy Shield
- **Resistances:** Fire, Cold, Lightning, Chaos
- **Attributes:** Strength, Dexterity, Intelligence
- **Resources:** maximum Life, maximum Mana, Life Regeneration, Mana Regeneration
- **Offensive:** increased Physical Damage, Attack Speed, Cast Speed, Critical Strike Chance,
  Critical Strike Multiplier, increased Spell Damage, flat/percent Fire Damage, flat/percent Cold
  Damage, flat/percent Lightning Damage
- **Defensive utility:** increased Stun Threshold
- **Economy:** increased Rarity of Items found

Not yet implemented: energy shield regeneration, movement speed as an explicit affix,
damage-over-time, item quantity modifiers, cooldown recovery, mana cost, and attribute requirement
modifiers. The engine is generic enough (`statKey` + `display` + `tiers`) that adding these is a
data change, not a code change.

### 18. Affix conflict system — 🟡 partially implemented

`exclusiveGroup` prevents rolling two affixes from the same group (e.g. two different Fire
Resistance rolls). The selector also supports optional `requiresTags`/`forbiddenTags` fields per
affix, but no current affix definitions populate them yet — `conflictGroup`/`modGroup`-style cross-
affix conflict rules beyond `exclusiveGroup` are still planned.

### 19. Currency system — 🟡 partially implemented

Currency is a first-class loot category (`STACKABLE_ITEMS`, currency codex table, currency drops
in the loot simulator) with `id`, `name`, `description`, `stackSize`, and `rarity`. Currencies do
not yet carry a `craftingEffect`/`targetRestrictions` payload because there is no crafting engine
to consume them (see [§20](#20-crafting-system--not-implemented)).

### 20. Crafting system — ⬜ not implemented

Planned: a `CraftingOperation` that accepts `(item, currency, craftingContext, randomGenerator)`
and returns a `CraftingResult` (`success`, `modifiedItem`, `changes`, `consumedCurrency`,
`messages`), designed so new currencies don't require changes to the `Item` model.

### 21. Inventory system — 🟡 partially implemented

The bagpack is a Path of Exile-style grid: **12 columns × 5 rows (60 slots) per tab**, with up to
**24 tabs** (1,440 slots total) selectable via a scrollable tab bar showing each tab's item count.
The inventory cap input (60–1,440, in 60-slot increments) trims the oldest items once the
configured cap is exceeded. Items render with emoji-based slot icons (weapon, offhand, helm,
chest, gloves, boots, ring, amulet, belt, plus category icons for currency/potion/consumable/
special/miscellaneous). A toolbar above the grid offers a sort control (Recently Added, or
Type → Slot → Rarity) and per-rarity bulk delete buttons (Normal/Magic/Rare/Unique/God), which use
a click-to-arm, click-again-to-confirm pattern to avoid accidental data loss.

Implemented: add/remove items, currency stacking, item inspection, and item comparison (hover
tooltip), multi-tab storage and navigation, sorting, and bulk delete. Not yet implemented:
move/split/merge stacks between slots or tabs, and equip/unequip (there are no equipment slots —
items only ever live in the bagpack grid).

### 22. Item inspection UI — 🟡 partially implemented

The hover tooltip uses a simplified Path of Exile layout: item name (rarity-colored), base
type/slot line, implicit modifiers, then explicit prefixes/suffixes each tagged `T1`–`T6`; holding
**Alt** reveals each explicit modifier's tier `(min-max)` range instead of its rolled value. The
detail panel shows the same data plus requirements. `Crafted`/`Special` modifier categories are
not shown separately yet since crafting doesn't exist.

### 23. Loot simulator — 🟡 partially implemented

Implemented: monster, area, difficulty, quantity %, rarity %, and seed controls, plus a kill-count
field. Not yet implemented: one-click presets (100 / 1,000 / 10,000 / 100,000 / 1,000,000 /
10,000,000) — the field currently accepts any custom number but has no preset buttons.

### 24. Large-scale simulation — ⬜ not implemented

The current engine generates and keeps a full item object per roll for a single run. Aggregating
millions of kills into running totals (`currencyResults["gold_pile"] += amount`) without
materializing per-kill UI objects or writing per-drop records is planned but not yet built.

### 25. See generated gear — 🟡 partially implemented

The loot list and detail panel let you inspect every item from the current run, but there's no
rarity/slot/base/affix filtering yet, and no sampling strategy for runs large enough that
retaining every item wouldn't be practical.

### 26. Simulation results — 🟡 partially implemented

Metric cards (drops, rolls, etc.) and a rarity breakdown table are implemented. Deeper statistics
(observed vs. expected variance, "1 in X" framing for rare drops) are not yet shown.

### 27. See theoretical vs. observed results — ⬜ not implemented

Planned: show the configured theoretical drop rate next to the simulation's observed rate (e.g.
"Configured: 0.001% · Observed over 1,000,000 kills: 0.0007%") without implying a single run
should exactly match the theoretical probability.

### 28. See actual drop details — ✅ implemented

Clicking a loot or inventory item opens a detail panel/tooltip showing rarity, item level,
modifiers, and prefix/suffix/tier breakdown — not just aggregate statistics.

### 29. Crafting / inventory page — ⬜ not implemented

Currently one simulator view plus an inventory grid, not a dedicated second "Inventory & Crafting"
page with crafting preview/apply/before-after comparison.

### 30. Item history — 🟡 partially implemented

Run history persists to `localStorage`. A step-by-step per-item history log (created → currency
applied → modifier changed → …) doesn't exist yet, since there is no crafting system to generate
those steps.

### 31. See the loot generation process — ⬜ not implemented

Planned: an opt-in developer/debug mode that explains a single generation in detail (base rolls,
quantity bonus, effective rolls, eligible affix count, selected prefix/suffix + tier) — disabled
during large simulations to avoid excessive logging.

### 32. See and edit configuration — 🟡 partially implemented

The Design Data / Codex views expose `CONFIG`, base items, the affix library, and the currency
library as read-only tables. In-app editing of this data isn't implemented; edits currently
require changing `affixes.json` or the data blocks in `index.html` directly.

### 33. See development data — ⬜ not implemented

Planned developer tools: generate a specific rarity/base/affix on demand, inspect an affix pool or
loot table directly, or set the RNG seed from a dev panel rather than only via "Reroll Seed".

### 34. See deterministic simulations — ✅ implemented

`createRng`/`hashSeed` provide a seeded PRNG. The same seed, monster, area, difficulty, quantity,
and rarity always reproduce the same run — the app never calls a global unseeded random function
for gameplay-affecting rolls.

### 35. Performance — 🟡 partially implemented

Fine for interactive, single-run use. Not yet validated or optimized for 1,000,000+ kill batch
simulations (no aggregation-only mode, no background worker, no periodic progress updates).

### 36. Simulation progress — ⬜ not implemented

Runs are synchronous and immediate; there is no progress indicator for long-running simulations
because large-scale simulation ([§24](#24-large-scale-simulation--not-implemented)) doesn't exist yet.

### 37. Export results — 🟡 partially implemented

JSON export is implemented (`exportResults`, downloaded via `Blob`/`URL.createObjectURL`) and
includes the run configuration, seed, and results. CSV export is not yet implemented.

### 38. Testing — ⬜ not implemented

No automated test suite exists yet (loot generation, item rarity generation, affix selection,
currency, inventory, or simulation reproducibility tests).

### 39. Statistical testing — ⬜ not implemented

Planned: verify observed rates fall within an acceptable statistical tolerance of configured rates
over large sample sizes, rather than asserting exact counts.

### 40. Architecture — 🟡 partially implemented

The app is intentionally a single HTML file (see [Core architecture](#core-architecture)), so the
conceptual `LootSystem` / `ItemSystem` / `AffixSystem` / `CraftingSystem` / `InventorySystem` /
`SimulationSystem` split exists as clearly-commented sections within one script rather than
separate modules/folders.

### 41. Strict separation of UI and gameplay logic — 🟡 partially implemented

Generation/engine functions (`generateRun`, `normalizeContext`, `selectEligibleAffixes`,
`pickAffixTier`, …) are separate from rendering functions (`renderLootList`, `renderDetail`, …) and
are safely callable without touching the DOM. They still live in the same script scope rather than
separate files, which is an intentional tradeoff for a dependency-free, single-file browser app.

### 42. No hard-coded special cases — ✅ implemented

Affix eligibility is entirely data-driven (slot + tags + group + item level + exclusivity). There
is no `if item.slot === "helmet": addLife()`-style branching anywhere in the generator.

### 43. Extensibility — 🟡 partially implemented

Adding a new affix is a pure data change in [affixes.json](affixes.json) — no code changes needed.
Adding a new base item, currency, unique, god item, or monster still requires editing the relevant
array in `index.html` directly rather than a separate config file/UI.

### 44. Required initial UI — 🟡 partially implemented

**Page 1 — Loot Simulator:** simulation config (monster/area/difficulty/quantity/rarity/seed, run
simulation) and the inventory are the focus of this page — the rarity distribution table, monster
loot table, and item/affix/special-item design summaries live in the Wiki tab instead so they
don't push the inventory down the page. Generated loot renders directly under the inventory in
its own scrollable list. Missing: progress indicator for large runs, result filters, CSV export.

**Page 2 — Wiki:** areas, base gear catalog, modifier library, currency library, rarity
distribution, monster loot table, and item/affix/special-item design summaries are all
implemented as reference tables. Crafting (preview, apply, before/after, history) is not.

### 45. Implementation order

Suggested phase order for the remaining work (adapt to the existing single-file architecture
rather than creating new folders that don't fit it):

1. **Domain models** — Item, BaseItem, Equipment, Currency, Rarity, Affix, Modifier, Loot *(mostly ✅)*
2. **Loot engine** — loot tables, quantity, rarity, drop categories, RNG, equipment generation *(✅)*
3. **Affix engine** — prefix/suffix, pools, weights, tiers, restrictions, validation *(✅)*
4. **Special items** — Unique, God *(✅)*
5. **Currency / crafting** — generalized currency + crafting framework *(🟡 currency only)*
6. **Inventory** — inventory, stacking, equipment slots, item inspection *(🟡)*
7. **Simulation** — seeded simulation, aggregation, statistics, large-scale performance *(🟡)*
8. **UI** — Loot Simulator, Inventory, Item Viewer, Crafting *(🟡 crafting UI missing)*
9. **Testing** — unit, integration, statistical, performance tests *(⬜)*
10. **Polish** — performance, UX, validation, error handling, loading/empty states *(🟡 empty states ✅, rest ongoing)*

### 46. Definition of done

This is the actual project to-do list — check items off as they land:

- [x] Monsters can generate loot.
- [x] Loot has multiple categories.
- [x] Quantity affects loot volume.
- [x] Rarity affects item quality distribution.
- [x] Equipment can be Normal, Magic, Rare, Unique, or God.
- [x] Normal items have no explicit random modifiers.
- [x] Magic items support prefix/suffix generation.
- [x] Rare items support multiple prefixes/suffixes.
- [x] Unique items use predefined definitions.
- [x] God items use predefined definitions.
- [x] Affixes use weighted selection.
- [x] Affixes respect item level (with corrected tier direction).
- [x] Affixes respect item slots.
- [x] Affix conflicts are enforced (exclusive groups; full conflict-group system still planned).
- [ ] Currency exists as a generalized loot **and crafting** system (loot side is done; crafting is not).
- [x] Inventory works (add/remove/stack/inspect/compare; no equip slots or split/merge yet).
- [x] Items can be inspected.
- [ ] Items can be crafted.
- [ ] Loot can be simulated at 1,000,000+ kills without freezing the UI.
- [ ] Simulation results are aggregated efficiently at that scale.
- [x] Generated equipment can be inspected.
- [x] Simulations are reproducible with a seed.
- [x] Statistics are displayed (basic metrics; theoretical-vs-observed view still planned).
- [ ] Results can be exported as both JSON and CSV (JSON only today).
- [ ] Automated tests exist.
- [ ] Large simulations do not freeze the UI.
- [x] Gameplay data is separated from UI code (data-driven config + affix library).
- [x] The system is configurable (`CONFIG`, `BASE_ITEMS`, `affixes.json`, currency/unique/god defs).
- [x] New affixes can be added without touching the core engine (edit `affixes.json` only).
- [ ] New currencies, monsters, and loot tables can be added without touching the core engine (still requires editing arrays in `index.html`).

### 47. Final instruction to the coding agent

Do not stop at designing the architecture — implement changes directly in this repository.

1. Inspect the existing code and follow its established patterns (single-file, data-driven,
   seeded RNG, defensive coding) rather than introducing a new framework or folder structure.
2. Implement incrementally: domain/data changes first, then the engine, then the UI.
3. After any change, verify it: run a syntax check, validate any JSON data files, and — where
   practical — exercise the affected code path (e.g. simulate a click) rather than only reading
   the code.
4. Fix all errors found during verification before considering the task complete.
5. Don't break existing functionality (bagpack inventory, tooltips, codex views, currency
   library, collapsible panels) while implementing new features.
6. Summarize what changed, what was verified, and what remains for the next iteration.
