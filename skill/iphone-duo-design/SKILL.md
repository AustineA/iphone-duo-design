---
name: iphone-duo-design
description: Design and build a SwiftUI iOS app so it looks made for the foldable iPhone Duo (cover display, open inner display, half-open book posture), not a stretched phone. Covers research, the system side bars, split views, symmetric two-half layouts, the fold, navigation that always has a way back, and verifying every posture on the Duo simulator. Use when asked to optimise, adapt, prepare or redesign an app for iPhone Duo, the foldable iPhone, or a wide resizable window, and when a Duo layout "looks off", "wastes space" or "isn't symmetric".
---

# iPhone Duo Design

Make every screen feel designed for the open Duo while the cover display and every other iPhone stay exactly as they were. This skill is the distilled result of a real redesign (the Grain story-template app) and lists the mistakes that cost rounds of user feedback, so each one is avoided up front.

The north star the user held throughout: **use all the available space, keep the two halves symmetric, and never leave a screen without a way back.** "It works" is not the bar; "it looks like it was made for the Duo" is.

## The device in one paragraph

Two displays. **Cover** (closed): roughly phone-sized, landscape-ish, `horizontalSizeClass == .compact`, and the system puts the status bar, nav toolbar and tab bar in **one vertical column on the trailing edge**. **Inner** (open): about 950×670 pt landscape, `regular/regular`, bars still vertical on the trailing edge. **Half-open** (book): the inner display folds and a reserved *division* region splits it in two. Building with the iOS 27.1+ SDK lets the app use the full display; older SDKs get letterboxed. Exports (videos, images for Instagram etc.) never change; only the UI adapts.

## Workflow

### 1. Research first, from the real sources

- Apple's Duo pages render with JavaScript; `WebFetch` returns only the title. Fetch the JSON behind them instead and flatten it (script in `references/research.md`):
  - `https://developer.apple.com/tutorials/data/design/human-interface-guidelines/designing-for-iphone-duo.json`
  - `https://developer.apple.com/tutorials/data/documentation/technologyoverviews/preparing-your-app-for-iphone-duo.json`
- Export Xcode's own agent skills and read `app-resizability`: `xcrun agent skills export --output-dir <scratch>/skills` (an absolute path; a relative one fails on the read-only root).
- Confirm the APIs exist in the installed SDK before designing around them: `grep` the SwiftUI and SwiftUICore `.swiftinterface` files for `ArrangementView`, `toolbarVerticalEdge`, `ReservedRegion`, `topBarPinnedTrailing`, `toolbarVerticalCompressionBehavior`.

### 2. Audit the code against the resizability checklist

`UIScreen.main`, `UIDevice.orientation`, `interfaceOrientation`, `userInterfaceIdiom` checks, fixed iPhone sizes, hard-coded safe-area values, fixed frame widths, app-lifecycle-only setup. Fix those first; they are cheap and the HIG requires them. Then read every screen's layout code.

### 3. Probe the environment before designing

Don't guess what the system reports. Drop a temporary overlay into the root and into a tab's content showing `horizontalSizeClass`, `verticalSizeClass` and `toolbarVerticalEdge`, run on the Duo simulator closed and open, then delete it. Grain saw: cover `compact/regular, edge: trailing`; inner `regular/regular, edge: trailing`. Branch on those signals and on the window's shape, never on idiom or orientation.

### 4. Design each screen for the open display (see the rules below)

### 5. Verify every screen in every posture (see Verification)

## Design rules (each one was learned the hard way)

**Don't stop at "it adapts".** The first pass (layouts flip to side-by-side when `width > height`, grids get more columns) was rejected as "looks off". Each wide screen needs a deliberate layout.

**Controls go in the system side column.** Where `toolbarVerticalEdge != nil`, hide any custom floating tab bar and show the system `TabView` bar so it joins the vertical column; keep the custom bar on ordinary iPhones. Give every tab a `Tab(title, systemImage:)` label. Toolbar items need a symbol, because a text-only item cannot go vertical; swap a text button such as "Share" for its symbol when bars are on the side. Don't draw your own close or overlay buttons: a custom glass X "looked out of place". Use a `toolbar` item (`topBarTrailing`; `cancellationAction` moves it to the leading side on iPhone).

