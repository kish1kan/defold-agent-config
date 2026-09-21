# Druid component catalog

Constructors live on the instance from `druid.new(self)`. First arguments that take a node accept a **node id string** or a node.

Prefer `set_text` / `set_value` over deprecated `set_to`.

After adding a dependency, inspect `.deps/druid/` or https://raw.githubusercontent.com/Insality/druid/master/api/quick_api_reference.md for fields not listed here.

## Base helpers (every component and widget)

```lua
component:get_node(node_id)
component:get_druid([template], [nodes])
component:set_input_enabled(state)
component:set_input_priority(value, [is_temporary])
component:reset_input_priority()
component:set_style([style_table])
self.druid:remove(component)
```

Widgets also have `self.druid` (a nested instance) so inner `new_button` calls belong to that widget.

## Button

```lua
local button = self.druid:new_button(node, [callback], [params], [anim_node])
```

- `node` — click zone. `anim_node` — node to scale/animate (icon on a large hit area).
- Constructor callback: `function(self, params, button)`.
- Events: `on_click`, `on_pressed`, `on_repeated_click`, `on_long_click`, `on_double_click`, `on_hold_callback`, `on_click_outside`.
- `button:set_enabled(false)` — not clickable; style `on_set_enabled` runs.
- `button:set_key_trigger("key_space")`
- `button:set_click_zone(stencil_or_parent)` — extra clip; stencil parents are auto-detected.
- `button:set_check_function(fn, failure_cb)` — `fn` must return true to allow the click.
- `button:set_animations_disabled()` — skip default scale/hover style.
- HTML5 clipboard/keyboard: `button:set_web_user_interaction(true)` (on_click only).

Create **later** than content underneath so the button wins the input stack.

## Text

```lua
local text = self.druid:new_text(node, [value], [adjust_type])
```

Fits text into the **node size from the .gui file**. Default adjust is `"downscale"`.

| `adjust_type` | Behavior |
|---|---|
| `downscale` | Scale down to fit (default) |
| `trim` / `trim_left` | Ellipsis (`style.TRIM_POSTFIX`) |
| `no_adjust` | Raw Defold text |
| `downscale_limited` | Scale down to `set_minimal_scale` |
| `scroll` / `scale_then_scroll` | Pivot-shift inside the box |
| `scale_then_trim` / `scale_then_trim_left` | Scale then trim |

```lua
text:set_text(str)
text:get_text()
text:set_color(vmath.vector4(1))
text:set_alpha(1)
text:set_pivot(gui.PIVOT_W)
text:set_text_adjust("trim", [minimal_scale])
text:get_text_size([str])  -- width, height
```

Events: `on_set_text`, `on_update_text_scale`, `on_set_pivot`.

## Lang text

```lua
local lang_text = self.druid:new_lang_text(node, [locale_id], [adjust_type])
lang_text:translate("ui_shop_title", extra_format_args...)
lang_text:format(a, b, c)  -- format current locale string
```

Requires `druid.set_text_function` (see `setup.md`). After changing language call `druid.on_language_change()`.

## Scroll

```lua
local scroll = self.druid:new_scroll(view_node, content_node)
```

- `view_node` — static hit area; enable **stencil clipping** on it.
- `content_node` — **child** of view; its size is the scrollable range. Druid moves this node.

```lua
scroll:bind_grid(grid)          -- content size follows grid
scroll:bind_layout(layout)
scroll:set_horizontal_scroll(false)
scroll:set_vertical_scroll(true)
scroll:set_inert(true)
scroll:set_extra_stretch_size(0) -- 0 = no rubber band
scroll:scroll_to(vmath.vector3(x, y, 0), [instant])
scroll:scroll_to_percent(vmath.vector3(0, 1, 0), [instant])  -- 0..1
scroll:scroll_to_make_node_visible(node, [instant])
scroll:is_node_in_view(node)
```

Events: `on_scroll`, `on_scroll_to`, `on_point_scroll`.

In Druid 1.3, `scroll.hover` is **removed**. If you need hover on the view, `new_hover(view_node)` yourself.

Wheel events pass through when the scroll cannot move (nested scrolls work).

## Static grid

```lua
local grid = self.druid:new_grid(parent_node, item, [in_row])
```

- `parent_node` — usually the scroll content node.
- `item` — prefab node (or size source). Grid reads item size.
- `in_row` — columns. `1` = vertical list.

