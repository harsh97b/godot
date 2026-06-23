# TalkBack Accessibility — Agent Guide for this Godot Project

**Audience: an AI coding agent making this Godot project accessible to Android TalkBack.**
This project runs on a **custom Godot 4.7 build** (an AAR with AccessKit-based TalkBack support, plus engine changes
that expose **3D nodes** and fix **scroll-container traversal**). This guide is the contract: it tells you exactly
which properties/signals exist, what they do at runtime, and the rules for applying them. Follow it literally — the
behaviors described are what this specific build does, not generic Godot.

> **Scope.** You change **project-side GDScript / scenes only**. Do **not** try to modify engine/Java/AAR code — the
> engine support already exists. Your job: set the right properties on the right nodes, in the right order, and wire a
> few signals. Keep the accessible set **small and meaningful** — a screen reader UX is hurt by exposing clutter.

---

## 0. Mental model (read once)

A TalkBack user navigates by: **swipe right/left** = next/previous element; **drag one finger** = read whatever is
under it (explore-by-touch); **double-tap** = activate the focused element. Each focused element is announced as:

```
<name>  <role>  <state>  <hint>      e.g. "Living Room Light, Button, double-tap to activate"
   ↑       ↑        ↑        ↑
 you set  auto    auto    auto(TalkBack)
```

You almost always only set the **name** (and sometimes description / relations / clickability). Role and state are
automatic. **Two orderings are decided by scene-tree child order**, so they matter as much as the properties:
- **Swipe order** = depth-first scene-tree order.
- **Explore-by-touch on overlapping elements**: Godot sends children in scene-tree order; **TalkBack resolves a
  touch on overlapping elements to the LATER one** (its own hit-test, like paint order — a runtime behavior, not an
  engine feature, but reliable and verified in this project). So put the visually-behind group *before* the in-front
  group in the tree.

---

## 1. Project setup (do once)

In **Project Settings** (enable Advanced Settings to see them):

| Setting | Value | Notes |
|---|---|---|
| `accessibility/general/accessibility_support` | **`1` (Always Active)** for testing, or leave **`0` (Auto)** | `0` = active only while a screen reader runs (fine for shipping on Android, where TalkBack triggers it). **Never `2` — that is _Disabled_.** |
| `accessibility/general/accessibility_driver` | `accesskit` (already the default) | Leave as-is. `dummy` disables it. |
| `accessibility/general/updates_per_second` | `60` (default) | The a11y refresh rate. Leave as-is. |

Nothing else is global; everything below is per-node.

---

## 2. 2D `Control` nodes

`Control`-derived nodes are **automatically in the accessibility tree** with an automatic role (Button→"Button",
CheckBox→"Check box", HSlider→"Slider", LineEdit→"Edit box", Label→static text, …). You mostly add **names**.

### 2.1 Properties available on every Control (Inspector group "Accessibility", or GDScript)

| Property | Type | Effect | Use when |
|---|---|---|---|
| `accessibility_name` | String | The spoken label, said first. **No fallback** — empty ⇒ only the role is announced. Translated (`tr`). | Always, on interactive/meaningful controls. **Mandatory on icon-only buttons.** |
| `accessibility_description` | String | Extra text spoken after name+role. | Helpful context / gesture hints (e.g. "double tap and hold to move"). Never put must-know info only here. |
| `accessibility_live` | enum `AccessibilityServer.LIVE_OFF/LIVE_POLITE/LIVE_ASSERTIVE` | Announce **content changes** while focus is elsewhere. | Status text that updates on its own. Use **POLITE**; reserve ASSERTIVE for urgent/errors. |
| `accessibility_labeled_by_nodes` | `Array[NodePath]` | "Those nodes label me" — field announced using the label's text. Set **on the field**, pointing **at the label**. | A `LineEdit`/input whose label is a separate `Label`. |
| `accessibility_described_by_nodes` | `Array[NodePath]` | "Those nodes are my description" — appended as description. | Visible helper/status text belonging to a control. |
| `accessibility_controls_nodes` | `Array[NodePath]` | "Activating me changes those." | Tabs, expanders, switches whose target isn't obvious. |
| `accessibility_flow_to_nodes` | `Array[NodePath]` | Override reading order: "read that next." **Build chains: ONE entry per node** (multiple = an ambiguous branch). | Reading order must differ from tree order and you can't reorder the tree. |

