# iOS 26 Engineering — Architecture, Data, Concurrency, Testing

## Table of contents
1. App architecture patterns
2. State management hierarchy
3. Swift 6.2 concurrency model
4. SwiftData persistence
5. Networking patterns
6. Foundation Models (on-device AI)
7. App lifecycle and scenes
8. App Intents and system integration
9. UIKit interop from SwiftUI
10. Error handling patterns
11. Testing strategies
12. Xcode 26 tooling
13. Performance profiling
14. Security and privacy
15. Distribution and deployment

---

## 1. App architecture patterns

### Recommended: feature-based modules with @Observable models

```
MyApp/
├── App/
│   ├── MyApp.swift              // @main, WindowGroup, root navigation
│   └── AppState.swift           // Global @Observable state
├── Features/
│   ├── Feed/
│   │   ├── FeedView.swift       // SwiftUI view
│   │   ├── FeedModel.swift      // @Observable business logic
│   │   └── FeedService.swift    // Network/data access
│   ├── Profile/
│   │   ├── ProfileView.swift
│   │   ├── ProfileModel.swift
│   │   └── ProfileService.swift
│   └── Settings/
├── Shared/
│   ├── Models/                  // SwiftData models, DTOs
│   ├── Services/                // Shared networking, auth, analytics
│   ├── Components/              // Reusable UI components
│   └── Extensions/              // Swift/SwiftUI extensions
├── Resources/
│   ├── Assets.xcassets
│   └── Localizable.xcstrings
└── Tests/
```

### Key architectural decisions

**@Observable replaces ObservableObject** — Property-level tracking means views only re-render when accessed properties change. This is the default for iOS 17+ and mandatory style for iOS 26.

**Views are dumb** — Views declare layout and bindings. Business logic lives in @Observable model classes. Views never call network/persistence directly.

**Dependency injection via Environment** — Pass services and models through `.environment()` rather than init parameters:

```swift
@Observable class AuthService {
    private(set) var currentUser: User?
    func signIn(email: String, password: String) async throws { ... }
    func signOut() { ... }
}

// Inject at root
@main
struct MyApp: App {
    @State private var auth = AuthService()
    
    var body: some Scene {
        WindowGroup {
            RootView()
                .environment(auth)
        }
    }
}

// Consume anywhere
struct ProfileView: View {
    @Environment(AuthService.self) private var auth
    
    var body: some View {
        if let user = auth.currentUser {
            UserProfile(user: user)
        }
    }
}
```

**Unidirectional data flow** — State changes flow: User Action → Model Method → State Mutation → View Update (automatic via @Observable). Never mutate state from inside `body`.

### When to use MVVM vs other patterns
- **Simple screens**: View + @Observable model is enough (no separate ViewModel)
- **Complex screens**: Dedicated ViewModel (@Observable class) mediating between View and Services
- **Shared state**: @Observable class injected via .environment()
- **Navigation state**: @Observable NavigationModel with @State path in view

## 2. State management hierarchy

### Complete property wrapper guide

| Wrapper | Owns data? | Scope | Use for |
|---------|-----------|-------|---------|
| `@State` | Yes | View-local | Simple UI state (toggles, text fields, sheet visibility) |
| `@Binding` | No | Parent→Child | Passing mutable state to child views |
| `@Bindable` | No | View-local | Creating bindings from @Observable properties |
| `@Environment` | No | Injected tree | Reading injected @Observable objects or system values |
| `@Observable` (macro) | — | Class | Making a reference type's properties observable |
| `@AppStorage` | Yes | UserDefaults | Small persistent preferences (strings, ints, bools) |
| `@SceneStorage` | Yes | Scene-specific | Scene restoration state (scroll position, tab selection) |
| `@FocusState` | Yes | View-local | Keyboard and accessibility focus management |
| `@AccessibilityFocusState` | Yes | View-local | VoiceOver focus management |

### @Observable in detail (iOS 17+, default for iOS 26)

```swift
import Observation

@Observable class TaskStore {
    var tasks: [Task] = []          // Tracked automatically
    var isLoading = false           // Tracked automatically
    @ObservationIgnored var cache: NSCache<...>  // Not tracked
    
    func addTask(_ task: Task) {
        tasks.append(task)          // Views accessing `tasks` re-render
        // Views only accessing `isLoading` do NOT re-render
    }
}
```

### Observations (Swift 6.2 / iOS 26 — AsyncSequence-based observation)

```swift
import Observation

let store = TaskStore()

// Continuous observation via AsyncSequence
for await taskCount in Observations(of: store) { store.tasks.count } {
    print("Task count changed to: \(taskCount)")
}
```

Replaces the limited `withObservationTracking` from iOS 17. Enables non-SwiftUI code (services, coordinators, UIKit controllers) to observe @Observable changes continuously.

### UIKit automatic observation (iOS 26)

```swift
class ProfileViewController: UIViewController {
    let model: ProfileModel  // @Observable

    override func updateProperties() {
        super.updateProperties()
        // Any @Observable property accessed here is automatically tracked
        // View updates when tracked properties change
        nameLabel.text = model.name
        avatarView.image = model.avatar
    }
}
```

`updateProperties()` runs before `layoutSubviews()`. It replaces the manual `didSet { updateUI() }` pattern. Enabled by default in iOS 26.

