# BlockSpin — Items, Weapons & Trade Values

Community-maintained (blockspinvalues.com, driffle list Oct-2026). Values drift
weekly; durability scales price (half durability ~60%). Pawn pays ~30%.

## Weapon trade values

| Tier | Items | Approx. value |
|---|---|---|
| Omega | M249 | $9M |
| Omega | Anaconda | $1.3M |
| Legendary | Pumpkin Launcher | $800K (gamepass variant ~$3M) |
| Legendary | **RPG** | **~$175K** (durability 96; pawn ~$45K) |
| Legendary | M16 | $100K |
| Legendary | AK47, Remington ($55K), MP5, Crossbow | — |
| Epic | Firework Launcher, M24, Draco, Skorpion, Double Barrel | — |
| Rare | Uzi, Sawnoff, P226, Glock, Hunting Rifle | — |
| Misc/exclusive | Emergency Care Kit (~$3M), Bandage (~$1M), Brainrot Slapper ($50K) | — |
| Halloween | Pumpkin Bomb x5 ($20K), Golden Pumpkin Seeds ($50K) | — |

Melee/common: Baseball Bat (~$900), Sledge Hammer ($6K, pawn $7.5K), C9.

## Vehicles

| Vehicle | Value | Durability |
|---|---|---|
| Humvee | ~$6.1M | ~2700 |
| Rezvani | ~$2.8M | ~2700 |
| Stang | ~$1M | ~1500 |
| Mazede 3 | ~$40K | ~1500 |

## Untradeable (cannot be dropped — networth only)

- Crate cosmetics: gun/vehicle wraps, sights, grips, barrels, rims, spoilers, glows.
- Some crate cars: Cullivane, Turbolin, Hurricane (rare ones ~$100M networth).
- If someone "sells" these, it's a scam.

## Item identity notes for scripts

- Items may carry attributes such as durability and a unique ID — read
  `item:GetAttributes()` before assuming `Name` identifies the item.
- Weapons degrade with use; a Workbench in the house repairs durability for cash.
- Crate-tier items can also be looted from killed players.
