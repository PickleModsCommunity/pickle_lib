# TEBBY.md - pickle_lib

## Overview

`pickle_lib` is a small, shared Lua utility library written for the **Helix** game framework (an Unreal Engine multiplayer framework, not FiveM). It wraps common low-level operations — interval/thread management, world object creation and attachment, shutdown cleanup — into a clean, documented `Lib` table so that other Pickle Modifications scripts can call them without repeating boilerplate. There is no server or client split; the single source file `modules/shared.lua` is loaded as a shared module.

## Code design

The entire library lives in one file (`modules/shared.lua`). Its architecture is:

1. **Global helpers overridden at startup** — `print` and `error` are replaced with versions that prefix `[INFO]`/`[ERROR]`, the current context (`CLIENT` or `SERVER`), and the caller's source location. A `printf` global is added for named-placeholder string formatting.
2. **`Lib` table** — the public API surface. All methods are static (no instances). Internal state is kept in three module-local tables:
   - `Intervals` — tracks live timer IDs so they can all be cancelled on shutdown.
   - `ShutdownFunctions` — ordered list of callbacks to run when `Lib.startShutdown()` is called.
   - `Objects` — tracks `StaticMesh` objects so they can all be deleted on shutdown.
3. **`onShutdown()` global** — a bare global function (not on `Lib`) that the host framework is expected to call. It calls `Lib.startShutdown()`, clears every tracked interval, and destroys every tracked object. This is the cleanup entry point for the whole library.
4. **Delegation pattern** — `Lib.CreateInterval`/`Wait`/`CreateThread` are thin wrappers around the engine-provided `Timer.*` globals. The library does not reimplement timing; it adds tracking and cleanup on top.
5. **Manifest** — `project.json` declares which glob patterns constitute the shared, server, and client module lists. Currently only the shared list is populated (`modules/shared.lua`). `package.json` mirrors the same declaration and is the file the Helix package manager reads.

The code follows a single-file, no-dependency pattern. All engine types (`StaticMesh`, `Vector`, `Rotator`, `CollisionType`, `Timer`, `UE`, etc.) are expected to be provided by the Helix runtime globals — they are not imported.

## Important files

| Path | Why it matters |
|------|----------------|
| `modules/shared.lua` | The entire library source. Contains all `Lib` functions, the overridden `print`/`error` globals, `printf`, and the `onShutdown` shutdown hook. |
| `project.json` | Helix module manifest. Maps `shared`/`server`/`client` roles to file globs. Only the `shared` list is populated; used by the Helix loader to know what to load. |
| `package.json` | Package manager manifest (mirrors `project.json`). Consumed by the Helix package tooling; also lists only the shared module. |
| `README.md` | Brief human-readable description of the library's purpose. |
| `docs.json` | Auto-generated or hand-authored API documentation in JSON. Lists every function signature, parameter type, and description for `Lib` and the global helpers. Used by documentation tooling, not at runtime. |

## Intended behaviors

- **`print` and `error` are intentionally overridden as globals** (`---@diagnostic disable: lowercase-global` is at the top of the file). This is not a mistake or a naming collision — the library deliberately shadows the Lua builtins to inject context tags and caller location into every log line. A fix agent must not "restore" the originals.
- **`onShutdown()` is a bare global, not `Lib.onShutdown`**. The framework calls this bare global on resource stop. `Lib.onShutdown` (on the table) registers a _callback_; the bare `onShutdown()` is the _trigger_. They are intentionally different things and must not be merged.
- **Tracked collections (`Intervals`, `Objects`) are module-local** — they are not exposed on `Lib`. External code cannot manipulate them directly. This is intentional encapsulation.
- **Known bug in `Lib.AttachEntityToCharacter`**: the function accepts an `offset` parameter but uses `coords` (undefined in scope) when calling `K2_SetActorRelativeLocation`. This is a real bug, not an intentional design. The fix is to replace `coords` with `offset` in that call.

## Common issues & fixes

| Issue | Likely cause | Fix |
|-------|-------------|-----|
| `attempt to index a nil value (global 'coords')` inside `Lib.AttachEntityToCharacter` | The function body references `coords` instead of the `offset` parameter passed in. | Replace `coords.X`, `coords.Y`, `coords.Z` with `offset.X`, `offset.Y`, `offset.Z` in `modules/shared.lua` (the `K2_SetActorRelativeLocation` call). |
| Intervals or objects not cleaned up on resource restart | Caller forgot to let the framework invoke `onShutdown()`, or replaced it. | Ensure the host framework is wired to call the global `onShutdown()` when the resource stops. Do not rename it. |
| `Timer`, `StaticMesh`, `Vector`, etc. are nil | Library loaded outside the Helix runtime, or loaded before the engine globals are available. | Ensure `pickle_lib` is started after the Helix runtime initialises its globals. |
| `printf` placeholders not substituting | Keys in the `args` table don't match `{key}` tokens in the format string (case-sensitive). | Align key names exactly; `printf` uses `{key}` syntax, not `%s`. |
| Locale/text issues (from the Pickle ecosystem pattern) | Resource folder name differs from the name the host script expects. | Ensure the package/folder is named `pickle_lib` exactly, matching whatever `require` or `import` reference uses. |

## Conventions

- **Lua style**: all functions use `local` variables internally; public API is attached to the `Lib` table. No global state is exported except the intentional overrides (`print`, `error`, `printf`, `Lib`, `onShutdown`).
- **EmmyLua annotations**: every public function has `---@param`, `---@return`, and `---@class` annotations for IDE support. New functions added to `Lib` must include matching annotations.
- **`---@diagnostic disable`** directives are used at the file top for lowercase-global warnings. Do not remove them; the overridden globals are intentional.
- **Module file naming**: shared modules go in `modules/**/shared.lua`, server-only in `modules/**/server.lua`, client-only in `modules/**/client.lua`, matching the globs in `project.json`/`package.json`.
- **No external dependencies**: the library must remain self-contained. All types it uses are Helix/UE runtime globals. Do not add `require` calls or external imports.
- **Shutdown safety**: any new function that allocates a long-lived resource (timer, world object) must register it in the appropriate tracking table (`Intervals` or `Objects`) so it is cleaned up by `onShutdown`.
