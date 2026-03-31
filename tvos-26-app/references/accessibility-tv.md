# Accessibility on tvOS — Complete Reference

## Table of contents
1. Overview — what's available on tvOS
2. VoiceOver on tvOS
3. Dynamic Type
4. Closed Captions and Audio Descriptions
5. Reduce Motion / Transparency / Contrast
6. Focus and VoiceOver interaction
7. Accessibility identifiers for UI testing
8. Accessibility testing checklist

---

## 1. Overview — what's available on tvOS

Apple TV supports the same core accessibility technologies as iOS, adapted for the 10-foot, focus-driven, lean-back experience.

**Available on tvOS:**
- **VoiceOver** — screen reader navigated via Siri Remote
- **Dynamic Type** — user-adjustable text scaling
- **Increase Contrast** — stronger borders, more opaque glass surfaces
- **Reduce Motion** — disables parallax and spring animations
- **Reduce Transparency** — replaces Liquid Glass with solid surfaces
- **Bold Text** — heavier font weights system-wide
- **Closed Captions / SDH** — for video content in AVPlayerViewController
- **Audio Descriptions (AD)** — for video content
- **Switch Control** — focus-based navigation for motor impairments

**Not available on tvOS (differs from iOS):**
- Voice Control (no app navigation by voice — Siri is separate)
- AssistiveTouch (no touchscreen)
- Braille display support
- Smart Invert Colors (not applicable to TV context)

---

## 2. VoiceOver on tvOS

VoiceOver on Apple TV is navigated with Siri Remote gestures. The system reads the currently focused element — so correct focus order = correct VoiceOver order.

### Siri Remote VoiceOver gestures

| Action | Gesture |
|--------|---------|
| Next/previous item | Swipe right/left on clickpad |
| Activate focused item | Press clickpad center |
| Read item details | Swipe up/down (changes rotor) |
| Go back | Press Back/Menu button |
| Pause reading | Single tap clickpad |

### Providing accessibility labels

```swift
// Poster card — combine title + metadata for a useful label
PosterCard(movie: movie)
    .focusable()
    .accessibilityLabel("\(movie.title), \(movie.year), rated \(movie.rating)")

// Action button — label + hint
Button { play() } label: {
    PosterCard(movie: movie)
}
.accessibilityLabel("Play \(movie.title)")
.accessibilityHint("Activate to start playback")

// Decorative image — hide from VoiceOver
Image("background-art")
    .accessibilityHidden(true)

// Group related elements (poster + metadata)
HStack {
    AsyncImage(url: movie.posterURL)
        .accessibilityHidden(true)
    VStack(alignment: .leading) {
        Text(movie.title)
        Text(movie.year)
    }
}
.accessibilityElement(children: .combine)
.accessibilityLabel("\(movie.title), \(movie.year)")
```

### Accessibility traits on tvOS

```swift
// Section headers — essential for VoiceOver rotor navigation
Text("Continue Watching")
    .font(.title2)
    .accessibilityAddTraits(.isHeader)

// Selected state (e.g., active tab)
.accessibilityAddTraits(isSelected ? [.isSelected] : [])

// Custom focusable view acting as a button
customView
    .focusable()
    .accessibilityAddTraits(.isButton)
    .accessibilityLabel("Open \(item.title)")
```

### VoiceOver announcements

```swift
// Announce content load completion
.task {
    await catalog.load()
    AccessibilityNotification.Announcement(
        "Content loaded. \(catalog.shelves.count) categories available."
    ).post()
}

// Announce screen change after navigation
.onAppear {
    AccessibilityNotification.ScreenChanged(nil).post()
}

// Announce dynamic count update
.onChange(of: results.count) { _, count in
    AccessibilityNotification.Announcement("\(count) results found").post()
}
```

---

## 3. Dynamic Type

tvOS Dynamic Type scales from the TV-appropriate baseline (body: ~29pt). All text must use system text styles — never hardcode point sizes.

