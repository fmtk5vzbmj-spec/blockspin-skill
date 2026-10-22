# BlockSpin — Client Internals (for script/tool authors)

Sources: decompiled client `Net` module, leaked "sasware" client source,
public farming/PvP scripts that call real game remotes, and the
`Modules.Core.Data` API used by account-logger scripts.
Names below marked **CONFIRMED** appear verbatim in real client code.

## Networking layer — CONFIRMED protocol

```lua
-- ReplicatedStorage.Remotes has exactly two children:
local Send   = Remotes.Send   -- RemoteEvent
local Get    = Remotes.Get    -- RemoteFunction

-- The real Net module (decompiled):
local counters = { event = 0, func = 0 }

function Net.send(name, ...)
    counters.event += 1
    Send:FireServer(counters.event, name, ...)   -- arg1 = anti-replay counter
end

function Net.get(name, ...)                    -- NOTE: "get", not "invoke"
    counters.func += 1
    return Get:InvokeServer(counters.func, name, ...)
end
```

- Always call `Net.send` / `Net.get` — never the raw remotes (the counter
  must increment or the server rejects the call).
- Server→client traffic: `Send.OnClientEvent` dispatches by event name into
  client-registered hooks; `Get.OnClientInvoke("connect")` is the handshake;
  the client registers a `"ping"` hook returning `true` (heartbeat check).
- The counter table is reachable via
  `debug.getupvalue(require(Modules.Core.Net).get, 1)` if ever needed.
- `Net.send("issue", "...")` is the client's own error-report event —
  the game phones home script errors through it.

## Data module — CONFIRMED player-state schema

```lua
local DataCore = require(ReplicatedStorage.Modules.Core.Data)

DataCore.money.hand          -- cash on hand
DataCore.money.bank          -- banked cash (safe from death-drop)
DataCore.xp.total_level      -- raw xp numbers; convert with formula below
DataCore.xp.stamina / shelf_stocker / atm_hacker / janitor / cook / fishing / farmer
DataCore.items[category][itemName] = { item1, item2, ... }
-- categories include: gun, melee, misc, car, bike, ...
-- each item entry: { guid = "...", amount = n, location = "hand"/"safe"/... }
```

XP → level (reversed from a logger script):

```lua
local C1, C2 = 1.1040895136738123, 0.10408951367381225
local function level_to_xp(l)
    return math.floor(0.125 * (l*l - l + 600 * ((2^(l/7) - C1) / C2)) + 0.5)
end
local function xp_to_level(xp)
    local lo, hi = 0, 100
    while lo <= hi do
        local m = math.floor((lo + hi) / 2)
        if level_to_xp(m) <= xp then lo = m + 1 else hi = m - 1 end
    end
    return hi
end
```

## Event names — CONFIRMED / high-confidence