## 3. Swift 6.2 concurrency model

### Default MainActor isolation

Swift 6.2 allows configuring entire modules or files to default to `@MainActor`:

```swift
// In Package.swift or build settings
// defaultIsolation: MainActor

// OR per-file
@MainActor  // All declarations in this file run on MainActor
import SwiftUI

class MyModel { ... }  // Implicitly @MainActor
```

This simplifies most app code since UI apps are predominantly main-thread. Opt out for specific types:

```swift
nonisolated class NetworkService { ... }  // Explicitly off MainActor
```

### Structured concurrency patterns

```swift
// Sequential async
func loadProfile() async throws -> Profile {
    let user = try await fetchUser()
    let avatar = try await fetchAvatar(for: user)
    return Profile(user: user, avatar: avatar)
}

// Parallel async
func loadDashboard() async throws -> Dashboard {
    async let stats = fetchStats()
    async let notifications = fetchNotifications()
    async let recommendations = fetchRecommendations()
    return try await Dashboard(
        stats: stats,
        notifications: notifications,
        recommendations: recommendations
    )
}

// Task groups for dynamic parallelism
func loadAllImages(urls: [URL]) async throws -> [UIImage] {
    try await withThrowingTaskGroup(of: (Int, UIImage).self) { group in
        for (index, url) in urls.enumerated() {
            group.addTask {
                let (data, _) = try await URLSession.shared.data(from: url)
                guard let image = UIImage(data: data) else {
                    throw ImageError.decodingFailed
                }
                return (index, image)
            }
        }
        var results = [(Int, UIImage)]()
        for try await result in group {
            results.append(result)
        }
        return results.sorted(by: { $0.0 < $1.0 }).map(\.1)
    }
}
```

### Task management in views

```swift
struct FeedView: View {
    @Environment(FeedModel.self) var model
    
    var body: some View {
        List(model.items) { item in
            ItemRow(item: item)
        }
        .task {
            // Tied to view lifecycle — auto-cancelled when view disappears
            await model.loadFeed()
        }
        .refreshable {
            await model.refresh()
        }
        .task(id: selectedCategory) {
            // Re-runs when selectedCategory changes
            await model.loadCategory(selectedCategory)
        }
    }
}
```

### Actor isolation

```swift
actor ImageCache {
    private var cache: [URL: UIImage] = [:]
    
    func image(for url: URL) -> UIImage? {
        cache[url]
    }
    
    func store(_ image: UIImage, for url: URL) {
        cache[url] = image
    }
}

// Usage
let cache = ImageCache()
if let cached = await cache.image(for: url) {
    return cached
}
```

### Sendable compliance
Mark types crossing isolation boundaries as `Sendable`:
```swift
struct UserDTO: Sendable, Codable {
    let id: UUID
    let name: String
    let email: String
}
```

Value types (structs, enums) with Sendable properties are implicitly Sendable. Reference types need explicit conformance or `@unchecked Sendable` (use carefully).

## 4. SwiftData persistence

### Model definition
```swift
import SwiftData

@Model
class Task {
    var title: String
    var isCompleted: Bool
    var createdAt: Date
    var category: Category?  // Relationship
    
    @Attribute(.unique) var id: UUID
    @Attribute(.externalStorage) var imageData: Data?  // Large blobs stored externally
    @Attribute(.transformable(by: ColorTransformer.self))
    var color: UIColor?
    
    init(title: String) {
        self.id = UUID()
        self.title = title
        self.isCompleted = false
        self.createdAt = .now
    }
}

@Model
class Category {
    var name: String
    @Relationship(deleteRule: .cascade, inverse: \Task.category)
    var tasks: [Task] = []
    
    init(name: String) { self.name = name }
}
```

### iOS 26: Model inheritance (new)
```swift
@Model
class BaseItem {
    var title: String
    var createdAt: Date
}

@Model
class NoteItem: BaseItem {
    var body: String
}

@Model
class BookmarkItem: BaseItem {
    var url: URL
}
```

### Container setup
```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: [Task.self, Category.self])
    }
}
```

### Querying
```swift
struct TaskListView: View {
    @Query(
        filter: #Predicate<Task> { !$0.isCompleted },
        sort: \Task.createdAt,
        order: .reverse
    ) var pendingTasks: [Task]
    
    @Environment(\.modelContext) var context
    
    var body: some View {
        List(pendingTasks) { task in
            TaskRow(task: task)
                .swipeActions {
                    Button("Complete") {
                        task.isCompleted = true
                        // No explicit save — SwiftData auto-saves
                    }
                }
        }
    }
    
    func addTask(title: String) {
        let task = Task(title: title)
        context.insert(task)
    }
    
    func deleteTask(_ task: Task) {
        context.delete(task)
    }
}
```

### Background operations
```swift
let container = try ModelContainer(for: Task.self)
let context = ModelContext(container)  // Background context

Task.detached {
    let context = ModelContext(container)
    // Perform bulk operations
    let tasks = try context.fetch(FetchDescriptor<Task>())
    for task in tasks where task.isCompleted {
        context.delete(task)
    }
    try context.save()
}
```

### CloudKit sync

SwiftData supports automatic iCloud sync via CloudKit with one line of configuration.

