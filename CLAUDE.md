# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

`pokeemerald-expansion` (RHH) is a GBA ROM-hack base built on pret's `pokeemerald` decompilation. It is C (gnu17) + ARM assembly compiled with `arm-none-eabi` (devkitARM or system toolchain), always the "modern" toolchain (`-DMODERN=1`). It also supports building FireRed/LeafGreen variants from the same source. See `INSTALL.md` for toolchain setup and `docs/` (published at https://rh-hideout.github.io/pokeemerald-expansion/) for tutorials.

## Commands

```sh
make -j$(nproc)                 # build pokeemerald.gba (Emerald)
make firered -j$(nproc)         # FireRed build (GAME_VERSION=FIRERED); leafgreen likewise
make debug -j$(nproc)           # -Og -g build, separate obj dir
make release -j$(nproc)         # optimized, -DRELEASE (NDEBUG, disables debug features)
make check -j$(nproc)           # build pokeemerald-test.elf and run all tests headlessly via mgba-rom-test-hydra
make check TESTS="Stun Spore"   # run only tests whose name starts with the given prefix
make pokeemerald-test.elf TESTS="Spikes"   # build a test ROM to step through in mGBA
make tidy                       # remove build outputs (keeps tools/generated assets); `make clean` removes everything
```

CI (`.github/workflows/build.yml`) runs: `make all`, `make firered`, `make leafgreen`, `make release`, `make check`. Each build mode uses its own object dir under `build/` (`<name>-test`, `-debug`, `-release`), so switching modes doesn't force a full rebuild.

## Architecture

- `src/` — game C code. `include/` — headers; `include/constants/` holds IDs/enums (species, moves, abilities, items, flags, vars).
- `include/config/*.h` — feature toggles and generation-behaviour configs (e.g. `B_*` battle configs in `battle.h`, `P_*` in `pokemon.h`, `OW_*` in `overworld.h`, AI in `ai.h`, test settings in `test.h`). Per the style guide, check config defines inside normal control flow (`if (B_FOO >= GEN_6)`) rather than `#if`, unless a data structure must change.
- **Data tables** live in `src/data/`: `moves_info.h` (gMovesInfo, all move data), `abilities.h`, `items.h`, `pokemon/species_info/gen_N_families.h` (species data grouped by family), learnsets, `battle_move_effects.h` (maps `EFFECT_*` to battle scripts).
- **Generated files — do not edit directly**: `src/data/trainers.h`, `trainers_frlg.h`, `battle_partners.h`, `debug_trainers.h`, and `test/battle/*_control.h` are generated from the matching `.party` files by `tools/trainerproc`. Map headers/events are generated from `data/maps/*/map.json` and `data/layouts/` JSON (`map_data_rules.mk`). Other generated assets are listed in the `*_rules.mk` files.
- **Battle engine**:
  - `battle_main.c` (turn loop/action order), `battle_script_commands.c` (`gBattleScriptingCommandsTable`, the bytecode interpreter for `data/battle_scripts_*.s`), `battle_util.c` (abilities/items/shared helpers), plus split-out modules (`battle_move_resolution.c`, `battle_end_turn.c`, `battle_switch_in.c`, `battle_hold_effects.c`, `battle_stat_change.c`, gimmicks: `battle_dynamax.c`, `battle_terastal.c`, `battle_z_move.c`, `battle_gimmick.c`).
  - Battle scripts are assembly macros (`asm/macros/battle_script.inc`) in `data/battle_scripts_1.s`.
  - Controllers (`battle_controller_*.c`) handle per-battler UI/input and communicate via buffered commands; tests rely on this to observe player-visible output.
  - AI: `battle_ai_main.c`, `battle_ai_util.c`, `battle_ai_switch.c`, `battle_ai_items.c`; see `docs/tutorials/ai_*.md`.
- **Scripts/maps**: overworld event scripts are `.inc` files under `data/maps/` and `data/scripts/`; script commands in `data/script_cmd_table.inc`, specials in `data/specials.inc`.

## Tests

Tests live in `test/` (battle tests under `test/battle/`, organized by ability, move_effect, hold_effect, etc.) and run inside the ROM on mGBA. Full guide: `docs/tutorials/how_to_testing_system.md`. Key conventions:

- `SINGLE_BATTLE_TEST` / `DOUBLE_BATTLE_TEST` with `GIVEN` (parties) → `WHEN` (`TURN { MOVE(...) }`) → `SCENE` (player-visible checks: `ANIMATION`, `MESSAGE`, `HP_BAR`, `STATUS_ICON`, `ABILITY_POPUP`, `NOT ...`) → optional `THEN`/`FINALLY`.
- Test names should start with a shared prefix (e.g. the move/ability name) so `TESTS="..."` filters them.
- Put data dependencies in `ASSUMPTIONS { ASSUME(...); }` or `ASSUME` inside `GIVEN`, so hacks that change data skip rather than fail.
- Prefer asserting on what the player sees (SCENE) over internal state like `gBattleMons`.
- `PARAMETRIZE` for variants; `PASSES_RANDOMLY(successes, trials, tag)` for RNG (slow — prefer forcing RNG via `WITH_RNG`/tags where possible).
- Use `SPECIES_WOBBUFFET` as the neutral default mon.

## Style (docs/STYLEGUIDE.md)

- 4 spaces in `.c`/`.h`; tabs in `.s` and `.inc`.
- `PascalCase` functions/structs, `camelCase` variables/fields, `gGlobal`, `sStatic`, `CAPS_WITH_UNDERSCORES` macros/constants.
- Allman braces; `switch` cases aligned with the `switch`; one space after `if`/`for`/`while`.
- Comments explain *why*, are capitalized and end with a period.

## Working rules

- After every change, run `make -j$(nproc)` and fix all errors before stopping.
- Make one feature per commit, with a clear commit message.
- Never edit files under `tools/` unless the user asks.

## Design decisions

The hack's design decisions are listed here. Add entries only when the user specifies them.

_None yet._

## Contribution branches

PRs that fix bugs present in `master` target `master`; everything else (features, refactors) targets `upcoming`. See `CONTRIBUTING.md`; version migration scripts for downstream hacks live in `migration_scripts/`.
