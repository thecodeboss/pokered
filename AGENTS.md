# AGENTS Guide (pokered)

## Quick Facts
- Game Boy Color disassembly of Pokémon Red/Blue built with RGBDS (see `Makefile`). Entry includes `main.asm` (banks), `home.asm` (ROM0 core), `audio.asm`, `maps.asm`, `ram.asm`, `text.asm`, `gfx/*.asm`.
- `includes.asm` injects macros, script helpers, and constant tables (hardware regs, map ids, species/move ids, etc.) for every assembled unit.
- Build outputs `pokered.gbc`, `pokeblue.gbc`, debug/VC variants; graphics converted via `rgbgfx` with optional postprocessing by `tools/gfx`.

## Directory Layout
- `home/`: ROM0 utilities that must be always loaded—startup (`start.asm`, `init.asm`), VBlank/interrupt handling, text box routines, bankswitch helpers, overworld core loop, common math/inventory/list helpers, palette/OAM/tilemap loaders.
- `engine/`: Banked systems grouped by domain: `battle/` (core turn engine, move effects, AI), `overworld/` (movement, collisions, auto movement, tilesets, wild mons, day care), `menus/`, `items/`, `pokemon/` (load/learn/remove/add mon data), `movie/` (intro/Oak speech/title), `link/`, `events/`, `gfx/` (HP bars, sprite OAM helpers), `debug/`, `slots/`.
- `audio/`: Music and SFX data plus headers per bank. Uses RGBDS music macros (`tempo`, `duty_cycle`, `note_type`, `octave`, `note`, `rest`, `sound_loop` etc.). Also contains cries and wave samples.
- `data/`: Structured tables (moves, types, trainers, growth rates, text pointer tables, map header banks, sprite sets, tilesets, town map entries, wild encounters, predef pointers).
- `gfx/`: Source PNGs and converted `.2bpp/.1bpp` assets for sprites, tilesets, and pics. `gfx/pics.asm`, `gfx/sprites.asm`, `gfx/tilesets.asm` gather assets into banks.
- `maps/` + `data/maps/` + `scripts/`: Block data (`*.blk`), headers (`data/maps/headers/*.asm`), object placements (`data/maps/objects/*.asm`), sprite sets/hide-show tables, and per-map scripts/text in `scripts/*.asm`.
- `text/` + `data/text/*.asm`: Text blobs and pointer tables; `text.asm` includes all map text banks.
- `ram/`: VRAM/WRAM/SRAM/HRAM layouts; `ram.asm` includes the subfiles. Key RAM macros in `macros/ram.asm`.
- `constants/`, `macros/`: Shared definitions/macros (bank-safe calls, coordinate helpers, table sizing, event flag helpers, script/text/audio macros).
- `tools/`, `scripts/`: Build helpers (include scanning, gfx tweaks, patch generator) invoked by `Makefile`.
- `vc/`: Assets/templates for Virtual Console patches.

## Conventions & Macros
- Constants enumerated via `const_def/const/shift_const`; tables often use `table_width` to enforce row size.
- Bankswitch helpers: `farcall/callfar/farjp/jpfar` call banked code; `homecall` temporarily switches to call from other banks to ROM0.
- Data helpers in `macros/data.asm` (`dbw/dwb/dn/dc/bigdw/dba`, BCD macros, `tmhm` bitmap builder). Coordinate macros in `macros/coords.asm` (`hlcoord`, `bgcoord`, `owcoord`, `event_displacement`) compute tilemap or overworld offsets.
- Script/text/audio DSL:
  - Text: `text/line/para/next/cont/page/done/prompt` and `text_*` command macros (TX_*).
  - Event flags: `CheckEvent`, `SetEvent`, `ResetEvent`, bit reuse macros to minimize reloads.
  - Map definitions: `map_header`, `end_map_header`, `connection`, `warp_event`, `bg_event`, `object_event`, `def_*` counters ensure counts stay within limits.
  - Audio: channel macros (`tempo`, `volume`, `duty_cycle`, `vibrato`, `note_type`, `toggle_perfect_pitch`, `sound_call/ret`, `sound_loop`).
- Naming: `w*` for WRAM, `s*` for SRAM, `h*` for HRAM, `v*` for VRAM symbols. Double-colon labels are exported. Game state flags live in `wStatusFlags*`, event flags in `wEventFlags`.

