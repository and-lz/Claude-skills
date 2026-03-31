# Companion App, Handoff & Cross-Device Patterns

## Table of contents
1. Handoff (NSUserActivity)
2. Continuity Keyboard
3. QR code sign-in
4. Universal Links (tvOS + iOS)
5. CloudKit cross-device sync
6. Rules and anti-patterns

---

## 1. Handoff (NSUserActivity)

Handoff lets users seamlessly continue a task from Apple TV on iPhone/iPad, or vice versa. Most common pattern: start watching on iPhone → continue on TV, or start on TV → continue on phone.

### tvOS side — advertising an activity

```swift
struct DetailView: View {
    let movie: Movie
    @State var currentTime: TimeInterval = 0

    var body: some View {
        ContentView()
            .userActivity("com.yourapp.watching") { activity in
                activity.title = movie.title
                activity.userInfo = [
                    "movieID": movie.id,
                    "playbackTime": currentTime
                ]
                activity.webpageURL = URL(string: "https://yourapp.com/movie/\(movie.id)")
                activity.isEligibleForHandoff = true
                activity.becomeCurrent()
            }
    }
}
```

### iOS side — receiving the activity

```swift
@main struct MyIOSApp: App {
    var body: some Scene {
        WindowGroup {
            RootView()
                .onContinueUserActivity("com.yourapp.watching") { activity in
                    guard
                        let movieID = activity.userInfo?["movieID"] as? String,
                        let time = activity.userInfo?["playbackTime"] as? TimeInterval
                    else { return }
                    navigationModel.openMovie(id: movieID, resumeAt: time)
                }
        }
    }
}
```

### Info.plist requirements (both iOS and tvOS targets)

```xml
<key>NSUserActivityTypes</key>
<array>
    <string>com.yourapp.watching</string>
    <string>com.yourapp.browsing</string>
</array>
```

**Handoff requirements:**
- Both devices signed into the same iCloud account with Bluetooth/Wi-Fi enabled
- App installed on both platforms with the same Apple Developer Team ID
- Same `NSUserActivityTypes` entries in **both** iOS and tvOS Info.plist files
- Activity `isEligibleForHandoff = true`

---

## 2. Continuity Keyboard

When a text field is focused on Apple TV, nearby iPhone/iPad/Mac devices show an input prompt. No code required — automatic for `TextField`, `SecureField`, and `.searchable()`.

```swift
// These trigger Continuity Keyboard automatically:
TextField("Search", text: $query)

NavigationStack { }
    .searchable(text: $query, prompt: "Movies, shows, people")

SecureField("Password", text: $password)
```

**Design guidance:**
- Minimize text input fields — prefer selection lists, pickers, and Siri dictation
- For search, `.searchable()` > raw `TextField` — better Siri and Continuity Keyboard integration
- Login: always offer QR code or TV Provider SSO as alternatives to typing credentials

---

## 3. QR code sign-in

The most user-friendly authentication flow for tvOS. User scans a QR code on the TV screen with their iPhone camera. Used by Netflix, Disney+, HBO Max. Implements OAuth 2.0 Device Authorization Grant (RFC 8628).

```swift
import CoreImage.CIFilterBuiltins

struct QRSignInView: View {
    @State private var verificationURL: URL?
    @State private var pollTask: Task<Void, Never>?

    var body: some View {
        VStack(spacing: 40) {
            if let url = verificationURL {
                QRCodeImage(url: url)
                    .frame(width: 280, height: 280)
                Text("Scan with your iPhone to sign in")
                    .font(.title3)
                    .foregroundStyle(.secondary)
                Text("Or visit \(url.host ?? "")")
                    .font(.footnote)
                    .foregroundStyle(.tertiary)
            } else {
                ProgressView("Preparing sign-in…")
            }
        }
        .task { await startDeviceFlow() }
        .onDisappear { pollTask?.cancel() }
    }

    func startDeviceFlow() async {
        guard let response = try? await api.startDeviceAuthorization() else { return }
        verificationURL = response.verificationURI

        pollTask = Task {
            while !Task.isCancelled {
                try? await Task.sleep(for: .seconds(Double(response.interval)))
                if let token = try? await api.pollDeviceToken(response.deviceCode) {
                    await auth.setToken(token)
                    return
                }
            }
        }
    }
}

struct QRCodeImage: View {
    let url: URL

    var qrImage: UIImage? {
        let filter = CIFilter.qrCodeGenerator()
        filter.message = Data(url.absoluteString.utf8)
        filter.correctionLevel = "M"
        guard let ci = filter.outputImage else { return nil }
        let scaled = ci.transformed(by: CGAffineTransform(scaleX: 10, y: 10))
        return UIImage(ciImage: scaled)
    }

    var body: some View {
        if let image = qrImage {
            Image(uiImage: image)
                .interpolation(.none)
                .resizable()
                .scaledToFit()
                .accessibilityLabel("QR code — scan with your iPhone to sign in")
        }
    }
}
```