```swift
// Enable CloudKit sync — add CloudKit capability in Xcode first
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup { ContentView() }
        .modelContainer(
            for: [Task.self, Category.self],
            configurations: ModelConfiguration(
                cloudKitDatabase: .automatic  // Uses default iCloud container
            )
        )
    }
}
```

**CloudKit sync requirements:**
- Add CloudKit capability to target (same container ID across devices/platforms)
- `@Model` properties must be Optional or have default values (CloudKit limitation)
- No optional → required or non-optional → optional changes without migration
- Avoid storing large binary blobs — use external storage or store URLs

**Encrypt sensitive fields:**
```swift
@Model class UserPreferences {
    @Attribute(.allowsCloudEncryption) var authToken: String?  // Stored encrypted in CloudKit
    var theme: String = "system"
}
```

**Conflict resolution**: CloudKit private database uses last-writer-wins by default. For user-generated content where "most recent wins" isn't right, add a `lastModified: Date` field and resolve in your model methods.

**Limitations in iOS 26**: Only private databases (per-user iCloud). Shared/public CloudKit databases require `NSPersistentCloudKitContainer` (UIKit/Core Data) rather than SwiftData.

### When to use SwiftData vs Core Data
- **SwiftData**: New projects, Swift-native, automatic migration, simpler API, iOS 17+
- **Core Data**: Legacy projects, complex migration needs, need NSFetchedResultsController, pre-iOS 17

## 5. Networking patterns

### URLSession with async/await
```swift
struct APIClient {
    let session: URLSession
    let decoder: JSONDecoder
    let baseURL: URL
    
    func fetch<T: Decodable>(_ path: String) async throws -> T {
        let url = baseURL.appendingPathComponent(path)
        let (data, response) = try await session.data(from: url)
        
        guard let http = response as? HTTPURLResponse,
              200..<300 ~= http.statusCode else {
            throw APIError.invalidResponse
        }
        
        return try decoder.decode(T.self, from: data)
    }
    
    func upload<T: Encodable, R: Decodable>(
        _ path: String,
        body: T
    ) async throws -> R {
        var request = URLRequest(url: baseURL.appendingPathComponent(path))
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.httpBody = try JSONEncoder().encode(body)
        
        let (data, _) = try await session.data(for: request)
        return try decoder.decode(R.self, from: data)
    }
}
```

### Image caching with URLCache + AsyncImage
```swift
// Configure cache at app startup
let cache = URLCache(
    memoryCapacity: 50 * 1024 * 1024,   // 50 MB memory
    diskCapacity: 200 * 1024 * 1024      // 200 MB disk
)
URLCache.shared = cache

// AsyncImage uses URLSession.shared which respects URLCache
AsyncImage(url: imageURL) { phase in ... }
```

### Pagination pattern
```swift
@Observable class PaginatedFeed {
    private(set) var items: [Item] = []
    private(set) var isLoading = false
    private var currentPage = 0
    private var hasMore = true
    
    func loadMore() async {
        guard !isLoading, hasMore else { return }
        isLoading = true
        defer { isLoading = false }
        
        let page = try? await api.fetchItems(page: currentPage + 1)
        if let page {
            items.append(contentsOf: page.items)
            hasMore = page.hasNextPage
            currentPage += 1
        }
    }
}

// In view — trigger on last item appearance
List(model.items) { item in
    ItemRow(item: item)
        .onAppear {
            if item == model.items.last {
                Task { await model.loadMore() }
            }
        }
}
```

## 6. Foundation Models (on-device AI — iOS 26)

The Foundation Models framework gives apps direct access to Apple's ~3B parameter on-device language model. All inference runs locally on the Neural Engine — free, private, and available offline.

### 6.1 Availability checking

Always check before using the model — not all devices support it.

```swift
import FoundationModels

let model = SystemLanguageModel.default

switch model.availability {
case .available:
    // Model ready — proceed
    break
case .unavailable(let reason):
    switch reason {
    case .appleIntelligenceNotEnabled:
        // User must enable Apple Intelligence in Settings
        break
    case .modelNotReady:
        // Model still downloading — show "try again later"
        break
    case .deviceNotEligible:
        // Hardware doesn't support it (pre-A17 Pro)
        break
    @unknown default:
        break
    }
}
```

In SwiftUI, gate features on availability:
```swift
struct AIFeatureView: View {
    private let model = SystemLanguageModel.default

    var body: some View {
        switch model.availability {
        case .available:
            SmartSummaryView()
        case .unavailable(.deviceNotEligible):
            ContentUnavailableView("Not Supported",
                systemImage: "cpu",
                description: Text("Requires iPhone 15 Pro or later."))
        default:
            ContentUnavailableView("AI Unavailable",
                systemImage: "sparkles",
                description: Text("Enable Apple Intelligence in Settings."))
        }
    }
}
```

### 6.2 Session creation and instructions

```swift
// Minimal session
let session = LanguageModelSession()

// Session with system instructions (developer-defined, separate from user prompts)
let session = LanguageModelSession {
    """
    You are a helpful recipe assistant. Provide concise answers.
    Always include preparation time and difficulty level.
    """
}

// Session with specific model, tools, and guardrails
let session = LanguageModelSession(
    model: SystemLanguageModel(useCase: .contentTagging), // specialized model
    guardrails: .default,  // safety filters (cannot be disabled)
    tools: [myTool1, myTool2],
    instructions: {
        "You are a content classifier. Tag items by category."
    }
)

// Resume a previous conversation
let session = LanguageModelSession(transcript: previousTranscript)
```

