# FF16Tools - Claude Instructions

## Overview

This is FF16Tools, a C# toolkit for unpacking and modding Final Fantasy Tactics: The Ivalice Chronicles (FFT:IVC) game files. The goal is to unpack the game's `.pac` archives and convert `.nxd` database files to SQLite for inspection and modding.

## Prerequisites

- **Windows** (required — uses DirectStorage for GDeflate decompression, which is Windows-only)
- **.NET 9.0 SDK** installed (`dotnet --version` should show 9.x)

## Build

```bash
cd FF16Tools
dotnet build FF16Tools.CLI/FF16Tools.CLI.csproj -c Release
```

Warnings are expected and harmless. Should produce 0 errors.

## Step 1: Unpack .pac files

The game data is at the Steam install path. Find it with:
```bash
# Typical location:
# C:\Program Files (x86)\Steam\steamapps\common\FINAL FANTASY TACTICS - The Ivalice Chronicles\data\
```

There are two data directories: `data\classic\` and `data\enhanced\`. Unpack both:

```bash
dotnet run --project FF16Tools.CLI -c Release -- unpack-all-packs -i "<STEAM_PATH>\data\enhanced" -o ".\unpacked\enhanced" -g fft -s
dotnet run --project FF16Tools.CLI -c Release -- unpack-all-packs -i "<STEAM_PATH>\data\classic" -o ".\unpacked\classic" -g fft -s
```

Replace `<STEAM_PATH>` with the actual Steam game directory. The `-g fft` flag is required (default is ffxvi). The `-s` flag skips confirmation prompts.

This will extract all game assets including:
- `nxd/` — database files (.nxd) — the main target for data modding
- `ui/` — UI textures and layouts
- `bg/` — background/map textures
- `movie/` — cutscene videos (.usm)
- `script/` — event scripts
- `system/` — fonts, system data
- `fftpack/` — localized game data

## Step 2: Convert .nxd to SQLite

The `.nxd` files are binary database tables. Convert them to SQLite for easy viewing/editing:

```bash
# Convert enhanced nxd files (has more tables than classic)
dotnet run --project FF16Tools.CLI -c Release -- nxd-to-sqlite -i ".\unpacked\enhanced\nxd" -o ".\unpacked\enhanced\nxd\fft_enhanced.sqlite" -g fft

# Convert classic nxd files
dotnet run --project FF16Tools.CLI -c Release -- nxd-to-sqlite -i ".\unpacked\classic\nxd" -o ".\unpacked\classic\nxd\fft_classic.sqlite" -g fft
```

The layout definitions that map binary columns to named fields come from the git submodule at `FF16Tools.Files/Nex/Layouts/ffto/`. These correspond to the `fftivc-nex-layouts` repo. If the submodule isn't populated:

```bash
git submodule update --init --recursive
```

## Step 3: Explore the SQLite databases

Use any SQLite viewer (SQLiteStudio, DB Browser for SQLite, or the sqlite3 CLI) to browse the converted data. Tables include game data for abilities, jobs, items, battles, maps, UI, characters, etc.

## Key Architecture Notes

- The solution has 7 projects. The CLI (`FF16Tools.CLI`) is the main entry point.
- `.pac` files use the PACK format with GDeflate compression (via DirectStorage).
- `.nxd` files use the "Nex" binary database format. Layout files (`.layout`) define the schema.
- The `-g fft` flag switches between FFXVI and FFT:IVC format handling.
- Enhanced version has ~195 nxd tables vs ~77 for classic. Enhanced is the more complete dataset.
- Localized data lives in language-suffixed files (e.g., `ability.en.nxd`, `ability.ja.nxd`).

## Modding Workflow (if needed later)

1. Edit tables in SQLite
2. Convert back: `sqlite-to-nxd -i <sqlite_file> -o <output_dir> -g fft`
3. Repack into `.diff.pac` for the game to load
4. Use Reloaded-II mod loader for proper mod management

## Troubleshooting

- If decompression fails with DirectStorage errors, ensure you're on Windows with DirectStorage support
- If nxd-to-sqlite warns about missing layouts, ensure git submodules are initialized
- The tool requires the game to be installed but NOT running