`Container` nodes (HBox/VBox/Panel/ScrollContainer/…) additionally have:

| Property | Type | Effect | Use when |
|---|---|---|---|
| `accessibility_region` | bool | Promotes the container to a **named landmark region** (set `accessibility_name` too). Users can jump region-to-region via TalkBack reading controls. | Mark the **3–6 big areas** of a screen (top bar, map, bottom panel). Not more. |

Method (all nodes): `queue_accessibility_update()` — re-pushes this node's a11y data. Call after changing a name/state
at runtime if the change doesn't already trigger it.

### 2.2 What is a swipe stop?

- A `Control` registers as a focusable swipe stop when its `focus_mode` allows it. **Buttons/most interactive
  controls default to focusable.** A bare `Label` (default `FOCUS_NONE`) is not "focusable" in Godot's sense, but
  TalkBack still stops on it because it has **text** — text/labels are read on swipe.
- To make a normally-non-focusable Control a deliberate stop **without** making it mouse/keyboard-focusable, set
  `focus_mode = Control.FOCUS_ACCESSIBILITY` and give it a name.

### 2.3 Per-node-type rules

| Node | Do |
|---|---|
| `Button`, `TextureButton`, `CheckBox`, `CheckButton`, `OptionButton`, links | Set `accessibility_name` **if** the visible text isn't self-explanatory; **always** for icon-only. Role + pressed/checked state are automatic. |
| `Label` | Its text is read automatically. Set `accessibility_name` only to override what's spoken. Make it a live region if it updates. |
| `LineEdit` / `TextEdit` | Name it, or link a `Label` via `accessibility_labeled_by_nodes`. |
| `HSlider`/`VSlider`, `ProgressBar` | Name it; value/percent is automatic. |
| Layout containers (HBox/VBox/Margin/Grid) | Leave silent. Optionally mark a major one as a `accessibility_region`. |
| **Decorative** `ColorRect`, `NinePatchRect`, decorative `TextureRect`, separators | **Leave silent — no name.** Don't expose. |

### 2.4 Live regions (dynamic text)

```gdscript
%StatusLabel.accessibility_live = AccessibilityServer.LIVE_POLITE
%StatusLabel.text = "Heating"          # setting Label.text re-queues the a11y update automatically
# If you change content some other way, call %StatusLabel.queue_accessibility_update() after.
```

### 2.5 ScrollContainers and off-screen items (fixed in this build)

This build makes **off-screen children of a `ScrollContainer` reachable by swipe** — when TalkBack reaches the last
visible item it **auto-scrolls** and continues, like native Android.

- **You usually do nothing special.** Just make the scroll children normal focus stops (named buttons, labels, etc.).
  The engine now reports the scroll position/range and re-queues on scroll, so TalkBack can auto-scroll.
- Off-screen children remain clipped (this is correct — like native ListView items: TalkBack scrolls, *then* focuses).
- Don't disable the container's `clip_contents`; it's expected to be on.

### 2.6 Ordering (2D)

- **Swipe order = scene-tree child order.** Arrange the tree top→bottom in the order you want things read. Prefer this
  over `accessibility_flow_to_nodes`.
- A `CanvasLayer`'s **tree position** sets its a11y order even though it renders on top by `layer`.

---

## 3. 3D nodes (`Node3D`) — custom support in this build

Plain Godot does **not** expose 3D to screen readers. This build adds an **opt-in** API on `Node3D`, so 3D geometry
(rooms, device icons, furniture, map objects) can be focused, announced, and activated like Controls.

### 3.1 The API (on `Node3D` and every 3D subclass)