**Instructions vs prompts**: Instructions are developer-defined directives sent before user prompts. They help guard against prompt injection by separating system behavior from user input. Write instructions in English for best results.

### 6.3 Basic text generation

```swift
let session = LanguageModelSession()
let response = try await session.respond(to: "Summarize this text: \(inputText)")
print(response.content)  // String output
```

### 6.4 Generation options

```swift
let options = GenerationOptions(
    temperature: 0.8,              // 0.0–2.0: lower = deterministic, higher = creative
    maximumResponseTokens: 200,    // limit output length
    sampling: .greedy              // always pick most likely token
)

// Or with top-p / top-k sampling
let options = GenerationOptions(
    sampling: .random(probabilityThreshold: 0.9, seed: 42) // top-p with reproducible seed
    // .random(top: 40, seed: 42)  // top-k alternative
)

let response = try await session.respond(to: "Write a haiku", options: options)
```

### 6.5 Prewarming for faster first response

```swift
// Eagerly load model resources before user interaction
try await session.prewarm(promptPrefix: "Summarize the following:")

// Good practice: prewarm in .task {} when the view appears
.task {
    try? await session.prewarm(promptPrefix: "")
}
```

### 6.6 Guided generation with @Generable

The `@Generable` macro creates type-safe structured output — no JSON parsing needed. The compiler generates a JSON schema automatically.

```swift
@Generable
struct RecipeSuggestion {
    @Guide(description: "The name of the recipe")
    let name: String

    @Guide(description: "List of ingredients with quantities")
    let ingredients: [String]

    @Guide(description: "Preparation time in minutes", .range(5...120))
    let prepTimeMinutes: Int

    @Guide(.anyOf(["Easy", "Medium", "Hard"]))
    let difficulty: String
}

let session = LanguageModelSession()
let response = try await session.respond(
    to: "Suggest a quick dinner recipe with chicken and rice",
    generating: RecipeSuggestion.self
)
let recipe = response.content  // RecipeSuggestion — fully typed
print(recipe.name, recipe.prepTimeMinutes, recipe.difficulty)
```

### @Guide constraints

| Constraint | Example | Purpose |
|-----------|---------|---------|
| `description:` | `@Guide(description: "The user's age")` | Natural language hint for the model |
| `.anyOf([...])` | `@Guide(.anyOf(["PG", "PG-13", "R"]))` | Restrict to enumerated values |
| `.range(...)` | `@Guide(.range(1...100))` | Constrain numeric range |
| `.count(n)` | `@Guide(.count(5))` | Fix array length |

### Nested @Generable types

```swift
@Generable
struct MovieReview {
    let title: String
    let rating: Int
    let pros: [String]
    let cons: [String]
    @Guide(description: "Put summary last — it depends on pros and cons")
    let summary: String  // ordering tip: dependent properties last
}

@Generable
enum Sentiment: String, Codable {
    case positive, neutral, negative
}
```

**Performance tip**: Only include properties you actually need — the model generates all properties regardless. Put dependent properties (like summaries) last.

### 6.7 Streaming with PartiallyGenerated snapshots

When streaming structured output, each element is a `PartiallyGenerated` snapshot where all properties are optional and fill incrementally.

```swift
@Generable
struct TripPlan {
    let destination: String
    let activities: [String]
    let estimatedBudget: Int
}

let stream = session.streamResponse(
    to: "Plan a weekend trip to Tokyo",
    generating: TripPlan.self
)

for try await partial in stream {
    // partial is TripPlan.PartiallyGenerated — all properties optional
    if let destination = partial.destination {
        destinationLabel = destination  // animate UI updates
    }
    if let activities = partial.activities {
        activityList = activities
    }
}
```

Streaming plain text:
```swift
let stream = session.streamResponse(to: "Write a short story about a cat")
for try await partial in stream {
    text = partial.content  // updated snapshot, not delta
}
```

### 6.8 Tool calling

Tools let the model call back into your app when it needs external information.

```swift
struct SearchTool: Tool {
    let name = "searchProducts"
    let description = "Search the product catalog by query"

    @Generable
    struct Arguments {
        @Guide(description: "Search query string")
        let query: String
        @Guide(description: "Max results to return", .range(1...10))
        let limit: Int
    }

    func call(arguments: Arguments) async throws -> ToolOutput {
        let results = try await catalog.search(arguments.query, limit: arguments.limit)
        let formatted = results.map { "- \($0.name): $\($0.price)" }.joined(separator: "\n")
        return ToolOutput(formatted)
    }
}

// Register tools with session — model decides when to invoke them
let session = LanguageModelSession(
    tools: [SearchTool()],
    instructions: {
        "You are a shopping assistant. Use searchProducts to find items."
    }
)
let response = try await session.respond(to: "Find me running shoes under $100")
```

**Tool requirements**: Conform to `Tool` protocol. `name` must be identifier-safe (no spaces). `Arguments` must be `@Generable`. Return `ToolOutput` with string or `@Generable` content.

