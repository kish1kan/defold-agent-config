# Druid widgets

Widgets replace the old `druid.component.create()` custom components. Use widgets for every reusable UI piece.

A widget is:

1. A `.gui` scene with a **`root` node** (size = widget bounds; all other nodes are children of `root`).
2. A `.lua` module next to it that returns a table of colon methods.

No `.gui_script` on the widget GUI. Template scripts **do not run**. The parent GUI's script owns Druid.

## Module shape

```lua
---@class shop_row: druid.widget
local M = {}

function M:init(item_id)
    self.item_id = item_id
    self.root = self:get_node("root")
    self.icon = self:get_node("icon")
    self.title = self.druid:new_text("title")
    self.buy = self.druid:new_button("buy", self.on_buy)
end

function M:on_buy()
    msg.post(".", "shop_buy", { item_id = self.item_id })
end

function M:set_title(text)
    self.title:set_text(text)
end

return M
```

Rules:

- `require` with parentheses and dots: `require("widgets.shop_row.shop_row")`.
- Store per-instance fields on `self` (the widget table). Module locals are shared across **all** instances of that widget.
- Local functions only at module scope, never nested inside `M:init`.
- LuaCATS: `---@class name: druid.widget`.
- Colon methods (`function M:init()`) are required so Druid can inject `self.druid`, `get_node`, and lifecycle.
- Do not introduce metatables or a parallel class system.

Inside a widget, **always** `self:get_node("child_id")`. `gui.get_node("child_id")` ignores the template prefix (`"shop_row/child_id"`).

## Instantiation

```lua
self.druid:new_widget(widget_module, [template_id], [nodes], ...)
```

| Argument | When |
|---|---|
| `widget_module` | Result of `require("widgets.foo.foo")` |
| `template_id` | Id of the `TYPE_TEMPLATE` node in the parent `.gui`. `nil` if the widget reads the parent scene's nodes directly |
| `nodes` | `nil` for placed templates; a `gui.clone_tree` table; **or** `"root"` / a root node to clone from |
| `...` | Forwarded to `M:init(...)` |

Placed once in the parent scene:

```lua
local shop_row = require("widgets.shop_row.shop_row")
self.row = self.druid:new_widget(shop_row, "shop_row")
self.row:set_title("Potion")
```

Clone many copies from a prefab template (third argument `"root"` clones that template's root):

```lua
local rows = {}
for i = 1, 10 do
    local row = self.druid:new_widget(shop_row, "shop_row", "root", item_ids[i])
    rows[i] = row
end
```

Explicit clone table:

```lua
local nodes = gui.clone_tree(gui.get_node("shop_row/root"))
local row = self.druid:new_widget(shop_row, "shop_row", nodes, item_id)
```

No template (widget binds parent-scene node ids as-is):

```lua
self.hotkeys = self.druid:new_widget(hotkeys_widget)  -- template_id nil
```

## Lifecycle (optional; define only what you need)

Druid calls these on the widget table:

| Method | When |
|---|---|
| `init(...)` | `new_widget` / `get_widget` |
| `update(dt)` | Every frame (parent must call `self.druid:update`) |
| `on_input(action_id, action)` | Return `true` to consume |
| `on_message(message_id, message, sender)` | GUI messages |
| `on_remove()` | `druid:remove(widget)` or `druid:final()` |
| `on_language_change()` | `druid.on_language_change()` |
| `on_layout_change()` | GUI layout switch |
| `on_window_resized()` | Window listener (see setup.md) |
| `on_input_interrupt()` | Higher-priority component ate input |
| `on_focus_lost()` / `on_focus_gained()` | Focus |

Do not add empty `update` on a widget unless it actually ticks.

## Events on widgets

Expose Defold Event objects so parents subscribe without requiring the widget's internals:

```lua
local event = require("event.event")

function M:init()
    self.on_close = event.create()
    self.druid:new_button("close", function()
        self.on_close:trigger()
    end)
end
```

Parent:

```lua
self.window.on_close:subscribe(function()
    monarch.back()
end)
```

You can also subscribe to inner component events: `self.window.buy.on_click:subscribe(...)`.

## Nested widgets

Create child widgets with the widget's own `self.druid`:

```lua
function M:init()
    local row = require("widgets.shop_row.shop_row")
    self.row_a = self.druid:new_widget(row, "row_a")
    self.row_b = self.druid:new_widget(row, "row_b")
end
```

Template ids are **relative to this widget's template**. Input filters set on `self.druid` inside a widget apply only to that subtree (Druid 1.3+).

## Parent .gui template node

The parent scene instances the widget GUI:

```protobuf
nodes {
  type: TYPE_TEMPLATE
  id: "health_bar"
  template: "/widgets/health_bar/health_bar.gui"
  inherit_alpha: true
}
```

`id` **must** match the `template_id` passed to `new_widget`.

Write `.gui` files with `defold-proto-file-editing`. Put fonts/textures on **both** the widget GUI (so it previews) and the parent if the parent also uses them.

If this project has Sharp Sprite, set `material: "/sharp_sprite/rgss/materials/gui.material"` on the widget GUI.

## Editor helper

In the Defold editor: right-click a `.gui` → **[Druid] → Create Druid Widget**, or **Edit → Create Druid Widget**. That writes a sibling `.lua` from `/druid/templates/widget.lua.template`. Agents should write the Lua file directly with the shape above; they cannot click the editor menu.

## GO widgets (advanced)

Use only when a **game object `.script`** must drive a GUI that has no custom gui_script.

1. Set the GUI's script to `/druid/druid_widget.gui_script` (path inside the Druid library).
2. Defer `druid.get_widget` until after GUI `init` (post a message to self):

```lua
local druid = require("druid.druid")
local panel = require("widgets.panel.panel")

function init(self)
    msg.post("#", "late_init")
end

function on_message(self, message_id, message, sender)
    if message_id == hash("late_init") then
        local gui_url = msg.url(nil, nil, "gui")
        self.panel = druid.get_widget(panel, gui_url, { mode = "hud" })
        self.panel.on_event:subscribe(function()
            msg.post(".", "panel_event")
        end)
    end
end
```

Limits:

- Only **top-level** widget functions and events are wrapped for the GO script.
- Nested widget methods are not exposed.
- The GO script cannot call `gui.*` on widget nodes; go through the widget API.
- Prefer a normal `.gui_script` + `new_widget` for screens and popups.

## Removing widgets

```lua
self.druid:remove(self.row)
-- then delete nodes if you cloned them
gui.delete_node(root)
```

If a whitelist listed that widget, `set_whitelist(nil)` (or rebuild the list) before remove.

## Checklist

- [ ] `root` node exists; children are under it
- [ ] Widget `.lua` next to `.gui`, same `snake_case` name
- [ ] `---@class ...: druid.widget` and `return M`
- [ ] Nodes via `self:get_node`; components via `self.druid:new_*`
- [ ] Parent uses `TYPE_TEMPLATE` + matching `new_widget(module, "id")`
- [ ] Game logic via `msg.post` / events, not `require` of gameplay modules
- [ ] Cloned prefabs start disabled in the scene