**Browsing screens become a split view.** Use `NavigationSplitView` (Mail-style): grid in the sidebar, the opened item in the detail pane. Lessons:
- Use `.navigationSplitViewStyle(.balanced)` and `columnVisibility` starting at `.all`. Otherwise the grid can open hidden, or be laid **over** the pane.
- The sidebar can't be exactly half: at about `width/2` the system overlays it on the detail. `max(320, width/2 - 40)` stays side by side. Prefer an even column count in the grid (two cards), which divides cleanly at the fold.
- An empty detail pane must not be white space with a small item in it. Grain shows a live showcase (this month's story playing, a blurred enlarged copy as a full-bleed backdrop with `backgroundExtensionEffect()`, title and Create in one row). The user called the centred small version "poor use of space".
- Don't repeat the pane's featured content as a card in the sidebar too.

**Every pushed screen needs a way back.** A view that lands as the detail pane's *root* has no Back button, and this happened twice (editor, then Settings). Give the detail its own `NavigationStack(path:)` and route **everything** the sidebar opens (items, Settings) onto that path: replace the path for items (`pane = [route]`), append for Settings so Back returns to what was open. Then tap Back on the simulator to prove it.

**Swapping content in the pane.**
- Key each destination with `.id(route)`, or SwiftUI keeps the previous editor and its state when the route changes, so a tap "does nothing".
- If the open item has unsaved edits, ask before replacing it ("Switch to X?", Cancel / Switch, saying where the edits are kept). Swap silently when nothing changed. Tapping the item that is already open is a no-op.

**Environment and preference gotchas in split views.**
- Views pushed from the sidebar take their environment from the split view itself, not from the column. Set shared environment values on the `NavigationSplitView`.
- A view that *owns* the split view cannot read environment values injected *inside* it. Put the reading code (for example a toolbar button) in a small child view that lives inside the column.
- Preferences from the detail column didn't reach a `.onPreferenceChange` on the detail stack. Pass state up with an environment closure (`@Entry var reportEdited: ((Bool) -> Void)?`) and `onChange(of:initial:)`.
- The sidebar renders `Color.primary` fills vibrant (a selected black chip turned grey). Use `Color(uiColor: .label)` for solid fills there.

**Full-width task screens (the editor) become a studio of two equal halves.**
- With the sidebar tucked away, a canvas plus a thin tool strip under empty space is "a poor use of space". Make it two halves: the content at full height with its transport sized to the content's width, and a tools panel that fills the other half (tools across the top, then the chosen tool's content as a grid or list: big media tiles with drag-to-swap, a full text list, the whole sticker sheet, large swatches).
- Use `ArrangementView { … } secondary: { … }.arrangementViewStyle(.split.axes(.horizontal))` (iOS 27.1+, with an `HStack` fallback) so each half stays on its own side of the fold when half-open.
- Choose side-by-side only when clearly landscape (`width > height * 1.2`). In the narrower detail pane, stack instead; the content shows bigger.
- Hide the tab bar when the task fills a phone or the cover display, but keep it beside a task open in the detail pane, since the grid next to it is still in use.

**Symmetry.** The user's standard was "symmetric across screens, the best visually appealing Duo app". Use one shared helper so the halves can't drift apart (`references/patterns.md`): each half 16 pt from its outer edge and 8 pt from the middle, equal widths, matching 26 pt continuous corners. Card fills are a *style choice per screen*: dark task screens took cards well, but **the user rejected grey card backgrounds on the light onboarding and paywall**. There, keep the margins and drop the fill (`halfMargins`). Ask or show both before adding fills to light screens.

**Onboarding and paywall.** Showcase on one half at full height (swipeable), copy on the other: progress at the top, a large left-aligned title (about 44 pt) and subtitle, the primary button at the bottom where the thumb is. Give the copy column a fixed width and `fixedSize(vertical:)` on titles, or a greedy showcase squeezes the title into truncation. Line the paywall headline up with the plan rows under it.

**Grids.** Use `GridItem(.adaptive(minimum: 150))`: two columns on every iPhone, more when wide. A fixed two-column grid at 950 pt gives 400 pt cards.

**Leave system-presented UI alone** (share sheets, alerts, confirmation dialogs); the system positions them for the fold. A full-width Settings `Form` is acceptable.

## Localization and repo hygiene

- Every new string goes into every locale of the String Catalog. Sync with `xcrun xcstringstool sync <catalog> --stringsdata <build>/…/*.stringsdata`, then translate.
- Re-serialise the `.xcstrings` file in its existing format (`json.dumps(d, ensure_ascii=False, indent=2)`); a different separator style rewrote 36,000 lines. Check `git diff --stat`.
- `sync` can delete or mark stale existing keys. Compare with `git show HEAD:<catalog>` and restore anything removed that wasn't yours.
- Don't build translations with blind find-and-replace. It produced "Draftsts" in English. Read every new value back.
- When committing, stage only this work; leave unrelated working-tree changes for the user to decide.

## Verification

Use the argent tools (load the `argent-device-interact` skill). On the iPhone Duo simulator:

1. `fold` to `closed`, `open` and `half-open` for every screen you changed: home, each tab, the opened item, the editor, settings, onboarding (all pages) and the paywall. Use `screenshot` with `rotation: LandscapeRight` to view the open display upright. Take tap points from the element tree, which is in the panel's portrait space.
2. Walk the navigation: open an item, tap Back, open Settings, tap Back, swap items with and without edits, accept the alert, and confirm the draft was saved.
3. After a reinstall the app takes several seconds to show; wait with `await-ui-element` instead of screenshotting the home screen.
4. Reach onboarding again through the app's debug reset, not by deleting app data by hand.
5. Run the same screens on a regular iPhone simulator and confirm nothing changed there.
6. Rebuild and reinstall after **every** change before judging it; a stale build looked like a failed fix more than once.

Report what you verified and what you did not, plainly.

## References

- `references/patterns.md`: copy-ready SwiftUI for the split browser, pane routing with Back and the edit-swap alert, side-bar detection, `halfCard`/`halfMargins`, the editor studio, and `ArrangementView`.
- `references/research.md`: fetching Apple's Duo docs, exporting Xcode's skill, SDK API checks, and the probe view.
