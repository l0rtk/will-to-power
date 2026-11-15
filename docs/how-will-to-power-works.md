# How Will to Power works

Nov 15, 2026

## The big picture

The game is three layers stacked on each other: an engine that runs the machine, game logic that says what things can do, and content that says which things exist.

```
Content      text files that say which things exist        mods/ra
   |         read by the game logic when the game starts
   v
Game logic   traits: the behaviours things can have        OpenRA.Mods.Common, OpenRA.Mods.Cnc
   |         run by the engine on every tick
   v
Engine       window, drawing, sound, input, network, time  OpenRA.Game, OpenRA.Platforms.Default
```

The top layer is where most early changes happen. The two code layers below only change when you need a behaviour that does not exist yet.

| Layer | What it does | Where it lives | Written in |
| --- | --- | --- | --- |
| Engine | Opens the window, draws pictures, plays sound, reads the mouse and keyboard, talks to the network and keeps time | `OpenRA.Game`, `OpenRA.Platforms.Default` | C# |
| Game logic | Small reusable behaviours called traits: can move, can shoot, has health, can be built | `OpenRA.Mods.Common`, `OpenRA.Mods.Cnc` | C# |
| Content | Text files that build each tank, soldier and building by listing traits and numbers, plus maps, missions and menus | `mods/ra` | YAML text, Lua scripts |

The engine knows nothing about tanks. A tank is a short list in a text file: this picture, this speed, this gun. The engine reads that list and brings the tank to life.

This split is why you can change a lot of the game by editing text, and only need C# when you want a behaviour that does not exist yet.

## The engine

The engine is a loop that repeats two jobs many times a second: move the game forward one small step, then draw a picture of it.

### Starting up

1. `launch-game.sh` starts the program and names the mod to load: `Game.Mod=ra`.
2. The engine reads your settings and finds the installed mods. Any folder under `mods/` that has a `mod.yaml` file counts as a mod.
3. It opens the window and starts the sound system.
4. It loads the mod: its C# code, its text files, fonts and the list of maps.
5. The mod, not the engine, shows the main menu. The battle playing behind the menu is a real small map called the shell map.

This order is in `Game.Initialize` in `OpenRA.Game/Game.cs` (around line 344).

### The loop

The loop is `Game.Loop()` in `OpenRA.Game/Game.cs` (around line 771). It keeps two clocks.

- **Tick**: one step of the game. Units move a little, guns reload, money counts up. The default is one tick every 40 ms, which is 25 ticks a second.
- **Frame**: one picture drawn on screen. Frames can come more often than ticks, which is why movement looks smooth.

The game speed setting only changes the tick length. The choices run from 80 ms (slowest) to 20 ms (fastest) and are listed under `GameSpeeds` in `mods/ra/mod.yaml`. If the computer falls more than 250 ms behind, the loop drops the backlog and carries on.

### The four jobs around the loop

| Job | How it works | Where it lives |
| --- | --- | --- |
| Drawing | Collects everything visible, sorts it back to front, and sends the small pictures (sprites) to the graphics card in large batches through OpenGL | `OpenRA.Game/Graphics`, `glsl/`, `OpenRA.Platforms.Default` |
| Sound | Decodes sound files and plays them through OpenAL, up to 256 sounds at once | `OpenRA.Game/Sound` |
| Input | Once per frame, reads the mouse and keyboard through SDL2 and hands each event to the menus and buttons (widgets), which pass it on to the game world | `OpenRA.Game/Input`, `OpenRA.Game/Widgets` |
| Mod loading | Reads `mod.yaml`, then loads the code and file packages it lists | `OpenRA.Game/ModData.cs`, `OpenRA.Game/Manifest.cs` |

## Game logic

Everything on the map is an actor, and an actor is an empty box until traits are put inside it.

| Word | Plain meaning | Example | Where it lives |
| --- | --- | --- | --- |
| World | The whole battle: the map, every actor, every player | One match | `OpenRA.Game/World.cs` |
| Actor | One thing in the world | A tank, a soldier, a building, a tree | `OpenRA.Game/Actor.cs` |
| Trait | One behaviour an actor has | `Health`, `Mobile`, `Armament`, `Buildable` | `OpenRA.Mods.Common/Traits/` |
| Order | A command, small enough to send over the network | "Move to this spot" | `OpenRA.Game/Network/Order.cs` |
| Activity | A job that takes many ticks, kept in a queue | Driving to that spot | `OpenRA.Game/Activities/Activity.cs` |

### A trait has two halves

- **The Info half** holds the numbers from the text file: speed, cost, maximum health. All tanks of one type share it.
- **The live half** holds what is true right now for one tank on the map: its current health, where it is heading.

