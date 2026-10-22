# BlockSpin — Map & Locations

Layout: ~6x5 road grid, southern-US suburb. There is NO in-game map; floating
markers point to key places. Center = the crossroads of the two big streets.
Workspace tree: `workspace.Map.Tiles.<TileName>.<Building>.Interior`.

| Location | Purpose | From crossroads |
|---|---|---|
| Gas Station | Spawn area, vehicle fuel | At crossroads |
| Burger Place | Janitor job | At crossroads |
| Jack's Hardware | Job tools, fishing gear | At crossroads |
| Patriot Shooting Range | Weapon crates (knife icon) | At crossroads |
| Urban Cycles & Skateboards | Bicycle/skateboard shop | Just above Burger Place |
| Barbershop | Haircuts | Below hardware store |
| Iced Out | Jewelry | Below crossroads, next to Barbershop |
| The Butcher's Cut | Cook job; hack-tool dealer in back alley | Below crossroads, right side |
| Quick-11 | Shelf Stocker job, Blood Bags | — |
| Medical Center | Blood Bags ($100) | One block left of shooting range |
| Cemetery | Decorative | One street below Medical Center |
| City Hall | Decorative park | One street below crossroads |
| Car Wash | Vehicle wash | One street above Gas Station |
| Sam's Motel | Illegal Dealer in alley (hack tools, lockpick, shiesty) | — |
| Basketball Court / SW Nassu St | Dealer (gloves, energy shots) | — |
| Pawn Shop | Sell loot (~30% of value) | Top-left corner, by the sea |
| Spin Club | Party venue | Top-left, next to Pawn Shop |
| Skate Park | Skating area | Between sea and City Hall |
| Vellaro | Outfits | Across street from Skate Park |
| Beach | Pawn shop area | Top-left / west edge |
| Mansions + Mansion Underground | High-tier homes, extra safes | Mansion coast |
| Offshore deep-sea spot | Level-75 fishing | Off the mansion coast |
| ATMs | Deposit cash / hack targets | Scattered, incl. Police Station area |
| Dealerships (car icon) | Car crates | Map icons |

Useful confirmed object paths (leaks):

```lua
workspace.Map.Tiles.ShoppingTile.SteakHouse.Interior
workspace.Map.Tiles.BurgerPlaceTile.BurgerPlace.Interior.Puddles
workspace.Map.Tiles.GunShopTile.PatriotWeapons.Interior.Crates["Weapon Crate"].CrateOptions.Superior
workspace.Beacon.Base
workspace.Vehicles        -- spawned cars (OwnerUserId attribute, Motors child)
workspace.DroppedItems    -- dropped loot Models (PickUpZone inside)
```

Teleport coordinates (from leaked client Teleport_Positions — Y ≈ 255-261):

| Place | Vector3 |
|---|---|
| Bike Shop | 256, 261, -206 |
| Burger Place | 113, 260, -212 |
| Gun Shop | -140, 259, -221 |
| House block | 124, 261, -393 |
| Hospital | -133, 261, -555 |
| Dealer | -222, 255, 388 |
| Car Dealership | 118, 260, 431 |
| Gas Station | 125, 258, 137 |
| Pawn Shop | 1177, 258, -535 |
| Spin Club | 1253, 260, -418 |

PlayerGui names observed: `JobApplication`, `ProgressBar`, `TopRightHud`
(stamina bar at `TopRightHud.Holder.StaminaBar`).
