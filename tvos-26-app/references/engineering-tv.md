# tvOS Engineering — Complete Reference

## Table of contents
1. Architecture patterns
2. State management
3. Concurrency
4. Networking
5. Persistence (SwiftData)
6. Framework availability
7. Performance on Apple TV
8. Testing
9. App lifecycle
10. Multi-target (iOS + tvOS)
11. Rules and anti-patterns

---

## 1. Architecture patterns

### Recommended: @Observable + Environment injection

```swift
// Model layer
@Observable
class CatalogService {
    var shelves: [Shelf] = []
    var isLoading = false
    var error: CatalogError?

    func loadCatalog() async {
        isLoading = true
        defer { isLoading = false }

        do {
            shelves = try await api.fetchCatalog()
        } catch {
            self.error = .fetchFailed(error)
        }
    }
}

// App root
@main
struct MyTVApp: App {
    @State private var catalog = CatalogService()
    @State private var player = PlayerService()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(catalog)
                .environment(player)
        }
    }
}

// View layer
struct HomeView: View {
    @Environment(CatalogService.self) var catalog

    var body: some View {
        Group {
            if catalog.isLoading {
                ProgressView()
            } else {
                BrowseContent(shelves: catalog.shelves)
            }
        }
        .task {
            await catalog.loadCatalog()
        }
    }
}
```

### View-local state

```swift
struct DetailView: View {
    let item: ContentItem
    @State private var isFavorite = false
    @State private var showPlayer = false

    var body: some View {
        // Local state — no need for model class
        ScrollView { /* ... */ }
    }
}
```

## 2. State management

Same patterns as iOS 26:

| Property Wrapper | Usage |
|-----------------|-------|
| `@State` | View-local value types |
| `@Binding` | Two-way connection to parent's @State |
| `@Environment` | Inject @Observable services or system values |
| `@Bindable` | Create bindings from @Observable properties |
| `@Observable` | Class-level state (replaces ObservableObject) |
| `@AppStorage` | UserDefaults-backed persistence |

```swift
@Observable
class PlayerState {
    var isPlaying = false
    var currentTime: TimeInterval = 0
    var duration: TimeInterval = 0
    var currentItem: MediaItem?

    var progress: Double {
        guard duration > 0 else { return 0 }
        return currentTime / duration
    }
}

struct TransportBar: View {
    @Bindable var player: PlayerState

    var body: some View {
        VStack {
            ProgressView(value: player.progress)
            HStack {
                Button(player.isPlaying ? "Pause" : "Play") {
                    player.isPlaying.toggle()
                }
            }
        }
    }
}
```

## 3. Concurrency

Swift 6.2 structured concurrency — same as iOS:

```swift
// .task {} for view-lifecycle-tied async work
struct CatalogView: View {
    @State private var items: [Item] = []

    var body: some View {
        List(items) { item in /* ... */ }
            .task {
                // Automatically cancelled on view disappear
                items = await fetchItems()
            }
            .task(id: selectedCategory) {
                // Re-runs when selectedCategory changes
                items = await fetchItems(for: selectedCategory)
            }
    }
}

// Parallel fetching
func loadHomeScreen() async throws -> HomeData {
    async let trending = api.fetchTrending()
    async let newReleases = api.fetchNewReleases()
    async let continueWatching = api.fetchContinueWatching()

    return HomeData(
        trending: try await trending,
        newReleases: try await newReleases,
        continueWatching: try await continueWatching
    )
}

// TaskGroup for dynamic parallelism
func loadAllShelves(categories: [Category]) async throws -> [Shelf] {
    try await withThrowingTaskGroup(of: Shelf.self) { group in
        for category in categories {
            group.addTask {
                try await api.fetchShelf(for: category)
            }
        }
        return try await group.reduce(into: []) { $0.append($1) }
    }
}

// Actor for thread-safe state
actor DownloadManager {
    private var downloads: [String: DownloadTask] = [:]

    func startDownload(for item: MediaItem) async throws {
        // Safe: actor isolates mutable state
    }
}
```

## 4. Networking

