# Liquid Glass — Complete Reference

## Table of contents
1. What Liquid Glass is
2. Rendering layers
3. Variants (Regular, Clear, Identity)
4. Shape system (Fixed, Capsule, Concentric)
5. SwiftUI API
6. UIKit API
7. Grouping with GlassEffectContainer
8. Morphing and transitions
9. Tinting
10. Interaction feedback
11. Accessibility adaptations
12. Migration from iOS 18
13. Rules and anti-patterns

---

## 1. What Liquid Glass is

Liquid Glass is a translucent meta-material that bends and concentrates light in real time rather than scattering it (like the old frosted blur). It creates a physical glass appearance — highlights track light sources, shadows adapt to content, and touch causes inner illumination. It unifies all Apple platforms under one visual language.

Liquid Glass is for the **navigation layer only**: navigation bars, toolbars, tab bars, sidebars, floating action buttons, and system controls. It is never used for content (lists, cards, tables, media).

## 2. Rendering layers

Liquid Glass renders through six simultaneous layers:

| Layer | Behavior |
|-------|----------|
| **Lensing** | Bends background light through the element, providing definition while remaining transparent. This is the core property. |
| **Highlights** | Specular highlights respond to geometry, device motion (gyroscope), and light source position |
| **Adaptive shadows** | Shadow opacity increases over text areas (for separation) and decreases over solid light backgrounds |
| **Inner illumination** | On touch, light spreads from the fingertip outward to nearby glass elements |
| **Tint layer** | Continuously adapts color based on content underneath, shifting dynamic range for legibility |
| **Materialization** | Appearance/disappearance through gradual light modulation rather than opacity fading |

## 3. Variants

### Regular (default)
Full adaptive effects. Small glass elements (nav bars, tab bars) automatically flip between light and dark appearance based on underlying content. Use this for all standard UI chrome.

```swift
.glassEffect()                              // Regular, capsule shape
.glassEffect(.regular)                       // Explicit regular
.glassEffect(.regular, in: .capsule)         // Explicit shape
```

### Clear
Permanently more transparent, no adaptive light/dark flipping. Applies a dimming layer to content underneath. Use only over media-rich content where dimming is acceptable.

```swift
.glassEffect(.clear)
```

### Identity
No glass effect at all. Use for conditional toggling.

```swift
.glassEffect(showGlass ? .regular : .identity)
```

**Rule**: Never mix Regular and Clear variants in the same container or adjacent elements.

## 4. Shape system

### Fixed shapes
Constant corner radius regardless of size:
```swift
.glassEffect(.regular, in: .rect(cornerRadius: 16))
```

### Capsule shapes (primary choice)
Radius = half the container height. Most ergonomic for touch:
```swift
.glassEffect(.regular, in: .capsule)  // Default
```

### Concentric shapes
Inner radius = outer radius minus padding. Maintains visual harmony in nested elements:
```swift
.glassEffect(.regular, in: .concentricRectangle)
```

`ConcentricRectangle` is a new shape type in iOS 26 that matches the device's physical corner radius.

## 5. SwiftUI API

### Basic application
```swift
// Simple glass button
Button("Action") { }
    .glassEffect()

// Glass with custom shape
Button("Action") { }
    .glassEffect(.regular, in: .rect(cornerRadius: 12))

// Tinted glass
Button("Primary") { }
    .glassEffect(.regular.tint(.blue))

// Interactive glass (scaling, bounce, shimmer on press)
Button("Interactive") { }
    .glassEffect(.regular.interactive())

// Button styles
Button("Secondary") { }
    .buttonStyle(.glass)          // Translucent secondary

Button("Primary") { }
    .buttonStyle(.glassProminent) // Opaque primary
```

### Spacing control
```swift
.glassEffect(.regular, spacing: .custom(20)) // Custom spacing from other glass
.glassEffect(.regular, spacing: .connected)   // Elements visually merge
```

