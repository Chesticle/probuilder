# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Your role here: By following sections of provided guides create or edit assets and if required place it into the world.

You have live access to this project so you can understand and edit it. Explain what you 
change and why. Always double check your work and if you worked in c++ tell me if I need to do a full restart and compile or if I can do a live code compile.

- Do not take theg guide as gospel. Think for yourself as a senior developer implamenting a guide made by a co-worker at your same level feel free to itterate if you spot issues. and if something doesnt make sense or creates logic errors fix them yourself. If you need to delete or replace blueprints or code that is not specified inside the guides make sure to document those changes within a .md file so the guide can later be updated. 
- Do NOT push changes to version control on your own.
- The project follows the priciples of fragmented design patterns with the goal of making all features interchangable and removable without breaking by minimizing dependacies.
  Additionally this project focuses heavily on exposed settings and dials that allow for designers to change as much as possible without going to blueprints or scripts.

## Project facts

- Engine: Unreal Engine 5.8 (upgraded from 5.6)
- Platform: Windows 11
- Project type: Blueprint + C++
- Conventions: <Bool variables begin with “modal” verbs instead of b for example canDie/isDead instead of bDie/bDead. In guide files you may see Color with text. That is for me to make comments later you can safly ignore that same with anything to do with tooltips.



## Project overview

Rubicon3 is an Unreal Engine **5.8** third/first-person movement shooter. Gameplay (movement, combat, AI, gore/dismemberment, weapons) is implemented almost entirely in **Blueprints** under `Content/Rubicon/`; the C++ side is currently just the primary game module bootstrap with no gameplay classes yet. Expect most tasks in this repo to involve reading/reasoning about Blueprint asset organization, `.ini` config, and (when present) small amounts of C++ module code, rather than large C++ codebases.

This is not a git repository — version control is **Diversion** (Unreal-oriented VCS), configured via the `Diversion` plugin and `.dvignore` (a superset of common Unity/Godot/Unreal ignore rules). Do not assume `git` commands work here.

## Build

There are no npm/pytest-style CLI build scripts. This is a standard Unreal Engine project built via UnrealBuildTool (UBT), through one of:

- **Visual Studio**: open `Rubicon3.sln` (generated; `Rubicon3.slnx` is the newer slnx-format equivalent) and build the `Development Editor` configuration for `Win64`. This is the normal way to compile C++ changes.
- **Regenerate project files**: if `.sln`/`.slnx` are stale after adding/removing source files, right-click `Rubicon3.uproject` → "Generate Visual Studio project files" (requires Epic-Games-Launcher-installed UE 5.8, since `EngineAssociation` in `Rubicon3.uproject` is `"5.8"` and there is no in-repo Engine source).
- **Editor Live Coding**: for iterating on the `Rubicon3` module while the editor is open, Live Coding (Ctrl+Alt+F11 in-editor, or via VS "Compile") is faster than a full rebuild.

There are two build targets, both pulling in the single `Rubicon3` module:
- `Rubicon3Target.cs` — `Game` target
- `Rubicon3EditorTarget.cs` — `Editor` target

**Note the module lives at `Source/Testing/`, not `Source/Rubicon3/`.** The project was renamed from the `TP_Blank` template (originally called "Testing") to Rubicon3; the module class/target names were updated but the source folder was not. Don't "fix" this by moving files unless explicitly asked — it's a known quirk, not a bug to clean up.

`Source/Testing/Rubicon3.Build.cs` currently declares dependencies on `GameplayAbilities`, `GameplayTasks`, and `GameplayTags` (private) alongside `Core`/`CoreUObject`/`Engine`/`InputCore`/`EnhancedInput` (public), but no GAS C++ classes exist yet — the gameplay/combat system so far is built as Blueprint actor components (see Architecture below), not GAS Ability/Effect classes.

## Tests

No test suite exists in this repo yet (`Source/Testing` is a leftover folder name, not a test target — see above).

## Editor automation via MCP

`.mcp.json` wires up an `unreal-mcp` MCP server at `http://127.0.0.1:8000/mcp`. This exposes live Unreal Editor control (`mcp__unreal-mcp__*` tools) when the editor is running with the corresponding plugin listening — use it for tasks that require inspecting/mutating the live editor state (e.g. querying or editing Blueprints, actors, or the open level) rather than trying to parse `.uasset` binary files directly.

## Architecture

### Content layout (`Content/Rubicon/`)
The game-specific content tree is organized by discipline, not by feature:
- `Blueprints/` — gameplay logic, split into `Character/`, `Enemies/`, `AI/`, `Weapons/`, `GoreSystem/`, `Interfaces/`, `Chaos/`, `AnimNotifiers/`
- `GlobalFragments/` — shared actor components attached across characters (see Fragment pattern below)
- `Animations/`, `Meshes/`, `Materials/`, `Textures/`, `Particles/` — per-asset-type art content, further split by character/system name
- `Maps/` — levels, including `Maps/_GENERATED/<developer>/` per-developer scratch levels
- `UI/` — CommonUI-based UI (see `DefaultGame.ini`, `CommonButtonAcceptKeyHandling` is configured)

