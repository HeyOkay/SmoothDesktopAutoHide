# Smooth Desktop Icons Auto-Hide

**Smooth Desktop Icons Auto-Hide** is a Windhawk mod for Windows Explorer that automatically hides desktop icons when they are not needed and smoothly reveals them when the desktop is used.

## Features

- **Automatic icon hiding**
  - Desktop icons are hidden after a configurable period of inactivity.
  - Default timeout: **5 seconds**.
  - Configurable from **1 to 60 seconds**.

- **Single-click to show**
  - Clicking an empty area of the desktop reveals the icons.
  - Mouse movement alone does not reveal them.

- **Double-click to hide**
  - Double-clicking an empty area of the desktop hides the icons.

- **Smooth fade animation**
  - Icons appear and disappear using a smooth alpha fade.
  - Animation duration is configurable from **50 to 1000 ms**.
  - Default: **250 ms**.

- **Drag & Drop support**
  - Dragging a file toward the desktop reveals the icons before the drop.
  - The icons remain available while the drag is active.

- **Desktop activity awareness**
  - After the user interacts with the desktop, auto-hide is paused while the desktop remains the active surface.
  - When the desktop becomes inactive, a new full auto-hide countdown starts.

- **Win+D support**
  - The mod tracks the standard Windows `Win+D` desktop shortcut and keeps icon visibility and auto-hide state synchronized with desktop activation.


## How It Works

Windows Explorer displays desktop icons through a `SysListView32` window located inside `SHELLDLL_DefView`.

The mod discovers the desktop ShellView and its ListView, then subclasses these existing Explorer windows to monitor desktop interaction and control icon visibility.

```text
Windows Explorer
       │
       ▼
SHELLDLL_DefView
       │
       ▼
SysListView32
       │
       ├── Desktop icon visibility
       ├── Mouse interaction
       ├── Double-click handling
       └── Drag & Drop detection
```

### Desktop window discovery

The mod monitors creation of Explorer windows and looks for:

- `SHELLDLL_DefView` — the desktop ShellView.
- `SysListView32` — the ListView that contains the desktop icons.

Existing desktop windows are also discovered when the mod starts.

The mod subclasses both relevant windows so that it can process their messages without replacing or recreating the desktop ListView.

### Showing and hiding icons

The mod directly controls the existing desktop `SysListView32` with:

```text
ShowWindow(list, SW_SHOW)
ShowWindow(list, SW_HIDE)
```

The existing ListView remains in place. The mod does not use Explorer's internal desktop-icon toggle command for normal show/hide operations.

Keeping the same ListView preserves its existing icon state and avoids an unnecessary Explorer desktop-icon refresh during visibility changes.

### Fade animation

The fade effect is implemented on the existing ListView using `WS_EX_LAYERED` and `SetLayeredWindowAttributes`.

When showing:

```text
ListView hidden
      │
      ▼
ShowWindow(SW_SHOW)
      │
      ▼
Alpha = 0
      │
      ▼
Gradually increase alpha
      │
      ▼
Alpha = 255
      │
      ▼
Fully visible
```

When hiding, the same process runs in reverse:

```text
Alpha = 255
      │
      ▼
Gradually decrease alpha
      │
      ▼
Alpha = 0
      │
      ▼
ShowWindow(SW_HIDE)
```

The animation uses smoothstep easing:

```text
t² × (3 − 2t)
```

This produces a smooth acceleration/deceleration instead of a linear opacity change.

### Icon state machine

The mod tracks four icon states:

```text
Hidden
   │
   ▼
Showing
   │
   ▼
Visible
   │
   ▼
Hiding
   │
   ▼
Hidden
```

A new show/hide request can interrupt the current transition, allowing the animation to move smoothly toward the new target state.

### Click handling

When the icons are hidden, a left-click on the empty desktop is handled by `SHELLDLL_DefView` and reveals the ListView.

When the icons are visible, `SysListView32` receives the mouse interaction. The mod performs a ListView hit test so that double-click behavior is applied only to empty desktop space and not to an actual icon.

A real desktop click also marks the desktop as actively used and temporarily cancels the auto-hide countdown.

### Auto-hide logic

Auto-hide is state-aware rather than an unconditional timer.

```text
User activates desktop
        │
        ▼
   Icons visible
        │
        ▼
 Auto-hide paused
        │
        ▼
Desktop becomes inactive
        │
        ▼
Start fresh countdown
        │
        ▼
     Timeout
        │
        ▼
   Fade-out animation
        │
        ▼
   Icons hidden
```

The mod checks the actual foreground/root window relationship to determine whether the desktop is still the active surface.

This prevents the icons from disappearing while the user is actively working with the desktop.

### Drag detection

The mod does not install a global low-level mouse hook.

Instead, a lightweight polling timer checks the physical left-button state and the window under the cursor. When the left button is held and the cursor enters the desktop surface, the mod treats this as a drag/desktop interaction and reveals the icons.

This allows a drag that started in another Explorer window to reveal the desktop icons before the file is dropped.

Mouse movement with no button held does not reveal the icons.


### Win+D handling

`Win+D` is handled through lightweight key-state polling because Explorer does not always send a reliable mouse or focus message to the desktop ShellView when the shortcut changes the active surface.

The mod uses this state to distinguish:

```text
Win+D
  │
  ├── Application → Desktop
  │       └── reveal icons / pause auto-hide
  │
  └── Desktop → Application
          └── start a fresh auto-hide countdown
```

### Timers

The implementation uses separate timers for different jobs:

- **Auto-hide timer** — controls the user-configured inactivity timeout.
- **Interaction/drag polling timer** — runs at a short interval to detect drag entry, reconcile desktop activity, recover menu/button state, and track `Win+D`.
- **Animation timer** — updates the ListView alpha during fade transitions.

These responsibilities are kept separate so that interaction polling does not reset the user's auto-hide countdown.

## Cleanup and Safety

When the mod is unloaded or disabled, it restores the desktop ListView to a normal Explorer state.

Cleanup:

1. Stops active timers.
2. Restores full opacity.
3. Removes the temporary layered-window style.
4. Makes the desktop ListView visible if it was hidden by the mod.
5. Removes the ListView and ShellView subclasses.
6. Resets the internal state.

This prevents the mod from leaving desktop icons permanently hidden after the mod is disabled or Explorer is restarted.

## Settings

### Enable Auto-hide

Enables or disables automatic hiding.

### Hide after seconds

Time of inactivity before the icons are hidden.

- **Range:** 1–60 seconds
- **Default:** 5 seconds

### Animation duration (ms)

Duration of the fade animation.

- **Range:** 50–1000 ms
- **Default:** 250 ms

## Known Issue

### Virtual Desktops

When switching to a newly created Windows Virtual Desktop, a **black background/visual artifact** may occasionally appear behind the desktop icons while the icons are visible.

This is related to the interaction between the layered `SysListView32` window and Explorer's rendering/composition behavior during Virtual Desktop transitions.

The issue does not affect the core functionality of the mod and is currently considered a known issue.
