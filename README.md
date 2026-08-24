# NBA 2K27 MyPLAYER Builder — Rules Dataset

Measured game data behind NBA 2K27's MyPLAYER builder: attribute ceilings,
overall-rating pricing, badge requirements and token economics, cap-breaker
gains, takeover predicates, and the full animation glossary.

These are not community estimates or reverse-engineered approximations. Almost
every number here was produced by calling the game's own rules engine directly
and recording what it returned.

## This dataset describes one exact game build

| | |
|---|---|
| Game | NBA 2K27 |
| Source app | NBA 2K HQ (`com.VisualConcepts.MyNBASherpa`) |
| `api_version` | `202750199` |
| `live_tuning_version` | `993560759169487438` |
| Captured | 2026-08-22 |

Every file repeats this in its `_meta.build` block. **2K patches tuning
server-side**, so any of these values can change without a client update. Check
the version before trusting a number against a live game.

## Layout

```
reference/    what the integer codes mean — read this first
bodies/       legal positions, heights, weights, wingspans; attribute ceilings
overall/      overall-rating (OVR) pricing measurements
badges/       badge definitions, tier requirements, token costs, slot allocation
cap_breakers/ how much each cap breaker raises an attribute
takeovers/    takeover attribute predicates
animations/   the official animation glossary
tuning/       the shipped tuning tables the engine reads
```

Every `.json` file has the same shape, so one loader handles all of them:

```python
import json
with open("badges/token_costs.json") as f:
    doc = json.load(f)
doc["_meta"]     # provenance, field descriptions, caveats
doc["data"]      # list of records
```

`_meta` is worth reading per file — it carries the field semantics and the
caveats specific to that measurement.

---

## Background: what the numbers describe

Enough context that the values mean something. In NBA 2K's career mode you build
a **MyPLAYER** by picking a position and a body — height, weight, wingspan — then
distributing a budget across 21 attributes rated 25–99 (25 is the starting value
for everything).

- **Attribute ceilings.** Your body decides how high each attribute can go. A
  7-foot centre physically cannot reach elite Speed With Ball; a 6-foot guard
  cannot reach elite Standing Dunk. This trade-off is the core of the system.
- **Overall rating (OVR).** A weighted price for your attribute spread. The
  engine prices the build against **15 archetypes** ("player types") and keeps
  the *highest* result, so your OVR is set by whichever archetype your spread
  most resembles. Weights are height-specific, and the per-rating scale is
  non-linear — raising a key attribute 94→95 costs far more OVR than 60→61.
  Builds are created up to a 99 OVR ceiling.
- **Cap breakers.** Post-creation rewards that raise an attribute's ceiling.
  Each attribute accepts at most **five**, they are permanent, and the gain is
  much larger on low-rated attributes — a 25 can jump by double digits while a 95
  moves by one. They push an attribute towards its body-derived maximum but never
  past it.
- **Badges.** 53 in 2K27 (up from 40 in 2K26: 19 added, 6 removed). Each has
  attribute thresholds across four builder tiers — **bronze, silver, gold, hall
  of fame** — plus a fifth **legend** tier that cannot be reached in the builder
  at all and is only unlocked in-game through the Synergy system. New for 2K27,
  you **spend badge tokens** to equip badges: every build gets 20 slots split
  across six disciplines, and a badge's token cost changes with your height.
- **Disciplines.** Six groups used for both attributes and badges: finishing,
  shooting, playmaking, defense, rebounding, physicals.
- **Takeovers.** Temporary in-game boosts gated behind attribute thresholds. 2K
  publicly describes 24 takeover abilities across 5 slots; the engine returned
  requirement records for 29, so the extra 5 are presumably unshipped or
  internal. The dataset publishes what the engine returned.
- **Animations.** Jumpshots, dunks, dribble moves — gated by attribute minimums
  and by **size family** (`SMALLS` / `SWINGS` / `BIGS`).

Public reference: 2K's [MyPLAYER Builder page](https://nba.2k.com/2k27/features/myplayer-builder/)
and the 18 August 2026 Courtside Report.

---

## Reading conventions

