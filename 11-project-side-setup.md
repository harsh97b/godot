# 11 — Project-side setup: what YOU do in your Godot project

**The authoritative, current how-to** for making your home-map project TalkBack-accessible using the engine-side 3D
accessibility shipped in **`build_num_07`**. This is the project-side companion to the engine design in
[`10-architecture-3d-a11y.md`](10-architecture-3d-a11y.md). It supersedes the older overlay advice in
[`06-project-accessibility-guide.md`](06-project-accessibility-guide.md) for 3D, and the pre-implementation specifics
in [`07-usecase-home-map.md`](07-usecase-home-map.md).

> **No engine work needed on your side** — the AAR already contains everything. You only set a few properties and
> connect one signal in GDScript (or the Inspector). All of this runs on the **4.7 editor + the `build_num_07` AAR**.

---

## 0. One-time project setup

**Project → Project Settings** (enable *Advanced Settings* to see them):
- `accessibility/general/accessibility_support` = **Always Active** (`2`)
- `accessibility/general/accessibility_driver` = **`accesskit`**
- (optional) `accessibility/general/updates_per_second` — the a11y flush rate (default ~10 Hz). Leave as-is.

That's it for setup. Everything else is per-node.

---

## 1. The API you use (added to `Node3D`, so every 3D node inherits it)

Set these on the **geometry leaf** (the node that actually has the shape) — see §2 for which node that is.

| Member | Type | What it does |
|---|---|---|
| `accessibility_name` | `String` | The spoken label. **Empty ⇒ the node is silent** (not a swipe stop). Setting it is the whole opt-in. |
| `accessibility_description` | `String` | Extra text spoken after the name (e.g. a status). Optional. |
| `accessibility_clickable` | `bool` | `false` ⇒ announce-only (`ROLE_IMAGE`). `true` ⇒ a **button** (`ROLE_BUTTON`) that TalkBack can **double-tap**. Use for devices. |
| **signal** `accessibility_action_click` | — | Emitted when TalkBack **double-taps** the node. Connect it to open your popup. |
| **signal** `accessibility_focus_entered` / `accessibility_focus_exited` | — | Emitted when TalkBack focus enters / leaves the node. Optional (handy for "follow focus" effects). |
| `queue_accessibility_update()` *(inherited from `Node`)* | — | Re-runs the node's accessibility update (re-projects its on-screen bounds). Call after the camera/map moves, or after you change a name/clickable at runtime. |

