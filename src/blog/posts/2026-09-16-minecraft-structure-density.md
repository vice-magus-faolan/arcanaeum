---
title: "Tuning Minecraft Structure Density Instead of Guessing at Spawn Rates"
date: 2026-09-16
author: vice-magus-faolan
tags:
  - minecraft
  - worldgen
  - modded-minecraft
  - server-admin
  - troubleshooting
excerpt: |
  Structure spacing is a candidate-grid setting, not a promise that a landmark will appear every certain number of blocks. Here is how I read Moog’s spacing multipliers, calculate a concrete example, and tune MSS and MOS without mistaking grid density for successful generation.
featured: false
draft: true
---

# Tuning Minecraft Structure Density Instead of Guessing at Spawn Rates

*September 16, 2026 — retrospective.*

Minecraft structure settings invite a tempting kind of math: find a spacing number, multiply it by a config value, and announce that players will find a castle every few kilometers. The arithmetic can be right while the prediction is wrong. A spacing value describes a candidate placement grid; it is not a travel-distance guarantee, a biome acceptance rate, or a promise that a structure will successfully generate.

That distinction matters when tuning Moog’s structure mods. I wanted settings that make exploration feel deliberate without turning every trip into a vacant scenic tour—or filling the map with landmarks. The useful question is not “what are the spawn rates?” but “what candidate grid does this configuration ask the game to consider, and what do we still not know after calculating it?”

## Spacing is not frequency

In Moog’s Structure Lib (MSL) 3.2.0, an advanced random-spread placement starts with the structure set’s declared `spacing` and `separation`. The placement code first scales both values by a built-in factor of 1.65 and rounds them to integers. It then applies the configured spacing multiplier and rounds again. The resulting spacing and separation are used to calculate candidate positions on a random-spread grid.[12]

So a config multiplier of `2.0` does not simply mean “half as many structures.” It roughly doubles the grid’s side length for affected sets. Since a square cell’s area depends on both dimensions, the nominal candidate-cell area becomes about four times as large. Conversely, changing a multiplier from `2.0` to `1.0` makes the side length about half as large and the nominal cell area about one quarter. That is a useful density comparison, but it is about candidate grids—not completed structures or how often a player encounters one.

The rounding steps also matter. You cannot always reproduce the result by multiplying the raw spacing by `1.65 × config multiplier` and rounding once. Small fractions can land on different integers at the two stages; Java’s positive `Math.round` behavior rounds by taking the floor after adding 0.5. For a rough mental model, the combined multiplier is fine. For an exact example, follow the actual order.

## A worked example, not a spawn-rate promise

Take the MSS `cherry_river` structure set. Its pinned definition declares `spacing: 53`.[26] With the final per-mod setting of `mss: 2.0` and the universal multiplier at `1.0`, here is the calculation:

1. Built-in pass: `53 × 1.65 = 87.45`, rounded to `87`.
2. Config pass: `87 × 2.0 = 174` chunks.
3. Nominal grid side: `174` chunks, or `174 × 16 = 2,784` blocks.
4. Nominal square area: `2.784² = 7.750656 km²` when using 1,000 blocks as one kilometer.[12][26]

That last number is an area ratio for one nominal grid cell, not “one successfully generated cherry river structure per 7.75 square kilometers.” It also does not mean every point is exactly 2,784 blocks from the next candidate. The grid is randomized, and the candidate is only one stage in placement. To avoid a false sense of precision, I would use this calculation to compare settings, not to promise a nearest-structure distance or a walking encounter rate.

## What can prevent a candidate from becoming a landmark?

A candidate position is not the same thing as a structure a player can find. The placement code can apply frequency reduction, exclusion zones, a minimum distance from world origin, and other checks.[12] After placement eligibility, the structure's terrain and biome requirements still matter. Even the two examples here differ: the arena definition has a 1,000-block minimum distance from the origin, while `cherry_river` specifies 250 blocks.[26][27]

Those conditions make simplistic aggregate estimates especially fragile. Summing the inverse areas of 35 candidate grids would not produce a reliable “one structure every X blocks” prediction: sets have different spacing, conditions, and geography, and grid candidates can fail later checks. A player’s route is not a uniform sample of map area either. Someone following a coastline, crossing a mountain range, or revisiting explored chunks experiences a different encounter pattern from a theoretical scan across fresh terrain.

It is also worth keeping “35 sets” in perspective. The MSS data contains 35 separately defined structure sets.[13] That is a count of placement definitions, not a guarantee of 35 equally common designs in every biome or region. A rare particular design can coexist with a better chance of encountering some member of the collection. But the definitions alone do not justify a precise probability for either outcome.

## Tune the collection by mod first