### 6.9 Transcript management

Sessions are stateful — all prompts and responses are recorded.

```swift
let session = LanguageModelSession()
_ = try await session.respond(to: "My name is Alex")
_ = try await session.respond(to: "What's my name?")
// Model remembers: "Alex"

// Access conversation history
let transcript = session.transcript
for entry in transcript.entries {
    // Inspect user messages, assistant responses, tool calls
}

// Save and restore conversations
let savedTranscript = session.transcript
// ... later ...
let resumedSession = LanguageModelSession(transcript: savedTranscript)
```

### 6.10 Key constraints and best practices

| Constraint | Detail |
|-----------|--------|
| **Model size** | ~3B parameters — optimized for utility tasks (summarization, extraction, classification), not general chat |
| **Hardware** | Apple Silicon: A17 Pro, M-series, or later |
| **Token limit** | Combined input + output: **4096 tokens** — keep prompts focused |
| **Latency** | ~0.6ms/token on iPhone 15 Pro |
| **Privacy** | 100% on-device — no data leaves the phone, works offline |
| **Guardrails** | `.default` only — cannot be disabled; model refuses harmful content |
| **Language** | Best results in English; instructions should be in English |
| **Cost** | Free — no API keys, no server, no usage limits |

### When to use Foundation Models vs other frameworks

| Task | Framework |
|------|-----------|
| Text summarization, extraction, classification | **Foundation Models** |
| Structured data generation from natural language | **Foundation Models** (@Generable) |
| Image classification, object detection | **CoreML** |
| OCR, face detection, body pose | **Vision** |
| Tokenization, NER, sentiment scoring | **NaturalLanguage** |
| Speech-to-text transcription | **SpeechAnalyzer** (Speech framework) |
| Language translation | **Translation** framework |
| Image generation from text/concepts | **Image Playground** (ImageCreator) |
| Text rewriting, proofreading | **Writing Tools** (system-provided) |

## 7. App lifecycle and scenes

```swift
@main
struct MyApp: App {
    @State private var appState = AppState()
    
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(appState)
        }
        .modelContainer(for: [Task.self])
        .commands {
            CommandGroup(after: .newItem) {
                Button("New Task") { appState.createTask() }
                    .keyboardShortcut("n", modifiers: .command)
            }
        }
        
        // Additional windows (iPad/Mac)
        WindowGroup("Player", id: "player", for: Track.ID.self) { $trackID in
            PlayerView(trackID: trackID)
        }
    }
}
```

### Scene phases
```swift
@Environment(\.scenePhase) var scenePhase

.onChange(of: scenePhase) { _, newPhase in
    switch newPhase {
    case .active:     // App is in foreground and interactive
        break
    case .inactive:   // App is visible but not interactive (e.g., switcher)
        break
    case .background: // App is in background — save state, stop timers
        saveState()
    @unknown default:
        break
    }
}
```

### Deep linking
```swift
.onOpenURL { url in
    guard let route = Route(from: url) else { return }
    navigationModel.navigate(to: route)
}
```

## 8. App Intents, Siri, and Apple Intelligence integration

App Intents is the single framework for exposing your app's actions and content to Siri, Spotlight, Shortcuts, Visual Intelligence, and the system-wide Apple Intelligence layer.

### 8.1 Basic App Intent

```swift
import AppIntents

struct CreateTaskIntent: AppIntent {
    static var title: LocalizedStringResource = "Create Task"
    static var description = IntentDescription("Create a new task")

    @Parameter(title: "Title")
    var title: String

    @Parameter(title: "Priority")
    var priority: TaskPriority?

    func perform() async throws -> some IntentResult & ProvidesDialog {
        let task = Task(title: title, priority: priority ?? .medium)
        try await taskStore.save(task)
        return .result(dialog: "Created task: \(title)")
    }
}
```

### 8.2 Siri + Apple Intelligence with Assistant Schemas (iOS 26)

Assistant Schemas let Apple Intelligence's LLM understand your app's domain. The compiler enforces that all referenced entities are properly exposed.

```swift
// Mark intents with @AssistantIntent for Siri/Apple Intelligence
@AssistantIntent(schema: .photos.createAlbum)
struct CreateAlbumIntent: AppIntent {
    @Parameter(title: "Album Name")
    var name: String

    func perform() async throws -> some IntentResult {
        let album = try await photoLibrary.createAlbum(named: name)
        return .result()
    }
}

// Expose domain entities with @AssistantEntity
@AssistantEntity(schema: .photos.asset)
struct PhotoEntity: AppEntity {
    static var defaultQuery = PhotoEntityQuery()
    var id: String
    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(id)")
    }
    static var typeDisplayRepresentation = TypeDisplayRepresentation(name: "Photo")
}

// Expose enums with @AssistantEnum
@AssistantEnum(schema: .photos.assetType)
enum MediaType: String, AppEnum {
    case photo, video
    static var typeDisplayRepresentation = TypeDisplayRepresentation(name: "Media Type")
    static var caseDisplayRepresentations: [MediaType: DisplayRepresentation] = [
        .photo: "Photo",
        .video: "Video"
    ]
}
```

**Key schemas**: `.system.search`, `.photos.createAlbum`, `.photos.asset`, `.mail.send`, `.messages.send`, and 100+ predefined actions across 12 domains.