**Rules the engine enforces (so you don't have to):**
- Only a **`VisualInstance3D`** (`MeshInstance3D`, `Sprite3D`, `CSG*`, `MultiMeshInstance3D`, `Label3D`, …) with a
  **non-empty `accessibility_name`** becomes a focusable element. A named *plain* `Node3D` (no geometry/AABB) stays
  silent — so put the name on the visual leaf, not a logical parent.
- The focus rectangle is **projected from the node's 3D AABB to the screen**, and needs an **active `Camera3D`** in
  the viewport. No camera → no bounds (the node just won't get a box that frame; it won't crash).
- A node whose `is_visible_in_tree()` is `false` is **removed from TalkBack** (so hiding a parent hides the subtree).
- **Swipe order = scene-tree child order** (depth-first). Arrange the tree in the order you want things read.

---

## 2. Your three focusable target types — where the name goes

| Target | Your node structure | Node that carries `accessibility_name` |
|---|---|---|
| **Room** | a `MeshInstance3D` floor | the **`MeshInstance3D`** |
| **Plane component** | your class `extends StaticBody3D`, with a **child `MeshInstance3D`** (mesh set from data) | the **child `MeshInstance3D`** (NOT the `StaticBody3D`) |
| **Device** | a PackedScene, **root `Node3D`** + `Sprite3D` layers (`Icon`/`IconBase`/`IconBase2D`/`IconShadow`) + a `CollisionShape3D` sibling | the **`IconBase` `Sprite3D`** |

The `StaticBody3D` parent, the device-root `Node3D`, the other `Sprite3D` layers, and the `CollisionShape3D` all stay
**unnamed → silent**, so each plane/device is exactly **one** swipe stop.

---

## 3. Wire it up

### 3.1 Rooms (announce-only)
```gdscript
# In the code that builds the floor, or via the Inspector "Accessibility" group.
room_mesh.accessibility_name = "Living Room"          # room_mesh is the MeshInstance3D
```

### 3.2 Plane components (announce-only)
```gdscript
# Inside your StaticBody3D-derived plane class, after you add the child MeshInstance3D:
mesh_instance.accessibility_name = plane_data.display_name      # e.g. "North Wall"
# (leave the StaticBody3D itself unnamed)
```

### 3.3 Devices (clickable → popup)
Devices are loaded as `PackedScene`s and instanced many times. For each instance, name the `IconBase`, mark it
clickable, and connect the click signal to the **same handler your touch popup already uses**:
```gdscript
func _spawn_device(device) -> void:
    var inst: Node3D = preload("res://devices/device.tscn").instantiate()
    add_child(inst)

    var icon := inst.get_node("IconArea/IconBase") as Sprite3D     # the named leaf
    icon.accessibility_name = device.display_name                  # e.g. "Smart Light, Living Room"
    icon.accessibility_description = device.status_text            # optional, e.g. "On"
    icon.accessibility_clickable = true                            # ROLE_BUTTON + double-tap
    icon.accessibility_action_click.connect(_on_device_activated.bind(device))

func _on_device_activated(device) -> void:
    # The SAME function your CollisionShape3D / input_event path calls to open the popup.
    _open_device_popup(device)
```
- **Touch and TalkBack both open the popup**: touch still goes through your `CollisionShape3D` picking; a TalkBack
  double-tap fires `accessibility_action_click` → `_on_device_activated`. They call the same code, so behavior matches.
- The engine invokes the click **deferred on the main thread**, so opening UI from the handler is safe.
- **Avoid double-firing:** make sure `_open_device_popup` is idempotent (or guards against an already-open popup), so
  a stray case where both paths fire doesn't open it twice.

### 3.4 Swipe order — structure the scene tree top→bottom
Traversal is depth-first scene-tree order, so lay the main scene out in reading order:
```
Main
├── TopUI         (Control / CanvasLayer)      ← read FIRST  (top controls)
├── Map           (Node3D)                     ← read NEXT
│    ├── Rooms       (Node3D, unnamed → silent group)
│    │    ├── LivingRoom (MeshInstance3D)        accessibility_name = "Living Room"
│    │    └── Kitchen    (MeshInstance3D)        …
│    └── Devices     (Node3D, unnamed → silent group)
│         ├── Light1     (device .tscn → IconBase named + clickable)
│         └── Thermostat1(device .tscn → …)
└── BottomUI      (Control / CanvasLayer)       ← read LAST   (bottom controls)
```
Result: **top controls → rooms → devices → bottom controls.** Keep `Rooms` before `Devices` so rooms are announced
first. (A `CanvasLayer`'s *tree position* sets its a11y order even though it still renders on top.)

> If you ever can't physically reorder the tree, fall back to `accessibility_flow_to_nodes` to chain the sequence, or
> a root node overriding `accessibility_override_tree_hierarchy()` (the `TabContainer` pattern).

### 3.5 Keep the focus box on the geometry as the camera moves
The box re-projects whenever a node gets an accessibility update. During a rotate/tilt you don't want that every
frame — do it **once when motion settles**:
```gdscript
# Call _on_camera_settled() when your camera/map tween finishes,
# or when gesture velocity ≈ 0 (debounce so it fires once).
func _on_camera_settled() -> void:
    for n in $Map.find_children("*", "VisualInstance3D", true, false):
        if not String(n.accessibility_name).is_empty():
            n.queue_accessibility_update()       # re-project → focus rectangle follows
```
A static device's transform doesn't change when only the camera moves, so its box won't move on its own — this
re-queue is what makes it follow. One call at the end of the move is enough (the projection uses the camera's
current transform at update time).

### 3.6 Per-screen behavior (home ↔ edit)
**If the edit screen hides the map** (simplest):
```gdscript
func show_edit_screen() -> void:
    $Map.visible = false        # is_visible_in_tree()==false → rooms/devices leave TalkBack
    $HomeUI.visible = false
    $EditUI.visible = true
func show_home_screen() -> void:
    $Map.visible = true         # rooms/devices return
    $HomeUI.visible = true
    $EditUI.visible = false
```
The engine re-runs each descendant's handler on the visibility change, so the 3D nodes go silent immediately and
restore on show — no off-screen leakage.

**If the edit screen keeps the map rendered but should not expose it to TalkBack**, don't use `visible=false` (that
hides it visually). Instead clear the names while in edit mode and restore them on home — opt-out is just an empty
name:
```gdscript
func _set_map_accessible(on: bool) -> void:
    for n in $Map.find_children("*", "VisualInstance3D", true, false):
        if on:
            n.accessibility_name = _saved_names.get(n, "")   # restore (cache names when you first set them)
        else:
            n.accessibility_name = ""                        # silent, but still rendered
```

---

## 4. The 2D Control UI (the easy part)
`Control` nodes are auto-exposed; you mostly just give them names.
- **Set `accessibility_name`** on every `Button` / `TextureButton` (icon-only buttons **must** have one), and on
  meaningful `Label`s/icons. Role is automatic per Control type.
- **Leave decorative nodes silent** (no name): `ColorRect`, `NinePatchRect`, decorative `TextureRect`,
  `HSeparator`/`VSeparator`, pure layout containers.
- **Live status text** (e.g. a changing value): set `accessibility_live = LIVE_POLITE` and call
  `queue_accessibility_update()` when the text changes, so TalkBack announces it without needing focus.
- **Relationships:** `accessibility_labeled_by_nodes` / `accessibility_described_by_nodes` to borrow a nearby label;
  `accessibility_controls_nodes` / `accessibility_flow_to_nodes` for read-order hints.

> Two known 2D traversal quirks (off-screen scroll-container items; certain deeply-nested buttons skipped on swipe)
> are **engine-side** issues being handled separately — **not** something you fix in the project. See
> [`09-requirements.md`](09-requirements.md) R3.7.

---

## 5. On-device checklist (TalkBack on)
- [ ] Swipe right from the top: **top controls → each room → each device → bottom controls**, in order; left reverses.
- [ ] Each **room** announces its name; the green focus box ≈ the **floor area** (not a dot).
- [ ] Each **device** announces "name, room"; box ≈ the **icon**; **double-tap opens the popup** (exactly once).
- [ ] Rotate/tilt the map, let it **settle** → focus boxes **follow** the new positions.
- [ ] **Edit screen** → rooms/devices are **silent**; only the edit Control UI is reachable. Back to **home** → 3D returns.
- [ ] **Scaled display:** if your window uses `content_scale_factor ≠ 1`, the box still lands on the geometry. *(This
      is the #1 thing to confirm — see [`10-architecture-3d-a11y.md`](10-architecture-3d-a11y.md) MUST-verify list.)*
- [ ] **Game touch / pan / zoom / tap still work** exactly as before — engine-side exposure does not intercept input.
- Capture `adb logcat godot:V AccessKit:V TalkBack:V *:E` for anything that misses, and report it.

## 6. Quick reference
```gdscript
# Make a 3D node a TalkBack swipe stop:
node.accessibility_name = "Spoken label"          # node must be a VisualInstance3D; empty = silent
node.accessibility_description = "extra text"      # optional
node.accessibility_clickable = true               # optional → button + double-tap
node.accessibility_action_click.connect(handler)  # optional → handler runs on double-tap
node.queue_accessibility_update()                 # after camera/map move, or runtime name change
```
