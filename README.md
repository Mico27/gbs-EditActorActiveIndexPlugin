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
3. [Size Limits and Restrictions](#size-limits-and-restrictions)
4. [Engine Settings](#engine-settings)
5. [Events Reference](#events-reference)
6. [Memory Footprint](#memory-footprint)

---

## Concepts

### The Active Actor Linked List

GB Studio keeps its active actors in an ordered list. Each frame the engine walks that list from the **tail to the head**, assigning OAM hardware sprite slots as it goes. Actors encountered first (near the tail) receive the lowest OAM indices; actors near the head receive the highest.

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
2. Configure the engine settings (see [Engine Settings](#engine-settings)) to enable or disable the Flicker and Y-Sort features as needed.

---

### How to Use

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

Use **Set actor sort mode** to switch the mode that runs at the end of every frame:

- **None (0)** — no automatic sorting. Manual index manipulation is preserved.
- **Flicker (1)** — rotates the actor at the tail to the head each frame. Distributes OAM priority clipping over time.
- **Sort Vertically (2)** — re-sorts by Y position every 8 frames. Actors lower on screen render on top.

### Manual One-Shot Y Sort

Use **Sort actors vertically** to trigger a single Y-sort without enabling the automatic sort mode. Useful for sorting once when a scene starts.

---

## Size Limits and Restrictions

### CGB Only for Visual Render-Order Effects

On DMG, sprite rendering order is determined by OAM index and X position simultaneously. Moving actors in the list still changes their OAM index, but the visual effect of "in front / behind" differs from CGB. Flicker is still effective on DMG for scanline overflow distribution.

### Set Actor Active Index Only Works on Active Actors

If the target actor is currently inactive — off screen, or explicitly deactivated — the event does nothing. The actor must be active for the repositioning to take effect.

### Get Actor Active Index Only Works on Active Actors

If the target actor is inactive, the event does not write to the output variable, which keeps its previous value.

### Y-Sort Runs Every 8 Frames

When sort mode is **Sort Vertically**, the sort runs only once every 8 frames. This reduces CPU cost at the expense of a maximum 8-frame delay before sort order catches up to position changes.

### Flicker Shares the Engine Frame Counter

Flicker mode advances the same per-frame counter the engine uses to spread actor visibility checks across frames. The interaction is intentional: it keeps different actors being checked on different frames, distributing CPU load.

### Modified Engine File

The plugin replaces the stock actor rendering file to add the sort-mode switch. Another plugin that patches the same file needs a merged build or a matching compatibility variant.

---

## Engine Settings

These settings appear in GB Studio under **Settings → Engine → Change Actor Render Order**.

| Setting | Default | Description |
|---|---|---|
| **Enable Feature: Flicker** | Enabled | Compiles the flicker code into the engine. Turn it off to save ROM if you do not use flicker. |
| **Exclude Player Actor from Flicker** | Disabled | Skips the player in the flicker rotation, so the player sprite never flickers when many actors are on screen. Other actors still rotate normally. Only shown while Flicker is enabled. |
| **Enable Feature: Vertical Sort** | Enabled | Compiles the vertical sort code into the engine. Turn it off to save ROM if you do not use Y-sort. |

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

### Get Actor Active Index By Index

**Event ID:** `EVENT_GET_ACTOR_ACTIVE_INDEX_BY_INDEX`  
**Group:** Actor

Same as **Get Actor Active Index**, but the target actor is given as a raw actor index (script value) instead of an actor picker — for addressing actors dynamically (e.g. pool actors).

| Field | Type | Default | Description |
|---|---|---|---|
| Actor Index | Value expression | 0 | Index of the actor whose active list position to read. |
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

### Set Actor Active Index By Index

**Event ID:** `EVENT_SET_ACTOR_ACTIVE_INDEX_BY_INDEX`  
**Group:** Actor

Same as **Set Actor Active Index**, but the actor to reposition is given as a raw actor index (script value) instead of an actor picker — for addressing actors dynamically (e.g. pool actors).

| Field | Type | Default | Description |
|---|---|---|---|
| Actor Index | Value expression | 0 | Index of the actor to reposition. Must currently be active. |
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

<!-- SETTINGCOST:BEGIN -->
### What each engine setting costs

Every setting here changes what gets compiled. Figures are what you **get back by
turning the setting off**; rows marked *off by default* show what turning it **on**
costs instead, and sliders show the cost per step. A dash means that budget does not
move.

| Setting | Bank 0 | WRAM | Banked ROM |
|---|---|---|---|
| Enable Feature: Flicker | **142 B** | — | — |
| Exclude player actor from flicker *(off by default — cost of turning it on)* | +47 B | — | — |
| Enable Feature: Vertical Sort | **27 B** | **8 B** | **420 B** |

Turning off every on-by-default switch above frees **169 B** of bank 0, **8 B** of WRAM, **420 B** of banked ROM — the full
span between this plugin at its fullest and stripped to nothing. Treat it as a
ceiling rather than a recipe: you keep whatever your game actually uses.

- **Exclude player actor from flicker** only applies when *Enable Feature: Flicker* is enabled.

<details><summary>How these were measured</summary>

GB Studio 4.3.0-e1. This plugin's `engine/src/**/*.c` was compiled with the
toolchain and flags GB Studio itself uses (`lcc -msm83:gb -Wf--max-allocs-per-node 3000
-DHUGE_TRACKER -DRUMBLE_ENABLE=0x08u`) against a merged include tree, and the SDCC object
files' area records were read: `_HOME` is bank 0, `_DATA`/`_INITIALIZED`/`_BSS` are WRAM,
and `_CODE*`/`_CONST`/`_LIT`/`_INITIALIZER` are banked ROM.

Two caveats. Only this plugin's own engine sources are measured, so a setting that also
changes a struct shared with stock engine files can move a few more bytes in files the
plugin does not ship. And each setting is toggled on its own: a handful measure slightly
*negative* because enabling their code lets the compiler drop a fallback path elsewhere,
and settings that gate other settings only show their own contribution.

</details>
<!-- SETTINGCOST:END -->

## Memory Footprint

Measured against the stock GB Studio **4.3.0-e1** engine (per-file SDCC compile with GB Studio's build flags, default engine settings). Values are the plugin's *delta* versus the stock engine; DMG build, with CGB noted where it differs. ROM cost lands in banked ROM (GB Studio's autobanker spreads it across switchable banks); using the plugin's events additionally compiles a few bytes of GBVM script per call into your project's script banks.

| | Cost |
|---|---|
| WRAM | +8 bytes |
| ROM | +1,162 bytes |

- **WRAM:** 8 bytes of bookkeeping.
- **Engine WRAM headroom:** the stock GB Studio 4.3.0 engine leaves about **854 bytes** of WRAM free (usable engine WRAM is 7,776 bytes at 0xC0A0–0xDF00; the stock engine uses 6,922 bytes). With this plugin installed roughly **846 bytes** remain. This figure does not depend on how many global variables your project defines: the script memory array has a fixed size of VM_HEAP_SIZE + (VM_MAX_CONTEXTS × VM_CONTEXT_STACK_SIZE) words — 768 + 16 × 64 = 1,792 words (3,584 bytes) with stock engine settings.
- **SRAM:** not used.

---

<!-- BANK0:BEGIN -->
## Bank 0 (HOME) Usage

Bank 0 is the 16 KB non-switchable ROM bank that the GB Studio engine core,
the interrupt handlers and the GBDK runtime all share. Banked ROM is cheap
(add another bank), bank 0 is not, so it is usually the first thing a project
runs out of.

| | Bytes |
|---|---|
| Bank 0 used by this plugin | **-20** |
| Bank 0 free with this plugin installed | **1,471** of 16,384 (91% used) |

**This plugin gives bank 0 space back.** Its replacements for stock engine
files compile smaller than the originals, freeing 20 bytes.

| Module | This plugin | Stock engine | Bank 0 cost |
|---|---|---|---|
| `actor.c` | 851 | 871 | -20 |

Modules that replace or patch a stock engine file only cost the *difference*:
the stock version's bank 0 bytes were being spent anyway.

<details><summary>How this was measured</summary>

GB Studio 4.3.2, DMG target, default engine settings. Each module's bank 0
contribution is the `A _HOME size` record that SDCC writes into its `.rel`
object, summed over the engine sources this plugin provides. Stock sizes come
from building projects whose only plugin ships no engine C, so every module in
them is the untouched engine; two such builds were compared and agreed on all
73 shared modules.

The "free" figure is a stock project with this plugin and nothing else. Your
own number will differ: other plugins, and any engine settings that change what
the core compiles, move it independently of this plugin.

</details>
<!-- BANK0:END -->

## Changelog

Grouped by the date each change was merged into the official
[gb-studio-plugins](https://github.com/gb-studio-dev/gb-studio-plugins) repository.

Only bug fixes, new features and feature changes are listed. Engine version
bumps, patch regeneration, packaging fixes and documentation edits are omitted.

### 2026-06-28

- Added ContinuousScenePlugin compatibility.

### 2026-06-14

- Added custom script parameter / stack support to the events.

### 2025-10-29

- Fixed Y sorting.

### 2025-08-07

- Fixed a desynchronised `actors_active_tail` when actors are deactivated.

### 2025-04-02

- Initial release.
- Added a "sort actors vertically" event.
- Inactive actors are now ignored when setting an active index.
- Fixed the actor overlapping functions.