### Labels inside glass
For bar buttons and controls, use `Label` with SF Symbols:
```swift
Button { } label: {
    Label("Share", systemImage: "square.and.arrow.up")
}
.glassEffect()
```

## 6. UIKit API

```swift
// Create effect
let effect = UIGlassEffect()
effect.tintColor = .systemBlue
effect.isInteractive = true

// Apply to visual effect view
let effectView = UIVisualEffectView(effect: effect)
effectView.frame = button.bounds
button.insertSubview(effectView, at: 0)

// Clear variant
let clearEffect = UIGlassEffect()
clearEffect.variant = .clear
```

## 7. Grouping with GlassEffectContainer

Glass cannot sample other glass — overlapping glass elements create visual artifacts. Group related glass elements in a container:

### SwiftUI
```swift
GlassEffectContainer(spacing: 40) {
    Button("Undo") { }
        .glassEffect()
    Button("Redo") { }
        .glassEffect()
}
```

The container ensures elements know about each other, preventing overlap artifacts. The `spacing` parameter controls how close elements can be before they begin to merge.

### UIKit
```swift
let container = UIGlassContainerEffect()
container.spacing = 40
// Apply to parent view containing multiple glass elements
```

## 8. Morphing and transitions

Glass elements can smoothly morph between states using matched identifiers:

```swift
@Namespace private var glassNamespace

// Source
Button("Open") { showDetail = true }
    .glassEffectID("detailButton", in: glassNamespace)
    .glassEffect()

// Destination (the glass morphs from the button into this toolbar)
.toolbar {
    ToolbarItem {
        Button("Close") { showDetail = false }
            .glassEffectID("detailButton", in: glassNamespace)
    }
}
```

### Materialization (not fade)
Glass elements don't fade in/out. They appear by gradually modulating light bending:
- Appearing: Lensing strength increases from 0 to full
- Disappearing: Lensing strength decreases to 0
This preserves the optical physics of the material.

## 9. Tinting

Tinting applies your app's accent color to glass elements. The tint color generates a range of tones that map to the brightness of the content underneath — just like colored glass in reality.

```swift
// Accent-tinted glass
.glassEffect(.regular.tint(.accentColor))

// Custom tint
.glassEffect(.regular.tint(Color.indigo))
```

**Rules**:
- Tint only primary actions and key interactive elements
- Don't tint every glass element — if everything is tinted, nothing stands out
- System tinting works with both Light and Dark modes
- Tinted glass automatically adjusts contrast for legibility

## 10. Text readability in glass

Glass is translucent — effective contrast depends on what is behind it. Apple's rendering handles much of this automatically, but developers must understand the mechanics and test thoroughly.

### How the system ensures legibility
- **Adaptive shadows** increase opacity over text areas, creating separation between labels and background content
- **Tint layer** continuously shifts dynamic range based on underlying brightness, keeping text readable
- **Lensing** bends and concentrates background light rather than scattering it, providing definition without obscuring

These layers work together so that system controls (nav bars, tab bars, toolbars) are legible out of the box.

### Developer responsibilities
- **Use `Label` with SF Symbols** inside glass elements — they are optically optimized for glass legibility
- **Keep text short** — glass is for labels, icons, and short actions, not body text or paragraphs
- **Test over varied backgrounds** — glass contrast changes with underlying content. Verify legibility over both light images and dark solid backgrounds
- **Use `.scrollEdgeEffectStyle(.soft, for: .top)`** when scroll content passes under floating glass bars — this blurs content edges at the scroll boundary, preventing harsh text/image collisions with glass
- **Test with Increase Contrast** — glass becomes predominantly opaque, changing visual weight. Layouts designed around transparency may need spacing or padding adjustments
- **Test with Reduce Transparency** — glass becomes frostier. Ensure text remains legible and hierarchy is maintained when the translucent effect is diminished
- **Avoid placing glass over rapidly changing content** (video, heavy animation) — readability degrades and GPU cost increases

