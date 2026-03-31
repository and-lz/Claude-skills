# Navigation Patterns — Complete Reference

## Table of contents
1. Tab bar (top of screen)
2. NavigationStack
3. NavigationSplitView (sidebar)
4. Full-screen browsing pattern
5. Modal presentations
6. Search
7. Settings/preferences
8. Rules and anti-patterns

---

## 1. Tab bar (top of screen)

On tvOS, the tab bar renders at the **top** of the screen (not bottom like iOS). Swiping up from content reveals the tab bar. In tvOS 26, the tab bar gets Liquid Glass treatment.

```swift
struct AppRootView: View {
    var body: some View {
        TabView {
            Tab("Home", systemImage: "house") {
                HomeView()
            }

            Tab("Movies", systemImage: "film") {
                MoviesView()
            }

            Tab("TV Shows", systemImage: "tv") {
                TVShowsView()
            }

            Tab("Search", systemImage: "magnifyingglass") {
                SearchView()
            }

            Tab("Settings", systemImage: "gear") {
                SettingsView()
            }
        }
        // tvOS: tabs appear at top
        // Swiping up on content reveals tab bar
        // 3-5 tabs recommended (max 7)
    }
}
```

### Tab bar behavior on tvOS
- **Hidden by default** when scrolling content
- **Revealed** when user swipes up past the top of content
- **Focus**: tab bar items are focusable; left/right swipe moves between tabs
- **tvOS 26**: Liquid Glass styling applied automatically
- **No badges** on tvOS (unlike iOS)

### Tab with NavigationStack

```swift
Tab("Movies", systemImage: "film") {
    NavigationStack {
        MovieBrowseView()
            .navigationDestination(for: Movie.self) { movie in
                MovieDetailView(movie: movie)
            }
    }
}
```

## 2. NavigationStack

Push/pop navigation on tvOS. Back navigation uses the **Menu/Back button** on the Siri Remote.

```swift
struct MovieBrowseView: View {
    @State private var path = NavigationPath()

    var body: some View {
        NavigationStack(path: $path) {
            ScrollView {
                LazyVStack(spacing: 50) {
                    HeroBanner(movie: featured)

                    ShelfRow(title: "Trending", items: trending)
                    ShelfRow(title: "New Releases", items: newReleases)
                    ShelfRow(title: "Top Rated", items: topRated)
                }
            }
            .navigationTitle("Movies")
            .navigationDestination(for: Movie.self) { movie in
                MovieDetailView(movie: movie)
            }
        }
    }
}
```

### Navigation transitions on tvOS

```swift
// Standard push (slide from right) — default
NavigationLink(value: movie) {
    PosterCard(movie: movie)
}

// Zoom transition (iOS 26 style) — also works on tvOS 26
NavigationLink(value: movie) {
    PosterCard(movie: movie)
        .matchedTransitionSource(id: movie.id, in: namespace)
}
// In destination:
MovieDetailView(movie: movie)
    .navigationTransition(.zoom(sourceID: movie.id, in: namespace))
```

## 3. NavigationSplitView (sidebar)

Sidebar pattern for category-based browsing. The sidebar slides in from the left when user swipes right on the remote.

```swift
struct LibraryView: View {
    @State private var selectedCategory: Category?

    var body: some View {
        NavigationSplitView {
            // Sidebar
            List(Category.allCases, selection: $selectedCategory) { category in
                Label(category.name, systemImage: category.icon)
            }
            .navigationTitle("Library")
        } detail: {
            // Detail content
            if let category = selectedCategory {
                CategoryContentView(category: category)
            } else {
                ContentUnavailableView(
                    "Select a Category",
                    systemImage: "rectangle.stack",
                    description: Text("Choose a category from the sidebar")
                )
            }
        }
    }
}
```

### Sidebar behavior on tvOS
- Sidebar **auto-hides** when user navigates into detail content
- User swipes **right** from left edge to reveal sidebar
- Sidebar items receive focus — up/down navigates, select enters detail
- In tvOS 26: sidebar gets Liquid Glass overlay treatment

### Three-column split

```swift
NavigationSplitView {
    // Sidebar (categories)
    List(categories, selection: $selectedCategory) { ... }
} content: {
    // Middle column (items in category)
    if let category = selectedCategory {
        List(category.items, selection: $selectedItem) { ... }
    }
} detail: {
    // Detail view
    if let item = selectedItem {
        ItemDetailView(item: item)
    }
}
```

## 4. Full-screen browsing pattern

The dominant pattern on Apple TV (used by Apple TV+, Netflix, Disney+):

```swift
struct HomeBrowseView: View {
    @State private var featuredItems: [FeaturedItem] = []
    @State private var shelves: [Shelf] = []
    @Namespace private var heroNamespace

    var body: some View {
        ScrollView(.vertical, showsIndicators: false) {
            LazyVStack(spacing: 50) {
                // 1. Hero carousel (auto-scrolling)
                HeroCarousel(items: featuredItems)
                    .frame(height: 600)
                    .focusSection()

                // 2. Content shelves
                ForEach(shelves) { shelf in
                    ShelfSection(shelf: shelf)
                        .focusSection()  // Each shelf is its own focus section
                }
            }
        }
    }
}

// Hero carousel with auto-scrolling
struct HeroCarousel: View {
    let items: [FeaturedItem]
    @State private var currentIndex = 0

    var body: some View {
        TabView(selection: $currentIndex) {
            ForEach(items.indices, id: \.self) { index in
                HeroBannerCard(item: items[index])
                    .tag(index)
            }
        }
        .tabViewStyle(.page)
        // Auto-advance every 6 seconds
        .task {
            while !Task.isCancelled {
                try? await Task.sleep(for: .seconds(6))
                withAnimation(.spring) {
                    currentIndex = (currentIndex + 1) % items.count
                }
            }
        }
    }
}

// Shelf section with title + horizontal scroll
struct ShelfSection: View {
    let shelf: Shelf

    var body: some View {
        VStack(alignment: .leading, spacing: 16) {
            HStack {
                Text(shelf.title)
                    .font(.title2)

                if shelf.hasMore {
                    Spacer()
                    NavigationLink("See All", value: shelf)
                        .font(.callout)
                }
            }
            .padding(.horizontal)

            ScrollView(.horizontal, showsIndicators: false) {
                LazyHStack(spacing: 40) {
                    ForEach(shelf.items) { item in
                        NavigationLink(value: item) {
                            PosterCard(item: item)
                        }
                        .buttonStyle(.card)
                    }
                }
                .padding(.horizontal)
                .padding(.vertical, 20)  // Space for focus scale
            }
        }
    }
}
```

