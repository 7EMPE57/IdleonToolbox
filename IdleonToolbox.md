# IdleonToolbox — Full Project Understanding

A complete reference for the IdleonToolbox Next.js web app, written for porting game logic to Python automation.

---

## What This Project Is

**IdleonToolbox** is a fan-made web dashboard for the idle game *Legends of Idleon*. It:
1. Authenticates the player via Firebase (Google, Apple, or Steam)
2. Reads the player's live cloud save from Firestore (`_data/{uid}`)
3. Parses the raw save data through a chain of TypeScript parsers
4. Displays the structured results in a React/Next.js UI (dashboard, per-world pages, character viewer, etc.)

The project is **not** affiliated with the game developer. All game data formulas are reverse-engineered.

---

## Data Flow Overview

```
Firebase Realtime DB / Firestore
        │
        ▼
firebase/index.js  ← auth + subscribe()
        │  raw IdleonData (JSON cloud-save blob)
        ▼
parsers/index.ts → parseData()
        │  3-pass serializeData() loop
        ▼
{ account: Account, characters: Character[] }
        │
        ▼
React components  (pages/, components/)
```

---

## Directory Structure

```
IdleonToolbox/
├── firebase/           Firebase auth + Firestore/RTDB read logic
├── parsers/            TypeScript parsers — the core data layer
│   ├── index.ts        Entry point: parseData()
│   ├── character.ts    Per-character initialization
│   ├── misc.ts         Account-wide helpers
│   ├── world-1/        Anvil, stamps, statues, forge, bribes, owl
│   ├── world-2/        Alchemy, arcade, islands, kangaroo, voteBallot, weeklyBosses
│   ├── world-3/        Construction, refinery, printer, shrines, postoffice,
│   │                   deathNote, atomCollider, equinox, worship, saltLick,
│   │                   prayers, traps, armorSmithy, hatRack
│   ├── world-4/        Cooking, lab, breeding, rift, tome
│   ├── world-5/        Sailing, divinity, gaming, hole, caverns/
│   ├── world-6/        Sneaking, farming, summoning, emperor
│   ├── world-7/        Spelunking, gallery, coralReef, clamWork, research,
│   │                   minehead, button, tournament, sushiStation, legendTalents
│   ├── class-specific/ Grimoire, compass, tesseract
│   ├── clickers/       Bubba
│   └── misc/           upgradeVault, activeCalculator
├── utility/
│   ├── helpers.js      Core utility functions (tryToParse, growth, etc.)
│   ├── lavaRand.js     Seeded PRNG used by game formulas
│   ├── consts.js       Tool constants
│   ├── migrations.js   Dashboard config version migrations
│   └── dashboard/
│       ├── account.js  Account-wide dashboard alert logic
│       └── characters.js Per-character dashboard alert logic
├── data/
│   ├── website-data.json   Static game data (formulas, names, defs) — 142 keys
│   └── account-options-enum.js  Named index map for accountOptions[]
├── components/         React UI components (.jsx)
├── pages/              Next.js page routes
├── hooks/              React hooks (useFormatDate, useRealDate, useAlerts, etc.)
└── services/auth/      Server-side OAuth backends (Google, Apple, Steam)
```

---

## The Two JSON Sources

Everything in the system draws from exactly **two JSON sources**:

| Source | What it is | How accessed |
|--------|-----------|--------------|
| **`IdleonData`** (Firebase) | Live player save state | `idleonData.SomeField` — most fields are JSON strings |
| **`website-data.json`** | Static game definitions | Imported per parser: `import { stamps, cauldrons, ... } from '@website-data'` |

These never merge automatically. Every parser manually joins them: reads level/progress from `IdleonData`, looks up the formula from `website-data.json`, calls `growth()`.

---

## `IdleonData` — Raw Firebase Cloud Save

**Type definition:** `parsers/generated-firebase-types.ts`

The game saves to Firestore under `_data/{uid}`. The raw document shape is `IdleonData`.

### Key Conventions

- **Most string fields are JSON-encoded** — call `tryToParse()` before using them
- **Per-character fields** use `_N` suffix: `idleonData["SL_0"]` = character 0 skill levels
- **Sparse arrays** from Firebase look like `{ "0": v0, "1": v1, "length": N }` — must be converted

### Important Account-Wide Fields

| Field | Type | Meaning |
|-------|------|---------|
| `OptLacc` | JSON string | Account options array (`accountOptions[N]`) |
| `TimeAway` | JSON string | AFK time-away data |
| `CauldronBubbles` | JSON string | Which bubbles each character has equipped |
| `CauldronInfo` | SparseArray[] | Bubble levels per cauldron (4 cauldrons × bubbles) |
| `CauldronP2W` | JSON string | Pay-to-win alchemy (vials, sigils, cauldron upgrades) |
| `CauldUpgLVs` / `CauldUpgXPs` | arrays | Cauldron upgrade levels/XP |
| `StampLv` | JSON string | Stamp levels (3 arrays: combat, skills, misc) |
| `StampLvM` | JSON string | Stamp max levels |
| `Lab` | JSON string | Laboratory state (player positions, chips, jewels) |
| `Breeding` | JSON string | Pet eggs, territory, shiny progress |
| `Cooking` | JSON string | Meal levels, kitchen data |
| `Sailing` | JSON string | Boats, captains, artifacts, loot pile |
| `Divinity` | JSON string | God levels, player links |
| `Gaming` | JSON string | Imports, superbits, mutations, palette |
| `Hole` | JSON string | Cavern villagers and bonuses |
| `Summoning` | JSON string | Win bonuses, essence, familiars |
| `Farming` | JSON string | Crop plots, seed levels |
| `Emperor` | JSON string | World 6 Emperor bonuses |
| `Atoms` | number[] | Atom collider particle counts |
| `GemsOwned` | number | Gem currency (not a JSON string) |
| `MoneyBANK` | string/number | Gold in bank |
| `Cards0` / `Cards1` | JSON string | Card collection data |
| `AchieveReg` | JSON string | Achievement completion flags |
| `BribeStatus` | number[] | World 1 bribe completion |
| `WeeklyBoss` | JSON string | Weekly boss keys/state |
| `Guild` | JSON string | Guild stats |
| `Rift` | JSON string | Rift task completion |
| `Tome` | JSON string | Tome of knowledge scores |
| `ChestOrder` / `ChestQuantity` | arrays | Storage chest item IDs and quantities |