### 8.3 Visual Intelligence integration (iOS 26)

Users can point their camera or screenshot at content and your app can provide contextual results.

```swift
// Implement a query that accepts SemanticContentDescriptor
struct VisualSearchQuery: EntityQuery {
    func entities(matching descriptor: SemanticContentDescriptor) async throws -> [ProductEntity] {
        // descriptor contains the image/visual context from the camera
        let results = try await catalog.visualSearch(descriptor)
        return results.map { ProductEntity(product: $0) }
    }
}

// The system routes visual intelligence results to your app
// Users see your app's results alongside Google Search and other apps
```

### 8.4 Shortcuts provider

```swift
struct MyShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(
            intent: CreateTaskIntent(),
            phrases: [
                "Create a task in \(.applicationName)",
                "Add a new task to \(.applicationName)"
            ],
            shortTitle: "New Task",
            systemImageName: "plus"
        )
    }
}
```

### 8.5 "Use Model" Shortcut action

Users can chain your intents with on-device Foundation Models or Private Cloud Compute in Shortcuts — no extra code needed. Your App Intents automatically become composable with AI actions.

### Widgets (WidgetKit)
```swift
struct TaskWidget: Widget {
    var body: some WidgetConfiguration {
        StaticConfiguration(kind: "tasks", provider: TaskProvider()) { entry in
            TaskWidgetView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("Tasks")
        .supportedFamilies([.systemSmall, .systemMedium, .accessoryCircular])
    }
}
```

### Live Activities
```swift
struct DeliveryActivity: ActivityAttributes {
    struct ContentState: Codable, Hashable {
        var status: String
        var estimatedArrival: Date
    }
    var orderID: String
}
```

## 9. UIKit interop from SwiftUI

### UIViewRepresentable
```swift
struct MapView: UIViewRepresentable {
    @Binding var region: MKCoordinateRegion
    
    func makeUIView(context: Context) -> MKMapView {
        let map = MKMapView()
        map.delegate = context.coordinator
        return map
    }
    
    func updateUIView(_ map: MKMapView, context: Context) {
        map.setRegion(region, animated: true)
    }
    
    func makeCoordinator() -> Coordinator { Coordinator(self) }
    
    class Coordinator: NSObject, MKMapViewDelegate {
        let parent: MapView
        init(_ parent: MapView) { self.parent = parent }
    }
}
```

### UIViewControllerRepresentable
```swift
struct CameraView: UIViewControllerRepresentable {
    @Binding var image: UIImage?
    
    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }
    
    func updateUIViewController(_ vc: UIImagePickerController, context: Context) {}
    func makeCoordinator() -> Coordinator { Coordinator(self) }
    
    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: CameraView
        init(_ parent: CameraView) { self.parent = parent }
        
        func imagePickerController(_ picker: UIImagePickerController,
                                   didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]) {
            parent.image = info[.originalImage] as? UIImage
            picker.dismiss(animated: true)
        }
    }
}
```

### Hosting SwiftUI in UIKit
```swift
let hostingController = UIHostingController(rootView: SwiftUIView())
addChild(hostingController)
view.addSubview(hostingController.view)
hostingController.didMove(toParent: self)
```

## 10. Error handling patterns

```swift
enum AppError: LocalizedError {
    case networkUnavailable
    case unauthorized
    case serverError(Int)
    case decodingFailed
    case notFound
    
    var errorDescription: String? {
        switch self {
        case .networkUnavailable: return String(localized: "No internet connection")
        case .unauthorized: return String(localized: "Please sign in again")
        case .serverError(let code): return String(localized: "Server error (\(code))")
        case .decodingFailed: return String(localized: "Unexpected response format")
        case .notFound: return String(localized: "Content not found")
        }
    }
    
    var recoverySuggestion: String? {
        switch self {
        case .networkUnavailable: return String(localized: "Check your connection and try again")
        case .unauthorized: return String(localized: "Tap to sign in")
        default: return String(localized: "Try again later")
        }
    }
}

// In views
struct ContentView: View {
    @State private var error: AppError?
    
    var body: some View {
        content
            .alert(error: $error) // Custom alert modifier
    }
}
```

## 11. Testing strategies

### Swift Testing (iOS 26 / Xcode 26)
```swift
import Testing
@testable import MyApp

@Suite("Task Store")
struct TaskStoreTests {
    @Test("Adding a task increases count")
    func addTask() async {
        let store = TaskStore()
        await store.addTask(Task(title: "Test"))
        #expect(store.tasks.count == 1)
    }
    
    @Test("Completing task updates state")
    func completeTask() {
        let task = Task(title: "Test")
        task.isCompleted = true
        #expect(task.isCompleted)
    }
    
    @Test("Load feed with network error", arguments: [
        AppError.networkUnavailable,
        AppError.serverError(500)
    ])
    func loadFeedError(error: AppError) async {
        let store = TaskStore(api: MockAPI(error: error))
        await store.loadFeed()
        #expect(store.error != nil)
    }
}
```