```swift
// Correct — scales with Dynamic Type
Text(movie.title)
    .font(.title2)       // ~22pt default, scales up

Text(movie.synopsis)
    .font(.body)         // ~29pt default on tvOS, scales up

// ScaledMetric for non-text dimensions
@ScaledMetric(relativeTo: .body) private var posterWidth: CGFloat = 250
@ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 44

// Adapt layout at accessibility sizes
@Environment(\.dynamicTypeSize) var typeSize

var itemRow: some View {
    Group {
        if typeSize.isAccessibilitySize {
            VStack(alignment: .leading, spacing: 12) {
                thumbnailView
                metadataView
            }
        } else {
            HStack(alignment: .top, spacing: 24) {
                thumbnailView
                metadataView
            }
        }
    }
}
```

**tvOS text size baseline**: Body 29pt, Titles 48pt+. Dynamic Type at accessibility sizes increases these further. At accessibility sizes, horizontal shelf rows may need to reflow to vertical stacks.

---

## 4. Closed Captions and Audio Descriptions

For media apps, caption and audio description support is required for App Store approval in many markets.

### AVPlayerViewController — automatic support

`AVPlayerViewController` handles captions and audio descriptions automatically when the media asset includes them. The transport bar shows CC/AD controls when tracks are available.

```swift
struct TVPlayerView: UIViewControllerRepresentable {
    let url: URL

    func makeUIViewController(context: Context) -> AVPlayerViewController {
        let vc = AVPlayerViewController()
        let player = AVPlayer(url: url)
        vc.player = player

        // Caption and AD tracks in HLS stream are surfaced automatically:
        // HLS: include SUBTITLES and CLOSED-CAPTIONS groups in the manifest
        // For file playback: embed subtitle tracks or provide sidecar .srt/.vtt files

        // User's caption style (Settings > Accessibility > Subtitles & Captioning)
        // is applied automatically by AVPlayerViewController
        player.play()
        return vc
    }

    func updateUIViewController(_ vc: AVPlayerViewController, context: Context) { }
}
```

### Respecting user media selection preferences

```swift
func applyMediaSelectionPreferences(to item: AVPlayerItem) async {
    let asset = item.asset

    // Legible (captions/subtitles) — honor user's language preferences
    if let legibleGroup = try? await asset.loadMediaSelectionGroup(for: .legible) {
        let preferred = AVMediaSelectionGroup.mediaSelectionOptions(
            from: legibleGroup,
            filteredAndSortedAccordingToPreferredLanguages: Locale.preferredLanguages
        )
        item.select(preferred.first, in: legibleGroup)
    }

    // Audible — select audio descriptions if user has accessibility captions enabled
    if let audibleGroup = try? await asset.loadMediaSelectionGroup(for: .audible) {
        let adOptions = AVMediaSelectionGroup.mediaSelectionOptions(
            from: audibleGroup,
            with: [.describesVideoForAccessibility]
        )
        if UIAccessibility.isClosedCaptioningEnabled, let adOption = adOptions.first {
            item.select(adOption, in: audibleGroup)
        }
    }
}
```

---

## 5. Reduce Motion / Transparency / Contrast

Same environment values as iOS, with TV-specific impact especially on parallax.

### Reduce Motion

```swift
@Environment(\.accessibilityReduceMotion) var reduceMotion

// Parallax MUST be disabled when Reduce Motion is on
PosterCard(movie: movie)
    .focusEffect(reduceMotion ? .none : ParallaxEffect())

// Spring → simple easing
.animation(
    reduceMotion ? .easeInOut(duration: 0.2) : .spring(duration: 0.4, bounce: 0.2),
    value: isFocused
)

// Scale transitions → opacity
.transition(reduceMotion ? .opacity : .scale.combined(with: .opacity))
```

### Reduce Transparency (Liquid Glass)