```lua
grid:add(node, [index], [shift_policy], [is_instant])
grid:remove(index, [shift_policy], [is_instant])
grid:clear()
grid:set_in_row(n)
grid:set_item_size(w, h)
grid:set_anchor(vmath.vector3(0.5, 1, 0))
grid:refresh()
```

Events: `on_add_item`, `on_remove_item`, `on_change_items`, `on_clear`, `on_update_positions`.

Always `scroll:bind_grid(grid)` when the grid sits in a scroll.

## Data list (virtualized)

Use when the list can be long. Only visible rows exist.

```lua
local data_list = self.druid:new_data_list(scroll, grid, create_function)
data_list:set_data(array)
```

`create_function(self, data, index, data_list)` **must return** `(root_node, [component])`.

```lua
function M:create_item_callback(item_data, index)
    local nodes = gui.clone_tree(self.prefab)
    local prefix = self:get_template() .. "/"
    local root = nodes[prefix .. "prefab"]
    local label = nodes[prefix .. "text"]
    gui.set_enabled(root, true)
    gui.set_text(label, item_data.title)
    local button = self.druid:new_button(root, self.on_row_click, item_data)
    return root, button
end
```

In a **gui_script** (no template), clone keys are `"/prefab"` and `"/text"` (leading slash, no template prefix).

```lua
data_list:add(data, [index], [shift_policy])
data_list:remove([index], [shift_policy])
data_list:clear()
data_list:scroll_to_index(index)
data_list:refresh()   -- call if item size changed without data change
data_list:set_use_cache(true)  -- pair with on_element_add / on_element_remove
```

Events: `on_element_add`, `on_element_remove`, `on_scroll_progress_change`.

Hide the prefab: `gui.set_enabled(prefab, false)` in `init`.

## Blocker

```lua
local blocker = self.druid:new_blocker(node)
blocker:set_enabled(true)
```

Eats input over the node AABB. Create **after** the dimmed content so it sits on top of the input stack. Use a full-screen box behind a popup.

## Back handler

```lua
self.druid:new_back_handler(callback, [params])
```

Fires on Android `key_back` / keyboard `key_backspace`. Typical popup: `monarch.back`.

## Hover / swipe / drag

```lua
self.druid:new_hover(node, [on_hover], [on_mouse_hover])
self.druid:new_swipe(node, [on_swipe])
self.druid:new_drag(node, [on_drag])
```

- Hover events: `on_hover`, `on_mouse_hover` (touch vs mouse).
- Swipe event: `on_swipe`. `swipe:set_enabled(state)` (1.3+).
- Drag events: `on_touch_start`, `on_touch_end`, `on_drag_start`, `on_drag`, `on_drag_end`.
- `drag.hover` exists only if **DefOS** is in the project; otherwise it is `nil`. Guard with `if drag.hover then`.
- Extra drag buttons: `drag:add_drag_action(action_id)`.

## Progress

```lua
local progress = self.druid:new_progress(node, "x", [init_value])  -- "x" or "y", value 0..1
progress:set_value(0.5)     -- instant
progress:to(0.8, [callback]) -- animated
progress:fill()
progress:empty()
progress:get_value()
progress:set_max_size(vmath.vector3(w, h, 0))
progress:set_steps({0.25, 0.5, 0.75, 1}, callback)
```

The node size in the `.gui` is the **full** bar. 9-slice: size until min slice, then scale. Event: `on_change`.

If the fill looks dark/glitchy, disable mipmaps on that atlas in texture profiles.

## Slider

```lua
local slider = self.druid:new_slider(pin_node, end_pos, [callback])
slider:set(value, [is_silent])  -- 0..1
slider:set_steps({0, 0.25, 0.5, 0.75, 1})
slider:set_input_node(track_node) -- click anywhere on the track
slider:set_enabled(true)
```

`end_pos` is a `vector3` offset of the pin at value `1`. Event: `on_change_value`.

## Input (text field)

```lua
local input = self.druid:new_input(click_node, text_node, [keyboard_type])
input:set_text("hello")
input:get_text()
input:get_text_visual()  -- may differ if text adjust trims
input:set_max_length(16)
input:set_allowed_characters("[%a%d]")  -- Lua pattern
input:select()
input:unselect()
```

`keyboard_type` is a `gui.KEYBOARD_TYPE_*` constant.

Events: `on_input_select`, `on_input_unselect`, `on_input_text`, `on_input_empty`, `on_input_full`, `on_input_wrong`, `on_select_cursor_change`.

