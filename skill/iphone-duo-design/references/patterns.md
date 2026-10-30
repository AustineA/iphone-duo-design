# iPhone Duo SwiftUI patterns

Proven in the Grain app (iOS 26 deployment target, built with the iOS 27.2 SDK). Adapt names to the project.

## Side-bar detection

```swift
/// Reports whether the system draws bars vertically on the side, as on iPhone Duo.
struct SideBarsReader: View {
    let changed: (Bool) -> Void

    var body: some View {
        if #available(iOS 27.1, *) { Reader(changed: changed) }
    }

    @available(iOS 27.1, *)
    private struct Reader: View {
        let changed: (Bool) -> Void
        @Environment(\.toolbarVerticalEdge) private var edge

        var body: some View {
            Color.clear.onChange(of: edge != nil, initial: true) { _, side in changed(side) }
        }
    }
}
```

The root view uses it to swap a custom floating tab bar for the system one:

```swift
let tabBar: Visibility = sideBars && !barHidden ? .automatic : .hidden
TabView(selection: $section) {
    Tab("Templates", systemImage: "square.grid.2x2", value: .templates) { LibraryView().toolbarVisibility(tabBar, for: .tabBar) }
    // …each tab the same, Search with role: .search
}
.background { SideBarsReader { sideBars = $0 } }
.contentMargins(.bottom, barHidden || sideBars ? 0 : 90, for: .scrollContent)
.overlay(alignment: .bottom) { if !barHidden && !sideBars { FloatingTabBar(…) } }
```

## Split browser with a pane that always has a way back

```swift
enum Route: Hashable { case settings, template(String), draft(UUID) }

struct StoryBrowser<Sidebar: View>: View {
    @ViewBuilder let sidebar: Sidebar
    @Environment(\.horizontalSizeClass) private var sizeClass
    @State private var pane: [Route] = []
    @State private var paneEdited = false
    @State private var pending: Route?
    @State private var width = 0.0
    @State private var columns = NavigationSplitViewVisibility.all

    var body: some View {
        let split = sizeClass == .regular
        NavigationSplitView(columnVisibility: $columns) {
            sidebar
                .storyDestinations()                       // pushes on phones / cover
                .navigationSplitViewColumnWidth(max(320, width / 2 - 40))
                .environment(\.openStory, split ? open : nil)
        } detail: {
            NavigationStack(path: $pane) {                 // the pane's own stack: Back buttons
                Showcase().storyDestinations()
            }
        }
        .alert(pendingTitle, isPresented: Binding { pending != nil } set: { if !$0 { pending = nil } }) {
            Button("Switch") { if let pending { pane = [pending] }; pending = nil }
            Button("Cancel", role: .cancel) { pending = nil }
        } message: { Text("Your changes to this story are kept in Drafts.") }
        .navigationSplitViewStyle(.balanced)
        .onGeometryChange(for: Double.self) { $0.size.width } action: { width = $0 }
        // On the split view itself: what the sidebar pushes takes its environment from here.
        .environment(\.storyPaneShown, split)
        .environment(\.reportEdited, { paneEdited = $0 })
    }

    private func open(_ route: Route) {
        guard pane.last != route else { return }
        if route == .settings { pane.append(route); return }          // Back returns to what was open
        if paneEdited, !pane.isEmpty { pending = route } else { pane = [route] }
    }
}

extension EnvironmentValues {
    @Entry var openStory: ((Route) -> Void)?
    @Entry var storyPaneShown = false
    @Entry var reportEdited: ((Bool) -> Void)?
}

/// A card or button that opens in the pane when there is one, else pushes.
struct StoryLink<Label: View>: View {
    let route: Route
    @ViewBuilder let label: Label
    @Environment(\.openStory) private var openStory
    var body: some View {
        if let openStory { Button { openStory(route) } label: { label } }
        else { NavigationLink(value: route) { label } }
    }
}

extension View {
    func storyDestinations() -> some View {
        navigationDestination(for: Route.self) { route in
            switch route {
            case .settings: SettingsView()
            case .template(let id): EditorView(id: id).id(route)   // .id: a new route gets a fresh editor
            case .draft(let id): DraftEditor(id: id).id(route)
            }
        }
    }
}
```

