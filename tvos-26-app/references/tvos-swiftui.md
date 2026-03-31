# tvOS-Specific SwiftUI — Complete Reference

## Table of contents
1. Modifiers that behave differently on tvOS
2. tvOS-only modifiers
3. Button styles on tvOS
4. Lists and forms on tvOS
5. Sheets and presentations
6. Search on tvOS
7. Context menus
8. Conditional compilation
9. Rules and anti-patterns

---

## 1. Modifiers that behave differently on tvOS

### TabView — top bar, not bottom

```swift
// tvOS: renders at TOP of screen
// iOS: renders at BOTTOM of screen
TabView {
    Tab("Home", systemImage: "house") { HomeView() }
    Tab("Browse", systemImage: "rectangle.grid.2x2") { BrowseView() }
    Tab("Search", systemImage: "magnifyingglass") { SearchView() }
}
// Tab bar hides when content has focus
// Swipe up reveals tab bar
```

### .sheet() — always full screen

```swift
// tvOS: sheet is ALWAYS full screen (no half-sheet, no detents)
// iOS: defaults to half-sheet with detents
.sheet(isPresented: $showSheet) {
    SheetContent()
    // .presentationDetents is IGNORED on tvOS
    // .presentationDragIndicator is IGNORED on tvOS
}
```

### .searchable() — full-screen keyboard

```swift
// tvOS: opens a dedicated full-screen search interface with on-screen keyboard
// iOS: inline search bar at top/bottom of list
NavigationStack {
    ContentList()
}
.searchable(text: $query, prompt: "Search movies")
// On tvOS: tapping Search tab opens the keyboard UI
```

### List — focusable rows

```swift
// tvOS: list rows are focusable, highlight on focus, scale slightly
// iOS: list rows are tappable, highlight on press
List(items) { item in
    HStack {
        AsyncImage(url: item.thumbnailURL)
            .frame(width: 80, height: 80)
        VStack(alignment: .leading) {
            Text(item.title).font(.headline)
            Text(item.subtitle).font(.subheadline)
        }
    }
}
// Rows get automatic focus appearance on tvOS
// No swipe-to-delete on tvOS (use context menu instead)
```

### Button — auto focus effect

```swift
// tvOS: buttons automatically scale + highlight on focus
// iOS: buttons respond to touch with highlight
Button("Action") { doSomething() }
// Automatic: scale 1.05x, shadow, focus ring on tvOS
// No manual styling needed for standard buttons
```

### Toggle — focus to toggle

```swift
// tvOS: focus on toggle, press clickpad to change
// iOS: tap toggle directly
Toggle("Enable Feature", isOn: $isEnabled)
// Same API, different interaction model
```

### Picker — focus-based selection

```swift
// tvOS: picker options are focusable, select with click
Picker("Quality", selection: $quality) {
    Text("Auto").tag(Quality.auto)
    Text("1080p").tag(Quality.hd)
    Text("4K").tag(Quality.uhd)
}
.pickerStyle(.inline)  // Shows all options on tvOS
// .pickerStyle(.menu) also works on tvOS
```

## 2. tvOS-only modifiers

### .onPlayPauseCommand

```swift
// ONLY available on tvOS and watchOS
.onPlayPauseCommand {
    player.isPlaying ? player.pause() : player.play()
}
```

### .onExitCommand

```swift
// ONLY available on tvOS
.onExitCommand {
    // Called when user presses Back/Menu button
    if isShowingOverlay {
        isShowingOverlay = false
    }
    // If you don't handle it, system performs default back navigation
}
```

### .onMoveCommand

```swift
// ONLY available on tvOS and macOS
.onMoveCommand { direction in
    switch direction {
    case .up: moveUp()
    case .down: moveDown()
    case .left: moveLeft()
    case .right: moveRight()
    @unknown default: break
    }
}
```

### .focusable() — critical on tvOS

