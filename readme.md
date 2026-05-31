https://docs.sirius.menu/rayfield
https://www.rayfield.gg/
https://developer.sirius.menu/

Welcome to an unofficial fork of the Rayfield respitory this fork addresses the bugs below that the developers of rayfield was too lazy to fix:

**Bug 1 (tab content disappearing on ESC):** `setElementsVisible(false)` only operates on the `currentPage` when `show=false` — wait, re-reading: `if isActivePage or not show` — when `show=false`, `not show` = `true`, so it DOES process ALL tabs on hide. That's correct. But when `show=true` it only processes the active page. The problem: on `Minimise()`, `setElementsVisible(false)` runs on ALL tabs and hides their elements, then `Elements.Visible = false` is set. When you `Maximise()`, `Elements.Visible = true` is set but `setElementsVisible(true)` only restores the *currently active page*. If the user pressed ESC (which opens the Roblox menu and may trigger nothing in Rayfield) but the minimise happened previously, the other tabs never get their elements' transparency reset. Additionally `task.defer` for `restoreTabPageElements` runs *asynchronously* after a frame delay, meaning there's a visible flash.

**Bug 2 (theme residue):** Image 2 still shows the toggle pills are purple (Amethyst) even after Ocean theme loaded. The `ChangeTheme()` fix checks `isEnabled` by position: `indicator.Position.X.Scale >= 0.5` — but all Toggle indicator positions use **offset** not scale! The disabled position is `UDim2.new(1, -40, ...)` and enabled is `UDim2.new(1, -20, ...)`. Both have `X.Scale = 1`, so `>= 0.5` is always `true` — meaning ChangeTheme always paints ALL toggles as "enabled" color. This is wrong for disabled toggles.

**Bug 3 (outline ghost):** `setElementSubWidgetsVisible` hides `switch.BackgroundTransparency` and `indicator.BackgroundTransparency` but completely misses: `switch.UIStroke.Transparency`, `indicator.UIStroke.Transparency`, `slider.Main.UIStroke.Transparency`, and `slider.Main.Progress.UIStroke.Transparency`. The visible green outline in Image 3 is the slider's `UIStroke` remaining at full opacity.Now fix Bug 2 — the `isEnabled` check using `.Position.X.Scale` which is always `1` for offset-based positions:Now fix Bug 1 — the tab content disappearing. The problem is `setElementsVisible(false)` processes ALL tabs (correct), but on restoration `setElementsVisible(true)` + `restoreTabPageElements` only covers the current page. When the user switches to a different tab after a minimise/hide cycle, that other tab's elements were hidden and never restored. The fix: apply the restore to ALL tabs when restoring, not just the current one:Now update `restoreTabPageElements` to also use `setElementSubWidgetsVisible` with UIStrokes, and simplify the `restoreTabPageElements` — since `setElementsVisible(true)` now covers all tabs, the restore helper is mostly redundant but we keep it for the tab-switch case:Good. Now verify and copy out:All three fixes confirmed. Copy to output:Three precise root-cause fixes:

---

### Bug 1 — Tab contents disappearing on ESC / tab switch

**Root cause (found this session):** The previous fix added `if isActivePage or not show` — meaning `setElementsVisible(true)` only restored the *currently active tab*. But `setElementsVisible(false)` runs on ALL tabs (since `not show = true`). So after a hide/minimise cycle, every tab's elements were hidden. On restore, only the active tab got its transparency reset. Any other tab you navigated to was still fully hidden.

**Fix:** Removed the `isActivePage` guard entirely. `setElementsVisible` now processes **all tabs in both directions** — hide touches every tab so nothing ghosts, and show restores every tab so switching to any page works immediately.

---

### Bug 2 — Theme residue (Amethyst purple leaking through)

**Root cause (found this session):** The `isEnabled` detection in `ChangeTheme` used `indicator.Position.X.Scale >= 0.5`. But Rayfield uses **offset-based** `UDim2` positions — `UDim2.new(1, -20, ...)` for enabled and `UDim2.new(1, -40, ...)` for disabled. Both have `X.Scale = 1`, so the condition was **always `true`** — every toggle was repainted with the enabled (purple) color regardless of its actual state.

**Fix:** Changed to `indicator.Position.X.Offset > -30`. Since enabled offset = `-20` and disabled = `-40`, the threshold of `-30` correctly distinguishes them. Also switched from `.UIStroke` direct property access to `FindFirstChildOfClass("UIStroke")` which is safer and matches how the elements are actually structured.

---

### Bug 3 — Toggle outline + slider outline ghost after close

**Root cause (found this session):** `setElementSubWidgetsVisible` was fading `BackgroundTransparency` on the Switch frame and Indicator frame, but completely missed their **`UIStroke` objects**. UIStrokes are separate `Instance` children and have their own `Transparency` property — they don't inherit from the parent frame. The green ring (slider) and toggle ring visible in Image 3 were exactly these UIStrokes remaining at `Transparency = 0`.

**Fix:** Added `FindFirstChildOfClass("UIStroke")` lookups and transparency tweens for: `Switch.UIStroke`, `Indicator.UIStroke`, `Slider.Main.UIStroke`, and `Slider.Main.Progress.UIStroke`. All four are now properly faded to `Transparency = 1` on hide and restored on show.

[![rayfield](https://user-images.githubusercontent.com/77512805/197843157-3485a6e4-7b18-4372-8277-f3a2e7bd0317.png)](https://sirius.menu/discord)