When a tank is built, the engine reads its list of Infos and creates one live trait for each. That happens in the `Actor` constructor in `OpenRA.Game/Actor.cs` (around line 169).

### What one tick does

`World.Tick` in `OpenRA.Game/World.cs` (around line 413) runs these in order:

1. Every actor carries on with its current activity.
2. Every trait that asked to be ticked runs once.
3. Effects such as bullets and explosions move forward.
4. Clean-up jobs saved for the end of the tick run.

### How a shot does damage

1. The `Armament` trait fires a weapon.
2. The weapon creates a projectile, which flies across the map as an effect.
3. When it lands, each of the weapon's warheads runs.
4. A damage warhead takes points off the target's `Health` trait.

### How much there is

There are roughly 580 traits. About 490 are in `OpenRA.Mods.Common`, about 70 in `OpenRA.Mods.Cnc` and 13 in `OpenRA.Mods.D2k`. `OpenRA.Mods.Cnc` is shared by all the Command & Conquer style mods, not only Red Alert.

## Content

Red Alert itself is a folder of text files, `mods/ra`. No C# code is written for any single tank.

### One tank, start to finish

The Medium Tank is 46 lines in `mods/ra/rules/vehicles.yaml` (line 99). This is a shortened copy:

```yaml
2TNK:
	Inherits: ^TrackedVehicle
	Buildable:
		Queue: Vehicle
		Prerequisites: fix, ~vehicles.allies, ~techlevel.medium
	Valued:
		Cost: 850
	Health:
		HP: 46000
	Armor:
		Type: Heavy
	Mobile:
		Speed: 72
	Armament:
		Weapon: 90mm
```

| Line | What it means |
| --- | --- |
| `2TNK:` | The tank's internal name |
| `Inherits: ^TrackedVehicle` | Start with everything all tracked vehicles share. That template is in `mods/ra/rules/defaults.yaml` |
| `Buildable` | Shows it in the vehicle tab. `Prerequisites` is the tech tree: you need a Service Depot (`fix`) and an Allied War Factory |
| `Valued: Cost: 850` | The price |
| `Health`, `Armor` | Hit points, and the armour class that weapons are compared against |
| `Mobile: Speed: 72` | How fast it drives |
| `Armament: Weapon: 90mm` | Which gun it carries |

Each indented name (`Health`, `Mobile`, `Armament`) is a trait from the game logic layer. The numbers under it fill that trait's Info half.

The gun and the picture are defined in two other files, linked by name:

- **The weapon** `90mm` is in `mods/ra/weapons/ballistics.yaml`. It inherits a shared cannon: damage 4000, reload 50 ticks, and 115% damage against Heavy armour.
- **The pictures** are the `2tnk:` entry in `mods/ra/sequences/vehicles.yaml`. A sequence names an image file and says which frames make up each animation.
- **The name** "Medium Tank" is in `mods/ra/fluent/rules.ftl`, where all on-screen text is kept.

### The file format

The files use OpenRA's own simple version of YAML, called MiniYAML. It has four rules worth knowing.

- Indent with one tab per level.
- `Inherits:` copies another block first. Names starting with `^` are templates, not real units.
- A line starting with `-` removes something that was inherited.
- `@` plus a label, as in `RevealsShroud@GAPGEN`, lets one unit have the same trait twice.

### What is in mods/ra

| Folder or file | What it holds |
| --- | --- |
| `mod.yaml` | The table of contents: a list of every other file to load |
| `rules/` | Units, buildings, players and the world |
| `weapons/` | Guns, missiles and explosions |
| `sequences/` | Which image files and frames each unit uses |
| `bits/`, `uibits/` | Extra art that ships with the project |
| `audio/` | Lists of unit voices, announcer lines and music |
| `chrome/`, `chrome.yaml` | Menu and sidebar layout. Most shared menus are in `mods/common/chrome` |
| `fluent/` | All on-screen text, in English |
| `maps/` | Multiplayer maps and missions |
| `tilesets/` | Terrain types and ground tiles |

### Maps and missions

A map is a folder or a zip file ending in `.oramap`. It holds `map.yaml` (players and placed units), `map.bin` (the terrain) and `map.png` (the preview).

A mission is a map with a Lua script added. The script sets objectives and reacts to events such as "this building was destroyed". A short real one is `mods/ra/maps/soviet-01/soviet01.lua`, at 105 lines.

### Where the original art and sound come from

The original Red Alert files are not in the project. The installer, `mods/ra-content`, downloads EA's free 2008 release. On Linux the files go in `~/.config/openra/Content/ra/v2/`.