The config supports three layers: a universal multiplier, per-mod multipliers keyed by namespace, and optional per-structure multipliers. The effective value is their product. Defaults are `1.0`, so if you set only a per-mod value, it scales that mod’s placements without changing unrelated structure mods.[11]

For a baseline that keeps the two installed Moog collections distinct, the spacing section looks like this. Merge this section into the existing configuration; it is not a replacement for unrelated settings such as presets:

```json
{
  "spacing": {
    "universal_multiplier": 1.0,
    "per_mod": {
      "mss": 2.0,
      "mos": 3.0
    },
    "per_structure": {}
  }
}
```

Here MSS means Moog’s Soaring Structures and MOS means Moog’s Ocean Structures. The values are intentionally different: `mss: 2.0` widens its grids relative to that collection’s own baseline, while `mos: 3.0` applies a larger multiplier to the ocean collection’s own baseline. Different raw spacings mean the multipliers alone cannot tell us which collection is absolutely denser. This is a starting policy, not evidence that islands or ocean landmarks have a particular real-world encounter rate. The code combines the namespace multiplier with universal and exact structure-set values when calculating the effective placement scale.[11][12]

I prefer this per-mod refinement to turning a universal dial when the tuning question is specifically about MSS versus MOS. If later evidence points to one set that is too common or too scarce, `per_structure` can target that ID without moving every set in the same collection. Keep the universal value at `1.0` unless the goal really is to affect all applicable mods; otherwise, a global value can quietly compound with mod-specific values.[11]

## A practical tuning loop

Start from a known config and change one scope at a time. Record the old and new multipliers, and compare their square-area ratios rather than translating them into exact counts per journey. A change from `2.0` to `2.5`, for example, makes the nominal side length 1.25 times larger and cell area about 1.5625 times larger before rounding. That is a clean comparison of grid scale, not an assertion that players will see 36 percent fewer structures.

Then test in a disposable world or an appropriate controlled environment: inspect generated chunks across more than one seed and biome, and distinguish placement candidates from visible, successfully generated structures. For a live world, spacing edits should not be treated as retroactive remodeling. The config documentation says spacing changes apply on world reload and affect newly generated chunks; already generated chunks are not a clean test of the new setting.[11] I have not run a new world-generation study for this draft, so I am not presenting sampled acceptance rates, screenshots, or a measured performance change.

If the complaint is “we rarely see any Moog structure,” do not immediately lower every multiplier. First ask whether the explored area included relevant biomes and fresh chunks, whether the specific design has an origin restriction, and whether the expectation is about one design or the whole collection. If the complaint is “the map feels crowded,” likewise identify which collection or set is responsible before raising a global multiplier. That makes the next change interpretable rather than turning config tuning into a guessing contest with nicer JSON formatting.

## The honest answer to “how often?”

A spacing multiplier lets me compare nominal candidate-grid scale. It does not let me promise a structure every kilometer, assign an exact percent chance within a radius, or infer a player’s route encounter rate. For the `cherry_river` example at `2.0`, we can show the two rounding steps and a 174-chunk nominal grid spacing; we cannot honestly turn that alone into a guaranteed successfully generated structure at a fixed distance.

That is still useful. It tells us which direction a setting moves density, how universal and per-mod values combine, and how to make a controlled adjustment. I would rather say “this makes that set’s nominal candidate cells about four times the area of the same setting at 1.0” than tell players to expect a landmark every so-many blocks. One statement is traceable to the placement code. The other would need actual world-generation and exploration measurements we do not have.

## Sources

[11] https://raw.githubusercontent.com/FinnSetchell/MoogsStructureLib/82ae45f0bd45d28cfba20104e3706b1f805f0134/common/src/main/java/com/finndog/moogs_structures/config/MslConfig.java — Moog’s Structure Lib 3.2.0 configuration
[12] https://raw.githubusercontent.com/FinnSetchell/MoogsStructureLib/82ae45f0bd45d28cfba20104e3706b1f805f0134/common/src/main/java/com/finndog/moogs_structures/world/structures/placements/AdvancedRandomSpread.java — Moog’s Structure Lib 3.2.0 placement implementation
[13] https://api.github.com/repos/FinnSetchell/MoogsSoaringStructures/git/trees/8b92537772289ec26c866ca5aee8489fa0f9ace5?recursive=1 — MSS 2.1.2 pinned structure-set definitions
[26] https://raw.githubusercontent.com/FinnSetchell/MoogsSoaringStructures/8b92537772289ec26c866ca5aee8489fa0f9ace5/src/main/resources/data/mss/worldgen/structure_set/cherry_river.json — MSS 2.1.2 cherry_river placement definition
[27] https://raw.githubusercontent.com/FinnSetchell/MoogsSoaringStructures/8b92537772289ec26c866ca5aee8489fa0f9ace5/src/main/resources/data/mss/worldgen/structure_set/arena.json — MSS 2.1.2 arena placement definition
