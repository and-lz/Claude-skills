# iOS 26 Component Catalog

## Table of contents
1. Navigation components
2. Buttons and controls
3. Lists and collections
4. Forms and input
5. Sheets, alerts, and overlays
6. Search
7. Charts and data visualization
8. Media and content
9. Loading and progress
10. Empty states
11. New iOS 26 components

---

## 1. Navigation components

### Tab bar (floating, Liquid Glass)
```swift
TabView {
    Tab("Home", systemImage: "house") {
        HomeView()
    }
    Tab("Search", systemImage: "magnifyingglass", role: .search) {
        SearchView()
    }
    Tab("Profile", systemImage: "person") {
        ProfileView()
    }
}
.tabBarMinimizeBehavior(.onScrollDown)  // Collapse on scroll
.tabViewBottomAccessory {               // Persistent glass accessory
    NowPlayingBar()
}
```

Rules:
- 3–5 tabs maximum (2 minimum)
- Tab bars navigate between sections — never trigger actions
- Use `role: .search` for search tabs — they transform into a search field when selected
- Don't show tab bar and toolbar together at the bottom
- Long-press a tab shows selection bubble for quick switching

### Tab bar → Sidebar adaptivity (iPad)
```swift
TabView {
    TabSection("Library") {
        Tab("Books", systemImage: "book") { BooksView() }
        Tab("Audiobooks", systemImage: "headphones") { AudiobooksView() }
    }
    Tab("Settings", systemImage: "gear") { SettingsView() }
}
.tabViewStyle(.sidebarAdaptable)  // Floating tab on iPhone, sidebar on iPad
```

### NavigationStack
```swift
NavigationStack(path: $path) {
    List(items) { item in
        NavigationLink(value: item) {
            ItemRow(item: item)
        }
    }
    .navigationTitle("Items")
    .navigationDestination(for: Item.self) { item in
        ItemDetail(item: item)
    }
}
```

Standard push transitions are always-interactive in iOS 26 — users can swipe back from anywhere in the content area (not just the leading edge), interact during transitions, and tap back multiple times.

### NavigationSplitView (iPad multi-column)
```swift
NavigationSplitView {
    // Sidebar (gets Liquid Glass styling automatically)
    List(categories, selection: $selectedCategory) { cat in
        Label(cat.name, systemImage: cat.icon)
    }
} content: {
    // Content column
    if let category = selectedCategory {
        ItemList(category: category, selection: $selectedItem)
    }
} detail: {
    // Detail column
    if let item = selectedItem {
        ItemDetail(item: item)
    } else {
        ContentUnavailableView("Select an Item",
            systemImage: "doc.text",
            description: Text("Choose from the list"))
    }
}
.navigationSplitViewColumnWidth(min: 200, ideal: 250, max: 300)
```

### Toolbar
```swift
.toolbar {
    ToolbarItem(placement: .primaryAction) {
        Button { } label: {
            Label("Add", systemImage: "plus")
        }
    }
    ToolbarItem(placement: .secondaryAction) {
        Menu {
            Button("Edit") { }
            Button("Share") { }
        } label: {
            Label("More", systemImage: "ellipsis.circle")
        }
    }
}
```

Bar button items sharing similar styling are automatically grouped into a shared glass background. In iOS 26, `UINavigationItem` supports `.subtitle`, `attributedTitle`, and `largeSubtitleView`.

## 2. Buttons and controls

### Button styles (complete hierarchy)
```swift
// Primary action — opaque glass, tinted
Button("Save") { }
    .buttonStyle(.glassProminent)    // NEW iOS 26

// Secondary action — translucent glass
Button("Cancel") { }
    .buttonStyle(.glass)             // NEW iOS 26

// Standard bordered (pre-glass, still valid)
Button("Option") { }
    .buttonStyle(.bordered)

// Prominent bordered (solid fill)
Button("Buy") { }
    .buttonStyle(.borderedProminent)

// Borderless (text only)
Button("Learn More") { }
    .buttonStyle(.borderless)

// Plain (no styling)
Button("Link") { }
    .buttonStyle(.plain)
```