| Member | Type | Effect |
|---|---|---|
| `accessibility_name` | String | **The opt-in.** Empty ⇒ silent (not exposed). Non-empty ⇒ the node becomes a focusable screen-reader element announced with this name. |
| `accessibility_description` | String | Extra spoken detail after the name. |
| `accessibility_clickable` | bool | `false` (default) ⇒ announce-only, role "image". `true` ⇒ role "button" + double-tap fires `accessibility_action_click`. |
| **signal** `accessibility_action_click` | — | Emitted on TalkBack double-tap (only when `accessibility_clickable`). Connect it to your activation handler. |
| **signal** `accessibility_focus_entered` / `accessibility_focus_exited` | — | Emitted when TalkBack focus enters/leaves. Optional. |
| `queue_accessibility_update()` | — | Re-projects this node's on-screen bounds. **Call when the camera/map has moved** (see §3.4). |

### 3.2 The cardinal rule: name the GEOMETRY LEAF, and only it

- Exposure requires the named node to be a **`VisualInstance3D`** (`MeshInstance3D`, `Sprite3D`/`SpriteBase3D`,
  `CSGShape3D`, `MultiMeshInstance3D`, `Label3D`, …) — it needs an AABB to project a focus rectangle. A named **plain
  `Node3D`** (a logical parent/group with no geometry) is **not** exposed.
- So put the name on the **one** visual leaf that represents the object, and leave its siblings/parents unnamed →
  one swipe stop per object. Examples:
  - **Room / floor:** the `MeshInstance3D`.
  - **A class extending `StaticBody3D` with a child `MeshInstance3D`:** name the **child `MeshInstance3D`**, not the body.
  - **A device `.tscn` (root `Node3D` + several `Sprite3D` layers + a `CollisionShape3D`):** name the **one base
    `Sprite3D`** (e.g. `IconBase`); the other layers, the root, and the collider stay unnamed/silent.
  - **A `.glb` instance (furniture):** name its **main `MeshInstance3D`**; other meshes in the glb stay silent.

```gdscript
# Announce-only room:
room_mesh.accessibility_name = "Living Room"          # room_mesh is a MeshInstance3D

# Clickable device (double-tap opens your popup):
var icon := device_instance.get_node("IconArea/IconBase") as Sprite3D
icon.accessibility_name = device.display_name
icon.accessibility_clickable = true
icon.accessibility_action_click.connect(_on_device_activated.bind(device))
# Route to the SAME handler your touch/collision popup uses, so touch and TalkBack behave identically.
# Keep that handler idempotent (don't open the popup twice if both paths fire).
```

### 3.3 Bounds need a camera

The focus rectangle is projected from the node's AABB through the active `Camera3D` (`get_viewport().get_camera_3d()`).
No active camera ⇒ no rectangle. The projection is recomputed whenever the node gets an accessibility update.

### 3.4 Moving map / camera → re-project "on stop" (REQUIRED for moving scenes)

A static object's transform doesn't change when only the **camera** moves, so its rectangle won't follow on its own.
After any camera/map move **settles**, re-queue the named 3D nodes once (not every frame):

```gdscript
func _on_camera_settled() -> void:            # call when your pan/tilt/zoom tween finishes or gesture velocity ≈ 0
    for n in map_root.find_children("*", "VisualInstance3D", true, false):
        if not String(n.accessibility_name).is_empty():
            n.queue_accessibility_update()
```
> Known limitation in this build: a named node that has moved fully off-screen can linger in the swipe order with a
> stale/empty rectangle until its next update. The settle re-queue above is the mitigation — always run it after the
> map moves (including when entering/leaving an edit view that shifts the map).

### 3.5 Ordering (3D) — this also fixes "touch selects the wrong thing"

3D objects overlap on screen (a device icon sits inside a room plane's rectangle). The engine sends children in
scene-tree order; **TalkBack's explore-by-touch resolves the overlap to the LATER element** (its own hit-test, like
paint order — verified empirically in this project: reordering fixed it). Therefore, in the scene tree:

```
Map (Node3D)
├── Rooms     (Node3D group)   ← EARLIER  → touch on empty floor hits the room
│    └── … room MeshInstance3D …
└── Devices   (Node3D group)   ← LATER    → touch on a device icon hits the device (wins over the room behind it)
```
This single ordering gives both the correct touch target (devices win) **and** the natural swipe order
(rooms → devices). If devices come first, touching a device announces the room instead — a real bug. Always put the
**containing/background group before the contained/foreground group.**

### 3.6 Per-screen visibility (e.g. main map vs edit map)

Two cases:

- **The 3D group is hidden on the other screen:** set the group `visible = false`. `is_visible_in_tree() == false`
  removes the whole subtree from TalkBack; `true` restores it. Simplest, correct.
- **The map stays rendered but should be silent on one screen** (e.g. an edit page that only toggles UI and shifts the
  map): you can't use `visible = false`. Instead **toggle names** — exposure is opt-in by name:

```gdscript
func set_furniture_accessible(on: bool) -> void:
    for piece in furniture_root.get_children():
        var mesh := piece.find_children("*", "MeshInstance3D", true, false)[0]   # the representative leaf
        mesh.accessibility_name = piece.display_name if on else ""
    _on_camera_settled()    # the map shifted; re-project everything for the new position
```

Use this exact pattern when something must be accessible on one page and silent on another while staying on screen.

---

## 4. Reusable patterns

### 4.1 Announce a one-off message (e.g. "Sofa placed in Kitchen")
**Preferred (build_num_11 and newer):** one call on the window — it's spoken immediately, no focus needed.
```gdscript
get_window().accessibility_announcement("Sofa placed in Kitchen")
# To re-announce the SAME text, clear first: get_window().accessibility_announcement(""); then call again.
```

**Fallback (older builds where the call above isn't available):** a **live-region Label** that stays *visible in the
tree* (zero-size / transparent / parked off-screen — do **not** use `visible = false`, which prunes it). This is
exactly what the engine does internally:
```gdscript
@onready var announcer: Label = %Announcer

func _ready() -> void:
    announcer.accessibility_live = AccessibilityServer.LIVE_ASSERTIVE   # or LIVE_POLITE
    announcer.modulate.a = 0.0                                          # invisible but still visible_in_tree

func announce(msg: String) -> void:
    announcer.text = ""        # clear first so re-announcing the SAME text still fires (a no-change isn't spoken)
    announcer.text = msg       # setting Label.text re-queues the a11y update -> TalkBack speaks it
```
Call `announce("Sofa placed in Kitchen")`. (`LIVE_ASSERTIVE` interrupts; use `LIVE_POLITE` for non-urgent messages.)

### 4.2 Move screen-reader focus programmatically (Control)
```gdscript
some_control.grab_focus()   # TalkBack's focus follows; the engine forwards GUI focus to the screen reader.
# (For FOCUS_ACCESSIBILITY controls this works only while a screen reader is active.)
```
3D elements have no `grab_focus()`; TalkBack focus follows the *element* across moves, so use a live-region
announcement (4.1) to convey results.

### 4.3 "Hold and move" / drag-to-reposition element
1. The gesture already works: TalkBack **double-tap-and-hold passes through as a real long-press** at the focused
   element → your existing hold+drag code runs.
2. Advertise it: `element.accessibility_description = "Double tap and hold, then drag to move"`.
3. Announce the result with the live-region Label (§4.1); if it's a Control, `grab_focus()` it afterward.
4. Better for many users: also offer discrete **custom actions** (next section) like "Move to Kitchen" instead of
   relying only on free-form drag.

### 4.4 Custom actions (advanced; appear in TalkBack's actions menu)
Custom actions must be (re)registered inside the node's `NOTIFICATION_ACCESSIBILITY_UPDATE`:
```gdscript
func _notification(what):
    if what == NOTIFICATION_ACCESSIBILITY_UPDATE:
        var ae := get_accessibility_element()
        AccessibilityServer.get_singleton().update_add_custom_action(ae, 1, "Move to Kitchen")
        AccessibilityServer.get_singleton().update_add_custom_action(ae, 2, "Move to Bedroom")
# Handle the chosen action via the accessibility action callback the server invokes (action id is passed through).
```
Only use this when discrete actions genuinely improve the UX; otherwise keep it simple.

---

## 5. Do / Don't (the rules that matter most)

**DO**
- Name every interactive or meaningful element; **always** name icon-only buttons.
- For 3D, name the single **VisualInstance3D geometry leaf**, set `accessibility_clickable` + connect
  `accessibility_action_click` for activatable 3D objects.
- Order siblings so the **background group precedes the foreground group** (rooms before devices) — fixes touch + swipe.
- Re-queue named 3D nodes after the camera/map settles.
- Use `accessibility_live = LIVE_POLITE` for self-updating status text.
- Mark only the 3–6 major areas as `accessibility_region`.

**DON'T**
- Don't name decorative nodes (images, separators, layout containers) — keep them silent.
- Don't put the role word in the name ("Settings button" → announced "Settings button, Button").
- Don't name a plain `Node3D` group and expect it to be focusable — it has no bounds; name the geometry leaf.
- Don't fan out `accessibility_flow_to_nodes` (multiple targets = ambiguous branch); build single-entry chains.
- Don't overuse `LIVE_ASSERTIVE` — it interrupts the user.
- Don't set `accessibility_support = 2` (that's Disabled).