| Call | Channel | Args | Source |
|---|---|---|---|
| `drop_item` | send | `guid, amount` | real client drop path (dupe + drop-all code) |
| `pickup_dropped_item` | get | `droppedItem Model` | item-aura code |
| `transfer_funds` | get | `"hand"/"bank", "bank"/"hand", amount` | ATM deposit/withdraw |
| `toggle_equip_item` | get | `guid` | equip item / spawn vehicle |
| `open_crate` | get | `CrateOptions child, "money"` | gun-crate opener |
| `request_respawn` | send | — | respawn request |
| `respawn` | send | — | respawn confirm |
| `death_screen_request_respawn` | get | — | death screen |
| `melee_attack` | send | `tool, hitPlayers{}, pivot, factor` | silent-aim hook |
| `shoot_gun` | send | `gun, ?, aimCFrame, ...` | silent-aim hook |
| `throw_hit` | send | `target, part, cframe` | fist combat |
| `throw_item` | send | `tool, position, velocity` | thrown weapons |
| `run_over` | send | `car, target, assembly` | vehicle kills |
| `start_grilling` | send | `grillInstance` | cook job |
| `finish_grilling` | send | `grillInstance, "Perfect"` | cook job |
| `player_started_stocking_shelf` | send | `shelf` | stocker job |
| `player_stocked_shelf` | send | `shelf` | stocker job |
| `apply_for_job` | send | `beacon` | job boards |
| `purchase_consumable` | get | `shopPromptPart, itemName` | shop buys |
| `redeem_referral_code` | get | `code` | referral codes |
| `win` | send | `atmInstance` | ATM hack success |
| `throw_rod` | get | `rod, waterPosition` | fishing |
| `reel_ended` | get/send | `rod, true` | fishing |
| `change_accessory` | send | `category, accessoryName, bool` | character creator |
| `enter_character_creator` | send | `pivot` | creator |
| `leave_character_creator` | send | — | creator |
| `lock_vehicle` | send | `vehicle, true` | car locks |
| `end_ragdoll_early` | send | — | ragdoll |
| `clear_ragdoll` | send | — | ragdoll |
| `sprint` / `set_sprinting_1` | send | `bool` | sprint toggle (both seen) |
| `replicate_billboard_gui` | send | — | high-frequency; don't spam |
| `replicate_stamina_bar` | send | — | high-frequency; don't spam |
| `invalid_entry` | send | — | server rejects bad input |
| `issue` | send | `errorMsg` | client error report |
| `claim_quest` | send/get | `?` | quests (name only) |
| `get_random_free_grill` | get | `?` | cook job (name only) |
| `fix_desync_hip_height` | get | `?` | desync fix (name only) |

NOT yet confirmed: the house-safe store event name. Capture it by spying
`Net.get`/`Net.get`-equivalent calls while manually depositing an item.

## Dropped loot — CONFIRMED structure

```lua
workspace.DroppedItems            -- folder of Model children
-- each Model: name = item name ("Money" for cash)
--   PickUpZone            -- BasePart, the real pickup trigger (touch)
--   AmountBillboardGui    -- cash amount label (money drops)
--   item_type (attribute) -- category key into ReplicatedStorage.Items
```

Pickup that works:

```lua
Net.get("pickup_dropped_item", itemModel)                 -- remote path
for i = 1, 25 do
    firetouchinterest(item.PickUpZone, hrp, 1)
    firetouchinterest(item.PickUpZone, hrp, 0)            -- touch path
end
```

## Item catalog

```lua
ReplicatedStorage.Items.<category>.<itemName>   -- template Models
-- categories observed: gun, melee, car, bike, misc
-- attributes: RarityName ("Common/Uncommon/Rare/Epic/Legendary/Omega")
-- vehicles have a Motors child: acceleration, forwardMaxSpeed attributes
ReplicatedStorage.Items[item:GetAttribute("item_type")][item.Name]  -- dropped item → template
```

- Guns are `Tool` instances with attributes:
  `Accuracy, Automatic, FireRate, Range, Recoil` (+ `AmmoType` → gun vs melee).
- Melee tools: `ConeAngle, Range` attributes; `Tool.states.range.get()` reads live value.
- Consumable items (steaks, etc.): `IsCookable` attribute.
- `Modules.Game.Inventory.ItemUtils.get_item_rarity_price(name)` → price,
  `get_item_info(dataModule, guid)` → item info by GUID.

## Module tree (ReplicatedStorage.Modules)

```
Core/
  Net        -- send/get (see protocol above)
  Data       -- money/items/xp live player state
  Char
  UI         -- UI.get("FrameName") named-element registry
  UIFX       -- scale_open / scale_close
Game/
  Inventory/ItemUtils
  ItemTypes/{Gun, Melee}
  CrateSystem/Crate      -- .spin (crate animation)
  Sprint                 -- sprint_bar upvalue: consume_stamina, update
  Jobs/SteakhouseCook    -- grill_class.objects; states.user_id_assigned.get()
  ATM/ATM
  VehicleSystem/Vehicle
  Skills/SkillsList
  Minigames/SliderMinigame
  Emotes/{EmotesList, EmotesUI}
  UI/{BuyPromptUI, NotificationsUI, SubMenuBackground}
  GameInfo/Updates
```

