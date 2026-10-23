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

## Shop prices (community-sourced — verify in-game)

Weapon crates (Patriot Shooting Range, knife icon — random gun per tier):

| Crate | Price |
|---|---|
| Basic | $55 |
| Advanced | $110 |
| Superior | $540 |
| Elite | $5,050 |
| Legendary | $22,800 |

Confirmed open path: `Net.get("open_crate", crateOptionChild, "money")` —
options under `GunShopTile.PatriotWeapons.Interior.Crates["<Crate>"].CrateOptions`.

Dealer items (Basketball Court, SW Nassu St): Surgical Gloves $200
(hides fingerprints), Energy Shot $250 (stamina).

Illegal dealer (alley behind Sam's Motel / Butcher's Cut):

| Item | Price | Use |
|---|---|---|
| Basic Hack Tool | $10 | ATM/bank hacks, entry tier |
| Pro Hack Tool | $150 | higher payout/success |
| Ultimate Hack Tool | $350 | higher still |
| Quantum Hack Tool | $550 | top tier, LIMITED stock (nightclub shop) |
| Lockpick | $300 | steal vehicles / enter houses |
| Shiesty | $200 | mask — hides identity while robbing |

Payout reference: Basic ≈$50/ATM, Pro ≈$250/ATM (old guides; drift).

## Vehicles

| Vehicle | Value | Durability |
|---|---|---|
| Humvee | ~$6.1M | ~2700 |
| Rezvani | ~$2.8M | ~2700 |
| Stang | ~$1M | ~1500 |
| Mazede 3 | ~$40K | ~1500 |

### Getting vehicles (community-priced)

You can't pick a specific car — you buy a **car crate** at a garage
(car icon on the map; Car Dealership 118, 260, 431, Prestige Motors lot)
or steal parked cars with a **Lockpick ($300**, illegal dealer).

| Crate | Price | Known contents |
|---|---|---|
| Basic Car Crate | $540 | Mower, Civic, Go Kart, DirtBike, Forklift |
| Rare Car Crate | $1,350 | F-150, ATV, Go Kart, DirtBike, better commons |
| Elite Car Crate | $12,400 | Urus, Trackhawk, G-Cruzer, premium bikes |

Bikes/skateboards are sold at **Urban Cycles / Bike Shop** (256, 261, -206).

### Vehicle internals (CONFIRMED from client)

```lua
-- Catalog: ReplicatedStorage.Items.car + ReplicatedStorage.Items.bike
--   Model templates, each with a "Motors" folder.
--   Motors:GetAttribute("forwardMaxSpeed") = top speed → rank vehicles
--   by sorting templates on this attribute (real farm does this).
-- Spawned cars live under workspace.Vehicles with OwnerUserId attr —
--   yours = vehicle:GetAttribute("OwnerUserId") == LocalPlayer.UserId.
-- Enter: Vehicle.Chassis.DrivePromptAttachment.DrivePrompt (ProximityPrompt)
-- Module: require(Modules.VehicleSystem.Vehicle)
-- Cars are DataCore.items.car / .bike entries (guid, location) —
--   spawning = equipping the item (toggle_equip_item, same as tools).
-- Vehicle crates open like weapon crates:
--   Net.get("open_crate", crateOptionChild, "money")
--   options live under the dealership Interior's Crates folder.
```

Car customization (paint, rims, suspension) is cosmetic-only — mods draw
attention but add no speed.

## Untradeable (cannot be dropped — networth only)

- Crate cosmetics: gun/vehicle wraps, sights, grips, barrels, rims, spoilers, glows.
- Some crate cars: Cullivane, Turbolin, Hurricane (rare ones ~$100M networth).
- If someone "sells" these, it's a scam.

## Item identity notes for scripts

- Items may carry attributes such as durability and a unique ID — read
  `item:GetAttributes()` before assuming `Name` identifies the item.
- Weapons degrade with use; a Workbench in the house repairs durability for cash.
- Crate-tier items can also be looted from killed players.
