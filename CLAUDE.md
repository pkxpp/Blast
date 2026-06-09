# CLAUDE.md

This file provides guidance to kscc (claude.ai/code) when working with code in this repository.

## Project Overview

NVIDIA Blast is a destruction library (version 1.1.10) with a three-layer API design:
1. **NvBlast** (low-level) — C-style stateless API. No globals, no allocations, no task spawning. Portable memory layout allows memcpy cloning and direct binary serialization.
2. **NvBlastTk** (toolkit) — C++ API with a global framework. Manages object lifecycle, generates worker objects for multithreaded damage processing, uses an event system for actor splitting/chunk fracturing and joint updates.
3. **NvBlastExt\*** (extensions) — Reference implementations for physics integration, authoring, serialization, etc.

Neither NvBlast nor NvBlastTk includes physics or graphics representations. Extensions (especially ExtPhysX) bridge this gap.

## Build System

### Prerequisites
- Windows: Visual Studio 2017+ (vc15) or Visual Studio 2022 (vc16/vc17). CMake 3.3+.
- Linux: GCC with C++14 support. CMake 3.3+.
- Dependencies are managed by **packman** (NVIDIA's package manager) and downloaded to `NVIDIA/packman-repo` at the drive root on first build.

### Project Generation

**Windows (VS2017):**
```
generate_projects_vc15win64.bat
```
Opens solution at `compiler/vc15win64-cmake/BlastAll.sln`.

**Windows (VS2022):**
```
generate_projects_vc16win64.bat
```
Generates to `compiler/vc17win64-cmake/`.

**Linux:**
```
./generate_projects_linux.sh
```
Generates makefiles in `compiler/linux64-CONFIG-gcc/` (CONFIG = debug or release).

### Build Configurations
Four configurations: `debug`, `checked` (release + asserts), `profile` (release + debug info), `release`.

### Output Directories
- Libraries: `lib/vc17win64-cmake/{debug,checked,profile,release}/`
- DLLs/EXEs: `bin/vc17win64-cmake/{debug,checked,profile,release}/`

### Sample Resources
Before running the sample, download asset files:
```
download_sample_resources.bat
```

## Architecture

### SDK Source Layout (`sdk/`)

| Directory | Library | Description |
|-----------|---------|-------------|
| `lowlevel/include/` | NvBlast | Public C API headers (`NvBlast.h`, `NvBlastTypes.h`) |
| `lowlevel/source/` | NvBlast | Core solver: Actor, Asset, Family, FamilyGraph |
| `toolkit/include/` | NvBlastTk | Public C++ API headers (`NvBlastTk*.h`) |
| `toolkit/source/` | NvBlastTk | Tk implementations: Framework, Actor, Asset, Family, Group, Joint |
| `common/` | (shared) | Containers (HashMap, HashSet, FixedArray, etc.), math, memory utilities |
| `globals/include/` | NvBlastGlobals | Global allocator and profiler callbacks |
| `extensions/physx/` | NvBlastExtPhysX | PhysX integration: PxActor/PxJoint management, impact damage, stress solver |
| `extensions/authoring/` | NvBlastExtAuthoring | Mesh splitting, fracture tools, cutout patterns |
| `extensions/serialization/` | NvBlastExtSerialization | Cap'n Proto-based serialization (Ll, Tk, Px layers) |
| `extensions/stress/` | NvBlastExtStress | Stress calculations without external physics lib |
| `extensions/shaders/` | NvBlastExtShaders | Sample damage shader functions |
| `extensions/assetutils/` | NvBlastExtAssetUtils | World bonds, asset merging, geometry transforms |
| `extensions/import/` | NvBlastExtImport | APEX Destructible Asset import |
| `extensions/exporter/` | NvBlastExtExporter | FBX/OBJ/JSON mesh and collision export |

### Key Concepts

- **Asset** (`NvBlastAsset`): Immutable destructible template. Defines chunk hierarchy and bonds.
- **Actor** (`NvBlastActor`): A runtime instance of part of an asset. Created by splitting.
- **Family** (`NvBlastFamily`): All actors derived from a single asset. Shared memory block.
- **Chunk**: Hierarchical piece of a destructible. Leaf chunks are the smallest fracture units. Support chunks provide structural coverage.
- **Bond**: Connection between two chunks. Broken by damage shaders.
- **Damage Shader**: User-supplied function that determines how damage affects chunks and bonds. Passed to `NvBlastActorDamage` / `TkActorDamage`.

### Dependency Chain
```
NvBlast (standalone)
NvBlastGlobals → NvBlast
NvBlastTk → NvBlast, NvBlastGlobals, PxShared, PhysX
NvBlastExtShaders → NvBlast
NvBlastExtStress → NvBlast
NvBlastExtPhysX → NvBlastTk, PhysX
NvBlastExtAuthoring → NvBlastTk, Boost, FBXSDK
NvBlastExtSerialization → NvBlastTk, Cap'n Proto
```

### CMake Structure
Each library is defined in `sdk/compiler/cmake/{LibraryName}.cmake` with platform-specific overrides in `sdk/compiler/cmake/{windows,linux}/{LibraryName}.cmake`. The platform CMakeLists (`sdk/compiler/cmake/{windows,linux}/CMakeLists.txt`) include all library cmake files.

## Tests

Test sources are in `test/src/`:
- `unit/` — Unit tests using Google Test (ActorTests, APITests, AssetTests, CoreTests, FamilyGraphTests, MultithreadingTests, SyncTests, TkTests, TkCompositeTests)
- `perf/` — Performance benchmarks
- `utils/` — Test utility helpers

Tests are built as part of the main solution (BlastTests project).

## Tools

- **AuthoringTool** (`tools/AuthoringTool/`) — Command-line mesh fracturing and Blast asset creation
- **ApexImporter** (`tools/ApexImporter/`) — APEX Destructible to Blast asset converter
- **LegacyConverter** (`tools/LegacyConverter/`) — Legacy format converter

## Coding Conventions

- C++14 standard (Linux: `-std=c++14`)
- No RTTI, no exceptions (`/GR-`, `-fno-rtti -fno-exceptions`)
- Windows: `/W4 /WX` (warnings as errors), fast math (`/fp:fast`)
- Linux: `-Wextra -Werror`, fast math (`-ffast-math`)
- Compile definitions: `NV_DEBUG=1` (debug), `NV_CHECKED=1` (checked), `NV_PROFILE=1` (profile)
- API prefix convention: `NvBlast` (low-level), `NvBlastTk` (toolkit), `NvBlastExt*` (extensions)
- Low-level API is C-style (`NVBLAST_API` exports); toolkit and extensions are C++