---

## 4. Universal Links (tvOS + iOS)

Universal Links work on tvOS, routing web URLs directly into your TV app.

```swift
@main struct MyTVApp: App {
    var body: some Scene {
        WindowGroup {
            RootView()
                // Universal Links (web URLs)
                .onContinueUserActivity(NSUserActivityTypeBrowsingWeb) { activity in
                    guard let url = activity.webpageURL else { return }
                    router.handle(url: url)
                }
                // Custom URL schemes
                .onOpenURL { url in
                    router.handle(url: url)
                }
        }
    }
}
```

### Apple App Site Association (AASA)

Include **both** iOS and tvOS app IDs in your AASA file. Host at `https://yourapp.com/.well-known/apple-app-site-association`.

```json
{
  "applinks": {
    "details": [
      {
        "appIDs": [
          "TEAM123.com.yourapp",
          "TEAM123.com.yourapp.tvos"
        ],
        "components": [
          { "/": "/movie/*" },
          { "/": "/show/*" },
          { "/": "/browse" }
        ]
      }
    ]
  }
}
```

Add Associated Domains entitlement: `applinks:yourapp.com` to **both** iOS and tvOS targets.

---

## 5. CloudKit cross-device sync

Sync watchlists, preferences, and playback progress between tvOS and iOS/iPadOS via CloudKit. Both platforms fully support CloudKit.

### SwiftData + CloudKit (recommended)

```swift
// Shared model — same file compiled into both iOS and tvOS targets
@Model
class WatchlistItem {
    @Attribute(.unique) var contentID: String
    var title: String
    var progress: Double      // 0.0 to 1.0 playback position
    var addedDate: Date
    var lastUpdated: Date

    init(contentID: String, title: String) {
        self.contentID = contentID
        self.title = title
        self.progress = 0.0
        self.addedDate = .now
        self.lastUpdated = .now
    }
}

// App root — enable CloudKit sync (same code in both iOS and tvOS App targets)
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup { ContentView() }
        .modelContainer(
            for: WatchlistItem.self,
            configurations: ModelConfiguration(
                cloudKitDatabase: .automatic  // Uses default iCloud container
            )
        )
    }
}
```

**CloudKit sync requirements:**
- CloudKit capability added to both iOS and tvOS targets (same container ID)
- User signed into iCloud on both devices
- `@Model` properties must be Optional or have default values (CloudKit limitation)
- No optional → required schema changes without migration

### Encrypt sensitive data

```swift
@Model
class UserPreferences {
    @Attribute(.allowsCloudEncryption) var authToken: String?  // Encrypted in CloudKit
    var theme: String = "system"
    var subtitlesEnabled: Bool = false
}
```

### Conflict resolution for playback progress

CloudKit private database uses last-writer-wins by default. For playback progress, keep the further-along value:

```swift
extension WatchlistItem {
    func syncProgress(_ newProgress: Double, updatedAt date: Date) {
        // Keep the greater progress, or the more recent update
        if date > lastUpdated || newProgress > progress {
            progress = newProgress
            lastUpdated = date
        }
    }
}
```

---

## 6. Rules and anti-patterns

### DO:
- Implement Handoff for playback continuity (TV ↔ iPhone)
- Use QR code sign-in as the primary first-time auth flow
- Share `@Model` types across iOS and tvOS in a shared Swift Package or folder
- Include both app IDs in the AASA file for Universal Links on both platforms
- Use `cloudKitDatabase: .automatic` — simpler than manual container config
- Declare `NSUserActivityTypes` in **both** Info.plist files

### DON'T:
- Don't require a companion iOS app — tvOS app must function completely standalone
- Don't use Handoff for real-time state sync — it's opportunistic (Bluetooth proximity required)
- Don't make sign-in require typing email/password on-screen — provide QR code alternative
- Don't sync large media blobs via CloudKit — store asset URLs, sync metadata only
- Don't forget to declare `NSUserActivityTypes` in **both** iOS and tvOS Info.plist files
- Don't block app launch on CloudKit sync — show local data immediately, sync in background