---

## 6. Verification checklist (turn TalkBack ON and test)

- [ ] Swipe right from the top walks elements in the intended order; nothing important is skipped; no junk stops.
- [ ] Every interactive element announces a clear name (no bare "Button"/"Image").
- [ ] Icon-only buttons are named.
- [ ] 3D: each room/device/furniture announces its name; the green box ≈ the object; double-tap on a clickable one
      activates it once.
- [ ] **Touch a device icon → the device is announced, not the room behind it** (ordering check).
- [ ] Move the map and let it settle → focus boxes follow.
- [ ] Scroll row: swiping past the last visible item auto-scrolls and continues to the off-screen items.
- [ ] Per-screen: the right things are silent/accessible on each page.
- [ ] Game touch/clicks still work normally (accessibility doesn't intercept input).

---

## 7. Out of scope / known limitations (do NOT attempt to "fix" these here)

- **Native Android popups/dialogs shown by the host app** (not drawn by Godot) are separate Android views — making
  them TalkBack-reachable is host-app work, not project-side Godot.
- **A 3D element fully off-screen** can still appear as a stale/empty swipe stop in this build (see §3.4) — mitigate
  with the settle re-queue; a full engine fix is pending.
- **Spoken role wording for 3D** (e.g. dropping the "Image"/"Button" word) is not configurable from the project here.
- Engine/AAR changes are out of scope — this guide is project-side only.

---

## 8. Quick reference

```gdscript
# 2D Control
ctrl.accessibility_name = "Settings"
ctrl.accessibility_description = "Opens app settings"
ctrl.accessibility_live = AccessibilityServer.LIVE_POLITE
ctrl.focus_mode = Control.FOCUS_ACCESSIBILITY          # make a non-interactive control a deliberate stop
input.accessibility_labeled_by_nodes = [label.get_path()]
container.accessibility_region = true; container.accessibility_name = "Bottom panel"
ctrl.queue_accessibility_update()                       # after a runtime change that didn't auto-refresh

# 3D Node3D (this build)
mesh.accessibility_name = "Living Room"                 # VisualInstance3D leaf; empty = silent
icon.accessibility_clickable = true
icon.accessibility_action_click.connect(handler)
mesh.queue_accessibility_update()                       # after the camera/map settles

# App-level announcement (build_num_11+); older builds: a live-region Label, see §4.1
get_window().accessibility_announcement("Saved")        # one-off spoken message
```
</content>
