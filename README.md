# Freedom Fighters (PC) — reverse-engineering notes

Research notes on the PC build of *Freedom Fighters* (IO Interactive, Glacier engine), run under Wine on Linux. The goal is to find out how feasible **local co-op in the campaign** is, starting from the multiplayer code that is still compiled into the PC executable.

> **No game files are in this repository.** You need your own legal copy of the game. Nothing here redistributes the executable, level data, or other assets. Only notes, hashes, and small scripts are included.

## Status

| Question | Answer so far |
|---|---|
| Does the PC exe still contain the multiplayer code? | **Yes** (observed) |
| Is it reachable from the PC menus? | **Probably not** (inferred from decompile, not yet runtime-tested) |
| Are the multiplayer maps shipped with the PC build? | **No** (observed: no `Multiplayer` scene folder in the install) |
| Do campaign levels contain second-player objects? | **Unknown** — this is the key open check |
| Can the in-game console enable multiplayer or add a player? | **No** (observed: no such command in the command list) |

## Environment

- Linux (CachyOS), game run under Wine with a dedicated prefix
- Ghidra **12.1.4** (headless, used by REA) and Ghidra 12.1.2 (GUI, used with GhidraMCP)
- [REA](https://github.com/morluto/rea) 3.2.1 as an MCP server for read-only analysis; GhidraMCP for annotation
- An AI coding agent driving both through MCP

Target build (identify yours by hash before comparing addresses):

| File | SHA-256 |
|---|---|
| `Freedom.Exe` as analysed (two 2-byte/6-byte cover-visualiser patches applied) | `6bf3c3e4b879354d3b5848c7436fce7e5c54e36bd57dbe53d3ce4b18f31ed473` |
| `Freedom.Exe` original | `d1525e1855bf15701ec84d20db52c72c1ff099afc5fb70ba954a9d403691a86c` |

PE32 x86, about 3.3 MB. The patches touch only the cover-visualiser code, so addresses elsewhere match the original.

## How findings are labelled

- **Observed** — I saw it directly (file listing, log output, console, tool output).
- **Inferred** — read from Ghidra decompiler output, often summarised by an agent. Plausible, not confirmed at runtime.
- **Wrong** — something stated earlier that later evidence contradicted. Kept on purpose.

## Findings

### Binary overview (observed)

REA's Ghidra provider (12.1.4) imports the 32-bit PE on Linux and finishes auto-analysis in about 153 seconds: 13,221 procedures, 7,316 strings. Image base 0x400000, `.text` at 0x401000–0x675800.

The exe contains a `.gfids` section, which suggests it was rebuilt with a modern toolchain (consistent with the 2020 re-release). The level data inside the archives is dated August 2003.

### Multiplayer code is compiled in (observed strings, inferred behaviour)

Strings found at these addresses: `MP_NumberOfPlayers` (0x68fcec), `MP_Player%iActive` / `%iTeam` / `%dSlot`, `MultiplayerKOTH`, `ZWINDOW_MultiplayerKOTH`, `rMULTIPLAYERKOTH`, `UseMultiTap`, and the source path `...\MP_GameController.cpp`.

Functions (names are my labels, from decompilation):

| Address | Role (inferred) |
|---|---|
| `0x0056d270` | Lobby "start match" handler: counts active slots and teams, requires players on both teams, sets `MP_NumberOfPlayers`, loads the map |
| `0x0056cee0` | Command handler of the multiplayer lobby window; the only caller of `0x0056d270` |
| `0x005a3710` | Multiplayer game controller constructor; if `Multiplayer` is unset it defaults to 2 players |
| `0x005a4750` | Viewport/camera setup per player count (2, 3 or 4); an `int3` path for other counts |
| `0x005a86c0` | HUD/viewport layout |
| `0x0058e720` | Player entity activation: binds to `HitmanGround` / `MainCamera`, with numbered variants for later slots (indexing not verified) |

Reachability: the lobby window class has no cross-references from the PC frontend classes (`CBootMenu`, `CNewMenu`). Static xrefs can miss dynamically registered classes, so "unreachable" is likely, not proven.

### Multiplayer data is missing (observed)

The install's `Scenes/` folder contains only `AllLevels`, `Cutscenes` and `Singleplayer`. The code builds paths like `Multiplayer\Koth_02\Loader`, which do not exist here. Re-enabling the lobby alone would not give a playable match.

### Configuration (observed, with corrections)

- The engine does **not** read the `Freedom.ini` in the game folder. The launcher passes the engine `@"...\AppData\Roaming\IO Interactive\Freedom Fighters\Freedom.ini" -SKIP_LAUNCHER`, so the live file is in the Wine prefix's AppData (`freedom.ini`).
- Syntax is `Key value` separated by a space (for example `Resolution 1920x1080`). The `-dbg_ini` switch prints the preprocessed file, which is the easiest way to see what the engine parsed.
- The engine silently accepts unknown ini keys, so a clean parse proves nothing about an option's validity. Test by behaviour.
- Adding `EnableConsole 1` and `EnableCheats 1` makes the backtick key open a developer console on the main menu.

**Wrong, earlier in this investigation:** that the default config is `main.ini` read with `Key=Value`; that the console supports `startscene`, `reloadengine` and `printstatus`; that `Freedom.ini` in the game folder was in use.

### Console commands (observed)

Includes: `cams`, `dir` (scene tree), `dirclip`, `hira`, `hiraclip`, `globals`, `zdefines`, `reload`, `reloadscripts`, `scriptplot`, `scriptstatus`, `killscript`, `dumpai`, `actorinfo`, `show_follow`, `show_squads`, `herocoords`, `goto`, `showheroroom`, `god`, `infammo`, `giveall`, `giveneeded`, `no_damage`, `invisible`, `blockfire`, plus rendering and memory dump commands. There is no command that sets a player count, spawns a second hero, or starts a multiplayer match.

### Campaign data layout (observed)

Each mission is a folder such as `Scenes/Singleplayer/C01A/` with `Loader.ZIP`, `FF-C01A_MAIN.ZIP`, and audio files. Archives hold Glacier formats (`ZGF`, `GMS`, `TEX`, `SND`, `LOC`, `OCT`, `PRM`, and others). No readable script source was seen.

## Open questions

1. **Do campaign scene files contain second-slot objects** (`HitmanGround2`, `MainCamera2`, `Hero1`…) or only the single-player names? This decides whether co-op is mainly a code project or also a per-level data project.
2. Is the multiplayer lobby truly unreachable at runtime, or registered by name somewhere?
3. Does `DefaultScene` accept a campaign `.GMS` path, and what exact string does the engine expect?
4. How does a second controller map to a player slot in the campaign build?

## Reproducing

```text
# Ghidra provider check
rea doctor --json
rea analyze Freedom.Exe --provider ghidra

# Show what the engine parsed from its config
wine Freedom.Exe -SKIP_LAUNCHER -dbg_ini

# Open the console: add to the AppData freedom.ini
EnableConsole 1
EnableCheats 1
# then press ` on the main menu

# Check level data for player/camera object names
unzip -o -q Scenes/Singleplayer/C01A/Loader.ZIP -d work/c01a
unzip -o -q Scenes/Singleplayer/C01A/FF-C01A_MAIN.ZIP -d work/c01a
grep -a -o -i -h -E 'HitmanGround[0-9]*|MainCamera[0-9]*|Hero[0-9]+' -r work/c01a | sort | uniq -c
```

## Tooling notes

- REA 3.2.1 needs Ghidra 12.1.4 exactly; older REA releases pinned 12.1.2.
- The REA MCP entry needs `GHIDRA_INSTALL_DIR` and a JDK 21 on `PATH`.
- REA's Ghidra provider is read-only; renames and comments do not persist between sessions. Use GhidraMCP in the GUI for annotation.
- Agent-produced decompilation summaries were right about names and structure more often than about defaults, field offsets and behaviour. Verify before building on them.
# freedom-fighters-re
