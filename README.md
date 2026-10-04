# Smooth Desktop Icons Auto-Hide

Automatically hides desktop icons when you're not using the desktop and smoothly fades them back in when you are.

![Demo](https://raw.githubusercontent.com/HeyOkay/SmoothDesktopAutoHide/main/assets/demo.gif)

*Dragging a file onto the desktop reveals the icons with a smooth fade-in; they fade out once the desktop loses focus.*

## Features

- **Auto-hide.** Icons fade out a few seconds after you leave the desktop. The countdown is paused while the desktop is the active surface, so icons never disappear while you're looking at them.
- **Click to show.** A single click on empty desktop reveals the icons. Mouse movement alone never does.
- **Double-click to hide.** Double-click empty desktop to hide the icons right away.
- **Drag & drop.** Dragging a file onto the desktop reveals the icons before you drop it.
- **Win+D and Show desktop.** Showing the desktop with `Win+D`, the taskbar's **Show desktop** button, or by minimizing the last window reveals the icons; leaving it starts a fresh countdown.
- **Smooth fade.** Configurable duration, no wallpaper dimming, works on all Virtual Desktops.
- **Safe while hidden.** Hidden icons can't be clicked and are deselected. Pressing a key on the desktop (including Delete, F2, Ctrl+A) reveals them instead of acting on invisible files.

## How this differs from similar mods

- **[ZenDesktop: Desktop Icon Toggle and Auto-Hide](https://windhawk.net/mods/zen-desktop-toggle-icons)** toggles icons by double-click and hides them after N seconds without any input anywhere in the system (`GetLastInputInfo()`), restoring them on any input. This mod instead ties auto-hide to the desktop itself: the countdown starts when the desktop stops being the active surface and is paused while you're on it. It also adds a smooth fade, reveals the icons with a *single* click on empty desktop, and reveals them when a file is dragged onto the desktop.
- **[Desktop Icon Section Auto-Hide & Fluent Hover Reveal](https://windhawk.net/mods/desktop-icon-section-autohide)** reveals icons on hover and offers modes, per-app pinning and click-and-hold peek. This mod deliberately does **not** react to mouse movement: icons appear only on an explicit action (click, drag, desktop activation) and stay while the desktop is in use. It has no modes or whitelist and only three settings.

## How it works

The desktop icon list is never hidden or made transparent as a window. Instead, the mod blends the icons, labels and selection highlights onto the wallpaper with the current opacity while Explorer paints them. Showing and hiding react to events (clicks, drag & drop, the desktop becoming the active window) rather than constant polling, so the mod does nothing while idle. When the mod is disabled, everything is restored to Explorer's normal state.

## Settings

- **Enable Auto-hide** — hide icons automatically after leaving the desktop. When off, icons start visible and are hidden only by double-clicking empty desktop.
- **Hide after seconds** — delay before icons are hidden (1–60, default 5).
- **Animation duration (ms)** — fade duration (50–1000, default 250).

## Credits

- The paint-time opacity technique (blending icon and label drawing instead of making the ListView layered) follows [desktop-icon-section-autohide](https://github.com/ramensoftware/windhawk-mods/blob/main/mods/desktop-icon-section-autohide.wh.cpp) by Piyush Das, which builds on [desktop-icons-transparency](https://github.com/ramensoftware/windhawk-mods/blob/main/mods/desktop-icons-transparency.wh.cpp) by zed712969-crypto.
- Inspired by [ZenDesktop: Desktop Icon Toggle and Auto-Hide](https://github.com/ramensoftware/windhawk-mods/blob/main/mods/zen-desktop-toggle-icons.wh.cpp) by Lanbo, including its desktop window discovery approach (CreateWindowExW hook, `Progman`/`WorkerW` enumeration, subclassing `SHELLDLL_DefView` and `SysListView32`).