The music and video files are an extra download that the installer offers.

## Follow one click

A click never moves a tank directly. It becomes an order, the order becomes a job, and the job moves the tank a little on every tick.

The example: you left-click a tank, then right-click empty ground.

1. **The click arrives.** At the end of each drawn frame the engine collects mouse events and hands them to the widgets. The widget that covers the battlefield is `WorldInteractionControllerWidget` in `OpenRA.Mods.Common/Widgets/`.
2. **Left-click selects.** The widget asks which actor is under the mouse and stores it as your selection in `OpenRA.Mods.Common/Traits/World/Selection.cs`. Selection is private to your screen. Nothing is sent to anyone.
3. **Right-click asks the tank what the click means.** `UnitOrderGenerator` in `OpenRA.Mods.Common/Orders/` asks each trait on the tank whether it can do something with that spot. For bare ground, the `Mobile` trait answers "I can move there".
4. **An order is written.** `Mobile` creates `Order("Move", tank, target)` in `OpenRA.Mods.Common/Traits/Mobile.cs` (around line 929). You hear the "acknowledged" voice right away, but the tank has not moved yet.
5. **The order makes a round trip.** `OrderManager` in `OpenRA.Game/Network/` sends the order out and waits for it to come back. This happens even in single player, where the game sends the order to itself, one step later.
6. **The order becomes a job.** When the order comes back, `Mobile` checks it and puts a `Move` activity in the tank's queue (`Mobile.cs`, around line 937). Whatever the tank was doing before is cancelled.
7. **A route is found.** On its first tick, `Move` asks the pathfinder for a route around cliffs, water and other units. The code is in `OpenRA.Mods.Common/Pathfinder/` and uses a well-known method called A* search.
8. **The tank moves a little each tick.** `Move` in `OpenRA.Mods.Common/Activities/Move/Move.cs` turns the tank, claims the next cell, and pushes the tank's position forward by its speed.
9. **The picture catches up.** Each frame, the renderer asks the tank's `RenderSprites` trait where to draw it. That trait reads the tank's current position, so the tank appears to drive.

Steps 1 to 4 happen only on your computer. Steps 5 to 8 happen the same way on every player's computer. That difference is what the next section is about.

## Multiplayer

Players never send each other the state of the battle. They send only their orders, and every computer runs the whole game itself from the same orders.

This method is called deterministic lockstep. "Deterministic" means the same orders always give exactly the same result. "Lockstep" means nobody moves to the next step until everyone's orders for it have arrived.

### How the games stay identical

- **Orders only.** The server passes orders between players and checks them. It does not run the battle. The player side of this is `OrderManager` in `OpenRA.Game/Network/`, and the server is `OpenRA.Game/Server/Server.cs`.
- **A checksum every step.** Each computer boils its whole game down to one number, the sync hash, and sends it to the others. If two numbers differ, the game stops with an "out of sync" error. The code is in `OpenRA.Game/Sync.cs`.
- **Shared dice.** Anything random in the battle uses one shared random number generator, so every computer rolls the same numbers.
- **Whole numbers only.** Positions and distances use integer types such as `WPos` and `WDist`. Decimal maths can round differently on different computers, so it is kept out of the battle.
- **The screen may not touch the battle.** Menus, selection and drawing run inside a guard that throws an error if they change the game state.

### What this gives you for free

| Feature | How it works |
| --- | --- |
| Replays | A replay is a recording of orders. Watching one runs a fresh game and feeds the orders back in. |
| Saved games | A save is also a recording of orders. Loading replays them at full speed up to the save point, then play continues. |
| Bots | A bot is a set of traits on a player. It runs on the host's computer and can act only by issuing the same orders a human would. Its code is in `OpenRA.Mods.Common/Traits/BotModules/`. |

### The rule to remember when you write code

Code that runs inside the battle must give the same result on every computer. Use the shared random generator and the whole-number types, and never let menu or drawing code change a unit.

## Where to change what

Most changes are one line in a text file, and they take effect the next time you start the game. Only a new kind of behaviour needs C# and a rebuild.

All paths below start at the project folder. The examples use the Medium Tank, `2TNK`.

