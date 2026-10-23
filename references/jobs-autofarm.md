# Jobs & money skills — literal autofarm specs

Everything needed to automate each income path, from level 1 to cap. All
module paths, event names, states and world coordinates below were read from
real working farm scripts, not guessed.

Skill data lives at `DataCore.xp.<skill>` and skill trees at
`SkillsList.list.<skill>`; check ownership with
`SkillsList.has_skill(DataCore, "<skill>", "<skillName>")`.
Current job: `player:GetAttribute("Job")` (`"shelf_stocker"`, `"steakhouse_cook"`,
`"janitor"` or nil). Confirmed `DataCore.xp` keys: `total_level`, `stamina`,
`shelf_stocker`, `atm_hacker`, `janitor`, `cook`, `fishing`, `farmer`.
Extra module: `Modules.Game.Jobs.JobUtil`.

## XP curve — the real formula (CONFIRMED)

Same RuneScape curve the game's `level_to_xp` uses; binary-search it to
convert `DataCore.xp.<skill>` → level:

```lua
local function level_to_xp(lvl)
    local e = 2 ^ (lvl / 7)
    return math.floor(0.125 * (lvl*lvl - lvl
        + 600 * ((e - 1.1040895136738123) / 0.10408951367381225)) + 0.5)
end
-- xp_to_level: binary search highest L where level_to_xp(L) <= xp (0..100)
```

| Lvl | XP needed | | Lvl | XP needed | | Lvl | XP needed |
|---|---|---|---|---|---|---|---|
| 1 | 0 | | 30 | 13,367 | | 60 | 273,748 |
| 10 | 1,155 | | 40 | 37,229 | | 70 | 737,635 |
| 20 | 4,473 | | 45 | 61,517 | | 75 | 1,210,429 |
| 25 | 7,845 | | 50 | 101,339 | | 80 | 1,986,076 |
| | | | | | | 99 | 13,034,441 |

Levels are exponential — 70→80 costs more than 1→70 combined. Plan a
bot's skill targets accordingly (e.g. stop fishing at 75 if offshore
payoff is reached).

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

## How an autofarm literally controls the game

Real farms are not "press buttons" — they are a **coroutine state machine**
built from five primitives:

### 1. The loop skeleton (every farm, same shape)

```lua
local FarmRoutine = coroutine.create(function()
    while task.wait(0.2) do            -- tick rate per farm (0.2–1s)
        if not Toggle("MyFarm") then continue end          -- gate
        if not (AssertArgument("EventA") and ...) then     -- remote names
            break                                        -- still exist?
        end
        pcall(function()
            -- one full work cycle; error() anywhere = abort cycle,
            -- next tick restarts clean
        end)
    end
end)
```

- `AssertCharacter()` — re-grabs `LocalPlayer.Character` every cycle
  (death mid-cycle must not nil-crash the loop).
- `AssertArgument("X")` — checks the game's own argument-check table still
  has the remote name; if a patch renames it, the farm disables itself
  instead of firing dead remotes.
- Everything inside `pcall` + hard timeouts (`os.clock()` deadlines) —
  5 s to receive a steak, 15 s to find a grill, 10 s for grill
  assignment. Timeout → `error()` → cycle restarts.

### 2. Movement primitives (how the character "walks")

```lua
Utils.TP(cframe)              -- instant pivot teleport (short hops,
                              -- entering buildings)
Utils.LerpTP(pos, seconds)    -- smooth tween of the character pivot —
                              -- looks like fast walking, not a blink
SecureLerpTeleport(origin, target, studsPerSecond, conditional)
-- THE core mover: interpolates CFrame every Heartbeat at a fixed speed,
-- follows terrain height (shapecast down each step), optional SpeedRamp
-- (accelerates 1 s like a real walk then constant), and evaluates
-- `conditional()` per step — return false mid-path and it aborts
-- (this is how dynamic ATM rerouting works).
SecureLerpTeleportWithPathfinding(...)
-- PathLib.GetPath (PathfindingService) → waypoints → lerp between them,
-- slowing to ~15 % speed within 30 studs of a >35° turn so it looks human.
Utils.LockPosition(pos)       -- Heartbeat pin — holds you at a spot
                              -- (used while stocking at the box pile)
```