**Units are normalised.** All lengths are **inches**, all weights are **pounds**.
The engine reports lengths in centimetres; conversion was exact (`cm = in × 2.54`
for every value in the dataset) and applied at build time.

**Six-element arrays** named `tokens`, `slots`, `minimums`, `maximums` are always
in `discipline_order`:

```
[finishing, shooting, playmaking, defense, rebounding, physicals]
```

**`attribute` integers** index `reference/attributes.json` (0–20), and records
carry a `name` alongside the index so files are readable without a join. The one
exception is `takeovers/requirements.json`, which uses a different, unresolved
enumeration — see its warning.

**Reference body.** Measurements that depend on a specific build were taken at
**PG, 6'3" (75"), 198 lb, 78" wingspan**, and say so in `_meta.reference_body`.
The tuning tables generalise to every body; these probe samples do not.

---

## Files

### `reference/`

| File | Records | Contents |
|---|---|---|
| `attributes.json` | 21 | The 21 attributes in probe index order, with discipline and UI colour. **The key to every `attribute` field in the dataset.** |
| `enums.json` | 75 | Every other integer code: positions, disciplines, badge tiers, all 53 badge IDs → names, and `discipline_order`. |
| `native_symbols.txt` | — | The builder functions the app's native library exports. This is the list of entry points the capture called, and the starting point for extending it. |
| `ui_presentation.json` | 3 | Discipline colours and max badge tier. Cosmetic, not rules. |

### `bodies/`

| File | Records | Contents |
|---|---|---|
| `legal_bodies.json` | 5 | Per position: legal height range, default height, and for each height the legal weight range, wingspan range, and defaults. Wingspan is always `height … height+6`. |
| `attribute_caps_sample.json` | 21 | The 21 attribute ceilings at the reference body. A validation sample — compute ceilings for arbitrary bodies from `tuning/`. |

### `overall/`

| File | Records | Contents |
|---|---|---|
| `uniform_ratings.json` | 75 | OVR with all 21 attributes set to the same value, 25→99. |
| `mixed_vectors.json` | 256 | OVR for 256 pseudo-random full attribute vectors, with `detailed`, `best` and `uncapped` values and each one's winning archetype. **The parity set for any reimplementation of the pricing function.** |
| `official_ui_verified.json` | 2 | Two vectors reproduced in the signed-in official builder UI, capturing the 98→99 completion edge. The only records here corroborated outside the engine. |
| `single_attribute.json` | 1,328 | One attribute raised, the other 20 left at floor. ⚠️ **Semantics unverified — not a cost curve.** See below. |

`uniform_ratings.json` and `mixed_vectors.json` are the trustworthy pricing
records: both reproduce exactly from `tuning/`. `best` and `detailed` are two
different native routines and they diverge — from rating 84 up, `best` saturates
at `99.0` and reports archetype 0 while `detailed` reports `98.999992` with the
true winning archetype.

⚠️ **`single_attribute.json` is retained but not safe to build on.** It is the
one file in `overall/` that could *not* be reproduced from the tuning tables —
only its 21 all-floor rows match a reimplementation. 14 of the 21 attributes
report exactly `25.0` at every rating, and the 7 that move only move in the top
few points of their range (`three_point` first moves at 88, `close_shot` at 94),
which is what saturation against the 25 floor looks like. The routine was
evidently called in a configuration that was never pinned down. It is kept as a
raw measurement for anyone who resolves it. **If you want a per-attribute cost
curve, derive it from `tuning/`** — that math is confirmed on all 256 mixed
vectors.

Its sparsity is meaningful, though, and doubles as a cross-check: each attribute
was swept from 25 only as far as its *ceiling* at the reference body, so the row
count is `Σ(ceiling − 24) = 1,328`, matching
`bodies/attribute_caps_sample.json` exactly.

### `badges/`

