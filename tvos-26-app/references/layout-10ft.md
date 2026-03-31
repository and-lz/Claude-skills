# Layout & 10-Foot UI — Complete Reference

## Table of contents
1. Screen dimensions and resolution
2. Safe areas and overscan
3. Typography scale
4. Spacing system
5. Card and poster sizing
6. Focus target sizing
7. Shelf and grid layouts
8. Color and contrast for TV
9. Rules and anti-patterns

---

## 1. Screen dimensions and resolution

| Property | Value |
|----------|-------|
| Logical resolution | **1920 × 1080 points** |
| Scale factor | **1x** (unlike iOS 2x/3x) |
| Apple TV 4K output | 3840 × 2160 pixels (4K), downscaled to 1920x1080 logical |
| Apple TV HD output | 1920 × 1080 pixels (1080p) |
| Aspect ratio | 16:9 |
| Color space | sRGB (SDR) or Display P3 (HDR) |

**Key difference from iOS**: tvOS uses 1x scale factor. A 100pt view is 100 pixels on screen. This means assets need to be provided at 1x resolution (or use SF Symbols / vector assets).

```swift
// tvOS layout — 1920x1080 logical points
struct ContentView: View {
    var body: some View {
        GeometryReader { geo in
            // geo.size = CGSize(width: 1920, height: 1080)
            VStack {
                // Full-width hero: ~1920 x 600pt
                HeroBanner()
                    .frame(height: 600)

                // Content shelves below
                ShelfSection()
            }
        }
    }
}
```

## 2. Safe areas and overscan

**Overscan**: Many TVs crop the outer edges of the display. Apple TV compensates with safe area insets.

| Edge | Safe area inset |
|------|----------------|
| Top | **60pt** |
| Bottom | **60pt** |
| Left | **60pt** |
| Right | **60pt** |

**Usable area**: 1800 × 960 points (after 60pt insets on all edges)

```swift
// ALWAYS use safe area — never hardcode insets
struct BrowseView: View {
    var body: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: 50) {
                // Content automatically respects safe area
                Text("Trending")
                    .font(.title)
                    .padding(.leading)  // Additional padding within safe area

                ShelfRow(items: trending)
            }
        }
        // DO NOT: .padding(60) — safe area handles this
        // DO NOT: .ignoresSafeArea() on content
    }
}

// Exception: background images/video CAN ignore safe area
struct HeroBanner: View {
    var body: some View {
        AsyncImage(url: backdropURL)
            .ignoresSafeArea()  // OK for backgrounds only
            .overlay(alignment: .bottomLeading) {
                // Text/buttons MUST be within safe area
                VStack(alignment: .leading) {
                    Text(title).font(.largeTitle)
                    Text(subtitle).font(.headline)
                }
                .padding(.leading, 80)  // Well within safe area
                .padding(.bottom, 40)
            }
    }
}
```

## 3. Typography scale

tvOS text sizes are significantly larger than iOS to ensure readability at 10 feet (~3 meters).

| Text Style | tvOS Size | iOS Size | Usage |
|------------|-----------|----------|-------|
| `.largeTitle` | ~76pt | ~34pt | Screen titles, hero text |
| `.title` | ~48pt | ~28pt | Section headers |
| `.title2` | ~38pt | ~22pt | Subsection headers |
| `.title3` | ~31pt | ~20pt | Emphasized body |
| `.headline` | ~29pt (bold) | ~17pt | Row titles, bold labels |
| `.body` | ~29pt | ~17pt | Primary content text |
| `.callout` | ~26pt | ~16pt | Secondary descriptions |
| `.subheadline` | ~24pt | ~15pt | Metadata, timestamps |
| `.footnote` | ~21pt | ~13pt | Tertiary info, legal |
| `.caption` | ~19pt | ~12pt | Minimum readable text |
| `.caption2` | ~17pt | ~11pt | Absolute minimum (avoid) |

```swift
// ALWAYS use text styles — never hardcode sizes
Text("Featured Movies")
    .font(.title)              // ~48pt on tvOS

Text("2025 · Action · 2h 15m")
    .font(.subheadline)        // ~24pt on tvOS
    .foregroundStyle(.secondary)

// NEVER do this:
Text("Title").font(.system(size: 17))  // Too small for TV!
```