Raycast filters for movement exclude: character, `workspace.Vehicles`,
`Map.BoundaryWalls`, `Map.Vegetation.Trees`, `Map.Props.PowerLines`,
`Map.Props.PrestigeMotorsLargeSign`, `Map.RoadNetwork.TrafficLights`,
and non-`NewProps`/non-`FuseBox` props.

```lua
Net.send("sprint", true)      -- real sprint remote, toggled during travel
WaitForPing()                 -- task.wait(max(0.2, ping*2)) after any
                              -- network-touching action
```

Stamina guard: `LocalPlayer:GetAttribute("SprintBar")` 0→1. If < 0.01 the
farm stops moving until ≥ 0.95 (`Storage.MovementSpoof` toggles with it).

### 3. Interaction primitives

```lua
fireproximityprompt(prompt, holdTime)   -- job boards, boxes, fridge,
                                        -- ATM prompt, vehicle DrivePrompt
Net.send("event", ...)                   -- FireServer actions
Net.get("event", ...)                    -- InvokeServer request/response
Obj.states.<name>.get()                  -- reactive world state
SliderMinigame.enabled.hook(fn)          -- fires when a minigame opens
getconnections(button.MouseButton1Click) -- "click" GUI buttons
player:GetAttribute("Job"/"SprintBar"/"LastHackedAtmTimestamp")
```

### 4. Vehicle control

```lua
local seat = vehicle:FindFirstChildWhichIsA("VehicleSeat", true)
local prompt = vehicle.Chassis.DrivePromptAttachment.DrivePrompt
fireproximityprompt(prompt, 0.5)
-- if not seated in 0.5s: nudge the seat upward (velocity jitter) to grab
-- network ownership, then re-fire the prompt.
-- While hacking: Humanoid:ChangeState(Jumping) to un-seat, and
-- vehicle:PivotTo(charPos + yAxis*12) parks it overhead.
```

### 5. Recovery pattern

Every cycle is wrapped so any step can `error()`:

- missing object → `error("...")` → cycle restarts (never hard-crashes)
- `Humanoid.Health == 0` → respawn wait → `apply_for_job` again → resume
- prompt didn't deliver item in 5 s → restart (server dropped it)
- grill/ATM state didn't flip in 10 s → `UnequipTools()` + restart
  (grab desynced)

The farm never trusts a send — it always polls the resulting *state*
(`mopped`, `user_id_assigned`, `hacker`, `Bobber` child, `amount`)
before moving to the next step. That is the whole discipline.

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
6. Pay lands in `hand` — when it crosses a threshold, walk to the
   **in-store ATM** (`...Interior.ATM`) and
   `Net.get("transfer_funds", "hand", "bank", amount)` before restocking
   resumes. Indoor + instant = safest deposit in the game.

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
workspace.Map.Tiles.ShoppingTile.SteakHouse.Interior
  .SteakHouseBeacon            -- job board; Net.send("apply_for_job", it)
  .Fridge."Base" (BasePart) → Attachment → ProximityPrompt   -- take steak
  .ATM                         -- in-store ATM for auto-deposit
Module: Modules.Game.Jobs.SteakhouseCook
  .grill_class.objects[]     -- grills
  .grill_class.get(instance) -- object for a grill instance
  per grill: .states.user_id_assigned.get()  -- 0 = free
             .states.perfect_cook_time.get() -- EXACT seconds for "Perfect"
  grill instance child: .GrillArea (stand position)