| I want to change | Open | Edit |
| --- | --- | --- |
| A unit's cost | `mods/ra/rules/vehicles.yaml` (or `infantry.yaml`, `aircraft.yaml`, `ships.yaml`, `structures.yaml`) | `Valued: Cost: 850` |
| A unit's speed | The same unit block | `Mobile: Speed: 72` |
| A unit's health or armour | The same unit block | `Health: HP: 46000`, `Armor: Type: Heavy` |
| Weapon damage, range, reload | `mods/ra/weapons/ballistics.yaml` and the other files in `weapons/` | `Damage: 4000`, `Range: 4c768`, `ReloadDelay: 50` |
| Which gun a unit carries | The unit block | `Armament: Weapon: 90mm` |
| Build time | The unit block, under `Buildable:` | Add `BuildDurationModifier: 50`. Without it, build time comes from cost |
| What you must own before building it | The unit block, under `Buildable:` | `Prerequisites: fix, ~vehicles.allies` |
| Starting cash | `mods/ra/rules/player.yaml`, under `PlayerResources:` | Add `DefaultCash: 10000`. The built-in default is 5000 |
| Starting units | `mods/ra/rules/world.yaml` | The `StartingUnits@...` blocks |
| How a unit looks | `mods/ra/sequences/vehicles.yaml`, with image files in `mods/ra/bits/` | The `2tnk:` entry |
| A unit's name and description | `mods/ra/fluent/rules.ftl` | `actor-2tnk` and the lines under it |
| How the computer player behaves | `mods/ra/rules/ai.yaml` | Build weights such as `2tnk: 50`, squad sizes |
| A menu label | `mods/common/fluent/chrome.ftl` | `label-singleplayer-title = Singleplayer` |
| A menu's layout | `mods/common/chrome/mainmenu.yaml` | Button position and size. Keep the button's ID, because the C# code looks it up |
| A mission | `mods/ra/maps/<mission name>/` | The `.lua` script, `map.yaml` and `rules.yaml` |
| The game's name in the window title | `mods/ra/fluent/ra.ftl` | `mod-title = Red Alert` |
| The "OpenRA" heading on the main menu | `mods/common/fluent/chrome.ftl` | `label-main-menu-title = OpenRA` |
| Game speed choices | `mods/ra/mod.yaml`, under `GameSpeeds:` | `Timestep: 40` |
| A new unit | `rules/`, `sequences/`, `bits/` and `fluent/rules.ftl` together | Copy the `2TNK` block, its `2tnk:` sequence and its `actor-2tnk` text, then rename |
| A new behaviour | `OpenRA.Mods.Common/Traits/` | Write a new trait in C#, then rebuild |

Nothing in this table was tried in the game for this doc. The file locations and example lines were read from the project files.

### The working loop

| Step | Command |
| --- | --- |
| Build after changing C# | `make` |
| Run the game | `./launch-game.sh Game.Mod=ra` |
| Check the text files for mistakes | `make test` |
| Check code style | `make check` |
| Run the code tests | `make tests` |

Run these from the project folder with the .NET 10 SDK on your `PATH`. Text file changes need only a restart of the game, not a build.

One thing to know before changing rules: players can only play together if their rules match. A changed game cannot join matches with standard OpenRA players.

## Glossary

| Word | Plain meaning |
| --- | --- |
| Engine | The reusable program under a game: window, drawing, sound, input, network, timing |
| Mod | A game built on the engine. Red Alert is the `ra` mod |
| Game loop | The endless repeat of "advance the game a step, draw it" |
| Tick | One step of the game. 25 a second at normal speed |
| Frame | One picture drawn on screen |
| Sprite | A small 2D picture, such as a tank seen from one angle |
| Sequence | A named set of sprite frames that form one animation |
| Shader | A small program that runs on the graphics card to colour pixels |
| Widget | One piece of the interface: a button, a label, the sidebar |
| Chrome | OpenRA's word for the interface as a whole |
| Actor | One thing in the world: a unit, a building, a tree |
| Trait | One behaviour attached to an actor |
| Order | A command from a player or bot, such as "move here" |
| Activity | A job an actor works on over many ticks |
| Pathfinding | Working out a route around obstacles |
| Cell | One square of the map grid |
| Tech tree | The rules for what you must own before you can build something |
| Shroud and fog | The black unexplored area, and the dimmed area you cannot currently see |
| Lockstep | Multiplayer where every computer runs the same game from the same orders |
| Deterministic | Same input always gives exactly the same result |
| Desync | Two computers' games no longer match. Also called "out of sync" |
| YAML | A plain text format that uses indentation to show structure |
| Lua | A small scripting language, used here for missions |
| Fluent | The text format (`.ftl` files) that holds on-screen wording |
| SDL2, OpenGL, OpenAL | The standard libraries used for the window and input, for drawing, and for sound |

## Sources

Everything here was read from this project's own files, a copy of [OpenRA](https://github.com/OpenRA/OpenRA) made on 15 November 2026. The OpenRA wiki and documentation site were not used. Line numbers will drift as the code changes.