Other top-level `Content/` folders (`CozyNature`, `DemoRoomAssets`, `Fab/Megascans`, `MSPresets`, `OpenVAT_Unreal`, `Stylish_Fire_VFX`, `UsdAssets`) are third-party/Marketplace/Fab asset packs — treat as vendored, not project-authored. `Content/Developers/<name>/` is per-developer personal sandbox content (standard UE convention).

### Fragment-based actor composition
Instead of using the GAS `GameplayAbilities` dependency directly, gameplay behavior is built from Blueprint **ActorComponent "fragments"** (naming convention `AC_*Fragment`), e.g. `AC_CombatFragment`, `AC_HealthFragment`, `AC_PlayerHealthFragment`, `AC_DismembermentFragment`, `AC_ProjectileFragment` in `Content/Rubicon/GlobalFragments/`. When adding new gameplay behavior, prefer following this fragment/component pattern over introducing GAS Abilities/Effects unless asked otherwise.

### Character movement & input
The player uses **Enhanced Input** (`Content/Rubicon/Blueprints/Character/`): `IMC_CMovement` mapping context with `IA_C*` actions for Aim, Camera, Dash, Grapple, GroundPound, Holster, Jump, Movement, Reload, Shoot, Slide, SwitchGrapple, WallRun. `GM_CMovementShooter` is the game mode; `BP_CPlayerController` the player controller; `E_GrappleState` an enum driving the grapple hook state machine.

### AI
Enemy AI uses the **GameplayStateTree** plugin: `Content/Rubicon/Blueprints/AI/` holds `ST_MeleeEnemy` (a StateTree asset), `BP_EnemyAIController`, plus supporting `AIFragments/`, `Evaluators/`, `ST_Tasks/`, and `Notifies/` subfolders. `Content/Rubicon/Blueprints/Enemies/` holds the enemy character Blueprints (`BP_DarksolCharacter`, `BP_RangedTowerEnemy`) and `VATs/` (vertex-animation-texture variants, paired with the `AnimToTexture` plugin, for crowds/perf).

### Gore / dismemberment system
Data-driven, under `Content/Rubicon/Blueprints/GoreSystem/`: `DA_DismembermentProfile` and `DA_ChunkLayout` data assets (with per-enemy instances like `ChunkLayout_Grunt_RangedTowerEnemy`) describe how a character breaks apart; `E_BodyRegion`, `E_OverflowMode(Override)` enums and `Structures/` define the schema; `BP_ChunkLayoutGen` and `BP_SeveredPart` do the runtime spawning/simulation. Driven at runtime by `AC_DismembermentFragment`.

### Weapons & damage
`BPI_Weapon` and `BPI_Damageable` are Blueprint Interfaces defining the weapon and damageable-actor contracts respectively. `BP_WeaponsMaster` is the base weapon Blueprint; `BP_RevolverLoaded` an implementation; `FS_MasterField_Bullet` a struct describing projectile/bullet data, tied to `AC_ProjectileFragment`.

### Config
Standard `Config/Default*.ini` split (`Engine`, `Game`, `Editor`, `Input`, `Niagara`). Notable non-default entries live in `DefaultGame.ini`: a custom `Prismatiscape` plugin setting (`PrismatiscapeManagerClass`) and `AssetManagerSettings` primary asset type scanning for `Map` and `GameFeatureData`.

### Notable plugins (`Rubicon3.uproject`)
- `GameplayStateTree` — enemy AI (see above)
- `AnimToTexture` — VAT for enemy crowds
- `GeometryScripting`, `ModelingToolsEditorMode` — procedural/editor mesh tooling
- `Diversion` — version control (this repo's VCS; not git)
- `ModelContextProtocol`, `AllToolsets`, `Terminal` — backing the `unreal-mcp` MCP server integration
- `AutoSizeComments`, `BlueprintAssist`, `LogViewerPro` — editor QoL, no runtime impact

## Active work: gore system — roadmap and guides

I am partway through building a gore system, following the guides below. There are other guides however the bolow are the ones that are needed for what im currently working on. Parts 1-7 of the roadmap have been completed so far, plus the accuracy work in `03b` (ray-marched chunk resolution) and `03c` (overlap projectile path).

- @docs/GoreSystem/UE5_Dismemberment_00_Roadmap.md
- @docs/GoreSystem/UE5_ActiveRagdoll_00_Roadmap.md
- @docs/GoreSystem/UE5_ActiveRagdoll_01_CppFoundations_ShapeSpike.md
- @docs/GoreSystem/UE5_Dismemberment_Tool_BlenderAdjacencyExport.md
- @docs/GoreSystem/UE5_Dismemberment_Tool_SeamBasedAdjacency.md
- @docs/GoreSystem/UE5_Dismemberment_02_ChunkRegistry.md
- @docs/GoreSystem/UE5_Dismemberment_02_ChunkLayoutAutogenTool.md
- @docs/GoreSystem/UE5_ActiveRagdoll_02_ShapePipeline_LiveRefit.md
- @docs/GoreSystem/UE5_Dismemberment_Tool_ChunkLayoutGenerator.md
- @docs/GoreSystem/tools/chunk_bounds.csv
- @docs/GoreSystem/tools/chunk_bounds_export.py
- @docs/GoreSystem/tools/chunk_bounds_paste.txt