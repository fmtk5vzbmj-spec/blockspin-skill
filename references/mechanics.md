# BlockSpin — Gameplay Mechanics

Core loop (official description): work jobs -> unbox random reward cases ->
level up -> fight to become top of the city. Full-loot PvP everywhere.

## Death & loss rules

- Death drops the ENTIRE carried inventory; house safe contents survive.
- Carried cash mostly drops (~72% drops / 8% burned / ~20% kept, community).
- Leaving during combat also drops items.
- ATM deposits secure cash. Bank cash before fighting or hacking.

## Jobs (each has an independent skill level, leveled by repetition)

| Job | Location | Mechanic |
|---|---|---|
| Janitor | Burger Place | Mop puddles (`BurgerPlace.Interior.Puddles`, 5s/10s) |
| Cook | The Butcher's Cut (steakhouse) | Fridge prompt + ProgressBar minigame |
| Shelf Stocker | Quick-11 | Stocking tasks |
| Swiper | ATMs map-wide | Hack minigame (stop needle on green, 5 tries, ~$50) |
| Fishing | Shore / level-75 offshore spot near mansions | Rods from Jack's Hardware |
| Farming | (exists, poorly documented) | — |
| Stamina | passive skill | Rises by moving; needed to outrun |

`player:GetAttribute("Job")` returns e.g. `steakhouse_cook`.
Job board = blue circle; briefcase icon marks job sites. Spawn is near the
gas station at the center crossroads (spawns randomize later).

## Combat

- Finishers: timed kill moves granting bonus XP + extra victim loot.
- Blood Bag ($100, Medical Center / Quick-11) heals instantly.
- Energy Shot ($250, Dealer at Basketball Court / SW Nassu St) stamina boost.
- Surgical Gloves ($200) hide fingerprints at crime scenes.
- Players camp job sites and ATMs to rob workers — transfer spots should be
  remote, not town center. Dynamic weather (Sep 2026) changes visibility.

## Crime tools (Illegal Dealer — alley behind Sam's Motel / Butcher's Cut)

| Item | Price | Use |
|---|---|---|
| Basic Hack Tool | $10 | ATM/bank robbery |
| Pro/Ultimate/Quantum Hack Tool | $150/$350/$550 | Better hacks |
| Shiesty mask | $200 | Hide identity |
| Lockpick | $300 | Steal locked cars / break into houses |

Car theft: hold E on driver door -> click-green minigame x3 -> drive off.

## Houses & storage

- Rent via door/"for rent" prompt (mailbox ~$250 in Mar-2025 footage).
- Inside: Safe (protected storage) + Workbench (repairs weapon durability).
- Mansion Underground update: underground rooms + extra safes (high-tier homes).
- Fishing update: additional storage utility + level-75 offshore fishing.

## Vehicles

- Car crates at garages (car icon): Basic $540 / Rare $1,350 / Elite $12,400.
- Fuel at Gas Station, wash at Car Wash. Lockable (Lockpick defeats).
- Tradeable cars: Humvee (~$6.1M), Rezvani (~$2.8M), Stang (~$1M), Mazede 3.
- Untradeable crate cars: Cullivane, Turbolin, Hurricane (networth only).

## Airdrops & crates

- Periodic "Large Crate will drop at <place> in <time>" alerts + orange flare;
  hold E to open (~$200 + items). Heavy PvP competition.
- Weapon crates (knife icon / Patriot Shooting Range): Basic $55, Advanced $110,
  Superior $540, Elite $5,050, Legendary $22,800 (Mar-2025 prices, drifted).
- Everything Crate = top mixed tier, distributed via promo codes.

## Codes (verify in-game; one referral only per account)

- W7C28D — $500 referral (official description; redeem AFTER ~10-15 min or you
  get "Could not validate your account (Alt Account)").
- M16_RELEASE, UNDER_THE_BARREL — weapon/Everything crates.
- BLOCKSPIN_GRIPS_UPDATE — cases + cash (grips/barrel attachments exist).

## Transfer-relevant threats

- No VIP servers — pick the emptiest public server or the New Players shard.
- Looters camp ATMs, job sites, airdrops. Meet somewhere remote (own house).
- Cash drop tax: receiver ~72-73% (drop ~$137K to net $100K).
- Untradeable items exist (wraps, sights, grips, barrels, rims, spoilers,
  glows, some crate cars) — they cannot be dropped at all.
- Alt-account detection exists at least for codes; brand-new accounts may be
  flagged/routed to the New Players shard.

## Scripted transfer (confirmed internals)

- Drop: `Net.send("drop_item", guid, amount)` — the item must be in a
  carried location ("hand"); safe-stored items can't be dropped anyway.
- Ground loot: `workspace.DroppedItems.<itemName>` Model with `PickUpZone`.
- Pickup: `Net.get("pickup_dropped_item", itemModel)` and/or touching the
  PickUpZone (`firetouchinterest` ~25×).
- Bank cash: `Net.get("transfer_funds", "hand", "bank", amount)`.
- See internals.md for the counter protocol and the full event table.
