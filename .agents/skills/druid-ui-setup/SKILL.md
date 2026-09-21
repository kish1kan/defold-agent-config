---
name: druid-ui-setup
description: "Creates and edits Defold game UI with the Druid component framework (buttons, lists, HUD, popups, inputs, widgets). Use when building or modifying GUI screens, .gui / .gui_script files, Druid widgets, HUD, menus, scroll lists, progress bars, sliders, or any Insality/druid UI."
---

# Creating Game UI with Druid

Druid attaches **logic** (button, scroll, list, input, …) to GUI nodes that already exist in a `.gui` scene. Place nodes in the GUI file, then bind Druid components in Lua.

Official docs: [Insality/druid](https://github.com/Insality/druid). Live examples: https://insality.github.io/druid/

## Prerequisite: Druid + Defold Event

Before applying this skill, confirm both libraries are present.

1. Check `game.project` for `Insality/druid` **and** `Insality/defold-event`.
2. Alternatively, check `.deps/` for `druid/` and `event/`.

If either is missing, **add both** under `[project]` (Druid 1.3.x requires Defold Event **16**):

```
[project]
dependencies#N = https://github.com/Insality/defold-event/archive/refs/tags/16.zip
dependencies#N = https://github.com/Insality/druid/archive/refs/tags/1.3.1.zip
```

Then run the `defold-project-setup` skill so `.deps/` is updated, and tell the user: **In the Defold editor, go to Project → Fetch Libraries to sync.**

This project already uses `/builtins/input/all.input_binding`, which Druid expects. Do not switch away from it without reading `references/setup.md`.

## Compose with other skills

| Task | Skill |
|---|---|
| `.gui` node trees, templates, fonts, materials | `defold-proto-file-editing` (`references/gui.md`) |
| `.gui_script` / `.lua` file rules | `defold-scripts-editing` |
| Screen / popup collections and navigation | `monarch-screen-setup` |
| Druid Lua API details not in this skill | Inspect `.deps/druid/` or fetch https://raw.githubusercontent.com/Insality/druid/master/api/quick_api_reference.md |

This skill covers **Druid usage**. It does not replace the proto/script skills: load those before writing `.gui` or Lua files.

## Directory layout

```
project/
├── screens/<screen_name>/
│   ├── <screen_name>.collection
│   ├── <screen_name>.gui              -- scene nodes + templates
│   └── <screen_name>.gui_script       -- druid.new(self) + widgets
├── popups/<popup_name>/
│   ├── <popup_name>.collection
│   ├── <popup_name>.gui
│   └── <popup_name>.gui_script
└── widgets/<widget_name>/             -- reusable Druid widgets
    ├── <widget_name>.gui              -- MUST have a `root` node
    └── <widget_name>.lua              -- widget module (not a gui_script)
```

- **Screens / popups** own a `.gui_script` that creates a Druid instance and composes widgets.
- **Widgets** are a `.gui` + a `.lua` module next to it. They have **no** `.gui_script` (template scripts never run).
- Widget names use `snake_case` and must match the GUI template node id when instantiated.

## Always-on gui_script skeleton

Every GUI that uses Druid **must** forward these callbacks. Omit none of them: Scroll, Progress, Drag, Timer, and Layout need `update`; input components need `on_input`.

```lua
local druid = require("druid.druid")

function init(self)
    self.druid = druid.new(self)
    -- create components / widgets here
end

function final(self)
    self.druid:final()
end

function update(self, dt)
    self.druid:update(dt)
end

function on_message(self, message_id, message, sender)
    self.druid:on_message(message_id, message, sender)
    -- handle game -> UI messages here
end

function on_input(self, action_id, action)
    return self.druid:on_input(action_id, action)
end
```

**Required:** `return self.druid:on_input(...)` so consumed input does not leak to other GUIs.

Druid calls `acquire_input_focus` automatically when an input component exists. Do **not** also `msg.post(".", "acquire_input_focus")` unless `[druid] no_auto_input = 1`.

Pass **node id strings** to constructors (`"play"`, not `gui.get_node("play")`) unless you already hold a node.

## Widget-first workflow

Prefer widgets for any reusable piece (buttons with labels, health bars, shop rows, windows). A widget is a Lua table with colon methods — this is Druid's API, not a custom class/metatable system.

```lua
---@class health_bar: druid.widget
local M = {}

function M:init()
    self.root = self:get_node("root")
    self.progress = self.druid:new_progress("fill", "x", 1)
    self.label = self.druid:new_text("label")
end

function M:set_health(current, max)
    self.progress:set_value(current / max)
    self.label:set_text(tostring(current) .. "/" .. tostring(max))
end

return M
```

Use `self:get_node("id")` inside widgets (never `gui.get_node("id")` — that ignores the template prefix).

Instantiate from the parent gui_script:

```lua
local health_bar = require("widgets.health_bar.health_bar")

self.health_bar = self.druid:new_widget(health_bar, "health_bar")
self.health_bar:set_health(80, 100)
```

The second argument is the **template node id** in the parent `.gui`. Extra varargs after template/nodes go to `M:init(...)`.

Full widget rules, cloning, events, and GO widgets: `references/widgets.md`.

## GUI nodes vs Druid components

1. Build the node tree in the `.gui` file (`defold-proto-file-editing`).
2. Bind Druid components to those node ids in `init`.
3. Keep a disabled **prefab** node in the scene for lists; clone it at runtime.

Common node trees (scroll, list, progress, input): `references/gui-layouts.md`.

Component constructors, methods, and events: `references/components.md`.

If this project has `defold-sharp-sprite`, GUI material is `/sharp_sprite/rgss/materials/gui.material`, not builtins.

## Input stack

Druid checks **last created first**. Create background / lower layers first, then foreground, then blockers/overlays. A later `new_button` steals clicks from an earlier one under the same point.

When removing a component or deleting its nodes, call `self.druid:remove(component)` first.

## Game state stays out of GUI Lua

GUI scripts and widgets may `require("druid.druid")`, `require("event.event")`, and `require("monarch.monarch")`. They must **not** require gameplay/business-logic modules.

- **UI → game:** `msg.post(url, "action_name", data)` from button/widget callbacks.
- **Game → UI:** `on_message` updates widgets (`self.health_bar:set_health(...)`).
- Keep the GUI data-driven: it displays what messages tell it to display.

Button callbacks receive `(self, params, button_instance)` where `self` is the `druid.new(self)` context (the gui_script `self`, or the widget table when created inside a widget).

## Events

All component callbacks are [Defold Event](https://github.com/Insality/defold-event) objects. Subscribe extra listeners without wrapping constructors:

```lua
self.button.on_click:subscribe(function(self, params, button)
    msg.post(".", "play_pressed")
end)

self.scroll.on_scroll:subscribe(function(self, position)
end)
```

Widget-owned events:

```lua
local event = require("event.event")

function M:init()
    self.on_close = event.create()
    self.druid:new_button("close", function()
        self.on_close:trigger()
    end)
end
```

## Styles, sounds, localization, input remaps

See `references/setup.md` for `druid.set_default_style`, `set_sound_function`, `set_text_function`, `[druid]` game.project keys, and whitelist/blacklist (scoped in 1.3+).

## Workflow: new screen or popup UI

1. If it is a Monarch screen/popup, follow `monarch-screen-setup` for collections and registration.
2. Confirm Druid + Event dependencies (section above).
3. Create/edit the `.gui` with `defold-proto-file-editing`. Put a `root` node on widgets. Use stencil clipping on scroll views.
4. Extract reusable pieces into `widgets/<name>/<name>.gui` + `<name>.lua`.
5. Write the screen/popup `.gui_script` with the skeleton above. Create widgets/components **back to front**.
6. Wire `msg.post` / `on_message` for game state. Use `monarch.show` / `monarch.back` only for navigation.
7. Forward `final` / `update` / `on_message` / `on_input` to Druid.

## Workflow: edit existing Druid UI

1. Read the `.gui` and the `.gui_script` / widget `.lua`.
2. Preserve node ids that components already bind to.
3. Add new nodes in the `.gui` first, then bind them in Lua.
4. Do not replace widget colon methods with a different OOP style.
5. Do not drop lifecycle forwarding when touching a gui_script.

## Do not

- Do not use old `component.create()` custom components; widgets replaced them.
- Do not `require "druid.druid"` without parentheses, or use slash paths.
- Do not call `gui.get_node` inside a widget for template children; use `self:get_node`.
- Do not delete GUI nodes that still have a live Druid component.
- Do not put game simulation, save data, or combat math in `.gui_script` / widget files.
- Do not skip `return` on `on_input`.

## Quick recipes

**Button + text (screen gui_script):**
```lua
local monarch = require("monarch.monarch")

self.title = self.druid:new_text("title", "Shop")
self.close = self.druid:new_button("close", function()
    monarch.back()
end)
```

**Scroll + grid (small list, clone all items):**
```lua
self.scroll = self.druid:new_scroll("view", "content")
self.grid = self.druid:new_grid("content", "prefab", 1)
self.scroll:bind_grid(self.grid)

local prefab = gui.get_node("prefab")
gui.set_enabled(prefab, false)
for index = 1, 10 do
    local nodes = gui.clone_tree(prefab)
    local root = nodes["/prefab"]
    gui.set_enabled(root, true)
    self.grid:add(root)
end
```
Inside a widget, replace `gui.get_node("prefab")` with `self:get_node("prefab")` and clone keys with `self:get_template() .. "/prefab"`.

**Long list:** `new_data_list(scroll, grid, create_function)` — only visible rows are created. See `references/components.md`.

**Progress:** node size in the `.gui` is the **full** bar; `new_progress("fill", "x", 0)` then `progress:set_value(0.5)` or `progress:to(0.5)`.

**Popup close:** `new_blocker("dim")` under the window, `new_button("close", monarch.back)`, `new_back_handler(monarch.back)`.
