# Accessibility and Internationalization

## Table of contents
1. Accessibility overview
2. VoiceOver
3. Dynamic Type
4. Color and contrast
5. Reduce Motion
6. Reduce Transparency
7. Bold Text
8. Switch Control and assistive tech
9. Focus management
10. Accessibility testing checklist
11. iOS 26 accessibility features
12. Internationalization overview
13. RTL layout support
14. String Catalogs
15. Pluralization
16. Date, number, and currency formatting
17. Text expansion planning
18. Testing localization

---

## 1. Accessibility overview

Accessibility is not optional — it's a legal requirement in many jurisdictions and a core Apple value. Every iOS app should be usable by people who:
- Cannot see the screen (VoiceOver, Braille)
- Have low vision (Dynamic Type, Zoom, color filters)
- Cannot hear (visual alerts, captions)
- Have motor impairments (Switch Control, Voice Control, AssistiveTouch)
- Have cognitive differences (Reduce Motion, simplified layouts)

Apple reviews accessibility during App Store review and highlights accessible apps.

## 2. VoiceOver

### Label, Value, Trait, Hint
VoiceOver reads elements in this order:
1. **Label** — What the element is ("Play button", "Volume")
2. **Value** — Current state ("50%", "On")
3. **Trait** — What type of control (.button, .header, .link)
4. **Hint** — What happens on activation ("Double tap to play")

### Writing good labels
```swift
// DO: Short, descriptive, no control type (VoiceOver adds it from traits)
Image(systemName: "heart.fill")
    .accessibilityLabel("Favorite")

// DON'T: Include control type — VoiceOver says "Favorite button, button"
    .accessibilityLabel("Favorite button")

// DON'T: Describe the visual
    .accessibilityLabel("Heart icon")

// DO: Include context when needed
    .accessibilityLabel("Favorite \(item.title)")
```

### Common patterns
```swift
// Image that conveys information
Image("chart")
    .accessibilityLabel("Sales trending upward, 15% growth")

// Decorative image — hide from VoiceOver
Image("background")
    .accessibilityHidden(true)

// Custom action
.accessibilityAction(named: "Delete") { delete() }

// Group related elements
VStack {
    Text(item.title)
    Text(item.price)
}
.accessibilityElement(children: .combine)

// Custom reading order
.accessibilitySortPriority(1)  // Higher = read first
```

### Traits
```swift
.accessibilityAddTraits(.isButton)
.accessibilityAddTraits(.isHeader)     // Section headers — essential for navigation
.accessibilityAddTraits(.isLink)
.accessibilityAddTraits(.isImage)
.accessibilityAddTraits(.isModal)      // Trap focus in custom dialogs
.accessibilityAddTraits(.isSelected)
.accessibilityAddTraits(.startsMediaSession)
.accessibilityAddTraits(.updatesFrequently)
.accessibilityRemoveTraits(.isButton)  // Remove default trait
```

### Rotor actions
Custom actions accessible via VoiceOver rotor gesture:
```swift
.accessibilityAction(.magicTap) { togglePlayback() }
.accessibilityAction(.escape) { dismiss() }

// Custom rotor
.accessibilityRotor("Headings") {
    ForEach(headings) { heading in
        AccessibilityRotorEntry(heading.title, id: heading.id)
    }
}
```

### Announcements
```swift
// Announce dynamic content changes
AccessibilityNotification.Announcement("3 new messages").post()

// Announce screen changes
AccessibilityNotification.ScreenChanged(newView).post()

// Announce layout changes
AccessibilityNotification.LayoutChanged(updatedElement).post()
```

## 3. Dynamic Type

### Requirements
- ALL text must use text styles or custom fonts with `relativeTo:`
- ALL layouts must accommodate text from xSmall through AX5
- At accessibility sizes, layouts should reflow (horizontal → vertical)
- Never disable Dynamic Type entirely

### Testing Dynamic Type
```swift
// Preview at different sizes
#Preview {
    ContentView()
        .dynamicTypeSize(.accessibility5)
}

// Environment override for testing
.environment(\.dynamicTypeSize, .accessibility3)
```

