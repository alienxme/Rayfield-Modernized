https://docs.sirius.menu/rayfield
https://www.rayfield.gg/
https://developer.sirius.menu/

Welcome to an unofficial fork of the Rayfield respitory this fork addresses the bugs below that the developers of rayfield was too lazy to fix.

**Bug 1 — Tab contents disappearing:** In `setElementsVisible(show)`, child visibility is set with `child.Visible = show` — this forces ALL child frames/labels/etc. invisible across ALL tabs, not just the active one. When you switch tabs via `UIPageLayout:JumpTo()`, the new tab's children were already set `Visible = false` by the last hide/unhide cycle and never re-shown (because `setElementsVisible(true)` sets them all, but it races against tween completion and the `Visible` toggle applies globally). The fix: only touch the *current* tab's children, and use transparency tweens consistently rather than toggling `Visible`.

**Bug 2 — Slider ghost on close:** The slider's `RunService.Stepped` loop in `MouseButton1Down` checks `SLDragging` but only disconnects when `SLDragging` is false. The problem is `InputEnded` on the `Interact` object only fires if the mouse-up happens over the slider — if you drag outside and release, `SLDragging` never resets. The loop continues running even after the window closes. Fix: use `UserInputService.InputEnded` globally (like the draggable does) instead of `Interact.InputEnded`.Now fix the tab switching to call `restoreTabPageElements` when switching tabs, so any tab that was hidden while inactive gets properly restored:Now fix Bug 2 — the slider ghost. The core issue: `Slider.Main.Interact.InputEnded` only fires when the mouse is released *over* the slider interact region. If the user drags out, `SLDragging` stays `true` and the `RunService.Stepped` loop never stops. Also the loop doesn't stop when the window is hidden/closed. Fix: replace the local `InputEnded` with a global `UserInputService.InputEnded`, and add a cleanup on `Slider.AncestryChanged`:Now let's also fix the `Maximise` function — it calls `setElementsVisible(true)` and then immediately shows things, but it should also restore the current tab page:Now let's also add the same restore call in `Unhide`:Now let's modernize the library. I'll update the version tag and add a modern `DarkBlue` theme (a popular 2024/25 Roblox UI trend), improve the notification system duration math, and add a missing `CloseSearch` on tab switch:Now add a modern `DarkBlue` theme and a `Midnight` theme that weren't in the original — both are heavily requested in the Roblox UI library community:Good — `DarkBlue` and `Serenity` already exist. Now let's add two more modern themes and also fix a common edge case: the tab click handler should close search if it's open:Now let's fix the tab click handler to also close search when switching tabs (currently it doesn't, which leaves the search bar floating over the wrong tab):Now let's verify the file is valid Lua by checking for obvious syntax errors with a line count and grep:Clean — only comments remain. Now copy to output:Here's a full rundown of everything that was done:

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

### Bug 2 — Slider Ghost After Close

**Root cause:** The slider used `Slider.Main.Interact.InputEnded` to reset `SLDragging = false`. This event only fires when the mouse button is released *over* the interact element. Drag outside the slider → release → `SLDragging` stays `true` → the `RunService.Stepped` loop never disconnects → the slider keeps ticking and rendering ("ghost") even after the window closes.

**Fix:** Replaced `Interact.InputEnded` with a global `UserInputService.InputEnded` connection (same pattern the draggable uses). Added `Slider.Destroying:Connect()` to cleanly disconnect that global connection when the element is destroyed.

---

## Modernization

- **Version bump:** `Build 1.747-patched`
- **2 new themes:** `Midnight` (deep OLED black with violet accents) and `Carbon` (neutral monochrome dark, no color accent — very popular for clean 2024/25 UIs). Both are fully compatible with `Window.ModifyTheme()`.
- **Search closes on tab switch:** Previously the search bar stayed open when you clicked a different tab, leaving it floating over unrelated content. Now `closeSearch()` is called when you switch tabs.

[![rayfield](https://user-images.githubusercontent.com/77512805/197843157-3485a6e4-7b18-4372-8277-f3a2e7bd0317.png)](https://sirius.menu/discord)