### Per-Character Fields (suffix `_N`, N = 0-based index)

| Firebase key | Renamed to in parser | Notes |
|-------------|---------------------|-------|
| `SL_N` | `SkillLevels` | Sparse array — skill levels by index |
| `SM_N` | `SkillLevelsMAX` | Max skill levels |
| `PVStatList_N` | `PersonalValuesMap.StatList` | Stats: HP, MP, level, STR, etc. `[4]` = char level |
| `PVtStarSign_N` | `PersonalValuesMap.StarSign` | Equipped star sign string |
| `PVFishingToolkit_N` | `PersonalValuesMap.FishingToolkit` | Fishing kit |
| `EquipOrder_N` | `EquipmentOrder` | Equipped items (createArrayOfArrays applied) |
| `EquipQTY_N` | `EquipmentQuantity` | Equipped item quantities |
| `EMm0_N` | `EquipmentMap[0]` | Equipment slot 0 (weapon/armor) |
| `EMm1_N` | `EquipmentMap[1]` | Equipment slot 1 (food) |
| `IMm_N` | `InventoryMap` | Inventory item map |
| `ItemQTY_N` | `ItemQuantity` | Inventory quantities |
| `BuffsActive_N` | `BuffsActive` | Active buff slots |
| `AnvilPA_N` | `AnvilPA` | Anvil production data |
| `POu_N` | `PostOfficeInfo` | Post office box levels |
| `ObolEqO0_N` | `ObolEquippedOrder` | Equipped obols |
| `ObolEqMAP_N` | `ObolEquippedMap` | Obol slot map |
| `SLpre_N` | `SkillPreset` | Skill preset |
| `KLA_N` | `KillsLeft2Advance` | Kills needed to advance map |
| `AtkCD_N` | `AttackCooldowns` | Attack cooldown values |
| `PTimeAway_N` | `PlayerAwayTime` | AFK time (×1000 to convert ms) |
| All others `XYZ_N` | `XYZ` | Strip `_N` suffix |

---

## Skill Index Map

**File:** `parsers/parseMaps.ts`

Maps the numeric index in `SkillLevels_N` to skill names. Also used in stamp `skillIndex` and talent bonus fields.

| Index | Skill |
|-------|-------|
| 0 | character (base class level) |
| 1 | mining |
| 2 | smithing |
| 3 | chopping |
| 4 | fishing |
| 5 | alchemy |
| 6 | catching |
| 7 | trapping |
| 8 | construction |
| 9 | worship |
| 10 | cooking |
| 11 | breeding |
| 12 | laboratory |
| 13 | sailing |
| 14 | divinity |
| 15 | gaming |
| 16 | farming |
| 17 | sneaking |
| 18 | summoning |
| 19 | spelunking |
| 20 | research |