| File | Records | Contents |
|---|---|---|
| `definitions.json` | 53 | Discipline, native group code, and height eligibility per badge. A `63–91` range means unrestricted. |
| `tier_requirements.json` | 212 | Attribute requirements per badge per creation tier — 53 × 4. Requirements are an ordered predicate list combined by each entry's `operator_to_next`. |
| `token_costs.json` | 5,300 | Token price per badge, per tier, per height — 53 × 5 × 20. All 1,060 **legend** rows cost `0` — legend cannot be equipped at creation, so the builder has no price for it. Read `0` at legend as "not purchasable here", not "free". |
| `token_contributions.json` | 31,500 | Tokens earned from one attribute at one rating at one height — 21 × 75 × 20. **The token function is additive across attributes**, so summing a build's 21 rows gives its exact token budget. Verified against 2,048 native vectors. |
| `slot_allocations.json` | 2,123 | Ground-truth slot allocation for full vectors (2,048 random across 10 heights + 75 uniform). Slot allocation is *not* additive, so it cannot be derived from the contribution table — these records are what pin the allocator down. |
| `slot_allocator_constants.json` | 8 | Allocator inputs: 20 total slots, blend factor, per-discipline minimums/maximums, tie-break orders, and the default allocation. |

⚠️ **The allocator formula is unresolved.** The plausible reading of the
constants — blend each discipline's share of token potential (weight `blend`)
with its share of eligible badges (weight `1 − blend`), scale by `total_slots`,
clamp to the per-discipline limits, then distribute rounding remainders by the
tie-break orders — **does not reproduce the ground truth**, and two specific
things rule it out:

- **The stated `maximums` are not caps.** `maximums` is `[7,7,7,7,5,6]`, but the
  2,123 recorded allocations reach 11 finishing, 8 shooting, 20 playmaking, 11
  defense and 9 rebounding. Whatever `maximums` governs, it is not a ceiling on
  the allocation.
- **No blend value fits.** Sweeping `blend` from 0 to 1 with largest-remainder
  rounding tops out at 569/2,123 exact (27%), and the best-fitting value is
  ≈0.075, not the shipped `0.4`.

What *is* established: the allocation is a **deterministic function of
(per-discipline token totals, per-discipline count of badges the build qualifies
for at any tier)**. Across the 2,084 distinct feature keys in the ground truth,
no key maps to two different allocations, and adding height as a feature changes
nothing. So the inputs are identified and the problem is well-posed — only the
combining rule is missing. Until it is recovered, use
`badges/slot_allocations.json` as a lookup and interpolation base rather than
trusting any formula, including the one above.

One shipped quirk worth knowing: `unpluckable` at hall of fame requires
`post_control ≥ 100`, above the 99 maximum. It sits on an `OR` branch, so the
effect is that the branch is dead and the other clause is the real requirement.
That is genuine data, not a transcription error.

### `cap_breakers/`

| File | Records | Contents |
|---|---|---|
| `gains_by_rating.json` | 13,280 | Rating points gained per cap-breaker application, for every attribute, every starting rating, and each of the five permitted applications. |

Two scenarios, because the gain depends on the *whole* build via its winning
archetype: `isolated` (all other attributes at floor) and `near_caps` (all others
at their ceiling). Sparse by design — 13,280 of a possible 15,750 rows;
combinations with no headroom left are absent.

### `takeovers/`

| File | Records | Contents |
|---|---|---|
| `requirements.json` | 29 | Attribute predicates per takeover ability. |

⚠️ **Unresolved enumeration.** The `attribute_code_unresolved` values here are
**not** the 0–20 indices used elsewhere; observed codes reach 26, so these
abilities index a larger `PlayerData`-style enum. The mapping was never
confirmed, so codes are published raw. A partial contiguous run was recovered
from app metadata but Block, Speed and Vertical are missing from it. Also, 11 of
the 29 abilities return zero requirements, which may mean "unconditional" or may
mean "sentinel value" — never disambiguated. **This is the least trustworthy part
of the dataset.**

### `animations/`

| File | Entries | Contents |
|---|---|---|
| `glossary.json` | 2,914 | The official animation glossary, verbatim, in 10 languages. |

