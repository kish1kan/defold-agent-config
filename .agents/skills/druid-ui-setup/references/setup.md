# Druid project setup

## Dependencies

Add **both** (order does not matter). Druid 1.3.x documents Defold Event **16**:

```
https://github.com/Insality/defold-event/archive/refs/tags/16.zip
https://github.com/Insality/druid/archive/refs/tags/1.3.1.zip
```

Require paths after fetch:

- `require("druid.druid")`
- `require("event.event")`
- `require("druid.helper")`
- `require("druid.color")`
- `require("druid.const")`

After changing `game.project` dependencies, run `defold-project-setup` and tell the user to **Project → Fetch Libraries**.

## Input bindings

Druid defaults to names from `/builtins/input/all.input_binding`. This repo already sets:

```
[input]
game_binding = /builtins/input/all.input_bindingc
```

Required mappings:

| Trigger | Action |
|---|---|
| Mouse Button 1 | `touch` |
| Wheel up / down | `mouse_wheel_up` / `mouse_wheel_down` |
| Backspace | `key_backspace` |
| Back (Android) | `key_back` |
| Touch multi | `touch_multi` |

Optional but used by Input / Rich Input / Hotkey / navigation: `key_enter`, `key_esc`, `key_space`, arrows, `key_lshift`, `key_lctrl`, `key_lalt`, `key_lsuper`, `key_tab`.

To rename actions, set `[druid]` keys in `game.project` (defaults shown):

```
[druid]
input_touch = touch
input_text = text
input_marked_text = marked_text
input_key_esc = key_esc
input_key_back = key_back
input_key_enter = key_enter
input_key_space = key_space
input_key_backspace = key_backspace
input_multitouch = touch_multi
input_scroll_up = mouse_wheel_up
input_scroll_down = mouse_wheel_down
input_key_left = key_left
input_key_right = key_right
input_key_up = key_up
input_key_down = key_down
input_key_lshift = key_lshift
input_key_lctrl = key_lctrl
input_key_lalt = key_lalt
input_key_lsuper = key_lsuper
input_key_tab = key_tab
```

Disable auto `acquire_input_focus`:

```
[druid]
no_auto_input = 1
```

Only do this if something else already acquires focus for that GUI.

## Styles

A style is a table keyed by component name. Set it once at boot (main script or first GUI), not in every widget.

```lua
local druid = require("druid.druid")
local style = require("main.ui_style") -- copy from druid/styles/default/style.lua in .deps

druid.set_default_style(style)
```

Per instance: `druid.new(self, style)`. Per component: `component:set_style(style)`.

Patch one field after create:

```lua
self.grid.style.IS_ALIGN_LAST_ROW = true
self.drag.style.DRAG_DEADZONE = 0
```

Do not edit files inside `.deps/`. Copy the default style into the project (`main/ui_style.lua` or similar) and require that.

Default button style scales on hover/click and can play named sounds via `set_sound_function`. Call `button:set_animations_disabled()` for a mute, no-scale button.

## Sounds

```lua
druid.set_sound_function(function(sound_id)
    sound.play("/sounds#" .. sound_id)
end)
```

`sound_id` is a string from the style (for example button click). GUI scripts have `sound.*` available. Keep actual `.sound` components on a game object; the callback only plays by id.

## Localization

```lua
druid.set_text_function(function(locale_id)
    return my_strings[locale_id] or locale_id
end)
```

Use `new_lang_text("title", "ui_shop")`. When the language changes:

```lua
druid.on_language_change()
```

That refreshes lang_text (and widget `on_language_change`).

If the project later adds Defold-Lang (`Insality/defold-lang`), pass `lang.txp` as the text function.

## Window resize

Containers and some components need window events:

```lua
druid.init_window_listener()
```

This **replaces** any existing `window.set_listener`. If the game already has a listener, call Druid from it instead:

```lua
window.set_listener(function(self, event)
    druid.on_window_callback(event)
    -- game-specific handling
end)
```

Call this once at boot, not in every GUI.

## Monarch + Druid

They stack cleanly:

- Monarch loads the collection that contains the GUI.
- The GUI script creates `druid.new(self)` and widgets.
- Screen change: `monarch.show("gameplay", { clear = true })` from a button callback.
- Popup close: `monarch.back()` from `new_button` / `new_back_handler`.
- Still do not `require` gameplay modules from the GUI.

Wait for Monarch registration before the first `show` (see `monarch-screen-setup`). Druid init runs in the **screen's** gui_script `init`, which is after that screen is shown — no extra wait needed for Druid.

## Logging

`print()` on widget init, button clicks that start transactions, and data_list size. Druid also has `druid.set_logger` / `druid.get_logger` if the project wants structured logs; default `print` is enough.

## Size

All components ship in 1.1+. Stripping unused ones means forking `druid/system/druid_instance.lua` **inside the project** (never `.deps/`). Skip unless binary size is a stated goal. See https://github.com/Insality/druid/blob/master/wiki/optimize_druid_size.md.

## Official links

- README / setup: https://github.com/Insality/druid
- Quick API: https://raw.githubusercontent.com/Insality/druid/master/api/quick_api_reference.md
- Widgets wiki: https://raw.githubusercontent.com/Insality/druid/master/wiki/widgets.md
- Styles wiki: https://raw.githubusercontent.com/Insality/druid/master/wiki/styles.md
- Advanced setup: https://raw.githubusercontent.com/Insality/druid/master/wiki/advanced-setup.md
- HTML examples: https://insality.github.io/druid/
