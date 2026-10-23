# Accounts, cookies & session launching

Everything a tool needs to authenticate Roblox accounts and place multiple
clients on the same BlockSpin server.

## .ROBLOSECURITY — the only reliable credential

Roblox blocks programmatic username/password logins: `auth.roblox.com/v2/login`
demands a FunCaptcha token for almost every non-browser attempt and returns the
misleading error "Username and Password are required." Treat password login as
unusable in tooling.

The `.ROBLOSECURITY` cookie is the real credential — it authenticates every API
call and can mint launch tickets.

### How to capture it (all confirmed working)

1. **Manual (F12)**: log in at roblox.com → DevTools → Application/Storage →
   Cookies → `https://www.roblox.com` → copy `.ROBLOSECURITY`. It starts with
   `_|WARNING:-DO-NOT-SHARE-THIS.--Sharing-this-will-allow-someone-to-log-in...`
2. **Embedded browser (best UX)**: host a WebView2/WebBrowser control on
   `https://www.roblox.com/login`, let the user log in normally (captcha works
   because it is a real browser), then poll the cookie jar:

   ```csharp
   var cookies = await webView.CoreWebView2.CookieManager
       .GetCookiesAsync("https://www.roblox.com");
   var c = cookies.FirstOrDefault(c => c.Name == ".ROBLOSECURITY")?.Value;
   ```
3. **Combo paste**: accept `username:password:_|WARNING...` — split on the
   first two `:` only (the cookie itself contains `:`). Also accept bare
   `_|WARNING...` and `.ROBLOSECURITY=_|WARNING...` pastes.

Validate immediately: `GET https://users.roblox.com/v1/users/authenticated`
with header `Cookie: .ROBLOSECURITY=<value>` → `{id, name, displayName}`.

**Security**: a cookie grants full account access without a password. Treat
like a password — never log, never commit, warn users not to share pastes.

## Launching an account into a specific server

1. `POST https://auth.roblox.com/v1/authentication-ticket` with the cookie +
   `Referer/Origin: https://www.roblox.com`. First call returns 403 +
   `x-csrf-token`; retry once with `X-CSRF-TOKEN`. Response:
   `{ "authenticationTicket": "..." }`.
2. Build the protocol URL and let the shell launch it — Roblox opens a normal
   **visible** window (not headless):

   ```
   roblox-player:1+launchmode:play
     +gameinfo:<url-encoded ticket>
     +launchtime:<unix ms>
     +placelauncherurl:<url-encoded https://www.roblox.com/game/PlaceLauncher.ashx?request=RequestGameJob&browserTrackerId=N&placeId=PID&gameInstanceId=JOBID&isPlayTogetherGame=false>
     +browsertrackerid:N
     +robloxlocale:en_us+gamelocale:en_us
   ```
   `request=RequestGame` (any server) or `RequestGameJob` + `gameInstanceId`
   (exact server).

3. Bring the window forward once loaded: the process is `RobloxPlayerBeta.exe`;
   wait until `WorkingSet > ~200MB` (load heuristic), then `ShowWindow(SW_RESTORE)`
   + `SetForegroundWindow` on `MainWindowHandle` (it is `0` until the window
   actually appears — retry for ~10s).

## Picking the server (low-witness transfers)

`GET https://games.roblox.com/v1/games/{placeId}/servers/Public?sortOrder=Asc&limit=100`
→ pick the server with most free slots (`maxPlayers - playing >= accounts`).
Always join the SAME `placeId` + `jobId` for both accounts, then verify in-game:
`game.PlaceId` matches and `Players:FindFirstChild(targetName)` resolves.
Prefer the New Players shard (`132247642420063`) — live servers with 1–2
players exist there.

## Multi-instance on one machine

Roblox enforces one client per machine with a named **mutex/mutant** containing
`ROBLOX` in its name. To run several clients:

1. Launch instance 1.
2. Enumerate system handles (`NtQuerySystemInformation`,
   SystemExtendedHandleInformation = 64), find handles owned by
   `RobloxPlayerBeta.exe`, filter `Type == "Mutant"` and name containing
   `ROBLOX` + `Mutex`.
3. `DuplicateHandle` them with `DUPLICATE_CLOSE_SOURCE` → mutex closed.
4. Launch instance 2+ (each also opens a `RobloxPlayerLauncher` →
   `RobloxPlayerBeta`).

## Account safety notes

- The game rejects fresh alts: promo-code redemption can return
  "Could not validate your account (Alt Account)". Use accounts with some
  play history.
- Accounts may be routed to different sub-places on join — always confirm the
  landed `game.PlaceId` before dropping anything.
- Logging out of roblox.com or changing the password **invalidates the cookie**.
