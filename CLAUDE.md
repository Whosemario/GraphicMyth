# CLAUDE.md — GraphicMyth (Piccolo Engine)

## Project Overview

**Piccolo Engine** is a tiny C++17 game engine built for the GAMES104 course by BoomingTech. It features a Vulkan-based deferred rendering pipeline, Jolt physics integration, a custom C++ reflection system, Lua scripting, and a built-in ImGui editor.

- **License:** MIT
- **Version:** 0.0.8
- **Primary Language:** C++17
- **Build System:** CMake 3.19+

---

## Repository Structure

```
GraphicMyth/
├── CMakeLists.txt              # Root CMake (version, global options)
├── build_linux.sh              # Linux build script (supports debug/release, make/ninja)
├── build_macos.sh              # macOS Xcode build script
├── build_windows.bat           # Windows Visual Studio build script
├── .clang-format               # Code style rules (Google style, 120-col limit)
├── .clang-tidy                 # Static analysis configuration
├── cmake/                      # Reusable CMake utility modules
├── engine/
│   ├── 3rdparty/               # Vendored third-party libraries (git submodules)
│   ├── asset/                  # Game assets (JSON world/level/object defs, textures)
│   ├── configs/
│   │   ├── development/        # Dev config (PiccoloEditor.ini)
│   │   └── deployment/         # Deployment config
│   ├── shader/
│   │   ├── glsl/               # GLSL shader source (.vert, .frag, .comp, .geom)
│   │   ├── include/            # Shader headers
│   │   └── generated/cpp/      # SPIR-V compiled to C++ headers (auto-generated)
│   ├── source/
│   │   ├── editor/             # PiccoloEditor: UI, input, scene management
│   │   ├── runtime/            # Core engine runtime (subsystems listed below)
│   │   ├── meta_parser/        # Standalone tool: parses C++ for reflection metadata
│   │   ├── precompile/         # Drives meta_parser during build
│   │   └── test/               # Tests (currently commented out in CMakeLists)
│   └── template/               # Project templates
├── scripts/                    # Utility scripts
└── .github/workflows/          # GitHub Actions CI (linux/macos/windows)
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | C++17 |
| Build | CMake 3.19+, Ninja or Make |
| Graphics API | Vulkan |
| Windowing | GLFW |
| UI | Dear ImGui |
| Physics | Jolt Physics |
| GPU Memory | Vulkan Memory Allocator |
| Scripting | Lua 5.4.4 + Sol2 3.3.0 |
| Logging | spdlog |
| JSON | json11 |
| Image Loading | stb |
| Mesh Loading | tinyobjloader |
| Shaders | GLSL → SPIR-V via glslangValidator |

---

## Build Instructions

### Prerequisites

- CMake >= 3.19
- Vulkan SDK (required on all platforms)
- Platform-specific: Visual Studio 2019+ (Windows), Xcode 12.3+ (macOS), Clang (Linux)

### Linux

```bash
./build_linux.sh         # defaults to debug + make
./build_linux.sh release ninja
```

### macOS

```bash
./build_macos.sh
```

> macOS: only x86_64 is supported. Apple Silicon (M1/M2) is not yet fully supported.

### Windows

```bat
build_windows.bat
```

### Manual CMake

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --parallel
```

Binaries land in `bin/`. The main executable is `bin/PiccoloEditor`.

### Compilation Database (for LSP / clangd)

```bash
cmake -S . -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
```

---

## Architecture

### Startup Flow

```
main()  [editor/source/main.cpp]
  └─> PiccoloEngine::startEngine(config_path)
  └─> PiccoloEngine::initialize()
  └─> PiccoloEditor::initialize(engine)
  └─> PiccoloEditor::run()          ← main loop
  └─> editor->clear()
  └─> engine->shutdownEngine()
```

### Engine Layers

```
editor/         PiccoloEditor, EditorUI (ImGui), SceneManager, InputManager
   │
runtime/        All engine subsystems
   ├─ core/     Math, logging, reflection/meta, hash utilities
   ├─ function/ Render, Framework (ECS), Physics, Animation, Input,
   │            Character, Controller, Particle, UI, Global managers
   └─ resource/ AssetManager, ConfigManager, resource loaders
   │
meta_parser/    Standalone C++ tool — reads annotated headers, emits reflection metadata
precompile/     Drives meta_parser as a CMake precompile step
```

