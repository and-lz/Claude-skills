# Focus System — Complete Reference

## Table of contents
1. How focus works on tvOS
2. SwiftUI focus APIs
3. UIKit focus APIs (TVUIKit)
4. Focus guides
5. Focus sections and scopes
6. Programmatic focus control
7. Custom focus appearance
8. Focus debugging
9. Rules and anti-patterns

---

## 1. How focus works on tvOS

The focus engine is the fundamental interaction model on Apple TV. There is no touch — all navigation is driven by the Siri Remote's directional input (swipe/click on the touch-sensitive clickpad).

**Core rules:**
- Exactly ONE element is focused at any time across the entire screen
- The focus engine determines the next focused element based on directional input (up/down/left/right)
- The system draws a default focus effect: slight scale-up (~1.05x) + shadow + highlight
- Focus moves to the nearest focusable element in the swipe direction
- Only elements marked as focusable can receive focus (buttons, links, list rows are focusable by default)

**Focus engine algorithm:**
1. User swipes a direction on the remote
2. Engine casts a "focus ray" in that direction from the current focused element
3. Candidates are scored by distance, alignment, and visibility
4. Nearest eligible element wins focus
5. If no candidate exists in that direction, focus stays put (with a subtle "thud" animation)

## 2. SwiftUI focus APIs

### @FocusState — programmatic focus binding

```swift
enum Field: Hashable {
    case username, password
}

struct LoginView: View {
    @FocusState private var focusedField: Field?
    @State private var username = ""
    @State private var password = ""

    var body: some View {
        VStack(spacing: 40) {
            TextField("Username", text: $username)
                .focused($focusedField, equals: .username)

            SecureField("Password", text: $password)
                .focused($focusedField, equals: .password)

            Button("Sign In") {
                if username.isEmpty {
                    focusedField = .username  // Move focus programmatically
                } else if password.isEmpty {
                    focusedField = .password
                }
            }
        }
    }
}
```

### focusable() — make any view focusable

```swift
// Make a custom view focusable
PosterCard(movie: movie)
    .focusable(true, interactions: .automatic)

// interactions options:
// .automatic — system determines behavior (default)
// .activate — element is tappable when focused (like a button)
// .edit — element enters edit mode when focused (like a text field)
```

### isFocused — detect focus state

```swift
struct PosterCard: View {
    let movie: Movie
    @Environment(\.isFocused) var isFocused

    var body: some View {
        VStack {
            AsyncImage(url: movie.posterURL)
                .frame(width: 250, height: 375)
                .clipShape(RoundedRectangle(cornerRadius: 12))
                .scaleEffect(isFocused ? 1.08 : 1.0)
                .shadow(radius: isFocused ? 20 : 5)

            Text(movie.title)
                .font(.headline)
                .opacity(isFocused ? 1.0 : 0.7)
        }
        .animation(.spring(response: 0.3), value: isFocused)
    }
}
```

### prefersDefaultFocus() — set initial focus

```swift
struct BrowseView: View {
    @Namespace private var focusNamespace

    var body: some View {
        VStack {
            // Hero banner gets focus on appear
            HeroBanner(item: featured)
                .prefersDefaultFocus(true, in: focusNamespace)

            ShelfRow(items: trending)
            ShelfRow(items: newReleases)
        }
        .focusScope(focusNamespace)
    }
}
```

### focusSection() — define directional focus regions

```swift
// Each shelf is an isolated focus section
// This prevents vertical swipes from jumping across shelves unpredictably
ScrollView(.vertical) {
    LazyVStack(spacing: 50) {
        ForEach(shelves) { shelf in
            VStack(alignment: .leading) {
                Text(shelf.title).font(.title2)

                ScrollView(.horizontal) {
                    LazyHStack(spacing: 40) {
                        ForEach(shelf.items) { item in
                            PosterCard(item: item)
                        }
                    }
                }
            }
            .focusSection()  // Isolate this shelf's focus
        }
    }
}
```

### focusScope() — isolate focus regions with namespace

```swift
struct SidebarView: View {
    @Namespace private var sidebarScope
    @Namespace private var contentScope

    var body: some View {
        HStack {
            // Sidebar region
            VStack {
                ForEach(categories) { category in
                    CategoryButton(category: category)
                }
            }
            .focusScope(sidebarScope)

            // Content region
            ContentGrid(items: filteredItems)
                .focusScope(contentScope)
        }
    }
}
```

### resetFocus — return to default focus

```swift
struct SettingsView: View {
    @Namespace private var settingsScope
    @Environment(\.resetFocus) var resetFocus

    var body: some View {
        VStack {
            // ... settings content ...

            Button("Reset to Defaults") {
                resetDefaults()
                resetFocus(in: settingsScope)  // Jump focus back to preferred element
            }
        }
        .focusScope(settingsScope)
    }
}
```

## 3. UIKit focus APIs (TVUIKit)

### UIFocusEnvironment protocol

```swift
class BrowseViewController: UIViewController {

    @IBOutlet weak var heroButton: UIButton!
    @IBOutlet weak var firstShelfItem: UIButton!

    // Set preferred initial focus
    override var preferredFocusEnvironments: [UIFocusEnvironment] {
        return [heroButton]
    }

    // React to focus changes
    override func didUpdateFocus(
        in context: UIFocusUpdateContext,
        with coordinator: UIFocusAnimationCoordinator
    ) {
        // context.previouslyFocusedItem — what lost focus
        // context.nextFocusedItem — what gained focus

        coordinator.addCoordinatedAnimations({
            // Animate alongside focus transition
            if let next = context.nextFocusedItem as? UIView {
                next.transform = CGAffineTransform(scaleX: 1.1, y: 1.1)
            }
            if let prev = context.previouslyFocusedItem as? UIView {
                prev.transform = .identity
            }
        })
    }

    // Trigger focus update
    func moveFocusToHero() {
        setNeedsFocusUpdate()
        updateFocusIfNeeded()
    }
}
```

