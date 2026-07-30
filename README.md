# Saving Throw — Addon Authoring Guide

This repository (`raabelo/st-addons`) hosts the public addons for
[Saving Throw](https://savingthrow.vercel.app/), a modular virtual
tabletop platform. **Saving Throw has zero hardcoded RPG logic** — every
character sheet, dice system, action, and piece of narrative content comes
from an addon manifest. This document is the canonical reference for
building one.

You do **not** need access to the Saving Throw application source to write
an addon. Everything an addon can declare, and every rule the platform
enforces on it, is described below, sourced directly from the platform's
engine package that validate every manifest before it's accepted.

> This is a living document. As the addon engine gains new manifest fields
> or field types, this README should be updated in the same spirit — one
> section per concern, real schema references, no guessing.

---

## Table of Contents

1. [What an addon is](#1-what-an-addon-is)
2. [How addons are loaded and validated](#2-how-addons-are-loaded-and-validated)
3. [Repository layout](#3-repository-layout)
4. [Full manifest.json spec](#4-full-manifestjson-spec)
5. [Character (and adversary) schema authoring](#5-character-and-adversary-schema-authoring)
6. [Items](#6-items)
7. [Presets](#7-presets)
8. [Dice systems](#8-dice-systems)
9. [Dice skins](#9-dice-skins)
10. [Quick rolls](#10-quick-rolls)
11. [Prompt templates (PROMPT addons)](#11-prompt-templates-prompt-addons)
12. [Integrations (INTEGRATION addons)](#12-integrations-integration-addons)
13. [Dependencies](#13-dependencies)
14. [Capabilities](#14-capabilities)
15. [Licensing](#15-licensing)
16. [Locales](#16-locales)
17. [Addon composition (multiple active addons)](#17-addon-composition-multiple-active-addons)
18. [Migrations (versioning your manifest)](#18-migrations-versioning-your-manifest)
19. [Validation checklist](#19-validation-checklist)
20. [Security & isolation model](#20-security--isolation-model)
21. [Minimal end-to-end example](#21-minimal-end-to-end-example)
22. [Publishing checklist](#22-publishing-checklist)

---

## 1. What an addon is

An addon is a **folder in this repository** (or any public repo you control)
containing a `manifest.json` plus optional supporting assets (locale files,
textures, sounds, etc.). The manifest is the entire contract between your
content and the platform — there is no plugin code, no JavaScript, no
server your addon runs. Everything is declarative JSON, validated by Zod
schemas on the platform side and interpreted by generic engines
(character-schema renderer, dice roller, board renderer, etc.).

An addon can define, in any combination:

- A **character sheet schema** (fields + sections) — how PC sheets render
- An **adversary/statblock schema** — how GM-side NPC/monster sheets render
- **Item types** — structured inventory items (weapons, gear, consumables)
- **Presets** — named option sets (classes, races, backgrounds…) that
  auto-fill other fields and drive conditional visibility
- **Dice definitions** — named notations (e.g. `1d20`, `2d12`)
- **Dice skins** — cosmetic dice textures/materials/sounds
- **Quick rolls** — one-click roll shortcuts
- **Prompt templates** — safe, allowlisted text fragments for AI narration
- **Integrations** — outbound webhooks to your own HTTPS server
- **Dependencies** — other addons yours builds on top of

Addon types (`addon.types`, see [§4](#4-full-manifestjson-spec)) categorize
what your addon is *for*; a single addon can declare multiple types (e.g.
`["CORE", "RULES"]` for a full RPG system like `generic-core`).

---

## 2. How addons are loaded and validated

```
Addon Discovery                (platform fetches manifest.json from your repo)
    ↓
Manifest Validation            (Zod: AddonManifestSchema.safeParse)
    ↓
Dependency Resolution          (semver constraint check + DFS cycle detection)
    ↓
Schema Composition             (merge with other active addons — see §17)
    ↓
Runtime Activation             (character/dice/board engines interpret the
                                 composed schema; frontend renders generically)
```

- **Manifest validation** (`validateManifest` / `validateManifestFromString`
  in `@saving-throw/addon-engine`) runs your raw `manifest.json` through a
  single Zod schema, `AddonManifestSchema`. Any unknown top-level key,
  wrong type, missing required field, or violated cross-field rule
  (§4/§19) causes the **entire manifest to be rejected** — there is no
  partial acceptance.
- **Dependency resolution** (`resolveDependencies` in `resolver.ts`) checks
  that every non-optional entry in `dependencies` is installed and satisfies
  its semver constraint, and runs a DFS cycle check across the dependency
  graph. Circular dependencies are always rejected.
- **Composition** (`composeAddonSchemas` in `compose.ts`) merges the
  "system" addon (the primary RULES/CORE addon a campaign is built on) with
  any number of extra active addons — see [§17](#17-addon-composition-multiple-active-addons)
  for the exact merge rules per field.
- Only the **validated, typed output of the Zod parse** is ever used
  downstream — never your raw JSON. This means prototype-pollution-style
  payloads (`__proto__`, unknown keys, etc.) are structurally impossible to
  smuggle through: `.strict()` is used on every object schema in the
  manifest, so unrecognized keys fail validation outright.

---

## 3. Repository layout

Each addon lives in its own top-level folder, named after its slug segment
(the part after `author/` for community addons, or the flat name for
platform-verified ones):

```
st-addons/
  generic-core/
    manifest.json
    locales/
      en.json
      pt.json
      es.json
      fr.json
  dice-skin-gilded-bronze/
    manifest.json
    assets/
      textures/
        bronze01.webp
      sounds/
        dicehit/
          dicehit_metal1.mp3
          ...
  your-namespace/your-addon/
    manifest.json
    locales/
      en.json
```

Conventions used across existing addons in this repo:

- `manifest.json` — required, at the addon's root
- `locales/<locale>.json` — optional per-locale translation bundle (see
  [§16](#16-locales))
- `assets/` — optional, for any addon-hosted media referenced by URL from
  the manifest (currently used by dice skins via `assetBaseUrl`; only
  `https://` URLs pointing at your own hosted files are ever accepted —
  see [§9](#9-dice-skins) and [§20](#20-security--isolation-model))

The platform fetches your addon's files directly via GitHub's raw content
API, at:

```
https://raw.githubusercontent.com/<owner>/<repo>/<branch>/<addon-folder>/manifest.json
https://raw.githubusercontent.com/<owner>/<repo>/<branch>/<addon-folder>/locales/<locale>.json
```

For addons published in this repo specifically, that resolves to
`https://raw.githubusercontent.com/raabelo/st-addons/main/<addon-folder>/...`.
If you publish in your own repo, use that repo's URL as `repositoryUrl` in
your manifest — see [§4](#4-full-manifestjson-spec).

Because this fetch path is unauthenticated raw GitHub access, **your
repository must be public**.

---

## 4. Full manifest.json spec

Every manifest is a single JSON object with one top-level key, `addon`:

```json
{ "addon": { /* ... */ } }
```

### 4.1 Required fields

| Field | Type | Constraints | Purpose |
|---|---|---|---|
| `slug` | `string` | 3–80 chars, kebab-case, optionally namespaced `author/addon-name`; regex `^[a-z0-9][a-z0-9-]*(?:\/[a-z0-9][a-z0-9-]*)?$` | Unique addon identifier. **Community addons must use the namespaced form** (`author/addon-name`). Flat slugs (`dnd5e-core`) are reserved for platform-verified addons — an unauthorized flat slug is an impersonation vector and will be rejected/blocked at the platform trust layer even though it may pass schema validation. |
| `name` | `string` | 3–64 chars, no HTML | Human-readable display name. |
| `version` | `string` | strict semver `major.minor.patch` (no pre-release tags) | Addon version. **Published versions are immutable** — you cannot republish the same `slug@version` with different content; bump the version instead. |
| `description` | `string` | 10–500 chars, no HTML | Shown to GMs before activation. |
| `repositoryUrl` | `string` | HTTPS URL only | Where the addon's source lives. Must point at the actual repo hosting this manifest. |
| `author` | `string` | 2–64 chars, no HTML | Author/publisher display name. |
| `types` | `array<ADDON_TYPE>` | 1–5 entries | See [§4.3](#43-addon-types) below. |
| `tags` | `array<string>` | 0–10 entries, each 1–32 chars, no HTML | Freeform discovery tags (e.g. `"fantasy"`, `"homebrew"`). |

### 4.2 Optional fields

| Field | Type | Limit | Purpose |
|---|---|---|---|
| `license` | object | — | Structured license metadata — see [§15](#15-licensing). |
| `thumbnailUrl` | HTTPS URL | — | Square thumbnail. |
| `thumbnailWideUrl` | HTTPS URL | — | Wide/banner thumbnail. |
| `capabilities` | `array<ADDON_CAPABILITY>` | max 12 | Explicit permission declarations — see [§14](#14-capabilities). Defaults to `[]`. |
| `minPlatformVersion` | semver string | — | Minimum platform version required. |
| `characterFields` | `array<CharacterField>` | max 200 | PC sheet field definitions — see [§5](#5-character-and-adversary-schema-authoring). |
| `characterSections` | `array<CharacterSection>` | max 20 | PC sheet layout grouping — see [§5](#5-character-and-adversary-schema-authoring). |
| `adversaryFields` | `array<AdversaryField>` | max 200 | GM-side NPC/monster statblock fields — same shape as `characterFields`. |
| `adversarySections` | `array<AdversarySection>` | max 20 | NPC statblock layout grouping. |
| `itemTypes` | `array<ItemType>` | max 50 | Inventory item type definitions — see [§6](#6-items). |
| `presets` | `Record<category, PresetEntry[]>` | max 60 entries per category | Named option sets (classes, ancestries, etc.) — see [§7](#7-presets). |
| `promptTemplates` | `Record<key, string>` | key regex `^[a-z_][a-z0-9_]*$`, max 64 chars | AI prompt fragments — requires `PROMPT` type + `ai:prompt` capability — see [§11](#11-prompt-templates-prompt-addons). |
| `diceDefinitions` | `array<DiceDefinition>` | max 20 | Named dice notations — requires `dice:custom` capability — see [§8](#8-dice-systems). |
| `diceSkins` | `array<DiceSkin>` | max 20 | Cosmetic dice appearance — requires `dice:skin` capability — see [§9](#9-dice-skins). |
| `quickRolls` | `array<QuickRoll>` | max 20 | One-click roll shortcuts — see [§10](#10-quick-rolls). |
| `integrations` | `array<Integration>` | max 5 | Outbound webhooks — requires `INTEGRATION` type + `integration:external` capability — see [§12](#12-integrations-integration-addons). |
| `dependencies` | `array<Dependency>` | max 20 | Other addons yours depends on — see [§13](#13-dependencies). |
| `enableSkillUse` | `boolean` | — | Enables the skill-use selector UI and realtime `skill:use` events for this campaign's composed schema. |

All object schemas in the manifest use Zod's `.strict()` mode: **any key
not in this spec causes the whole manifest to fail validation.** There is
no "extra data" escape hatch — if you need custom data, model it as a
`characterField`/`itemType` field, not a free-form blob.

### 4.3 Addon types

`types` is an array of 1–5 values from:

| Type | Purpose |
|---|---|
| `CORE` | Foundational dice/action systems (dice definitions, default rolls, initiative). |
| `RULES` | Character schemas, skills, attributes, inventory, progression. |
| `PROMPT` | AI narrative assistant behavior. Requires the `ai:prompt` capability. |
| `NARRATIVE` | Lore, factions, worldbuilding, encounter content. |
| `INTEGRATION` | External connectors (Discord, APIs). Requires the `integration:external` capability. |
| `UTILITY` | Optional helpers, calculators, cosmetic content (e.g. dice skins). |

A single addon commonly declares more than one type — e.g. a full RPG
system addon is typically `["CORE", "RULES"]`.

### 4.4 Cross-field rules enforced at manifest-parse time

These are `.refine()`/`.superRefine()` checks on top of the per-field
shapes above. A manifest that violates any of these is rejected outright,
even if every individual field is otherwise well-typed:

- `types` includes `PROMPT` ⇒ `capabilities` must include `ai:prompt`
- `types` includes `INTEGRATION` ⇒ `capabilities` must include `integration:external`
- `promptTemplates` non-empty ⇒ `capabilities` must include `ai:prompt`
- `integrations` non-empty ⇒ `capabilities` must include `integration:external`
- `diceDefinitions` non-empty ⇒ `capabilities` must include `dice:custom`
- `diceSkins` non-empty ⇒ `capabilities` must include `dice:skin`
- `diceSkins[].key` must be unique within the addon
- `characterFields[].labelFrom` (if set) must reference an existing `characterFields[].key`
- any `characterFields[]` field with `flags.quickRollDice: true` must also declare `labelFrom`
- same two `labelFrom`/`quickRollDice` rules apply to `adversaryFields[]`
- `itemTypes[].key` must be unique within the addon
- `characterFields[].linkedItemType` (if set) must reference an existing `itemTypes[].key`, or be the literal `"default"`
- `presets[category][].key` must be unique within its category
- `characterFields[].flags.allowCustom` may only be `true` when `type` is `"select"`
- if both `presets.class` and `presets.domain` exist, every `presets.class[].fieldValues.domains` string value must reference a `presets.domain[].key` in the same addon (cross-addon references are only validated at composition time, not per-addon)
- `characterFields[].visibleWhen.field` (if set) must reference an existing `characterFields[].key`
- if `license.type === "dpcgl"`: `license.monetized` must be `false`, `license.attributionText` must include the exact DPCGL §4.1 credit-line fragment, `license.sourceUrl` is required, and `license.compatibilityStatement` must contain the word "compatible" — see [§15](#15-licensing)

---

## 5. Character (and adversary) schema authoring

Character sheets are **never hardcoded** by the platform. Every field a
player sees on their sheet comes from your `characterFields` +
`characterSections` arrays. `adversaryFields`/`adversarySections` follow an
identical shape for GM-facing NPC/monster statblocks (kept as a separate
schema from character fields intentionally, since statblocks are expected
to diverge over time — thresholds, tiers, action economy).

### 5.1 CharacterField shape

```ts
{
  key: string;            // snake_case, ^[a-z_][a-z0-9_]*$, max 64 chars — stable, never localized, never renamed without a migration (§18)
  label: string;           // 1-64 chars, no HTML — display label (localizable via locales/*.json)
  type: FieldType;          // see table below
  section?: string;         // max 64 chars — which CharacterSection this field belongs to
  required?: boolean;
  options?: string[];        // max 100 entries, each 1-64 chars — for select/multiselect
  min?: number;
  max?: number;
  capacity?: number;         // 1-20 — box count for checkbox_track
  capacityFrom?: string;     // max 64 chars — key of another field supplying capacity instead
  iconType?: string;         // max 32 chars — track icon variant for shield_track
  flags?: {
    usableAsModifier?: boolean;  // field value may be added as a roll modifier
    allowSkillUse?: boolean;     // item entries can be invoked as skill checks
    realtimeDisplay?: boolean;   // sync state visually to all players live
    quickRollDice?: boolean;     // dice field appears in quickroll menus — requires labelFrom
    allowCustom?: boolean;       // select renders as combobox, free text allowed — only valid on type: "select"
  };
  default?: string | number | boolean;  // string default max 256 chars
  hidden?: boolean;           // field exists for data/quickroll but renders in no section
  labelFrom?: string;         // key whose runtime value supplies the quickroll label
  weaponSlots?: { primary?: WeaponSlotConfig; secondary?: WeaponSlotConfig }; // only for type: "weapon_loadout"
  armorSlots?: ArmorSlotConfig;  // only for type: "armor_piece"
  linkedItemType?: string;    // itemTypes[].key this field's "@" autocomplete searches, or "default"
  visibleWhen?: { field: string; presetKey: string };  // only render once `field` resolves to `presetKey`
}
```

### 5.2 Field types (`FieldType`)

| Type | Description |
|---|---|
| `text` | Single-line text. |
| `number` | Numeric input, respects `min`/`max`. |
| `boolean` | Checkbox/toggle. |
| `select` | Dropdown from `options` (or free text if `flags.allowCustom`). |
| `multiselect` | Multiple choices from `options`. |
| `textarea` | Short multi-line text. |
| `longtext` | Long narrative text (backstory, ability descriptions) — collapsible/eye-button preview in the UI. |
| `richtext` | WYSIWYG HTML text (Tiptap editor) — sanitized on render. |
| `resource` | A current/max pair — HP, MP, stress, etc. |
| `list` | Freeform list of entries. |
| `table` | Tabular data. |
| `checkbox_track` | N fillable boxes; box count from `capacity`. |
| `dynamic_list` | Ordered list of text items; supports `usableAsModifier`. |
| `weapon_slot` | Structured object: name, damage, range, traits. |
| `image_gallery` | Array of asset IDs/URLs. |
| `ability_card_list` | Array of `{title, description}` cards. |
| `shield_track` | N shield icons; capacity from `capacityFrom`; realtime-sync aware. |
| `weapon_loadout` | Primary + optional secondary weapon with hand-slot visualization; driven by `weaponSlots`. |
| `armor_piece` | Worn armor: name, damage thresholds, feature/ability; driven by `armorSlots`. |
| `experience_list` | `[{name, modifier, usableAsModifier}]`; max count from a `max` field; all entries are modifier sources. |
| `domain_card_list` | Array of `{title, description, image, allowSkillUse}` — domain/skill cards, feeds the skill-use system. |
| `image_dossier` | Array of `{title, image}` — freeform reference images with optional title. |
| `dice` | Dice notation field — selectable from your `diceDefinitions` or free text; supports `flags.quickRollDice`. |
| `feature_list` | Array of `{name, type, description}` where `type` is one of `passive`, `action`, `reaction`, `other`. |
| `attack_profile` | Structured object `{range, damageDice, damageType}` — a single named attack's profile. |

### 5.3 CharacterSection shape

```ts
{
  id: string;                                  // snake_case field-key format
  title: string;                                // 1-64 chars, no HTML
  layout: "grid" | "list" | "cards" | "table";
  fields: string[];                              // up to 50 CharacterField keys, in display order
}
```

### 5.4 Concrete example (from `generic-core/manifest.json`, real addon in this repo)

```json
{
  "addon": {
    "slug": "generic-core",
    "name": "Generic Core",
    "version": "1.0.0",
    "description": "A system-agnostic starter ruleset...",
    "repositoryUrl": "https://github.com/raabelo/st-addons",
    "author": "atelier.dev",
    "license": { "type": "original" },
    "types": ["CORE", "RULES"],
    "tags": ["generic", "template", "original"],
    "capabilities": ["character:read", "character:write", "dice:custom"],
    "characterFields": [
      { "key": "character_name", "label": "Character Name", "type": "text", "section": "identity", "required": true },
      { "key": "level", "label": "Level", "type": "number", "section": "identity", "min": 1, "max": 20, "default": 1 },
      { "key": "power", "label": "Power", "type": "number", "section": "attributes", "min": 1, "max": 20, "default": 10 },
      { "key": "hit_points", "label": "Hit Points", "type": "resource", "section": "combat" },
      { "key": "notes", "label": "Notes", "type": "textarea", "section": "background" }
    ],
    "characterSections": [
      { "id": "identity", "title": "Identity", "layout": "grid", "fields": ["character_name", "level"] },
      { "id": "attributes", "title": "Attributes", "layout": "grid", "fields": ["power"] },
      { "id": "combat", "title": "Combat", "layout": "grid", "fields": ["hit_points"] },
      { "id": "background", "title": "Background", "layout": "list", "fields": ["notes"] }
    ],
    "diceDefinitions": [
      { "key": "basic_check", "label": "Basic Check", "notation": "1d20", "description": "Standard check against a difficulty" }
    ]
  }
}
```

See the real, full file at [`generic-core/manifest.json`](./generic-core/manifest.json)
in this repo for a complete working addon — it's the recommended starting
template for a new system addon.

---

## 6. Items

`itemTypes` defines structured inventory item types (weapons, gear,
consumables, etc.) reusing the same field vocabulary as character fields
(minus `weaponSlots`/`armorSlots`, which only make sense on the character
side).

```ts
{
  key: string;              // snake_case
  label: string;             // 1-64 chars
  fields: ItemField[];        // max 50 — same shape as CharacterField, without weaponSlots/armorSlots
  linkableFrom?: FieldType[];  // max 10 — which character field types may "@"-link to this item type,
                                 // e.g. ["weapon_loadout"]
}
```

Every addon automatically receives a `"default"` item type
(`{ key: "default", label: "Default", fields: [{ key: "description", label: "Description", type: "richtext" }] }`)
if it doesn't declare its own override for that key — so items can always
be created even by addons with no `itemTypes` of their own.

`characterFields[].linkedItemType` wires a character field's "@"
autocomplete to search a given item type (or `"default"`).

---

## 7. Presets

Presets are named, localizable option sets that back a `select`-style
character field — e.g. `character_class`, `subclass`, `ancestry`. A preset
entry:

1. Supplies the field's selectable options via its per-locale `aliases`
2. Optionally auto-fills other, currently-empty character fields when
   chosen (`fieldValues`)
3. Optionally drives conditional field visibility, via
   `characterFields[].visibleWhen`

```ts
presets: {
  [categoryKey: string]: PresetEntry[]  // categoryKey: lower_snake_case, e.g. "class", "domain"
}

PresetEntry = {
  key: string;                 // stable id, snake_case — never localized, never renamed
  aliases: Record<localeCode, string>;  // must include at least "en"; locale codes like "en" or "en-US"
  fieldValues?: Record<fieldKey, PresetFieldValue>;
  requiresClass?: string;       // child-side dependency, e.g. a subclass preset referencing its parent class preset key
}

PresetFieldValue =
  | Array<{ title: string; description: string }>   // max 20 — card-shaped values, e.g. class_abilities
  | Array<string>                                     // max 50 — e.g. domains: ["blade", "valor"]
  | string | number | boolean                          // scalar direct write
  | Record<string, string | number | boolean>           // shallow-merge into an existing object field,
                                                           // e.g. hit_points: { max: 7 } merges into { current, max }
```

Presets are **concatenated, not overridden**, when multiple addons are
active in the same campaign — see [§17](#17-addon-composition-multiple-active-addons).
Preset keys must be unique per category within your own addon; a
cross-addon key collision in the same category is rejected at composition
time.

---

## 8. Dice systems

`diceDefinitions` declares named, reusable dice notations (requires the
`dice:custom` capability):

```ts
{
  key: string;               // snake_case
  label: string;              // 1-64 chars
  notation: string;            // 1-64 chars, matches /^[0-9d+\-*/() khl]+$/i — digits, d, k, h, l, +, -, *, /, (, ), spaces
  description?: string;        // 1-256 chars
  dice?: DieMetadata[];         // max 20 — { label?: string (max 32); color?: "#RGB"|"#RRGGBB" }
}
```

Notation is a strict character allowlist (`kh`/`kl` for keep-highest/
keep-lowest, e.g. `4d6kh3`) — the platform's dice engine **parses** this
notation, it never `eval()`s it. There is no way to smuggle arbitrary
expressions through a dice notation string.

`characterFields[]` can use `type: "dice"` to let a field hold/select
dice notation (from your `diceDefinitions` or free text), and can flag
`flags.quickRollDice: true` (paired with a required `labelFrom`) to surface
that field as a one-click quickroll.

---

## 9. Dice skins

`diceSkins` declares cosmetic dice appearances (requires the `dice:skin`
capability). The 3D dice renderer (`dice-box-threejs`) only exposes a
**fixed, closed set** of texture keys and materials — it has no runtime API
for loading arbitrary addon-hosted textures by name, so an addon instead
hosts its own texture/sound files at its own HTTPS base URL, laid out under
the library's fixed relative naming convention:

```ts
{
  key: string;                // snake_case, unique within the addon
  label: string;                // 1-64 chars
  assetBaseUrl?: string;         // your own https:// host — omitted or failed load falls back to `background`
  textureKey?: DiceTextureKey;   // one of a fixed enum (cloudy, fire, marble, wood, metal, bronze01...bronze04, none, etc.)
  material?: "none" | "perfectmetal" | "metal" | "wood" | "glass";
  background: string;            // REQUIRED — hex color fallback, "#RGB" or "#RRGGBB"
}
```

`background` is mandatory precisely because `assetBaseUrl` is
best-effort — if your hosted texture is absent, slow, or fails to load, the
dice must still render sensibly.

Real example, [`dice-skin-gilded-bronze/manifest.json`](./dice-skin-gilded-bronze/manifest.json):

```json
{
  "addon": {
    "slug": "dice-skin-gilded-bronze",
    "name": "Gilded Bronze Dice Skin",
    "version": "1.0.0",
    "description": "A standalone cosmetic dice skin addon...",
    "repositoryUrl": "https://github.com/raabelo/st-addons",
    "author": "saving-throw",
    "license": { "type": "original" },
    "types": ["UTILITY"],
    "tags": ["cosmetic", "dice-skin"],
    "capabilities": ["dice:skin"],
    "diceSkins": [
      {
        "key": "gilded_bronze",
        "label": "Gilded Bronze",
        "assetBaseUrl": "https://raw.githubusercontent.com/raabelo/st-addons/main/dice-skin-gilded-bronze/assets/",
        "textureKey": "bronze01",
        "material": "metal",
        "background": "#C9A227"
      }
    ]
  }
}
```

---

## 10. Quick rolls

`quickRolls` declares one-click roll shortcuts, independent of any
character field:

```ts
{
  key: string;         // snake_case
  label: string;         // 1-64 chars
  notation: string;       // same DiceNotation format as diceDefinitions
  dice?: DieMetadata[];    // max 20
}
```

---

## 11. Prompt templates (PROMPT addons)

`promptTemplates` is a `Record<key, string>` map of AI-prompt text
fragments. Requires `types` to include `PROMPT` and `capabilities` to
include `ai:prompt`.

- Map keys: `^[a-z_][a-z0-9_]*$`, max 64 chars.
- Values: max 2000 chars, **no HTML**.
- Values may only reference the following allowlisted `{{...}}` variables —
  **any other `{{token}}` fails validation**:

  ```
  {{character.name}}   {{character.class}}   {{character.race}}
  {{campaign.name}}    {{campaign.system}}    {{scene.description}}
  {{session.tone}}
  ```

This allowlist exists specifically to block prompt injection: a malicious
addon cannot reference a template variable that resolves to
attacker-controlled free text at runtime. Whatever variables you do
reference are always inserted by the platform inside XML-delimited blocks
at generation time (e.g. `<character_name>{{character.name}}</character_name>`)
— never via raw string concatenation.

---

## 12. Integrations (INTEGRATION addons)

`integrations` declares outbound webhooks to your own server. Requires
`types` to include `INTEGRATION` and `capabilities` to include
`integration:external`. Max 5 entries.

```ts
{
  endpoint: string;              // HTTPS URL only — no http://
  method: "GET" | "POST";
  serverSideOnly: true;           // REQUIRED literal true — the client can never call this directly
  events: EventName[];             // 1-10 of: session.start, session.end, roll.complete,
                                     // combat.start, combat.end, character.update
  headers?: Record<string, string>; // key max 64 chars, value max 256 chars
}
```

`serverSideOnly: true` is a required literal, not just a default — the
platform always calls your endpoint from its own server-side proxy, never
from a player's browser. At runtime, that proxy additionally blocks
requests to private IPv4/IPv6 ranges and cloud metadata addresses
(`169.254.169.254`) as SSRF protection; this is enforced platform-side and
is not something your manifest can opt out of.

---

## 13. Dependencies

```ts
{
  slug: string;               // another addon's slug
  version: string;             // semver constraint: ^1.0.0, ~1.0.0, >=1.0.0, <=1.0.0, <1.0.0, or an exact "1.0.0"
  optional?: boolean;
}
```

- Max 20 dependencies.
- Non-optional dependencies must be installed and version-compatible or
  your addon cannot be activated (`resolveDependencies` reports `missing`/
  `conflicts`).
- Circular dependency graphs (via non-optional edges) are detected by DFS
  and rejected.
- **Trust is never inherited through dependencies.** Depending on a
  well-known addon does not grant your addon any elevated capability or
  bypass its own independent validation.

---

## 14. Capabilities

`capabilities` is an explicit, minimal permission declaration — max 12
entries, default `[]`:

| Capability | Meaning |
|---|---|
| `character:read` | Read character sheet data. |
| `character:write` | Write character sheet data. |
| `campaign:read` | Read campaign-level state. |
| `campaign:write` | Write campaign-level state. |
| `session:read` | Read live session state. |
| `session:write` | Write live session state. |
| `dice:custom` | Required to declare `diceDefinitions`. |
| `dice:skin` | Required to declare `diceSkins`. |
| `board:extend` | Extend tactical-mode board behavior. |
| `chat:read` | Read chat/log messages. |
| `chat:write` | Write chat/log messages. |
| `ai:prompt` | Inject into the AI system prompt — high trust; requires `PROMPT` type; required to declare `promptTemplates`. |
| `integration:external` | Outbound HTTP via the server proxy — requires `INTEGRATION` type; required to declare `integrations`. |

GMs see your declared capabilities before activating your addon in their
campaign — **declare only what you actually use.** Over-declaring is a
trust signal issue even though it isn't a schema error.

---

## 15. Licensing

`license` is optional but strongly recommended, especially if your content
is adapted from a published RPG system:

```ts
{
  type: "original" | "ogl-1.0a" | "orc-1.0" | "cc-by-4.0" | "dpcgl" | "other";
  attributionText?: string;    // 1-1000 chars, no HTML
  sourceUrl?: string;           // HTTPS URL
  compatibilityStatement?: string; // 1-200 chars, no HTML
  monetized?: boolean;           // default false
}
```

If `type` is `"dpcgl"` (Darrington Press Community Gaming License, used for
Daggerheart-derived content), additional rules are enforced at manifest
parse time:

- `monetized` **must** be `false` — DPCGL content may never be monetized.
- `attributionText` **must** contain the exact required credit-line
  fragment: *"...© Critical Role, LLC. under the terms of the Darrington
  Press Community Gaming (DPCGL) License..."* (case-insensitive match on
  the operative clause).
- `sourceUrl` is required.
- `compatibilityStatement` is required and must contain the word
  "compatible".

If you're adapting content under a specific license, use the matching
`type` and fill in `attributionText`/`sourceUrl` even when not strictly
required by the schema — it's the right thing to do and other addon
authors/GMs will check it.

---

## 16. Locales

Addons may ship an optional `locales/<code>.json` file per supported
locale (e.g. `locales/en.json`, `locales/pt.json`), fetched from:

```
https://raw.githubusercontent.com/<owner>/<repo>/<branch>/<addon-folder>/locales/<locale>.json
```

Locale files are a flat translation bundle keyed by the manifest field/
section/dice keys they translate. Real example, `generic-core/locales/en.json`:

```json
{
  "name": "Generic Core",
  "description": "A system-agnostic starter ruleset...",
  "fields": {
    "character_name": "Character Name",
    "level": "Level"
  },
  "sections": {
    "identity": "Identity"
  },
  "dice": {
    "basic_check": "Basic Check"
  }
}
```

- `name`/`description` override the manifest's top-level display strings
  for that locale.
- `fields.<characterFields[].key>` overrides that field's `label`.
  `sections.<characterSections[].id>` overrides that section's `title`.
  `dice.<diceDefinitions[].key>` overrides that dice definition's `label`.
- Locale files are best-effort: a missing locale file, or a missing key
  within one, simply falls back to the manifest's inline `label`/`title`/
  `description` — an addon with no `locales/` folder at all is valid, it
  just isn't translated.
- Locale codes match `^[a-z]{2}(-[A-Z]{2})?$` (e.g. `en`, `en-US`)
  everywhere a locale code appears in the manifest itself (e.g.
  `presets[...].aliases`).

---

## 17. Addon composition (multiple active addons)

A campaign is never limited to a single addon. One "system" (primary)
addon plus any number of extra addons can be active simultaneously — your
addon must never assume it's the only one loaded. `composeAddonSchemas`
merges them with these rules:

| Field | Merge rule |
|---|---|
| `characterFields` | First (primary/system) addon wins on key conflict; extra addons only add fields whose key isn't already claimed. |
| `characterSections` | Appended in order; extra-addon section IDs are prefixed `"<addon-slug>__<id>"` to avoid collisions; a section is dropped if none of its fields survived composition. |
| `adversaryFields` / `adversarySections` | Same rules as above, independently. |
| `itemTypes` | First addon wins on key conflict. Each addon gets its own `"default"` item type injected before merge (see [§6](#6-items)). |
| `diceDefinitions` | First addon wins on key conflict. |
| `promptTemplates` | First addon wins on key conflict. |
| `presets` | **Concatenated per category** across all active addons (not first-wins) — e.g. a homebrew addon's `presets.class` entries are additive to the system addon's. A key collision within the same category across two different addons throws at composition time. |
| `enableSkillUse` | `true` if *any* active addon sets it. |

Practical implications for addon authors:

- Pick field/section/item-type/dice/prompt keys unlikely to collide with a
  well-known system addon if you intend your addon to *extend* an existing
  system rather than replace it.
- If you're building presets meant to extend another addon's category
  (e.g. adding homebrew classes to `presets.class`), use a distinct
  `key` — collisions are a hard error, not a silent override.
- Never write logic (there is none to write) that assumes your addon's
  sections/fields are the only ones on the sheet.

---

## 18. Migrations (versioning your manifest)

When you bump your addon's `version` and change field keys/types in a
breaking way, existing characters built against the old schema need a
migration path. `AddonMigrationFileSchema` supports exactly four
transforms — there is no general-purpose expression evaluator:

```ts
{
  fromVersion: string;   // semver
  toVersion: string;     // semver
  operations: MigrationOperation[];  // max 200
}

MigrationOperation =
  | { op: "renameField"; from: string; to: string }
  | { op: "dropField"; field: string; archive: boolean }
  | { op: "addField"; field: string; default: unknown }
  | { op: "changeType"; field: string; transform: "toString" | "toNumber" | "toBoolean" }
```

Plan field keys as **permanent identifiers** — treat a `key` rename as a
breaking change requiring a migration file, not just a manifest edit.

---

## 19. Validation checklist

Before publishing, confirm every one of these — a single failure rejects
the whole manifest:

- [ ] `addon.slug`, `name`, `version`, `description`, `repositoryUrl`, `author`, `types`, `tags` are all present
- [ ] `slug` is namespaced (`author/addon-name`) — you are **not** using a bare flat slug
- [ ] `version` is strict `major.minor.patch` semver, no pre-release suffix
- [ ] `repositoryUrl` (and every other URL field: `thumbnailUrl`, `assetBaseUrl`, `license.sourceUrl`, `integrations[].endpoint`) is `https://`
- [ ] No field contains raw HTML — every text field is checked against `/<[^>]*>/`
- [ ] `characterFields` ≤ 200, `characterSections` ≤ 20, `adversaryFields` ≤ 200, `adversarySections` ≤ 20, `itemTypes` ≤ 50, `diceDefinitions` ≤ 20, `diceSkins` ≤ 20, `quickRolls` ≤ 20, `integrations` ≤ 5, `dependencies` ≤ 20, `capabilities` ≤ 12, `options` per field ≤ 100
- [ ] `capabilities` matches declared usage: `PROMPT` type ⇒ `ai:prompt`; `INTEGRATION` type ⇒ `integration:external`; any `diceDefinitions` ⇒ `dice:custom`; any `diceSkins` ⇒ `dice:skin`; any `promptTemplates` ⇒ `ai:prompt`; any `integrations` ⇒ `integration:external`
- [ ] All `key` fields are snake_case (`^[a-z_][a-z0-9_]*$`) and unique where uniqueness is required (dice skins, item types, presets per category)
- [ ] `labelFrom`, `linkedItemType`, `visibleWhen.field` all reference real, existing keys within your own manifest
- [ ] `flags.allowCustom` only set on `type: "select"` fields
- [ ] `flags.quickRollDice: true` always paired with a `labelFrom`
- [ ] Dice notations only use `[0-9d+\-*/() khl]`
- [ ] `promptTemplates` values only use allowlisted `{{...}}` variables, ≤ 2000 chars, no HTML
- [ ] `integrations[].serverSideOnly` is the literal `true`
- [ ] No regex-based validation fields anywhere in your manifest (the schema doesn't accept any — don't try to add custom pattern fields expecting the platform to run your regex)
- [ ] If `license.type === "dpcgl"`: `monetized: false`, correct attribution fragment, `sourceUrl` set, `compatibilityStatement` contains "compatible"
- [ ] `dependencies` ≤ 20, each `version` is a valid semver constraint, and no non-optional circular dependency exists

---

## 20. Security & isolation model

Saving Throw treats **all addon data as untrusted input until validated**.
The platform enforces the following regardless of what any individual
addon manifest contains:

- **No code execution.** An addon is pure declarative JSON. There is no
  mechanism for an addon to ship or run arbitrary code — the platform's
  generic engines interpret your schema, they never `eval()` or `import()`
  anything you provide.
- **No `dangerouslySetInnerHTML`.** Text fields render as text nodes.
  `richtext` fields are the sole exception (WYSIWYG HTML), and are
  sanitized on render by the platform, not trusted verbatim.
- **XSS is blocked at the schema layer.** Every text field is validated
  against `/<[^>]*>/` and rejected if it contains anything HTML-shaped.
- **No regex from addons.** The platform never accepts a regex pattern
  from an addon manifest (ReDoS vector) — all string validation is
  hardcoded Zod on the platform side.
- **Dice notation is parsed, never evaluated.** The notation character
  allowlist plus a real dice-notation parser make arbitrary expression
  injection structurally impossible.
- **Prompt injection is blocked by allowlist**, not by attempted
  sanitization — see [§11](#11-prompt-templates-prompt-addons).
- **SSRF protection is enforced server-side**, not addon-configurable —
  HTTPS-only URLs, `serverSideOnly: true` required literal, private-IP and
  cloud-metadata-IP blocking at the API proxy — see
  [§12](#12-integrations-integration-addons).
- **No prototype pollution.** `validateManifest()`'s Zod output is a fresh
  typed object; nothing from your raw JSON is ever merged into an existing
  object on the platform side. `.strict()` on every schema means unknown
  keys (e.g. `__proto__`) fail validation, they are never silently
  stripped-and-ignored-then-merged.
- **Published versions are immutable.** The same `slug@version` can never
  be republished with different content — always bump `version`.
- **Composability is mandatory, not optional** — see
  [§17](#17-addon-composition-multiple-active-addons). Do not assume your
  addon is the only one active in a campaign.
- **Capabilities are least-privilege and GM-visible** — declare only what
  your addon actually needs; GMs review the capability list before
  activating your addon.

---

## 21. Minimal end-to-end example

A brand-new homebrew system addon, published by a community author:

```
your-namespace/simple-adventures/
  manifest.json
  locales/
    en.json
```

`manifest.json`:

```json
{
  "addon": {
    "slug": "your-namespace/simple-adventures",
    "name": "Simple Adventures",
    "version": "1.0.0",
    "description": "A lightweight homebrew system for one-shot adventures.",
    "repositoryUrl": "https://github.com/your-namespace/simple-adventures-addon",
    "author": "Your Name",
    "license": { "type": "original" },
    "types": ["CORE", "RULES"],
    "tags": ["homebrew", "lightweight"],
    "capabilities": ["character:read", "character:write", "dice:custom"],
    "characterFields": [
      { "key": "character_name", "label": "Name", "type": "text", "section": "identity", "required": true },
      { "key": "grit", "label": "Grit", "type": "number", "section": "attributes", "min": 1, "max": 10, "default": 5 },
      { "key": "hit_points", "label": "Hit Points", "type": "resource", "section": "combat" }
    ],
    "characterSections": [
      { "id": "identity", "title": "Identity", "layout": "grid", "fields": ["character_name"] },
      { "id": "attributes", "title": "Attributes", "layout": "grid", "fields": ["grit"] },
      { "id": "combat", "title": "Combat", "layout": "grid", "fields": ["hit_points"] }
    ],
    "diceDefinitions": [
      { "key": "grit_check", "label": "Grit Check", "notation": "1d20", "description": "Roll to overcome a challenge" }
    ]
  }
}
```

`locales/en.json`:

```json
{
  "name": "Simple Adventures",
  "description": "A lightweight homebrew system for one-shot adventures.",
  "fields": {
    "character_name": "Name",
    "grit": "Grit",
    "hit_points": "Hit Points"
  },
  "sections": {
    "identity": "Identity",
    "attributes": "Attributes",
    "combat": "Combat"
  },
  "dice": {
    "grit_check": "Grit Check"
  }
}
```

For a full worked example that also uses `presets`, `itemTypes`, and
multiple `characterSections`, read [`generic-core/manifest.json`](./generic-core/manifest.json)
in this repo end to end — it's a complete, minimal-but-real system addon
and the recommended starting template.

---

## 22. Publishing checklist

1. Copy `generic-core/` as a starting template, or start from the minimal
   example above.
2. Choose a namespaced slug: `your-github-username/your-addon-name`.
3. Fill in every required manifest field (§4.1).
4. Run through the full [validation checklist](#19-validation-checklist).
5. Push to a **public** GitHub repository (raw-content fetches are
   unauthenticated and cannot access private repos).
6. Set `repositoryUrl` to that repo's URL.
7. Tag/commit your `1.0.0` release. Remember: once a `slug@version` is
   published, its content is immutable — ship a new `version` for any
   future change, however small.
8. Optionally add `locales/<code>.json` translation bundles.
9. Share the repo — GMs install by referencing your addon's slug; the
   platform fetches your manifest directly from your repo's raw content
   URL at activation/composition time.
