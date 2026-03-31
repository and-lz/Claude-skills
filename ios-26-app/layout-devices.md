# Layout, Spacing, and Device Dimensions

## Table of contents
1. Spacing grid system
2. Safe areas
3. iPhone screen dimensions
4. iPad screen dimensions
5. Size classes
6. Layout patterns
7. iPadOS 26 windowing
8. Responsive design patterns
9. Performance considerations

---

## 1. Spacing grid system

Apple uses an **8pt base grid**. All spacing, padding, and sizing should be multiples of 4pt or 8pt.

| Token | Value | Usage |
|-------|-------|-------|
| 4pt | Micro | Icon-to-label gap, tight related elements |
| 8pt | Small | Default stack spacing, compact padding |
| 12pt | - | Section header vertical padding |
| 16pt | Medium | System horizontal margins (compact width), card padding |
| 20pt | Large | System horizontal margins (regular width) |
| 24pt | - | Section spacing in forms |
| 32pt | XL | Major section separators |
| 40pt | XXL | Hero content spacing |

### System layout margins
- iPhone (compact width): **16pt** leading/trailing
- iPad (regular width): **20pt** leading/trailing
- `readableContentGuide` constrains text to comfortable reading width (~672pt max)
- Always use `layoutMarginsGuide` (UIKit) or default padding (SwiftUI) rather than hardcoding

### Minimum touch targets
**44×44 points** for all interactive elements. This has been the Apple standard since iOS 7 and remains unchanged. The visual size of a control can be smaller than 44pt as long as the tappable area is at least 44×44pt.

```swift
// SwiftUI: small visual, large tap area
Button { action() } label: {
    Image(systemName: "xmark")
        .font(.caption)
}
.frame(minWidth: 44, minHeight: 44)
```

## 2. Safe areas

### What safe areas protect
- **Top**: Status bar, Dynamic Island, camera housing
- **Bottom**: Home indicator (34pt on modern iPhones)
- **Leading/Trailing**: Curved screen edges, sensor housing

### SwiftUI safe area handling
```swift
// Content automatically respects safe areas
ScrollView {
    content
}

// Extend behind safe areas (for backgrounds)
Color.blue
    .ignoresSafeArea()

// Add custom safe area insets (for floating UI)
.safeAreaInset(edge: .bottom) {
    FloatingPlayer()
}

// Safe area padding for specific edges
.safeAreaPadding(.horizontal, 16)
```

### UIKit safe area handling
```swift
// Constrain to safe area
view.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor)

// Get inset values
let topInset = view.safeAreaInsets.top
let bottomInset = view.safeAreaInsets.bottom
```

### Never hardcode these values — they vary by device model.

## 3. iPhone screen dimensions (2024–2025 models)

### iPhone 17 series (2025)

| Model | Screen (pt) | Scale | Resolution (px) | Safe Top | Safe Bottom |
|-------|------------|-------|-----------------|----------|-------------|
| iPhone 17 | 402 × 874 | @3x | 1206 × 2622 | 62pt | 34pt |
| iPhone 17 Air | 420 × 912 | @3x | 1260 × 2736 | 68pt | 34pt |
| iPhone 17 Pro | 402 × 874 | @3x | 1206 × 2622 | 62pt | 34pt |
| iPhone 17 Pro Max | 440 × 956 | @3x | 1320 × 2868 | 62pt | 34pt |

### iPhone 16 series (2024)

| Model | Screen (pt) | Scale | Resolution (px) | Safe Top | Safe Bottom |
|-------|------------|-------|-----------------|----------|-------------|
| iPhone 16 / 15 | 393 × 852 | @3x | 1179 × 2556 | 59pt | 34pt |
| iPhone 16 Plus / 15 Plus | 430 × 932 | @3x | 1290 × 2796 | 59pt | 34pt |
| iPhone 16 Pro | 402 × 874 | @3x | 1206 × 2622 | 62pt | 34pt |
| iPhone 16 Pro Max | 440 × 956 | @3x | 1320 × 2868 | 62pt | 34pt |