**Dynamic Type on tvOS**: tvOS does NOT support Dynamic Type size adjustment (there's no accessibility text size slider on Apple TV). Text styles still map to fixed sizes optimized for 10-foot viewing. Always use text styles for future-proofing.

## 4. Spacing system

| Spacing | Value | Usage |
|---------|-------|-------|
| Micro | 8pt | Tight internal padding |
| Small | 16pt | Related element gap |
| Medium | 24pt | Section internal padding |
| Standard | 40pt | Between cards in a shelf |
| Large | 50pt | Between shelf rows |
| Section | 60-80pt | Between major sections |
| Extra large | 100pt+ | Hero to first shelf gap |

```swift
// Shelf row with proper spacing
ScrollView(.horizontal, showsIndicators: false) {
    LazyHStack(spacing: 40) {  // 40pt between cards
        ForEach(items) { item in
            PosterCard(item: item)
        }
    }
    .padding(.horizontal)  // Respect safe area
}

// Section spacing
LazyVStack(spacing: 50) {  // 50pt between shelves
    ForEach(sections) { section in
        ShelfSection(section: section)
    }
}
```

## 5. Card and poster sizing

### Standard poster sizes

| Aspect Ratio | Name | Dimensions | Usage |
|--------------|------|------------|-------|
| 2:3 | Portrait | 250 × 375pt | Movie posters, TV show art |
| 16:9 | Landscape | 350 × 197pt | Episode thumbnails, screenshots |
| 1:1 | Square | 250 × 250pt | Album art, app icons |
| 3:4 | Tall portrait | 250 × 333pt | Book covers |
| 2.39:1 | Cinematic | 450 × 188pt | Wide hero banners |

### Focus scaling

| State | Scale | Shadow |
|-------|-------|--------|
| Unfocused | 1.0x | Radius 5, opacity 0.2 |
| Focused | 1.05-1.10x | Radius 15-25, opacity 0.4-0.6 |

```swift
struct MoviePoster: View {
    let movie: Movie
    @Environment(\.isFocused) var isFocused

    var body: some View {
        VStack(spacing: 12) {
            AsyncImage(url: movie.posterURL) { image in
                image.resizable().aspectRatio(2/3, contentMode: .fill)
            } placeholder: {
                RoundedRectangle(cornerRadius: 12)
                    .fill(.quaternary)
            }
            .frame(width: 250, height: 375)
            .clipShape(RoundedRectangle(cornerRadius: 12))
            .scaleEffect(isFocused ? 1.08 : 1.0)
            .shadow(
                color: .black.opacity(isFocused ? 0.5 : 0.2),
                radius: isFocused ? 20 : 5,
                y: isFocused ? 15 : 3
            )

            Text(movie.title)
                .font(.callout)
                .lineLimit(2)
                .multilineTextAlignment(.center)
                .opacity(isFocused ? 1.0 : 0.7)
        }
        .frame(width: 250)
        .animation(.spring(response: 0.35, dampingFraction: 0.7), value: isFocused)
    }
}
```

### Hero banner sizing

```swift
// Full-width hero: spans safe area width, ~50-60% of screen height
struct HeroBanner: View {
    let item: FeaturedItem

    var body: some View {
        ZStack(alignment: .bottomLeading) {
            AsyncImage(url: item.backdropURL) { image in
                image.resizable().aspectRatio(contentMode: .fill)
            } placeholder: {
                Rectangle().fill(.quaternary)
            }
            .frame(height: 600)  // ~55% of 1080pt
            .clipped()

            // Gradient overlay for text legibility
            LinearGradient(
                colors: [.clear, .black.opacity(0.8)],
                startPoint: .center,
                endPoint: .bottom
            )

            // Metadata — within safe area
            VStack(alignment: .leading, spacing: 8) {
                Text(item.title)
                    .font(.largeTitle).bold()
                Text(item.subtitle)
                    .font(.headline)
                    .foregroundStyle(.secondary)

                HStack(spacing: 20) {
                    Button("Play") { play(item) }
                        .buttonStyle(.borderedProminent)
                    Button("More Info") { showDetail(item) }
                        .buttonStyle(.bordered)
                }
            }
            .padding(.leading, 80)
            .padding(.bottom, 60)
        }
    }
}
```

## 6. Focus target sizing

| Element | Minimum size | Recommended |
|---------|-------------|-------------|
| Focusable button | 66 × 66pt | 80 × 66pt+ |
| Poster card | 150 × 200pt | 250 × 375pt |
| List row | full width × 66pt | full width × 80pt |
| Icon button | 66 × 66pt | 80 × 80pt |
| Text field | 300 × 60pt | 400 × 66pt |

```swift
// Minimum focusable button
Button("Action") { }
    .frame(minWidth: 200, minHeight: 66)

// Icon button
Button {
    toggleFavorite()
} label: {
    Image(systemName: "heart")
        .font(.title2)
}
.frame(width: 80, height: 80)
```

## 7. Shelf and grid layouts

### Horizontal shelf (most common pattern)

```swift
struct ShelfRow: View {
    let title: String
    let items: [ContentItem]

    var body: some View {
        VStack(alignment: .leading, spacing: 16) {
            Text(title)
                .font(.title2)
                .padding(.leading)

            ScrollView(.horizontal, showsIndicators: false) {
                LazyHStack(spacing: 40) {
                    ForEach(items) { item in
                        NavigationLink(value: item) {
                            PosterCard(item: item)
                        }
                        .buttonStyle(.card)  // tvOS card button style
                    }
                }
                .padding(.horizontal)
                .padding(.vertical, 20)  // Space for focus scale
            }
            .focusSection()
        }
    }
}
```

### Grid layout

```swift
struct ContentGrid: View {
    let items: [ContentItem]

    // tvOS grid: 5-7 columns for portrait posters
    let columns = [
        GridItem(.adaptive(minimum: 250, maximum: 300), spacing: 40)
    ]

    var body: some View {
        ScrollView {
            LazyVGrid(columns: columns, spacing: 50) {
                ForEach(items) { item in
                    NavigationLink(value: item) {
                        PosterCard(item: item)
                    }
                    .buttonStyle(.card)
                }
            }
            .padding(.horizontal)
            .padding(.vertical, 20)
        }
    }
}
```

### Items per row guidelines

| Poster type | Items visible per row | Spacing |
|-------------|----------------------|---------|
| Portrait (250pt) | 5-6 | 40pt |
| Landscape (350pt) | 4-5 | 40pt |
| Square (250pt) | 5-6 | 40pt |
| Wide banner (450pt) | 3-4 | 40pt |

## 8. Color and contrast for TV

**TV viewing challenges:**
- Room lighting varies dramatically (dark theater to bright living room)
- LCD/OLED differences affect black levels
- Color temperature varies by TV brand
- Users sit 6-15 feet away

### Guidelines

```swift
// Use semantic colors — they adapt automatically
Text("Title").foregroundStyle(.primary)       // High contrast
Text("Subtitle").foregroundStyle(.secondary)  // Medium contrast
Text("Metadata").foregroundStyle(.tertiary)   // Lower contrast

// Background: prefer deep darks, not pure black
Color(white: 0.08)  // Slightly off-black for OLED-friendly dark mode
Color(white: 0.12)  // Card backgrounds

// Accent colors: bold and saturated for TV
// Muted tones disappear on TV at distance
Color.blue           // System blue — tested for TV readability
Color.red            // Warnings/destructive
Color.green          // Success/confirmation

// NEVER use:
Color(white: 0.5)    // Low-contrast gray text — unreadable at distance
Color.gray.opacity(0.3) // Subtle tints vanish on TV
```

### Contrast requirements
- **Primary text on dark background**: minimum 7:1 ratio (higher than iOS's 4.5:1)
- **Secondary text**: minimum 4.5:1 ratio
- **Interactive elements**: must be distinguishable from static content when unfocused
- **Focused elements**: must have dramatically different appearance from unfocused

## 9. Rules and anti-patterns

### DO:
- Use 1920x1080 as your design canvas
- Respect 60pt safe area insets on ALL edges
- Use system text styles (never hardcode font sizes)
- Space cards 40pt apart in shelves
- Scale focused elements 1.05-1.1x
- Use high-contrast colors for text
- Test on actual TVs in different lighting conditions
- Provide 1x assets (or use SF Symbols / vectors)

### DON'T:
- Don't use body text smaller than 29pt
- Don't place important content in the outer 60pt (overscan zone)
- Don't pack more than 6-7 items in a visible shelf row
- Don't use thin/light font weights for body text
- Don't use low-contrast or muted colors for primary text
- Don't ignore the fact that tvOS is 1x (not 2x/3x)
- Don't create layouts narrower than the full screen (no iPhone-style narrow columns)
- Don't use iOS sizing (44pt touch targets are too small for TV)
