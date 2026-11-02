# World of Tanks Random Vehicle MOD (mod_random_vehicle)

Press **F11** (rebindable) in the garage to randomly select one of your own vehicles. Purely a client-side feature — it is equivalent to you clicking a vehicle yourself. It does not read any server-side information that was not sent to the client, and contains no cheating logic.

## Quick Start

```
Double-click install_wotmod.bat    ← one-click install (prebuilt, fastest)
Double-click build_and_deploy.bat  ← one-click build + install (after source changes)
```

For detailed installation steps, directories, and file list, see **[INSTALL.md](INSTALL.md)**.

## Filtering

F11 only picks from vehicles that match the **filter conditions**. Filters are configured in the `<filters>` node of `res/gui/mods/randomVehicle.xml`:

- Premium status (any / premium only / standard only)
- Vehicle type (light / medium / heavy / tank destroyer / SPG)
- Nation, tier

Configuration examples and instructions are in **[INSTALL.md](INSTALL.md#筛选随机池配置文件方式)**.

## Directory Structure

```
wot-random-vehicle/
├── src/mod_random_vehicle.py       ← source (Python 2.7)
├── res/gui/mods/randomVehicle.xml  ← key configuration
├── build_wotmod.py                 ← build + deploy script (Python 3)
├── install_wotmod.py               ← one-click install script (Python 3)
├── install_wotmod.bat              ← double-click install entry point
├── build_and_deploy.bat            ← double-click build + install entry point
├── dist/mod_random_vehicle.wotmod  ← build artifact (package loaded by the game)
├── INSTALL.md                      ← detailed installation and deployment guide
├── AGENTS.md / CLAUDE.md           ← AI agent guides
└── README.md
```

## Command Line

```bash
python build_wotmod.py --deploy                          # build and deploy automatically
python install_wotmod.py --game-root "C:/Games/World_of_Tanks_CN"   # install prebuilt package only
```

## Deployment Locations (Summary)

```
<game root>/mods/<version>/mod_random_vehicle.wotmod      ← main MOD
<game root>/res_mods/<version>/gui/mods/randomVehicle.xml ← key configuration
```

Restart the game to activate. To rebind keys or adjust filters, use the "MOD configuration" panel in `install_gui.py`, or edit `randomVehicle.xml` directly (e.g. change `<key>` to `KEY_F10` / `KEY_F12`, matching the constant names in the `Keys` module). The configuration is hot-reloaded at runtime — changes take effect about 1 second after saving, with no restart needed.

## Troubleshooting

A `mod_random_vehicle.debug` file (step-by-step log) is generated in the game root. Search `python.log` for `[mod_random_vehicle]`. If you see `HOTKEY: select result=True` but the vehicle does not change, the vehicle-selection API version does not match — see the adaptation notes below.

## Version Adaptation Notes (Important — all from real-world testing)

This is a MOD mechanism specific to the CN client **2.3.x** (2026), and it differs greatly from classic WOT (1.x):

1. **The client version directory changes**: whichever version number `paths.xml` points to determines where the MOD goes in `mods/<version>/`. After a client update you must redeploy (`build_wotmod.py --deploy` detects the version number automatically). Symptom: the MOD is clearly present in `mods/2.3.1.0/` but does not load, because the client is already on 2.3.1.1.

2. **Loose `.py` files are no longer loaded**: `res_mods/<version>/scripts/client/mods/*.py` is **not executed** in this client (the legacy mechanism has been removed; `scripts/client/mods/` has zero references in the script package). Only wotmod packages work.

3. **wotmod format requirements**:
   - Must be an **uncompressed** zip (`ZIP_STORED`)
   - Must have a **`res/` mount root** at the top level, with scripts placed in `res/scripts/client/gui/mods/` (note the `gui` in the middle)
   - The filename must start with **`mod_`** (e.g. `mod_random_vehicle.pyc`); the loader only executes `mod_*.pyc`
   - On load failure the log reports `Mod package ... load error: mod is empty or dont have mount root`

4. **Scripts must be compiled to `.pyc`**: the native loader only executes `.pyc` (a `.py` file is not executed even at the correct path). The bytecode magic number must be standard Python 2.7 `03 f3 0d 0a` (compiling with official Python 2.7.18 is sufficient).

5. **Module interface**: providing `init()`/`fini()` (or `finit()`) causes the loader to call them; but **module-level top code runs immediately on import** (the sixthsense plugin is pure top-level code). This MOD starts via both top-level code and `init()` as a double safeguard.

6. **Key APIs (2.3.1.1)**:
   - Vehicle list: `dependency.instance(IItemsCache).items.getVehicles()` returns **every vehicle in the full tech tree** (including unpurchased ones); you must filter by `isInInventory` to get only vehicles you own
   - Vehicle selection: `from gui.shared.event_dispatcher import selectVehicleInHangar; selectVehicleInHangar(intCD)` (official API, confirmed by disassembly: dispatches the `SELECT_VEHICLE_IN_HANGAR` event → `g_currentVehicle.selectVehicle` → `showHangar`). Calling `g_currentVehicle.selectVehicle(intCD)` on its own **will not** drive the garage to switch vehicles
   - `skeletons.items_cache` has been removed in 2.3.1.1; `IItemsCache` lives in `skeletons.gui.shared`

7. **Working plugins for reference**: `mods/<version>/zanju.researchprogressbar_1.3.1.wotmod` (research progress bar) and `sixthsense_core.wotmod` (Sixth Sense) — their internal structure is the canonical answer.

## Uninstall

Delete `mods/<version>/mod_random_vehicle.wotmod` and `res_mods/<version>/gui/mods/randomVehicle.xml`.

## Compliance Statement

- Uses only data already available on the client (your own garage list); no auto-aim, no prediction, no enemy reload tracking, or any other cheating logic
- Equivalent to the player clicking a vehicle in the garage