Structure: `Anim Glossary Tabs` → 3 tabs (Scoring Moves, Dunks, Playmaking
Moves) → `Anim Groups` (12 / 12 / 32) → `Anims`. Per animation: `Anim ID`,
`Anim Name`, `Allowed Sizes`, `Attrib Reqs` (`Attrib Type` + `Attrib Min`),
`Attrib Reqs Operator` (`AND`/`OR`), `Is Prized`, `Is Season Specific`,
`Season Start`, `Season End`.

`Allowed Sizes` ∈ `ANY`, `SMALLS_ONLY`, `SWINGS_ONLY`, `BIGS_ONLY`,
`SWINGS_AND_SMALLS`, `BIGS_AND_SWINGS`. Note requirement attributes here use the
*internal* names (`ShotThree`, `BallControl`, …), not the numeric indices used by
the rest of the dataset.

The height boundaries that decide which size family a build belongs to are **not
in this dataset** — they were never recovered, and are deliberately not guessed.

### `tuning/`

| File | Contents |
|---|---|
| `progression_attributes.txt` | The **named** key/value export of the career-mode progression tuning — the only artifact that puts names on the builder's numbers. 16,114 lines: a 16-line provenance banner followed by the 16,098-line export, unmodified. `//` marks comments. |
| `progression_tuning.zlib` | The tuning blob exactly as the app ships it, sealed and unmodified. |

`progression_tuning.zlib` is a 16-byte header — ASCII magic `ZLIB`, decompressed
length as big-endian `uint32` (`0x003A98C2` = 3,840,194), compressed length, then
a version word — followed by a raw zlib stream:

```python
import zlib
raw = open("tuning/progression_tuning.zlib", "rb").read()
blob = zlib.decompress(raw[16:])        # 3,840,194 bytes
```

That blob is **keyless**: it addresses values by offset and contains zero
occurrences of strings like `PlayerRestrictions`. It is included as the sealed
upstream original, not as usable data — the named export is what you want, and
the probe measurements are what pin the values to behaviour.

Key families in `progression_attributes.txt`:

| Key family | Rows | Holds |
|---|---|---|
| `HeightBasedAttributeWeight[H][PLAYERTYPE][ATTR]` | 6,271 | per-archetype OVR weights |
| `AssociatedAttributeConstraints[...]` | 3,091 | linked-attribute minimums (`AssociatedAttribute` + `MaxDelta`) |
| `PlayerRestrictions[NBA].WeightMultiplier[...]` | 2,018 | weight → ceiling multipliers |
| `PlayerRestrictions[NBA].WingspanMultiplier[...]` | 1,934 | wingspan → ceiling multipliers |
| `PlayerRestrictions[NBA].HeightMultiplier[H][ATTR]` | 805 | height → ceiling multipliers |
| `AttributeRatingWeightScale[ATTR][rating]` | 551 | the non-linear per-rating scale |
| `HeightBasedOverallLerp[H].Value[i][j]` | 124 | final raw → OVR interpolation endpoints |
| `PlayerRestrictions[NBA].MinMax*`, `Min/MaxWeight`, `Default*` | 64 | legal body ranges per position |

**The attribute ceiling formula**, which reproduces all 21 measured ceilings in
`bodies/attribute_caps_sample.json` exactly:

```
ceiling = clamp(round(25 + 74 × height_mult × weight_mult × wingspan_mult), 25, 99)
```

Weight and wingspan multipliers interpolate linearly between the two endpoints
shipped for each whole-inch height. Position restricts which bodies are legal but
does **not** enter the equation.

**Linked attributes.** `AssociatedAttributeConstraints` encodes automatic
minimums: raising a source attribute forces each associated attribute to at least
`source − MaxDelta`. These cascade, since a forced raise can itself be a source.
The rules are height-specific.

---

## Verification

The dataset contains two independent kinds of artifact: the **named tuning
tables** the engine reads, and the **probe measurements** of what the engine
returned. That makes it self-checking — the math can be rebuilt from the tuning
tables alone and held against the measurements, with neither side informing the
other. Doing that gives:

