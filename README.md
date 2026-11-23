# SavantAPI

SavantAPI is the contract for Outpost 2 implementations.

Implementations are different takes of the same game. Each keeps its own code. This repository names the shared words they use, so the same play can run on each of them and the recordings can be compared.

The words are listed here first. Schemas for them land in this repository as each surface is written. Each surface is versioned on its own.

One repository holds the contract for now. The map document already has a home, and each game stays in its own repository. Those splits are described under [Where the other pieces live](#where-the-other-pieces-live).

A savant is one seat at that play: it is shown an observation and it answers with orders. The harness is the envelope around a play: a gameplay in, a recording out.

## Where the other pieces live

- **Map document.** [OP2MapJsonTools](https://github.com/leviathan400/OP2MapJsonToolsLibrary) owns the JSON. [OP2Utility](https://github.com/OutpostUniverse/OP2Utility) owns the `.map` binary. [OP2MapperQt](https://github.com/leviathan400/OP2MapperQt) edits both. This repository names that JSON as the map. It does not keep a second copy of the format.
- **Windows adapter.** [OP2TestHarness](https://github.com/TethysGame/OP2TestHarness) loads inside the original executable and speaks this contract for that process. The verb names below are the ones that adapter already runs.
- **Other implementations.** Each take of the game keeps its own code and depends on this repository.

## Reading order

The map and the other nouns come first. Orders are verbs over those nouns. A gameplay is a map plus a seed, a ruleset, players, setup, and orders. A recording is what an implementation writes while it plays that gameplay. A savant sees one seat of a recording's world and returns orders.

## Nouns

### Who is playing

- **Implementation.** Which take of the game ran the play. Two implementations can play one gameplay.
- **Ruleset.** Which behavior that implementation is expected to follow. A `1.4.1` run is a different ruleset from a `1.2.0.7` run, so a difference that the patch intends is not a failed comparison.
- **Surface version.** The version of one contract surface (nouns, verbs, map, harness, savant). A change to the meaning of a field is a new version of that surface.
- **Seat.** One player slot in the play. The occupant is a human or a savant.
- **Player.** The colony in that seat: id, human or AI, Eden or Plymouth, and color.
- **Spawn index.** A unit's place in the recipe's spawn list. Recordings are matched on this index. The game's runtime unit id is handed out as units appear, so it differs between builds and is not the comparison key.

### Time

- **Tick.** The game's clock. Orders and samples are stamped with it.
- **Mark.** 100 ticks. The game's own coarser clock.
- **Seed.** The gameplay random seed. The same seed is part of what makes a recording reproducible.
- **Game speed.** The in-game speed setting, 1 through 10. It changes how fast ticks run against the wall clock. It does not change the tick numbers in a script.
- **End reason.** Why the play stopped: still running, won, lost, or another `GameTermReasons` value.

### Map

The map noun is the JSON document [OP2MapJsonTools](https://github.com/leviathan400/OP2MapJsonToolsCmd) already writes, and that [OP2MapperQt](https://github.com/leviathan400/OP2MapperQt) imports and exports. A `.map` file is that same document in the binary layout from OP2Utility. A gameplay points at one map document, then adds the seed, the ruleset, the players, the spawns, and the orders.

The JSON has three grid layouts of the same data: flat, one array per row, and padded rows. The contract uses one of them. Which one is fixed when the schema is written.

- **header.** `width` and `height` in tiles, the file name (`map`), and the display `name`, `author`, and `notes`.
- **tiles.** The tile-mapping index of every tile. This is the picture the game draws.
- **cellTypes.** One cell type per tile. This is movement, and it is also how tubes, walls, and vents are stored. See the list below.
- **clipRect.** `x1`, `y1`, `x2`, `y2`, the visible area. It is included when that area differs from the default. A world map is 512 tiles wide and wraps on X.
- **tileset.** How tile indexes become graphics. `sources` names each tileset file and how many tiles it holds. `tileMappings` gives, for each mapping, the tileset index, the graphic index, the animation count, and the animation delay. `terrainTypes` gives a range of tiles and which mapping is used for dozed ground, rubble, tubes, walls, lava, flat ground, and scorched ground.
- **Tile flags.** Five bits on the binary tile that the published JSON does not yet carry: lava present (`bLava`), lava can enter (`bLavaPossible`), the expansion bit (`bExpansion`), blight present (`bMicrobe`), and wall or building present (`bWallOrBuilding`). Lava-possible is real map data: the mapper writes it around a generated volcano. These flags are an addition to this same document, not a second format.
- **Tile groups.** Named clumps of tiles stored on the binary map. The JSON sample does not carry them yet. Same rule as the flags: add them to this document if a port must reload them.
- **unitIndex.** The unit occupying a tile, stored on the binary tile. Save games use it. An authored map leaves it empty. Units placed for a play are spawns on the gameplay, not fields of the map.

Cell types are the 32 values of OP2Utility `CellType`:

- **Ground.** `FastPassible1`, `FastPassible2`, `MediumPassible1`, `MediumPassible2`, `SlowPassible1`, `SlowPassible2`. These set how fast a unit crosses the tile.
- **Blocked.** `Impassible1`, `Impassible2`. Units do not cross these.
- **Cliffs.** `NorthCliffs`, `CliffsHighSide`, `CliffsLowSide`.
- **VentsAndFumaroles.** The cell type for a vent or a fumarole. A mining beacon is not this. A beacon is a unit the gameplay spawns.
- **Pads.** `zPad12` through `zPad20`. Reserved cell values.
- **Built.** `DozedArea` (bulldozed, not tubed), `Rubble`, `NormalWall`, `MicrobeWall`, `LavaWall`, and `Tube0` through `Tube5`. `Tube0` is also the cell under a building.

Spawn markers and volcano markers in the editor are guides drawn on top of the view. The map file does not store them.

### Units

- **Type.** A sheet code name, such as `LIGHT_TANK`, `MINE`, or `CONVEC`. The set is closed: it is the game's unit sheet.
- **Weapon, kit, or cargo.** What an armed unit fires, what a ConVec is carrying as a structure kit, or what a truck holds.
- **Owner.** The player id that owns the unit.
- **Position.** The tile, the world pixels, and the 16.16 fixed-point position. Traces carry all three so a comparison can see a sub-tile difference.
- **Direction.** Rotation from 0 to 255. 0 is east, and the value increases clockwise.
- **HP.** Current hit points, and the type's maximum where a sample includes it.
- **Command and action.** The order the unit is carrying out, and the animation action it is in. `busy` says the unit will not take a new order.
- **Path.** The waypoints still to walk, the step directions of the current path fragment, and the rotation while the unit is turning.
- **Combat.** The weapon, the turret angle, the reload timer, and the current target.
- **Status timers.** Remaining time for EMP, stickyfoam, and ESG.
- **Building.** Footprint size, dock tile, rubble left on death, building flags, and the hive the building belongs to.
- **Name.** A `$NAME` bound to a unit, or `as NAME` on an order that creates a unit. Names are resolved when the order runs, so a script survives a shift in runtime ids.
- **Mining beacon.** A spawned unit: common or rare ore, a bar count of 3, 2, or 1, and a variant. The gameplay places it. The map does not.
- **Gaia.** Wrecks, shots, and explosions. A trace may list them beside the units. They have no seat.

### Colony

- **Stocks.** Food, common ore, and rare ore.
- **Population.** Workers, scientists, and children.
- **Morale.** The morale value and the morale level from Great down to Rotten.
- **Power.** Power produced and power consumed.
- **Food net.** Food produced minus food eaten, as the colony reports it.
- **Buildings.** How many buildings the player has, and how many of them are idle.
- **Techs.** The tech ids this player has completed.
- **Start setup.** Difficulty, starting resources (high, medium, or low), and how many vehicles the player starts with.
- **Economy sample.** Storage caps, food growth, and deaths. Present only on the detailed economy sample, not on every trace line.

### Rules the gameplay names

- **Tech tree.** The tree file the play loads, such as `multitek.txt` or `edentek.txt`.
- **Unit stats.** Speed, turn rate, track type, HP, and armor as loaded for this ruleset, including sheet values and any patch that ruleset applies.
- **Terrain speed.** The speed of each track (legged, wheeled, miner, tracked, hover) on the cell types, and which cell types block line of sight.
- **Morale table.** The levels and the modifiers that move morale.
- **Mine yields.** How much ore each bar count produces.
- **World switches.** Disasters on or off, morale held steady or not, day and night on or off.
- **Multiplayer type.** Last One Standing, Midas, Resource Race, Space Race, or Land Rush. This is the lobby's scenario type when the play is a multiplayer start.

### What a run writes

- **Trace line.** One sample of the units at a tick, plus the gameplay random number generator. This is the line a comparison walks field by field.
- **Telemetry line.** Players and units at a tick, coarser than the trace. Colony stocks live here.
- **Game data.** Written once at the start: map size, the cell grid as the running game sees it, and the unit stats.
- **Event.** A line for a disaster, an objective, a message, a hit, or the end of the play.
- **Summary.** The last tick, the end reason, and how the process exited.
- **Picture.** A window shot or a whole-map image. Optional. Comparison of the simulation ignores it. `frame` and `anim` on a unit follow drawing the same way.

## Verbs

A line is `<tick> <verb> <args>`. Three groups follow. A savant may return only an order. Setup is part of the gameplay the author writes. The Windows adapter's pixel and menu verbs are how that adapter drives the original window. They are not part of the simulation contract.

### Orders

A seat may send these. They become the command packets the game's own UI would send, and the game's own checks apply. An order at tick T is visible as the unit's command on that tick. The unit starts to carry it out on the next tick.

- **move.** Send one unit to a tile.
- **movewp.** Move through up to eight waypoints. A waypoint may carry a range.
- **dock.** Dock at a tile, such as a structure's dock tile.
- **dockeg.** Dock at an Eden garage. The unit ends facing north. This is the dock the game's UI sends there.
- **attack.** Attack a unit.
- **attackground.** Attack a tile.
- **guard.** Guard a unit.
- **scatter.** One scatter order for the listed units.
- **standground.** Hold a path of waypoints.
- **patrol.** Patrol a path of waypoints. `loop` is the waypoint the patrol returns to.
- **stop.** Cancel the unit's current order.
- **selfdestruct.** Order the unit to self-destruct.
- **lights.** Turn the unit's lights on or off.
- **research.** A lab starts a tech, with a scientist count.
- **produce.** A factory builds a unit type. A combat type may name a weapon.
- **idle.** Mark a factory idle.
- **unidle.** Clear that idle mark.
- **train.** A university trains a number of scientists.
- **bay.** Fill one factory bay with a kit.
- **transfercargo.** Move a bay's contents into the rocket on a spaceport pad.
- **route.** Put a cargo truck on a route between a mine and a smelter.
- **cargo.** Set a truck's cargo type and amount.
- **load.** The truck docked at a building loads.
- **unload.** The truck docked at a building unloads.
- **launch.** A spaceport launches the rocket on its pad. An optional tile is the launch target.
- **kit.** Put a structure kit into a ConVec.
- **build.** A ConVec builds its kit at a tile.
- **doze.** A Robo-Dozer bulldozes a rectangle. This writes `DozedArea` through play, the way the dozer's click does.
- **buildwall.** An Earthworker builds `TUBE`, `WALL`, `WALL_LAVA`, or `WALL_MICROBE` over a rectangle.
- **removewall.** An Earthworker removes the tube or wall on one tile.
- **repairobj.** Repair a target unit. Further units on the same line join the order.
- **reprogram.** Reprogram a target, the way the reprogram command does.
- **dismantle.** Dismantle a target.
- **gamespeed.** Set the game speed, 1 through 10, the way the in-game speed buttons do.
- **packet.** A raw command packet: a command type from 1 to 53 and its bytes. This is an escape for an order the named verbs do not cover. The bytes belong to one ruleset, because they are that executable's packet layout.

### Recipe setup

The author writes these on the gameplay. A savant does not send them. Several of them edit the world before or during the play. The same facts, where they are map facts, also have a place on the map document.

- **spawn.** Create a unit: type, owner, tile, weapon or kit, and direction. A mining beacon is a spawn. Spawns are not fields of the map.
- **setcell.** Paint a rectangle with a cell type. This is the setup form of writing `cellTypes`.
- **tube.** Lay tubes over a rectangle at runtime.
- **wall.** Place a wall type over a rectangle at runtime. On the map, the same result is a wall cell type.
- **coords.** Switch later tile numbers between internal tiles and file tiles. File tiles match the map document. Internal tiles add the map's padding.
- **player.** Set one colony field, such as workers, scientists, children, food, common ore, rare ore, or morale.
- **tech.** Mark a tech complete for a player, the way a mission grants a topic.
- **name.** Bind a name to a unit now. `as NAME` on `produce` or `build` binds the unit that order creates.
- **meteor.** Schedule a meteor at a tile, with a size.
- **quake.** Schedule an earthquake at a tile, with a magnitude.
- **eruption.** Schedule a volcanic eruption at a tile, with a lava speed.
- **lightning.** Schedule lightning along a segment, with a duration.
- **tornado.** Schedule a tornado along a segment, with a duration.
- **blight.** Schedule blight at a tile, with an optional speed.
- **empmissile.** Schedule an EMP missile from a tile to a tile, owned by a player.
- **lavaspeed.** Set how fast lava spreads.
- **lavapossible.** Mark a rectangle as able to hold lava, or not. On the map this is the `bLavaPossible` bit.
- **poke.** Write bytes on a live unit. A debugging tool of the Windows adapter. It is not part of the shared contract.
- **peek.** Read bytes from the live process. Same status as `poke`.

### Window and menu

These drive the original game's window through the Windows adapter: the mouse, the keyboard, and the command pane. A port that has no such window does not implement them. Orders issued this way pass through the UI, so they take effect several ticks later than the order verbs above. Selection itself is immediate.

- **click.** A left click on a unit, a tile, the minimap, or a named control.
- **rclick.** A right click at a window pixel.
- **drag.** A drag that box-selects, from one tile or pixel to another.
- **dblclick.** A double click.
- **mclick.** A middle click.
- **mouse, down, up.** Place the pointer, press a button, or release it.
- **key, keydown, keyup.** A key press, or the two halves of one.
- **type.** Type characters, such as a chat line.
- **select.** Select a unit, scrolling it into view first when it is off screen.
- **hotkeys.** Read the command pane's hotkey table.
- **view.** Center the view on a tile. Used so a picture shows the right place.
- **shot.** Take a window picture on this tick.
- **telemetry.** Write a telemetry sample on this tick.
- **trace.** Write a trace sample on this tick.
- **quit.** End the run.
- **savegame.** Save the play to a named save.
- **loadgame.** Load a named save.

### Mission groups

The original mission API drives squads through groups. The Windows adapter already accepts `group` with the operations below. They stay out of the savant seat until a savant is allowed to use that API.

- **new.** Create a fight, mining, or building group for a player.
- **take, remove.** Add or remove a unit.
- **rect, guardedrect, clearguarded.** Set the rectangle the group uses.
- **patrol, clearpatrol.** Set or clear a patrol path.
- **guardrect, patrolonly, attackenemy, guardgroup, guardunit, attackunit, exitmap.** What the group does.
- **mine.** Set up a mining group on a mine, a smelter, and a rectangle.
- **building, tube, wall.** Record a building or a piece of earthwork on a group.
- **targetunit, targetgroup, attacktype, targcount.** Aim the group.
- **reinforce, unreinforce.** Link a group as reinforcement.
- **lights, enable, disable, destroy, combine, follow.** The remaining group switches.

## Map contract

- A map is one JSON document: header, tiles, cellTypes, clipRect, and tileset.
- That document round-trips with a `.map` file through OP2MapJsonTools. OP2MapperQt is the editor.
- The five tile flags and the tile groups are the open additions. They belong on this document.
- A gameplay names a map, then adds the seed, the ruleset, the players, the spawns, and the orders.
- `setcell`, `tube`, `wall`, and `lavapossible` in a gameplay are edits applied on top of that map.

## Harness rules

- A **gameplay** is a map document, a seed, a ruleset, the players, a tick limit, the setup verbs, and the orders.
- An **adapter** is one implementation's side of the harness. Its calls are `start(gameplay)`, `tick()`, `command(verb, args)`, `snapshot()`, and `stop()`.
- Orders at tick T run after that cycle.
- An order that names a spawn the trace has already seen die is skipped. The game reuses object indexes, so the order would hit a different object.
- The gameplay is enough to regenerate the recording. The recording is the output, and the gameplay is what is kept.
- A **diff** walks the two traces field by field. The result is a match, or the first tick at which a field differs.
- `frame` and `anim` follow whether the picture was drawn. A simulation diff ignores them.
- Two implementations, one gameplay, two recordings, one diff.
- The ruleset says which behavior was required. GOG and 1.4.1 are compared inside a ruleset, not against each other, unless the gameplay asks for that comparison.

## Savant rules

- A savant occupies one seat.
- On each decision it receives an **observation** and returns orders. The orders use the order verbs above, and no others.
- **Fair** means the observation contains only what that seat can see. **Omniscient** means it contains the full trace. The gameplay names which one this seat gets.
- The same orders can be played back through the harness on every implementation.

## Later

Shell menus, LAN games, the window verbs, mission groups, and raw `packet` stay on the Windows adapter until a port needs them in the shared contract.

The first schemas to write are the map document as it exists today, the order verbs, the gameplay, the trace, and the savant seat.

## License

MIT. See [LICENSE](LICENSE).
