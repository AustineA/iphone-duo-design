# Researching iPhone Duo

## Apple's docs (JavaScript pages: fetch the JSON)

```bash
OUT=<scratch>/duo; mkdir -p "$OUT"; cd "$OUT"
for u in design/human-interface-guidelines/designing-for-iphone-duo \
         documentation/technologyoverviews/preparing-your-app-for-iphone-duo; do
  curl -sL "https://developer.apple.com/tutorials/data/$u.json" -o "$(basename $u).json"
done
```

Flatten to readable text (keep the script outside the download folder and run Python isolated):

```python
import json, sys
def walk(n, out):
    if isinstance(n, dict):
        t = n.get('type')
        if t == 'heading': out.append('\n## ' + n.get('text', ''))
        if t == 'text': out.append(n.get('text', ''))
        if t == 'codeVoice': out.append('`' + n.get('code', '') + '`')
        if t == 'reference': out.append('[' + n.get('identifier', '').split('/')[-1] + ']')
        if t == 'codeListing': out.append('\n```\n' + '\n'.join(n.get('code', [])) + '\n```\n')
        for k, v in n.items():
            if k != 'references': walk(v, out)
        if t in ('paragraph', 'listItem'): out.append('\n')
    elif isinstance(n, list):
        for x in n: walk(x, out)
d = json.load(open(sys.argv[1])); o = []
walk(d.get('primaryContentSections', d), o); print(''.join(o))
```

`python3 -I flat.py designing-for-iphone-duo.json`

Key points from the HIG (September 2026): build to resize with size classes, don't reinvent the app per pose; keep functionality and state across displays; follow the system's vertical bars and don't override them; prefer even grid columns; avoid extreme layout changes when folding; use split views and arrangement views; full-width immersive layouts are fine where nothing conflicts with the status bar; extend hero images under the vertical bar with `backgroundExtensionEffect()`; give toolbar items a title and a symbol; keep Back/Close at the top of the vertical axis.

## Xcode's resizability skill

```bash
xcrun agent skills export --output-dir "<absolute scratch path>/skills"
cat "<scratch>/skills/app-resizability/SKILL.md"   # plus references/*.md
```

It covers `UIScreen.main`, orientation, idiom, safe-area and scene-lifecycle migrations, plus launch-screen and `UIRequiresFullScreen` checks.

## Confirm APIs in the installed SDK

```bash
SDK=$(xcrun --sdk iphonesimulator --show-sdk-path)
for fw in SwiftUI SwiftUICore; do
  F=$(find $SDK/System/Library/Frameworks/$fw.framework -name "arm64-apple-ios-simulator.swiftinterface" | head -1)
  grep -n "struct ArrangementView\|func arrangementViewStyle\|toolbarVerticalEdge\|struct ReservedRegion\|topBarPinnedTrailing\|toolbarVerticalCompressionBehavior" "$F"
done
```

In the 27.2 SDK, `ArrangementView`, `ReservedRegion` and `toolbarVerticalEdge` live in SwiftUICore and are `@available(iOS 27.1, *)`; gate them behind `#available` when the deployment target is lower.

## Probe view (temporary; delete after reading)

```swift
struct DuoProbe: View {
    let tag: String
    @Environment(\.horizontalSizeClass) private var h
    @Environment(\.verticalSizeClass) private var v
    var body: some View {
        if #available(iOS 27.1, *) { Inner(tag: tag, h: "\(String(describing: h))", v: "\(String(describing: v))") }
    }
    @available(iOS 27.1, *)
    private struct Inner: View {
        let tag: String, h: String, v: String
        @Environment(\.toolbarVerticalEdge) private var edge
        var body: some View {
            Text(verbatim: "\(tag) h:\(h) v:\(v) edge:\(String(describing: edge))")
                .font(.caption2.monospaced()).padding(4).background(.yellow).foregroundStyle(.black)
                .allowsHitTesting(false)
        }
    }
}
```

Overlay it at the root and inside a tab, fold the simulator closed and open, read the values from `describe`, then remove the file.

## Simulator

`list-devices` shows the iPhone Duo with `foldable: true`. `fold` takes `posture: closed | half-open | open`. The open panel captures portrait; pass `rotation: LandscapeRight` to `screenshot` to view it upright, while taps use the unrotated element-tree coordinates.
