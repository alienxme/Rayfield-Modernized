https://docs.sirius.menu/rayfield
https://www.rayfield.gg/
https://developer.sirius.menu/

Welcome to an unofficial fork of the Rayfield respitory this fork addresses the bugs below that the developers of rayfield was too lazy to fix:

**Bug 1 — Tab contents disappearing:** In `setElementsVisible(show)`, child visibility is set with `child.Visible = show` — this forces ALL child frames/labels/etc. invisible across ALL tabs, not just the active one. When you switch tabs via `UIPageLayout:JumpTo()`, the new tab's children were already set `Visible = false` by the last hide/unhide cycle and never re-shown (because `setElementsVisible(true)` sets them all, but it races against tween completion and the `Visible` toggle applies globally). The fix: only touch the *current* tab's children, and use transparency tweens consistently rather than toggling `Visible`.

**Bug 2 (Toggle/Slider ghost):** `setElementsVisible(false)` tweens the parent element frame's `BackgroundTransparency` to 1, but `Toggle.Switch` and `Slider.Main` are **child Frames** inside the element. They have their own `BackgroundColor3` but no transparency tween is ever applied to them during hide. The old `child.Visible = show` masked this completely — we removed it correctly, but now need to add proper transparency tweens for these sub-frames.

**Bug 3 (theme residue):** `ChangeTheme()` only recolors `Element.BackgroundColor3` and `UIStroke` for top-level element frames. It misses `Toggle.Switch`, `Slider.Main`, `Slider.Main.Progress`, and all the indicator sub-colors. So when config loads and calls `ModifyTheme` *after* the default theme already painted the widgets with Amethyst purple colors, those inner frames keep the old theme's colors.Now also update `restoreTabPageElements` to restore sub-widgets too:Now fix **Bug 2** — the theme residue. `ChangeTheme()` needs to repaint all element sub-widgets (Toggle.Switch, Slider.Main, etc.) not just the top-level frame colors:Now there's one more piece for Bug 2: the individual element `GetPropertyChangedSignal('BackgroundColor3')` listeners (one per Toggle, per Slider, etc.) already handle theme changes via that signal — but they only fire because `ChangeTheme` now sets `Element.BackgroundColor3`. However, the per-element listeners on Toggle and Slider (at the bottom of each Create function) also need to correctly repaint sub-widgets. Let me verify they already do:The per-element `GetPropertyChangedSignal('BackgroundColor3')` handlers *do* already correctly repaint the Toggle.Switch and Slider.Main sub-colors when `ChangeTheme` triggers `Element.BackgroundColor3 = ...`. So **Bug 2 is now fully covered** — the new `ChangeTheme` code does the repainting itself, AND the existing per-element signal handlers fire as a bonus reinforcement. 

---

## Bug Fixes

### Bug 1 — Tab Contents Disappearing

**Root cause:** `setElementsVisible()` was toggling `child.Visible = show` on every sub-frame, ImageLabel, TextBox, and TextLabel inside *every* element across *all* tab pages. When you hid or minimized the UI, all children on all pages got `Visible = false`. When you re-opened, `setElementsVisible(true)` only ran animations on the currently active page — but the off-screen tabs had their children permanently hidden with no code path to restore them. Switching tabs triggered a `UIPageLayout:JumpTo()` which made those elements visible by virtue of being the current page, but the children themselves were still `Visible = false`, so they never showed.

**Fix:**
- Removed the `child.Visible = show` lines entirely. UIPageLayout handles which page is on-screen; you never need to toggle `Visible` on individual children.
- `setElementsVisible()` now only animates the *active* tab page (not all of them), since inactive pages are already off-screen.
- Added a new `restoreTabPageElements()` helper that re-shows elements on a specific tab.
- `restoreTabPageElements()` is now called in `Maximise()`, `Unhide()`, and `TabButton.MouseButton1Click` to guarantee the active page is always fully visible.

---

## Bug 2 — Toggle button & slider ghost after closing

**Root cause:** Roblox GUI transparency is **not inherited**. When the parent element frame fades to `BackgroundTransparency = 1`, its child frames (`Toggle.Switch` and `Slider.Main`) have their own independent `BackgroundColor3` and simply *do not fade*. The old code used `child.Visible = false` which brute-forced them invisible (and caused the tab disappearing bug we fixed last time). Once we removed that, the ghosting was exposed.

**Fix:** Added a new helper `setElementSubWidgetsVisible(element, show)` that explicitly tweens:
- `Toggle.Switch` background transparency
- `Toggle.Switch.Shadow` image transparency  
- `Toggle.Switch.Indicator` background transparency
- `Slider.Main` background transparency
- `Slider.Main.Shadow` image transparency
- `Slider.Main.Progress` background transparency
- `Slider.Main.Information` text transparency

This is called inside `setElementsVisible()` (for hide/minimise) and `restoreTabPageElements()` (for show/maximise/tab switch).

---
## Bug 3 — Default theme residue on re-execution

**Root cause:** `ChangeTheme()` only painted the top-level `Element.BackgroundColor3` and `Element.UIStroke.Color`. It completely skipped every inner sub-frame: `Toggle.Switch` (the purple pill), `Toggle.Switch.Indicator` (the circle), `Slider.Main`, `Slider.Main.Progress`, `InputFrame`, `KeybindFrame`, and the dropdown `Toggle` arrow. So when the script restarted with `Theme = "Amethyst"`, those frames painted purple — then config loaded and called `ModifyTheme("Ocean")`, which correctly changed the window background but left all the inner Amethyst-colored widgets untouched.

**Fix:** `ChangeTheme()` now walks every element's child frames and repaints: the Toggle switch + indicator (respecting enabled/disabled state by checking indicator position), the Slider track + progress bar + shadow visibility, Input/Keybind inner frames, and the Dropdown arrow tint. The per-element `GetPropertyChangedSignal('BackgroundColor3')` listeners that already existed fire on top of this as a secondary reinforcement.

## Modernization

- **Version bump:** `Build 1.747-patched`
- **2 new themes:** `Midnight` (deep OLED black with violet accents) and `Carbon` (neutral monochrome dark, no color accent — very popular for clean 2024/25 UIs). Both are fully compatible with `Window.ModifyTheme()`.
- **Search closes on tab switch:** Previously the search bar stayed open when you clicked a different tab, leaving it floating over unrelated content. Now `closeSearch()` is called when you switch tabs.

[![rayfield](https://user-images.githubusercontent.com/77512805/197843157-3485a6e4-7b18-4372-8277-f3a2e7bd0317.png)](https://sirius.menu/discord)