### Handling large text gracefully
```swift
@Environment(\.dynamicTypeSize) var typeSize

var body: some View {
    if typeSize.isAccessibilitySize {
        // Stack vertically for large text
        VStack(alignment: .leading) {
            label
            value
        }
    } else {
        // Side by side for standard sizes
        HStack {
            label
            Spacer()
            value
        }
    }
}
```

### @ScaledMetric for non-text dimensions
```swift
@ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24
@ScaledMetric(relativeTo: .body) private var spacing: CGFloat = 8
```

## 4. Color and contrast

### WCAG contrast requirements
| Level | Normal text (<18pt) | Large text (≥18pt or ≥14pt bold) |
|-------|--------------------|---------------------------------|
| AA (minimum) | **4.5:1** | **3:1** |
| AAA (enhanced) | **7:1** | **4.5:1** |

Apple requires AA compliance. AAA is recommended for critical content.

### Don't convey information by color alone
```swift
// BAD: Only color distinguishes states
Circle().fill(isOnline ? .green : .red)

// GOOD: Color + shape/icon/label
HStack {
    Image(systemName: isOnline ? "checkmark.circle.fill" : "xmark.circle")
        .foregroundStyle(isOnline ? .green : .red)
    Text(isOnline ? "Online" : "Offline")
}
```

### Differentiate Without Color accessibility setting
```swift
@Environment(\.accessibilityDifferentiateWithoutColor) var diffWithoutColor

// Add shapes, borders, or patterns when color can't be relied upon
```

### Invert Colors
Smart Invert reverses all colors except images, media, and apps that use dark color schemes. Mark media content:
```swift
.accessibilityIgnoresInvertColors(true)  // On images, video players
```

## 5. Reduce Motion

See `references/animation-transitions.md` section 10 for full implementation details.

Key points:
- Always check `@Environment(\.accessibilityReduceMotion)`
- Replace springs with crossfades or instant changes
- Zoom transitions automatically fall back to slide
- Parallax and gyroscope effects must be disabled
- Loading spinners and progress bars are exempt

## 6. Reduce Transparency

Critical for Liquid Glass apps — when enabled, glass becomes more opaque.

```swift
@Environment(\.accessibilityReduceTransparency) var reduceTransparency

if reduceTransparency {
    // Replace translucent backgrounds with solid
    content.background(Color(.systemBackground))
} else {
    content.background(.ultraThinMaterial)
}
```

Liquid Glass automatically adapts (becomes frostier), but custom translucent elements need manual handling.

## 7. Bold Text

```swift
@Environment(\.legibilityWeight) var legibilityWeight

// legibilityWeight is .bold when Bold Text is enabled
// System fonts handle this automatically
// Custom fonts may need manual weight adjustment
```

Bold Text increases all font weights. Ensure layouts have enough padding for wider bold text.

## 8. Switch Control and assistive technologies

### Switch Control
Users navigate sequentially through interactive elements. Ensure:
- Every interactive element is reachable
- Focus order is logical (top-to-bottom, leading-to-trailing)
- Custom controls declare appropriate traits
- Grouped elements use `.accessibilityElement(children: .combine)` or `.contain`

### Voice Control
Users speak element labels to interact. Ensure:
- All buttons have visible text or meaningful accessibility labels
- Labels match visible text exactly (Voice Control uses label text for commands)
- Custom number labels are available for complex interfaces

### Full Keyboard Access
iPad/Mac users navigate entirely by keyboard:
- Tab order follows a logical sequence
- Focus rings are visible on all interactive elements
- Custom controls support keyboard activation

### iOS 26: Brain-Computer Interface Protocol
New protocol enabling Switch Control via brain-computer interfaces for users with severe motor disabilities. No special app code needed — works with existing Switch Control support.

## 9. Focus management

### Moving focus programmatically
```swift
@AccessibilityFocusState private var focused: FocusField?

enum FocusField {
    case searchField, firstResult, errorMessage
}

// Move focus to search field
focused = .searchField

// Apply to view
TextField("Search", text: $query)
    .accessibilityFocused($focused, equals: .searchField)
```