### Older supported models

| Model | Screen (pt) | Scale | Safe Top | Safe Bottom |
|-------|------------|-------|----------|-------------|
| iPhone SE (3rd) | 375 × 667 | @2x | 20pt | 0pt |
| iPhone 14 | 390 × 844 | @3x | 59pt | 34pt |
| iPhone 14 Plus | 428 × 926 | @3x | 59pt | 34pt |
| iPhone 14 Pro | 393 × 852 | @3x | 59pt | 34pt |
| iPhone 14 Pro Max | 430 × 932 | @3x | 59pt | 34pt |

### Dynamic Island
Present on all models except iPhone SE. Compact height: ~37pt. Expanded max height: ~160pt. Status bar height on Dynamic Island phones: **54pt**.

### Key design widths to target
- **375pt** — smallest (iPhone SE)
- **390–402pt** — standard (iPhone 14–17)
- **420–440pt** — large (Air, Plus, Pro Max)

## 4. iPad screen dimensions

All current iPads use **@2x scale factor**.

| Model | Screen (pt) | Resolution (px) |
|-------|------------|-----------------|
| iPad (10th gen) | 820 × 1180 | 1640 × 2360 |
| iPad Air (M3, 11") | 820 × 1180 | 1640 × 2360 |
| iPad Air (M3, 13") | 1024 × 1366 | 2048 × 2732 |
| iPad Pro (M4, 11") | 834 × 1194 | 1668 × 2388 |
| iPad Pro (M4, 13") | 1024 × 1366 | 2048 × 2732 |
| iPad mini (A17, 8.3") | 744 × 1133 | 1488 × 2266 |

iPad safe areas: **24pt top**, **20pt bottom** (with home indicator).

## 5. Size classes

| Device/Configuration | Width | Height |
|---------------------|-------|--------|
| All iPhones portrait | **Compact** | **Regular** |
| Standard iPhones landscape | Compact | **Compact** |
| Plus/Max/Air iPhones landscape | **Regular** | Compact |
| All iPads (all orientations) | **Regular** | **Regular** |
| iPad Split View (narrow) | Compact | Regular |
| iPad Slide Over | Compact | Regular |

### SwiftUI size class usage
```swift
@Environment(\.horizontalSizeClass) var hSizeClass
@Environment(\.verticalSizeClass) var vSizeClass

var body: some View {
    if hSizeClass == .compact {
        // Single-column layout (iPhone)
        NavigationStack { ... }
    } else {
        // Multi-column layout (iPad)
        NavigationSplitView { ... }
    }
}
```

### ViewThatFits (iOS 16+)
```swift
ViewThatFits {
    HorizontalLayout()   // Tried first
    VerticalLayout()     // Fallback
    CompactLayout()      // Last resort
}
```

## 6. Layout patterns

### ScrollView with lazy loading
```swift
ScrollView {
    LazyVStack(spacing: 12, pinnedViews: .sectionHeaders) {
        ForEach(sections) { section in
            Section {
                ForEach(section.items) { item in
                    ItemRow(item: item)
                }
            } header: {
                SectionHeader(title: section.title)
            }
        }
    }
    .padding(.horizontal)
}
```

Use `LazyVStack` / `LazyHStack` for any list longer than ~20 items. They create views only when visible. Provide stable, unique IDs for all items.

### Grid layouts
```swift
// Adaptive grid — fills available width
let adaptive = [GridItem(.adaptive(minimum: 160), spacing: 16)]

// Fixed 2-column
let twoCol = [GridItem(.flexible()), GridItem(.flexible())]

// Mixed
let mixed = [
    GridItem(.fixed(80)),
    GridItem(.flexible(minimum: 100)),
    GridItem(.flexible(minimum: 100))
]
```

### GeometryReader (use sparingly)
```swift
GeometryReader { proxy in
    let width = proxy.size.width
    // Layout based on available width
}
```

Prefer `ViewThatFits`, size classes, and `containerRelativeFrame` over GeometryReader when possible.

### containerRelativeFrame (iOS 17+)
```swift
Image("hero")
    .containerRelativeFrame(.horizontal) { length, _ in
        length * 0.8  // 80% of container width
    }
```

## 7. iPadOS 26 windowing

iPadOS 26 replaces Split View and Slide Over with a macOS-like windowing system:

- **Freely resizable windows** with close/minimize/maximize controls
- **Window tiling** by flicking windows to screen edges (halves, quarters)
- **Exposé** for viewing all open windows
- **Menu bar** accessible via swipe or cursor
- **Stage Manager** available on ALL supported iPads (no longer M-chip only)

### Implications for app design
- Apps MUST support arbitrary window sizes (not just full-screen + split view ratios)
- Test at very narrow widths (similar to iPhone)
- Use size classes to adapt layout — don't assume iPad = full screen
- The `commands` SwiftUI modifier works identically on iPad and Mac:

```swift
.commands {
    CommandGroup(after: .newItem) {
        Button("New from Template") { }
            .keyboardShortcut("t", modifiers: [.command, .shift])
    }
}
```

### External display support
Apps can have independent scenes on external displays. Use `WindowGroup` with `openWindow` environment action:
```swift
@Environment(\.openWindow) var openWindow

Button("Open Player") {
    openWindow(id: "player")
}
```

## 8. Responsive design patterns

### Adaptive navigation (tab bar ↔ sidebar)
```swift
TabView {
    TabSection("Content") {
        Tab("Feed", systemImage: "list.bullet") { FeedView() }
        Tab("Explore", systemImage: "compass") { ExploreView() }
    }
    Tab("Settings", systemImage: "gear") { SettingsView() }
}
.tabViewStyle(.sidebarAdaptable)
```

### Adaptive list ↔ grid
```swift
@Environment(\.horizontalSizeClass) var sizeClass

var body: some View {
    if sizeClass == .compact {
        List(items) { item in ItemRow(item: item) }
    } else {
        ScrollView {
            LazyVGrid(columns: gridColumns) { ... }
        }
    }
}
```

### Safe area and scroll edge effects
```swift
ScrollView {
    content
}
.scrollEdgeEffectStyle(.soft, for: .top)    // Soft blur under nav bar
.scrollEdgeEffectStyle(.soft, for: .bottom) // Soft blur above tab bar
```

Two styles: **Soft** (default, translucent fade) and **Hard** (denser, for pinned headers). Use one scroll edge effect per edge per view.

## 9. Performance considerations

### Frame budget
- 60Hz devices: **16.67ms** per frame
- 120Hz ProMotion: **8.33ms** per frame — any work over this drops frames

### Rules
- Never block the main thread with network, disk, or heavy computation
- Use `.task { }` for async work tied to view lifecycle
- Use `LazyVStack`/`LazyHStack` in ScrollView — not regular `VStack` for lists
- Use `@Observable` (iOS 17+) for property-level change tracking — more efficient than `ObservableObject`
- Break complex views into small subviews to limit state invalidation
- Use `.id()` on ForEach with stable identifiers — don't use array indices
- Profile with Instruments (Time Profiler, SwiftUI Instruments) before optimizing
- iOS 26's rebuilt SwiftUI pipeline delivers 39% faster renders, 40% less GPU, 38% less memory — but only with clean view hierarchies

### Image optimization
```swift
// Downsampled thumbnails for lists
AsyncImage(url: url) { image in
    image.resizable()
        .aspectRatio(contentMode: .fill)
        .frame(width: 80, height: 80)
        .clipped()
} placeholder: {
    RoundedRectangle(cornerRadius: 8)
        .fill(.quaternary)
}
```

For large image collections, use `UIImage(contentsOfFile:)` with `prepareThumbnail(of:completionHandler:)` to downsample off the main thread.
