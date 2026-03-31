# iOS 26 Common UI Patterns Catalog

80+ common UI patterns seen in real-world iOS apps, with SwiftUI implementation guidance. Each pattern includes what it is, when to use it, and the SwiftUI components involved.

## Table of contents

1. [Navigation patterns](#1-navigation-patterns)
2. [Onboarding patterns](#2-onboarding-patterns)
3. [Authentication patterns](#3-authentication-patterns)
4. [List & feed patterns](#4-list--feed-patterns)
5. [Loading & state patterns](#5-loading--state-patterns)
6. [Search & filter patterns](#6-search--filter-patterns)
7. [Interaction patterns](#7-interaction-patterns)
8. [Overlay & alert patterns](#8-overlay--alert-patterns)
9. [Component patterns](#9-component-patterns)
10. [Settings & preferences](#10-settings--preferences)
11. [Dashboard & analytics](#11-dashboard--analytics)
12. [E-commerce patterns](#12-e-commerce-patterns)
13. [Social media patterns](#13-social-media-patterns)
14. [Messaging patterns](#14-messaging-patterns)
15. [Media patterns](#15-media-patterns)
16. [Form & input patterns](#16-form--input-patterns)
17. [iOS 26 Liquid Glass patterns](#17-ios-26-liquid-glass-patterns)
18. [Top 50 summary table](#18-top-50-summary-table)

---

## 1. Navigation patterns

### 1.1 Tab-based navigation

Bottom tab bar with 2-5 root destinations. Most common app structure.

```swift
TabView {
    Tab("Home", systemImage: "house") { HomeView() }
    Tab("Search", systemImage: "magnifyingglass") { SearchView() }
    Tab("Profile", systemImage: "person") { ProfileView() }
}
.tabBarMinimizeBehavior(.onScrollDown) // iOS 26: auto-hide on scroll
```

### 1.2 Hierarchical push navigation

Drill-down stack: list -> detail -> sub-detail.

```swift
NavigationStack {
    List(items) { item in
        NavigationLink(value: item) {
            ItemRow(item: item)
        }
    }
    .navigationDestination(for: Item.self) { item in
        ItemDetailView(item: item)
    }
    .navigationTitle("Items")
}
```

### 1.3 Modal / sheet presentation

Slides up for creating content, confirmations, or auxiliary tasks.

```swift
.sheet(isPresented: $showCreate) {
    CreateItemView()
        .presentationDetents([.medium, .large])
        .presentationDragIndicator(.visible)
}
```

### 1.4 Sidebar / split view (iPad)

Two or three columns for large screens. Adapts from tabs on iPhone.

```swift
NavigationSplitView {
    SidebarView(selection: $selected)
} content: {
    ContentListView(category: selected)
} detail: {
    DetailView(item: selectedItem)
}
```

### 1.5 Deep linking

Open specific screens from URLs, notifications, or widgets.

```swift
.onOpenURL { url in
    if let route = DeepLink.from(url) {
        switch route {
        case .product(let id): navPath.append(ProductRoute(id: id))
        case .profile(let name): navPath.append(ProfileRoute(name: name))
        }
    }
}
```

---

## 2. Onboarding patterns

### 2.1 Feature carousel

Horizontal paged walkthrough with 3-5 feature highlights.

```swift
TabView {
    OnboardingPage(image: "sparkles", title: "Welcome", subtitle: "Discover amazing features")
    OnboardingPage(image: "photo", title: "Photos", subtitle: "Organize your library")
    OnboardingPage(image: "checkmark", title: "Ready", subtitle: "Let's get started")
}
.tabViewStyle(.page)
.indexViewStyle(.page(backgroundDisplayMode: .always))
```

### 2.2 Permission request flow

Sequential screens explaining why each permission is needed before system prompt.

```swift
struct PermissionView: View {
    var body: some View {
        VStack(spacing: 24) {
            Image(systemName: "camera.fill").font(.system(size: 60))
            Text("Camera Access").font(.title2.bold())
            Text("We need camera access to scan documents.")
            Button("Allow Access") {
                AVCaptureDevice.requestAccess(for: .video) { _ in }
            }
            .buttonStyle(.borderedProminent)
            Button("Maybe Later") { skip() }
        }
    }
}
```

### 2.3 Progressive tooltips (TipKit)

Contextual hints shown inline as user discovers features.

```swift
struct FavoriteTip: Tip {
    var title: Text { Text("Save Favorites") }
    var message: Text? { Text("Tap the heart to save items you love.") }
    var image: Image? { Image(systemName: "heart") }
}

// In view:
Button("Favorite") { toggleFavorite() }
    .popoverTip(FavoriteTip())
```

### 2.4 Account setup wizard

Multi-step form collecting profile info, preferences, interests.

```swift
NavigationStack(path: $wizardPath) {
    NameStepView()
        .navigationDestination(for: WizardStep.self) { step in
            switch step {
            case .interests: InterestsStepView()
            case .avatar: AvatarStepView()
            case .complete: CompletionView()
            }
        }
}
```

---

## 3. Authentication patterns

### 3.1 Login / sign up form

```swift
Form {
    TextField("Email", text: $email)
        .textContentType(.emailAddress)
        .keyboardType(.emailAddress)
        .autocorrectionDisabled()
    SecureField("Password", text: $password)
        .textContentType(.password)
    Button("Sign In") { authenticate() }
        .buttonStyle(.borderedProminent)
}
```

### 3.2 Sign in with Apple

```swift
import AuthenticationServices

SignInWithAppleButton(.signIn) { request in
    request.requestedScopes = [.fullName, .email]
} onCompletion: { result in
    switch result {
    case .success(let auth):
        guard let credential = auth.credential as? ASAuthorizationAppleIDCredential else { return }
        handleCredential(credential)
    case .failure(let error):
        handleError(error)
    }
}
.signInWithAppleButtonStyle(.black)
.frame(height: 50)
```

### 3.3 Biometric authentication

```swift
import LocalAuthentication

func authenticate() async -> Bool {
    let context = LAContext()
    guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: nil) else { return false }
    return (try? await context.evaluatePolicy(
        .deviceOwnerAuthenticationWithBiometrics,
        localizedReason: "Unlock your account"
    )) ?? false
}
```

### 3.4 OTP / verification code

```swift
TextField("Code", text: $code)
    .textContentType(.oneTimeCode) // Auto-fill from SMS
    .keyboardType(.numberPad)
    .multilineTextAlignment(.center)
    .font(.title.monospaced())
```

---

## 4. List & feed patterns

### 4.1 Standard list with sections

**iOS 26 note**: List section headers now use sentence case and larger text. Remove any `.textCase(.uppercase)` modifiers from section headers — ALL CAPS headers are an iOS 18 pattern.

```swift
List {
    Section("Favorites") {
        ForEach(favorites) { item in ItemRow(item: item) }
    }
    Section("Recent") {
        ForEach(recent) { item in ItemRow(item: item) }
    }
}
.listStyle(.insetGrouped)
```

### 4.2 Infinite scroll / pagination

```swift
ScrollView {
    LazyVStack {
        ForEach(items) { item in
            ItemView(item: item)
                .onAppear {
                    if item == items.last { loadMore() }
                }
        }
        if isLoading { ProgressView() }
    }
}
```

### 4.3 Pull to refresh

```swift
List(items) { item in ItemRow(item: item) }
    .refreshable { await viewModel.refresh() }
```

### 4.4 Swipe actions

```swift
ForEach(items) { item in
    ItemRow(item: item)
        .swipeActions(edge: .trailing, allowsFullSwipe: true) {
            Button(role: .destructive) { delete(item) } label: {
                Label("Delete", systemImage: "trash")
            }
        }
        .swipeActions(edge: .leading) {
            Button { pin(item) } label: { Label("Pin", systemImage: "pin") }
                .tint(.orange)
        }
}
```

### 4.5 Grid layout

```swift
LazyVGrid(columns: [GridItem(.adaptive(minimum: 160))], spacing: 16) {
    ForEach(items) { item in ItemCard(item: item) }
}
.padding()
```

### 4.6 Horizontal carousel

```swift
ScrollView(.horizontal) {
    LazyHStack(spacing: 16) {
        ForEach(featured) { item in
            FeaturedCard(item: item)
                .containerRelativeFrame(.horizontal, count: 1, spacing: 16)
        }
    }
    .scrollTargetLayout()
}
.scrollTargetBehavior(.viewAligned)
.contentMargins(.horizontal, 16)
```

---

## 5. Loading & state patterns

### 5.1 Skeleton / shimmer loading

```swift
// Simple placeholder
ItemRow(item: .placeholder)
    .redacted(reason: .placeholder)

// Custom shimmer: LinearGradient mask with repeating animation
```

### 5.2 Progress indicator

```swift
ProgressView()                              // Indeterminate spinner
ProgressView(value: 0.6)                    // Determinate bar
ProgressView("Uploading...", value: 3, total: 10) // Labeled
```

### 5.3 Empty state

```swift
ContentUnavailableView(
    "No Results",
    systemImage: "magnifyingglass",
    description: Text("Try a different search term.")
)

// Search-specific:
ContentUnavailableView.search(text: searchQuery)
```

### 5.4 Error state

```swift
ContentUnavailableView {
    Label("Connection Error", systemImage: "wifi.slash")
} description: {
    Text("Check your internet connection and try again.")
} actions: {
    Button("Retry") { Task { await loadData() } }
        .buttonStyle(.borderedProminent)
}
```

### 5.5 Overlay loading

```swift
ZStack {
    ContentView()
        .disabled(isLoading)
        .blur(radius: isLoading ? 2 : 0)
    if isLoading {
        ProgressView("Processing...")
            .padding()
            .background(.regularMaterial, in: RoundedRectangle(cornerRadius: 12))
    }
}
```

---

## 6. Search & filter patterns

### 6.1 Search bar

**iOS 26 note**: In tab-based apps, the search bar automatically appears at the bottom of the screen. Use `Tab(..., role: .search)` for a dedicated search tab that transforms into a search field.

```swift
NavigationStack {
    List(filteredItems) { item in ItemRow(item: item) }
        .searchable(text: $searchText, prompt: "Search items...")
}
```

### 6.2 Search scopes

```swift
.searchable(text: $searchText)
.searchScopes($scope) {
    Text("All").tag(SearchScope.all)
    Text("People").tag(SearchScope.people)
    Text("Places").tag(SearchScope.places)
}
```

### 6.3 Search suggestions

```swift
.searchSuggestions {
    ForEach(suggestions) { suggestion in
        Text(suggestion.title).searchCompletion(suggestion.query)
    }
}
```

### 6.4 Filter pill tabs

```swift
ScrollView(.horizontal, showsIndicators: false) {
    HStack(spacing: 8) {
        ForEach(categories) { category in
            Button(category.name) { selected = category }
                .buttonStyle(.bordered)
                .tint(selected == category ? .accentColor : .secondary)
        }
    }
    .padding(.horizontal)
}
```

### 6.5 Segmented control

```swift
Picker("View", selection: $viewMode) {
    Text("List").tag(ViewMode.list)
    Text("Grid").tag(ViewMode.grid)
    Text("Map").tag(ViewMode.map)
}
.pickerStyle(.segmented)
```

---

## 7. Interaction patterns

### 7.1 Swipe gesture (card-based)

```swift
CardView(item: item)
    .offset(x: dragOffset.width)
    .rotationEffect(.degrees(dragOffset.width / 20))
    .gesture(
        DragGesture()
            .onChanged { value in dragOffset = value.translation }
            .onEnded { value in
                if abs(value.translation.width) > 150 {
                    swipe(direction: value.translation.width > 0 ? .right : .left)
                } else {
                    withAnimation(.spring) { dragOffset = .zero }
                }
            }
    )
```

### 7.2 Context menu

```swift
ItemView(item: item)
    .contextMenu {
        Button { copy(item) } label: { Label("Copy", systemImage: "doc.on.doc") }
        Button { share(item) } label: { Label("Share", systemImage: "square.and.arrow.up") }
        Button(role: .destructive) { delete(item) } label: { Label("Delete", systemImage: "trash") }
    } preview: {
        ItemPreview(item: item)
    }
```

### 7.3 Drag and drop

```swift
ForEach(items) { item in
    ItemRow(item: item).draggable(item)
}
.dropDestination(for: Item.self) { droppedItems, location in
    handleDrop(droppedItems)
    return true
}
```

### 7.4 Pinch to zoom

```swift
@State private var scale: CGFloat = 1.0

Image(uiImage: photo)
    .resizable().scaledToFit()
    .scaleEffect(scale)
    .gesture(
        MagnifyGesture()
            .onChanged { value in scale = value.magnification }
            .onEnded { _ in withAnimation { scale = max(1, min(scale, 5)) } }
    )
```

### 7.5 Haptic feedback

```swift
Button("Complete") { complete() }
    .sensoryFeedback(.success, trigger: isCompleted)
```

### 7.6 Double tap

```swift
Image(uiImage: photo)
    .onTapGesture(count: 2) {
        withAnimation(.spring) { isZoomed.toggle() }
    }
```

---

## 8. Overlay & alert patterns

### 8.1 Alert dialog

**iOS 26 note**: Alerts now use left-aligned text (no longer centered). No code changes needed — this is a system-level visual change.

```swift
.alert("Delete Item?", isPresented: $showDeleteAlert) {
    Button("Cancel", role: .cancel) {}
    Button("Delete", role: .destructive) { deleteItem() }
} message: {
    Text("This action cannot be undone.")
}
```

### 8.2 Confirmation dialog

**iOS 26 note**: Action sheets now anchor to the source view on all devices (no longer full-width bottom on iPhone). Verify your layout accommodates this change.

```swift
.confirmationDialog("Sort By", isPresented: $showSort, titleVisibility: .visible) {
    Button("Date") { sortBy = .date }
    Button("Name") { sortBy = .name }
    Button("Size") { sortBy = .size }
}
```

### 8.3 Toast / snackbar (custom)

```swift
ZStack(alignment: .bottom) {
    content
    if showToast {
        Text("Item saved successfully")
            .padding()
            .glassEffect(.regular, in: .capsule)  // iOS 26: use glass for floating feedback
            .transition(.move(edge: .bottom).combined(with: .opacity))
            .task { try? await Task.sleep(for: .seconds(2)); showToast = false }
    }
}
.animation(.spring, value: showToast)
```

### 8.4 Bottom sheet with detents

**iOS 26 note**: Sheets get Liquid Glass backgrounds automatically. Do NOT add `.presentationBackground()` — it blocks glass styling.

```swift
.sheet(isPresented: $showFilter) {
    FilterView()
        .presentationDetents([.fraction(0.3), .medium, .large])
        .presentationBackgroundInteraction(.enabled(upThrough: .medium))
        // iOS 26: remove .presentationBackground() to allow glass
}
```

### 8.5 Popover

```swift
Button("Info") { showInfo = true }
    .popover(isPresented: $showInfo) {
        VStack { Text("Details here") }.padding().frame(width: 250)
    }
```

---

## 9. Component patterns

### 9.1 Floating action button

A FAB is a floating navigation-layer element — use Liquid Glass in iOS 26.

```swift
ZStack(alignment: .bottomTrailing) {
    ScrollView { /* content */ }
    Button { showCreate = true } label: {
        Image(systemName: "plus")
            .font(.title2.bold())
            .frame(width: 56, height: 56)
    }
    .glassEffect(.regular, in: .circle)  // iOS 26: glass for floating controls
    .padding()
}
```

### 9.2 Badge

```swift
Tab("Inbox", systemImage: "tray") { InboxView() }
    .badge(unreadCount)
```

### 9.3 Step indicator

```swift
HStack {
    ForEach(0..<totalSteps, id: \.self) { step in
        Circle()
            .fill(step <= currentStep ? Color.accentColor : Color.gray.opacity(0.3))
            .frame(width: 12, height: 12)
        if step < totalSteps - 1 {
            Rectangle()
                .fill(step < currentStep ? Color.accentColor : Color.gray.opacity(0.3))
                .frame(height: 2)
        }
    }
}
```

### 9.4 Avatar

```swift
AsyncImage(url: user.avatarURL) { image in
    image.resizable().scaledToFill()
} placeholder: {
    Image(systemName: "person.fill").foregroundStyle(.secondary)
}
.frame(width: 44, height: 44)
.clipShape(Circle())
```

### 9.5 Chips / tags

```swift
// Use Layout protocol or WrappingHStack for flow layout
ForEach(tags) { tag in
    Text(tag.name)
        .font(.caption)
        .padding(.horizontal, 12)
        .padding(.vertical, 6)
        .background(.secondary.opacity(0.2), in: Capsule())
}
```

### 9.6 Countdown timer

```swift
Text(timerInterval: Date.now...endDate, countsDown: true)
    .font(.title.monospacedDigit())
```

---

## 10. Settings & preferences

### 10.1 Grouped settings form

```swift
Form {
    Section("General") {
        Toggle("Notifications", isOn: $notifications)
        Toggle("Dark Mode", isOn: $darkMode)
    }
    Section("Display") {
        Picker("Theme", selection: $theme) {
            ForEach(Theme.allCases) { Text($0.name).tag($0) }
        }
        Slider(value: $fontSize, in: 12...24, step: 1) { Text("Font Size") }
    }
}
.formStyle(.grouped)
```

### 10.2 Account section

```swift
Section {
    HStack(spacing: 16) {
        AsyncImage(url: user.avatar) { $0.resizable().scaledToFill() } placeholder: { Color.gray }
            .frame(width: 60, height: 60).clipShape(Circle())
        VStack(alignment: .leading) {
            Text(user.name).font(.headline)
            Text(user.email).font(.subheadline).foregroundStyle(.secondary)
        }
    }
}
```

### 10.3 About / legal

```swift
Section("About") {
    LabeledContent("Version", value: appVersion)
    NavigationLink("Privacy Policy") { WebView(url: privacyURL) }
    NavigationLink("Terms of Service") { WebView(url: termsURL) }
}
```

---

## 11. Dashboard & analytics

### 11.1 KPI tiles

```swift
LazyVGrid(columns: [GridItem(.adaptive(minimum: 150))], spacing: 16) {
    StatCard(title: "Revenue", value: "$12,450", delta: "+12%", positive: true)
    StatCard(title: "Users", value: "1,234", delta: "+5%", positive: true)
}
```

### 11.2 Charts (Swift Charts)

```swift
import Charts

Chart(salesData) { item in
    BarMark(x: .value("Month", item.month), y: .value("Sales", item.amount))
        .foregroundStyle(by: .value("Category", item.category))
}
.frame(height: 200)

// Pie/donut (iOS 17+)
Chart(breakdown) {
    SectorMark(angle: .value("Amount", $0.value), angularInset: 1)
        .foregroundStyle(by: .value("Category", $0.name))
}
```

### 11.3 Delta indicator

```swift
HStack(spacing: 4) {
    Image(systemName: isPositive ? "arrow.up.right" : "arrow.down.right")
    Text(deltaText)
}
.font(.caption.bold())
.foregroundStyle(isPositive ? .green : .red)
```

---

## 12. E-commerce patterns

### 12.1 Product card

```swift
VStack(alignment: .leading, spacing: 8) {
    AsyncImage(url: product.imageURL) { $0.resizable().scaledToFill() } placeholder: { Color.gray }
        .frame(height: 180).clipped().clipShape(RoundedRectangle(cornerRadius: 12))
    Text(product.name).font(.subheadline).lineLimit(2)
    HStack {
        Text(product.price, format: .currency(code: "USD")).font(.headline)
        Spacer()
        HStack(spacing: 2) {
            Image(systemName: "star.fill").foregroundStyle(.yellow)
            Text(String(format: "%.1f", product.rating)).font(.caption)
        }
    }
}
```

### 12.2 Product detail

```swift
ScrollView {
    TabView { ForEach(product.images) { AsyncImage(url: $0) } }
        .tabViewStyle(.page).frame(height: 350)
    VStack(alignment: .leading) {
        Text(product.name).font(.title2.bold())
        Text(product.price, format: .currency(code: "USD")).font(.title3)
        Picker("Size", selection: $size) { ForEach(sizes) { Text($0).tag($0) } }
            .pickerStyle(.segmented)
    }.padding()
}
.safeAreaInset(edge: .bottom) {
    Button("Add to Cart") { addToCart() }
        .buttonStyle(.borderedProminent).frame(maxWidth: .infinity).padding()
}
```

### 12.3 Wishlist / favorites

```swift
Button { toggleFavorite() } label: {
    Image(systemName: isFavorite ? "heart.fill" : "heart")
        .foregroundStyle(isFavorite ? .red : .secondary)
}
.sensoryFeedback(.impact(flexibility: .soft), trigger: isFavorite)
```

---

## 13. Social media patterns

### 13.1 Social feed

```swift
ScrollView {
    LazyVStack(spacing: 0) {
        ForEach(posts) { post in
            PostView(post: post)
            Divider()
        }
    }
}
.refreshable { await viewModel.refresh() }
```

### 13.2 Stories bar

```swift
ScrollView(.horizontal, showsIndicators: false) {
    LazyHStack(spacing: 12) {
        ForEach(stories) { story in
            VStack {
                AsyncImage(url: story.avatar) { $0.resizable().scaledToFill() } placeholder: { Color.gray }
                    .frame(width: 64, height: 64).clipShape(Circle())
                    .overlay(Circle().stroke(
                        LinearGradient(colors: [.purple, .orange, .red],
                                       startPoint: .topLeading, endPoint: .bottomTrailing),
                        lineWidth: story.isViewed ? 0 : 3
                    ))
                Text(story.name).font(.caption2).lineLimit(1)
            }.frame(width: 72)
        }
    }.padding(.horizontal)
}
```

### 13.3 Like animation

```swift
Image(uiImage: photo)
    .onTapGesture(count: 2) {
        isLiked = true; showHeart = true
        Task { try? await Task.sleep(for: .seconds(1)); showHeart = false }
    }
    .overlay {
        Image(systemName: "heart.fill")
            .font(.system(size: 80)).foregroundStyle(.white)
            .scaleEffect(showHeart ? 1 : 0).opacity(showHeart ? 1 : 0)
            .animation(.spring(duration: 0.4), value: showHeart)
    }
    .sensoryFeedback(.impact, trigger: showHeart)
```

### 13.4 Profile screen

```swift
ScrollView {
    // Header with avatar + stats
    HStack(spacing: 16) {
        AsyncImage(url: user.avatar) { $0.resizable().scaledToFill() } placeholder: { Color.gray }
            .frame(width: 80, height: 80).clipShape(Circle())
        HStack(spacing: 24) {
            StatColumn(value: "\(user.posts)", label: "Posts")
            StatColumn(value: "\(user.followers)", label: "Followers")
            StatColumn(value: "\(user.following)", label: "Following")
        }
    }
    // Tab picker + grid
    Picker("Content", selection: $tab) {
        Image(systemName: "square.grid.3x3").tag(0)
        Image(systemName: "heart").tag(1)
    }.pickerStyle(.segmented)
    LazyVGrid(columns: [GridItem(.adaptive(minimum: 110))], spacing: 2) {
        ForEach(contentItems) { item in
            AsyncImage(url: item.thumbnail) { $0.resizable().scaledToFill() } placeholder: { Color.gray }
                .frame(height: 120).clipped()
        }
    }
}
```

---

## 14. Messaging patterns

### 14.1 Conversation list

```swift
List(conversations) { convo in
    NavigationLink(value: convo) {
        HStack(spacing: 12) {
            AsyncImage(url: convo.avatar) { $0.resizable().scaledToFill() } placeholder: { Color.gray }
                .frame(width: 50, height: 50).clipShape(Circle())
            VStack(alignment: .leading) {
                HStack {
                    Text(convo.name).font(.headline)
                    Spacer()
                    Text(convo.timestamp, style: .relative).font(.caption).foregroundStyle(.secondary)
                }
                Text(convo.lastMessage).font(.subheadline).foregroundStyle(.secondary).lineLimit(1)
            }
        }
    }
    .badge(convo.unreadCount)
}
```

### 14.2 Chat bubbles

```swift
ScrollViewReader { proxy in
    ScrollView {
        LazyVStack(spacing: 4) {
            ForEach(messages) { msg in
                HStack {
                    if msg.isMine { Spacer(minLength: 60) }
                    Text(msg.text)
                        .padding(12)
                        .background(msg.isMine ? Color.accentColor : Color(.systemGray5))
                        .foregroundStyle(msg.isMine ? .white : .primary)
                        .clipShape(RoundedRectangle(cornerRadius: 16))
                    if !msg.isMine { Spacer(minLength: 60) }
                }.id(msg.id)
            }
        }.padding()
    }
    .onChange(of: messages.count) { proxy.scrollTo(messages.last?.id, anchor: .bottom) }
}
```

### 14.3 Message input bar

```swift
.safeAreaInset(edge: .bottom) {
    HStack(spacing: 12) {
        Button { showAttachments = true } label: { Image(systemName: "plus.circle.fill") }
        TextField("Message", text: $messageText, axis: .vertical)
            .textFieldStyle(.roundedBorder)
            .lineLimit(1...5)
        Button { sendMessage() } label: { Image(systemName: "arrow.up.circle.fill") }
            .disabled(messageText.isEmpty)
    }
    .padding()
    .background(.bar)
}
```

### 14.4 Typing indicator

```swift
HStack(spacing: 4) {
    ForEach(0..<3, id: \.self) { i in
        Circle().fill(.secondary).frame(width: 8, height: 8)
            .offset(y: isAnimating ? -4 : 4)
            .animation(.easeInOut(duration: 0.5).repeatForever().delay(Double(i) * 0.15),
                        value: isAnimating)
    }
}
.padding(12)
.background(Color(.systemGray5), in: RoundedRectangle(cornerRadius: 16))
.onAppear { isAnimating = true }
```

---

## 15. Media patterns

### 15.1 Image viewer with zoom

```swift
TabView(selection: $currentIndex) {
    ForEach(Array(images.enumerated()), id: \.offset) { index, image in
        AsyncImage(url: image.url) { $0.resizable().scaledToFit() } placeholder: { ProgressView() }
            .scaleEffect(scale)
            .gesture(MagnifyGesture().onChanged { scale = $0.magnification }
                .onEnded { _ in withAnimation { scale = max(1, min(scale, 5)) } })
            .onTapGesture(count: 2) { withAnimation { scale = scale > 1 ? 1 : 3 } }
            .tag(index)
    }
}
.tabViewStyle(.page)
```

### 15.2 Video player

```swift
import AVKit
VideoPlayer(player: AVPlayer(url: videoURL))
    .frame(height: 300)
```

### 15.3 Persistent mini player

**iOS 26**: Use `.tabViewBottomAccessory {}` for persistent mini players above the tab bar. This is the native approach — it remains visible across tab switches and integrates with the glass tab bar.

```swift
TabView { /* tabs */ }
    .tabViewBottomAccessory {
        if isPlaying {
            MiniPlayerBar(track: currentTrack) { showFullPlayer = true }
        }
    }
    .sheet(isPresented: $showFullPlayer) { FullPlayerView() }
```

**Pre-iOS 26 fallback** (if supporting older versions):
```swift
TabView { /* tabs */ }
    .safeAreaInset(edge: .bottom) {
        if isPlaying {
            MiniPlayerBar(track: currentTrack) { showFullPlayer = true }
                .background(.bar)
        }
    }
```

---

## 16. Form & input patterns

### 16.1 Multi-step wizard

```swift
@State private var step = 1
TabView(selection: $step) {
    StepOneView(onNext: { step = 2 }).tag(1)
    StepTwoView(onNext: { step = 3 }, onBack: { step = 1 }).tag(2)
    StepThreeView(onSubmit: { submit() }, onBack: { step = 2 }).tag(3)
}
.tabViewStyle(.page(indexDisplayMode: .never))
```

### 16.2 Inline validation

```swift
VStack(alignment: .leading) {
    TextField("Email", text: $email)
        .textContentType(.emailAddress)
        .onChange(of: email) { _, new in emailError = validateEmail(new) }
    if let error = emailError {
        Text(error).font(.caption).foregroundStyle(.red)
    }
}
```

### 16.3 Photo picker

```swift
import PhotosUI
PhotosPicker(selection: $selectedItem, matching: .images) {
    Label("Select Photo", systemImage: "photo.on.rectangle")
}
```

### 16.4 Rich text editor (iOS 26)

```swift
@State private var text = AttributedString("Hello, world!")
TextEditor(text: $text) // iOS 26: AttributedString binding for rich text
```

---

## 17. iOS 26 Liquid Glass patterns

### 17.1 Glass navigation & tab bars

Automatic -- all NavigationStack toolbars and TabView tab bars get Liquid Glass on iOS 26. Do NOT add custom `.background()` to toolbars.

### 17.2 Custom glass effect

```swift
Button { action() } label: { Label("Action", systemImage: "star") }
    .glassEffect()           // Regular glass
    .glassEffect(.clear)     // Subtle glass
    .glassEffect(.identity)  // No effect (for conditional toggling)
```

### 17.3 Glass grouping

```swift
GlassEffectContainer {
    HStack(spacing: 8) {
        Button("A") {}.glassEffect()
        Button("B") {}.glassEffect()
    }
}
```

### 17.4 Glass tinting

```swift
.glassEffect(.regular.tint(.blue))
```

### 17.5 Tab bar minimization

```swift
TabView { /* tabs */ }
    .tabBarMinimizeBehavior(.onScrollDown)
```

### 17.6 WebView (NEW)

```swift
WebView(url: URL(string: "https://example.com")!)
```

---

## 18. Top 50 summary table

| # | Pattern | Frequency | Category |
|---|---------|-----------|----------|
| 1 | Tab-based navigation | Every app | Navigation |
| 2 | Push navigation (NavigationStack) | Every app | Navigation |
| 3 | List with sections | Very common | Lists |
| 4 | Pull to refresh | Very common | Lists |
| 5 | Sheet / modal | Very common | Navigation |
| 6 | Search bar (.searchable) | Very common | Search |
| 7 | Form / settings screen | Very common | Settings |
| 8 | Empty state (ContentUnavailableView) | Very common | States |
| 9 | Loading indicator (ProgressView) | Very common | States |
| 10 | Alert dialog | Very common | Overlays |
| 11 | Swipe actions on rows | Common | Interaction |
| 12 | Context menu (long press) | Common | Interaction |
| 13 | Infinite scroll / pagination | Common | Lists |
| 14 | Segmented control | Common | Components |
| 15 | Bottom sheet with detents | Common | Overlays |
| 16 | Grid layout (LazyVGrid) | Common | Lists |
| 17 | Horizontal carousel | Common | Lists |
| 18 | Skeleton loading | Common | States |
| 19 | Onboarding carousel | Common | Onboarding |
| 20 | Login / sign up | Common | Auth |
| 21 | Sign in with Apple | Common | Auth |
| 22 | Profile screen | Common | Social |
| 23 | Social feed | Common | Social |
| 24 | Chat bubbles | Common | Messaging |
| 25 | FAB (floating action button) | Common | Components |
| 26 | Badge / notification count | Common | Components |
| 27 | Toast / snackbar | Common | Overlays |
| 28 | Haptic feedback | Common | Interaction |
| 29 | Date picker | Common | Forms |
| 30 | Photo picker | Common | Forms |
| 31 | Charts (Swift Charts) | Common | Dashboard |
| 32 | Product card | Common | E-commerce |
| 33 | Filter pills | Common | Search |
| 34 | Confirmation dialog | Common | Overlays |
| 35 | Drag and drop | Moderate | Interaction |
| 36 | Pinch to zoom | Moderate | Interaction |
| 37 | Multi-step wizard | Moderate | Forms |
| 38 | Step indicator | Moderate | Components |
| 39 | Stories bar | Moderate | Social |
| 40 | Typing indicator | Moderate | Messaging |
| 41 | Video player | Moderate | Media |
| 42 | Image gallery/viewer | Moderate | Media |
| 43 | Shopping cart | Moderate | E-commerce |
| 44 | Checkout flow | Moderate | E-commerce |
| 45 | Wishlist/favorites | Moderate | E-commerce |
| 46 | Dashboard KPI tiles | Moderate | Dashboard |
| 47 | Mini player bar | Moderate | Media |
| 48 | Custom glass effects | Growing (iOS 26) | iOS 26 |
| 49 | Rich text editor | Growing (iOS 26) | iOS 26 |
| 50 | WebView | Growing (iOS 26) | iOS 26 |
