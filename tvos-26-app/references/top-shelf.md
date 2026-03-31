# Top Shelf — Complete Reference

## Table of contents
1. What is Top Shelf
2. TVTopShelfContentProvider
3. Sectioned content
4. Inset content
5. Deep linking
6. Updating content
7. Configuration
8. Rules and anti-patterns

---

## 1. What is Top Shelf

The Top Shelf is the large banner area that appears at the top of the Apple TV Home screen when your app's icon is in the **top row**. It's prime real estate for showcasing content — movies, shows, music, or featured items.

Two display styles:
- **Sectioned**: Multiple horizontal rows of poster-sized items (like shelf rows)
- **Inset**: Full-width banner images that scroll horizontally (more cinematic)

## 2. TVTopShelfContentProvider

Create a **TV Top Shelf Extension** target in your Xcode project.

```swift
import TVServices

class TopShelfProvider: TVTopShelfContentProvider {

    override func loadTopShelfContent() async -> TVTopShelfContent? {
        do {
            let featured = try await fetchFeaturedContent()
            return buildSectionedContent(from: featured)
            // OR: return buildInsetContent(from: featured)
        } catch {
            return nil  // Fallback to static Top Shelf image
        }
    }
}
```

## 3. Sectioned content

Multiple rows of poster-sized items. Best for browsing categories.

```swift
func buildSectionedContent(from data: FeaturedData) -> TVTopShelfSectionedContent {

    // Build sections
    let trendingSection = TVTopShelfItemCollection<TVTopShelfSectionedItem>(
        items: data.trending.map { item in
            let topShelfItem = TVTopShelfSectionedItem(identifier: item.id)
            topShelfItem.title = item.title

            // Image: 250x375pt for portrait posters (1x scale)
            topShelfItem.setImageURL(item.posterURL, for: .screenScale1x)
            // 2x image for Apple TV 4K
            topShelfItem.setImageURL(item.posterURL2x, for: .screenScale2x)

            // Actions
            topShelfItem.displayAction = TVTopShelfAction(url: item.deepLinkURL)
            topShelfItem.playAction = TVTopShelfAction(url: item.playURL)

            return topShelfItem
        }
    )
    trendingSection.title = "Trending Now"

    let newReleasesSection = TVTopShelfItemCollection<TVTopShelfSectionedItem>(
        items: data.newReleases.map { item in
            let topShelfItem = TVTopShelfSectionedItem(identifier: item.id)
            topShelfItem.title = item.title
            topShelfItem.setImageURL(item.posterURL, for: .screenScale1x)
            topShelfItem.displayAction = TVTopShelfAction(url: item.deepLinkURL)
            return topShelfItem
        }
    )
    newReleasesSection.title = "New Releases"

    return TVTopShelfSectionedContent(sections: [trendingSection, newReleasesSection])
}
```

### Image sizes for sectioned content

| Style | Size (1x) | Size (2x) | Aspect ratio |
|-------|-----------|-----------|--------------|
| Portrait poster | 250 × 375pt | 500 × 750px | 2:3 |
| Landscape poster | 350 × 197pt | 700 × 394px | 16:9 |
| Square | 250 × 250pt | 500 × 500px | 1:1 |

## 4. Inset content

Full-width cinematic banners. Best for featured/hero content.

```swift
func buildInsetContent(from data: FeaturedData) -> TVTopShelfInsetContent {
    let items = data.featured.map { item in
        let insetItem = TVTopShelfInsetItem(identifier: item.id)
        insetItem.title = item.title

        // Full-width banner image: 1940x1140pt (extends beyond safe area)
        insetItem.setImageURL(item.bannerURL, for: .screenScale1x)
        insetItem.setImageURL(item.bannerURL2x, for: .screenScale2x)

        // Content image (smaller, for focus state detail)
        insetItem.setContentURL(item.contentURL, for: .screenScale1x)

        // Actions
        insetItem.displayAction = TVTopShelfAction(url: item.deepLinkURL)
        insetItem.playAction = TVTopShelfAction(url: item.playURL)

        return insetItem
    }

    return TVTopShelfInsetContent(items: items)
}
```

