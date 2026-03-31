# Liquid Glass on TV — Complete Reference

## Table of contents
1. How Liquid Glass works on Apple TV
2. Key differences from iOS
3. Variants
4. SwiftUI API
5. Tinting
6. GlassEffectContainer
7. Focus-driven effects
8. Migration from previous tvOS versions
9. Rules and anti-patterns

---

## 1. How Liquid Glass works on Apple TV

Liquid Glass is the unified material system introduced across all Apple platforms in 2025. On Apple TV, it appears on:
- **Tab bar** (top of screen)
- **Navigation bars**
- **Control Center**
- **Player transport controls**
- **System alerts and dialogs**
- **Sidebar overlays**
- **Custom floating UI elements**

The same six rendering layers apply:

| Layer | Behavior on TV |
|-------|---------------|
| **Lensing** | Bends background through the element — more pronounced on large screens |
| **Highlights** | Specular highlights respond to ambient content (no gyroscope tilt on TV) |
| **Adaptive shadows** | Shadow intensity increases over text for separation |
| **Inner illumination** | Triggered by **focus** (not touch — no touchscreen on TV) |
| **Tint layer** | Adapts color based on content underneath |
| **Materialization** | Appearance/disappearance through light modulation |

## 2. Key differences from iOS

| Aspect | iOS 26 | tvOS 26 |
|--------|--------|---------|
| **Illumination trigger** | Touch (fingertip on screen) | Focus (focus engine highlights element) |
| **Tab bar position** | Bottom, floating pill | Top, floating bar |
| **Gyroscope effects** | Highlights shift with device tilt | Static (no gyroscope on Apple TV box) |
| **Material scale** | Thin glass on small screens | Thicker, more prominent glass on large screens |
| **Scroll behavior** | Tab bar minimizes on scroll | Tab bar hides when content has focus |
| **Context** | Many glass elements (bars, sheets, buttons) | Fewer glass surfaces (TV chrome is minimal) |

## 3. Variants

### Regular (default)
Full adaptive effects. Automatically adjusts between light/dark based on underlying content. Use for all standard navigation chrome.

```swift
.glassEffect()                              // Regular, capsule shape
.glassEffect(.regular)                       // Explicit regular
.glassEffect(.regular, in: .capsule)         // Explicit shape
.glassEffect(.regular, in: .rect(cornerRadius: 16))
.glassEffect(.regular, in: .circle)
```

### Clear
Transparent with minimal lensing. Used when glass needs to be nearly invisible but still provide grouping. Requires dimming the content behind it.

```swift
.glassEffect(.clear)
.glassEffect(.clear, in: .capsule)
```

### Identity
No glass effect. Used to opt out of glass in a `GlassEffectContainer` while keeping layout participation.

```swift
.glassEffect(.identity)
```

## 4. SwiftUI API

### Basic glass on custom elements

```swift
// Floating action bar with glass
HStack(spacing: 30) {
    Button("Play") { play() }
    Button("Add to List") { addToList() }
    Button("More Info") { showInfo() }
}
.padding(.horizontal, 40)
.padding(.vertical, 20)
.glassEffect(.regular, in: .capsule)

// Glass button styles
Button("Primary Action") { }
    .buttonStyle(.glassProminent)  // Tinted glass (primary)

Button("Secondary Action") { }
    .buttonStyle(.glass)           // Clear glass (secondary)
```

### Glass shapes

```swift
.glassEffect(.regular, in: .capsule)              // Pill shape
.glassEffect(.regular, in: .circle)                // Circle
.glassEffect(.regular, in: .rect(cornerRadius: 16)) // Rounded rect
.glassEffect(.regular, in: .rect)                  // Sharp rect (avoid)
.glassEffect(.regular, in: .ellipse)               // Ellipse

// Concentric shapes — inner glass follows outer glass curvature
.glassEffect(.regular, in: .rect(cornerRadius: 20).concentric())
```

## 5. Tinting

```swift
// Tinted glass — use app accent color sparingly
.glassEffect(.regular.tint(.blue))              // Blue tint
.glassEffect(.regular.tint(.accentColor))        // App accent tint
.glassEffect(.regular.tint(.red))               // Destructive tint

// Tinting rules on TV:
// - Tint ONLY primary interactive elements
// - Secondary buttons: no tint (plain .glassEffect())
// - Never tint the entire tab bar or navigation bar
// - Tint should draw focus to the single most important action
```

