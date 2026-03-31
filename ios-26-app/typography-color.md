# Typography, Color, and Visual Style

## Table of contents
1. Font families
2. Text styles and Dynamic Type
3. Typography best practices
4. Color system
5. The 80/20 accent color rule
6. Semantic colors
7. Dark mode
8. Tinting with Liquid Glass
9. Vibrancy and materials
10. App icon design (iOS 26)
11. SF Symbols usage

---

## 1. Font families

### SF Pro (primary system font)
- 9 weights: Ultralight, Thin, Light, Regular, Medium, Semibold, Bold, Heavy, Black
- Automatic optical size: **Text** variant below 20pt (wider spacing, open forms), **Display** variant at 20pt+ (tighter spacing)
- Supports 150+ languages including Latin, Cyrillic, Greek, Arabic, Hebrew, Thai, Devanagari, CJK
- Dynamic tracking: automatically tightens at larger Display sizes, loosens at smaller Text sizes

### SF Pro Rounded
Same weights as SF Pro but with rounded stroke terminals. Use for softer, friendlier contexts (activity rings, badges, casual interfaces). Don't mix with SF Pro in the same view hierarchy.

### SF Mono
6 weights (Light through Heavy). Fixed-width for code, terminal output, data tables with aligned columns. Includes programming ligatures.

### New York
Apple's serif companion. 4 optical sizes (Small, Medium, Large, Extra Large). Available in 6 weights. Use for editorial/reading contexts, article bodies, literary content. Pairs well with SF Pro for headings.

### Using custom fonts
```swift
// Register in Info.plist under "Fonts provided by application"
Text("Custom")
    .font(.custom("MyFont-Regular", size: 17, relativeTo: .body))
// relativeTo: enables Dynamic Type scaling relative to that style
```

Always use `relativeTo:` to maintain Dynamic Type support with custom fonts. Use `UIFontMetrics` in UIKit:
```swift
let customFont = UIFont(name: "MyFont-Regular", size: 17)!
let scaledFont = UIFontMetrics(forTextStyle: .body).scaledFont(for: customFont)
label.font = scaledFont
label.adjustsFontForContentSizeCategory = true
```

## 2. Text styles and Dynamic Type

### Complete type scale at default (Large) content size

| Style | SwiftUI | Weight | Size | Leading | Tracking |
|-------|---------|--------|------|---------|----------|
| Large Title | `.largeTitle` | Regular | 34pt | 41pt | 0.37 |
| Title 1 | `.title` | Regular | 28pt | 34pt | 0.36 |
| Title 2 | `.title2` | Regular | 22pt | 28pt | 0.35 |
| Title 3 | `.title3` | Regular | 20pt | 25pt | 0.38 |
| Headline | `.headline` | **Semibold** | 17pt | 22pt | -0.41 |
| Body | `.body` | Regular | 17pt | 22pt | -0.41 |
| Callout | `.callout` | Regular | 16pt | 21pt | -0.32 |
| Subheadline | `.subheadline` | Regular | 15pt | 20pt | -0.24 |
| Footnote | `.footnote` | Regular | 13pt | 18pt | -0.08 |
| Caption 1 | `.caption` | Regular | 12pt | 16pt | 0 |
| Caption 2 | `.caption2` | Regular | 11pt | 13pt | 0.07 |

Headline is the only style that defaults to **Semibold**. All others are Regular. Apply weight modifiers for emphasis: `.fontWeight(.bold)`, `.fontWeight(.semibold)`.

### Dynamic Type sizes across all 12 content size categories

**Body font size at each category:**

| Category | Body (pt) | Large Title (pt) |
|----------|-----------|------------------|
| xSmall | 14 | 31 |
| Small | 15 | 32 |
| Medium | 16 | 33 |
| **Large (default)** | **17** | **34** |
| xLarge | 19 | 36 |
| xxLarge | 21 | 38 |
| xxxLarge | 23 | 40 |
| AX1 | 28 | 44 |
| AX2 | 33 | 48 |
| AX3 | 40 | 52 |
| AX4 | 47 | 56 |
| AX5 | 53 | 60 |

**Important behaviors:**
- Caption 2 has a hard minimum of **11pt** — it never scales below this
- Headline caps at **23pt** (xxxLarge) and doesn't increase in accessibility sizes
- At AX sizes, layouts should reflow (e.g., side-by-side labels stack vertically)

### @ScaledMetric for non-text dimensions
```swift
@ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24

Image(systemName: "star")
    .frame(width: iconSize, height: iconSize)
```

Scales spacing, icon sizes, and other dimensions proportionally with Dynamic Type.

### Limiting Dynamic Type range (use sparingly)
```swift
.dynamicTypeSize(.small ... .accessibility3)
// Only when truly necessary — prefer supporting ALL sizes
```

## 3. Typography best practices

### Hierarchy through weight, not size
Use one or two sizes per screen section, varying weight for hierarchy:
```swift
VStack(alignment: .leading, spacing: 4) {
    Text(item.title)
        .font(.headline)        // 17pt Semibold
    Text(item.subtitle)
        .font(.subheadline)     // 15pt Regular
        .foregroundStyle(.secondary)
}
```

