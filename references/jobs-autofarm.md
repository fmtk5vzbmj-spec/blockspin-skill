# Jobs & money skills — literal autofarm specs

Everything needed to automate each income path, from level 1 to cap. All
module paths, event names, states and world coordinates below were read from
real working farm scripts, not guessed.

Skill data lives at `DataCore.xp.<skill>` and skill trees at
`SkillsList.list.<skill>`; check ownership with
`SkillsList.has_skill(DataCore, "<skill>", "<skillName>")`.
Current job: `player:GetAttribute("Job")` (`"shelf_stocker"`, `"steakhouse_cook"`,
`"janitor"` or nil). Skills to level: `janitor`, `cook`, `shelf_stocker`,
`atm_hacker`, `fishing`, `farmer`, `stamina`.

Generic pattern every job object uses:

```lua
Obj.instance                 -- the world Model/BasePart
Obj.states.<name>.get()      -- reactive state read (user_id_assigned,
                             -- mopped, mop_length, disabled, hacker,
                             -- perfect_cook_time, ...)
Obj.states.<name>.set(v)     -- client-side set (rarely needed)
Char.held_tool.get()         -- currently equipped Tool or nil
player:GetAttribute("SprintBar")        -- stamina 0..1
player:GetAttribute("LastHackedAtmTimestamp") -- ATM cooldown tracking
```

Job apply: `Net.send("apply_for_job", beaconInstance)` where the beacon is a
real world instance (e.g. `...Interior.Quick11Beacon`), not a GUI click.

---

## Shelf Stocker — Quick-11 (safest indoor grind)

**World tree (CONFIRMED)**

```
workspace.Map.Tiles.GasStationTile.Quick11.Interior
  .Quick11Beacon                 -- job board; Net.send("apply_for_job", it)
  .ShelfStockingJob.Shelves      -- children: BaseParts literally named "Shelf"
  .ShelfStockingJob.NormalBox    -- ProximityPrompt → gives a Box tool
  .DoorSystem.DoorBase           -- CanCollide=false lets you walk through
  .ATM                           -- in-store ATM for auto-deposit
Job checkpoint: Vector3.new(159, 255, 226)   Box area: ~(149, 255, 209)
```

**Autofarm loop (CONFIRMED)**

1. If `Job != "shelf_stocker"` → stand at checkpoint →
   `Net.send("apply_for_job", Quick11Beacon)`.
2. `fireproximityprompt(NormalBox)` ×5 → wait until `Char.held_tool.get()` or
   a Backpack tool named like `Box` exists (equip it if in Backpack).
