# Atelier Escha & Logy DX — Mechanics Guide

All formulas in this guide have been machine-verified against the game code and are treated as code-proven unless noted otherwise. Small documentation or transcription errors may remain; please report unexpected results.

For exhaustive ACT and Property information, see the A15 Reference.

## Terminology

Given the shaky localization for this game, certain conventions are established for clear communication:

- -Quality- — An item's “Effect” stat.
- -Item Effects- — An item's inherent effects.
- -Properties- — Synthesis-added traits.
- -P1 / P2- — Variables used by ACTs. Meaning varies by ACT.
- -Raw Item Power (RawIP)- — General item-power modifier used by effects such as Destruction/Healing Up and Super Properties.
- -AddP1- — P1 increase from `ITEM_SUB_CHANGE_ACTUAL_POWER` Properties such as Secret Healing/Explosive. It is calculated from the original P1 and added after RawIP scaling, so AddP1 itself is not multiplied by RawIP; later eligible multipliers still apply.
- -Fixed Damage- — Damage which is not affected by ATK/DEF/Level/Resistances or RawIP*/AddP1/Quality. Calculates normally otherwise.
- -Damage Rate- — The action-wide combo-rate modifier visible during support attack chains.

*Fixed item damage actually can lose damage from negative RawIP.

---

# 1. How A15 rounds numbers

A15 frequently truncates mid-calculation, converting back to an integer between steps. To truncate here is to drop the decimal: positive values round down and negative values round up toward zero.

## Why intermediate truncation matters

Suppose a hypothetical two-stage calculation does this:

stage1 = trunc(101 * 1.5) = 151  
stage2 = trunc(151 * 0.8) = 120

If you incorrectly combine both multipliers first:

trunc(101 * 1.5 * 0.8) = trunc(121.2) = 121

you get a different answer. Truncation is only applied at the integer boundaries the game actually uses; do not assume every multiplication creates a new truncation step.

---

# 2. Character damage

This chapter covers ordinary character attacks and battle skills using the standard HP-damage path.

## 2.1 Base power, ATK, and level

- `P` = the selected ACT P1 power
- `A` = current attacker ATK
- `L` = effective damage level (= actual level + eligible effective-level bonuses like Awakened Soul)

The standard non-item base is:

baseProduct = trunc(P * A / 100)  
levelFactor(L) = 1 + L^1.65 / 5000  (the target's level does not enter this factor)  
raw = trunc(baseProduct * levelFactor(L))

For normal character skills at level 99:

levelFactor(99) ≈ 1.3924895216

At effective level 119:

levelFactor(119) ≈ 1.5317193706

So +20 effective damage levels at level 99 is roughly a 10% damage increase to this pre-defense component.

## 2.2 Physical damage and DEF

For physical/default damage:

defense = trunc(targetDEF / 4)  
physicalCore = max(raw - defense, trunc(raw / 5))

The second branch is a floor: standard physical DEF cannot reduce this core below about 20% of raw pre-DEF damage. That does not mean all damage in the game has a universal 20% minimum; it is specifically the standard physical core described here.

Example: P1 = 160, ATK = 300, level = 50, DEF = 240.

baseProduct = trunc(160 * 300 / 100) = 480  
levelFactor(50) ≈ 1.1271541394  
raw = trunc(480 * 1.1271541394) = 541  
defense = trunc(240 / 4) = 60  
physicalCore = max(541 - 60, trunc(541 / 5)) = max(481, 108) = 481

Damage Rate, Critical Power, Conditional Output, Shared Output, and Battle-skill-only Power can still modify eligible output after the standard core.

## 2.3 Elemental standard damage

Fire, Water, Wind, and Earth standard damage do -not- use the ordinary physical DEF subtraction. Instead the damage path resolves the matching elemental resistance:

R_effective = min(resistance, 100)  
afterResistance = trunc(elementalCore * (100 - R_effective) / 100)

Resistance is capped at 100 on the upper end, but there is no matching lower clamp. Negative resistance therefore increases damage. This is why a physical hit and an elemental hit with similar pre-defense power can respond very differently to the same enemy.

## 2.4 Fixed damage is not “unmodifiable damage”

`ACT_DAMAGE_FIX` uses P1 as its fixed base. It is not affected by ATK, DEF, effective level, elemental resistance, AddP1, or Quality. Positive RawIP cannot strengthen it because its RawIP cap is 0, but negative RawIP can still reduce P1. Damage Rate, Critical Power, Shared Output, Skill Power, and other eligible later multipliers still apply. “Fixed” describes how the base damage is built, not immunity to later modifiers.

## 2.5 HP-left damage

`ACT_DAMAGE_HP_LEFT` preprocesses P1 before ordinary damage resolution:

bonus = min(P1, trunc(P2 * targetMaxHP / max(targetCurrentHP, 1)))  
effectiveP1 = P1 + bonus  (maximum: 2 * P1)

A notable data case is Fatal Attack. Its P2 is 200, already at least as large as any of its P1 rolls, so it reaches the full doubled-P1 cap even against a full-HP target. It is therefore misleading to read the name alone and assume it starts weak at full enemy HP.

## 2.6 Multi-hit attacks

A visual hit is not automatically a separate damage judgment. When a move truly resolves multiple independent physical judgments, DEF is applied to each judgment separately:

total = sum_i max(raw_i - trunc(DEF / 4), trunc(raw_i / 5))

A move split into several real judgments can therefore deal a different total from one aggregate hit with the same summed pre-DEF raw power. Animation alone is not evidence; some attacks show many slashes or projectiles while resolving fewer actual damage judgments.

---

# 3. Items: Quality, Raw Item Power, P1, and Properties

This is one of the easiest systems to misunderstand because several different numbers can all make an item “stronger.” They are not interchangeable.

## 3.1 The concepts to keep separate

- -Item Effect P1/P2- — The ACT's own parameters. Their meaning depends on the ACT.
- -Raw Item Power (RawIP)- — A percentage modifier that preprocesses P1 and, where the ACT allows it, P2.
- -AddP1- — A separate additive correction to eligible P1 after RawIP. It never changes P2.
- -Used-item Quality Coefficient- — The integer Quality operand used by ordinary consumable-item standard damage and other documented item-output paths.
- -Equipment Quality Factor- — The separate persistent-equipment Quality transform applied while an equipped item's own base/eligible inherent contributions are being built.
- -Properties- — Synthesis-added traits. Depending on the Property, they can supply RawIP or AddP1, generate another ACT, alter downstream output, change WT, or affect another mechanic entirely.
- -Downstream output- — B1–B5 and the external scalars described later. These modify eligible ACT output after its parameters and base result are established; they do not scale the stored P2 parameter.

For a normal eligible used-item effect, the useful calculation order is:

item-effect P1/P2 -> RawIP scaling -> AddP1 on P1 only -> item output core/Quality -> eligible downstream output

Not every ACT passes through every stage. The A15 Reference lists the per-ACT gates.

## 3.2 Raw Item Power (P0)

When an item ACT allows RawIP:

scaled P1 = trunc(base P1 * (100 + effective RawIP) / 100)  
scaled P2 = trunc(base P2 * (100 + effective RawIP) / 100)

Positive RawIP is limited by the -ACT-specific RawIP cap- in metadata. That cap means “this ACT accepts at most this much positive RawIP”; it is not a cap on final damage, P1, Quality, or downstream B4 output, and the existence of a cap does not prove that the ACT scales through RawIP at all. Some ACTs block positive RawIP while still permitting negative RawIP penalties.

For the full RawIP source list, caps, scaling, and dormant warnings, see the A15 Reference's ITEM_SUB table.

## 3.3 AddP1

`ITEM_SUB_CHANGE_ACTUAL_POWER`, used by the Fixed Increase / Secret Explosive / Secret Healing family, applies the AddP1 correction. Let:

AddP1 = total AddP1 value  
base P1 = selected runtime P1 before RawIP/AddP1 processing

AddP1 <= 0       -> added P1 = 0  
base P1 <= AddP1 -> added P1 = AddP1  
otherwise        -> added P1 = trunc(AddP1 * AddP1 / base P1)

After RawIP:

final P1 = scaled P1 + added P1  
final P2 = scaled P2

P2 never receives AddP1. The relative effect is larger on weaker base P1 values, which is why this family can be unusually strong in specific low-P1 cases without being universally superior.

Example: base P1 = 80, RawIP = +75, AddP1 = 20.

scaled P1 = trunc(80 * 175 / 100) = 140  
added P1 = trunc(20 * 20 / 80) = 5  
final P1 = 140 + 5 = 145

## 3.4 Used-item Quality Coefficient

After P1/P2 preprocessing, ordinary consumable-item standard damage uses Quality in the attack-like role that ATK occupies for character standard damage:

Used-item Quality Coefficient = trunc(75 + Quality / 2)

Examples:

Quality 0   -> coefficient 75  
Quality 50  -> coefficient 100  
Quality 100 -> coefficient 125  
Quality 120 -> coefficient 135

Ordinary item standard damage uses level 0 and bypasses the ordinary physical DEF term:

A = Used-item Quality Coefficient  
L = 0  
item core = trunc(final P1 * A / 100)

Continuing the previous example at Quality 100:

Used-item Quality Coefficient = trunc(75 + 100 / 2) = 125  
item core = trunc(145 * 125 / 100) = 181

That 181 is before eligible downstream output stages. The full order is therefore: choose base P1/P2 -> apply eligible RawIP -> add eligible AddP1 to P1 -> build the item core using Quality where applicable -> apply eligible downstream output.

## 3.5 Equipment Quality Factor

Equipped weapons, armor, and accessories use the same Quality curve but a different exact calculation stage:

Equipment Quality Factor = (75 + Quality / 2) / 100

The equipment path preserves the half-point at odd Quality values rather than first truncating to the used-item integer coefficient. For an individual equipment piece and one quality-aware parameter/effect channel, the useful model is:

pre-Property subtotal = that item's base contribution + its quality-eligible inherent Item Effect contributions  
quality-adjusted contribution = trunc(pre-Property subtotal * Equipment Quality Factor)  
final contribution = quality-adjusted contribution + Property contributions

The important consequences are:

- the item's own base equipment contribution can participate in the Quality stage;
- eligible inherent Item Effects on weapons, armor, and accessories are handled by the same system;
- the exact P1/P2 field decides whether an inherent effect is quality-sensitive — not every numerical equipment effect scales;
- Properties are processed afterward and do not have their own magnitudes multiplied by equipment Quality;
- each equipment piece is scaled/truncated separately before contributions from different pieces are added.

At Quality 120, the factor is 1.35. Forge Body is a clean example:

inherent Forge Body: +15 Max HP -> trunc(15 * 1.35) = +20  
Stagnant Water Property adding the exact same Forge Body through #7: +15 remains +15

Other quality-sensitive inherent fields include ordinary equipment stats, Critical Chance, Critical Power, Broad Power, Skill WT Reduction, Evasion, and several other dedicated channels. Other fields, such as Hit Rate and physical damage reduction, remain literal even when inherent. The A15 Reference gives the complete field-by-field list.

Weapon Quality is -not- a hidden multiplier on the P1 of a normal attack or battle skill. A high-Quality weapon can increase the current equipment-derived stats/effects that the action later reads, but the action's own damage ACT does not become `P1 * weapon Quality`.

## 3.6 How Properties can affect Item Effect output

A Property can affect item output in several mechanically different ways:

- add RawIP to the item action
- contribute AddP1
- add or modify B4 item output
- generate a separate damage/healing/status ACT
- alter item WT or target-count/rank/last-use conditions that feed RawIP
- modify another context-specific system

For equipped items, the general rule is simpler: -Property magnitudes do not scale with equipment Quality.- This includes direct stat Properties and Property-added Item Effects. Quality-increasing Properties such as Effect Up or Super Quality simply add their listed amount to the item's Quality; the Property value itself is still literal.

A Property-generated damage ACT is still its own runtime record. For example, Warlord Soul-style `ACT_DAMAGE_HP` is not “X% of the parent skill's damage”; it has its own P1 and then inherits whatever qualifying action-wide output context applies to that record.

For Warlord Soul, the P1 roll is `60..100`:

Main attack damage = standard physical formula (parent skill P1, parent action context, ...)  
Warlord Soul damage = standard physical formula (Warlord P1, same qualifying action context, ...)

The source weapon's Quality does not multiply Warlord Soul's generated P1. Battle-skill-only output can still apply to the separate damage instance when the parent action is a battle skill, even though the parent skill P1 never multiplies Warlord Soul's P1.

## 3.7 Random Base Up is not RawIP

`ITEM_SUB_ITM_RAND_BASEUP`, from sources such as Reliable Effect, changes or narrows an inherent item's random effect roll range. It does not add Raw Item Power.

---

# 4. Damage and output layers: P0 + B1–B5

These labels keep bonuses that sound similar from being collapsed into the wrong stage. When reproducing exact in-game integers, keep each stage and its stated truncation boundary separate rather than combining the whole chain into one multiplication.

## 4.1 The map and terminology

P0 = item ACT parameter preprocessing  
B1 = Damage Rate  
B2 = Critical Power  
B3 = Conditional Output  
B4 = Shared Output  
B5 = Battle-skill-only Power

Terms used throughout this chapter:

- P0 — Item-only P1/P2 preprocessing from RawIP and AddP1.
- Damage Rate — The action-wide additive rate used by B1.
- Critical Power — The critical-only output factor used by B2. A successful critical begins with +25 base Critical Power before additional Critical Power bonuses.
- Conditional Output — The shared B3 numerator for conditional source/target effects.
- Inner Fire — A B3 term supplied by `ACT_PASSIVE_INNER_FIRE`; it can contribute positively from the source or negatively from the target when its condition passes.
- Great Battle — A source-side B3 term supplied by `ACT_PASSIVE_GREAT_BATTLE` against a boss-flag target.
- Mitigate Damage — The target-side B3 subtraction supplied by `ACT_MITIGATE_DAMAGE`.
- Analyze adjustment — A separate B3 term from `ACT_PROTECT_DAMEGE_NUM`; it is not `ACT_MITIGATE_DAMAGE`.
- Shared Output — The B4 numerator containing broad power and context-specific item/non-item output terms.
- Broad Power — The B4 component fed by sources such as `ACT_EQUIP_ADD_POWER` and `ITEM_SUB_EQI_ADD_SKILLPOWER`. Its positive aggregate caps at +100.
- Non-item status output — The B4 contribution from `ACT_CHANGE_DAMAGE` in non-item context.
- Field/race output — The B4 contribution from `ACT_FIELD_RACE_STRENGTH` in non-item context.
- Item-effect output — The B4 contribution from `ACT_CHANGE_ITEMEFFECT` in item context.
- Passive item output — The B4 contribution from `ACT_PASSIVE_CHANGE_ITEM_POWER` in item context.
- Item correction — The item-context `ACT_CHANGE_ITEM_POWER` correction/gate applied within B4.
- Battle-skill-only Power — The B5 factor used by ordinary battle skills.
- External scalar — An eligible output multiplier outside P0 and B1–B5, such as the script/action scalar or MP-sufficiency scalar.

These are calculation labels for this guide, not terms the player needs to memorize.

## 4.2 P0 — item P1/P2 preprocessing

P0 is the item-only preprocessing discussed in Chapter 3: RawIP can scale eligible P1/P2, AddP1 can add to eligible P1 only, and ACT metadata decides eligibility and caps. P0 is not a final output multiplier.

## 4.3 B1 — Damage Rate

B1 is an additive rate whose currently applicable points are converted into a multiplier:

B1 = (100 + applicable Damage Rate points) / 100

Contributors include:

- intrinsic Damage Rate attached to individual ACT types
- successful critical: `+10`
- Back Attack: `+25`
- `ACT_ADD_CHAIN`: `+P1`
- accepted opponent-directed bad-status/disable event: `+5`
- nonzero opponent-directed status/modifier event: `+5`
- other specifically documented ACT contributions

The B1 accumulator has no upper bound. Back Attack can publish its +25 for ordinary item actions as well as character actions.

The two event-driven +5 flags are broader than “debuff bonuses.” The accepted-status flag requires at least one opponent-directed bad-status/disable component to pass its application/proc check and reach status routing. The other flag comes from a nonzero opponent-directed modifier event and can include effects that are beneficial to the target or manipulate its timing. For example, a modded attack that gives an enemy a positive buff, pushes/pulls its scheduled turn, or even grants it an immediate action can still qualify this modifier-event +5 because the event is opponent-directed and nonzero.

These event flags are booleans: several qualifying components do not repeatedly add +5 through the same flag.

Timing matters. A Damage Rate contribution generated by the current action does not retroactively change a damage judgment that has already resolved. Later judgments in the same action or later chain actions can see newly published B1 state where their path permits it. For this reason, the worked example below simply assumes an already-applicable B1 value rather than deriving that value from the same damage judgment.

## 4.4 B2 — Critical Power

A successful critical has +25 base Critical Power. Additional Critical Power bonuses add to that base before one B2 multiplication:

total Critical Power = 25 + additional Critical Power  
B2 = (100 + total Critical Power) / 100

Therefore:

critical with no extra Critical Power -> 125% output  
critical with Wind Soul +15 -> 25 + 15 = 40 Critical Power -> 140% output

Permanent `ACT_EQUIP_ADD_CRITBONUS` and temporary `ACT_INCREASE_CRITICAL_DAMAGE` can contribute additional Critical Power. A successful critical also independently publishes +10 Damage Rate into B1, subject to the B1 timing described above.

## 4.5 B3 — Conditional Output

B3 uses a shared 100-based numerator:

B3 = (100 + source Inner Fire + source Great Battle - target Inner Fire - target Mitigate Damage - Analyze P2) / 100

The established B3 sources are source Inner Fire, source Great Battle, target Inner Fire, target Mitigate Damage, and Analyze. Source Inner Fire and Great Battle add B3 points; target Inner Fire and Mitigate Damage subtract them. Analyze is a separate status from Mitigate Damage: it uses P1 as its qualifying trigger count and P2 = -25, so subtracting its negative P2 adds +25 B3 points for the qualifying output, after which its P1 count is consumed/decremented. Each term still has its own source, target, and action conditions.

Among shipped vanilla attacks that deal damage immediately, three named attack paths ignore B3:

- Knowledge Book — Resonant Damage (`ACT_DAMAGE_TARGET_LEFT`)
- Giant Slag — Finger Vulcan (`ACT_DAMAGE_RANDOM`)
- final boss — Wing Storm / Dark Feather (`ACT_DAMEGE_TIMECARD_NUM`)

Recurring-damage statuses such as `ACT_DAMAGE_HP_TURN` are also B3-off, but they are not immediate direct-damage attacks and are documented separately in the A15 Reference.

Heroic Soul and Lineage of Kings use the Great Battle / Inner Fire family. Against an equal-level boss, Heroic 7 + Lineage 5 can contribute:

7 + 5 Great Battle  
+ 7 + 5 Inner Fire  
= +24 B3 points

## 4.6 B4 — Shared Output

B4 is a broad shared-output numerator:

B4 = (100 + broad power + non-item status output + field/race output + item-effect output + passive item output + item correction) / 100

The positive broad-power aggregate caps at +100, but that cap does not apply to the completed B4 numerator after its other terms are added. `ACT_EQUIP_ADD_POWER` and `ITEM_SUB_EQI_ADD_SKILLPOWER` feed broad power. `ACT_CHANGE_DAMAGE` contributes in non-item context, `ACT_FIELD_RACE_STRENGTH` contributes field/race output in non-item context, and `ACT_CHANGE_ITEMEFFECT` plus `ACT_PASSIVE_CHANGE_ITEM_POWER` contribute through the item-context paths. These are not RawIP.

## 4.7 B5 — Battle-skill-only Power

B5 is the battle-skill-only layer. Ordinary player battle skills receive it; normal attacks and ordinary items do not. The only currently identified direct source of B5 points is `ITEM_SUB_EQI_ADD_SKILLPOWER`, which contributes the same P value to both broad power and B5. `ACT_EQUIP_ADD_POWER` contributes to broad power but not B5.

A Property-added runtime damage record inside a battle skill can inherit the action's B5 context when its ACT uses the relevant output path, but that inheritance is not another source of B5 points. Some vanilla effects using `ACT_EQUIP_ADD_POWER` are described as increasing skill power even though they contribute only to broad power; the Skill Up L/M/S family is a notable example.

## 4.8 External scalars

Two context scalars sit outside B1–B5. The first is a script/action scalar: ordinary actions supply neutral 1.0, while the engine exposes a script-controlled percentage path. Finishing Attack animation markers are a concrete use of this scalar family: their marker percentages divide the already-built finisher output across presentation points rather than creating a second stronger finisher skill.

The second is an MP-sufficiency scalar used by `ACT_CONSUME_MP` item/effect handling:

G = min(100, trunc(currentMP * 100 / cost))  
scalar = G / 100

If the character has insufficient MP, eligible output can be proportionally reduced. Neither scalar is another B bucket.

## 4.9 Final calculation example

Suppose a battle-skill damage judgment has:

Core Power = 481  
applicable Damage Rate = +40  
base critical with no additional Critical Power  
B3 = +24  
B4 = +30  
B5 = +20

Its external scalars are neutral for this example.

after B1 = trunc(481 * 140 / 100) = 673  -- 140% Damage Rate  
after B2 = trunc(673 * 125 / 100) = 841  -- base critical: 25 Critical Power, 125%  
after B3 = trunc(841 * 124 / 100) = 1042 -- 124% Conditional Output; e.g. Heroic Soul + Lineage of Kings versus a boss  
after B4 = trunc(1042 * 130 / 100) = 1354 -- 130% Shared Output  
after B5 = trunc(1354 * 120 / 100) = 1624 -- 120% Battle-skill-only Power

---

# 5. Criticals

A15 separates critical chance from Critical Power.

## 5.1 Critical chance

Critical-chance sources differ by action type.

For ordinary attacks and battle skills:

- `ACT_EQUIP_ADD_CRITICAL` adds permanent critical-chance points.
- `ACT_PASSIVE_CRITICAL` adds permanent ordinary attack/skill critical-chance points.
- `ACT_CHANGE_CRITICAL` adds a temporary critical-chance modifier.
- `ACT_INSTANT_CRITICAL` adds action-local critical-chance points.
- a matching Slayer race condition adds +100 critical-chance points.

For items:

- `ITEM_SUB_ITM_ADD_EFFCRIT` adds the used item's Property-derived critical-chance points; vanilla examples include Critical, Critical+, Critical++, Critical Hit, and Fatal Blow.
- `ACT_PASSIVE_PINCH_HITTER` can contribute P1 to item critical chance in item-critical mode.
- `ACT_EQUIP_ADD_CRITICAL` does not contribute.
- `ACT_PASSIVE_CRITICAL` does not contribute.
- the ordinary Slayer race-critical path does not contribute.

The A15 Reference lists the complete ACT/Property source details.

## 5.2 Critical results and Critical Power

A successful critical has two distinct consequences where eligible:

- it publishes +10 Damage Rate into B1;
- it receives +25 base Critical Power in addition to other critical power bonuses.

With no extra Critical Power:

B2 = (100 + 25) / 100 = 1.25

Additional Critical Power adds to that 25. Wind Soul's +15 therefore gives:

25 base + 15 Wind Soul = 40 Critical Power  
B2 = 1.40

It is not `1.25 * 1.15`.

Permanent `ACT_EQUIP_ADD_CRITBONUS` can benefit eligible item critical output as well as character critical output. If the Critical Power comes from an inherent equipment Item Effect, its authored value may first receive that equipment piece's Equipment Quality Factor; Property-added Critical Power remains literal.

## 5.3 Temporary Critical Power

`ACT_INCREASE_CRITICAL_DAMAGE` installs a temporary Critical Power status. Within one status store, the newest copy replaces the previous copy regardless of P1, reapplication refreshes its duration to 5, and a critical hit does -not- consume it. A normal copy and a field-effect copy can coexist; their values are added along with permanent Critical Power and the base +25 before B2 is formed.

---

# 6. Status effects, buffs, and debuffs

This chapter focuses on status mechanics that materially affect player decisions rather than cataloguing every status enum.

## 6.1 Application checks

For the bad-status/check family discussed here:

check modifier = attacker success bonus - target check resistance  
final application chance = native ACT chance + check modifier

This family includes Sleep, Poison, Slow, Curse, Blind, Triplet, Random Ailment, Percent Death, and Assist Faint. The same success/check bonus is not a universal modifier to ordinary hit rate, critical rate, or unrelated RNG procs.

Application chance and status severity are separate concepts. A target can reduce the chance that a status lands and can also mitigate the depth/severity of a status that does land, depending on the ACT involved.

## 6.2 Stat buffs and debuffs are flat stat changes

The ordinary ATK/DEF/SPD up/down ACTs use flat points, not percentages.

For example:

ATK Down P1 = 20 -> current ATK is reduced by 20 points  
DEF Down P1 = 20 -> current DEF is reduced by 20 points  
SPD Down P1 = 20 -> current SPD is reduced by 20 points

Their gameplay impact then comes from whatever later formula reads that stat. An ATK debuff changes the `P * ATK / 100` damage base; a DEF debuff changes the target's `trunc(DEF / 4)` physical-defense term; a SPD debuff can affect both WT and the level/SPD hit-rate buffer.

`ACT_PLUS_ALL` / `ACT_MINUS_ALL` use P1 as the flat ATK/DEF/SPD change and P2 as the all-element resistance change.

For used-item runtime effects, ordinary single-stat and all-stat up/down ACTs can accept RawIP where their ACT metadata allows it, with the ordinary stat family allowing a large positive RawIP cap. AddP1 does not strengthen these status magnitudes. General Skill Power, B4, and B5 do not multiply a buff/debuff's stored P1/P2 merely because the action also deals more damage. Special category-specific raw-parameter scalers, such as support-skill power effects that explicitly preprocess ACT parameters, are separate mechanics and are documented per ACT.

Single-element `ACT_PLUS_RESIST` / `ACT_MINUS_RESIST` are another useful contrast: their element selector/magnitude path does not use the generic RawIP scaling applied to the ordinary stat-up/down family.

## 6.3 Immunity, check resistance, and mitigation are different

Some enemies and passives use hard immunity flags such as all-status immunity or individual Poison, Curse, Blind, Death, or Sleep-family immunity. A hard immunity is a presence check, not a very large negative percentage. If the relevant immunity flag applies, adding more attacker success chance does not “outroll” it.

Check resistance is softer: it subtracts points from the application check and can therefore be opposed by attacker success bonuses. Status mitigation can separately reduce the payload/severity after the application check.

This distinction matters against bosses: “the status missed because the chance was low,” “the target's check resistance reduced the chance,” and “the target is outright immune” are three different outcomes.

## 6.4 Status stores and replacement

Battlers have a normal status store and a separate field-effect status store. Ordinary skill/item statuses go to the normal store, while field-effect generation can route statuses to the field store.

For the indefinite bad-status family discussed here, such as Poison, Slow, Curse, and Blind, same-status copies in one store do not sum; the higher-severity value is retained in that store. A normal-store copy and a field-effect copy can coexist rather than replacing each other. Whether both retained copies combine or are read separately depends on the downstream mechanic.

Do not generalize this rule to stat buffs, debuffs, Critical Power, drains, or WT statuses; those families have their own replacement and combination rules.

## 6.5 Poison, Slow, Curse, and Blind

- Poison — `ACT_BAD_POISON`: P1 = application chance, P2 = severity. `poison % = min(trunc(severity / 50), 20)` and `tick = max(1, trunc(currentHP * poison % / 100))`, so the percentage component caps at 20%.
- Slow — `ACT_BAD_SLOW`: P1 = application chance, P2 = severity. `Slow WT = 50 + trunc(severity / 2)`. This is a WT addition before the later common/skill/item percentage-reduction stages.
- Curse — P1 = application chance, P2 = severity. Eligible outgoing output uses `Curse factor = 1 - clamp(severity / 1000, 0, 1)`, so severity 250 gives a 0.75 factor. Curse is not part of P0 or any B1–B5 accumulator; treat this factor as a separate eligible outgoing-output multiplier alongside those stages.
- Blind — P1 = application chance, P2 = severity. Ordinary hit calculations receive `attacker Blind contribution = trunc(severity * -0.1)` or `target Blind contribution = trunc(severity * +0.1)`. Blind therefore makes the status-holder both less accurate and easier to hit.

## 6.6 Triplet and Comet Attack

Triplet independently attempts Poison, Curse, and Blind. P1 is the application chance for each component and P2 is their shared mitigated severity. `ACT_BAD_TRIPLET` itself contributes +7 Damage Rate.

If at least one component passes the opponent-directed status acceptance path, the action also publishes the separate hostile-status event worth +5 Damage Rate. Absent unrelated B1 sources:

all three components fail -> +7 published Damage Rate  
at least one component is accepted -> +12 published Damage Rate

The +5 is not hidden Triplet metadata; it is the general accepted-hostile-status event described in Chapter 4. A later status-store rejection does not undo that event, so a successful application can qualify the +5 even if an equal or higher-severity ailment remains.

Comet Attack combines `ACT_DAMAGE_4ELEMENT` with `ACT_BAD_TRIPLET`, which produces this +7 / +12 Damage Rate behavior for subsequent applicable output/state; do not assume the newly generated rate retroactively changes damage that already resolved earlier in the action.

---

# 7. WT: from an action's base wait to the next turn

WT is a staged pipeline. -Do not add every “WT -X%” source into one giant percentage.- Sources that feed the same stage add first; distinct stages multiply sequentially and preserve their own truncation boundaries.

The scheduler stores ordinary remaining WT in the range 0..1000. Values can be above or below that range during intermediate arithmetic, but the scheduled key is clamped before/at final assignment. WT 0 is immediately eligible; 1000 is the stored ceiling.

## 7.1 High-level pipeline

For a battle skill:

base skill WT -> SPD reduction -> Slow addition -> common `ACT_REDUCE_WT` -> Skill WT Reduction -> optional action-local Skill WT Reduction -> later Quick/proc reduction -> scheduler clamp/assignment -> possible Immediate Action override

For an item:

base item/action WT -> common stages -> item-only WT reduction -> later Quick/proc reduction -> scheduler clamp/assignment -> possible Immediate Action override

The exact source list differs by context.

## 7.2 Common WT stages

-SPD-: positive SPD-derived WT reduction caps at 25%. This is its own stage and does not merge into Skill WT Reduction or item Quick Use.

-Slow-: after the SPD stage, Slow adds:

Slow WT = 50 + trunc(severity / 2)

-`ACT_REDUCE_WT`-: this common temporary reduction uses:

r = combined normal + field reduction, positive side capped at 50  
WT = trunc(WT * (100 - r) / 100)

Same-store copies replace/refresh rather than self-add. One normal copy and one field-effect copy can coexist. They add together before the 50% cap.

## 7.3 Skill WT Reduction

Permanent `ACT_PASSIVE_REDUCE_WT_SKILL` sources add into the character's -Skill WT Reduction- parameter. Source provenance matters before they are summed:

- Master Crusher contributes its literal +15.
- Dark God Soul's Property-added value is literal +10.
- Acrobatic Armor's inherent Swift Knowledge +7 and Swift Secrets +10 are first adjusted by that armor's Equipment Quality Factor. At Quality 120 they contribute +9 and +13 respectively.

The permanent sources sum, then positive Skill WT Reduction is capped at 50:

q = min(positive Skill WT Reduction, 50)  
WT = trunc(WT * (100 - q) / 100)

Negative q increases WT.

There is also a separate action-local path using the same ACT, `ACT_PASSIVE_REDUCE_WT_SKILL` (ACT151). This does -not- mean the game scans your equipment/passives a second time. It means that if the current action itself carries an ACT151 record, that record's P1 can supply another sequential WT multiplier.

For illustration, if the permanent Skill WT Reduction stage has already produced 308 WT and an action-local ACT151 P1=20 is present:

308 -> trunc(308 * 80 / 100) = 246

The later Quick stage would then start from 246. This action-local path is a special record path rather than a second copy of Dark God Soul/Swift Secrets/etc.

## 7.4 Item-only WT reduction

Items have their own reduction bucket, which can include permanent `ACT_PASSIVE_REDUCE_WT_ITEM`, the `ITEM_SUB_BTL_QUICK_USE` / Quick Use family, and retained `ACT_REDUCE_WT_ITEM`. Positive combined item-WT reduction is capped at 50 before multiplication:

item reduction = min(positive combined item-WT reduction, 50)  
WT = trunc(WT * (100 - item reduction) / 100)

Vanilla Quick Use-family examples are Quick Use +10, Quick Use+ +20, Sonic Throw +30, and Mach Throw +50; negative variants increase WT. `ACT_REDUCE_WT_ITEM` is a separate retained item-WT status and has no shipped vanilla skill/item-effect emitter.

## 7.5 Later Quick reductions

`ACT_ADD_PERCENT_QUICK` is a temporary Quick status with P1 = proc chance. On success it requests a hardcoded 25% WT reduction:

WT = trunc(WT * 75 / 100)

P2 is ignored by this WT calculation, and the status lifetime is fixed at 5.

`ACT_PASSIVE_PERCENT_QUICK` is the permanent proc version. Matching permanent sources sum P1 and P2 separately:

chance = sum(P1)  
reduction = sum(P2)

On a successful passive proc:

WT = trunc(WT * (100 - reduction) / 100)

If both temporary and permanent Quick are present, they do -not- add their chances together and they do -not- apply two sequential WT multipliers. The engine performs separate proc checks, then applies one Quick reduction:

- if the permanent Quick proc succeeds, use its summed P2 reduction;
- otherwise, if the temporary Quick proc succeeded, use the temporary hardcoded 25%;
- otherwise, apply no Quick reduction.

So the permanent Quick result takes precedence if both checks succeed.

## 7.6 Immediate Action Chance

Despite its internal “Quick” name, `ACT_PERCENT_QUICK` is not a normal percentage WT reduction:

P1 = Immediate Action Chance  
P2 = Repeat Chance Penalty

The actor keeps a signed accumulator `A`. For each copy:

threshold = P1 + A; roll integer RNG 0..99  
success -> A = A - P2  
failure -> A unchanged

There is no explicit 0..100 clamp: threshold >= 100 is guaranteed success and threshold <= 0 is impossible. A one-shot preserve latch carries the reduced accumulator into the immediate follow-up; otherwise ordinary cleanup clears it. Rain Thrust 100/33 therefore gives successive link chances of 100%, 67%, 34%, 1%, then a negative/impossible threshold if every prior attempt succeeds.

A successful Immediate Action is a scheduler override, not another percentage reduction on the computed WT: the actor's normal remaining-WT key is made immediately eligible at 0.

## 7.7 Worked WT example

Suppose a 600-WT skill has already reached 480 WT after the separate SPD stage-:

Slow severity 40 -> 50 + trunc(40 / 2) = 70 -> 480 + 70 = 550  
common WT reduction 30% -> trunc(550 * 70 / 100) = 385  
Skill WT Reduction 20% (Dark God Soul +10 + Swift Secrets +10 on Quality-50 Acrobatic Armor) -> trunc(385 * 80 / 100) = 308  
temporary Quick succeeds, passive Quick does not -> trunc(308 * 75 / 100) = 231

This example assumes no action-local ACT151 record and no Immediate Action proc. The resulting 231 is already inside the scheduler's 0..1000 range, so it is stored as 231. Adding the named percentages together and multiplying once would produce a different result.

If an action-local ACT151 P1=20 were present, it would be applied before the Quick:

308 -> 246 -> temporary Quick -> 184

If an Immediate Action proc then succeeds, the scheduler is instead made immediately eligible at WT 0; that is not `184 * another percentage`.

---

# 8. What the timeline UI actually means

## 8.1 Scheduler order

Battle scheduling uses a fixed pool of 16 timeline entries. The authoritative key is an integer -remaining WT- value, not a persistent absolute timestamp.

When time advances, entries are stably sorted by remaining WT; the first active entry's countdown becomes the elapsed amount `D`; `D` is subtracted from later entries; values are clamped into the scheduler range; then the vector is sorted again. Example: `A=40, B=90, C=150` becomes `A=0, B=50, C=110` when A comes due.

When an actor finishes a normal action, its newly calculated final WT -replaces- the normal entry's remaining WT; it is not added to the old value. The stored range is 0..1000: 0 is immediately eligible and any value above 1000 at the assignment/clamp stage is stored as 1000 before later battle-time rebasing.

The sort is stable and compares only remaining WT. If two entries have equal WT, their existing vector order is preserved, so another mechanic that explicitly moves an entry earlier or later can matter when their keys later tie.

## 8.2 Visible cards are rank-based, not a WT ruler

The UI maps scheduler -rank- to fixed card slots; horizontal distance does not represent the numerical difference in remaining WT. Card order is meaningful, but a gap twice as wide does not mean twice as much WT.

Inactive entries remain in the raw 16-entry vector and can consume a rank while their cards are hidden:

scheduler: Escha | hidden inactive | enemy A | Logy  
visible UI: [Escha] [blank] [enemy A] [Logy]

The blank is a hidden scheduler entry, not “missing WT distance.” If a delayed effect, revive, field entry, or another scheduler operation reuses/reactivates that entry, a card can appear in the blank slot without moving through some hidden quantity of WT.

Card movement is also tweened from its previous UI coordinates. Logical order can update before the cards finish moving, producing temporary overlaps or gaps. -Trust the settled card order; do not infer exact WT from card spacing.-

---

# 9. Timeline manipulation

## 9.1 Return WT / knockback

`ACT_RETURN_WT` pushes a target's normal timeline entry later. Each handler invocation increments a per-actor application count `n`, so the push diminishes approximately like `P1 / n` before knockback resistance is applied.

`ACT_RESIST_KNOCKBACK` then reduces the push by `(100 - resistance) / 100`; resistance 100 gives immunity. Matching permanent resistance sources sum before this stage.

The diminishing behavior matters for true multi-judgment or repeatedly applied Return-WT effects: repeated applications to the same actor before its reset become progressively weaker.

## 9.2 When the diminishing counter resets

The Return-WT counter does -not- reset at generic action end, support-chain end, the attacker's turn, or a universal “round” boundary. It resets when -that actor's non-field timeline entry is selected- by the scheduler.

“Non-field” is a scheduler entry type, not shorthand for “not a time card.” A delayed/time-card component can be actor-associated and non-field; when such an entry is selected, it resets the -owner's- Return-WT counter. A true field-effect timeline entry follows the separate field path and does not perform this reset.

So if actor X owns a delayed card that later hits enemy Y, selecting that non-field delayed entry resets X's Return-WT count, not Y's. Return-WT behavior can therefore persist across support/immediate chains longer than a simple “once per turn” description suggests.

A simple example of this unintuitive behaviour:
Awin uses Sparkbitter -> Awin is hit with 100 WT delay -> Sparkbitter activates -> Awin is hit with another 100 WT delay
This sequence results in a total turn delay of 200, not 150.

## 9.3 Direct mutations and off-turn actions

At the scheduler level:

- Return-WT/push adds to the existing normal entry's remaining key and clamps it to the scheduler range.
- pull/advance subtracts from the key and moves the pointer earlier.
- immediate action writes remaining WT 0 and repositions the entry toward the front.

These operations can also alter vector order relative to equal-WT entries.

Normal actor cards are persistent/reused scheduler entries. Support actions and Double Draw are off-turn actions executed in their action context while the actors' normal future turns remain represented by the scheduler; they do not receive a separate private scheduler-card class. Delayed/time-card actions are different and can have independent scheduler entries.

---

# 10. Delayed and time-card actions

## 10.1 Reconstruction at activation

A delayed action carries scheduling and identity information, then reconstructs a fresh action when the card fires. Its entry carries information such as schedule, owner identity, deferred descriptor/payload, selector, and target identity.

At activation, the engine constructs a fresh action and resolves the selected follow-up component. The system is therefore predominantly -reconstruction, not damage snapshotting-: owner/target objects, selected skill/ACT data, current stats, current statuses/modifiers, and the final ACT/damage judgments are generally looked up or rebuilt at fire time.

A delayed card should not be modelled as “the parent calculated 500 damage now and stored 500 for later.” If relevant state changes before the card fires, the component can evaluate under the later state, subject to that component's own rules. Sparkbitter, Wind Call, and Twin Call follow this family; Twin Call is especially clear because the parent does not freeze its final follow-up choice, and the delayed component chooses the relevant child/elemental follow-up when it fires.

## 10.2 Support response

Deferred/Time Card activation suppresses the normal Support Attack/Support Guard response window. A delayed component firing from the timeline therefore does not open a new Support Guard opportunity at that activation point.

---

# 11. Support attacks, guards, chains, and Finishing Attacks

## 11.1 Action selectors

Character action selectors are:

0 = Normal Attack  
1–3 = Ordinary battle skills  
4 = Support Special  
5 = Passive 1  
6 = Passive 2  
7 = Finishing Attack  
8 = Support Attack  
9 = Support Guard  
10 = Move  
11 = Change  
14–16 = Enhanced support variants

These selectors matter to category-gated effects such as `ACT_FIX_SKILL_BONUS`. Selector IDs are a separate namespace from ACT IDs.

## 11.2 Support Attack and chain Damage Rate

The standard support-attack damage ACT is `ACT_ASSIST_ATTACK`, whose intrinsic Damage Rate contribution is +25. The support action still uses standard damage behavior allowed by its action context.

Linca's Stunning Blow is a useful example:

ACT_ASSIST_ATTACK -> intrinsic Damage Rate +25  
ACT_RETURN_WT -> intrinsic Damage Rate +5  
total published intrinsic Damage Rate -> +30

Its Return-WT component also performs the timeline push from Chapter 9.

Damage Rate is action-wide state, and support actions can publish their own intrinsic B1 contributions. Separate permanent chain sources such as `ACT_ADD_CHAIN` add P1 directly into Damage Rate rather than secretly multiplying the support-attack P1. As with other B1 sources, do not assume newly published Damage Rate retroactively changes the damage judgment that generated it.

## 11.3 Support Guard and internal Guard state

Retail Support Guard uses `ACT_ASSIST_GUARD`:

guarded damage = max(1, trunc(incoming damage * (100 - P1) / 100))

P1 = 33 -> 33% reduction  (Initial Guard)
P1 = 50 -> 50% reduction  (Hard Guard)

This is a target-side percentage reduction. A passive Support-Guard power source can scale the raw Support Guard ACT parameters for the appropriate support-guard action categories before the guard handler.

A15 also contains a separate transient internal -Guard state-. When its latch is active, metadata-eligible ACTs can receive a fixed x0.5 runtime output/effect scalar after P0 and outside B1–B5. This is a source-side internal state, not selector-9 Support Guard, and normal skill data does not set the internal Guard-state latch.

## 11.4 Finishing Attack animation behaviour

All nine real player Finishing Attacks use one native Finishing Attack definition per character. They do not have a separate higher-P1 “lethal version”: the normal finisher uses its ordinary full-strength damage ACT, with Finishing Attack accuracy supplied separately.

The engine evaluates the normal full finisher outcome before the special choreography is selected. If the finisher is not lethal, the short presentation is used; if it is lethal, the extended presentation is selected, with the battle-ending version adding its extra presentation/BGM behavior.

The animation scripts then divide the already-built finisher output across damage markers rather than substituting a stronger finisher ACT. Several elaborate finishers have marker percentages that total exactly 100%; Micie's extended animation is especially clear because it contains one actual 100% damage marker despite many additional visual sword strikes.

Therefore getting the enemy into the long Finishing Attack animation does -not- grant a hidden execution-damage bonus. The long animation is selected because the ordinary finisher already qualifies as lethal; it is not what makes the finisher lethal.

---

# 12. Special item behavior and retail bugs

## 12.1 Slag Statue chooses HP heal, MP heal, or revive from target state

Call Slag's helper effect is a selector. Its own helper-tier P1/P2 values are not the final recovery amounts. The target's state chooses one of three ordinary replacement effects:

- target is KO -> Recover KO S / `ACT_CURE_REVIVE`, P1 32..45: revive and restore that percentage of max HP;
- target is alive at 50% HP or lower -> HP Recovery S / `ACT_HEAL_HP`, P1 48..68;
- target is alive above 50% HP -> MP Recovery M / `ACT_HEAL_MP`, P1 21..30.

The internal name BADHEAL refers to the above-50%-HP branch, but mechanically that branch is ordinary MP recovery. All three Call Slag helper tiers use the same downstream base recovery distributions after the branch is chosen; the helper-tier authoring values are discarded as recovery operands.

The HP/MP recovery branches then use their normal item-healing scaling rules, including eligible RawIP/AddP1, Quality, and B4. The revive branch follows the ordinary `ACT_CURE_REVIVE` percentage-of-max-HP rule.

## 12.2 Retail bugs and counterintuitive data

-English “Special Attacks” selector mismatch-  
English `ITEM_EFF_YOBI_005` (bonus finisher damage, attached to Treasure Grimoire's Special Attacks effect) is authored with action selector 6, while Finishing Attack is selector 7. Base/JP uses selector 7. The shipped English effect therefore checks the wrong action category and does not affect Finishing Attacks as intended.

-Awakened Soul's stat-growth component cannot be collected-  
Awakened Soul has two independent level-related effects. `ACT_EQUIP_PARAM_LEVEL +20` works and raises effective damage level for standard non-item damage. Its separate `ACT_EQUIP_PARAM_LEVEL_PROPOTION 3` would add `trunc(baseStat * actualLevel * 3 / 1000)` to base ATK/DEF/SPD, but that collector scans accessory slots while Awakened Soul is an armor Property and cannot legally be placed on accessories. The proportional-stat component is therefore nonfunctional on Awakened Soul; the +20 effective-level component still works.

-Angelic Healing does not grant Guts-  
Angelic Healing points to the Item Effect named Avoid KO, but that Item Effect is wired to `ACT_AUTO_REVIVE` with P1=25 rather than the Guts ACT. The result is a 25%-max-HP Auto Revive effect, not the advertised Guts/Avoid-KO behavior.

-Stat Drain replacement favors the wrong sign on the target-  
Exact duplicate drain-status IDs retain the signed numerical maximum. This works sensibly for the source-side positive stolen-stat bonus because the larger positive value wins. On the target side, however, drain losses are negative, so the numerically larger value is the one closer to zero. A weaker existing negative drain can therefore block a stronger later negative drain from replacing it.

-Equipment physical reduction can override a stronger reduction-  
`ACT_EQUIP_REDUCE_SUFFERDMG` is the permanent physical/default standard-damage reduction channel, capped at 50%. `ACT_AUTO_DAMAGE_REDUCE` is a separate physical/default reduction that can reach 80%. In the traced physical branch, if the equipment reduction is present it replaces the selected Auto Damage Reduce value instead of adding to it or choosing the stronger value. A small equipment reduction can therefore make the result worse than the other reduction alone. This does not apply to elemental standard damage or Fixed Damage, which use different defensive paths.

-Fatal Attack's low-HP ramp is already maxed at full HP-  
This is better described as a shipped-data quirk than a broken formula. `ACT_DAMAGE_HP_LEFT` itself works normally, but Fatal Attack uses P2=200. Even against a full-HP target, that is already enough to hit the ACT's maximum +100% P1 bonus, so the intended “stronger as the target gets low” progression is functionally invisible: it starts at the cap.

-Sleep's retail application path is broken-  
`ACT_BAD_SLEEP` exists in shipped data and still contributes its intrinsic +3 Damage Rate, but the shared bad-status mapper rejects the Sleep ACT enum before a Sleep status is installed. It therefore does not create the intended Sleep status through this retail path and does not earn the accepted-status +5.

## 12.3 Dormant mechanics

Some implemented mechanics are unused by shipped content. `ACT_REDUCE_WT_ITEM` is understood but has no vanilla skill/item-effect emitter, and the ACT87 one-shot common-WT global is implemented but likewise has no shipped emitter. These can matter to modders without being presented as normal vanilla player mechanics.

---

# 13. FAQ / common questions

-How does A15 round damage and other calculations?-  
It truncates at specific integer boundaries throughout the calculation. Do not multiply every percentage together first and round once at the end.

-How does Quality affect a used item?-  
Ordinary used-item standard damage uses `trunc(75 + Quality/2)` as its attack-like coefficient at level 0. Other item ACTs use Quality only where their documented output path permits it.

-How does Quality affect equipment?-  
Each equipment piece has an Equipment Quality Factor `(75 + Quality/2)/100`. Its quality-aware base/inherent contributions are scaled before Properties are added. Properties themselves do not scale with equipment Quality.

-Does weapon Quality directly multiply my normal attack or battle-skill P1?-  
No. It can improve the weapon's quality-sensitive equipment stats/effects, which the attack may later use, but there is no generic `skill P1 * weapon Quality` preprocessing stage.

-Does character level increase ordinary item damage?-  
Not through the normal character level factor. Ordinary item standard damage uses level 0.

-Does physical DEF reduce ordinary item standard damage?-  
No. Ordinary item standard damage bypasses the normal physical DEF subtraction. Elemental resistance and other special defensive mechanics can still matter where their paths apply.

-Does Fixed Damage ignore every modifier?-  
No. It bypasses ATK, effective level, DEF, elemental resistance, AddP1, and positive RawIP strengthening; negative RawIP can still reduce it, and eligible later output multipliers can still apply.

-Do all WT reductions add into one percentage?-  
No. Same-stage sources can add, but SPD, Slow, common WT reduction, Skill/Item WT Reduction, and late Quick are distinct stages with separate truncation points.

“Do the blank cards on the timeline represent a certain WT value?”
No. They are unused scheduler slots, not fixed amounts of WT. New Time Cards and other scheduled events can later use them, while existing cards can shift across these blank positions as the timeline is reordered. Essentially, the blanks are just unused capacity; their position and spacing have little gameplay meaning.

“What happens if all card slots are filled and another card is created?”
Testing is still required, but the game appears to have no behaviour for safely handling an overflow. If a 17th card were created with no reusable slot available, the game would likely violate an assumed invariant and probably crash. This situation may not be reachable in vanilla gameplay.

-Do delayed cards store the original damage number?-  
No. They reconstruct the action/component at activation and can read then-current state.

-Can a delayed component open a new Support Guard response when it fires?-  
No. Deferred activation suppresses the normal support-response window in the established path.

-Does a long Finishing Attack animation deal bonus damage?-  
No. The ordinary finisher is evaluated first; the long animation is selected because that result is already lethal.

-Is Triplet always +12 Damage Rate?-  
No. The ACT itself publishes +7. The additional +5 requires at least one component to pass the accepted hostile-status event path.

-Is Sleep merely difficult to land?-  
No. The shipped `ACT_BAD_SLEEP` application path is broken and installs no Sleep status through that route. The sleep status itself also has multiple issues and cannot be easily reimplemented. ACT_BAD_SLEEP is technically in-game but is only attached to the Yggdrasil boss' Left Hook.

---

# 14. Calculating in Practice:

For a real attack:

1. Identify the action type: normal attack, battle skill, support action, item, delayed component, etc.
2. Identify the ACTs actually being resolved.
3. For a used item, resolve P0 in order: base P1/P2 -> eligible RawIP and its ACT-specific cap -> eligible AddP1 on P1 only.
4. Build the appropriate core:
   - physical character standard damage: P1 and current ATK -> effective damage level -> physical DEF;
   - elemental character standard damage: P1 and current ATK -> effective damage level -> matching elemental resistance instead of physical DEF;
   - ordinary item standard damage: final P1 and Used-item Quality Coefficient at level 0, bypassing ordinary physical DEF;
   - Fixed Damage or another special ACT: use that ACT's own core formula instead of forcing it through the standard formula.
5. Determine the B1 Damage Rate that is actually applicable when this damage judgment resolves. Do not retroactively add B1 points that the same judgment only publishes afterward.
6. If the result is critical, build B2 from the base +25 Critical Power plus any additional Critical Power bonuses.
7. Apply only the B3 conditions that actually qualify for this source, target, and action.
8. Build B4 from the appropriate item or non-item context.
9. Apply B5 only to an actual battle skill.
10. Keep external/script scalars separate and preserve every stated truncation boundary.

For WT, start a separate staged calculation rather than reusing damage “power” concepts.

-----------

# 15. Burger King foo- Afterword:

This mechanics guide was created because I was frustrated with the game's ambiguities, bugs, and sometimes straight-up incorrect descriptions. I started by asking ChatGPT for basic hard-to-search info, and it eventually said, "hey bro, toss me the exe". One thing led to another, and here we are. This reference was created after a large amount of code inspection and reverse engineering by ChatGPT. While I've done my best to guide the process, find contradictions/misunderstandings, audit documents, and include all relevant info, I'm sure there will be qualms here or there. If anything doesn't line up with your experience, or if a result differs from what you'd expect given the info here, please report it to the appropriate forum (probably Github). 