### Control sizes
```swift
.controlSize(.mini)        // Compact inline controls
.controlSize(.small)       // Secondary buttons
.controlSize(.regular)     // Default
.controlSize(.large)       // Primary actions
.controlSize(.extraLarge)  // NEW iOS 26 — hero buttons
```

### Toggle
```swift
Toggle("Notifications", isOn: $enabled)
Toggle(isOn: $enabled) {
    Label("Airplane Mode", systemImage: "airplane")
}
```
Toggles receive Liquid Glass treatment during interaction in iOS 26.

### Slider (enhanced in iOS 26)
```swift
Slider(value: $volume, in: 0...100) {
    Text("Volume")
} minimumValueLabel: {
    Image(systemName: "speaker")
} maximumValueLabel: {
    Image(systemName: "speaker.wave.3")
}

// iOS 26 enhancements:
Slider(value: $temp, in: 60...80) {
    Text("Temperature")
} ticks: {
    // Custom tick marks
    ForEach(stride(from: 60, through: 80, by: 5), id: \.self) { val in
        SliderTick(value: Double(val))
    }
}
.sliderNeutralValue(72)  // Visual midpoint indicator
```

New slider features: custom tick marks, neutral values, thumbless style, momentum (flick to accelerate).

### Picker
```swift
// Wheel
Picker("Flavor", selection: $flavor) {
    ForEach(flavors) { Text($0.name).tag($0) }
}
.pickerStyle(.wheel)

// Segmented (gets glass thumb in iOS 26)
Picker("View", selection: $viewMode) {
    Text("Grid").tag(ViewMode.grid)
    Text("List").tag(ViewMode.list)
}
.pickerStyle(.segmented)

// Menu (default)
Picker("Sort", selection: $sort) { ... }

// Inline (in Form)
Picker("Country", selection: $country) { ... }
    .pickerStyle(.inline)
```

### DatePicker
```swift
DatePicker("Date", selection: $date, displayedComponents: [.date])
DatePicker("Time", selection: $time, displayedComponents: .hourAndMinute)
    .datePickerStyle(.wheel)     // or .graphical, .compact
```

### Stepper
```swift
Stepper("Quantity: \(qty)", value: $qty, in: 1...99)
Stepper {
    Text("Adults: \(adults)")
} onIncrement: {
    adults += 1
} onDecrement: {
    adults = max(0, adults - 1)
}
```

### Menu
```swift
Menu("Options") {
    Button("Copy", action: copy)
    Button("Paste", action: paste)
    Divider()
    Menu("Share") {
        Button("Messages", action: shareMessages)
        Button("Mail", action: shareMail)
    }
    Button("Delete", role: .destructive, action: delete)
}
```

### Context menu
```swift
Text("Hold me")
    .contextMenu {
        Button("Copy") { }
        Button("Share") { }
        Button("Delete", role: .destructive) { }
    } preview: {
        ItemPreview(item: item)  // Rich preview
    }
```

## 3. Lists and collections

### List styles
```swift
List { ... }
    .listStyle(.plain)          // No background
    .listStyle(.grouped)        // Grouped sections with background
    .listStyle(.insetGrouped)   // Rounded grouped (most common)
    .listStyle(.sidebar)        // Sidebar navigation (iPad)
    .listStyle(.inset)          // Inset without grouping
```

### Section headers (iOS 26: sentence case, not ALL CAPS)
```swift
List {
    Section("Recent items") {    // Sentence case in iOS 26
        ForEach(recentItems) { item in
            ItemRow(item: item)
        }
    }
    Section("All items") {
        ForEach(allItems) { item in
            ItemRow(item: item)
        }
    }
}
```

### Swipe actions
```swift
ForEach(items) { item in
    ItemRow(item: item)
        .swipeActions(edge: .trailing) {
            Button("Delete", role: .destructive) { delete(item) }
            Button("Archive") { archive(item) }
                .tint(.blue)
        }
        .swipeActions(edge: .leading) {
            Button("Pin") { pin(item) }
                .tint(.yellow)
        }
}
```

### Lazy grids
```swift
let columns = [
    GridItem(.adaptive(minimum: 160, maximum: 200), spacing: 16)
]

ScrollView {
    LazyVGrid(columns: columns, spacing: 16) {
        ForEach(items) { item in
            CardView(item: item)
        }
    }
    .padding(.horizontal)
}
```