```swift
@Environment(\.accessibilityReduceTransparency) var reduceTransparency

// .glassEffect() adapts automatically — no manual check needed for glass elements
// Custom translucent overlays need manual handling:
infoOverlay
    .background(
        reduceTransparency
            ? Color(.systemBackground).opacity(0.95)
            : Color(.systemBackground).opacity(0.55)
    )
```

### Increase Contrast

```swift
@Environment(\.colorSchemeContrast) var contrast

// Glass elements adapt automatically
// Custom cards may need stronger borders:
posterCard
    .overlay(
        contrast == .increased
            ? RoundedRectangle(cornerRadius: 12).stroke(.primary, lineWidth: 2)
            : nil
    )
```

---

## 6. Focus and VoiceOver interaction

On tvOS, VoiceOver follows the focus system. Good focus design = good VoiceOver experience:

- **One element focused at a time** — VoiceOver reads the focused element
- **Focus sections = VoiceOver sections** — `.focusSection()` boundaries help VoiceOver users navigate by region
- **Section headers** — mark shelf labels with `.isHeader` so VoiceOver rotor can jump between sections

```swift
// Shelf row accessible as a distinct section
VStack(alignment: .leading) {
    // Header — navigable via VoiceOver rotor
    Text("Popular Now")
        .font(.title2)
        .accessibilityAddTraits(.isHeader)

    ScrollView(.horizontal) {
        LazyHStack(spacing: 40) {
            ForEach(popularMovies) { movie in
                Button { play(movie) } label: {
                    PosterCard(movie: movie)
                }
                .accessibilityLabel("\(movie.title), \(movie.year)")
                .accessibilityHint("Activate to play")
            }
        }
    }
}
.focusSection()  // Isolates this shelf for both focus engine and VoiceOver
```

### Custom actions (VoiceOver rotor)

```swift
// Expose secondary actions without cluttering the main focus item
MoviePosterView(movie: movie)
    .accessibilityLabel(movie.title)
    .accessibilityAction(named: "Add to Watchlist") { addToWatchlist(movie) }
    .accessibilityAction(named: "View Details") { showDetails(movie) }
    // Primary action (play) is the default .activate
```

---

## 7. Accessibility identifiers for UI testing

```swift
// In views
Button("Play") { play() }
    .accessibilityIdentifier("play-button-\(movie.id)")

LazyHStack {
    ForEach(movies) { movie in
        MovieCard(movie: movie)
            .accessibilityIdentifier("poster-\(movie.id)")
    }
}
.accessibilityIdentifier("featured-shelf")

// In XCUITest (tvOS)
let playButton = app.buttons["play-button-tt1234567"]
XCTAssertTrue(playButton.waitForExistence(timeout: 5))
XCUIRemote.shared.press(.select)   // Activate focused element
```

---

## 8. Accessibility testing checklist

Run through this for every screen:

- [ ] **VoiceOver**: Enable (Settings > Accessibility > VoiceOver). Navigate entire screen — is every element described correctly and reachable?
- [ ] **Focus order**: Is VoiceOver reading order (left-right, top-bottom per shelf) logical?
- [ ] **Section headers**: Are shelf labels marked `.isHeader` for VoiceOver rotor navigation?
- [ ] **Dynamic Type**: Enable largest accessibility size — does text fit within 60pt safe area?
- [ ] **Reduce Motion**: Enable — is parallax disabled? Are spring animations simplified to easing?
- [ ] **Reduce Transparency**: Enable — is text over backgrounds readable?
- [ ] **Increase Contrast**: Enable — are all text/background pairs clearly readable at 10 feet?
- [ ] **Bold Text**: Enable — does bold text still fit in poster cards, shelf labels, buttons?
- [ ] **Captions**: Test with Closed Captions enabled in Accessibility > Subtitles & Captioning
- [ ] **Focus targets**: All interactive elements ≥ 66×66pt
- [ ] **Decorative images**: Background art and non-informative icons hidden with `.accessibilityHidden(true)`
- [ ] **Announcements**: Content load completion announced via `AccessibilityNotification.Announcement`