### Key Classes

| Class | Location | Purpose |
|---|---|---|
| `PiccoloEngine` | `runtime/engine.h` | Engine lifecycle, game loop, tick/render dispatch |
| `PiccoloEditor` | `editor/editor.h` | Editor wrapper, ImGui orchestration |
| `RenderSystem` | `runtime/function/render/render_system.h` | Vulkan deferred pipeline |
| `GObject` | `runtime/function/framework/object/object.h` | Game object (entity) |
| `Component` | `runtime/function/framework/component/` | Base class for all components |
| `Level` | `runtime/function/framework/level/level.h` | Scene container |
| `World` | `runtime/function/framework/world/world.h` | Level manager |
| `AssetManager` | `runtime/resource/asset_manager/` | Asset loading & caching |
| `ConfigManager` | `runtime/resource/config_manager/` | App config (`.ini` files) |

### Component System

All game objects (`GObject`) are assembled from components:

| Component | Purpose |
|---|---|
| `TransformComponent` | Position, rotation, scale (double-buffered for frame safety) |
| `MeshComponent` | Static mesh rendering |
| `CameraComponent` | View/projection matrices |
| `RigidbodyComponent` | Jolt physics body |
| `AnimationComponent` | Skeletal animation playback |
| `LuaComponent` | Lua scripting attachment |
| `MotorComponent` | Character movement logic |
| `ParticleComponent` | GPU particle effects |

### Rendering Pipeline

Deferred rendering implemented across multiple passes:

1. **G-Buffer Pass** — geometry, normals, albedo, material properties
2. **Shadow Map Pass** — directional and point light shadows
3. **Deferred Lighting Pass** — PBR lighting evaluation
4. **Skybox Pass** — environment cube map
5. **Particle Pass** — GPU-simulated particle systems
6. **FXAA Pass** — temporal anti-aliasing post-process
7. **Tone Mapping Pass** — HDR → LDR with LUT color grading

Vulkan resources are abstracted behind an RHI layer (`runtime/function/render/interface/`).

---

## Reflection System

Piccolo uses a **custom C++ reflection system** driven by a standalone parser tool.

### Annotation Macros (add to headers)

```cpp
REFLECTION_TYPE(ClassName)
CLASS(ClassName, Fields)       // or STRUCT(...)
{
    REFLECTION_BODY(ClassName)
public:
    META(Enable)
    float m_field;             // reflected field
};
```

### How it Works

1. `meta_parser/PiccoloParser` is built first as a CMake target.
2. During `precompile`, it scans annotated headers and generates `*.gen.h` files containing reflection metadata.
3. Generated files allow JSON serialization, editor UI introspection, and runtime field access.

### Serialization

Use `Serializer` from `runtime/core/meta/serializer/` — reads/writes JSON via the reflection metadata automatically.

---

## Asset & Data Formats

All game data is JSON:

| Extension | Purpose | Example |
|---|---|---|
| `.world.json` | World scene (lists levels) | `asset/world/hello.world.json` |
| `.level.json` | Level layout (objects) | `asset/level/1-1.level.json` |
| `.object.json` | Reusable object template | `asset/objects/character.object.json` |
| `.material.json` | Material properties | PBR material definition |
| `.skeleton.json` | Skeleton hierarchy | |
| `.animation_clip.json` | Animation data | |
| `.global.json` | Global rendering/particle config | `asset/global/rendering.global.json` |

### Config Files

- `engine/configs/development/PiccoloEditor.ini` — editor startup config (asset paths, default world, UI icons)
- `engine/asset/global/rendering.global.json` — skybox, IBL, camera, lights, FXAA, LUT

---

## Code Style & Conventions

### Enforced by `.clang-format` (Google style, customized)

- **Indentation:** 4 spaces (no tabs)
- **Line length:** 120 characters max
- **Braces:** Attach for most constructs, new line for functions/classes
- **Alignment:** Aligned assignments and consecutive declarations

Always run clang-format before committing:
```bash
clang-format -i <file>
# Or via cmake build target if available
```

### Naming Conventions