## 6. GlassEffectContainer

When multiple glass elements are adjacent, they must be in a `GlassEffectContainer` so their light effects coordinate and they don't create visual artifacts.

```swift
// Adjacent glass buttons
GlassEffectContainer(spacing: 20) {
    Button("Play") { play() }
        .glassEffect(.regular.tint(.blue), in: .capsule)

    Button("Trailer") { showTrailer() }
        .glassEffect(.regular, in: .capsule)

    Button("Add to List") { addToList() }
        .glassEffect(.regular, in: .capsule)
}

// Glass card group
GlassEffectContainer(spacing: 40) {
    ForEach(quickActions) { action in
        ActionButton(action: action)
            .glassEffect(.regular, in: .rect(cornerRadius: 16))
    }
}
```

## 7. Focus-driven effects

On tvOS, glass responds to **focus** instead of touch:

```swift
struct GlassActionButton: View {
    let title: String
    let icon: String
    let action: () -> Void
    @Environment(\.isFocused) var isFocused

    var body: some View {
        Button(action: action) {
            Label(title, systemImage: icon)
                .font(.headline)
                .padding(.horizontal, 30)
                .padding(.vertical, 16)
        }
        .buttonStyle(.glass)
        // Glass automatically brightens inner illumination when focused
        // The focus engine handles this — no manual coding needed

        // For custom glass elements (not buttons):
        // .glassEffect(.regular, in: .capsule)
        // The system applies focus highlight automatically if .focusable()
    }
}
```

### Materialization (appear/disappear)

```swift
// Glass elements materialize (light modulation), never fade
struct FloatingBar: View {
    @State private var isVisible = false

    var body: some View {
        if isVisible {
            HStack { /* controls */ }
                .glassEffect(.regular, in: .capsule)
                .transition(.opacity)  // System applies glass materialization automatically
        }
    }
}
```

## 8. Migration from previous tvOS versions

### What to remove
```swift
// REMOVE — old material-based approaches
.background(.ultraThinMaterial)        // → .glassEffect()
.background(.thinMaterial)             // → .glassEffect(.clear)
.background(.regularMaterial)          // → .glassEffect(.regular)

// REMOVE — custom blur effects
VisualEffectView(effect: UIBlurEffect(style: .dark))  // → .glassEffect()

// REMOVE — manual bar appearances
UINavigationBarAppearance()            // Glass is automatic on recompile
UITabBarAppearance()                   // Glass is automatic on recompile
```

### What to add
```swift
// Standard tab bar — glass is automatic in tvOS 26
TabView { ... }

// Custom floating elements — add glass explicitly
.glassEffect(.regular, in: .capsule)

// Glass buttons
.buttonStyle(.glass)
.buttonStyle(.glassProminent)

// Group adjacent glass
GlassEffectContainer(spacing: 20) { ... }
```

## 9. Rules and anti-patterns

### DO:
- Use `.glassEffect()` for custom floating navigation elements
- Use `.buttonStyle(.glass)` / `.buttonStyle(.glassProminent)` for glass buttons
- Group adjacent glass elements in `GlassEffectContainer`
- Let standard components (TabView, NavigationStack) get glass automatically
- Tint only the primary action with accent color

### DON'T:
- **Don't stack glass on glass** — glass cannot sample other glass. Use `GlassEffectContainer`.
- **Don't use glass for content** — glass is navigation layer only. Content (posters, lists, text) stays opaque.
- **Don't tint everything** — reserve tint for primary actions. If everything is tinted, nothing stands out.
- **Don't mix Regular and Clear** — pick one per context.
- **Don't use `.ultraThinMaterial` in new code** — use `.glassEffect()` instead.
- **Don't manually set bar appearances** — glass is applied automatically to system bars.
- **Don't create glass sandwiches** — stacking multiple glass overlays destroys hierarchy.
- **Don't apply glass to large content areas** — glass is for small, floating chrome elements.
- **Don't fade glass in/out** — let the system handle materialization.
