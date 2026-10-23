# blockspin-skill

An agent skill (SKILL.md) containing a complete knowledge base for the Roblox
game **BlockSpin** — mechanics, map, item values, client internals (networking
layer, anti-cheat), sub-place shards, and a verified-safe workflow for moving
items between accounts.

## Layout

- `SKILL.md` — the skill entry point (frontmatter + core knowledge + workflow)
- `references/mechanics.md` — jobs, combat, houses/safes, vehicles, crime tools
- `references/map.md` — full location list + confirmed workspace paths
- `references/items-values.md` — weapon/vehicle values, durability, untradeables
- `references/internals.md` — Net module, anti-cheat, drop/pickup strategies
- `references/accounts-sessions.md` — .ROBLOSECURITY capture, auth tickets,
  `roblox-player:` launch URL, server picking, multi-instance mutex handling
- `references/jobs-autofarm.md` — literal automation specs per job/skill:
  shelf stocker, cook, janitor, ATM hacker, fishing 1→80, farmer, economy map

## Install

Copy the folder into your skills directory:

- Personal: `~/.agents/skills/blockspin/` (or `~/.cursor/skills/blockspin/`)
- Project: `.devin/skills/blockspin/` (or `.cursor/skills/blockspin/`)

## Accuracy notes

Facts are tagged by confidence inside the files. Public guides drift with the
game's frequent updates; names/paths from leaked client code are marked as
observed-not-official. The exact drop/store remote events are not publicly
documented — the skill describes how to capture them in-game with a remote spy.