## 5. Modal presentations

On tvOS, sheets always present **full-screen** (no half-sheets, no detents).

```swift
// Sheet — full screen on tvOS
struct ContentView: View {
    @State private var showingDetail = false

    var body: some View {
        Button("Show Detail") {
            showingDetail = true
        }
        .sheet(isPresented: $showingDetail) {
            DetailSheet()
        }
    }
}

// fullScreenCover — also full screen (same visual as .sheet on tvOS)
.fullScreenCover(isPresented: $showingPlayer) {
    PlayerView(url: videoURL)
}

// Alert
.alert("Delete Item?", isPresented: $showingDelete) {
    Button("Cancel", role: .cancel) { }
    Button("Delete", role: .destructive) { deleteItem() }
} message: {
    Text("This action cannot be undone.")
}

// Confirmation dialog — centered on tvOS
.confirmationDialog("Options", isPresented: $showingOptions) {
    Button("Add to Watchlist") { addToWatchlist() }
    Button("Share") { share() }
    Button("Report") { report() }
    Button("Cancel", role: .cancel) { }
}
```

## 6. Search

tvOS search uses a full-screen keyboard interface. `.searchable()` on tvOS opens a dedicated search screen.

```swift
struct SearchView: View {
    @State private var query = ""
    @State private var results: [SearchResult] = []

    var body: some View {
        NavigationStack {
            Group {
                if query.isEmpty {
                    // Show suggestions / trending
                    TrendingGrid()
                } else if results.isEmpty {
                    ContentUnavailableView.search(text: query)
                } else {
                    // Show results in shelves by type
                    ScrollView {
                        LazyVStack(spacing: 50) {
                            if !results.movies.isEmpty {
                                ShelfRow(title: "Movies", items: results.movies)
                            }
                            if !results.shows.isEmpty {
                                ShelfRow(title: "TV Shows", items: results.shows)
                            }
                            if !results.people.isEmpty {
                                PeopleRow(title: "People", items: results.people)
                            }
                        }
                    }
                }
            }
            .searchable(text: $query, prompt: "Movies, Shows, People")
            .task(id: query) {
                // Debounced search
                try? await Task.sleep(for: .milliseconds(300))
                await performSearch(query)
            }
        }
    }
}
```

### Search keyboard on tvOS
- Full-screen on-screen keyboard appears
- User can swipe on remote to select letters
- Siri dictation available (hold Siri button)
- Results update as user types
- Recently searched items shown below keyboard

## 7. Settings / Preferences

```swift
struct SettingsView: View {
    @AppStorage("quality") private var quality = StreamQuality.auto
    @AppStorage("subtitles") private var subtitlesEnabled = true
    @AppStorage("autoPlay") private var autoPlayNext = true

    var body: some View {
        NavigationStack {
            List {
                Section("Playback") {
                    Picker("Quality", selection: $quality) {
                        ForEach(StreamQuality.allCases) { q in
                            Text(q.displayName).tag(q)
                        }
                    }

                    Toggle("Subtitles", isOn: $subtitlesEnabled)
                    Toggle("Auto-Play Next", isOn: $autoPlayNext)
                }

                Section("Account") {
                    NavigationLink("Profile") {
                        ProfileView()
                    }
                    NavigationLink("Subscription") {
                        SubscriptionView()
                    }
                    Button("Sign Out", role: .destructive) {
                        signOut()
                    }
                }

                Section("About") {
                    LabeledContent("Version", value: "1.0.0")
                    NavigationLink("Privacy Policy") {
                        WebView(url: privacyURL)
                    }
                }
            }
            .navigationTitle("Settings")
        }
    }
}
```

## 8. Rules and anti-patterns

### DO:
- Use `TabView` for top-level navigation (3-5 tabs)
- Wrap each tab in its own `NavigationStack`
- Use `.focusSection()` on each horizontal shelf
- Use `.buttonStyle(.card)` for poster NavigationLinks
- Handle Menu/Back button gracefully in modals
- Use `ContentUnavailableView` for empty states
- Group search results by type in horizontal shelves

### DON'T:
- Don't place tab bar at the bottom (it's always at the top on tvOS)
- Don't create bottom bars or toolbars (tvOS has no bottom chrome)
- Don't use half-sheets or `.presentationDetents` (everything is full screen)
- Don't use `.presentationDragIndicator` (no drag on TV)
- Don't nest `NavigationStack` inside `NavigationStack`
- Don't use more than 7 tabs
- Don't forget that search opens a full keyboard screen
- Don't use pull-to-refresh (`.refreshable`) — not a TV pattern
- Don't use swipe-to-delete on list rows — use context menu instead