```swift
// URLSession async/await — same as iOS
struct APIClient {
    let baseURL: URL
    let session: URLSession

    func fetch<T: Decodable>(_ path: String) async throws -> T {
        let url = baseURL.appendingPathComponent(path)
        let (data, response) = try await session.data(from: url)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw APIError.badResponse
        }

        return try JSONDecoder().decode(T.self, from: data)
    }

    // Streaming responses
    func streamContent(_ url: URL) -> AsyncThrowingStream<Data, Error> {
        AsyncThrowingStream { continuation in
            let task = session.dataTask(with: url) { data, _, error in
                if let error { continuation.finish(throwing: error) }
                if let data { continuation.yield(data) }
                continuation.finish()
            }
            task.resume()
            continuation.onTermination = { _ in task.cancel() }
        }
    }
}

// Image loading (tvOS has no system image cache like iOS Photos)
struct RemoteImage: View {
    let url: URL

    var body: some View {
        AsyncImage(url: url) { phase in
            switch phase {
            case .empty:
                ProgressView()
            case .success(let image):
                image.resizable().aspectRatio(contentMode: .fill)
            case .failure:
                Image(systemName: "photo")
                    .foregroundStyle(.tertiary)
            @unknown default:
                EmptyView()
            }
        }
    }
}
```

## 5. Persistence (SwiftData)

```swift
import SwiftData

// Model
@Model
class WatchlistItem {
    var contentID: String
    var title: String
    var posterURL: URL
    var addedDate: Date
    var lastWatchedDate: Date?
    var progress: Double  // 0.0 to 1.0

    init(contentID: String, title: String, posterURL: URL) {
        self.contentID = contentID
        self.title = title
        self.posterURL = posterURL
        self.addedDate = .now
    }
}

// Container setup
@main
struct MyTVApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: [WatchlistItem.self])
    }
}

// Query in views
struct WatchlistView: View {
    @Query(sort: \WatchlistItem.addedDate, order: .reverse)
    var watchlist: [WatchlistItem]

    @Environment(\.modelContext) var context

    var body: some View {
        List(watchlist) { item in
            WatchlistRow(item: item)
        }
    }

    func addToWatchlist(_ content: ContentItem) {
        let item = WatchlistItem(
            contentID: content.id,
            title: content.title,
            posterURL: content.posterURL
        )
        context.insert(item)
    }
}
```

### CloudKit sync with SwiftData

Sync watchlist and preferences between tvOS and iOS via CloudKit. Both platforms support CloudKit fully.

```swift
// Shared @Model (compile into both iOS and tvOS targets)
@Model
class WatchlistItem {
    @Attribute(.unique) var contentID: String
    var title: String
    var progress: Double  // 0.0–1.0
    var lastUpdated: Date

    init(contentID: String, title: String) {
        self.contentID = contentID
        self.title = title
        self.progress = 0.0
        self.lastUpdated = .now
    }
}

// App root — enable CloudKit sync
@main
struct MyTVApp: App {
    var body: some Scene {
        WindowGroup { ContentView() }
        .modelContainer(
            for: WatchlistItem.self,
            configurations: ModelConfiguration(
                cloudKitDatabase: .automatic  // Same iCloud container as iOS target
            )
        )
    }
}
```

**Requirements**: CloudKit capability on both iOS and tvOS targets with same container ID. `@Model` properties must be Optional or have default values.

See `references/companion-handoff.md` for full cross-device sync patterns and conflict resolution.

## 6. Framework availability

### Available on tvOS 26

| Framework | Notes |
|-----------|-------|
| **SwiftUI** | Full support (with TV-specific behavior) |
| **SwiftData** | Full support including model inheritance |
| **URLSession** | Full async/await networking |
| **AVFoundation / AVKit** | Full media playback |
| **GameController** | Siri Remote + MFi + DualSense + Xbox |
| **TVUIKit** | TV-specific UI components |
| **TVServices** | Top Shelf, TV Provider auth |
| **Metal / MetalKit** | GPU rendering (Metal 4 on Apple TV 4K 3rd gen) |
| **SpriteKit** | 2D game framework |
| **SceneKit** | 3D scene framework |
| **CoreAnimation** | Low-level animations |
| **CoreGraphics** | 2D drawing |
| **CoreImage** | Image processing |
| **CoreText** | Advanced typography |
| **CloudKit** | iCloud sync |
| **Combine** | Reactive framework (prefer async/await for new code) |
| **Multipeer Connectivity** | Local peer-to-peer |
| **Network** | Low-level networking |
| **UserNotifications** | Local notifications (limited) |
| **StoreKit 2** | In-app purchases, subscriptions |
| **CryptoKit / Security** | Encryption, Keychain |
| **Swift Charts** | Data visualization |
| **OSLog / os.log** | Structured logging |
| **TVMLKit** | Legacy — deprecated, avoid in new code |

### NOT available on tvOS

