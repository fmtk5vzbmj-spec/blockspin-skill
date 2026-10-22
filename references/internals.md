# BlockSpin — Client Internals (for script/tool authors)

Everything below comes from leaked/published BlockSpin client code and public
exploit scripts. Treat event names as confirmed only where marked.

## Networking layer

```lua
local Net = require(game:GetService("ReplicatedStorage").Modules.Core.Net)
Net.send("request_respawn")          -- confirmed
Net.send("start_emote", emoteIdx)    -- confirmed
Net.send("throw_hit", target, part, cframe) -- confirmed combat event
Net.invoke(...)                      -- exists
```

- The concrete sender may hide as an upvalue:
  `debug.getupvalue(require(Modules.Game.Sprint).loaded, 11).send`.
- Other modules seen: `Modules.Game.Sprint`, `Modules.Game.Emotes.EmotesList`,
  `Modules.Game.Emotes.EmotesUI`, `Modules.Game.Minigames.SliderMinigame`,
  `Modules.Game.ATM.ATM`.
- To discover the real drop/store events, hook: every function field on the
  Net table, every `send|invoke|request|fire|emit`-named function inside
  upvalue tables of required modules, plus a `__namecall` fallback capturing
  `FireServer`/`InvokeServer` with args. Then perform the action manually.

## Anti-cheat (PlayerWellbeingObfuscated)

Published bypass patterns:

```lua
-- Neutralize watchdog functions
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
- IMPORTANT hookfunction pattern (avoids infinite recursion):

```lua
local orig
orig = hookfunction(target, newcclosure(function(...)
    -- log / modify
    if orig then return orig(...) end
end))
```

## Interaction model

- Universal interaction = `ProximityPrompt` (hold E). Airdrops, car locks,
  dropped loot, house doors, safes, job boards all use it.
- Executors: `fireproximityprompt(prompt)`, or
  `prompt:InputHoldBegin(); task.wait(prompt.HoldDuration); prompt:InputHoldEnd()`.
- Dropped loot appears as Models/Parts (often with prompt), not plain Tools.

## Inventory possibilities

Weapons may not be `Tool` instances — scan:

- `player.Backpack`, `player.Character`
- Custom containers under `player` or `PlayerGui`:
  `Inventory`, `Items`, `Weapons`, `Gear`, `Data`, `PlayerData`
- `getgc()` tables referencing the player (real data model often lives there).
- Item identity may be an attribute (`Id`/`ItemId`/`GUID`/`Uid`) rather than Name.

## Drop/store/pickup strategy (unverified until spied)

Layered attempts, verified by "did the item leave inventory":

1. `Net.send` candidate events with variants
   `{item instance, item name, item id, {item=, itemName=, id=}}`:
   - drop: `drop_item, dropitem, drop_weapon, request_drop, inventory_drop, drop, item_drop`
   - pickup: `pickup_item, pickup, collect_item, loot_item, take_item, grab_item, claim_item`
   - store: `store_item, deposit_item, safe_store, put_in_safe, move_to_safe, safe_deposit, stow_item, bank_item`
2. RemoteEvents/RemoteFunctions whose names contain drop/pickup/store keywords.
3. UI path: equip + Backspace (standard Roblox drop), "Drop"/"Deposit"/"Store"
   buttons in PlayerGui, drag-item-out-of-inventory via VirtualInputManager.
4. Verify after each layer: `item:IsDescendantOf(player)` false = success.
   Local `parent = workspace` does NOT replicate — last-resort only.

## Multi-instance joining (WPF/C# side)

- Auth ticket: POST `https://auth.roblox.com/v1/authentication-ticket` with
  `.ROBLOSECURITY` cookie header; ticket goes into
  `roblox-player:1+launchmode:play+gameinfo:<ticket>+placelauncherurl:...`.
- Server list: `GET https://games.roblox.com/v1/games/{placeId}/servers/Public?sortOrder=Asc&limit=N`
  — pick the emptiest server with enough free slots for all accounts.
- Multiple Roblox processes need `ROBLOX_singletonMutex` closed inside the
  existing processes (DuplicateHandle + DUPLICATE_CLOSE_SOURCE), not just
  opened/closed locally.
