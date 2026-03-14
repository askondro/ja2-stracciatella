# JA2-Stracciatella — Architecture Documentation

> **Version:** 0.22.x-dev
> **C++ Standard:** C++20
> **Last Updated:** 2026-03-14

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Repository Layout](#2-repository-layout)
3. [Technology Stack](#3-technology-stack)
4. [Startup & Initialization Flow](#4-startup--initialization-flow)
5. [Main Game Loop](#5-main-game-loop)
6. [Screen System](#6-screen-system)
7. [SGP — System Gaming Platform](#7-sgp--system-gaming-platform)
8. [Game Subsystems](#8-game-subsystems)
   - 8.1 [Tactical (Combat)](#81-tactical-combat)
   - 8.2 [TacticalAI (Enemy AI)](#82-tacticalai-enemy-ai)
   - 8.3 [Strategic (Campaign Map)](#83-strategic-campaign-map)
   - 8.4 [TileEngine (Rendering)](#84-tileengine-rendering)
   - 8.5 [Laptop (In-Game UI)](#85-laptop-in-game-ui)
   - 8.6 [Editor (Map Editor)](#86-editor-map-editor)
9. [Content & Externalized Data System](#9-content--externalized-data-system)
10. [Rust FFI Layer](#10-rust-ffi-layer)
11. [Build System](#11-build-system)
12. [Data Flow Diagram](#12-data-flow-diagram)
13. [Key Design Patterns](#13-key-design-patterns)
14. [Known Technical Debt](#14-known-technical-debt)
15. [Platform Support Matrix](#15-platform-support-matrix)

---

## 1. Project Overview

JA2-Stracciatella is a cross-platform continuation of the **Jagged Alliance 2** source port. Its goals are:

- Run the original game on modern operating systems without patching the data files
- Fix bugs present in the original release
- Provide a stable modding foundation via externalized JSON data
- Maintain fidelity to the original gameplay while allowing optional extensions

The project is a **C++20 application** backed by a **Rust library** for file I/O and resource management, using **SDL2** for graphics and input and **Lua** for in-game scripting.

---

## 2. Repository Layout

```
ja2-stracciatella/
├── assets/
│   ├── externalized/       # JSON game data (items, weapons, NPCs, policies…)
│   ├── mods/               # Bundled mod definitions
│   └── unittests/          # Test data assets
│
├── cmake/                  # CMake helper modules and toolchain files
├── dependencies/           # Vendored third-party libraries
│   ├── fltk/               # GUI toolkit for launcher
│   ├── google-test/        # GTest unit test framework
│   ├── lua/                # Lua 5.4 interpreter
│   ├── magic_enum/         # Enum reflection (header-only)
│   ├── miniaudio/          # Audio backend (header-only)
│   ├── smacker/            # Smacker video decoder
│   ├── sol2/               # Lua C++ bindings (header-only)
│   └── string_theory/      # Unicode-safe string library
│
├── docs/                   # Project documentation
│   ├── ARCHITECTURE.md     # This document
│   └── Release-checklist.md
│
├── rust/                   # Rust workspace
│   ├── stracciatella/      # Core Rust library
│   ├── stracciatella_bin/  # Standalone CLI tools
│   └── stracciatella_c_api/ # C FFI bindings (called from C++)
│
├── src/
│   ├── externalized/       # C++ content system (ContentManager, models)
│   ├── game/               # Game logic
│   │   ├── Editor/         # Map editor (52 files)
│   │   ├── Laptop/         # In-game laptop UI (117 files)
│   │   ├── Res/            # Resource helpers
│   │   ├── Strategic/      # Campaign map layer (91 files)
│   │   ├── Tactical/       # Turn-based combat (163 files)
│   │   ├── TacticalAI/     # Enemy AI (20 files)
│   │   ├── TileEngine/     # Isometric tile renderer (80 files)
│   │   └── Utils/          # Shared game utilities
│   ├── launcher/           # FLTK GUI launcher
│   └── sgp/                # System Gaming Platform (38 files)
│
├── CMakeLists.txt          # Root build definition
├── COMPILATION.md          # Platform-specific build instructions
├── CONTRIBUTING.md         # Coding standards and workflow
├── README.md               # User-facing overview
└── changes.md              # Changelog
```

---

## 3. Technology Stack

| Layer | Technology | Version | Purpose |
|---|---|---|---|
| Language | C++ | C++20 | Core game logic, rendering, UI |
| Language | Rust | stable | File I/O, JSON, resource packing |
| Scripting | Lua | 5.4 | In-game event scripting |
| Graphics/Input | SDL2 | ≥ 2.0.5 | Window, renderer, input events |
| GUI (Launcher) | FLTK | bundled | Cross-platform launcher window |
| Audio | miniaudio | bundled | Audio playback backend |
| Lua bindings | sol2 | bundled | Type-safe Lua/C++ FFI |
| Enum reflection | magic_enum | 0.9.7 | Compile-time enum utilities |
| Strings | string_theory | 3.8 | Unicode-aware string class |
| Video | Smacker | bundled | Intro video playback |
| Testing | GTest | bundled | Unit test framework |
| Build | CMake | ≥ 3.18.1 | Build orchestration |
| Build | Cargo | stable | Rust build system |
| CI (Windows) | AppVeyor | — | MSVC x86/x64, MinGW builds |
| CI (Linux/macOS) | GitHub Actions | — | GCC, Clang builds |
| Static analysis | Coverity Scan | — | Defect detection |

---

## 4. Startup & Initialization Flow

```
OS → main()                             [src/sgp/SGP.cc]
       │
       ├─ Parse CLI options             [Rust: config::EngineOptions]
       ├─ Init SDL2 (video, audio)
       ├─ Init Video Manager            [SGP: Video.cc]
       ├─ Init Sound Manager            [SGP: SoundMan.cc]
       ├─ Init Button System            [SGP: Button_System.cc]
       ├─ Create ContentManager        [externalized: DefaultContentManager]
       │     └─ Load JSON data files   [Rust: json/, vfs/]
       │
       ├─ InitializeGame()             [game/Init.cc]
       │     ├─ Animation system
       │     ├─ Lighting system
       │     ├─ Dialogue system
       │     ├─ Strategic engine
       │     ├─ Tactical engine
       │     ├─ World / tile cache
       │     └─ Font & help system
       │
       └─ MainLoop()                   [SGP.cc] ← runs until quit
```

### ContentManager Lifecycle

```
DefaultContentManager (or ModPackContentManager)
       │
       ├─ Discovers vanilla game data directory
       ├─ Initialises Rust VFS layers (SLF, directory, Android)
       ├─ Loads & validates all JSON files via Rust json/ module
       ├─ Constructs model objects (WeaponModel, ItemModel, TownModel…)
       └─ Exposes unified interface to C++ game code via GCM global
```

---

## 5. Main Game Loop

**File:** `src/game/GameLoop.cc`, `src/sgp/SGP.cc`

```
MainLoop() — 144 Hz target
    │
    ├─ Poll SDL events
    │     ├─ SDL_KEYDOWN / SDL_KEYUP  → SGP input queue
    │     ├─ SDL_MOUSEMOTION / BUTTON → MouseSystem
    │     ├─ SDL_FINGERDOWN / MOVE    → touch (Android)
    │     └─ SDL_QUIT                 → exit flag
    │
    ├─ GameLoop()
    │     ├─ Handle pending screen transition (guiPendingScreen)
    │     │     ├─ Shutdown current screen
    │     │     └─ Initialize new screen
    │     └─ Dispatch to active screen handler (fn pointer table)
    │           e.g. HandleTacticalScreen(), HandleMapScreen()…
    │
    ├─ Blit frame to SDL window (Video Manager)
    └─ FPS cap / sleep
```

### Screen State Machine

```
guiCurrentScreen ──────────────────────────────────────────────────
        │
        ▼
 MAINMENU_SCREEN ──(new game)──► INTRO_SCREEN ──► GAME_SCREEN
                                                        │
                                              ┌─────────┴──────────┐
                                              ▼                     ▼
                                    TACTICAL_SCREEN          MAPSCREEN_SCREEN
                                              │                     │
                                              └─────────┬───────────┘
                                                        ▼
                                              SAVE / LOAD SCREENS
                                                        │
                                                   EDIT_SCREEN (editor)
```

Screen transitions are requested via `SetPendingNewScreen(screen_id)` and applied at the top of the next `GameLoop()` iteration.

---

## 6. Screen System

Each screen is registered with three function pointers:

```cpp
struct GameScreen {
    ScreenID  (*InitializeScreen)();
    ScreenID  (*HandleScreen)();
    void      (*ShutdownScreen)();
};
```

| Screen ID | Description |
|---|---|
| `MAINMENU_SCREEN` | Main menu |
| `GAME_SCREEN` | Tactical combat view |
| `MAPSCREEN_SCREEN` | Strategic campaign map |
| `EDIT_SCREEN` | Map editor |
| `LAPTOP_SCREEN` | In-game laptop/email UI |
| `SAVE_LOAD_SCREEN` | Save/load game dialog |
| `INTRO_SCREEN` | Opening cinematics |
| `FADE_SCREEN` | Transition fade |
| `ERROR_SCREEN` | Error display |

---

## 7. SGP — System Gaming Platform

**Directory:** `src/sgp/` (38 files)

The SGP is the **platform abstraction layer** between the game code and the OS/SDL2. All platform-specific code lives here so the rest of the game is portable.

### 7.1 Subsystems

#### Video

| File | Responsibility |
|---|---|
| `Video.h/cc` | Video manager: SDL window, renderer lifecycle |
| `VSurface.h/cc` | In-memory surfaces; software blitting |
| `VObject.h/cc` | Sprite/image objects; palette management |
| `VObject_Blitters.h/cc` | Pixel-level blit operations |
| `HImage.h/cc` | Generic image container |
| `PCX.h/cc`, `ImpTGA.h/cc` | Image format loaders |
| `Shading.h/cc` | Lighting shading tables |

#### Input

| File | Responsibility |
|---|---|
| `Input.h/cc` | Keyboard event queue, key state |
| `MouseSystem.h/cc` | Mouse position, button state, region callbacks |
| `Button_System.h/cc` | Retained-mode button UI widgets |
| `Cursor_Control.h/cc` | Cursor image management |

#### File I/O

| File | Responsibility |
|---|---|
| `FileMan.h/cc` | Cross-platform file system operations |
| `SGPFile.h/cc` | Abstraction over real files and SLF archives |
| `DirFs.h/cc` | Directory-backed virtual file system |

#### Audio

| File | Responsibility |
|---|---|
| `SoundMan.h/cc` | Sound effect and music playback via miniaudio |

#### Utilities

| File | Responsibility |
|---|---|
| `Font.h/cc` | Bitmap font rendering |
| `Logger.h` | Structured logging |
| `Timer.h` | High-resolution timing |
| `Random.h/cc` | PRNG utilities |
| `FPS.h/cc` | Frame rate measurement and capping |
| `LoadSaveData.h/cc` | Binary serialization helpers |
| `Buffer.h` | Typed memory buffer |
| `Object_Cache.h/cc` | LRU resource cache |

---

## 8. Game Subsystems

### 8.1 Tactical (Combat)

**Directory:** `src/game/Tactical/` — 163 files

The largest subsystem. Handles everything that happens inside a combat sector.

#### Responsibilities

- **Turn management** — action point (AP) accounting, turn order, end-of-turn processing (`Overhead.cc`)
- **Soldier simulation** — character state machine, animation, pathfinding, facing, crouching (`Soldier_Control.cc`)
- **Combat mechanics** — to-hit calculation, hit resolution, suppression, morale, bleeding (`Weapons.cc`, `Points.cc`)
- **Line of Sight / FOV** — occlusion, vision cones, darkness (`LOS.cc`, `FOV.cc`)
- **Items** — inventory, equip/unequip, usage (`Items.cc`, `Handle_Items.cc`, `Interface_Items.cc`)
- **Dialogue** — NPC conversation triggers and responses (`Dialogue_Control.cc`)
- **Doors & interactables** — door states, locks, traps (`Handle_Doors.cc`, `Keys.cc`)
- **Physics** — bullet trajectories, explosions, fire spread (`Bullets.cc`)
- **UI** — HUD, action cursor, radial menus (`Handle_UI.cc`, `Interface_Panels.cc`)
- **Persistence** — save/load of tactical state (`LoadSave*.cc`)
- **Special mechanics** — Boxing, Drug effects, Vehicles

#### Key Files by Size

| File | ~Lines | Role |
|---|---|---|
| `Soldier_Control.cc` | 9,122 | Character state machine |
| `Overhead.cc` | 6,240 | Turn management |
| `Interface_Items.cc` | 5,388 | Item manipulation UI |
| `Handle_UI.cc` | 5,351 | UI event router |
| `OppList.cc` | 5,090 | Opponent awareness tracking |
| `LOS.cc` | 4,600 | Line of sight |
| `Weapons.cc` | 3,904 | Ballistics & damage |

---

### 8.2 TacticalAI (Enemy AI)

**Directory:** `src/game/TacticalAI/` — 20 files

Governs how computer-controlled soldiers behave during the player's combat turns.

#### Decision Pipeline

```
AIMain.cc — per-soldier AI tick
    │
    ├─ Determine threat awareness (OppList)
    ├─ Select goal: attack / move / cover / flee / idle
    │
    ├─ DecideAction.cc — choose specific action
    │     ├─ Evaluate attack options      [Attacks.cc]
    │     ├─ Evaluate movement positions  [AIUtils.cc]
    │     └─ Evaluate special actions     [NPC.cc]
    │
    └─ Execute chosen action → Tactical subsystem
```

#### Key Files

| File | Role |
|---|---|
| `AIMain.cc` | Per-soldier AI orchestration |
| `DecideAction.cc` | Action selection logic |
| `Attacks.cc` | Attack option evaluation |
| `AIUtils.cc` | Pathfinding, position scoring |
| `NPC.cc` | NPC-specific behaviours, squad tactics |

---

### 8.3 Strategic (Campaign Map)

**Directory:** `src/game/Strategic/` — 91 files

The campaign layer that sits above individual combat sectors.

#### Responsibilities

- **Map display** — scrollable strategic map with sector icons (`MapScreen.cc`)
- **Squad management** — merc assignments, travel orders, morale (`Assignments.cc`)
- **Movement** — squad travel with time simulation and interrupts (`Strategic_Movement.cc`, `Strategic_Pathing.cc`)
- **Enemy AI** — enemy army movements, reinforcements, town captures (`Strategic_AI.cc`)
- **Auto-resolve** — simulate battles without entering tactical mode (`Auto_Resolve.cc`)
- **Economy** — mine income, merc pay, financial tracking
- **Town loyalty** — civilian attitude, rebel support
- **Quests** — quest flags, triggers, NPC recruitment (`Quests.cc`)
- **Time** — game clock, scheduled events, turn advancement (`Strategic_Turns.cc`)

#### Key Files

| File | ~Lines | Role |
|---|---|---|
| `MapScreen.cc` | 8,025 | Strategic map display |
| `Assignments.cc` | 7,031 | Squad assignment UI |
| `Strategic_AI.cc` | 3,831 | Enemy strategic decisions |
| `Auto_Resolve.cc` | 3,779 | Battle simulation |

---

### 8.4 TileEngine (Rendering)

**Directory:** `src/game/TileEngine/` — 80 files

The isometric tile renderer. Converts the game world grid into pixels on screen.

#### Responsibilities

- **World data** — tile grid definition, tile type registry (`WorldDat.cc`, `WorldDef.cc`)
- **Rendering** — visible tile determination, z-ordering, dirty-rect optimisation (`RenderWorld.cc`, `Render_Dirty.cc`)
- **Lighting** — per-tile light levels, dynamic light sources, ambient (`Lighting.cc`)
- **Tile animation** — animated tiles (water, fire, flags) (`Tile_Animation.cc`)
- **Tile cache** — load/evict tile graphics from memory (`Tile_Cache.cc`)
- **Isometric math** — grid ↔ screen coordinate conversion (`Isometric_Utils.cc`)
- **Explosions** — blast radius, debris animation, damage tiles (`Explosion_Control.cc`)
- **Interactive tiles** — doors, switches, triggers (`Interactive_Tiles.cc`)
- **Ambient** — environmental sound and visual ambiance (`Ambient_Control.cc`)

---

### 8.5 Laptop (In-Game UI)

**Directory:** `src/game/Laptop/` — 117 files

Implements the in-game "Arulco Vitals Information Director" (A.I.M.) laptop computer the player uses between missions.

#### Screens / Sections

| Section | Files | Purpose |
|---|---|---|
| AIM | `AIM.cc`, `AIMMembers.cc`, `AIMSort.cc` | Hire mercenaries |
| M.E.R.C. | `Mercs.cc` | Alternative cheaper mercs |
| Bobby Ray's | `Bobby_R*.cc` | Online weapons shop |
| Email | `Email.cc`, `EmailList.cc` | In-game email inbox |
| Personnel | `Personnel.cc` | Hired merc roster |
| History | `History.cc` | Campaign timeline |
| Insurance | `Insurance.cc` | Merc life insurance |
| Florist | `Florist*.cc` | Optional gift service |
| Financial | (integrated) | Bank balance, transactions |

---

### 8.6 Editor (Map Editor)

**Directory:** `src/game/Editor/` — 52 files

A full map creation and editing tool shipped with the game binary (activated via `--editor` flag or `GAME_MODE_EDITOR`).

#### Capabilities

- Place / remove tiles, roofs, structures, roads
- Place NPCs and mercs with inventory and schedules
- Place items, triggers, lights
- Undo / redo history
- Save / load `.MAP` files
- Road and terrain smoothing tools

#### Key Files

| File | Role |
|---|---|
| `EditScreen.cc` | Main editor screen handler |
| `EditorMercs.cc` | Soldier placement and configuration |
| `Edit_Sys.cc` | Editor state management |
| `Smoothing_Utils.cc`, `Smooth.cc` | Terrain auto-smoothing |
| `PopupMenu.cc`, `SelectWin.cc` | Editor UI widgets |

---

## 9. Content & Externalized Data System

**C++ Directory:** `src/externalized/` — 29 files
**JSON Data:** `assets/externalized/`

All game balance and entity data is stored as **JSON files** outside the binary. This enables modding without recompilation and clean separation of data from logic.

### Interface Hierarchy

```
ContentManager  (ContentManager.h)
  ├─ ItemSystem   (ItemSystem.h)
  │     ├─ getWeapon(index) → WeaponModel*
  │     ├─ getMagazine(index) → MagazineModel*
  │     ├─ getExplosive(index) → ExplosiveModel*
  │     └─ getArmour(index) → ArmourModel*
  │
  └─ MercSystem   (MercSystem.h)
        ├─ getMercProfile(id) → MercProfile*
        └─ getMercById(id) → MercProfileModel*
```

`ContentManager` additionally provides:

| Category | Examples |
|---|---|
| File access | `openGameResForReading()`, `doesGameResExists()` |
| Strategic data | `getTown()`, `getMine()`, `getSamSites()` |
| Game policies | `getGamePolicy()`, `getStrategicAIPolicy()` |
| Loading screens | `getLoadingScreen(sectorId)` |
| NPC data | `getNpcPlacement()`, `loadNpcQuoteRecord()` |
| Music | `getMusicForMode()` |

### Implementations

| Class | When Used |
|---|---|
| `DefaultContentManager` | Normal game startup |
| `ModPackContentManager` | When a mod is active (adds JSON patch layer) |

The active instance is held in the `GCM` global pointer (`GameInstance.h`) and accessed throughout the codebase.

### JSON Loading Pipeline

```
assets/externalized/*.json
         │
         ▼ (Rust: vfs layer)
  Load raw bytes from VFS
         │
         ▼ (Rust: json/ module)
  Strip comments, apply JSON Patch (mods)
         │
         ▼ (Rust → C++ FFI)
  Validate against schema
         │
         ▼ (C++: JsonUtility.cc)
  Deserialise into model objects
         │
         ▼
  ContentManager exposes typed API
```

### Key JSON Data Files

| File | Contents |
|---|---|
| `items.json` | All item definitions |
| `weapons.json` | Weapon stats, attachments |
| `magazines.json` | Ammo calibre/capacity |
| `armours.json` | Armour protection values |
| `explosives.json` | Grenade / mine definitions |
| `dealers.json` | Arms dealer inventories |
| `npc-placements.json` | NPC spawn positions |
| `game-policy.json` | Core gameplay rules |
| `strategic-ai-policy.json` | Enemy AI parameters |
| `towns.json` | Town definitions |
| `mines.json` | Mine locations and yields |

---

## 10. Rust FFI Layer

**Directory:** `rust/`

The Rust layer is compiled to a **static library** (`stracciatella_c_api`) that is linked into the C++ executable. C++ calls into Rust through a C-compatible header (`RustInterface.h`).

### Crate Structure

```
rust/
├── stracciatella/          # Core library
│   └── src/
│       ├── config/         # CLI parsing, EngineOptions, vanilla version detection
│       ├── file_formats/   # SLF archive, STCI image format parsers
│       ├── fs/             # Path utilities, free-space checks
│       ├── vfs/            # Virtual filesystem (dir, SLF, Android)
│       ├── json/           # JSON loading, comment stripping, patch merging
│       ├── mods/           # Mod manifests, mod loading
│       ├── logger/         # Logging backend
│       ├── unicode/        # UTF-8 normalisation, charset detection
│       ├── math/           # Shared math utilities
│       └── schemas/        # Data structure schemas
│
├── stracciatella_bin/      # Standalone tools (ja2-resource-pack, etc.)
└── stracciatella_c_api/    # C FFI surface exposed to C++
    └── build.rs            # Generates C header from Rust types
```

### Virtual Filesystem (VFS)

The VFS is the key abstraction for resource loading:

```
VFS
 ├─ DirLayer    — reads from a real directory (modded content, patches)
 ├─ SlfLayer    — reads from .SLF archive files (vanilla game data)
 └─ AndroidLayer— reads from APK assets (Android builds)

Lookup order: DirLayer (highest priority) → SlfLayer → AndroidLayer
```

This means mods simply place files in the mod directory; if the path exists there, it shadows the vanilla SLF entry transparently.

### Responsibilities Summary

| Module | C++ Benefit |
|---|---|
| `vfs/` | Transparent mod/vanilla file resolution |
| `json/` | Safe JSON loading with comment support and patch merging |
| `config/` | CLI option parsing and settings persistence |
| `file_formats/` | SLF / STCI parsing without C++ buffer overflows |
| `unicode/` | Safe UTF-8 handling for paths and strings |
| `fs/` | Portable disk-space checks and path canonicalisation |
| `mods/` | Mod manifest parsing and activation |

---

## 11. Build System

### CMake Targets

| Target | Output | Description |
|---|---|---|
| `ja2` | executable (`.exe` / binary) | Main game (shared lib on Android) |
| `ja2-launcher` | executable | FLTK GUI launcher |
| `ja2_tests` | test executable | GTest unit tests |

### Build Profiles

| Profile | Flags | Use Case |
|---|---|---|
| `Debug` | no optimisation, symbols | Development, debugging |
| `RelWithDebInfo` | `-O2` + symbols | Default; recommended for testing |
| `Release` | full optimisation, no symbols | Distribution builds |

### Compilation Flags

| Compiler | Key Flags |
|---|---|
| MSVC | `/W3 /utf-8` |
| GCC / Clang | `-Wall -Wtype-limits -Wignored-qualifiers` |
| MinGW | `-Wa,-mbig-obj` (required for sol2 template expansion) |

> **Note:** MSVC warnings 4018 (signed/unsigned mismatch), 4244 (conversion loss), and 4996 (unsafe C functions) are currently suppressed due to pervasive legacy code patterns.

### Packaging (CPack)

| Platform | Format |
|---|---|
| Windows | NSIS installer |
| Linux | `.deb` package |
| macOS | `.app` bundle |

### Dependency Resolution

```
CMakeLists.txt
  ├─ find_package(SDL2)     — system SDL2 or bundled (Windows/macOS)
  ├─ add_subdirectory(dependencies/lua)
  ├─ add_subdirectory(dependencies/google-test)
  ├─ add_subdirectory(dependencies/fltk)  [if launcher enabled]
  └─ ExternalProject / FetchContent for Rust crates (via Cargo)
```

---

## 12. Data Flow Diagram

```
┌──────────────────────────────────────────────────────────┐
│                    Vanilla Game Data                      │
│           (.SLF archives, .MAP files, sounds…)           │
└───────────────────────┬──────────────────────────────────┘
                        │
                        ▼
              ┌─────────────────┐
              │  Rust VFS Layer │◄── Mod JSON patches (assets/mods/)
              │  (SLF + DirFs)  │◄── Externalized JSON (assets/externalized/)
              └────────┬────────┘
                       │
                       ▼
           ┌───────────────────────┐
           │   ContentManager      │
           │  (C++ interface)      │
           │ DefaultContentManager │
           └──────────┬────────────┘
                      │  item / NPC / policy queries
          ┌───────────┼────────────┐
          ▼           ▼            ▼
      Tactical    Strategic     Laptop/UI
      Subsystem   Subsystem     Subsystem
          │           │
          └─────┬─────┘
                ▼
          SGP (SDL2)
          Video / Audio / Input
                │
                ▼
          OS / Hardware
```

---

## 13. Key Design Patterns

### Global Content Manager (`GCM`)

A single global pointer (`GameInstance.h`) holds the active `ContentManager` instance. All game code queries content through this. While a global singleton, it is accessed exclusively through the typed interface, allowing the `ModPackContentManager` to be swapped in transparently at startup.

```cpp
// GameInstance.h
extern ContentManager* GCM;
```

### Screen Function Pointer Table

Screens are registered as plain function pointer structs, giving a lightweight state machine without virtual dispatch overhead.

### Externalized JSON Modding via JSON Patch (RFC 6902)

Mods provide patch files rather than full replacements. The Rust JSON layer merges patches before C++ ever sees the data, keeping mod files minimal and conflict-resistant.

### Unity Builds

`CMAKE_UNITY_BUILD` groups multiple `.cc` files into single translation units, significantly reducing compilation time for the large codebase.

### Rust ↔ C++ FFI

All Rust-to-C++ interaction goes through a C-compatible API (`stracciatella_c_api`). No C++ types cross the boundary; only primitive types and opaque pointers are exchanged.

---

## 14. Known Technical Debt

| Area | Issue | Risk |
|---|---|---|
| Memory management | 1,249 raw `new`/`delete` vs. 196 smart pointers | Memory leaks, use-after-free |
| God files | `EditorMercs.cc` (~102K lines), `NPC.cc` (~74K), `AIMain.cc` (~60K) | Unmaintainable, hard to test |
| Include coupling | `Soldier_Control.cc` includes 88 headers | Long build times, tight coupling |
| Suppressed warnings | MSVC 4018, 4244, 4996 disabled globally | Hidden type conversion bugs |
| Type limits | `GridNo` as `INT16`, `ProfileID` as `UINT8` | Max 255 NPCs, constrained map size |
| Unsafe Rust | 664 `unsafe` / `unwrap()` / `expect()` calls | FFI panics, undefined behaviour |
| Global state | `guiCurrentScreen`, `guiPendingScreen`, `GCM` | Hard to test in isolation |
| TODO markers | ~550 `TODO` / `FIXME` / `XXX` / `BUG` comments | Unresolved known issues |
| Test coverage | 31 test files for 42,000+ lines of game code | Regressions go undetected |

---

## 15. Platform Support Matrix

| Platform | Compiler(s) | Architecture | Notes |
|---|---|---|---|
| Windows | MSVC 2019+, MinGW-w64 | x86, x86-64 | MinGW needs `-Wa,-mbig-obj` |
| Linux | GCC, Clang | x86-64, ARM | Primary development platform |
| macOS | Clang (Xcode) | x86-64, Apple Silicon | Bundled SDL2 |
| Android | Clang (NDK) | ARM, ARM64 | Shared library, APK asset VFS |
| OpenBSD | Clang | x86-64 | Custom CMake toolchain |
| FreeBSD | Clang, GCC | x86-64 | — |

> **SDL 2.0.6 is explicitly blacklisted** — it causes segfaults on all platforms. Minimum required version is 2.0.5 (excluding 2.0.6).

---

*For build instructions see [COMPILATION.md](../COMPILATION.md).
For contribution guidelines see [CONTRIBUTING.md](../CONTRIBUTING.md).*