Grid item types:
- `.fixed(width)` — exact size
- `.flexible(minimum:maximum:)` — range
- `.adaptive(minimum:maximum:)` — as many as fit

### Section index (new iOS 26)
```swift
List {
    ForEach(groupedContacts) { group in
        Section(group.letter) {
            ForEach(group.contacts) { contact in
                ContactRow(contact: contact)
            }
            .sectionIndexLabel(group.letter)
        }
    }
}
```

## 4. Forms and input

### Form structure
```swift
Form {
    Section("Account") {
        TextField("Name", text: $name)
            .textContentType(.name)
        TextField("Email", text: $email)
            .textContentType(.emailAddress)
            .keyboardType(.emailAddress)
            .textInputAutocapitalization(.never)
    }

    Section("Security") {
        SecureField("Password", text: $password)
            .textContentType(.newPassword)
    }

    Section("Preferences") {
        Toggle("Notifications", isOn: $notifications)
        Picker("Theme", selection: $theme) {
            Text("System").tag(Theme.system)
            Text("Light").tag(Theme.light)
            Text("Dark").tag(Theme.dark)
        }
        DatePicker("Birthday", selection: $birthday, displayedComponents: .date)
    }

    Section {
        Button("Save") { save() }
            .buttonStyle(.glassProminent)
    }
}
```

### Focus management for forms
```swift
enum Field: Hashable {
    case name, email, password
}

@FocusState private var focused: Field?

TextField("Name", text: $name)
    .focused($focused, equals: .name)
    .submitLabel(.next)
    .onSubmit { focused = .email }

TextField("Email", text: $email)
    .focused($focused, equals: .email)
    .submitLabel(.next)
    .onSubmit { focused = .password }

SecureField("Password", text: $password)
    .focused($focused, equals: .password)
    .submitLabel(.done)
    .onSubmit { save() }
```

### Keyboard types
`.default`, `.asciiCapable`, `.numbersAndPunctuation`, `.URL`, `.numberPad`, `.phonePad`, `.namePhonePad`, `.emailAddress`, `.decimalPad`, `.twitter`, `.webSearch`, `.asciiCapableNumberPad`

### Text content types for auto-fill
`.username`, `.password`, `.newPassword`, `.emailAddress`, `.oneTimeCode`, `.creditCardNumber`, `.telephoneNumber`, `.fullStreetAddress`, `.postalCode`, `.name`, `.givenName`, `.familyName`

## 5. Sheets, alerts, and overlays

### Sheets
```swift
.sheet(isPresented: $showSheet) {
    SheetContent()
        .presentationDetents([.medium, .large])
        .presentationDragIndicator(.visible)
}

// Zoom transition from source (iOS 26)
@Namespace private var ns
Button("Open") { showSheet = true }
    .matchedTransitionSource(id: "item", in: ns)
.sheet(isPresented: $showSheet) {
    DetailView()
        .navigationTransition(.zoom(sourceID: "item", in: ns))
}
```

Sheets in iOS 26 feature Liquid Glass backgrounds — remove custom `.presentationBackground()` to let them render.

### Alerts (left-aligned text in iOS 26)
```swift
.alert("Delete Item?", isPresented: $showAlert) {
    Button("Cancel", role: .cancel) { }
    Button("Delete", role: .destructive) { delete() }
} message: {
    Text("This action cannot be undone.")
}
```

### Action sheets (anchor to source in iOS 26)
```swift
.confirmationDialog("Options", isPresented: $showOptions) {
    Button("Share") { }
    Button("Duplicate") { }
    Button("Delete", role: .destructive) { }
}
```

In iOS 26, action sheets anchor to their source view on ALL devices (not just iPad). Set `sourceItem` or `sourceView` on `popoverPresentationController` in UIKit.

### Popovers
```swift
.popover(isPresented: $showPopover) {
    PopoverContent()
        .frame(minWidth: 300, minHeight: 200)
}
```

## 6. Search