| Framework | Why |
|-----------|-----|
| **Foundation Models** | No capable Neural Engine (A15 in Apple TV 4K) |
| **UIKit (full)** | Subset available — no UITapGestureRecognizer on views (use press types) |
| **Camera / AVCapture** | No camera |
| **PhotoKit / PHPicker** | No photo library |
| **CoreLocation** | No GPS |
| **MapKit** | No maps |
| **HealthKit** | No health sensors |
| **ARKit / RealityKit** | No AR |
| **WidgetKit** | No home screen widgets |
| **ActivityKit** | No Live Activities |
| **PaperKit** | No document scanning |
| **SpeechAnalyzer** | No on-device speech processing |
| **Image Playground** | No AI image generation |
| **Writing Tools API** | No text editing context |
| **CoreHaptics / UIFeedbackGenerator** | No Taptic Engine |
| **CoreBluetooth** | Limited (game controllers go through GameController framework) |
| **NearbyInteraction** | No UWB chip |
| **CoreMotion** | No accelerometer/gyroscope on Apple TV box |
| **CoreNFC** | No NFC |
| **LocalAuthentication** | No Face ID / Touch ID on TV |
| **BackgroundTasks** | Limited — no BGAppRefreshTask in the iOS sense |
| **CallKit** | No phone calls |
| **EventKit** | No calendar |
| **Contacts** | No contacts database |
| **Messages** | No messaging |

## 7. Performance on Apple TV

### Hardware specs

| Apple TV model | Chip | RAM | Storage | Metal |
|---------------|------|-----|---------|-------|
| Apple TV 4K (3rd gen, 2022) | A15 Bionic | 4GB | 64/128GB | Metal 4 |
| Apple TV 4K (2nd gen, 2021) | A12 Bionic | 4GB | 32/64GB | Metal 3 |
| Apple TV 4K (1st gen, 2017) | A10X Fusion | 3GB | 32/64GB | Metal 2 |
| Apple TV HD (2015) | A8 | 2GB | 32GB | Metal |

### Performance tips

```swift
// 1. Use LazyVStack/LazyHStack for scrolling content
ScrollView(.horizontal) {
    LazyHStack(spacing: 40) {  // Only renders visible items
        ForEach(items) { item in
            PosterCard(item: item)
        }
    }
}

// 2. Prefetch images for smooth scrolling
struct ShelfRow: View {
    let items: [ContentItem]
    @State private var imageCache = ImageCache()

    var body: some View {
        ScrollView(.horizontal) {
            LazyHStack(spacing: 40) {
                ForEach(items) { item in
                    CachedImage(url: item.posterURL, cache: imageCache)
                }
            }
        }
        .task {
            // Prefetch first 10 images
            await imageCache.prefetch(urls: items.prefix(10).map(\.posterURL))
        }
    }
}

// 3. Avoid complex view hierarchies in frequently rendered cells
// Keep poster cards simple — minimal nesting, few modifiers

// 4. Use .drawingGroup() for complex rendering
ComplexView()
    .drawingGroup()  // Composites into a single Metal texture

// 5. Profile with Instruments
// Use SwiftUI Instruments template for:
// - View body evaluations
// - Animation hitches
// - GPU overdraw
```

## 8. Testing

```swift
// Swift Testing (unit tests)
import Testing

@Test("Catalog loads shelves")
func catalogLoads() async throws {
    let service = CatalogService(api: MockAPI())
    await service.loadCatalog()
    #expect(service.shelves.count > 0)
    #expect(service.error == nil)
}

@Test("Watchlist item persists")
func watchlistPersists() throws {
    let config = ModelConfiguration(isStoredInMemoryOnly: true)
    let container = try ModelContainer(for: WatchlistItem.self, configurations: config)
    let context = ModelContext(container)

    let item = WatchlistItem(contentID: "123", title: "Test", posterURL: URL(string: "https://example.com")!)
    context.insert(item)
    try context.save()

    let items = try context.fetch(FetchDescriptor<WatchlistItem>())
    #expect(items.count == 1)
    #expect(items.first?.title == "Test")
}

// XCTest UI tests (tvOS simulator)
import XCTest

class TVBrowseUITests: XCTestCase {
    let app = XCUIApplication()

    override func setUp() {
        app.launch()
    }

    func testNavigateToDetail() {
        // Focus on first poster
        let firstPoster = app.buttons["MoviePoster_0"]
        XCTAssertTrue(firstPoster.waitForExistence(timeout: 5))

        // Press select (clickpad)
        XCUIRemote.shared.press(.select)

        // Verify detail view appears
        let detailTitle = app.staticTexts["DetailTitle"]
        XCTAssertTrue(detailTitle.waitForExistence(timeout: 3))
    }

    func testMenuButtonGoesBack() {
        // Navigate forward
        XCUIRemote.shared.press(.select)

        // Press menu to go back
        XCUIRemote.shared.press(.menu)

        // Verify we're back on browse
        let browseTitle = app.navigationBars["Browse"]
        XCTAssertTrue(browseTitle.exists)
    }

    // XCUIRemote presses for tvOS:
    // .select — clickpad click
    // .menu — back button
    // .playPause — play/pause
    // .up, .down, .left, .right — directional
    // .home — home button
}
```