Index 0 is identical to `PersonalValuesMap.StatList[4]` (the character's displayed level).

---

## `firebase/index.js` — Data Fetching

**Auth providers:** Google, Apple, Steam (via `services/auth/`)

**`subscribe(uid, accessToken, callback)`**
- Reads `_uid/{uid}` from Realtime DB → character names list
- Reads `_vars/_vars` from Firestore → server variables
- Reads `_data/{uid}` from Firestore → cloud save snapshot
- Also fetches: companion (`_comp/{uid}`), guild ID (`_usgu/{uid}/g`), guild data (`_guild/{guildId}`), tournament data
- Sets up a **real-time listener** — fires `callback` on every save update

The callback signature:
```
callback(cloudsave, charNames, companion, guildData, tournament, serverVars, createTime, uid, accessToken)
```

**For Python:** Use Firebase REST API to authenticate, then GET/subscribe to `_data/{uid}` in Firestore. The field names match `IdleonData` exactly.

---

## `parsers/index.ts` — Entry Point

**`parseData(idleonData, charNames, companion, guildData, serverVars, accountCreateTime, tournament)`**

Returns `{ account: Account, characters: Character[] }`.

### Execution order

1. **`getStaticData()`** — parsers with no cross-dependencies. Runs once.
2. **`serializeData()`** — all dynamic parsers. Runs **3 times** (passes).

### Why 3 passes?

Some parsers depend on values that other parsers compute in the same step:
- Pass 1: base values established
- Pass 2: lab bonuses apply to stamps/vials/meals
- Pass 3: final values stabilize (atoms re-applied, farming updated, etc.)

---

## Static Parsers (run once, safe to port independently)

| Function | File | Returns |
|----------|------|---------|
| `getCharacters()` | `character.ts` | Raw per-character data arrays |
| `getTasks()` | `tasks.ts` | Task board, merits, recipe unlocks |
| `getConstellations()` | `starSigns.ts` | Star constellation progress |
| `getCompanions()` | `misc.ts` | Companion bonuses |
| `getBundles()` | `misc.ts` | Purchased bundle flags (`bun_a`…`bun_l`) |
| `getGemShop()` | `gemShop.ts` | Gem shop purchase counts |
| `getBribes()` | `world-1/bribes.ts` | Bribe completion status |
| `getObols()` | `obols.ts` | Obol family bonuses |
| `getSlab()` | `misc.ts` | Looty slab item checklist |
| `getPostOfficeShipments()` | `world-3/postoffice.ts` | Post office box levels |
| `getTowers()` | `world-3/construction.ts` | Construction tower data |
| `getAchievements()` | `achievements.ts` | Achievement completion |
| `getRift()` | `world-4/rift.ts` | Rift task completion |
| `getShops()` | `shops.ts` | Shop stock |
| `getTraps()` | `world-3/traps.ts` | Trapping data |
| `getTotems()` | `world-3/worship.ts` | Worship totems |
| `getAdviceFish()` | `misc.ts` | Advice fish bonuses |
| `getGuild()` | `guild.ts` | Guild bonuses |

---

## Dynamic Parsers (order matters — each may depend on prior outputs)

| Parser | File | Key dependencies |
|--------|------|-----------------|
| `getAlchemy()` | `world-2/alchemy.ts` | `accountOptions` |
| `getArmorSmithy()` | `world-3/armorSmithy.ts` | `account` |
| `getStorage()` | `storage.ts` | `account` |
| `getSaltLick()` | `world-3/saltLick.ts` | `storage.list` |
| `getDungeons()` | `dungeons.ts` | `accountOptions` |
| `getPrayers()` | `world-3/prayers.ts` | `storage.list` |
| `getCards()` | `cards.ts` | `account` |
| `getCurrencies()` | `misc.ts` | `account`, `idleonData` |
| `getStamps()` | `world-1/stamps.ts` | `account` |
| `getBreeding()` | `world-4/breeding.ts` | `account` |
| `getCooking()` | `world-4/cooking.ts` | `account` |
| `getDivinity()` | `world-5/divinity.ts` | `account`, characters |
| `getSneaking()` | `world-6/sneaking.ts` | `account`, characters |
| `getFarming()` | `world-6/farming.ts` | `account` |
| `getSummoning()` | `world-6/summoning.ts` | `account` |
| `getHole()` | `world-5/hole.ts` | `account` |
| `getLab()` | `world-4/lab.ts` | `account`, characters |
| `getShrines()` | `world-3/shrines.ts` | `account` |
| `getStatues()` | `world-1/statues.ts` | `account`, characters |
| `getArcade()` | `world-2/arcade.ts` | `account` |
| `applyStampsMulti()` | `world-1/stamps.ts` | `lab.labBonuses[7]` |
| `updateVials()` | `world-2/alchemy.ts` | `lab.labBonuses[10]`, rift, vault |
| `getEquinox()` | `world-3/equinox.ts` | `account` |
| `applyMealsMulti()` | `world-4/cooking.ts` | `lab.jewels[16]` |
| `getStarSigns()` | `starSigns.ts` | `account` |
| `initializeCharacter()` | `character.ts` | full `account` |
| `getGrimoire()` | `class-specific/grimoire.ts` | characters, account |
| `getCompass()` | `class-specific/compass.ts` | characters, account |
| `getTesseract()` | `class-specific/tesseract.ts` | characters, account |
| `getConstruction()` | `world-3/construction.ts` | `account` |
| `getAtoms()` | `world-3/atomCollider.ts` | `account` |
| `getArtifacts()` / `getSailing()` | `world-5/sailing.ts` | artifacts, characters, account |
| `getGaming()` | `world-5/gaming.ts` | characters, account |
| `getForge()` | `world-1/forge.ts` | `account` |
| `getRefinery()` | `world-3/refinery.ts` | `storage.list`, `tasks` |
| `getPrinter()` | `world-3/printer.ts` | characters, account |
| `getQuests()` | `quests.ts` | characters |
| `getIslands()` | `world-2/islands.ts` | account, characters |
| `getDeathNote()` | `world-3/deathNote.ts` | characters, account |
| `getKillRoy()` | `misc.ts` | characters, account |
| `getTome()` | `world-4/tome.ts` | account, characters |
| `getOwl()` | `world-1/owl.ts` | account |
| `getKangaroo()` | `world-2/kangaroo.ts` | account |
| `getVoteBallot()` | `world-2/voteBallot.ts` | account |
| `getUpgradeVault()` | `misc/upgradeVault.ts` | account, characters |
| `getEmperor()` | `world-6/emperor.ts` | account |
| `getLegendTalents()` | `world-7/legendTalents.ts` | account, characters |
| `getGallery()` | `world-7/gallery.ts` | account |
| `getCoralReef()` | `world-7/coralReef.ts` | account, characters |
| `getClamWork()` | `world-7/clamWork.ts` | account |
| `getMinehead()` | `world-7/minehead.ts` | account |
| `getTournament()` | `world-7/tournament.ts` | account |
| `getResearch()` | `world-7/research.ts` | account, characters |
| `getButton()` | `world-7/button.ts` | account |
| `getSushiStation()` | `world-7/sushiStation.ts` | account |
| `getBubba()` | `clickers/bubba.ts` | account |

---

## Core Helper Functions

### `tryToParse` — Universal Firebase Decoder

**File:** `utility/helpers.js:231`

Almost every `idleonData.*` field is a JSON-encoded string. Call this before using any value.

```js
export const tryToParse = (str) => {
  try { return JSON.parse(str); }
  catch (err) { return str; }
};
```

**Python:**
```python
import json
def try_to_parse(val):
    if isinstance(val, str):
        try: return json.loads(val)
        except: return val
    return val
```

### `createArrayOfArrays` — Sparse Array Converter

**File:** `utility/helpers.js:261`

Firebase stores arrays as `{ "0": v0, "1": v1, "length": N }`. This converts an array of those into a proper 2D array.

```python
def sparse_to_list(obj):
    if isinstance(obj, dict) and 'length' in obj:
        return [obj.get(str(i)) for i in range(int(obj['length']))]
    return obj

def create_array_of_arrays(arr):
    return [sparse_to_list(x) for x in arr]
```

### `createIndexedArray` — Sparse Object to Dense List

**File:** `utility/helpers.js:270`

Converts a sparse dict to a dense list, filling gaps with `{}`:
```
{ "0": 5, "2": 10, "length": 3 } → [5, {}, 10]
```

### `growth()` — The Core Game Formula Engine

**File:** `utility/helpers.js:287`

**The single most important function.** Every bonus value is computed through this — stamps, bubbles, vials, talents, cauldrons, etc.

```js
growth(func, level, x1, x2, shouldRound = true)
```

| `func` | Formula | Use case |
|--------|---------|----------|
| `'add'` | `(((x1+x2)/x2 + 0.5*(level-1)) / (x1/x2)) * level * x1` | Most stamps, bubbles |
| `'decay'` | `(x1 * level) / (level + x2)` | Cap-approaching bonuses |
| `'decayMulti'` | `1 + (x1 * level) / (level + x2)` | Multiplicative decay bonuses |
| `'bigBase'` | `x1 + x2 * level` | Simple linear bonuses |
| `'addDECAY'` | Linear to 50000, then soft-capped | Very high level stamps |
| `'reduce'` | `x1 - x2 * level` | Cost reductions |
| `'bigBaseLower'` | `x2` | Returns x2 only |
| `'intervalAdd'` | `x1 + floor(level / x2)` | Step-function bonuses |
| `'special1'` | `100 - (level * x1) / (level + x2)` | Percentage reductions |

`x1`, `x2`, `func` come from `website-data.json`. `level` comes from `IdleonData`.

**Python:**
```python
import math

def growth(func, level, x1, x2, should_round=True):
    result = 0
    if func == 'add':
        result = (((x1 + x2) / x2 + 0.5 * (level - 1)) / (x1 / x2)) * level * x1 if x2 != 0 else x1 * level
    elif func == 'decay':
        result = (x1 * level) / (level + x2)
    elif func == 'decayMulti':
        result = 1 + (x1 * level) / (level + x2)
    elif func == 'bigBase':
        result = x1 + x2 * level
    elif func == 'addDECAY':
        result = x1 * level if level < 50001 else x1 * min(50000, level) + ((level - 50000) / (level - 50000 + 150000)) * x1 * 50000
    elif func == 'reduce':
        result = x1 - x2 * level
    elif func == 'bigBaseLower':
        result = x2
    elif func == 'intervalAdd':
        result = x1 + math.floor(level / x2)
    elif func == 'special1':
        result = 100 - (level * x1) / (level + x2)
    return round(result * 100) / 100 if should_round else result
```

### Other Math Helpers

| Function | Formula | Purpose |
|----------|---------|---------|
| `lavaLog(n)` | `log(max(n,1)) / 2.30259` | Base-10 log approximation |
| `lavaLog2(n)` | `log(max(n,1)) / log(2)` | Base-2 log |
| `round(n)` | `round((n + ε) * 100) / 100` | 2-decimal rounding |
| `getCoinsArray(n)` | Splits gold into `[plat, gold, silver, copper]` | Currency display |
| `cleanUnderscore(str)` | `str.replace(/_/g, ' ')` | Display names |
| `notateNumber(n, type)` | K / M / B / T / Q notation | Large number display |

---

## `LavaRand` — Seeded PRNG

**File:** `utility/lavaRand.js`

The game uses a deterministic PRNG (not `Math.random()`). Outcomes are reproducible from save data. Used for: chip/jewel rotations, random events, drop tables, etc.

```js
const rng = new LavaRand(seed);
rng.random(100); // returns integer 0–99
```

**Python port:**
```python
import ctypes

class LavaRand:
    def __init__(self, seed):
        self.seed = seed
        self.seed2 = self._hash(seed)
        if self.seed == 0: self.seed = 1
        if self.seed2 == 0: self.seed2 = 1

    def _mul(self, a, b):
        return ctypes.c_int32(a * b).value

    def _hash(self, e, t=5381):
        e = self._mul(e, -862048943)
        t ^= self._mul((e << 15) | (e >> 17 & 0x7FFF), 461845907)
        t = (self._mul((t << 13) | (t >> 19 & 0x1FFF), 5) + -430675100) & 0xFFFFFFFF
        t = ctypes.c_int32(t).value
        t = self._mul(t ^ (t >> 16), -2048144789)
        return self._mul(t ^ (t >> 13), -1028477387) ^ (t >> 16)

    def random(self, e):
        self.seed = ctypes.c_int32(36969 * (self.seed & 65535) + (self.seed >> 16)).value
        self.seed2 = ctypes.c_int32(18000 * (self.seed2 & 65535) + (self.seed2 >> 16)).value
        return (1073741823 & ((self.seed << 16) + self.seed2)) % e
```

---

## How Bonuses Are Fetched — End-to-End Example

**"What is my bubble bonus for `CROPIUS_MAPPER`?"**

1. Firebase `CauldronInfo` (sparse array) → `createArrayOfArrays` → 4 cauldron level arrays
2. `getBubbles()` merges level from Firebase with static def from `website-data.json cauldrons['kazam'][index]`  
   Result: `{ level: 14, func: 'decay', x1: 60, x2: 40, bubbleName: 'CROPIUS_MAPPER', ... }`
3. Later: `getBubbleBonus(account, 'CROPIUS_MAPPER', false)` finds the merged object and calls:  
   `growth('decay', 14, 60, 40, false)` → `(60 * 14) / (14 + 40)` → `15.56`

That `15.56` is the % bonus value.

---

## Bonus Stacking Pattern

Multiple sources combine to form a final stat. Always:

```js
// Gather each source
const bubbleBonus  = getBubbleBonus(account, 'BUBBLE_NAME', false);
const stampBonus   = getStampsBonusByEffect(account, 'Effect_Name');
const vialBonus    = getVialsBonusByEffect(account.alchemy.vials, null, 'StatName');
const labBonus     = getLabBonus(account.lab.labBonuses, 7);
const mealBonus    = getMealsBonusByEffectOrStat(account, null, 'StatName', mealMulti);
const arcadeBonus  = getArcadeBonus(account.arcade.shop, 'Effect_Name')?.bonus;

// Combine — additive within a group, then multiplicative across
const total = baseStat
  * (1 + (bubbleBonus + stampBonus + vialBonus + mealBonus + arcadeBonus) / 100)
  * labBonus;
```

Every `getXxxBonus` function follows the same pattern:
1. Find the item in `account.*` (already merged with static data during parsing)
2. Call `growth(func, level, x1, x2)`
3. Return a plain number (usually a percentage like `15.3`)

---

## Key Bonus Getters

### Stamps (`parsers/world-1/stamps.ts`)

```
idleonData.StampLv → tryToParse → [combat[], skills[], misc[]]
                                    ↓ merged with website-data.json stamps
                                  account.stamps.{combat,skills,misc}[]
```

- `getStampBonus(account, stampTree, stampName, character?)` — finds stamp by rawName, applies skill-level reduction + charm + exalted multiplier, calls `growth()`
- `getStampsBonusByEffect(account, effectName)` — searches all 3 trees, sums all matches

### Bubbles (`parsers/world-2/alchemy.ts`)

`getBubbleBonus(account, bubbleName, round?, shouldMultiply?)` — 3-layer multiplier:
1. Base: `growth(func, level, x1, x2)`
2. Prisma multiplier (if bubble is prisma)
3. Primary multiplier from cauldron bubble index 1 (if `shouldMultiply=true`)
4. Secondary multiplier from cauldron bubble index 16 (for specific bubble indexes)

### Vials (`parsers/world-2/alchemy.ts`)

`getVialsBonusByEffect(vials, effectName, statName?)` — filters vials by desc/stat, sums `growth()` × multiplier.

Vial multiplier (`updateVials()`): lab bonus 10 × rift Vial_Mastery × upgradeVault × meritocracy.

### Lab Bonuses (`parsers/world-4/lab.ts`)

`getLabBonus(account.lab.labBonuses, index)` — returns bonus if that lab node is connected.

Key indices:
- `6` = Viaduct of Gods (liquid cauldron cap ×)
- `7` = Certified Stamp Book (stamp ×)
- `8` = Spelunker Obol (jewel ×)
- `10` = My 1st Chemistry Set (vial ×)

---

## `website-data.json` — All 142 Keys

Static game data. Never changes at runtime. Imported by parsers via `@website-data`.

### Bonus Systems

| Key | Shape | Content |
|-----|-------|---------|
| `stamps` | `{combat:[],skills:[],misc:[]}` | `{name,func,x1,x2,effect,baseCoinCost,baseMatCost,...}` |
| `cauldrons` | `{power:[],quicc:[],high-iq:[],kazam:[]}` | Bubble defs: `{name,bubbleName,func,x1,x2,desc}` |
| `vials` | `{0:[],1:[],...}` (by cauldron) | `{name,func,x1,x2,desc,stat}` |
| `sigils` | list[24] | `{name,unlockCost,boostCost,jadeCost,desc,x1,x2,func}` |
| `p2w` | list[4] | P2W formula triplets `[x1,x2,func]` |
| `prayers` | list[25] | `{name,effect,curse,x1,x2,func,cursex1,cursex2,cursefunc}` |
| `statues` | list[32] | `{index,name,effect,dk,bonus}` |
| `shrines` | dict | `{name,func,x1,x2,desc}` keyed by shrine ID |
| `talents` | dict | By class name → array of `{name,func,x1,x2,bonus,description}` |
| `arcadeShop` | list[68] | `{effect,x1,x2,func,bonusName}` |
| `labBonuses` | list[18] | `{index,x,y,range,bonusOn,bonusOff,name,func,x1,x2}` |
| `chips` | list[22] | `{index,name,bonus,bool1,bool2}` |
| `jewels` | list[24] | `{index,name,bonus,x,y,radius}` |
| `guildBonuses` | list[18] | `{name,bonus,gpBaseCost,gpIncrease}` |
| `gods` | list[10] | `{name,majorBonus,minorBonus,x1,x2}` |
| `artifacts` | list[41] | `{name,x1,baseFindChance,description}` |
| `atomsInfo` | list[15] | `{name,desc,x1,x2,func}` |
| `saltLicks` | list[10] | `{name,rawName,desc}` |
| `dungeonStats` | list[8] | `{effect,x1,x2,func,bonusName,type}` |
| `dungeonFlurboStats` | list[8] | `{effect,x1,x2,func,bonusName}` |
| `dungeonCreditShop` | list[48] | `{name,bonus,increment,rarity}` |
| `arenaBonuses` | list[16] | `{bonus,wave}` |
| `petUpgrades` | list[13] | Breeding upgrades |
| `petGenes` | list[36] | `{name,abilityType,combatAbility}` |
| `territory` | list[29] | `{territoryName,background,powerReq}` |
| `equinoxUpgrades` | list[14] | `{name,description}` |
| `equinoxChallenges` | list[36] | `{label,goal,reward}` |
| `riftInfo` | list[70] | `{monsterName,task}` |
| `postOffice` | list[24] | `{name,upgradeLevels,upgrades}` |
| `superbitsUpgrades` | list[72] | Gaming superbits |
| `gamingImports` | list[10] | `{boxName,boxDescription}` |
| `gamingPalette` | list[37] | Palette color and bonuses |
| `obols` | `{character:{},family:{}}` | Obol shape/stat defs |
| `cardBonuses` | dict | Card bonuses by set size (1-6) |
| `cardSets` | dict | Card set bonus defs |
| `cards` | dict | Per-monster card defs (keyed by rawName) |
| `constellations` | list[49] | `{rawIndex,mapIndex,requiredPlayers,points,name}` |
| `starSigns` | list[94] | `{starName,cost,bonuses:[{bonus,effect}]}` |
| `companions` | list[161] | `{name,rawName,effect}` |
| `merits` | list[6] | Merit board defs |
| `tasks` | list[6] | Task board world defs |
| `achievements` | list[420] | `{name,rawName,quantity,desc}` |
| `bribes` | list[41] | `{name,desc,value}` |

### Items & Equipment

| Key | Shape | Content |
|-----|-------|---------|
| `items` | dict | Every item by rawName: `{displayName,sellPrice,typeGen,ID,Type,...}` |
| `itemsArray` | list[2393] | Same items as flat indexed array |
| `crafts` | dict | Crafting recipes by item rawName |
| `anvilProducts` | `{0:[],1:[],...}` | Anvil tab products |
| `anvilUpgradeCost` | list[26] | Cost thresholds |
| `invBags` | dict | Inventory bag defs |
| `carryBags` | dict | Carry bag defs by type |
| `invStorage` | dict | Storage chest slot defs |
| `equipmentSets` | list[19] | `{setName,armors:[...]}` |

### World & Map Data

| Key | Shape | Content |
|-----|-------|---------|
| `mapNames` | dict | Map index → human name |
| `rawMapNames` | list[327] | Map raw names (asset paths) |
| `mapEnemies` | dict | Monster rawName → maps it appears on |
| `mapEnemiesArray` | list[327] | Per-map-index monster list |
| `mapDetails` | list[327] | `[[worldId,mapType],[w,h,overlap],[width,height,zoneLineY]]` |
| `mapPortals` | dict | Map portal connections |
| `monsters` | dict | Monster defs by rawName |
| `monsterDrops` | dict | Per-monster drop tables |
| `deathNote` | list[98] | `{rawName,name,world}` |

### World-Specific Systems

| Key | Shape | System |
|-----|-------|--------|
| `refinery` | dict | Refinery salt tiers (Refinery1-6) |
| `towers` | dict | Construction tower defs |
| `cogKeyMap` | dict | Cog type mapping |
| `cookingMenu` | list[74] | `{name,cookReq,rawName,baseStat}` |
| `islands` | list[17] | `{name,distance,unlockReq,cloudsUnlocked}` |
| `captainsBonuses` | list[5] | Captain specialty bonuses |
| `traps` | list[7] | Trap type defs |
| `totems` | list[8] | Worship totem defs |
| `ballsBonuses` | list[24] | Arcade ball bonuses |
| `weeklyBosses` | list[9] | Weekly boss defs |
| `weeklyBossesActions` | list[22] | Boss action options |
| `killRoySkullShop` | list[20] | Killroy shop items |
| `tomeData` | list[118] | Tome of knowledge entries |
| `jadeUpgrades` | list[50] | Sneaking jade upgrades |
| `ninjaUpgrades` | list[29] | Ninja upgrades |
| `pristineCharms` | list[23] | Pristine charm bonuses |
| `seedInfo` | list[7] | Farming seed types |
| `marketInfo` | list[16] | Farming market bonuses |
| `exoticMarketInfo` | list[80] | Exotic market bonuses |
| `summoningUpgrades` | list[82] | Summoning upgrade defs |
| `summoningEnemies` | list[133] | Summoning enemy defs |
| `summoningBonuses` | list[32] | `{bonusId,bonus}` |
| `divStyles` | list[8] | Divinity style defs |
| `owlData` | list[9] | Owl upgrade defs |
| `poppyBonuses` | list[12] | Kangaroo upgrade bonuses |
| `bubbaUpgrades` | list[28] | Bubba clicker upgrades |
| `holesInfo` | list[75] | Cavern layout data |
| `holesBuildings` | list[100] | Hole building defs |
| `lampWishes` | list[12] | Hole lamp wish defs |
| `grimoire` | list[55] | Grimoire upgrade defs |
| `compass` | list[173] | Compass upgrade defs |
| `abominations` | list[35] | Compass abomination defs |
| `tesseract` | list[63] | Tesseract upgrade defs |
| `upgradeVault` | list[90] | `{name,x1,x2,x3,maxLevel}` |
| `spelunkingUpgrades` | list[53] | Spelunking upgrade defs |
| `legendTalents` | list[50] | Legend talent bonuses |
| `coralReef` | list[6] | Coral reef upgrade defs |
| `mineheadUpgrades` | list[30] | Minehead upgrade defs |
| `sushiUpgrades` | list[45] | Sushi station upgrades |
| `ButtonTasks` | list[57] | Button clicker task defs |
| `researchGridSquares` | list[240] | Research grid cell defs |
| `emperorBonuses` | list[48] | Emperor bonus defs |
| `zenithMarket` | list[10] | Zenith market defs |

### Meta / Classes

| Key | Shape | Content |
|-----|-------|---------|
| `classes` | list[63] | Class name by ID (0-based) |
| `classFamilyBonuses` | list[42] | `{name,func,x1,x2,x3,order}` |
| `stats` | dict | Base stat defs |
| `randomList` | list[115] | Internal random data (LavaRand seeds) |
| `slab` | list[1886] | All item names for looty checklist |
| `shops` | dict | NPC shop inventories by index |
| `gemShop` | list[4] | Gem shop sections |
| `bundles` | dict | Bundle defs (`bun_a`…`bun_l`) |
| `quests` | dict | Quest defs by NPC name |
| `flagsReqs` | list[96] | Flag unlock requirements |
| `fishingKits` | dict | Fishing bait and line defs |
| `liquidsShop` | list[18] | Liquid shop items |

---

## `accountOptions` — Complete Index Reference

**Source:** `tryToParse(idleonData.OptLacc)` → `account.accountOptions[N]`  
**Named enum file:** `data/account-options-enum.js`

**General**
- `[33]` MINIGAME_PLAYS · `[37]` GUILD_DAILY_CHECK · `[38]` GUILD_ICON · `[39]` WEEKLY_RESET_SHOP
- `[55]` BASE_BOOK_COUNT · `[57]` GIANTS_SPAWNED · `[69]` STATUES_CHECK · `[71]` DUNGEON_PROGRESS
- `[89]` MAX_ARENA_LEVEL · `[99]` PEN_PALS · `[100]` COOKING_SPICES_CLAIMS · `[137]` RANDOM_EVENTS
- `[154]` GILDED_STAMPS · `[201]` SPIKE_MINIGAME_ROUNDS · `[310]` EVENT_SHOP_CURRENCY · `[311]` EVENT_SHOP_BONUSES

**Alchemy**
- `[106]` DRAGONIC_CAULDRON · `[123]` DRAGONIC_INDEX · `[133]` ATOM_COLLIDER_THRESHOLD
- `[135]` ALTERNATE_PARTICLES · `[383]` PRISMA_FRAGMENTS · `[384]` PRISMA_BUBBLES · `[409]` SIGIL_SYRUP

**Breeding / Arena**
- `[86]` BREEDING_MULTI · `[87]` TIME_TO_NEXT_EGG · `[89]` MAX_ARENA_LEVEL

**Sailing / Islands**
- `[124]` SAILING_TIME · `[125]` DAYS_SINCE_LAST_SAMPLE · `[160]` NUMBER_OF_DAYS_AFK
- `[161]` TRASH · `[162]` BOTTLES · `[163]` TRASH_UPGRADE_LEVEL · `[164]` BASE_BOTTLE_VALUE
- `[166]` RANDO_LOOT · `[167]` RANDO_DOUBLE_BOSS · `[169]` ISLANDS_KEYS · `[172]` BEST_DPS_EVER
- `[173]` SHIMMER_CURRENCY · `[179]` SHIMMER_BONUS · `[182]` SHIMMER_ISLAND · `[184]` FRACTAL_HOURS_AFK

**Weekly Bosses / Equinox**
- `[185]` WEEKLY_BOSSES_TURN · `[188]` WEEKLY_BOSSES_VALUE · `[189]` BOSS_BATTLE_TALENT
- `[190]` WEEKLY_BOSSES · `[192]` GAMING_NUGGETS_SINCE_UPGRADE · `[193]` DREAM_INDEX
- `[195]` GEMS_FROM_BOSSES · `[225]` DAYS_SINCE_MAGMUS · `[226]` DAYS_SINCE_SPIRITLORD · `[320]` EQUINOX_PENGUINS

**Owl (World 1)**
- `[253]` OWL_FEATHERS · `[254–261]` OWL_UPGRADE_0–7 · `[262]` OWL_MEGA_FEATHERS
- `[263]` OWL_PROGRESS · `[264]` OWL_SHINY_FEATHER

**Kangaroo (World 2)**
- `[267]` KANGAROO_FISH · `[268–279]` KANGAROO_UPGRADE_0–9 · `[280]` KANGAROO_PROGRESS
- `[281]` KANGAROO_MULTI_0 · `[289]` KANGAROO_SHINY_PROGRESS · `[291–295]` KANGAROO_RESET_BONUS_0–4
- `[296]` KANGAROO_TAR_FISH_OWNED · `[297]` KANGAROO_TAR_UPGRADE_0 · `[301]` TAR_UPGRADE_1

**Killroy (World 2)**
- `[105]` SKULLS · `[106–111]` KILLROY_LEVEL_0–5 · `[113]` KILLROY_GENERAL
- `[227]` UNLOCKED_THIRD_KILLROY · `[228–230]` KILLROY_ROOM_0–2 · `[467–471]` KILLROY_BONUS_3–7

**Sneaking (World 6)**
- `[231]` SELECTED_NINJA_MASTERY · `[232]` NINJA_MASTERY · `[233]` SNEAKING_BASE_VALUE
- `[402]` DAILY_CHARM_ROLL_COUNT

**Summoning (World 6):** `[319]` HIGHEST_ENDLESS_LEVEL

**Farming (World 6):** `[416]` EXOTIC_MARKET_UPGRADES_PURCHASED

**Hole (World 5):** `[318]` BROKEN_LAYERS_TODAY

**Spelunking (World 7):** `[410]` DAILY_PAGE_READS · `[478]` SPELUNKING_OPTION

**Research/Construction (World 7):** `[414]` JEWELED_COGS_PULLS

**Legend Talents (World 7):** `[480]` CHEAPER_MASTERCLASS_UPGRADES_USED

**Compass (class-specific)**
- `[357–362]` Dust types (Stardust, Moondust, Solardust, Cooldust, Novadust, Walker)
- `[365]` COMPASS_KILLS_LEFT · `[366]` COMPASS_UPGRADE

**Tesseract (class-specific)**
- `[388–393]` Tachyon variants · `[395]` TESSERACT_DROP_CHANCE
- `[396]` TESSERACT_WEAPON_DROP · `[397]` TESSERACT_RING_DROP

**Coral Reef / Divinity:** `[425]` CORAL_KID_LINKED · `[427]` CORAL_REEF_LEVEL · `[486]` STATUES_CLUSTERS

**Highscores:** `[99]` PEN_PALS_SCORE · `[242]` HOOPS_SCORE · `[442]` DARTS_SCORE

---

## Python Character Builder

```python
import re, json

def try_to_parse(val):
    if isinstance(val, str):
        try: return json.loads(val)
        except: return val
    return val

def sparse_to_list(obj):
    if isinstance(obj, dict) and 'length' in obj:
        return [obj.get(str(i)) for i in range(int(obj['length']))]
    return obj

def create_array_of_arrays(arr):
    return [sparse_to_list(x) for x in (arr if isinstance(arr, list) else list(arr.values()))]

def get_characters(idleon_data, char_names):
    characters = []
    for play_id, char_name in enumerate(char_names):
        char = {'name': char_name, 'playerId': play_id}
        for key, val in idleon_data.items():
            if not re.search(rf'_{play_id}$', key):
                continue
            parsed = try_to_parse(val)
            if 'EquipOrder' in key:
                char['EquipmentOrder'] = create_array_of_arrays(val)
            elif 'EquipQTY' in key:
                char['EquipmentQuantity'] = create_array_of_arrays(val)
            elif 'AnvilPA_' in key:
                char['AnvilPA'] = create_array_of_arrays(val)
            elif 'SL_' in key:
                char['SkillLevels'] = parsed
            elif 'SM_' in key:
                char['SkillLevelsMAX'] = parsed
            elif 'PVStatList' in key:
                char.setdefault('PersonalValuesMap', {})['StatList'] = parsed
            elif 'PVtStarSign' in key:
                char.setdefault('PersonalValuesMap', {})['StarSign'] = parsed
            elif 'POu_' in key:
                char['PostOfficeInfo'] = parsed
            elif 'PTimeAway' in key:
                char['PlayerAwayTime'] = (parsed or 0) * 1000
            elif 'IMm_' in key:
                char['InventoryMap'] = parsed
            elif 'BuffsActive' in key:
                char['BuffsActive'] = create_array_of_arrays(val)
            elif 'ItemQTY' in key:
                char['ItemQuantity'] = parsed
            elif 'ObolEqO0' in key:
                char['ObolEquippedOrder'] = parsed
            elif 'ObolEqMAP' in key:
                char['ObolEquippedMap'] = parsed
            elif 'KLA_' in key:
                char['KillsLeft2Advance'] = parsed
            elif 'AtkCD_' in key:
                char['AttackCooldowns'] = parsed
            else:
                clean_key = key.rsplit('_', 1)[0]
                char[clean_key] = parsed
        characters.append(char)
    return characters

# Usage
account_options = try_to_parse(idleon_data.get('OptLacc', '[]'))
char_level = characters[0]['PersonalValuesMap']['StatList'][4]
skill_levels = sparse_to_list(characters[0]['SkillLevels'])
mining_level = skill_levels[1]
```

---

## Dashboard Alert System

**Files:** `utility/dashboard/account.js`, `utility/dashboard/characters.js`

Each alert is a function `(account, characters, options) → null | alertObject`.

- `account.js` — account-wide alerts: refinery full, alchemy sigils ready, construction done, etc.
- `characters.js` — per-character alerts: anvil full, traps ready, worship charged, etc.

Alert config is versioned. When the schema changes, a migration function in `utility/migrations.js` updates stored configs. The version is tracked in `baseTrackers` in `pages/dashboard.jsx`.

---

## Navigation Structure

**File:** `components/constants.jsx`

```
navItems: dashboard, characters, account, tools, guilds, statistics, leaderboards
drawerPages: characters, account, tools  (have sub-navigation drawers)
offlinePages: tools, statistics, leaderboards  (no login needed)
```

`PAGES` object defines the full sidebar hierarchy: world → category → tabs → nestedTabs.

---

## Key Python Automation Patterns

1. **Every Firebase string field needs `try_to_parse()`** before use
2. **Sparse arrays** `{"0":v, "length":N}` → `[obj.get(str(i)) for i in range(N)]`
3. **Per-character fields** are `KEY_N` in Firebase — strip `_N` and apply renaming table above
4. **All bonus calculations** use `growth(func, level, x1, x2)` with data from `website-data.json`
5. **`accountOptions[N]`** from `OptLacc` is the main boolean/counter flag store
6. **`LavaRand(seed).random(max)`** for any deterministic game randomness
7. **Parser execution order matters** — alchemy must exist before stamps, lab before vials, etc.
8. **`website-data.json`** is the static game data bible — load it once, reference everywhere