## Rendering & Timing
- VRAM layout (`ram/vram.asm`): unions for generic chars/tilemaps or context-specific (battle/menu font/pics, overworld NPC/tileset, title logo).
- Tilemap buffers in WRAM (`wTileMap`, backups, surrounding block cache) feed VRAM; coordinate macros index into these. `wShadowOAM` buffers sprites before DMA.
- `VBlank` (in `home/vblank.asm`) handles BG scroll registers, auto BG transfers (`AutoBgMapTransfer`, `VBlankCopyBgMap`), redraws dirty rows/cols, copies queued data (`VBlankCopy*`), DMA OAM, updates random/frame counters, runs audio update per bank, polls joypad. `DelayFrame` waits on `hVBlankOccurred`.
- Graphics loading: `home/pics.asm`, `home/reload_tiles.asm`, `home/reload_sprites.asm` decompress/transfer assets; sprites decompressed via `UncompressSpriteData/UncompressSpriteFromDE`.
- Tile animation/palette fades handled in `home/fade.asm`, `home/fade_audio.asm`, with palette slots defined in `constants/palette_constants.asm` and helpers `ldpal/RGB`.

## Map & Script System
- Map IDs/sizes in `constants/map_constants.asm`; header banks in `data/maps/map_header_banks.asm`; music per map in `data/maps/songs.asm`; hide/show tables in `data/maps/hide_show_data.asm`; sprite sets in `data/maps/sprite_sets.asm`.
- Each map: header (`data/maps/headers/*` via `map_header`, connection macros), object list (`data/maps/objects/*` with warps/signs/objects/terrain flags), block data (`maps/*.blk`), script (`scripts/*.asm`) describing entry/scene scripts, text pointer tables, trainers, and callbacks.
- `EnterMap`/`OverworldLoop` (in `home/overworld.asm`) load map data, reset variables, handle bike/surf checks, process joypad, interact (A for signs/NPCs, START for menu), manage warps, and trigger battles/wild encounters via `engine/overworld/wild_mons.asm`. Map scripts set `wCurMapScript`/flags and often run through `DisplayTextID`/`DisplayTextIDInit`.
- Event flow relies on `wEventFlags`, per-map script state bytes (`w<Game>CurScript` family in WRAM), and missable object flags. `engine/overworld/missable_objects.asm` toggles objects (e.g., picked-up items).

## Audio System
- Audio state in WRAM start (`wSoundID`, channel state arrays, tempos, fade/mute flags, ROM bank tracking). Uses three audio banks; each has headers + data (`audio/headers`, `audio/sfx`, `audio/music`).
- `home/audio.asm` selects default map music, fades, bankswitches (`wAudioROMBank`, `wAudioSavedROMBank`), and calls `PlaySound`. `VBlank` calls `Audio{1,2,3}_UpdateMusic`, handles low-health alarm, and preserves ROM bank.
- Music files define four channels (Square 1/2, Wave, Noise) using note macros; SFX defined similarly and invoked via `wNewSoundID`. Wave samples shared (`audio/wave_samples.asm`).
- See gbdev pandocs for register meanings; channel flags map onto APU registers in update routines under `engine/audio` in ROM0 (`home/audio` calls).

## Pokémon/Battle Data
- Base stats per species in `data/pokemon/base_stats/*.asm` (dex id, stats, types, catch rate, base exp, pic pointers, level 1 moves, growth rate, TM/HM bitmap). Names in `data/pokemon/names.asm`; encounters/cry banks in `data/wild` and `audio/sfx/cry*.asm`.
- Moves in `data/moves/moves.asm` (animation/effect/power/type/accuracy/PP) with effects implemented in `engine/battle/move_effects/*.asm`. Types, critical hit data, and animations in `constants/type_constants.asm`, `engine/battle/animations.asm`, `data/battle_anims`.
- Trainers: classes/constants in `constants/trainer_constants.asm`; parties in `data/trainers/parties.asm`; AI/party reading in `engine/battle/read_trainer_party.asm` and `engine/battle/trainer_ai.asm`.
- Battle flow anchored in `engine/battle/core.asm` plus helpers (HUD drawing, transitions, status application, PP decrement, experience). Safari mechanics live in `engine/battle/safari_zone.asm`; wild encounter generation in `engine/battle/wild_encounters.asm`.