### Search bar (bottom-positioned in iOS 26 tab apps)
```swift
NavigationStack {
    List(filteredItems) { item in
        ItemRow(item: item)
    }
    .searchable(text: $searchText, prompt: "Search items")
    .searchSuggestions {
        ForEach(suggestions) { suggestion in
            Text(suggestion.name)
                .searchCompletion(suggestion.name)
        }
    }
}

// Dedicated search tab (iOS 26)
Tab("Search", systemImage: "magnifyingglass", role: .search) {
    SearchView()
}
```

### Search scopes
```swift
.searchScopes($scope) {
    Text("All").tag(SearchScope.all)
    Text("Recent").tag(SearchScope.recent)
    Text("Favorites").tag(SearchScope.favorites)
}
```

## 7. Charts and data visualization

### Swift Charts (2D)
```swift
Chart(salesData) { item in
    BarMark(
        x: .value("Month", item.month),
        y: .value("Sales", item.amount)
    )
    .foregroundStyle(by: .value("Category", item.category))
}
.chartXAxis { AxisMarks(position: .bottom) }
.chartYAxis { AxisMarks(position: .leading) }
```

Available marks: `BarMark`, `LineMark`, `PointMark`, `AreaMark`, `RuleMark`, `RectangleMark`, `SectorMark` (pie/donut), `LinePlot` (function-based).

### 3D Charts (new iOS 26)
```swift
Chart3D(data) { item in
    PointMark3D(
        x: .value("X", item.x),
        y: .value("Y", item.y),
        z: .value("Z", item.z)
    )
}
.chart3DPose($cameraPose)  // Interactive camera controls

// Surface plots
Chart3D {
    SurfacePlot(data, x: "lat", y: "elevation", z: "lon")
}
```

## 8. Media and content

### AsyncImage
```swift
AsyncImage(url: item.imageURL) { phase in
    switch phase {
    case .empty:
        ProgressView()
    case .success(let image):
        image.resizable().aspectRatio(contentMode: .fill)
    case .failure:
        Image(systemName: "photo").foregroundStyle(.secondary)
    @unknown default:
        EmptyView()
    }
}
.frame(width: 200, height: 200)
.clipShape(RoundedRectangle(cornerRadius: 12))
```

### WebView (new iOS 26)
```swift
WebView(url: URL(string: "https://example.com")!)

// With WebPage for state management
@State private var page = WebPage()
WebView(page)
    .onAppear { page.load(URLRequest(url: myURL)) }
```

### Rich TextEditor (new iOS 26)
```swift
@State private var text = AttributedString("Hello")
TextEditor(text: $text)
    // Supports bold, italic, underline, links
```

## 9. Loading and progress

### ProgressView
```swift
ProgressView()                                    // Indeterminate spinner
ProgressView("Loading...")                         // With label
ProgressView(value: 0.7)                          // Determinate bar
ProgressView(value: 0.7, total: 1.0) {
    Text("Downloading")
} currentValueLabel: {
    Text("70%")
}
```

### ContentUnavailableView (empty states)
```swift
ContentUnavailableView(
    "No Results",
    systemImage: "magnifyingglass",
    description: Text("Try a different search term")
)

ContentUnavailableView.search  // Built-in search empty state
```

## 10. New iOS 26 components summary

| Component | SwiftUI API | Purpose |
|-----------|------------|---------|
| Glass buttons | `.buttonStyle(.glass)` / `.glassProminent` | Primary/secondary actions with glass |
| WebView | `WebView(url:)` + `WebPage` | Native web content in SwiftUI |
| Rich TextEditor | `TextEditor` with `AttributedString` | Formatted text editing |
| Chart3D | `Chart3D`, `SurfacePlot`, `PointMark3D` | 3D data visualization |
| ConcentricRectangle | `ConcentricRectangle()` | Device-matching corner radii |
| Background Extension | `.backgroundExtensionEffect()` | Immersive blur behind sidebars |
| Extra Large controls | `.controlSize(.extraLarge)` | Hero-sized buttons |
| Scroll edge effects | `.scrollEdgeEffectStyle(.soft)` | Subtle blur at scroll boundaries |
| Section index labels | `.sectionIndexLabel()` | Alphabetical section scrubber |
| Slider ticks | `ticks: { }` closure | Custom tick marks on sliders |
| Slider neutral value | `.sliderNeutralValue(72)` | Visual midpoint indicator |
