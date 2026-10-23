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

Two parallel ways to find every ATM on the map:

```lua
-- Module route: richer state
ATM_Module.class.objects[]       -- Modules.ATM.ATM
  .instance                      -- the world Model
  .states.disabled.get()         -- true = on cooldown/empty
  .states.hacker.get()           -- Player currently hacking (or nil)
  .states.active_hack_tool.get() -- tool name mid-hack

-- World route: Models literally named "ATM" under workspace.Map.Props
for _, v in workspace.Map.Props:GetChildren() do
    if v.Name == "ATM" and not v:GetAttribute("disabled")
       and not v:FindFirstChildWhichIsA("ObjectValue").Value then
        -- v has child parts; each carries a ProximityPrompt
    end
end
```

ATM interiors exist too — e.g. Quick-11 has `.Interior.ATM` — but
indoor machines are worse farm targets (wall clips, players camp them).

**Enclosed-ATM detection (CONFIRMED trick)**

An ATM inside a building can't take a hover/teleport approach. Detect by
raycasting straight down onto it:

```lua
local params = RaycastParams.new()
params.FilterType = Enum.RaycastFilterType.Exclude
params.FilterDescendantsInstances = { atm, char, workspace.Map.RoadNetwork }
params.RespectCanCollide = true
local hit = workspace:Raycast(atm:GetPivot().Position + Vector3.yAxis*20,
                              Vector3.yAxis*-20, params)
if hit then skip = true end  -- roof overhead → enclosed, skip
```

**Hack flow — legit path (CONFIRMED)**

1. Stand next to the machine — interaction side is
   `atm:GetPivot() + atm:GetPivot().RightVector * -3.5`.
2. `fireproximityprompt(prompt)` on the ATM child part.
3. `PlayerGui.ATM.ATMHolder` opens → click
   `ATMHomePage.Title.ATMHackButton`
   (`getconnections(btn.MouseButton1Click)` fires it without a mouse).
4. `PlayerGui.SliderMinigame.SliderMinigameFrame` appears —
   `Bar.Needle` sweeps, stop it inside `Bar.Target`.
5. Or skip the needle: `Net.send("minigame_win", atmInstance)`.

**Hack flow — exploit path (CONFIRMED)**

```lua
Net.send(<hack-request>, atmInstance, "HackToolBasic") -- claims the ATM
task.wait(1.5 + jitter)                                -- looks human
Net.send("minigame_win", atmInstance)                  -- instant win
```

The request event name is resolved at runtime from the game's own
argument-check table (`ArgumentChecks.Request`); `"minigame_win"` is the
literal confirmed win string. While claimed, `states.hacker == LocalPlayer`
and `states.active_hack_tool` holds the tool name.

**Cooldown / rerouting (CONFIRMED)**

- After a win the server stamps
  `player:GetAttribute("LastHackedAtmTimestamp")` — keep ≥5 s between
  hack sends or they get eaten.
- `disabled.get()` flips true on an emptied/hacked ATM until restock.
- Dynamic rerouting: re-scan `ATM_Module.class.objects` for the closest
  unhacked machine; swap target only if >15% closer (prevents ping-pong).
- Vehicle rotation trick: leave the car parked ~12 studs above your
  position (`PivotTo(charPivot + Vector3.yAxis*12)`) — it stays loaded,
  you un-seat to hack, re-seat to move on.

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

Bait is tracked as the rod's ammo. When `ammo_amount <= 0`, call
`rod:Activate()` — activating the tool reloads bait from inventory —
then `task.wait(0.5)` and re-check.

**Cast + catch loop (CONFIRMED — auto-catch)**

```lua
-- WATER DETECTION: any water works — raycast finds it, no spot list needed.
-- Aim 20 studs in front of the character, cast 100 studs down:
local origin = hrp.Position + hrp.CFrame.LookVector * 20
local hit = workspace:Raycast(origin, Vector3.new(0,-100,0), params)
local waterPos = (hit and hit.Material == Enum.Material.Water
                  and (hit.Position - hrp.Position).Magnitude < 60)
                 and hit.Position or nil

-- CAST: "throw_rod" goes through Net.get (InvokeServer), not send.
-- A successful cast parents a "Bobber" child onto the rod tool —
-- Bobber exists = line is in the water, don't recast.
if waterPos and not rod:FindFirstChild("Bobber") then
    Net.get("throw_rod", rod, waterPos)
end

-- CATCH: hook the slider minigame's enabled state. The instant a fish
-- bites and the minigame opens, fire reel_ended with success=true —
-- you never see the slider.
SliderMinigame.enabled.hook(function(isOpen)
    if isOpen then
        Net.send("reel_ended", rod, true)
    end
end)
```

Full loop timing: poll once per second — check rod equipped → check bait
(activate to reload) → if no `Bobber`, raycast + throw. The bite and the
reel handle themselves through the minigame hook.

**Where to fish — best spot by level**

Water is detected by raycast (`Material.Water`), not a fixed list — but
ranked by payout/safety:

1. **Best overall: dock behind the Car Dealership** (118, 260, 431).
   Shallow pond, off the main road, far from spawn traffic — the standard
   quiet grind spot for levels 1–45.
2. **Best for selling: sea edge by the Pawn Shop** (1177, 258, -535).
   Fish at the beach, walk inside, sell — no cross-map trips. Pawn is the
   far corner so almost nobody passes through.
3. **Ponds around the house blocks** — fine mid-level; closer to players,
   watch for robbers.
4. **Level 75+: offshore deep-sea by the Mansions** — the Fishing Update
   zone; highest-tier fish (sailfish/marlin class). Go with a vehicle for
   the long return and a safe to store rods; dying out there drops your rod.

**Level path**

- 1–~45: any reachable water — ponds/docks, cheap rod + cheap bait.
- ~45–75: better rods/bait as skills unlock; same loop, higher fish value.
- 75+: deep-sea offshore near the Mansions.
- Sell fish at the Pawn Shop — store the cash at an ATM on the way back.

---

## Farmer

Least-documented skill. Confirmed only: `farmer` exists in `DataCore.xp`,
a `farming` item category exists in `ReplicatedStorage.Items`, and farming
tools are sold at Jack's Hardware. Spy `Net.*` while farming manually for
the real event names before automating.

---

## Getting around — vehicles for farming

- Cars/bikes are items (`DataCore.items.car`/`.bike`); spawn yours by
  equipping it (`Net.get("toggle_equip_item", guid)`), then enter via
  `Chassis.DrivePromptAttachment.DrivePrompt`.
- Speed lives on the model: `Motors:GetAttribute("forwardMaxSpeed")` —
  pick your fastest owned vehicle by comparing that attribute across
  `DataCore.items.car` + `.bike` names.
- ATM rotation trick: park the car ~12 studs overhead
  (`PivotTo(charPivot + Vector3.yAxis*12)`) so it stays loaded while you
  hop out to hack; re-seat via the DrivePrompt to move on.
- No car? Lockpick ($300, illegal dealer) opens parked ones — or buy a
  Basic car crate ($540) at the dealership; bikes at Urban Cycles
  (256, 261, -206) are the cheapest wheels.

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