```swift
// Available on all platforms but CRITICAL on tvOS
// Without focusable(), custom views can't receive focus
CustomCardView()
    .focusable()          // Make it focusable
    .focusable(true, interactions: .activate)  // Focusable + activatable
```

### .prefersDefaultFocus()

```swift
// Available on tvOS, macOS, watchOS (NOT iOS)
Button("Featured") { }
    .prefersDefaultFocus(true, in: namespace)
// Must be used with .focusScope()
```

### .focusSection()

```swift
// Available on tvOS and macOS
ScrollView(.horizontal) {
    LazyHStack { /* items */ }
}
.focusSection()  // Isolate focus within this section
```

### .buttonStyle(.card)

```swift
// tvOS-specific card button style
NavigationLink(value: item) {
    VStack {
        Image(item.poster)
        Text(item.title)
    }
}
.buttonStyle(.card)  // Renders as a focusable card with lift effect
// Similar to default focus effect but designed for card-based content
```

## 3. Button styles on tvOS

```swift
// Standard button — auto focus scale
Button("Default") { }
// Automatic: scale on focus, glass/border appearance

// Glass button (tvOS 26)
Button("Glass") { }
    .buttonStyle(.glass)

// Prominent glass button (tvOS 26)
Button("Primary") { }
    .buttonStyle(.glassProminent)

// Bordered
Button("Bordered") { }
    .buttonStyle(.bordered)
// tvOS: rounded rect with border, scales on focus

// Bordered prominent
Button("Prominent") { }
    .buttonStyle(.borderedProminent)
// tvOS: filled background, scales on focus

// Card (tvOS specific)
Button("Card") { }
    .buttonStyle(.card)
// Designed for poster/card content — lift + shadow on focus

// Plain (no default chrome)
Button("Plain") { }
    .buttonStyle(.plain)
// No focus effect — you must provide custom focus styling
// WARNING: use with custom focus styling or users won't know it's focusable
```

## 4. Lists and forms on tvOS

```swift
// List on tvOS
List {
    Section("Playback") {
        // Rows are focusable
        NavigationLink("Quality Settings") {
            QualitySettingsView()
        }

        Toggle("Auto-Play Next Episode", isOn: $autoPlay)

        Picker("Default Audio", selection: $audioLang) {
            ForEach(languages) { lang in
                Text(lang.name).tag(lang)
            }
        }
    }

    Section("Account") {
        LabeledContent("Username", value: user.name)

        Button("Sign Out", role: .destructive) {
            signOut()
        }
    }
}
// tvOS list:
// - Rows highlight/scale on focus
// - Navigation chevrons appear automatically for NavigationLink
// - Section headers are sentence case (tvOS 26)
// - No swipe actions (use context menu instead)
// - No pull-to-refresh
```

### Form on tvOS

```swift
Form {
    Section("Profile") {
        TextField("Display Name", text: $displayName)
        // Opens on-screen keyboard on focus + click

        Picker("Avatar", selection: $avatar) {
            ForEach(Avatar.allCases) { avatar in
                Image(avatar.imageName).tag(avatar)
            }
        }
    }
}
// Forms work similarly to iOS but:
// - Text fields trigger on-screen keyboard
// - Minimize text input — prefer pickers/toggles
```

## 5. Sheets and presentations

```swift
// Sheet — always full screen on tvOS
.sheet(isPresented: $show) {
    NavigationStack {
        DetailView()
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("Done") { show = false }
                }
            }
    }
}

// Full screen cover — same as sheet on tvOS (both are full screen)
.fullScreenCover(isPresented: $show) {
    PlayerView()
}

// Alert — centered, focus on buttons
.alert("Confirm", isPresented: $showAlert) {
    Button("Cancel", role: .cancel) { }
    Button("Delete", role: .destructive) { delete() }
} message: {
    Text("Are you sure?")
}
// tvOS 26: left-aligned text in alerts (same as iOS 26)

// Confirmation dialog — centered overlay
.confirmationDialog("Options", isPresented: $showOptions) {
    Button("Option A") { }
    Button("Option B") { }
    Button("Cancel", role: .cancel) { }
}
// tvOS: appears as centered panel (not anchored to source like iOS 26)
```