## 9. App lifecycle

```swift
@main
struct MyTVApp: App {
    @State private var catalog = CatalogService()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(catalog)
                .onAppear {
                    // App launched or became active
                }
        }
    }
}

// Scene phase observation
struct ContentView: View {
    @Environment(\.scenePhase) var scenePhase

    var body: some View {
        TabView { /* ... */ }
            .onChange(of: scenePhase) { _, newPhase in
                switch newPhase {
                case .active:
                    // App is foreground
                    refreshContentIfNeeded()
                case .inactive:
                    // App is transitioning (e.g., Control Center)
                    saveState()
                case .background:
                    // App is in background
                    pauseExpensiveOperations()
                @unknown default:
                    break
                }
            }
    }
}

// Deep linking from Top Shelf or Universal Links
.onOpenURL { url in
    handleDeepLink(url)
}
```

## 10. Multi-target (iOS + tvOS)

### Project setup

```swift
// In project.yml / Xcode:
// - Shared target for models, networking, business logic
// - iOS-specific target for iOS views
// - tvOS-specific target for tvOS views

// Shared code
// Sources/Shared/Models/Movie.swift
// Sources/Shared/Services/APIClient.swift
// Sources/Shared/Services/CatalogService.swift

// Platform-specific
// Sources/iOS/Views/HomeView_iOS.swift
// Sources/tvOS/Views/HomeView_tvOS.swift
```

### Conditional compilation

```swift
// Shared model
struct ContentItem: Identifiable, Codable {
    let id: String
    let title: String
    let posterURL: URL

    #if os(tvOS)
    // TV needs larger poster
    var displayPosterSize: CGSize { CGSize(width: 250, height: 375) }
    #else
    var displayPosterSize: CGSize { CGSize(width: 120, height: 180) }
    #endif
}

// Shared view with platform adaptations
struct PosterCard: View {
    let item: ContentItem

    var body: some View {
        AsyncImage(url: item.posterURL) { image in
            image.resizable().aspectRatio(2/3, contentMode: .fill)
        } placeholder: {
            RoundedRectangle(cornerRadius: cornerRadius).fill(.quaternary)
        }
        .frame(width: item.displayPosterSize.width,
               height: item.displayPosterSize.height)
        .clipShape(RoundedRectangle(cornerRadius: cornerRadius))
        #if os(tvOS)
        .focusable()
        #endif
    }

    var cornerRadius: CGFloat {
        #if os(tvOS)
        12
        #else
        8
        #endif
    }
}
```

## 11. Rules and anti-patterns

### DO:
- Use `@Observable` for all shared state
- Use `.task { }` for async work tied to view lifecycle
- Use `async let` for parallel data loading
- Use SwiftData for local persistence
- Handle `scenePhase` changes
- Handle `.onOpenURL` for deep links
- Share business logic between iOS and tvOS targets
- Use `#if os(tvOS)` for platform-specific code
- Profile with Instruments on Apple TV hardware

### DON'T:
- Don't use `ObservableObject` / `@Published` in new code
- Don't use Combine for new async work (prefer structured concurrency)
- Don't import frameworks that don't exist on tvOS (build will fail)
- Don't assume iOS hardware features (camera, GPS, haptics)
- Don't block the main thread with networking or heavy computation
- Don't store secrets in UserDefaults — use Keychain
- Don't forget the UIScene lifecycle requirement (mandatory in release following iOS 26)
- Don't skip testing with `XCUIRemote` — it simulates real remote input
- Don't assume Apple TV 4K performance on older models — test on lower-spec hardware

## 12. TV Provider SSO (VideoSubscriberAccount)

TV Provider Single Sign-On lets users who have a cable/satellite subscription authenticate automatically across streaming apps without re-entering credentials. Zero Sign-On (ZSO) silently authenticates users on participating ISP networks (Comcast, Charter, etc.).