## 4. Focus guides

Focus guides are invisible focusable regions that redirect focus — critical for filling gaps in layouts.

```swift
class GridViewController: UIViewController {

    let focusGuide = UIFocusGuide()

    override func viewDidLoad() {
        super.viewDidLoad()

        // Add focus guide to fill the gap
        view.addLayoutGuide(focusGuide)

        // Position the guide in the empty space
        NSLayoutConstraint.activate([
            focusGuide.topAnchor.constraint(equalTo: topButton.bottomAnchor),
            focusGuide.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            focusGuide.trailingAnchor.constraint(equalTo: topButton.trailingAnchor),
            focusGuide.bottomAnchor.constraint(equalTo: bottomButton.topAnchor)
        ])

        // Redirect focus to the desired target
        focusGuide.preferredFocusEnvironments = [bottomButton]
    }

    // Dynamic redirection based on context
    override func didUpdateFocus(
        in context: UIFocusUpdateContext,
        with coordinator: UIFocusAnimationCoordinator
    ) {
        if context.nextFocusedItem === topButton {
            focusGuide.preferredFocusEnvironments = [bottomButton]
        } else if context.nextFocusedItem === bottomButton {
            focusGuide.preferredFocusEnvironments = [topButton]
        }
    }
}
```

## 5. Focus sections and scopes — when to use which

| Concept | SwiftUI | Purpose |
|---------|---------|---------|
| **focusSection()** | `.focusSection()` | Groups views for directional focus — focus moves within the section before escaping |
| **focusScope()** | `.focusScope(namespace)` | Creates an isolated focus namespace for `prefersDefaultFocus` and `resetFocus` |
| **Focus guide** | UIKit `UIFocusGuide` | Fills empty space with invisible focusable area that redirects |

**Use focusSection() when:**
- You have horizontal shelves in a vertical scroll — each shelf is a section
- You want to prevent focus from "jumping" to unrelated elements during directional navigation

**Use focusScope() when:**
- You need to control which element gets initial focus within a region
- You want to programmatically reset focus to a specific element

## 6. Programmatic focus control

```swift
// SwiftUI — move focus with @FocusState
@FocusState var focused: ItemID?

// Set focus to specific item
focused = item.id

// Clear focus (system decides)
focused = nil

// UIKit — trigger focus update
setNeedsFocusUpdate()      // Mark focus as needing update
updateFocusIfNeeded()      // Execute the update immediately

// UIKit — check if focus update is possible
shouldUpdateFocus(in: context) -> Bool
```

## 7. Custom focus appearance

```swift
// Disable default focus effect (scale + shadow)
Button("Custom") { }
    .focusEffectDisabled()

// Then apply your own focus styling
struct CustomFocusCard: View {
    @Environment(\.isFocused) var isFocused

    var body: some View {
        RoundedRectangle(cornerRadius: 16)
            .fill(isFocused ? Color.blue : Color.gray.opacity(0.3))
            .overlay(
                RoundedRectangle(cornerRadius: 16)
                    .stroke(isFocused ? Color.white : Color.clear, lineWidth: 4)
            )
            .scaleEffect(isFocused ? 1.05 : 1.0)
            .shadow(color: .black.opacity(isFocused ? 0.5 : 0.2),
                    radius: isFocused ? 20 : 5)
            .animation(.spring(response: 0.35, dampingFraction: 0.7), value: isFocused)
            .focusable()
            .focusEffectDisabled()
    }
}
```

## 8. Focus debugging

**In Xcode:**
- `po UIFocusDebugger.status()` — prints current focus state
- `po UIFocusDebugger.checkFocusability(for: view)` — checks if a view can be focused
- `po UIFocusDebugger.simulateFocusUpdateRequest(from: environment)` — simulates focus update

**Common focus issues:**
1. **Focus gets stuck** — Check for empty space between focusable elements. Add `UIFocusGuide` or ensure elements are adjacent.
2. **Focus jumps to wrong element** — Use `.focusSection()` to constrain focus within logical groups.
3. **Can't set initial focus** — Ensure `.prefersDefaultFocus()` is within a `.focusScope()`.
4. **Hidden views capture focus** — Set `isHidden = true` or remove from hierarchy. Hidden views shouldn't be focusable but sometimes are.
5. **Scroll view doesn't follow focus** — Use `ScrollViewReader` with `.scrollTo()` in response to focus changes.

## 9. Rules and anti-patterns

### DO:
- Always provide visible focus states on every focusable element
- Use `.focusSection()` for each horizontal shelf in a vertical scroll
- Map focus flow mentally before coding — draw the focus graph
- Test with physical Siri Remote (simulator keyboard behaves differently)
- Use `UIFocusGuide` to bridge gaps in layouts
- Provide escape routes — focus should never get trapped

### DON'T:
- Don't create invisible focusable elements without visual feedback
- Don't use `.focusEffectDisabled()` without providing custom focus styling
- Don't assume focus order matches visual order — test directional movement
- Don't nest `ScrollView` inside `ScrollView` without careful focus section management
- Don't ignore the Menu button — always handle `.onExitCommand` gracefully
- Don't place focusable elements too close together (< 20pt gap) — focus engine gets confused