### When to move focus
- **New content appears**: Move focus to the new content (alert, error, new screen)
- **Modal dismissal**: Return focus to the element that triggered the modal
- **Delete action**: Move focus to the next item in a list after deletion
- **Form validation error**: Move focus to the first error field
- **Screen transition**: Let system handle standard navigation; intervene for custom transitions

### Focus trapping for modals
```swift
.accessibilityAddTraits(.isModal) // Prevents VoiceOver from escaping custom modals
```

## 10. Accessibility testing checklist

Run through this for every screen:

- [ ] **VoiceOver**: Navigate entire screen with VoiceOver. Is every element reachable? Is reading order logical? Do labels make sense without seeing the screen?
- [ ] **Dynamic Type**: Test at xSmall, Large (default), xxxLarge, AX3, and AX5. Does text truncate? Do layouts reflow?
- [ ] **Color contrast**: Check all text against backgrounds. Use Accessibility Inspector or manual calculation. Minimum 4.5:1 for normal text, 3:1 for large text.
- [ ] **Color alone**: Disable "Differentiate Without Color" — is all information still conveyed?
- [ ] **Bold Text**: Enable — does text still fit? Do layouts break?
- [ ] **Reduce Motion**: Enable — are all animations disabled/simplified?
- [ ] **Reduce Transparency**: Enable — are all translucent elements legible?
- [ ] **Increase Contrast**: Enable — does the UI remain usable with stronger borders?
- [ ] **Touch targets**: Verify 44×44pt minimum for all interactive elements
- [ ] **Focus management**: After navigation, modal dismiss, or content changes — is focus in the right place?
- [ ] **Smart Invert**: Enable — are images and media preserved?
- [ ] **Keyboard navigation**: On iPad, can all elements be reached via Tab?

## 11. iOS 26 accessibility features

### Accessibility Reader
System-wide reading mode with adjustable font size, spacing, and background colors. No app code needed — works with standard text rendering.

### Accessibility Nutrition Labels
App Store labels showing each app's accessibility features (like food nutrition labels). Developers declare supported features in App Store Connect. Support VoiceOver, Dynamic Type, and contrast at minimum.

### Braille Access
Full braille notepad with BRF editing and Nemeth code (math notation). Works with connected braille displays.

### Enhanced Liquid Glass accessibility
- Reduce Transparency: Glass becomes progressively more opaque
- Increase Contrast: Glass elements become solid black/white with borders
- Reduce Motion: Elastic glass properties disabled, materialization simplified

---

## 12. Internationalization overview

iOS supports 40+ languages natively. Your app should support:
- All text externalized for translation
- RTL layout for Arabic, Hebrew, Persian, Urdu
- Locale-aware date, time, number, and currency formatting
- Pluralization rules per language
- Text expansion accommodation

## 13. RTL layout support

### Use leading/trailing, never left/right
```swift
// DO
.padding(.leading, 16)
.frame(maxWidth: .infinity, alignment: .leading)
HStack { } // Automatically reverses in RTL

// DON'T
.padding(.left, 16)  // Doesn't mirror
.frame(maxWidth: .infinity, alignment: .left)
```

### What mirrors in RTL
- Navigation direction (back button moves to trailing edge)
- Tab bar order
- Text alignment (leading → trailing)
- Swipe actions (directions reverse)
- Padding (leading ↔ trailing)
- Slider direction

### What does NOT mirror in RTL
- Video playback controls (play/pause/seek)
- Phone keypads
- Numbers within text (Arabic numerals always LTR within RTL text)
- Clocks (always clockwise)
- Music notation
- Checkmarks (always left side)

### Testing RTL
```swift
// Preview in RTL
#Preview {
    ContentView()
        .environment(\.layoutDirection, .rightToLeft)
}
```

In Xcode: Edit Scheme → Run → Options → App Language → Right to Left Pseudolanguage

## 14. String Catalogs