### UI Testing
```swift
import XCTest

final class TaskFlowUITests: XCTestCase {
    let app = XCUIApplication()
    
    override func setUp() {
        continueAfterFailure = false
        app.launchArguments = ["--uitesting"]
        app.launch()
    }
    
    func testCreateTask() {
        app.buttons["Add Task"].tap()
        let textField = app.textFields["Task title"]
        textField.tap()
        textField.typeText("Buy groceries")
        app.buttons["Save"].tap()
        XCTAssertTrue(app.staticTexts["Buy groceries"].exists)
    }
}
```

### Snapshot testing, accessibility audits
Use XCTest's `performAccessibilityAudit()` (iOS 17+) to automatically detect common accessibility issues in UI tests.

## 12. Xcode 26 tooling

- **Predictive code completion** powered by on-device AI
- **#Preview** macros for inline SwiftUI previews with state
- **Swift Testing** framework as default (replaces XCTest for unit tests)
- **Instruments** with SwiftUI profiling template
- **Memory Graph Debugger** for retain cycle detection
- **View Hierarchy Debugger** shows Liquid Glass layers
- **String Catalog Editor** for localization management
- **Icon Composer** for multi-layer Liquid Glass app icons
- **Reality Composer Pro** for spatial/AR content

## 13. Performance profiling

### Key Instruments templates
- **SwiftUI** — View body evaluations, state invalidations, rendering time
- **Time Profiler** — CPU hotspots, main thread work
- **Core Animation** — Frame rate, offscreen rendering, blending
- **Allocations** — Memory usage, leaked objects
- **Network** — Request timing, payload sizes
- **Energy Log** — Battery impact per subsystem

### SwiftUI-specific profiling
```swift
// Debug view re-evaluations
let _ = Self._printChanges()  // In view body during debug

// Check render counts in previews
#Preview {
    ContentView()
        .environment(\.debugRenderCounts, true)
}
```

## 14. Security and privacy

### Keychain for credentials
```swift
import Security

func saveToKeychain(key: String, data: Data) throws {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: key,
        kSecValueData as String: data
    ]
    SecItemDelete(query as CFDictionary)
    let status = SecItemAdd(query as CFDictionary, nil)
    guard status == errSecSuccess else { throw KeychainError.saveFailed }
}
```

### App Transport Security
HTTPS is required by default. Any HTTP exceptions must be declared in Info.plist and justified during App Review.

### Privacy manifests (required — App Store rejection if missing)

Every app must include a `PrivacyInfo.xcprivacy` file (File > New > Privacy Manifest). Missing or incorrect entries are a common App Store rejection reason.

#### Required reason APIs

If your code (or any third-party SDK you use) accesses these APIs, you must declare a reason:

| API category | `NSPrivacyAccessedAPIType` value | Common reason codes |
|---|---|---|
| **UserDefaults** | `NSPrivacyAccessedAPITypeUserDefaults` | `CA92.1` — own app group; `1C8F.1` — user-initiated action; `C56D.1` — 3rd-party SDK crash reporting |
| **File timestamps** | `NSPrivacyAccessedAPITypeFileTimestamp` | `C617.1` — files inside app container; `3B52.1` — user-provided files; `0A2A.1` — 3rd-party SDK crash reporting |
| **System boot time** | `NSPrivacyAccessedAPITypeSystemBootTime` | `35F9.1` — measure elapsed time; `8FFB.1` — calculate absolute event timestamps; `3D61.1` — 3rd-party SDK crash reporting |
| **Disk space** | `NSPrivacyAccessedAPITypeDiskSpace` | `E174.1` — show available disk space; `85F4.1` — check space before writing app data |
| **Active keyboards** | `NSPrivacyAccessedAPITypeActiveKeyboards` | `3EC4.1` — custom keyboard app only; `54BD.1` — 3rd-party SDK |

**PrivacyInfo.xcprivacy structure (plist XML):**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" ...>
<plist version="1.0">
<dict>
    <!-- Required reason APIs -->
    <key>NSPrivacyAccessedAPITypes</key>
    <array>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPITypeUserDefaults</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>CA92.1</string>
            </array>
        </dict>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPITypeFileTimestamp</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>C617.1</string>
            </array>
        </dict>
    </array>

    <!-- Data collected -->
    <key>NSPrivacyCollectedDataTypes</key>
    <array>
        <!-- Add entries for each data type collected (email, usage data, etc.) -->
    </array>

    <!-- Tracking -->
    <key>NSPrivacyTracking</key>
    <false/>
    <key>NSPrivacyTrackingDomains</key>
    <array>
        <!-- Add tracking domains if NSPrivacyTracking is true -->
    </array>
</dict>
</plist>
```

**Third-party SDKs**: If an SDK you import accesses any required-reason API, you must declare the reasons even if your own code doesn't call them. Check each SDK's privacy manifest or documentation. Analytics SDKs (Firebase, Amplitude) almost always trigger `NSPrivacyAccessedAPITypeUserDefaults` and `NSPrivacyAccessedAPITypeSystemBootTime`.

### Biometric auth
```swift
import LocalAuthentication