3. Pick the shelf closest to the beacon.
4. Count boxes: attributes on the held tool whose value matches `"Box"`
   (one Tool can carry a stack — stock count = #Box* attributes).
5. Per box: `Net.send("player_started_stocking_shelf", shelf)` →
   wait `StockingTime` → `Net.send("player_stocked_shelf", shelf)`.

**Stocking time (CONFIRMED formula)**

```lua
base = JobData.job_info.shelf_stocker.stock_shelf_length
mult = 1 + sum(reward_info.speed_percentage_increase for owned skills) / 100
StockingTime = base / mult
```

**Progression**: level `shelf_stocker` → stocking speed ↑ + more boxes per
carry. Skills checked via `SkillsList.has_skill(DataCore, "shelf_stocker", name)`.

---

## Cook — The Butcher's Cut / SteakHouse (perfect cook every time)

**World tree (CONFIRMED)**

```
workspace.Map.Tiles.ShoppingTile.SteakHouse.Interior.Fridge
  └─ "Base" (BasePart) → Attachment → ProximityPrompt   -- take a steak
Module: Modules.Game.Jobs.SteakhouseCook
  .grill_class.objects[]     -- grills
  .grill_class.get(instance) -- object for a grill instance
  per grill: .states.user_id_assigned.get()  -- 0 = free
             .states.perfect_cook_time.get() -- EXACT seconds for "Perfect"
Nearby ATM waypoints: (-231.9, 256.6, 331.5), (-222.9, 256.6, 331.5),
reference ~(-225, 257, 488)
```

**Autofarm loop (CONFIRMED)**

1. `Job` must be `"steakhouse_cook"` (apply at the SteakHouse beacon).
2. If held tool lacks `IsCookable == true` → `fireproximityprompt(FridgePrompt)`
   → wait ≤5s for a cookable held tool.
3. Get a free grill: `Net.get("get_random_free_grill")` returns a grill
   instance → `grill_class.get(instance)`; or scan objects for
   `user_id_assigned.get() == 0` and take the nearest.
4. `Net.send("start_grilling", grillInstance)`.
5. Poll until `user_id_assigned.get() == LocalPlayer.UserId`, then read
   `perfect_cook_time.get()`.
6. `task.wait(perfectTime - 0.2)` →
   `Net.send("finish_grilling", grillInstance, "Perfect")` — perfect cook
   every cycle; `perfect_cook_time` is server-driven, so this stays exact
   as patches shift.

**Progression**: skillets shorten cook time; special steaks raise pay
(buy at the shop / Jack's Hardware as the skill unlocks them).

---

## Janitor — Burger Place

**Module (CONFIRMED)**: `require(Modules.Game.Jobs.Janitor)`

```lua
Janitor.class.objects[]  -- puddles
  .instance              -- the puddle BasePart
  .states.mopped.get()   -- true = already cleaned
  .states.mop_length.get() -- seconds the mop takes on this puddle
```

**Autofarm loop**

1. `Job == "janitor"`; hold a tool whose name contains `"mop"` or has the
   `Mop` tag (Backpack search same rule).
2. Nearest puddle where `not mopped.get()` → move within ~6 studs →
   wait `mop_length.get() + 0.3`.

**Progression**: higher `janitor` level unlocks extra spill spots + faster
mops (mop upgrades at Jack's Hardware).

---

## ATM Hacker ("Swiper") — best money, highest risk

**Buy tools (CONFIRMED)**

```
workspace.ShopZone_Illegal            -- hack-tool shop prompt (alleyway
                                      --  behind The Butcher's Cut)
workspace.ShopZone_IllegalNightclub   -- QUANTUM tool shop (Spin Club)
Net.get("purchase_consumable", shopPart, "HackToolBasic")
Net.get("purchase_consumable", shopPart, "HackToolPro")
Net.get("purchase_consumable", shopPart, "HackToolUltimate")
Net.get("purchase_consumable", shopPart, "HackToolQuantum")
Net.get("purchase_consumable", shopPart, "Shiesty")   -- mask item
```

Tool tiers unlock via `SkillsList.has_skill(DataCore, "atm_hacker",
"HackTool_Basic"/"HackTool_Pro"/"HackTool_Ultimate"/"HackTool_Quantum")`.
Auto-select = highest unlocked tier. Guides: Basic ~$50/hack, Pro (≈skill
12) ~$250/hack.

Stock check (Quantum is LIMITED):

```lua
local product = <any ConsumableItemsInfo folder>:FindFirstChild("HackToolQuantum")
local stock = HttpService:JSONDecode(product:GetAttribute("stock"))
-- {max_purchase, stock_timer}; consumed =
-- StockTimes.get_consumed_amount(name, stock_timer)
```

**ATM discovery (CONFIRMED)**

```lua
ATM_Module.class.objects[]       -- Modules.ATM.ATM
  .instance                      -- Model (also under workspace.Map.Props
                                 --  as "ATM" models)
  .states.disabled.get()         -- true = on cooldown/empty
  .states.hacker.get()           -- Player currently hacking (or nil)
```

**Hack flow (CONFIRMED — minigame can be skipped entirely)**

1. Pick ATM: `not disabled.get()` and `hacker.get() == nil`.
2. Trigger its hack prompt (ProximityPrompt / the `Request` argument) —
   the ATM's `states.hacker` becomes LocalPlayer and SliderMinigame opens.
3. **Skip the slider**: while `states.hacker.get() == LocalPlayer`,
   wait ~2s then `Net.send("win", atmInstance)`
   (`ArgumentChecks.Win` → literal `"minigame_win"` / `"win"` in client code).
4. Collect payout; ATM flips `disabled` until cooldown.
   `player:GetAttribute("LastHackedAtmTimestamp")` updates per hack.

**Safety**: hackers are robbery targets — deposit first
(`Net.get("transfer_funds", "hand", "bank", cash)`), hack near-quiet ATMs,
use a vehicle for rotation.

---

## Fishing — level 1 → 80 path

**Gear**: rods + bait from Jack's Hardware (crossroads shop). Higher rods
follow the Fishing skill. Old videos: ultimate rod ≈lvl 60, ultimate bait
≈lvl 45 — treat as stale; buy what the shop offers your level.

**Internals (CONFIRMED)**

```lua
local FishingRod = require(Modules.ItemTypes.FishingRod)
local rod = FishingRod.equipped_rod.get()          -- equipped rod tool or nil
local guid = rod:GetAttribute("ItemGUID")
local info = ItemUtils.get_item_info(DataCore, guid)
info.ammo_amount                                   -- = remaining bait!
```

When bait runs out: `rod:Activate()` (re-equip / re-bait), re-check.

**Cast + catch loop (CONFIRMED — auto-catch)**

```lua
-- find water: raycast 20 studs ahead, 100 down, Material.Water, <60 studs
local result = workspace:Raycast(hrp.Position + hrp.CFrame.LookVector*20,
                                 Vector3.new(0,-100,0), params)
if result.Material == Enum.Material.Water then waterPos = result.Position end

if not rod:FindFirstChild("Bobber") then
    Net.get("throw_rod", rod, waterPos)     -- cast; creates rod.Bobber
end

-- auto-win the catch minigame:
SliderMinigame.enabled.hook(function(isOpen)
    if isOpen then Net.send("reel_ended", rod, true) end
end)
```

**Level path**

- 1–~45: any reachable water — ponds/docks, cheap rod + cheap bait; popular
  quiet spot is the dock behind the Car Dealership.
- ~45–75: better rods/bait as skills unlock; same loop, higher fish value.
- 75+: deep-sea offshore near the Mansions (Fishing Update) — high-tier
  fish (sailfish/marlin class). Bring storage first; a stolen rod = dead grind.
- Sell fish at the Pawn Shop (far east corner, 1177, 258, -535) — store the
  cash at an ATM on the way back.

---

## Farmer

Least-documented skill. Confirmed only: `farmer` exists in `DataCore.xp`,
a `farming` item category exists in `ReplicatedStorage.Items`, and farming
tools are sold at Jack's Hardware. Spy `Net.*` while farming manually for
the real event names before automating.

---

## Money safety pattern (all farms)

- `Inventory.GetMoney()` → `(cash, bank)`; `Inventory.GetMoneyT().Cash`.
- `Inventory.GetInventoryStats().hand` → `{max, current}` — slot math
  before buying tools.
- Deposit at any enabled ATM: `Net.get("transfer_funds","hand","bank",cash)`.
- Never let a farmed payout ride in `hand` across the map — death drops it.

## Buying/selling map

| Need | Place | World path |
|---|---|---|
| Rods, bait, mops, skillets, farm tools | Jack's Hardware (crossroads) | shop prompt / ConsumableItemsInfo |
| Hack tools Basic→Ultimate | Illegal Dealer, alley behind Butcher's Cut | `workspace.ShopZone_Illegal` |
| HackToolQuantum (limited stock) | Spin Club area | `workspace.ShopZone_IllegalNightclub` |
| Ammo crates, weapon crates | Patriot Shooting Range / Gun Shop | `GunShopTile.PatriotWeapons.Interior.Crates` |
| Fish/loot selling (~30% value) | Pawn Shop, far NE beach (1177,258,-535) | pawn shop interior |
| Illegal consumables (Shiesty etc.) | same alley dealer | `workspace.ConsumableShopZone_Illegal` |