Modifier keys and `tab` are **not** consumed while selected (1.3+), so hotkeys still fire. Disable hotkeys on `on_input_select` if typing should block them.

## Rich text

```lua
local rich = self.druid:new_rich_text(text_node, [value])
rich:set_text("<color=#ff0000>Hi</color> <size=1.5>there</size>")
```

Markup (real tags, not HTML entities): `color`, `shadow`, `outline` (name / `r,g,b,a` / `#hex`), `font=FontName`, `size=2`, `br/`, `nobr`, `img=atlas:image[,w,h]`. Custom tags via `rich:tagged("mytag")`.

The text node is a prefab; Druid clones words from it. Set width with `rich:set_width(w)`.

## Rich input

```lua
local rich_input = self.druid:new_rich_input(template, [nodes])
```

Uses Druid's own GUI template (caret, placeholder). Copy/adapt from `.deps/druid/` examples. Prefer plain `new_input` unless you need that chrome.

```lua
rich_input:set_placeholder("Name")
rich_input:set_text("")
rich_input:get_text()
rich_input:set_allowed_characters("[%a%d ]")
```

## Layout

```lua
local layout = self.druid:new_layout(node, [mode])  -- "horizontal" | "vertical" | "horizontal_wrap"
layout:add(child_node)
layout:set_margin(x, y)
layout:set_padding(x, y, z, w)
layout:set_justify(true)
layout:set_hug_content(true, true)
layout:refresh_layout([is_instant])
```

Event: `on_size_changed`. Bind to scroll with `scroll:bind_layout(layout)`.

## Container (window resize)

```lua
local container = self.druid:new_container(node, [mode], [on_resize])
-- mode: "fit" | "stretch" | "stretch_x" | "stretch_y"
container:fit_into_window()
container:fit_into_size(size)
container:set_min_size(w, h)
container:set_max_size(w, h)
```

Call `druid.init_window_listener()` once (see `setup.md`) or window events never reach containers.

## Timer

```lua
local timer = self.druid:new_timer(text_node, [seconds_from], [seconds_to], [callback])
timer:set_interval(from, to)
timer:set_state(true)  -- running
timer:set_value(seconds)
```

Events: `on_tick`, `on_timer_end`, `on_set_enabled`. Needs `druid:update(dt)`.

## Hotkey

```lua
local hotkey = self.druid:new_hotkey({"key_lctrl", "key_s"}, callback, [arg])
hotkey:add_hotkey(keys, [arg])
hotkey:set_repeat(true)
```

`keys_array` is one action key plus optional modifiers (`key_lshift`, `key_lctrl`, `key_lalt`, `key_lsuper`). Events: `on_hotkey_pressed`, `on_hotkey_released`.

## Helper / color / const

```lua
local helper = require("druid.helper")
local color = require("druid.color")
local const = require("druid.const")
```

Useful helpers: `helper.pick_node`, `helper.get_closest_stencil_node`, `helper.is_mobile`, `helper.centrate_text_with_icon`, `helper.get_pivot_offset`.

Grid/data_list shift: `const.SHIFT.LEFT`, `const.SHIFT.RIGHT`, `const.SHIFT.NO_SHIFT` as `shift_policy`.

## Input filters (Druid 1.3+)

```lua
self.druid:set_whitelist({ component_or_widget })
self.druid:set_blacklist({ component_or_widget })
self.druid:set_whitelist(nil)  -- clear
```

- Called on the **gui_script** instance: filters the whole GUI.
- Called on `self.druid` **inside a widget**: filters that widget's subtree only.
- A listed component matches itself **and descendants** created later.
- The filter **owner** is never blocked by its own list.
- Clear a whitelist before `remove()` on its last member; an empty leftover whitelist blocks the whole scope.

## Choosing a component

| Need | Use |
|---|---|
| Click / hold / double-tap | Button |
| Label that must stay inside a box | Text |
| Localized label | Lang text |
| Overlay that eats clicks | Blocker |
| Android back / Esc-style back | Back handler |
| Small scrollable column | Scroll + Grid |
| Hundreds of rows | Data list |
| HP / reload / capture bar | Progress |
| Volume / settings value | Slider |
| Player types a string | Input |
| Colored / mixed-size copy | Rich text |
| Auto-pack children | Layout |
| Scale UI to window | Container |
| Keyboard shortcut | Hotkey |
| Reusable composite | Widget (`new_widget`) |