String Catalogs (`.xcstrings`, Xcode 15+) replace `.strings` and `.stringsdict` with a single unified file.

### SwiftUI automatic extraction
```swift
// These strings are automatically extracted:
Text("Hello, World!")
Button("Save") { }
Label("Settings", systemImage: "gear")
NavigationTitle("Profile")
```

### Explicit keys
```swift
Text("greeting_morning", tableName: "Greetings")
```

### String interpolation
```swift
Text("Welcome, \(username)")  // Extracted with %@ placeholder
Text("\(count) items")        // Extracted with %lld placeholder
```

### Avoiding hardcoded strings
```swift
// BAD
label.text = "Error"

// GOOD
label.text = String(localized: "error_title")
// or in SwiftUI: Text("error_title") — auto-extracted
```

Never concatenate localized strings — word order varies by language:
```swift
// BAD
Text(firstName + " " + lastName)

// GOOD — use PersonNameComponentsFormatter
let formatter = PersonNameComponentsFormatter()
Text(formatter.string(from: nameComponents))
```

## 15. Pluralization

Different languages need different plural forms:

| Language | Forms needed |
|----------|-------------|
| Chinese, Japanese, Korean | 1 (other) |
| English | 2 (one, other) |
| French | 2 (one, other) — but 0 is singular |
| Russian | 3 (one, few, many) |
| Arabic | **6** (zero, one, two, few, many, other) |
| Polish | 3 (one, few, many) |

String Catalogs handle all plural forms automatically. Define variants in the `.xcstrings` file for each locale.

```swift
// Automatic pluralization with String Catalogs
Text("^[\(count) item](inflect: true)")  // Uses Automatic Grammar Agreement
```

## 16. Date, number, and currency formatting

### Dates — never hardcode format
```swift
// SwiftUI
Text(date, style: .date)     // "June 15, 2025" or "15 juin 2025"
Text(date, style: .time)     // "3:30 PM" or "15:30"
Text(date, style: .relative) // "2 hours ago"
Text(date, style: .timer)    // "1:30:00"

// Formatted API (iOS 15+)
date.formatted(.dateTime.month().day().year())
date.formatted(.dateTime.hour().minute())
date.formatted(date: .abbreviated, time: .shortened)
```

### Numbers
```swift
let formatted = 1234567.89.formatted()  // "1,234,567.89" (US) or "1.234.567,89" (DE)
let percent = 0.75.formatted(.percent)   // "75%" or "75 %"
let currency = 49.99.formatted(.currency(code: "USD"))  // "$49.99"
```

### Measurements
```swift
let distance = Measurement(value: 5, unit: UnitLength.kilometers)
let formatted = distance.formatted()  // "5 km" or "3.1 mi" (locale-dependent)
```

## 17. Text expansion planning

When designing layouts, account for text expansion from English:

| Target language | Expansion |
|----------------|-----------|
| German | +30% |
| French | +15–20% |
| Finnish | +30–40% |
| Italian | +15% |
| Spanish | +15–25% |
| Arabic | +20–25% |
| Japanese/Chinese | -10–30% (shorter but taller characters) |

### Design strategies
- Use flexible layouts (don't hardcode widths for text)
- Allow text to wrap to multiple lines
- Test with Xcode's Double-Length Pseudolanguage
- Design buttons wide enough for 40% longer text
- Never truncate critical information

## 18. Testing localization

### Xcode pseudolanguages
- **Accented Pseudolanguage**: Adds diacritics to find hardcoded strings
- **Bounded Pseudolanguage**: Wraps strings in brackets [like this]
- **Right to Left Pseudolanguage**: Forces RTL layout
- **Double-Length Pseudolanguage**: Doubles all string lengths

### Automated testing
```swift
// UI test in different locales
let app = XCUIApplication()
app.launchArguments += ["-AppleLanguages", "(de)"]
app.launchArguments += ["-AppleLocale", "de_DE"]
app.launch()
```

### Export/import for translators
Xcode → Product → Export Localizations → generates `.xliff` files for each language. Translators work in standard XLIFF format. Import back via Product → Import Localizations.
