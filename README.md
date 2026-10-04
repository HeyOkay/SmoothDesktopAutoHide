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

- **Safe while hidden**
  - Hidden icons can't be clicked, and they are deselected when they fade out.
  - Pressing a key on the desktop while the icons are hidden (including shortcuts such as Delete, F2, Ctrl+A) reveals them instead of acting on invisible files.

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

- **Win+D and "Show desktop" support**
  - Activating the desktop with `Win+D`, the taskbar's **Show desktop** button, or by minimizing the last window reveals the icons; leaving it the same way starts a fresh countdown.

## How this differs from similar mods

- **[ZenDesktop: Desktop Icon Toggle and Auto-Hide](https://windhawk.net/mods/zen-desktop-toggle-icons)** toggles icons by double-click and hides them after N seconds without any input anywhere in the system (`GetLastInputInfo()`), restoring them on any input. This mod instead ties auto-hide to the desktop itself: the countdown starts when the desktop stops being the active surface and is paused while you're on it. It also adds a smooth fade, reveals the icons with a *single* click on empty desktop, and reveals them when a file is dragged onto the desktop.
- **[Desktop Icon Section Auto-Hide & Fluent Hover Reveal](https://windhawk.net/mods/desktop-icon-section-autohide)** reveals icons on hover and offers modes, per-app pinning and click-and-hold peek. This mod deliberately does **not** react to mouse movement: icons appear only on an explicit action (click, drag, desktop activation) and stay while the desktop is in use. It has no modes or whitelist and only three settings.

## Credits

- The paint-time opacity technique (blending icon and label drawing instead of making the ListView layered) follows [desktop-icon-section-autohide](https://github.com/ramensoftware/windhawk-mods/blob/main/mods/desktop-icon-section-autohide.wh.cpp) by Piyush Das, which builds on [desktop-icons-transparency](https://github.com/ramensoftware/windhawk-mods/blob/main/mods/desktop-icons-transparency.wh.cpp) by zed712969-crypto.
- Inspired by [ZenDesktop: Desktop Icon Toggle and Auto-Hide](https://github.com/ramensoftware/windhawk-mods/blob/main/mods/zen-desktop-toggle-icons.wh.cpp) by Lanbo, including its desktop window discovery approach (CreateWindowExW hook, `Progman`/`WorkerW` enumeration, subclassing `SHELLDLL_DefView` and `SysListView32`).


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

- `SHELLDLL_DefView` — the desktop ShellView, accepted only when its parent is `Progman` or `WorkerW`. File Explorer folder views and dialogs are never touched.
- `SysListView32` — the ListView that contains the desktop icons.

Existing desktop windows are also discovered when the mod starts. All mod state is created and used on the desktop window's own thread.

The mod subclasses both relevant windows so that it can process their messages without replacing or recreating the desktop ListView.

### Showing and hiding icons

The desktop `SysListView32` is **never hidden with `ShowWindow` and never made layered**.

Instead the mod controls how the ListView *paints*. While the ListView handles `WM_PAINT`, the calls Explorer uses to draw the desktop items are intercepted:

- `ImageList_DrawIndirect` — icons and overlays (drawn with `ILS_ALPHA`);
- `GdiAlphaBlend` — thumbnails, label shadows and other pre-blended bitmaps;
- `DrawShadowText` / `ExtTextOutW` — icon labels;
- `DrawThemeBackground` — selection and hover highlights.

Each of them is blended onto the already painted wallpaper with the current opacity. At opacity 0 nothing is drawn, so the ListView is fully transparent.

While the icons are hidden, the ListView returns `HTTRANSPARENT` from `WM_NCHITTEST` and ignores keyboard input, so invisible icons cannot be clicked, opened or selected; clicks go straight to `SHELLDLL_DefView` exactly as if the ListView were hidden.

Because there is no layered redirection surface and no show/hide transition, Explorer has nothing it can expose as a black background while Virtual Desktops are created or switched.

### Fade animation

Opacity uses the full 0–255 range. Progress is measured with `QueryPerformanceCounter`, frames are driven at ~8 ms with 1 ms timer resolution requested only while a fade is running, and every frame repaints the ListView once.

An interrupted fade continues from the current opacity, and its duration is scaled to the remaining distance so the perceived speed stays constant.

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

The mod does not poll the mouse and does not install a low-level mouse hook.

The desktop registers an OLE `IDropTarget`. Every drag over the desktop, from any application, calls that object's `DragEnter`, `DragLeave` and `Drop` on the desktop thread. The mod hooks these three methods (reacting only to the desktop's own drop target) and reveals the icons the moment a drag enters the desktop. Auto-hide stays paused until the drag leaves or the item is dropped.

### Desktop activation, Win+D and Show desktop

The mod installs an out-of-context `EVENT_SYSTEM_FOREGROUND` WinEvent hook on the desktop thread. When the foreground window changes:

```text
Application → Desktop   (Win+D, Show desktop button, last window minimized, click)
        └── reveal icons / pause auto-hide

Desktop → Application   (second Win+D / Show desktop, switching windows)
        └── start a fresh auto-hide countdown
```

`Win+D` and the taskbar's **Show desktop** button go through exactly the same path: both make the desktop the foreground surface, so the mod doesn't need to know which one was used. No keyboard state is polled.

### Timers

- **Auto-hide timer** — the user-configured inactivity timeout.
- **Animation timer** — runs only during a fade.
- **Menu check timer** — runs only while a desktop context menu is open (classic or Windows 11), to notice the menu closing when Explorer doesn't deliver `WM_EXITMENULOOP`.

When the icons are idle (hidden or visible) and no menu is open, the mod runs no periodic timers at all.

## Cleanup and Safety

When the mod is unloaded or disabled, it restores the desktop ListView to a normal Explorer state.

Cleanup:

1. Stops active timers and releases the timer resolution request.
2. Removes the foreground WinEvent hook.
3. Restores full opacity and repaints the icons.
4. Removes the ListView and ShellView subclasses.
5. Releases the offscreen drawing buffer.

Restoration runs on the desktop window's own thread.

This prevents the mod from leaving desktop icons permanently hidden after the mod is disabled or Explorer is restarted.

## Settings

### Enable Auto-hide

Enables or disables automatic hiding. When off, icons start visible and are hidden only by double-clicking empty desktop.

### Hide after seconds

Time of inactivity before the icons are hidden.

- **Range:** 1–60 seconds
- **Default:** 5 seconds

### Animation duration (ms)

Duration of the fade animation.

- **Range:** 50–1000 ms
- **Default:** 250 ms

