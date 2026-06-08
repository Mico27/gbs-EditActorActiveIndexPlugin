# gbs-EditActorActiveIndexPlugin

**Version 4.3.0 — Requires GB Studio ≥ 4.3.0**

A GB Studio engine plugin that provides fine-grained control over the order in which actors are rendered and assigned hardware OAM slots. It adds four events: read/write an actor's position in the active actor linked list, sort all active actors by their vertical (Y) position, and set a global automatic sort mode that runs each frame. It also introduces a **flicker** mode that cycles which actors occupy the first OAM hardware sprite slots, distributing the hardware 10-sprites-per-scanline limit across all actors over multiple frames.

> **Note:** OAM order affects sprite rendering priority on CGB only. On DMG hardware, all sprites are rendered regardless of OAM position order (the DMG uses a fixed priority based on X position / OAM index). The flicker and Y-sort modes are still useful on DMG for the OAM-index-based priority, but the visual effect of moving actors up and down the list is DMG-limited.

![image](https://github.com/user-attachments/assets/5a426393-584e-4f23-b652-16cc829d96bb)

Example of a walking-on-grass effect achieved by placing the grass actor above the player in the active list:

https://github.com/user-attachments/assets/646fd337-f69d-41b3-a1ad-e2ace08a900f

Example of vertical Y-sorting keeping foreground actors rendering in front of background actors based on their Y position:

https://github.com/user-attachments/assets/1b4fded2-95ee-4c1a-ab5a-b3616659223c

---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [How to Use](#how-to-use)
4. [Technicalities and Restrictions](#technicalities-and-restrictions)
5. [Engine Fields and Settings](#engine-fields-and-settings)
6. [Events Reference](#events-reference)
7. [Inner Workings](#inner-workings)

---

## Concepts

### The Active Actor Linked List

GB Studio manages actors as a doubly-linked list of active actors (`actors_active_head` → … → `actors_active_tail`). Each frame, `actors_render` iterates this list **tail-to-head** and calls `move_metasprite` for each actor, assigning OAM hardware sprite slots in order. Actors encountered first (near the tail) receive the lowest OAM indices; actors near the head receive the highest.

On CGB hardware, a lower OAM index means the sprite is drawn **on top** when two sprites overlap on the same pixel. Therefore:

- **Tail of the list** = rendered last in the iteration = **lowest OAM index** = drawn **on top** of other actors.
- **Head of the list** = rendered first in the iteration = **highest OAM index** = drawn **behind** other actors.

The "active index" in this plugin uses the same direction: **index 0 = head of the list** (rendered behind), **index N = tail of the list** (rendered on top).

### Flicker Mode

The Game Boy hardware can display at most 10 sprites per scanline. When more than 10 actors overlap on the same row of pixels, the lowest-indexed OAM entries take priority and the rest are invisible on that scanline. Flicker mode rotates which actors are at the start of the OAM table each frame, distributing the clipping across all actors over time so that no single actor is permanently invisible. This is the classic Game Boy flicker technique.

### Y-Sort Mode

In many top-down games, actors that are lower on the screen (higher Y position) should appear in front of actors that are higher on the screen (lower Y position). Y-sort mode automatically reorders the active list by Y position every 8 frames so that actors with higher Y values are nearer the tail (rendered on top).

---

## Project Setup

1. Copy the plugin folder into your GB Studio project's `plugins/` directory.
2. Configure the engine settings (see [Engine Fields and Settings](#engine-fields-and-settings)) to enable or disable the Flicker and Y-Sort features as needed.

---

## How to Use

### Controlling Render Order with Active Index

Use **Set actor active index** to control where in the rendering order a specific actor sits:

- Set to `0` → actor moves to the head of the list → rendered behind all other actors.
- Set to a large number (e.g. `255`) → actor moves to the tail → rendered in front of all other actors.
- Set to the index of a specific other actor (read with **Get actor active index**) → actor is inserted just before that actor.

**Example — render player behind a specific NPC:**
1. Use **Get actor active index** on the NPC, store the result in a variable.
2. Use **Set actor active index** on the player with the value from that variable.
   The player is now inserted before the NPC in the list, so the NPC renders on top.

### Automatic Sort Mode

Use **Set actor sort mode** to switch the mode that runs every frame at the end of `actors_render`:

- **None (0)** — no automatic sorting. Manual index manipulation is preserved.
- **Flicker (1)** — rotates the actor at the tail to the head each frame. Distributes OAM priority clipping over time.
- **Sort Vertically (2)** — runs `sort_actors_by_ypos` every 8 frames. Actors lower on screen render on top.

### Manual One-Shot Y Sort

Use **Sort actors vertically** to trigger a single Y-sort without enabling the automatic sort mode. Useful for sorting once when a scene starts.

---

## Technicalities and Restrictions

### CGB Only for Visual Render-Order Effects

On DMG, sprite rendering order is determined by OAM index and X position simultaneously. Moving actors in the list still changes their OAM index, but the visual effect of "in front / behind" differs from CGB. Flicker is still effective on DMG for scanline overflow distribution.

### Set Actor Active Index Only Works on Active Actors

If the target actor is currently inactive (not on screen or explicitly deactivated), `set_actor_active_index` does nothing. The actor must be active for the repositioning to take effect.

### Get Actor Active Index Only Works on Active Actors

If the target actor is inactive, `get_actor_active_index` does not write to the output variable. The variable retains its previous value.

### Y-Sort Runs Every 8 Frames

When sort mode is **Sort Vertically**, `sort_actors_by_ypos` is called only when `game_time & 0x07 == 0` (i.e. every 8 frames). This reduces CPU cost at the expense of a maximum 8-frame delay before sort order catches up to position changes.

### Flicker Changes `tmp_iterator_offset`

Flicker mode increments `tmp_iterator_offset` each frame. This variable is also used by `actors_update` to spread actor visibility checks across frames. The interaction is intentional — it ensures that different actors are on-screen-checked on different frames, distributing CPU load.

### Modified Engine File

The plugin replaces `engine/src/core/actor.c` to add the `actor_sort_mode` variable, the sort mode switch in `actors_render`, and the `actor_active_index.h` include.

---

## Engine Fields and Settings

These settings appear in GB Studio under **Settings → Engine → Change Actor Render Order**.

### Enable Feature: Flicker

**Key:** `ENABLE_FLICKER_FEATURE`  
**Type:** Checkbox  
**Default:** Enabled

When checked, the flicker code path is compiled into the engine. When unchecked, all flicker-related code is removed at compile time via `#ifdef`. Disable this if you are not using flicker to save ROM space.

### Exclude Player Actor from Flicker

**Key:** `EXCLUDE_PLAYER_FROM_FLICKER`  
**Type:** Checkbox  
**Default:** Disabled  
**Condition:** Only visible when **Enable Feature: Flicker** is checked.

When checked, the flicker rotation skips the player actor — the player always stays at its current list position and is never rotated to the head. This prevents the player sprite from flickering when many actors are on screen. Other actors still rotate normally.

### Enable Feature: Vertical Sort

**Key:** `ENABLE_SORTY_FEATURE`  
**Type:** Checkbox  
**Default:** Enabled

When checked, the vertical sort code path is compiled in. When unchecked, `sort_actors_by_ypos` and the Y-sort branch are removed at compile time. Disable this if you are not using Y-sort to save ROM space.

---

## Events Reference

### Get Actor Active Index

**Event ID:** `EVENT_GET_ACTOR_ACTIVE_INDEX`  
**Group:** Actor

Reads the current position of an actor in the active actor list and stores it in a variable. Index 0 = head of the list (behind others); higher indices = nearer the tail (in front).

| Field | Type | Default | Description |
|---|---|---|---|
| Actor | Actor picker | Self | The actor whose active list position to read. |
| Variable | Variable | Last variable | Receives the active index (0 = head). Only written if the actor is currently active. |

---

### Set Actor Active Index

**Event ID:** `EVENT_SET_ACTOR_ACTIVE_INDEX`  
**Group:** Actor

Repositions an actor within the active actor list at a given index. If the index exceeds the current list length, the actor is placed at the tail (rendered on top).

| Field | Type | Default | Description |
|---|---|---|---|
| Actor | Actor picker | Self | The actor to reposition. Must currently be active. |
| Active index | Value expression | 0 | Target position in the active list. 0 = head (behind all), large value = tail (in front of all). |

---

### Sort Actors Vertically

**Event ID:** `EVENT_SORT_ACTORS_VERTICALY`  
**Group:** Actor

Immediately performs a one-shot insertion sort of all active actors by their Y position (ascending — lower Y = nearer head = rendered behind; higher Y = nearer tail = rendered in front). Has no fields.

---

### Set Actor Sort Mode

**Event ID:** `EVENT_SET_ACTOR_SORT_MODE`  
**Group:** Actor

Sets the global automatic sort mode that runs at the end of every `actors_render` call. This persists until changed by another **Set actor sort mode** event.

| Field | Type | Default | Description |
|---|---|---|---|
| Actor Sort Mode | Select | None | `None` (0) — no automatic reordering. `Flicker` (1) — rotates tail actor to head each frame. `Sort Vertically` (2) — re-sorts by Y every 8 frames. |

---

## Inner Workings

### Active Actor Linked List Direction

```
actors_active_head ←→ actor ←→ actor ←→ … ←→ actors_active_tail
       index 0                                     index N
   (behind others)                              (in front)
```

`actors_render` iterates from `actors_active_tail` toward `actors_active_head`, so the tail actor occupies OAM slot 0 (highest CGB priority). The head actor gets the highest OAM slot number (rendered behind when overlapping).

### `set_actor_active_index`

The function first removes the actor from its current list position using `DL_REMOVE_ITEM`, updates the tail pointer if needed, then walks the list from the head to find the insertion point at the requested index:

```c
actor_t * target_actor = actors_active_head;
while (active_idx && target_actor->next) {
    active_idx--;
    target_actor = target_actor->next;
}
if (active_idx) {           // walked past the end → insert at tail
    target_actor->next = actor;
    actor->prev = target_actor;
    actor->next = 0;
    actors_active_tail = actor;
} else if (target_actor->prev) {  // insert in the middle
    actor->prev = target_actor->prev;
    target_actor->prev->next = actor;
    actor->next = target_actor;
    target_actor->prev = actor;
} else {                    // insert at head
    actor->prev = 0;
    actors_active_head = actor;
    actor->next = target_actor;
    target_actor->prev = actor;
}
```

The loop counts down `active_idx` while advancing through the list. If the index runs out before reaching the end, the actor is inserted just before the current `target_actor`. If the counter is still non-zero when `target_actor->next` is null, the requested index was past the end and the actor is appended at the tail.

### `get_actor_active_index`

Walks from `actors_active_head`, counting steps until it finds the target actor, then writes the count to `script_memory`:

```c
actor_t * target_actor = actors_active_head;
UBYTE active_idx = 0;
while (target_actor) {
    if (target_actor == actor) break;
    active_idx++;
    target_actor = target_actor->next;
}
script_memory[*(int16_t*)VM_REF_TO_PTR(FN_ARG1)] = active_idx;
```

### Flicker Mode in `actors_render`

Without `EXCLUDE_PLAYER_FROM_FLICKER`, the tail actor is moved to the head after rendering:

```c
actor = actors_active_tail;
if (actor != actors_active_head) {
    actor->next = actors_active_head;
    actors_active_head->prev = actor;
    actors_active_head = actor;
    actor->prev->next = 0;
    actors_active_tail = actor->prev;
    actor->prev = 0;
    tmp_iterator_offset++;
}
```

Each frame, the actor that was rendered first (on top) is moved to the back of the list, so a different actor gets the "on top" position. Over enough frames every actor gets equal access to the low OAM indices.

With `EXCLUDE_PLAYER_FROM_FLICKER`, the player is explicitly skipped: the actor just before the player in the list is moved to the head instead.

### Y-Sort: Insertion Sort on the Linked List

`sort_actors_by_ypos` performs an in-place insertion sort on the active linked list without allocating any extra memory. It maintains a sorted sub-list (`actor_a`) and inserts each node from the unsorted remainder (`actor_b`) into the correct position:

```
For each actor_b in the unsorted part:
  Walk actor_a to find the first node with pos.y > actor_b->pos.y
  Insert actor_b before that node (or at the tail if none found)
```

After the sort, actors with **smaller Y** (higher on screen) are at the **head** (rendered behind) and actors with **larger Y** (lower on screen) are at the **tail** (rendered in front). The sort runs every 8 frames (`IS_FRAME_8`).

### `actor_sort_mode` Variable

`actor_sort_mode` is a `UBYTE` defined in `actor.c` and directly written by the **Set actor sort mode** event via `_setConstMemUInt8('actor_sort_mode', value)`. The three `#define` constants map modes to integer values:

```c
#define ACTOR_SORT_MODE_NONE    0
#define ACTOR_SORT_MODE_FLICKER 1
#define ACTOR_SORT_MODE_YSORT   2
```

The `switch` statement at the end of `actors_render` reads this variable every frame and executes the appropriate path. Both the `FLICKER` and `YSORT` cases are wrapped in `#ifdef` guards so they compile away entirely when the corresponding engine feature is disabled.