### Image sizes for inset content

| Image type | Size (1x) | Size (2x) |
|------------|-----------|-----------|
| Banner (main) | 1940 × 1140pt | 3880 × 2280px |
| Content (detail) | 921 × 507pt | 1842 × 1014px |

## 5. Deep linking

Top Shelf items open your app via URL actions. Handle these in your app:

```swift
// App-level URL handling
@main
struct MyTVApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .onOpenURL { url in
                    handleTopShelfURL(url)
                }
        }
    }

    func handleTopShelfURL(_ url: URL) {
        // Parse URL scheme
        // e.g., myapp://content/movie/12345
        // e.g., myapp://play/movie/12345

        guard let components = URLComponents(url: url, resolvingAgainstBaseURL: false) else { return }

        let pathParts = components.path.split(separator: "/")

        if pathParts.first == "play" {
            // Start playback immediately
            navigateToPlayer(contentID: String(pathParts.last ?? ""))
        } else {
            // Show detail page
            navigateToDetail(contentID: String(pathParts.last ?? ""))
        }
    }
}
```

### URL scheme setup

```xml
<!-- Info.plist -->
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>myapp</string>
        </array>
    </dict>
</array>
```

## 6. Updating content

```swift
// Force Top Shelf to reload
import TVServices

func refreshTopShelf() {
    TVTopShelfContentProvider.topShelfContentDidChange()
}

// Call this when:
// - New content is available
// - User's watchlist changes
// - Featured content rotates
// - User signs in/out
```

### Sharing data with the extension

Top Shelf runs as a separate extension process. Share data via **App Groups**:

```swift
// In both app and extension targets:
let sharedDefaults = UserDefaults(suiteName: "group.com.myapp.shared")

// App writes
sharedDefaults?.set(featuredIDs, forKey: "topShelfFeatured")

// Extension reads
let featuredIDs = sharedDefaults?.array(forKey: "topShelfFeatured") as? [String] ?? []
```

Or use a shared Core Data / SwiftData store:

```swift
// Shared container
let container = try ModelContainer(
    for: ContentItem.self,
    configurations: ModelConfiguration(
        groupContainer: .identifier("group.com.myapp.shared")
    )
)
```

## 7. Configuration

### Adding the extension target

1. File → New → Target → **TV Top Shelf Extension**
2. Set the App Group capability on both app and extension
3. The extension's `Info.plist` needs:
   ```xml
   <key>NSExtension</key>
   <dict>
       <key>NSExtensionPointIdentifier</key>
       <string>com.apple.tv-top-shelf</string>
       <key>NSExtensionPrincipalClass</key>
       <string>$(PRODUCT_MODULE_NAME).TopShelfProvider</string>
   </dict>
   ```

### Static fallback

If the extension returns `nil` or fails, the system shows a static Top Shelf image from your app's asset catalog:
- Add an image set named **"Top Shelf Image"** to your tvOS asset catalog
- Size: 1920 × 720pt (1x), 3840 × 1440px (2x)
- Or **"Top Shelf Image Wide"**: 2320 × 720pt (1x)

## 8. Rules and anti-patterns

### DO:
- Provide both `displayAction` and `playAction` — display shows detail, play starts playback
- Use high-quality poster/banner images at correct sizes
- Share data between app and extension via App Groups
- Call `TVTopShelfContentProvider.topShelfContentDidChange()` when content updates
- Provide a static fallback image in case the extension fails
- Load content asynchronously — `loadTopShelfContent()` is async

### DON'T:
- Don't make network calls that take too long — the system has a timeout
- Don't return too many items per section (5-10 is ideal, max ~20)
- Don't use generic/placeholder images — Top Shelf is premium real estate
- Don't forget App Groups — the extension runs in a separate process
- Don't forget to handle deep link URLs in your app
- Don't use Top Shelf for non-content purposes (it's not a widget or notification)