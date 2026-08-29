# Atelier-Escha-Logy-Mechanics-Reference
This repository contains various documents which detail obscure and hidden calculations, effects, and other quirks in A15. May be expanded later to include other games.

#### All documents have been tested against my experience and vetted as best I could. The bulk will be accurate, but small errors may be present. Please report any errors or inconsistencies you find in the issues section.


Let's face it, A15 is as obtuse as the Atelier games come. It doesn't help that there are many descriptions that don't line up with the reality of the game, and even labels in the game files which are completely separate from what they actually do.
In a fit of frustration, I decided to violently throw this game at the metaphorical wall of ChatGPT, and to my surprise the game fractured into intelligible pieces. Here are those pieces:

## A15MechanicsReference
This document contains details regarding calculations for damage, WT, buffs/debuffs, status effects, and et cetera. Most of the general under-the-hood details are here.

## A15SkillReference
Contains general data on both player and enemy skills like base WT, act_tags, and targeting. Includes special notes where warranted.

## Future Documents

### A15Acts&ItemSubs
Each skill, attack, accessory, armour, item effect, property, and otherwise get their actual effects from act tags and item subs. This references includes all act_tags and item_subs with information such as scaling behaviour (quality, item power, caps, fixed power, etc) variables, and so forth. This will help you reconstruct precise details from the following documents:

### A15ItemReference
This will contain general and simple information about the items in the game. It'll list the same kind of info as the skill reference as well as synth data and any noteworthy quirks.


## Future Plans:
There's a module based patcher compatible with EN and JP versions of the game in-progress using the data from this datamining project. It works by backing up the native executables and replacing them with patched versions when the patcher is run. You only have to use the modules you want, the patcher will function with any combination. Data is configurable through user-editable jsons (essentially configs) for data buried in the exe. The vanilla-editing modules will ship with templates containing unmodified vanilla data. The current modules are Skills, Items, Music, Fun stuff, and Bug Fixes.

Skills - Attach different act_tags and edit the associated values. Make Escha's staff a one-shot machine gun if you want.

Items - Same as above, but with editable recipes, production amounts, and other relevant stuff.

Music - Ever notice that some music is incredibly loud (Rorona Plus) and others are ridiculously quiet (early Atelier stuff)? Fix or adjust it yourself. I'll ship a pre-configured version to balance the tracks as well.

Fun Stuff - Essentially just miscellaneous changes I thought might be fun. Currently only contains an execution mechanic which grants a damage bonus to finisher skills if that bonus would result in a kill. 

Bug Fixes - A set of individually toggleable bug fixes I noticed in-game and while conducting this project. Includes:

.Awakened Soul not giving stats as intended

.Angelic Healing granting auto-revive instead of the stated guts effect

.Stat Drains prioritizing weaker debuffs

.Non-functional Special Attacks effect on the Treasure Grimoire

.Equipment damage reduction nullifying temporary damage reduction

.Fatal Attack not scaling with enemy HP

I'll add fixes for any more bugs reported if possible.

These will be hosted on a separate repo.