## 6. Search on tvOS

```swift
struct TVSearchView: View {
    @State private var query = ""
    @State private var results: SearchResults?

    var body: some View {
        NavigationStack {
            Group {
                if query.isEmpty {
                    // Trending / suggested content
                    SuggestionsGrid()
                } else if let results, !results.isEmpty {
                    // Results organized in shelves
                    SearchResultsShelves(results: results)
                } else {
                    ContentUnavailableView.search(text: query)
                }
            }
            .navigationTitle("Search")
        }
        .searchable(text: $query, prompt: "Movies, Shows, People")
        .searchSuggestions {
            ForEach(suggestions) { suggestion in
                Text(suggestion.text)
                    .searchCompletion(suggestion.text)
            }
        }
        .task(id: query) {
            guard !query.isEmpty else {
                results = nil
                return
            }
            try? await Task.sleep(for: .milliseconds(300))
            results = await search(query)
        }
    }
}

// tvOS search flow:
// 1. Tab to Search shows on-screen keyboard at top
// 2. Results appear below as user types
// 3. Siri dictation available via remote button
// 4. Suggestions appear between keyboard and results
```

## 7. Context menus

```swift
// Context menus on tvOS: activated by LONG PRESS on clickpad
Button {
    playItem(item)
} label: {
    PosterCard(item: item)
}
.contextMenu {
    Button("Play") { playItem(item) }
    Button("Add to Watchlist") { addToWatchlist(item) }
    Button("Share") { share(item) }

    Divider()

    Button("Mark as Watched") { markWatched(item) }

    Divider()

    Button("Remove", role: .destructive) { remove(item) }
}
// tvOS: appears as a centered overlay panel
// iOS: appears as a popover from the element
```

## 8. Conditional compilation

```swift
// Compile for tvOS only
#if os(tvOS)
import TVUIKit
import TVServices

struct TVOnlyFeature: View {
    var body: some View {
        Text("Only on Apple TV")
            .onPlayPauseCommand { }
            .onExitCommand { }
    }
}
#endif

// Shared code with platform differences
struct AdaptiveView: View {
    var body: some View {
        VStack {
            Text("Title")
                .font(.title)

            #if os(tvOS)
            // TV-specific layout
            HorizontalShelf(items: items)
                .focusSection()
            #else
            // iOS layout
            VerticalList(items: items)
            #endif
        }
    }
}

// Check at runtime (less common)
#if targetEnvironment(simulator)
// Simulator-specific code
#endif
```

## 9. Rules and anti-patterns

### DO:
- Use `.focusable()` on all custom interactive views
- Use `.focusSection()` on horizontal shelf rows
- Use `.buttonStyle(.card)` for poster/card NavigationLinks
- Handle `.onExitCommand` in views with custom overlays
- Use `.onPlayPauseCommand` in media-related views
- Use `#if os(tvOS)` for TV-specific code in shared targets
- Use `ContentUnavailableView` for empty states and search

### DON'T:
- Don't use `.presentationDetents` (ignored on tvOS)
- Don't use `.refreshable` (no pull-to-refresh on TV)
- Don't use `.swipeActions` on list rows (not supported on TV)
- Don't use `.scrollDismissesKeyboard` (not applicable)
- Don't use `.toolbar(.hidden, for: .tabBar)` to hide tab bar permanently (users need it)
- Don't use `.buttonStyle(.plain)` without custom focus styling
- Don't use `.sensoryFeedback()` (no Taptic Engine on Apple TV)
- Don't use gestures (DragGesture, RotationGesture, MagnifyGesture) — they don't work on TV
- Don't use `.popover()` — not supported on tvOS; use `.sheet()` instead