### Line limits and truncation
```swift
Text(longText)
    .lineLimit(3)
    .truncationMode(.tail)

// iOS 16+: expandable
Text(longText)
    .lineLimit(3, reservesSpace: true)

// iOS 16+: range
Text(longText)
    .lineLimit(2...5)
```

### Text alignment
- **Left-aligned** (leading) for body text, list items, form labels — the default
- **Center-aligned** for titles in onboarding, empty states, and hero content
- **Right-aligned** only for numeric data in tables/financial contexts
- iOS 26 shifts alerts to **left-aligned text** for improved readability

### Minimum text size
Apple recommends **11pt** minimum. Never go smaller. For dense data tables, 12pt (Caption 1) is the practical minimum.

### iOS 26 typography changes
- List section titles changed from ALL CAPS to **sentence case** with larger text
- Alerts use **left-aligned, bolder** text
- Navigation bars support optional left-aligned titles with subtitle
- SF Pro Rounded refined for small-size readability under Liquid Glass

## 4. Color system

### System colors
iOS provides a set of dynamic system colors that automatically adapt to Light/Dark mode and accessibility settings:

**Semantic background colors:**
- `.systemBackground` — primary background (white/black)
- `.secondarySystemBackground` — grouped content background
- `.tertiarySystemBackground` — content within grouped
- `.systemGroupedBackground` — grouped table background
- `.secondarySystemGroupedBackground` — cells within grouped
- `.tertiarySystemGroupedBackground` — nested content within cells

**Semantic label colors:**
- `.label` — primary text (black/white)
- `.secondaryLabel` — secondary text (gray)
- `.tertiaryLabel` — disabled/placeholder text
- `.quaternaryLabel` — extremely subtle text

**Semantic fill colors (for controls):**
- `.systemFill` — thin overlays (e.g., text field background)
- `.secondarySystemFill` — medium fills
- `.tertiarySystemFill` — thick fills
- `.quaternarySystemFill` — heaviest fills

**Accent colors:**
`.systemRed`, `.systemOrange`, `.systemYellow`, `.systemGreen`, `.systemMint`, `.systemTeal`, `.systemCyan`, `.systemBlue`, `.systemIndigo`, `.systemPurple`, `.systemPink`, `.systemBrown`

All 12 accent colors are tuned for both Light and Dark modes and meet WCAG AA contrast on their respective backgrounds.

### SwiftUI color usage
```swift
Text("Primary").foregroundStyle(.primary)
Text("Secondary").foregroundStyle(.secondary)
Text("Tertiary").foregroundStyle(.tertiary)
Text("Accent").foregroundStyle(.accentColor)

// Hierarchical foreground
Label("Item", systemImage: "star")
    .foregroundStyle(.blue, .blue.opacity(0.3))  // Primary, secondary
```

## 5. The 80/20 accent color rule

This is a core design principle that predates iOS 26 but is amplified by Liquid Glass:

- **80% neutral/system colors** — backgrounds, text, dividers, non-interactive chrome
- **20% accent color** — primary actions, key interactive elements, selected states

### Why this matters more in iOS 26
Liquid Glass tinting uses your accent color to generate dynamic glass tones. If you tint every glass element, the navigation layer becomes a uniform color blob. Reserve tinting for:
- Primary action buttons (`.glassProminent` with tint)
- Active tab in tab bar
- Toggle/switch active state
- Navigation bar primary action
- Selected list items

### Implementation
```swift
// App-wide accent
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .tint(.indigo)  // Sets app-wide accent
        }
    }
}

// Local override
Button("Special") { }
    .tint(.orange)
```

## 6. Semantic colors in SwiftUI

Always prefer semantic colors over hardcoded values:

```swift
// DO
Text("Hello").foregroundStyle(.primary)
VStack { }.background(.background)

// DON'T
Text("Hello").foregroundColor(.black)  // Breaks in dark mode
VStack { }.background(.white)          // Breaks in dark mode
```

### Creating adaptive colors
```swift
// Asset catalog: define Light and Dark variants
// Or programmatically:
extension Color {
    static let cardBackground = Color(
        uiColor: UIColor { traits in
            traits.userInterfaceStyle == .dark
                ? UIColor(red: 0.15, green: 0.15, blue: 0.15, alpha: 1)
                : UIColor(red: 0.95, green: 0.95, blue: 0.95, alpha: 1)
        }
    )
}
```

## 7. Dark mode

### Requirements
- Every app must support both Light and Dark modes
- Use semantic colors to get automatic adaptation
- Test all screens in both modes
- Never assume white background or black text

### Elevated surfaces in dark mode
Dark mode uses two background levels:
- **Base**: `.systemBackground` — pure black on OLED
- **Elevated**: `.secondarySystemBackground` — slightly lighter, for sheets and overlays

Liquid Glass automatically handles elevation — glass on dark backgrounds is subtly lighter than glass on light backgrounds.