| Check | Result |
|---|---|
| 21 attribute ceilings recomputed from `tuning/` | 21 / 21 exact |
| `mixed_vectors` `detailed` recomputed from `tuning/` | 256 / 256 to 1e-4 |
| `mixed_vectors` winning archetype recomputed | 256 / 256 |
| `mixed_vectors` `best` and `uncapped` recomputed | 256 / 256 |
| `uniform_ratings` recomputed from `tuning/` | 75 / 75 |
| `single_attribute` recomputed from `tuning/` | **21 / 1,328 — see the warning above** |
| Token additivity: 2,048 vectors summed from the contribution table | consistent |
| Slot allocations summing to their declared totals | 2,123 / 2,123 |
| Cap-breaker gains non-increasing across the 5 applications | no violations |
| Cap-breaker gains never exceeding the body's ceiling | no violations |
| Badge tiers monotonically non-decreasing across all 53 badges | no violations |
| `single_attribute` row count vs `Σ(ceiling − 24)` | 1,328 = 1,328 |
| Every `attribute` index, badge ID and discipline resolving via `reference/` | complete |
| One build identity declared across every file | consistent |

Two of those deserve emphasis. The **256-vector agreement** is the strongest
result in the dataset: reimplementing the pricing function from the tuning
tables reproduces every measured float, including the non-linear rating scales
and archetype selection, so the tuning export and the probe corroborate each
other. And the **tier monotonicity** result matters because tier alignment was
the one non-trivial correction applied to the source data — a misaligned tier
would show up as a higher tier demanding a *lower* attribute minimum, and no
badge does that.

`overall/single_attribute.json` is the one measurement that failed to
corroborate. It is kept, clearly flagged, and excluded from the strong-confidence
list below.

Per-file results are recorded in each `_meta.verification`.

## Caveats

Read these before using the numbers.

1. **Overall accumulation is `float32`.** The engine accumulates in IEEE-754
   binary32. Reproducing the float values to the last decimal requires rounding
   intermediates to single precision; double-precision arithmetic drifts.
2. **`cap_breakers` and `bodies/attribute_caps_sample` are single-body.** Taken
   at the reference body only. The tuning tables generalise; these do not.
3. **`player_type` 1 never appears** as a winner in the 256 mixed vectors. There
   are 15 archetypes (0–14) and 14 were observed winning; absence is not evidence
   the archetype does not exist.
4. **Takeover attribute codes are unresolved.** See above.
5. **Badge display names** are high-confidence but not exhaustively verified.
   Badge *IDs* are authoritative. 2K publicly named only three of 2K27's new
   badges — pace, wall_up, flash — and all three match, which is good but partial
   corroboration.
6. **Animation size-family height boundaries are absent** by choice.
7. **`overall/single_attribute.json` has unverified semantics.** The only
   measurement in the dataset that does not reproduce from the tuning tables.
8. **Single capture.** Everything comes from one device capture of one game
   build. The cross-check described above is genuinely independent — tuning
   tables versus recorded engine output — but both come from the same shipped
   build, so a bug in the game itself would be reproduced faithfully rather than
   caught. Only 2 records were checked against the official UI.

### Confidence summary

**Strong**, and independently reproduced from the tuning tables — the attribute
ceiling formula (21/21 ceilings) and the OVR pricing function including archetype
selection and the non-linear rating scales (256/256 vectors, 75/75 uniform).

**Strong**, from measurement with internal invariants holding — badge tier
requirements (monotonic across all 53), token costs, token contributions
(additivity confirmed on 2,048 vectors), the 2,123 recorded slot allocations
themselves, cap-breaker gain sequences (13,280 rows, monotonic and
ceiling-respecting), legal bodies, linked-attribute minimums.

**Weak** — the slot-allocator *formula* (inputs identified, combining rule
unresolved; the recorded allocations are still solid); takeover enum mapping; the
5 extra takeover abilities beyond 2K's published 24; badge and takeover display
names; animation size-family boundaries; `overall/single_attribute.json`.

---

## How the data was captured

Three routes, in descending order of authority.

### 1. Calling the native rules engine — most of the dataset

The reason this dataset is measurements rather than estimates.