### Contrast and WCAG
System glass controls adapt automatically to meet contrast requirements. Custom glass elements do not get this guarantee — you must manually verify that text inside custom `.glassEffect()` views meets WCAG AA (4.5:1 normal text, 3:1 large text) across representative backgrounds.

## 11. Interaction feedback

When `isInteractive` is true (or using `.interactive()`):
1. **Press down**: Element scales slightly, inner illumination begins at touch point
2. **Light spread**: Illumination radiates to nearby glass elements in the same container
3. **Release**: Spring animation returns to resting state
4. **Shimmer**: Subtle light effect on release confirmation

This is automatic for `.buttonStyle(.glass)` and `.buttonStyle(.glassProminent)`.

## 11. Accessibility adaptations

### Reduce Transparency
Glass becomes frostier and more opaque, obscuring more background. At maximum settings, glass is essentially a solid surface. **You must test your layout with Reduce Transparency enabled** — ensure text remains legible and hierarchy is maintained.

```swift
@Environment(\.accessibilityReduceTransparency) var reduceTransparency
```

### Increase Contrast
Glass elements become predominantly solid black (Dark) or solid white (Light) with distinct contrasting borders. Tints become stronger and more saturated.

### Reduce Motion
Elastic/spring properties of glass are disabled. Materialization animations are simplified to basic opacity changes. Morphing transitions become crossfades.

**Developer impact**: Under Reduce Motion, Liquid Glass loses its characteristic motion — springs, materialization, and morphing are all simplified. Your UI must still function correctly when glass behaves as a simple translucent surface without dynamic motion effects. Don't rely on glass animation to convey essential information.

### Bold Text
Text inside glass elements becomes heavier. Ensure glass elements have enough padding to accommodate bolder, wider text without clipping.

## 12. Migration from iOS 18

### Automatic (just recompile with Xcode 26)
Standard UIKit/SwiftUI components automatically get Liquid Glass:
- UINavigationBar → floating glass buttons
- UITabBar → floating pill tab bar
- UIToolbar → glass toolbar
- UISearchBar → bottom-positioned glass search

### Remove to enable glass
Delete these customizations — they prevent glass from rendering:
```swift
// REMOVE these:
navigationBar.barTintColor = .white
navigationBar.backgroundColor = .systemBackground
let appearance = UINavigationBarAppearance()
appearance.configureWithOpaqueBackground() // blocks glass
navigationBar.standardAppearance = appearance

// REMOVE from sheets:
.presentationBackground(.ultraThinMaterial) // blocks glass styling
```

### Manual adoption for custom elements
For custom floating controls, overlays, or non-standard navigation:
```swift
// Before (iOS 18)
.background(.ultraThinMaterial)
.clipShape(RoundedRectangle(cornerRadius: 20))

// After (iOS 26)
.glassEffect(.regular, in: .capsule)
```

## 13. Rules and anti-patterns

### DO
- Use glass for navigation-layer elements (bars, toolbars, floating controls)
- Group adjacent glass elements with `GlassEffectContainer`
- Test with Reduce Transparency enabled
- Let small glass elements flip light/dark automatically
- Use capsule shapes for touch-friendly interfaces
- Use concentric shapes for nested elements
- Remove old bar appearance customizations
- Use `.scrollEdgeEffectStyle(.soft, for: .top)` when scroll content passes under floating glass bars — blurs content edges for readability

### DON'T
- Stack glass on glass (glass cannot sample glass)
- Use glass for content (lists, cards, images, text blocks)
- Mix Regular and Clear variants in the same visual group or adjacent elements
- Tint every glass element (breaks hierarchy)
- Add custom blur/material backgrounds behind glass bars
- Set explicit background colors on navigation bars
- Nest glass effects inside other glass effects
- Use glass purely for decoration with no interactive purpose — glass signals depth, feedback, or context changes
- Customize materialization timing — glass appearance/disappearance curves are system-managed and cannot be overridden