**Required**: TV Provider entitlement (`com.apple.developer.video-subscriber-single-sign-on`) — apply via Apple's Video Subscriber Account Framework program.

```swift
import VideoSubscriberAccount

class TVProviderAuthManager {
    let accountManager = VSAccountManager()

    // Silent check (Zero Sign-On) — call on app launch
    func checkSilentAuth() async -> Bool {
        return await withCheckedContinuation { continuation in
            let request = VSAccountMetadataRequest()
            request.includeAccountProviderIdentifier = true
            request.isInterruptionAllowed = false  // Silent — no UI shown

            accountManager.enqueue(request) { metadata, error in
                continuation.resume(returning: metadata?.accountProviderIdentifier != nil)
            }
        }
    }

    // Interactive sign-in — call after user taps "Sign in with TV Provider"
    func signIn() async throws -> VSAccountMetadata {
        return try await withCheckedThrowingContinuation { continuation in
            let request = VSAccountMetadataRequest()
            request.includeAccountProviderIdentifier = true
            request.isInterruptionAllowed = true        // Shows provider picker
            request.supportedAuthenticationSchemes = [.saml]

            accountManager.enqueue(request) { metadata, error in
                if let error { continuation.resume(throwing: error) }
                else if let metadata { continuation.resume(returning: metadata) }
            }
        }
    }
}
```

### Sign-in flow

```swift
struct SignInView: View {
    @State private var authManager = TVProviderAuthManager()

    var body: some View {
        VStack(spacing: 40) {
            // Option 1: TV Provider SSO (cable subscribers)
            Button("Sign in with TV Provider") {
                Task {
                    let metadata = try? await authManager.signIn()
                    // Exchange SAML token for your app's session
                }
            }
            .buttonStyle(.glassProminent)

            // Option 2: QR code (direct account sign-in)
            // See references/companion-handoff.md for QR implementation
            Button("Sign in with Email") {
                showQRSignIn = true
            }
            .buttonStyle(.glass)
        }
        .task {
            // Attempt silent ZSO on appear
            let authed = await authManager.checkSilentAuth()
            if authed { /* proceed */ }
        }
    }
}
```

**Rules:**
- Always provide an alternative sign-in path — not all users have cable subscriptions
- Use `isInterruptionAllowed = false` for ZSO attempts on app launch
- Only show the provider picker after the user explicitly initiates sign-in

## 13. Multi-User Profile Support

Apple TV supports multiple user profiles (added in tvOS 14). Users switch profiles via Control Center. Apps should personalize content per profile.

### Detecting profile changes (TVUserManager)

```swift
import TVServices

class ProfileManager {
    let userManager = TVUserManager()

    var currentUserID: String {
        userManager.currentUser?.identifier ?? "default"
    }

    func observeProfileChanges(onChange: @escaping (String) -> Void) {
        NotificationCenter.default.addObserver(
            forName: TVUserManager.currentUserDidChangeNotification,
            object: userManager,
            queue: .main
        ) { [weak self] _ in
            onChange(self?.currentUserID ?? "default")
        }
    }
}
```

### Per-user data patterns

```swift
// Pattern 1: CloudKit (recommended) — automatically per-user
// Each Apple TV user signs into a different Apple ID.
// SwiftData + cloudKitDatabase: .automatic is scoped per-Apple-ID automatically.

// Pattern 2: Local storage keyed by user ID
func userPreferencesURL(for userID: String) -> URL {
    FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
        .appendingPathComponent("prefs_\(userID).json")
}

// Pattern 3: @AppStorage with user-scoped keys
@AppStorage("autoplay_next") private var autoplayNext = true
// For per-user: append userID to key — "autoplay_next_<userID>"
```

### Responding to profile switches in SwiftUI

```swift
struct HomeView: View {
    @State private var profileManager = ProfileManager()
    @State private var userID: String = TVUserManager().currentUser?.identifier ?? "default"

    var body: some View {
        PersonalizedContent(userID: userID)
            .onReceive(
                NotificationCenter.default.publisher(
                    for: TVUserManager.currentUserDidChangeNotification
                )
            ) { _ in
                userID = TVUserManager().currentUser?.identifier ?? "default"
                // CloudKit data reloads automatically via @Query
                // Local data: reload manually
            }
    }
}
```

**Rules:**
- Don't build a custom profile switcher UI — Apple TV handles this in Control Center
- Store all personalized data per-user (CloudKit handles this automatically; local storage needs explicit keying)
- Reload personalized content on `TVUserManager.currentUserDidChangeNotification`
- Non-personalized content (catalog, app settings) doesn't need profile handling