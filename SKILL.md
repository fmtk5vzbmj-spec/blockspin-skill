---
name: blockspin
description: Complete knowledge base for the Roblox game BlockSpin (PlaceId 104715542330896) - game mechanics, death-drop rules, map, jobs, weapon/item values, the internal networking layer (ReplicatedStorage.Modules.Core.Net), sub-place shards, anti-cheat, manual drop/pickup trading, account-data inspection, and account/session mechanics (.ROBLOSECURITY cookies, auth tickets, multi-instance launching). Use when building tools, Lua scripts, or automation for BlockSpin, when transferring items between accounts, or when answering questions about the game.
---

# BlockSpin (Roblox) — Agent Knowledge Base

BlockSpin is an open-world crime game by **Cinnamon Go!** (beta, ~1.3B visits).
Root PlaceId `104715542330896`, UniverseId `6765805766`, 28 players per server,
R15 avatars, Florida-county map on a ~6x5 road grid. No private/VIP servers
(`createVipServersAllowed = false`) — the only privacy for item hand-offs is
choosing a near-empty public server.

## The one rule everything hangs on

The official description: **"You drop everything in your inventory on death;
place items into the safe in your house to keep them secure."**

- Death drops 100% of carried items + most carried cash (~72% drops, 8% burned,
  ~20% kept — community-sourced, not confirmed server logic).
- Leaving the game while in combat also drops items (combat-logging).
- House safes (rented via door/"for rent" prompt, ~$250 in older footage) are
  the only protected storage. Mansion Underground update added extra safes in
  high-tier homes; the Fishing update added another storage utility.
- ATMs deposit carried cash into safe storage. Pawn shop buys at ~30% of value.

## No trade window — transfers are physical

Every item hand-off is manual: one player drops the item on the ground and the
other picks it up via a ProximityPrompt (hold-E). Cash drops are taxed
(receiver gets ~72-73%). This is the ONLY exchange mechanism — and it is what
real traders (and marketplaces like zeusx selling "RPG INSTANT DROP") use.

## Sub-place shards — critical for multi-account work

The game has 6 places. Accounts may be routed to different shards, which makes
two clients joined to the "same" JobId still unable to see each other.

| Place | PlaceId | Notes |
|---|---|---|
| Main 🔪 | 104715542330896 | Root place, server list public |
| New Players | 132247642420063 | Live servers with 1-2 players — best privacy |
| PRO | 102036043572089 | No public servers observed |
| VC ONLY | 113269916962108 | Teleport-only (server API 400) |
| BlockSpin+ | 95898900045000 | Teleport-only |
| Server: 2 | 97556409405464 | Teleport-only |

Always verify `game.PlaceId` in-game and that `Players:FindFirstChild(targetName)`
actually resolves before dropping anything.

## Networking internals (CONFIRMED from decompiled client modules)

- Protocol: `Remotes.Send` (RemoteEvent) + `Remotes.Get` (RemoteFunction),
  wrapped by `Net.send`/`Net.get`. Every call carries an incrementing
  counter as arg1 — always go through the Net module, never raw remotes.
- Player state lives in `require(Modules.Core.Data)`:
  `money.hand`, `money.bank`, `items[cat][name] = {guid, amount, location}`,
  `xp.*`. Items are dropped/equipped **by GUID**, not by name.
- CONFIRMED core events:
  - drop: `Net.send("drop_item", guid, amount)`
  - pickup: `Net.get("pickup_dropped_item", droppedItemModel)` — plus
    touching `workspace.DroppedItems.<item>.PickUpZone`.
  - bank: `Net.get("transfer_funds", "hand"/"bank", "bank"/"hand", amount)`
  - equip/spawn: `Net.get("toggle_equip_item", guid)`
  - crates: `Net.get("open_crate", crateOptionsChild, "money")`
- Dropped loot = `workspace.DroppedItems` Models (name = item name,
  `item_type` attribute → `ReplicatedStorage.Items[cat][name]` with
  `RarityName`).
- Map tree: `workspace.Map.Tiles.<TileName>.<Building>.Interior`
  (e.g. `ShoppingTile.SteakHouse.Interior.Fridge`).

## Anti-cheat (real, present)

- Watchdog module `PlayerWellbeingObfuscated` — published bypass hooks every
  function sourced from it to `task.wait(9e9)`.
- Validates hooks via `getfenv`/`debug.traceback`/`debug.info` ("validity_check").
- Monitors `LogService.MessageOut` and `ScriptContext.Error` and reports to the
  server — never let a script error leak inside hooked code.
- The game flags fresh accounts: promo codes return
  "Could not validate your account (Alt Account)" for new alts.

## Accounts, cookies & launching (CONFIRMED)

- Password login via `auth.roblox.com/v2/login` is effectively blocked
  (FunCaptcha) — authenticate with the `.ROBLOSECURITY` cookie.
- Capture the cookie via DevTools (F12 → Application → Cookies), an embedded
  WebView2 pointed at roblox.com/login, or a `user:pass:cookie` paste.
- Launch a client into an exact server: `POST auth.roblox.com/v1/authentication-ticket`
  (cookie + CSRF retry) → `roblox-player:1+launchmode:play+gameinfo:<ticket>
  +placelauncherurl:...&gameInstanceId=<jobId>` → visible Roblox window.
- Multi-client on one machine: close the `ROBLOX`-named mutant handles inside
  running `RobloxPlayerBeta` processes before each additional launch.
- Full details: [references/accounts-sessions.md](references/accounts-sessions.md)

## What is NOT publicly known (verify in-game before trusting)

- The house-safe store event name/args — spy `Net.get`/`Net.send` while
  manually depositing a cheap item in a rented house safe.
- Whether `location` fields on `DataCore.items` distinguish "hand" vs
  "safe" cleanly in every category (observed values so far: `hand`).
- How to obtain the rest: run a remote spy that hooks every function field
  on the Net module + `send`-named functions found in module upvalues,
  plus a `__namecall` fallback logging `FireServer`/`InvokeServer`. Then
  manually drop/store a cheap item and read the captured names.

## Safe transfer workflow (cheap-item first)

1. Run an account-data dump + remote spy on a test account holding a CHEAP item.
2. Drop the item manually; capture the real event + args.
3. Hardwire the verified event; test source-drop → replicated ground loot →
   target pickup → target inventory, end to end.
4. Only then handle valuable items (RPG ~$175K trade value; durability ~96 —
   half-durability items sell ~60%). Never test with the valuable item itself.
5. Prefer the New Players shard or the emptiest public server; check both
   clients report the same PlaceId and the source sees the target player.

## References

- [references/mechanics.md](references/mechanics.md) — jobs, combat, houses,
  vehicles, ATM hacks, economy, codes, threats during transfers.
- [references/jobs-autofarm.md](references/jobs-autofarm.md) — literal
  per-job automation specs: shelf stocker, cook (perfect-cook trick),
  janitor, ATM hacker tool tiers + win event, fishing 1→80 with bait/rod
  internals, farmer notes, money-safety patterns, buying/selling map.
- [references/map.md](references/map.md) — full location list with purposes.
- [references/items-values.md](references/items-values.md) — weapons, rarities,
  trade values, durability, untradeable items.
- [references/internals.md](references/internals.md) — module/event details,
  anti-cheat techniques, drop/pickup/store strategy for scripts.
- [references/accounts-sessions.md](references/accounts-sessions.md) —
  .ROBLOSECURITY capture, auth tickets, protocol launch URL, server picking,
  multi-instance mutex handling, account-safety notes.