Entry checkpoints: (-227, 254.7, 326) → (-233, 254.7, 334) → inside
(-235, 256, 340). Fridge approach goal: (-279, 257, 334).
Nearby ATM waypoints: (-231.9, 256.6, 331.5), (-222.9, 256.6, 331.5),
reference ~(-225, 257, 488)
```

**Autofarm loop (CONFIRMED)**

1. `Job` must be `"steakhouse_cook"` — else teleport the checkpoints and
   `Net.send("apply_for_job", SteakHouseBeacon)`.
2. If held tool lacks `IsCookable == true` → path to fridge →
   `fireproximityprompt(FridgePrompt)` → wait ≤5s for a cookable tool
   (also accepted from `Backpack` — item is literally named `"Steak"`).
3. Get a free grill: `Net.get("get_random_free_grill")` returns a grill
   instance → `grill_class.get(instance)`; or scan objects for
   `user_id_assigned.get() == 0` and take the nearest. Retry ≤15 s if
   all grills are busy.
4. Path to `grill.GrillArea:GetPivot()` →
   `Net.send("start_grilling", grillInstance)`.
5. Poll ≤10 s until `user_id_assigned.get() == LocalPlayer.UserId`, then
   read `perfect_cook_time.get()` (if it never assigns → UnequipTools +
   restart — the grab desynced).
6. `task.wait(perfectTime - 0.2)` →
   `Net.send("finish_grilling", grillInstance, "Perfect")` — perfect cook
   every cycle; `perfect_cook_time` is server-driven, so this stays exact
   as patches shift.

**Complete cycle — literal pseudocode**

```lua
-- one cycle = one steak cooked + sold; wrapped in pcall by the loop
if player:GetAttribute("Job") ~= "steakhouse_cook" then
    SecureLerpTeleport(char:GetPivot(), CFrame.new(-227,254.7,326), speed)
    LerpTP(Vector3.new(-233,254.7,334), 3)
    TP(Vector3.new(-235,256,340))                 -- enter building
    Net.send("apply_for_job", SteakHouseBeacon)
    WaitForPing()
end

if not (held and held:GetAttribute("IsCookable")) then
    local path = PathLib.GetPath(charPos, Vector3.new(-279,257,334))
    for _, wp in path:GetWaypoints() do           -- walk the fridge path
        Net.send("sprint", true)
        LerpTP(wp.Position + Vector3.yAxis*3,
               PathLib.GetTravelTime(dist, 60))
    end
    Net.send("sprint", false)
    fireproximityprompt(fridgePrompt)
    waitUntil(function()                          -- 5s deadline
        return heldToolIsCookable() or backpackHas("Steak")
    end, 5)
end

local grill = repeat getFreeGrill() until grill   -- ≤15s
walkPathTo(grill.instance.GrillArea:GetPivot())
Net.send("start_grilling", grill.instance)

local perfect
waitUntil(function()                              -- ≤10s
    if grill.states.user_id_assigned.get() == myUserId then
        perfect = grill.states.perfect_cook_time.get()
        return perfect and perfect > 0
    end
end, 10) or error("grill desync")

task.wait(math.max(perfect - 0.2, 0.1))
Net.send("finish_grilling", grill.instance, "Perfect")
WaitForPing()
-- cycle repeats; pay lands in hand; ATM-deposit when over threshold
```

**Progression**: skillets shorten cook time; special steaks raise pay
(buy at the shop / Jack's Hardware as the skill unlocks them).

---

## Janitor — Burger Place

**Module (CONFIRMED)**: `require(Modules.Game.Jobs.Janitor)`
(+ `Modules.Game.Jobs.JobUtil` for shared job helpers)

```lua
Janitor.class.objects[]  -- puddles
  .instance              -- the puddle BasePart
  .states.mopped.get()   -- true = already cleaned
  .states.mop_length.get() -- seconds the mop takes on this puddle
```

**Job apply spots (CONFIRMED)**: (109.1, 257.8, -243.6) and
(110.6, 255.2, -309.6) — the Burger Place interior checkpoints.

**Autofarm loop (CONFIRMED)**

1. `Job == "janitor"`; hold a tool whose name contains `"mop"` or has the
   `Mop` tag — equip from `Backpack` if needed
   (`hum:EquipTool(bpTool)`).
2. Nearest puddle where `not mopped.get()` → path to within ~6 studs →
   wait `mop_length.get() + 0.3`.
3. No puddles left → `task.wait(1)` and rescan (they respawn).

**Movement tuning used by real farms**: WalkSpeed ~25, waypoint spacing
2, reach distance 4, forward-obstacle raycast 5.5 studs (doors excluded
— walk through), re-path every ~3.5 s, stuck = no progress 0.2 studs in
2.5 s.

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