let context = LAContext()
var error: NSError?
if context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) {
    try await context.evaluatePolicy(
        .deviceOwnerAuthenticationWithBiometrics,
        localizedReason: "Access your account"
    )
}
```

## 15. Distribution and deployment

### Minimum deployment targets (recommendations)
- **New apps**: iOS 26 (take full advantage of Liquid Glass, Foundation Models)
- **Existing apps**: iOS 17+ (minimum for @Observable, NavigationStack maturity)
- **Maximum reach**: iOS 16+ (but lose @Observable, many SwiftUI improvements)

### TestFlight
- Internal testing: up to 100 testers, no review needed
- External testing: up to 10,000 testers, requires beta review
- Automatic updates, crash reports, feedback screenshots

### App Store optimization for iOS 26
- Create Liquid Glass app icon using Icon Composer
- Declare Accessibility Nutrition Labels in App Store Connect
- Update screenshots showing iOS 26 design language
- Add Privacy Nutrition Labels for all data practices

## 16. Writing Tools API (Apple Intelligence)

Writing Tools provide system-wide rewrite, proofread, and summarize capabilities. Standard UIKit/SwiftUI text views get Writing Tools automatically. Custom text engines need explicit adoption.

### Automatic support (no code needed)

Apps using `TextField`, `TextEditor`, `UITextView`, or `UITextField` get Writing Tools for free — the system adds toolbar buttons and context menu items.

### Customizing behavior

```swift
// SwiftUI: control Writing Tools behavior on text views
TextEditor(text: $content)
    .writingToolsBehavior(.complete)  // .complete (default), .limited, .none

// .complete — full rewrite, proofread, summarize, follow-up requests
// .limited — only proofreading, no rewriting
// .none — disable Writing Tools for this view (e.g., code editors)
```

### UIKit custom text views

```swift
// For custom text engines (not UITextView), adopt UIWritingToolsCoordinating
class CustomTextView: UIView, UIWritingToolsCoordinating {
    var writingToolsCoordinator: UIWritingToolsCoordinator?

    func setupWritingTools() {
        writingToolsCoordinator = UIWritingToolsCoordinator(delegate: self)
    }
}

extension CustomTextView: UIWritingToolsCoordinatorDelegate {
    func writingToolsCoordinator(_ coordinator: UIWritingToolsCoordinator,
                                  replace range: NSRange,
                                  with replacement: NSAttributedString) {
        // Apply the AI-generated replacement to your text storage
        textStorage.replaceCharacters(in: range, with: replacement)
    }
}
```

### iOS 26 enhancements
- **Follow-up requests**: After an initial rewrite, users can say "make it warmer" or "more professional"
- **Shortcuts integration**: Writing Tools available as Shortcuts actions (Proofread, Rewrite, Summarize) for automation
- **Presentation intents**: Rich text support with semantic paragraph styles

## 17. Translation framework (on-device)

The Translation framework provides free, on-device language translation powered by CoreML models shared with the system Translate app.

### Quick translation overlay

```swift
import Translation

struct ContentView: View {
    @State private var showTranslation = false
    let foreignText = "Bonjour, comment allez-vous?"

    var body: some View {
        Text(foreignText)
            .translationPresentation(isPresented: $showTranslation, text: foreignText)
            .onTapGesture { showTranslation = true }
    }
}
```

### Programmatic translation with TranslationSession

```swift
struct TranslatorView: View {
    @State private var translatedText = ""

    var body: some View {
        Text(translatedText)
            .translationTask(source: .init(identifier: "fr"),
                             target: .init(identifier: "en")) { session in
                let response = try await session.translate("Bonjour le monde")
                translatedText = response.targetText
            }
    }
}
```

### Batch translation

```swift
.translationTask(source: .french, target: .english) { session in
    let requests = texts.map { TranslationSession.Request(sourceText: $0) }
    let responses = try await session.translations(from: requests)
    translatedTexts = responses.map(\.targetText)
}
```

### Key constraints
- **SwiftUI only** — all Translation APIs require SwiftUI
- **Real device required** — does not work in Simulator
- **Auto-downloads models** — framework prompts user to download language pairs if not installed
- **On-device** — no internet required after model download
- **Free** — no API keys, no usage limits

## 18. Smart Reply API (Apple Intelligence)

Smart Reply generates contextual reply suggestions for messaging and email apps using on-device Apple Intelligence.

### UIKit integration

```swift
import UIKit

class ChatViewController: UIViewController {
    let textView = UITextView()

    override func viewDidLoad() {
        super.viewDidLoad()
        // Enable Smart Reply for this text view
        textView.smartReplyConfiguration = UISmartReplyConfiguration(
            context: conversationContext
        )
    }

    // Provide conversation context for better suggestions
    var conversationContext: UISmartReplyContext {
        let messages = conversation.messages.map { msg in
            UISmartReplyContext.Message(
                text: msg.text,
                isFromCurrentUser: msg.isFromMe,
                date: msg.timestamp
            )
        }
        return UISmartReplyContext(messages: messages)
    }
}
```

### SwiftUI integration

```swift
TextEditor(text: $replyText)
    .smartReplyContext(messages: conversation.messages.map { msg in
        .init(text: msg.text, isFromCurrentUser: msg.isFromMe, date: msg.date)
    })
// Smart Reply suggestions appear above the keyboard automatically
```

### How it works
- Analyzes conversation history to generate 2–3 contextual reply suggestions
- Suggestions appear as tappable chips above the keyboard
- User taps a suggestion → text is inserted into the compose field
- All processing on-device — conversation content never leaves the phone
- Works best with English; expanding to more languages over time
