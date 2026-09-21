# GUI node trees for Druid

Write `.gui` files with the `defold-proto-file-editing` skill (`references/gui.md`). This page only describes **trees Druid expects**.

Node ids are `snake_case`. Default proto fields are omitted (no `position` at 0,0, no `color` at white).

If the project has `defold-sharp-sprite`, every `.gui` uses:

```protobuf
material: "/sharp_sprite/rgss/materials/gui.material"
adjust_reference: ADJUST_REFERENCE_PARENT
```

Otherwise `/builtins/materials/gui.material`.

## Widget root

Every widget GUI starts with a box named `root`. Size it to the widget's hit/layout bounds.

```
root                 TYPE_BOX, size = widget
├── background
├── fill             (progress)
├── label            TYPE_TEXT
└── button           TYPE_BOX (click target)
```

```protobuf
nodes {
  size {
    x: 320.0
    y: 80.0
  }
  type: TYPE_BOX
  id: "root"
  inherit_alpha: true
}
nodes {
  size {
    x: 280.0
    y: 40.0
  }
  type: TYPE_TEXT
  text: "0/100"
  font: "default"
  id: "label"
  parent: "root"
  inherit_alpha: true
}
```

## Button

Click node can be a box (with or without texture) or a text node. Put the visual to animate as a **child** and pass it as `anim_node` when the hit area is larger than the graphic.

```
play                 TYPE_BOX  -- new_button("play")
└── play_label       TYPE_TEXT
```

## Scroll view

```
view                 TYPE_BOX, CLIPPING_MODE_STENCIL, size = visible window
└── content          TYPE_BOX, child, taller/wider than view
    └── prefab       TYPE_BOX, disabled at runtime (clone source)
        └── text     TYPE_TEXT
```

```protobuf
nodes {
  size {
    x: 400.0
    y: 500.0
  }
  type: TYPE_BOX
  id: "view"
  inherit_alpha: true
  clipping_mode: CLIPPING_MODE_STENCIL
}
nodes {
  size {
    x: 400.0
    y: 1200.0
  }
  type: TYPE_BOX
  id: "content"
  parent: "view"
  inherit_alpha: true
}
```

- `view` size = visible area. Stencil on **view**, not content.
- `content` must be a **child** of `view`.
- Prefab: leave enabled in the editor for layout, disable in `init` before cloning: `gui.set_enabled(prefab, false)`.
- Lua: `new_scroll("view", "content")` then `new_grid("content", "prefab", 1)` then `scroll:bind_grid(grid)`.

For data lists, keep the same tree. `in_row` of `1` is a vertical list; `>1` is a grid.

## Progress bar

```
bar_bg               TYPE_BOX (track)
└── fill             TYPE_BOX, size = FULL bar, pivot usually PIVOT_W
```

Lua: `new_progress("fill", "x", 1)`. Direction `"y"` with `PIVOT_S` for vertical. Rotate the node in the scene for diagonal fills; do not invent a diagonal key.

9-slice fills work; keep slice borders smaller than the minimum value you will show.

## Slider

```
track                TYPE_BOX (optional input node)
└── pin              TYPE_BOX  -- new_slider("pin", end_pos)
```

`end_pos` is the pin's local offset at value `1` (for example `vmath.vector3(200, 0, 0)` for a 200px-wide track). Pass `track` to `slider:set_input_node` so clicks on the track move the pin.

## Text input

```
field                TYPE_BOX  -- click_node
└── value            TYPE_TEXT -- text_node, sized to the field
```

Lua: `new_input("field", "value", gui.KEYBOARD_TYPE_DEFAULT)`.

On Android this project's `game.project` already has `[android] input_method = HiddenInputField`.

## Popup / modal

```
root
├── dim              TYPE_BOX, full screen  -- new_blocker("dim")
└── window           TYPE_BOX
    ├── title        TYPE_TEXT
    ├── close        TYPE_BOX               -- new_button("close")
    └── body
```

Create `dim` **after** the screen content widgets, then `window` buttons last so they sit above the blocker on the input stack.

Also `new_back_handler` → `monarch.back()` for Android back.

## HUD (screen GUI)

Keep HUD nodes in the screen `.gui`. Extract repeated meters (health, currency) as widgets instanced via `TYPE_TEMPLATE`.

```protobuf
nodes {
  type: TYPE_TEMPLATE
  id: "health_bar"
  template: "/widgets/health_bar/health_bar.gui"
  inherit_alpha: true
}
```

Lua: `self.druid:new_widget(health_bar, "health_bar")`.

Parent GUI must list any fonts/textures the template needs if the parent also draws with them; the template GUI lists its own.

## Prefab clone keys

`gui.clone_tree` returns a table keyed by **full id**:

| Context | Root key | Child key |
|---|---|---|
| gui_script, node id `prefab` | `"/prefab"` | `"/text"` |
| widget / template id `shop_row`, child `prefab` | `"shop_row/prefab"` | `"shop_row/text"` |
| inside a widget | `self:get_template() .. "/prefab"` | `self:get_template() .. "/text"` |

Never guess; log `nodes` keys once if a clone is nil.

## Layers

Use named layers (`graphics` then `text`) so text batches. Druid does not require specific layer names. Editor command **[Druid] → Assign Layers** exists for humans; agents should add `layers { name: "..." }` in the `.gui` and set `layer` on nodes when draw order/batching matters.

## Layout / container nodes

`new_layout(node)` arranges **direct children** of `node`. Put the children in the `.gui` (or add them at runtime with `layout:add`).

`new_container(node)` treats that node's size as a window slot (`fit` / `stretch`). Pair with `druid.init_window_listener()`.

## Anchors

For HUD corners set `xanchor` / `yanchor` on the **root** of that cluster (or the template instance) so it tracks screen edges. Druid components follow the node; they do not replace Defold adjust mode.
