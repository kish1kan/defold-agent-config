# Agent Instructions

## Scope and priorities

This file applies to the entire repository. A nested `AGENTS.md`, if added later, may define stricter rules for its directory.

- `MUST` and `NEVER` mark correctness, safety, or architecture requirements.
- `SHOULD` marks the default preference; deviate only when the task requires it and explain why.
- Project-specific rules in this file take priority over generic Defold conventions.

## Project facts

This is a **Defold** game project. Run project commands from the repository root containing `game.project`.

- Bootstrap collection: `/main/main.collection`
- Main content: `main/`
- Screens: `screens/<screen_name>/`
- Popups: `popups/<popup_name>/`
- Reusable Druid widgets: `widgets/<widget_name>/`
- Assets: `assets/`
- Game modules: `modules/`
- Read-only dependency context: `.deps/`
- Declared libraries: Monarch, object interpolation, and Sharp Sprite

Values such as `main_collection`, `game_binding`, and `app_icon` in `game.project` are Defold resource identifiers. A trailing `c` denotes a compiled resource and is expected.

## Hard constraints

- NEVER edit files under `.deps/`. Use them only to resolve modules and inspect dependency APIs.
- If `.deps/` is missing or empty, or a dependency URL in `game.project` changes, load and run the `defold-project-setup` skill before other project work.
- Keep Defold resource paths absolute where the engine expects them, for example `/assets/...`.
- For new or modified Sprite, Spine, GUI, ParticleFX, Tilemap, Font, and Label resources, use RGSS materials from `/sharp_sprite/rgss/` instead of built-in materials. Existing legacy references in `popups/popup_example/popup_example.gui` and `assets/fonts/Unbounded-ExtraBold-50.font` may remain until those resources are touched; do not perform an unrelated repository-wide migration.
- GUI scripts MUST NOT import or read domain/game-state modules directly. Business data must arrive through messages. UI and navigation libraries such as Druid and Monarch are allowed.
- Store per-instance state in `self` in `.script`, `.gui_script`, and `.render_script` files. Module-level locals are shared across instances and are only for constants, functions, and stateless dependency handles returned by `require`.
- Pair acquired resources with their lifecycle cleanup: release input focus in `final()`, unregister persistent listeners such as `window.set_listener(nil)`, and cancel timers when their intended lifetime is shorter than the script lifetime. Defold destroys a script's timers with the script, so redundant deletion-time cancellation is not required.

## Task-to-skill routing

Load only the skills relevant to the files being changed, in this order when rules overlap:

1. Use `defold-project-setup` first when dependency setup is required.
2. For a new screen, popup, or navigation flow, load `monarch-screen-setup`.
3. For HUDs, menus, forms, popups, Druid widgets, or `.gui`/`.gui_script` UI, load `druid-ui-setup`.
4. Before editing Defold protobuf text resources such as `.collection`, `.go`, or `.gui`, load `defold-proto-file-editing`.
5. Before editing Lua, `.script`, or `.gui_script`, load `defold-scripts-editing`.
6. Before editing shaders or native extensions, load the corresponding `defold-shaders-editing` or `defold-native-extension-editing` skill.
7. For performance-critical vector, quaternion, or matrix code, load `xmath-usage`.

Use only documented Defold APIs. Load `defold-api-fetch` when adding, changing, or uncertain about an engine API; use `defold-docs-fetch` for concepts and `defold-examples-fetch` for implementation patterns. Existing, unchanged API calls do not need to be re-researched.

## Code conventions

- Use tabs for Lua indentation. Keep empty lines empty and remove trailing whitespace.
- Use `snake_case` for variables, functions, files, and folders; use `UPPER_CASE` for module-level constants.
- Use LuaCATS (`---@...`) for module/public API documentation and non-obvious data shapes.
- Call `require` with parentheses and a root-relative dotted module name: `require("modules.best_time")`. Do not use slashes or a leading slash.
- Define named local functions at module scope, not inside other functions. Inline anonymous callbacks are allowed.
- Prefer functional, data-oriented modules. Do not use metatables to imitate classes.
- Do not repeatedly check internal fields established by the same code path. Validate data at trust boundaries such as messages, saved data, network responses, and user input.
- Inline a hash used once. Reused hash values may be module-level `UPPER_CASE` constants.
- Log initialization, state transitions, persistence operations, and errors when useful. Do not add per-frame or noisy polling logs.

## Validation

- Lua or Defold resource changes MUST build successfully through the running editor using `defold-project-build`.
- UI, input, navigation, or lifecycle changes SHOULD receive a focused smoke test of the affected flow in addition to a successful build.
- Dependency changes MUST be followed by `defold-project-setup` before building.
- If the editor or another required validator is unavailable, report the validation as not run; do not claim success.
- Git commit messages MUST be short descriptions in English.