- `BuyPromptUI.drop_request_with_slider(...)` — the inventory UI calls this
  to drop; its callback eventually sends `drop_item`. `BuyPromptUI.enabled.set(false)`
  closes the prompt. `UI.get("SellPromptSlider" / "ItemBuyHolder" / "SellPromptButtons" / "BuyPromptButtons")`.
- `SteakHouseModule.grill_class.objects[i].states.user_id_assigned.get()` — reactive
  state pattern: `.states.<name>.get()/.set()` used across game classes.

## World layout — CONFIRMED

```lua
workspace.Map.Tiles.<Tile>.<Building>.Interior.<Props>
-- e.g. ShoppingTile.SteakHouse.Interior.Fridge (Base → Attachment → ProximityPrompt)
--      GunShopTile.PatriotWeapons.Interior.Crates["Weapon Crate"].CrateOptions.Superior
workspace.Vehicles      -- spawned cars; OwnerUserId attribute, Motors child
workspace.DroppedItems  -- dropped loot (see above)
workspace.Beacon.Base   -- job beacon
```

Player nametag: `Character.HumanoidRootPart.CharacterBillboardGui.PlayerName`
+ `LevelImage.LevelText` (reads `player:GetAttribute("level")`).

Teleport coordinates (from leaked Teleport_Positions):

| Place | Vector3 |
|---|---|
| Bike Shop | 256, 261, -206 |
| Burger Place | 113, 260, -212 |
| Gun Shop | -140, 259, -221 |
| House | 124, 261, -393 |
| Hospital | -133, 261, -555 |
| Dealer | -222, 255, 388 |
| Car Dealership | 118, 260, 431 |
| Gas Station | 125, 258, 137 |
| Pawn Shop | 1177, 258, -535 |
| Spin Club | 1253, 260, -418 |

## Player attributes

`Job` (e.g. `"steakhouse_cook"`), `level`, `StaminaRegen`, `MaxStamina`,
`StaminaConsumeMultiplier`, `IsDead`, `IsConnected`.

## Anti-cheat (PlayerWellbeingObfuscated)

Published bypass patterns:

```lua
for _, v in pairs(getgc(true)) do
    if typeof(v) == "function" and islclosure(v) then
        local info = debug.getinfo(v)
        if info and info.source and info.source:find("PlayerWellbeingObfuscated") then
            pcall(hookfunction, v, function(...) return task.wait(9e9) end)
        end
    end
end
```

- Hook-detection via `getfenv`/`debug.traceback` ("validity_check") — mask it.
- Error reporting via `LogService.MessageOut` + `ScriptContext.Error`
  connections — disable non-executor connections or your script errors get
  reported to the server.
- Sanity checks: intentionally loads a banned asset to detect hooking.
- A `PlayerWellbeing` body-mover gets added to the character — scripts
  disable its `DescendantAdded` connection.
- IMPORTANT hookfunction pattern (avoids infinite recursion):

```lua
local orig
orig = hookfunction(target, newcclosure(function(...)
    -- log / modify
    if orig then return orig(...) end
end))
```

## Multi-instance joining (WPF/C# side)

- Auth ticket: POST `https://auth.roblox.com/v1/authentication-ticket` with
  `.ROBLOSECURITY` cookie header; ticket goes into
  `roblox-player:1+launchmode:play+gameinfo:<ticket>+placelauncherurl:...`.
- Server list: `GET https://games.roblox.com/v1/games/{placeId}/servers/Public?sortOrder=Asc&limit=N`
  — pick the emptiest server with enough free slots for all accounts.
- Multiple Roblox processes need `ROBLOX_singletonMutex` closed inside the
  existing processes (DuplicateHandle + DUPLICATE_CLOSE_SOURCE), not just
  opened/closed locally.
- `createVipServersAllowed: false` — no private servers; lowest-population
  shard is the only privacy.