The editor reports edits up and keeps the tab bar beside the pane:

```swift
@Environment(\.storyPaneShown) private var storyPaneShown
@Environment(\.reportEdited) private var reportEdited
// …
.preference(key: TabBarHiddenKey.self, value: !storyPaneShown)
.onChange(of: draft.isEdited, initial: true) { _, edited in reportEdited?(edited) }
```

A toolbar button on the screen that *owns* `StoryBrowser` must be its own small view, so it reads `openStory` from inside the column:

```swift
private struct SettingsButton: View {
    let push: () -> Void
    @Environment(\.openStory) private var openStory
    var body: some View {
        Button("Settings", systemImage: "gearshape") {
            if let openStory { openStory(.settings) } else { push() }
        }
    }
}
```

## Symmetric halves

```swift
extension View {
    /// One of two equal cards side by side on a wide display: 16 pt from the display's edge and
    /// 8 pt from the middle, so the pair sits symmetric about it.
    func halfCard(edge: HorizontalEdge, fill: Color = Color(uiColor: .secondarySystemBackground)) -> some View {
        let shape = RoundedRectangle(cornerRadius: 26, style: .continuous)
        return background(fill, in: shape).clipShape(shape).halfMargins(edge: edge)
    }

    /// The same margins with no card, for light screens (the user rejected fills there).
    func halfMargins(edge: HorizontalEdge) -> some View {
        padding(.vertical, 16)
            .padding(edge == .leading ? .leading : .trailing, 16)
            .padding(edge == .leading ? .trailing : .leading, 8)
    }
}
```

## Fold-aware two-half screen

```swift
Group {
    if wide, #available(iOS 27.1, *) {
        ArrangementView {
            stage(story)          // .halfCard(edge: .leading, fill: Color(white: 0.08))
        } secondary: {
            inspector(story)      // .halfCard(edge: .trailing, fill: Color(white: 0.08))
        }
        .arrangementViewStyle(.split.axes(.horizontal))
    } else if wide {
        HStack(spacing: 0) { stage(story); inspector(story) }
    } else {
        VStack(spacing: 0) { canvas(story); controls(story) }   // the phone layout, unchanged
    }
}
.onGeometryChange(for: Bool.self) { $0.size.width > $0.size.height * 1.2 } action: { wide = $0 }
```

The stage sizes its transport to the content:

```swift
GeometryReader { geometry in
    let width = min(geometry.size.width, (geometry.size.height - 48) * 9 / 16)
    VStack(spacing: 4) {
        canvas(story)
        transport(story).frame(width: width)
    }
}
```

The inspector is the same tool component with a `panel: Bool` that switches each tool from a horizontal strip to a filling grid or list (`LazyVGrid` with three flexible columns for media using `.draggable` and `.dropDestination` to swap, an adaptive grid for stickers, a full-width list for text, big swatches with names for style).

## Pane showcase that fills the space

```swift
VStack(spacing: 18) {
    StoryLoop(templateID: "monthly-dump", playing: true).frame(maxHeight: .infinity)
    HStack {
        VStack(alignment: .leading) { Text("MONTHLY DUMP").font(.caption.weight(.semibold)); Text("October Dump").font(.title2.bold()) }
        Spacer()
        StoryLink(route: .template("monthly-dump")) { Text("Create") /* white capsule */ }
    }
    .frame(maxWidth: 420)
}
.padding(24)
.frame(maxWidth: .infinity, maxHeight: .infinity)
.background {
    Color.black.overlay { Thumbnail().scaleEffect(2.4).blur(radius: 48).opacity(0.85) }
        .clipped().backgroundExtensionEffect().ignoresSafeArea()
}
.environment(\.colorScheme, .dark)
```