## Save/Data Storage
- WRAM schemas in `ram/wram.asm`; key struct macros in `macros/ram.asm` (`box_struct`, `party_struct`, `battle_struct`, sprite state structs, `map_connection_struct`).
- Main persistent chunk (`wMainDataStart`..`wMainDataEnd`): Pokédex owned/seen flags, bag items, money (3-byte BCD), rival name/options/badges, map/tileset pointers, coordinates, warp/sign/sprite tables, missable object flags, map script state bytes, safari counters, fossils, bike/surf state, progress flags, and play time counters.
- Party data: `wPartyCount`, `wPartySpecies`, `wPartyMon*` (party_struct with box data + calculated stats), OT/nick arrays, and loaded mon cache `wLoadedMon`. Enemy party mirrors in `wEnemyMons` unioned with wild data.
- PC storage: Current box (`wBox*` in WRAM), other boxes in SRAM banks (`sBox1..sBox12`) with checksums per bank/box; current box number in `wCurrentBoxNum`.
- Other SRAM: `sGameData` (player name, main data blob, sprite/party/box data, tile animations), sprite buffers, Hall of Fame teams, checksums. Save section sizes mirror WRAM symbols; edits must keep struct sizes stable.

## Gameplay Outline
- Standard Red/Blue arc: start in Pallet Town (get starter in Oak’s Lab), rival encounters, traverse Kanto’s towns/routes (map ids in `constants/map_constants.asm`), collect eight badges (Gyms in Pewter/Cerulean/Vermilion/Celadon/Fuchsia/Saffron/Cinnabar/Viridian), clear Team Rocket plots (Mt. Moon, Game Corner hideout, Pokémon Tower, Silph Co.), explore dungeons (Seafoam, Victory Road, Cerulean Cave), defeat Elite Four/Champion at Indigo Plateau, and optional Safari Zone for HM/progress items.
- Wild data per map in `data/wild/grass_water.asm`; tilesets determine encounters and collision; special warps (Fly/dungeon) handled in `engine/overworld/special_warps.asm`.
- Player position tracked as block/tile coords (`wXCoord/wYCoord`, `wXBlockCoord/wYBlockCoord`), with current map header pointers describing layout and connections. Movement flags manage walking/biking/surfing, scripted motion, and ledge/jump states.
- Pokémon stats: battle struct holds species, HP, types, moves/PP, DVs/EVs (exp substats), level, and computed stats; status stored in `Status` byte. HP/PP use raw values; money/coins are BCD.
- PC boxes and party share `box_struct` layout; DVs and experience used to compute stats on load (`engine/pokemon/load_mon_data.asm`). PP/sleep/poison/burn status bytes persisted in saves.

## Common Patterns & Best Practices
- Always include `includes.asm` via `RGBASMFLAGS -P` to get macros/constants; new files should use `SECTION` with explicit banks (ROM0 vs ROMX).
- Use provided macros instead of manual offsets (event/map/object macros enforce limits; `hlcoord/bgcoord` prevent tile out-of-bounds; `tmhm` builds bitfields safely).
- When calling across banks, use `farcall/farjp`; to call ROM0 from banks use `homecall` variants. Preserve `hLoadedROMBank`/`rROMB` conventions.
- For text, compose with `text` macros and terminate with `done/prompt`; keep `@` terminator handling in mind (`dname` pads names, strings terminate with `@`).
- Rendering changes should queue data in tilemap buffers/OAM shadow; heavy VRAM writes belong in VBlank-safe routines. Mind palette offsets for Flash/dark rooms (`wMapPalOffset`).
- Save structure alignment matters: changes to WRAM structs ripple into SRAM saves. Reuse existing structs (`party_struct`, etc.) and adjust checksums if formats change.
- Reference GB/Z80 and RGBDS docs for instruction syntax, APU/GPU registers (see links in request) when tweaking low-level routines.

## Useful Starting Points
- Main loops: `home/start.asm`, `home/init.asm`, `home/overworld.asm` (overworld), `engine/battle/core.asm` (battles).
- Assets/pointers: `constants/*` for ids/flags, `data/maps/map_header_pointers.asm`, `data/maps/songs.asm`, `data/items/marts.asm`, `data/predef_pointers.asm`.
- Debug: `engine/debug/debug_menu.asm`, `engine/debug/debug_party.asm`; VC hooks in `vc/*.patch.template`.