```cpp
class PiccoloEngine {};           // PascalCase for classes/structs
enum class EDirection {};         // E prefix for enums

void myFunction();                // camelCase for functions/methods

float m_speed;                    // m_ prefix for member variables
static float s_instance_count;   // s_ prefix for static members
float local_var;                  // snake_case for locals

namespace Piccolo {}              // All code lives in Piccolo namespace
```

### Header Guards

Use `#pragma once` (not traditional include guards).

### Memory & Resource Management

- Use RAII — prefer smart pointers (`std::shared_ptr`, `std::unique_ptr`) over raw `new`/`delete`
- Vulkan resources managed via `VulkanMemoryAllocator`
- Asset handles use the `AssetManager` for caching

---

## Testing

Tests exist under `engine/source/test/` but are **currently disabled** (the `add_subdirectory(source/test)` line in CMakeLists is commented out). There is no active test suite running in CI.

To re-enable, uncomment in `engine/CMakeLists.txt`:
```cmake
# add_subdirectory(source/test)
```

---

## CI/CD

### GitHub Actions (`.github/workflows/`)

Three workflows run on push/PR:

| Workflow | Runner | Builds |
|---|---|---|
| `build_linux.yml` | ubuntu-latest | Debug + Release |
| `build_windows.yml` | windows-latest | Debug + Release |
| `build_macos.yml` | macos-latest | Debug + Release |

### GitLab CI (`.gitlab/.gitlab-ci.yml`)

Triggers only on MR to `main` or `upstream`. Runs macOS, Ubuntu (Docker), and Windows builds.

---

## Shader Development

Shaders live in `engine/shader/glsl/`. They are GLSL and compiled to SPIR-V at build time by `glslangValidator` (from the Vulkan SDK). Compiled bytecode is embedded as C++ headers in `engine/shader/generated/cpp/`.

**Naming conventions:**
- Vertex shaders: `<name>.vert`
- Fragment shaders: `<name>.frag`
- Compute shaders: `<name>.comp`
- Geometry shaders: `<name>.geom`

After editing a shader, a full rebuild is needed for the SPIR-V to regenerate.

---

## Common Development Tasks

### Adding a New Component

1. Create `MyComponent.h` / `MyComponent.cpp` under `runtime/function/framework/component/my_component/`
2. Annotate with reflection macros (`REFLECTION_TYPE`, `CLASS`, `META`)
3. Inherit from `Component`
4. Register any required tick/init behavior
5. Add to the component CMakeLists target sources
6. Rebuild — the meta_parser will generate reflection metadata automatically

### Adding a New Render Pass

1. Create a new RenderPass subclass under `runtime/function/render/passes/`
2. Implement `initialize()`, `draw()`, and resource setup
3. Insert the pass into `RenderPipeline` in the correct order
4. Add required descriptors, attachments, and pipeline state

### Adding a Lua Script

1. Attach a `LuaComponent` to a `GObject`
2. Set the script path in the component's JSON definition
3. Implement `on_tick`, `on_start`, etc. in Lua
4. Access engine APIs via the Sol2 bindings

---

## Important Notes for AI Assistants

1. **Never build in the source directory** — CMake enforces out-of-source builds and will fail if attempted in-source.

2. **Reflection metadata is auto-generated** — do not manually edit `*.gen.h` files. They are produced by `PiccoloParser` during precompilation.

3. **SPIR-V shaders are auto-generated** — do not manually edit files under `engine/shader/generated/`. Edit the `.glsl` sources instead.

4. **All engine code lives in the `Piccolo` namespace** — always include `namespace Piccolo { }` wrappers.

5. **JSON files drive game content** — world, level, object, and material definitions are JSON. Always keep JSON consistent with the corresponding C++ reflection schema.

6. **Physics debug renderer is Windows-only** — `ENABLE_PHYSICS_DEBUG_RENDERER` is gated to `WIN32` in CMake.

7. **Apple Silicon (M1/M2) is not yet supported** — macOS builds target x86_64 only.

8. **Tests are disabled** — do not assume tests pass or fail; the test infrastructure exists but is not actively maintained.

9. **The `3rdparty/` directory is git submodules** — when cloning, use `git clone --recursive` or run `git submodule update --init --recursive`.

10. **clang-format compliance is required** — all C++ changes should be formatted with the provided `.clang-format` config before committing.