### Image handling in dark mode
```swift
// Different images per mode
Image(colorScheme == .dark ? "hero-dark" : "hero-light")

// Symbol rendering adapts automatically
Image(systemName: "sun.max")
    .symbolRenderingMode(.hierarchical)
```

### Checking appearance
```swift
@Environment(\.colorScheme) var colorScheme

if colorScheme == .dark {
    // Dark-specific adjustments
}
```

## 8. Tinting with Liquid Glass

### How tinting works in iOS 26
When you apply a tint to Liquid Glass, the system generates a range of color tones mapped to the brightness of the content behind the glass — like colored glass in reality. Bright backgrounds produce lighter tints; dark backgrounds produce deeper tints.

```swift
.glassEffect(.regular.tint(.blue))
```

### Rules
- Tint communicates purpose — only tint elements that represent primary actions or active states
- Don't tint every glass element — this destroys hierarchy
- If you want color in your app, put it in the **content layer** (images, illustrations, data visualizations) not in the glass navigation layer
- Tinting works in both Light and Dark modes
- Respect Increase Contrast: tints become more saturated and opaque

### Contrast over glass
Glass is translucent — effective contrast depends on the content behind it. System glass controls (nav bars, tab bars, toolbars) adapt automatically via adaptive shadows and tint layer shifting. Custom glass elements using `.glassEffect()` do **not** get this guarantee. You must manually verify that text inside custom glass views meets WCAG AA (4.5:1 normal text, 3:1 large text) across representative backgrounds — test over both light images and dark solid content.

## 9. Vibrancy and materials (unchanged from iOS 18)

### System materials for non-glass contexts
```swift
.background(.ultraThinMaterial)
.background(.thinMaterial)
.background(.regularMaterial)
.background(.thickMaterial)
.background(.ultraThickMaterial)
```

These frosted-blur materials are still available and appropriate for content layers that need translucency but aren't navigation elements (e.g., overlaid text on images). Don't use them for navigation bars or toolbars — those should use Liquid Glass now.

### Vibrancy
Labels and symbols placed over materials automatically gain vibrancy — they appear brighter and more legible. Use `.foregroundStyle(.primary)` over materials for best results.

## 10. App icon design (iOS 26)

iOS 26 introduces multi-layer icons with glass effects:

- **Default**: Liquid Glass layered icon with shimmer and parallax
- **Dark mode**: Light/Dark/Auto sub-options
- **Clear**: New translucent see-through glass variant
- **Tinted**: Now supports both Light AND Dark (was dark-only in iOS 18)
  - Color picker with saturation slider
  - Auto-match to iPhone hardware color
  - MagSafe case color matching via camera

### Icon specs (unchanged)
- App icon: **1024×1024px** single asset (system generates all sizes)
- Corner radius: system-applied, ~17.5% of icon size
- No transparency in the base layer
- Don't include Apple hardware in icons

## 11. SF Symbols usage

### 6,900+ symbols in SF Symbols 7

### Rendering modes
```swift
Image(systemName: "heart.fill")
    .symbolRenderingMode(.monochrome)    // Single color
    .symbolRenderingMode(.hierarchical)  // Primary + secondary opacity
    .symbolRenderingMode(.palette)       // Custom color per layer
    .symbolRenderingMode(.multicolor)    // Apple-defined colors
    .symbolColorRenderingMode(.gradient) // NEW iOS 26: gradient from single color
```

### Palette mode (multi-color custom)
```swift
Image(systemName: "cloud.sun.fill")
    .symbolRenderingMode(.palette)
    .foregroundStyle(.blue, .yellow)
```

### Symbol effects
```swift
// Bounce on appear
Image(systemName: "bell").symbolEffect(.bounce, value: notificationCount)

// Pulse continuously
Image(systemName: "antenna.radiowaves.left.and.right").symbolEffect(.pulse)

// Variable color (progress)
Image(systemName: "wifi").symbolEffect(.variableColor.iterative)

// Scale
Image(systemName: "star").symbolEffect(.scale.up, isActive: isFavorite)

// Replace with transition
Image(systemName: isPlaying ? "pause.fill" : "play.fill")
    .contentTransition(.symbolEffect(.replace))

// NEW iOS 26: Draw on/off
Image(systemName: "checkmark.circle")
    .symbolEffect(.drawOn, isActive: isComplete)
Image(systemName: "xmark")
    .symbolEffect(.drawOff, isActive: isDismissed)
```

### Draw animation playback modes (iOS 26)
- **By Layer**: Staggered animation per symbol layer
- **Whole Symbol**: All paths animate simultaneously
- **Individually**: Sequential path-by-path (new)

### Symbol weights
Match symbol weight to surrounding text:
```swift
Image(systemName: "gearshape")
    .fontWeight(.semibold)  // Match Headline weight
```

### Symbol sizes relative to text
```swift
Label("Settings", systemImage: "gearshape")
    .font(.body)
    .imageScale(.medium)  // .small, .medium, .large
```

SF Symbols are designed to vertically center with their corresponding text style at `.medium` image scale. Use `.large` for standalone icons and `.small` for inline badge-like indicators.