The tuning blob the app ships is keyless, so reading floats out of it tells you
nothing about which float means what. Rather than guess at offsets, the app's own
engine was asked directly.

NBA 2K HQ ships its game logic in a native library, `libcsharplib_mobile`, which
**exports its builder functions as plain C symbols** — `ATTRIBUTES_CalculateBestOverallRating`,
`BADGES_GetBadgeTokenPotential`, `BADGES_GetNumDefaultSlotsUnlocked`,
`ATTRIBUTES_GetAssociatedConstrainedAttributes`, and others. The full symbol list
is in `reference/native_symbols.txt`.

The method:

1. Pull the APK and its native library off a device or emulator running the app.
2. List exported symbols (`nm -D`, `readelf --dyn-syms`, or `objdump -T`) and pick
   out the `ATTRIBUTES_*`, `BADGES_*` and takeover entry points.
3. Write a small C harness that `dlopen`s the library, resolves those symbols with
   `dlsym`, and calls them. Cross-compile it for `aarch64` with the Android NDK.
4. Push the harness to the device and run it **in-process with the live tuning
   blob loaded**, so the engine answers using real shipped data.
5. Sweep exhaustive input grids — every legal height, every attribute, every
   rating 25–99 — and dump each answer as one record.

That last step is what produced the record counts here: 21 × 75 × 20 = 31,500
token contributions, 53 × 5 × 20 = 5,300 token costs, and so on.

Requires a rooted device or a rooted emulator image, since loading the library
out-of-process needs to read the app's private files.

### 2. The shipped tuning export

`tuning/progression_attributes.txt` is a named key/value export of the same
career-mode tuning the engine reads. It supplies the multiplier tables,
per-archetype weights, non-linear rating scales, and linked-attribute rules under
readable keys — which is what makes the packed values legible. It is a separate
artifact from the binary blob: the blob carries values, this carries names.

### 3. Binary analysis

Symbol names and UI presentation constants (`reference/native_symbols.txt`,
`reference/ui_presentation.json`), read out of the native library. Lowest
authority, and labelled as such.

### Pulling the app's own files

The companion app writes its constants to external storage, readable without root
once you sign in and open the MyPLAYER builder so it fetches them:

```bash
PKG=com.VisualConcepts.MyNBASherpa
adb shell "find /sdcard/Android/data/$PKG/files -type f -iname '*.json'"
adb pull /sdcard/Android/data/$PKG/files/<file> .
```

`animations/glossary.json` came from there.

⚠️ **That directory also contains account files. Do not publish them.** In this
capture, `UserAccountData.json` held a live auth token, session key, account IDs
and a date of birth, and `Request2KConstants.json` held signed URLs for 2K
production services including a bearer-style token. Neither contains any rule
data. They are excluded from this dataset — delete them from your pull directory
before committing anything, and clear any saved cookies from the session.

---

## Extending the dataset

Worthwhile gaps, roughly in order of value:

- **Resolve the takeover attribute enum.** The highest-value fix. Recover the
  `CAREERMODE_ATTRIBUTE_TYPE` (or `_WITH_NONE`) enum from the binary's metadata
  and map the codes in `takeovers/requirements.json`.
- **Recover animation size-family boundaries** — the height cutoffs separating
  `SMALLS` / `SWINGS` / `BIGS`. Without them, animation eligibility can only be
  evaluated if the family is supplied explicitly.
- **Sweep cap breakers across more bodies.** Currently one reference body.
- **Verify more against the official UI.** Only 2 records are corroborated
  outside the engine; more would break the circularity noted in the caveats.
- **Recover badge and takeover display names** from app resources to replace the
  high-confidence name list with a verified one.

When 2K patches, re-pull the tuning and re-run the sweeps rather than editing
values by hand, and record the new `api_version` and `live_tuning_version`.

---

## Legal

Not affiliated with 2K Sports, Visual Concepts, or Take-Two Interactive. All game
data belongs to its owners. This is a personal research archive documenting the
observed behaviour of software the author had licensed access to, published for
interoperability and educational purposes. No game code, assets, or copyrighted
binaries are included — only measurements and the numeric tuning values needed to
interpret them.
