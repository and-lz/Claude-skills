# Deep Linking, Authentication, StoreKit 2 & SharePlay

## Table of contents
1. URL Schemes & Deep Linking
2. Universal Links
3. App Clips
4. Sign in with Apple
5. Passkeys (Platform Credentials)
6. StoreKit 2 (In-App Purchases)
7. Subscriptions & Paywalls
8. SharePlay / Group Activities
9. Declared Age Range (NEW in iOS 26)
10. Anti-Patterns

---

## 1. URL Schemes & Deep Linking

### Registering URL schemes in Info.plist

Add `CFBundleURLTypes` to your Info.plist (or project.yml for XcodeGen):

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLName</key>
        <string>com.example.myapp</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>myapp</string>
        </array>
    </dict>
</array>
```

XcodeGen equivalent in `project.yml`:
```yaml
settings:
  INFOPLIST_KEY_CFBundleURLTypes:
    - CFBundleURLName: com.example.myapp
      CFBundleURLSchemes: [myapp]
```

### Enum-based route parsing

```swift
enum DeepLink {
    case product(id: String)
    case profile(username: String)
    case settings
    case search(query: String)

    static func from(_ url: URL) -> DeepLink? {
        guard let components = URLComponents(url: url, resolvingAgainstBaseURL: false) else {
            return nil
        }

        let pathComponents = components.path
            .split(separator: "/")
            .map(String.init)

        switch pathComponents.first {
        case "product":
            guard let id = pathComponents.dropFirst().first else { return nil }
            return .product(id: id)
        case "profile":
            guard let username = pathComponents.dropFirst().first else { return nil }
            return .profile(username: username)
        case "settings":
            return .settings
        case "search":
            let query = components.queryItems?
                .first(where: { $0.name == "q" })?.value ?? ""
            return .search(query: query)
        default:
            return nil
        }
    }
}
```

### Handling in SwiftUI with NavigationPath

```swift
@Observable
final class Router {
    var path = NavigationPath()

    func handle(_ deepLink: DeepLink) {
        switch deepLink {
        case .product(let id):
            path.append(ProductDestination(id: id))
        case .profile(let username):
            path.append(ProfileDestination(username: username))
        case .settings:
            path.append(SettingsDestination())
        case .search(let query):
            path.append(SearchDestination(query: query))
        }
    }
}

struct ContentView: View {
    @State private var router = Router()

    var body: some View {
        NavigationStack(path: $router.path) {
            HomeView()
                .navigationDestination(for: ProductDestination.self) { dest in
                    ProductView(id: dest.id)
                }
                .navigationDestination(for: ProfileDestination.self) { dest in
                    ProfileView(username: dest.username)
                }
        }
        .environment(router)
        .onOpenURL { url in
            if let deepLink = DeepLink.from(url) {
                router.handle(deepLink)
            }
        }
    }
}
```

Rules:
- Place `.onOpenURL` on the root view so it catches all incoming URLs.
- Custom URL schemes are app-specific and not web-accessible — prefer universal links for content that has a web presence.
- URL schemes do not require entitlements or server configuration.
- Multiple schemes can be registered per app.

---

## 2. Universal Links

### apple-app-site-association (AASA) file

Host at `https://domain.com/.well-known/apple-app-site-association` (no file extension).

```json
{
    "applinks": {
        "details": [
            {
                "appIDs": ["ABCDE12345.com.example.myapp"],
                "components": [
                    { "/": "/product/*", "comment": "Product pages" },
                    { "/": "/profile/*", "comment": "User profiles" },
                    { "/": "/invite/*", "comment": "Invite links" },
                    { "/": "/app/*", "exclude": true, "comment": "Exclude app-only paths" }
                ]
            }
        ]
    }
}
```

Requirements:
| Requirement | Detail |
|---|---|
| Protocol | HTTPS only (no HTTP) |
| Content-Type | `application/json` |
| Location | `/.well-known/apple-app-site-association` |
| File extension | None (no `.json` suffix) |
| Signing | Not required (Apple CDN caches and validates) |
| Size limit | 128 KB maximum |

### Associated Domains entitlement

Add to your app's entitlements:
```xml
<key>com.apple.developer.associated-domains</key>
<array>
    <string>applinks:example.com</string>
    <string>applinks:www.example.com</string>
</array>
```

### Handling universal links in SwiftUI

```swift
struct ContentView: View {
    @State private var router = Router()

    var body: some View {
        NavigationStack(path: $router.path) {
            HomeView()
        }
        .onContinueUserActivity(NSUserActivityTypeBrowsingWeb) { activity in
            guard let url = activity.webpageURL else { return }
            if let deepLink = DeepLink.from(url) {
                router.handle(deepLink)
            }
        }
    }
}
```

### Debugging universal links

| Tool | Purpose |
|---|---|
| `swcutil dl -d example.com` | Query Apple CDN for cached AASA |
| Apple CDN validator | `https://app-site-association.cdn-apple.com/a/v1/example.com` |
| Console.app | Filter by `swcd` process for link resolution logs |
| Developer Settings | Associated Domains Development on device |

Rules:
- Both `.onOpenURL` and `.onContinueUserActivity` should be implemented — universal links use the activity path, URL schemes use `.onOpenURL`.
- AASA updates propagate via Apple CDN; allow 24-48 hours for changes.
- For development, add `?mode=developer` to the associated domain: `applinks:example.com?mode=developer`.

---

## 3. App Clips

### App Clip target setup

An App Clip is a lightweight version of your app (15 MB max uncompressed after thinning).

```swift
// App Clip entry point
@main
struct MyAppClip: App {
    var body: some Scene {
        WindowGroup {
            AppClipContentView()
                .appStoreOverlay(isPresented: $showFullAppOverlay) {
                    SKOverlay.AppClipConfiguration(position: .bottom)
                }
        }
    }
}
```

### Invocation sources

| Source | Setup |
|---|---|
| NFC tags | Encode App Clip experience URL |
| QR codes | Point to App Clip experience URL |
| Safari App Banner | `<meta name="apple-itunes-app" content="app-clip-bundle-id=...">` |
| Maps | Register in App Store Connect (place cards) |
| Messages | Shared links matching App Clip URL |
| App Clip Codes | Apple-designed visual codes |

### App Clip experience URL

Registered in App Store Connect. When a user scans/taps a matching URL, iOS launches the App Clip card showing metadata (title, subtitle, image, action button).

### SKOverlay for full app promotion

```swift
struct AppClipContentView: View {
    @State private var showOverlay = false

    var body: some View {
        VStack {
            // App Clip experience content
            OrderView()

            Button("Get Full App") {
                showOverlay = true
            }
        }
        .appStoreOverlay(isPresented: $showOverlay) {
            SKOverlay.AppClipConfiguration(position: .bottom)
        }
    }
}
```

### Data migration to full app

```swift
// Share data via App Groups
let sharedDefaults = UserDefaults(suiteName: "group.com.example.shared")
sharedDefaults?.set(userToken, forKey: "authToken")

// Share credentials via shared Keychain group
// Both App Clip and full app must use the same Keychain access group
```

### 8-hour ephemeral notification authorization

```swift
// App Clips get notification permission for 8 hours without prompting
let center = UNUserNotificationCenter.current()
let settings = await center.notificationSettings()
if settings.authorizationStatus == .ephemeral {
    // Can send notifications for 8 hours
}
```

Rules:
- 15 MB uncompressed size limit — measure via Xcode Archive > App Thinning Size Report.
- App Clips cannot access: HealthKit, CallKit, contacts, file system browsing, Bluetooth (background), motion data (persistent).
- App Clip data is deleted after a period of inactivity.
- Share code between App Clip and full app via shared frameworks or source files.

---

## 4. Sign in with Apple

### SwiftUI button

```swift
import AuthenticationServices

struct SignInView: View {
    @Environment(\.colorScheme) var colorScheme

    var body: some View {
        SignInWithAppleButton(.signIn) { request in
            request.requestedScopes = [.fullName, .email]
            request.nonce = generateNonce()  // For server-side validation
        } onCompletion: { result in
            switch result {
            case .success(let authorization):
                handleAuthorization(authorization)
            case .failure(let error):
                handleError(error)
            }
        }
        .signInWithAppleButtonStyle(
            colorScheme == .dark ? .white : .black
        )
        .frame(height: 50)
    }

    private func handleAuthorization(_ authorization: ASAuthorization) {
        guard let credential = authorization.credential
            as? ASAuthorizationAppleIDCredential else { return }

        let userID = credential.user                    // Stable, unique per app
        let fullName = credential.fullName              // Only on FIRST auth
        let email = credential.email                    // Only on FIRST auth
        let identityToken = credential.identityToken    // JWT for server validation
        let authCode = credential.authorizationCode     // Exchange for tokens server-side

        // Store userID in Keychain for credential state checks
        KeychainHelper.save(userID, forKey: "appleUserID")

        // Send identityToken + authCode to your server
        Task {
            await authenticateWithServer(
                userID: userID,
                identityToken: identityToken,
                authorizationCode: authCode,
                fullName: fullName,
                email: email
            )
        }
    }
}
```

### Credential state checking

```swift
func checkAppleIDCredentialState() async {
    guard let userID = KeychainHelper.read(forKey: "appleUserID") else {
        // No stored Apple ID — show sign-in
        return
    }

    let provider = ASAuthorizationAppleIDProvider()
    do {
        let state = try await provider.credentialState(forUserID: userID)
        switch state {
        case .authorized:
            break  // User is still signed in
        case .revoked:
            // User revoked — sign out, clear Keychain
            signOut()
        case .notFound:
            // No credential found — show sign-in
            signOut()
        case .transferred:
            // App transferred to new team — migrate user
            break
        @unknown default:
            break
        }
    } catch {
        // Handle error
    }
}
```

### Revocation handling

```swift
// Listen for revocation at app launch
NotificationCenter.default.addObserver(
    forName: ASAuthorizationAppleIDProvider.credentialRevokedNotification,
    object: nil,
    queue: .main
) { _ in
    signOut()
}
```

Rules:
- `fullName` and `email` are only provided on the FIRST authorization. Cache them server-side immediately.
- Store `credential.user` (the stable user identifier) in Keychain, never UserDefaults.
- Always check credential state at app launch.
- If user removes app from Apple ID settings, you receive `credentialRevokedNotification`.
- For server-side: validate the `identityToken` JWT against Apple's public keys (`https://appleid.apple.com/auth/keys`).

---

## 5. Passkeys (Platform Credentials)

### Registration (creating a passkey)

```swift
import AuthenticationServices

func registerPasskey() {
    let provider = ASAuthorizationPlatformPublicKeyCredentialProvider(
        relyingPartyIdentifier: "example.com"
    )

    let challenge = fetchChallengeFromServer()  // Server-generated
    let userID = Data("user-unique-id".utf8)

    let request = provider.createCredentialRegistrationRequest(
        challenge: challenge,
        name: "user@example.com",
        userID: userID
    )

    let controller = ASAuthorizationController(authorizationRequests: [request])
    controller.delegate = self
    controller.presentationContextProvider = self
    controller.performRequests()
}
```

### Authentication (signing in with a passkey)

```swift
func authenticateWithPasskey() {
    let provider = ASAuthorizationPlatformPublicKeyCredentialProvider(
        relyingPartyIdentifier: "example.com"
    )

    let challenge = fetchChallengeFromServer()

    let request = provider.createCredentialAssertionRequest(
        challenge: challenge
    )

    let controller = ASAuthorizationController(authorizationRequests: [request])
    controller.delegate = self
    controller.presentationContextProvider = self
    controller.performRequests()
}
```

### AutoFill-assisted passkey requests

```swift
func performAutoFillAssistedSignIn() {
    let provider = ASAuthorizationPlatformPublicKeyCredentialProvider(
        relyingPartyIdentifier: "example.com"
    )

    let challenge = fetchChallengeFromServer()
    let request = provider.createCredentialAssertionRequest(
        challenge: challenge
    )

    let controller = ASAuthorizationController(authorizationRequests: [request])
    controller.delegate = self
    controller.presentationContextProvider = self
    controller.performAutoFillAssistedRequests()  // Appears in keyboard suggestions
}
```

### Delegate handling

```swift
extension SignInController: ASAuthorizationControllerDelegate {
    func authorizationController(
        controller: ASAuthorizationController,
        didCompleteWithAuthorization authorization: ASAuthorization
    ) {
        switch authorization.credential {
        case let credential as ASAuthorizationPlatformPublicKeyCredentialRegistration:
            // Send credential.rawAttestationObject to server for verification
            let attestation = credential.rawAttestationObject
            let credentialID = credential.credentialID
            sendRegistrationToServer(attestation: attestation, credentialID: credentialID)

        case let credential as ASAuthorizationPlatformPublicKeyCredentialAssertion:
            // Send credential.rawAuthenticatorData + signature to server
            let authenticatorData = credential.rawAuthenticatorData
            let signature = credential.signature
            let credentialID = credential.credentialID
            sendAssertionToServer(
                authenticatorData: authenticatorData,
                signature: signature,
                credentialID: credentialID
            )

        default:
            break
        }
    }
}
```

### Associated Domains for passkeys

Add to entitlements:
```xml
<key>com.apple.developer.associated-domains</key>
<array>
    <string>webcredentials:example.com</string>
</array>
```

AASA file addition:
```json
{
    "webcredentials": {
        "apps": ["ABCDE12345.com.example.myapp"]
    }
}
```

Rules:
- Passkeys use the WebAuthn standard — your server must be a compliant relying party.
- `relyingPartyIdentifier` must match your domain and be declared in Associated Domains.
- Cross-device sign-in (QR code flow) works automatically when a nearby device has the passkey.
- Passkeys sync via iCloud Keychain across Apple devices.
- Combine with Sign in with Apple for a passwordless experience.

---

## 6. StoreKit 2 (In-App Purchases)

### Fetching products

```swift
import StoreKit

@Observable
final class StoreManager {
    private(set) var products: [Product] = []
    private(set) var purchasedProductIDs: Set<String> = []
    private var updateListenerTask: Task<Void, Never>?

    init() {
        updateListenerTask = listenForTransactionUpdates()
        Task { await loadProducts() }
        Task { await updatePurchasedProducts() }
    }

    deinit {
        updateListenerTask?.cancel()
    }

    func loadProducts() async {
        do {
            products = try await Product.products(for: [
                "com.app.premium.monthly",
                "com.app.premium.yearly",
                "com.app.coins.100"
            ])
        } catch {
            // Handle error
        }
    }
}
```

### Purchasing

```swift
extension StoreManager {
    func purchase(_ product: Product) async throws -> Transaction? {
        let result = try await product.purchase()

        switch result {
        case .success(let verification):
            let transaction = try checkVerified(verification)
            await updatePurchasedProducts()
            await transaction.finish()  // MUST call after delivering content
            return transaction

        case .pending:
            // Transaction awaiting approval (Ask to Buy, SCA)
            return nil

        case .userCancelled:
            return nil

        @unknown default:
            return nil
        }
    }

    private func checkVerified<T>(_ result: VerificationResult<T>) throws -> T {
        switch result {
        case .unverified(_, let error):
            throw error  // StoreKit could not verify the transaction
        case .verified(let safe):
            return safe
        }
    }
}
```

### Checking entitlements

```swift
extension StoreManager {
    func updatePurchasedProducts() async {
        var purchased: Set<String> = []
        for await result in Transaction.currentEntitlements {
            if let transaction = try? checkVerified(result) {
                purchased.insert(transaction.productID)
            }
        }
        purchasedProductIDs = purchased
    }

    func isPurchased(_ productID: String) -> Bool {
        purchasedProductIDs.contains(productID)
    }
}
```

### Transaction.updates listener

```swift
extension StoreManager {
    private func listenForTransactionUpdates() -> Task<Void, Never> {
        Task.detached {
            for await result in Transaction.updates {
                if let transaction = try? self.checkVerified(result) {
                    await self.updatePurchasedProducts()
                    await transaction.finish()
                }
            }
        }
    }
}
```

### Product types

| Type | Description | Example |
|---|---|---|
| `.consumable` | Used once, can repurchase | Coins, gems |
| `.nonConsumable` | Buy once, permanent | Remove ads, lifetime unlock |
| `.autoRenewable` | Recurring billing | Monthly/yearly premium |
| `.nonRenewing` | Time-limited, no auto-renew | Season pass |

### iOS 26 additions

- `appTransactionID` property on Transaction for unique transaction tracking.
- `originalPlatform` property indicates where the purchase was originally made.
- Offer codes now available for ALL product types (not just auto-renewable subscriptions).

Rules:
- MUST call `transaction.finish()` after delivering content — unfinished transactions retry and accumulate.
- MUST start `Transaction.updates` listener at app launch — catches external purchases, family sharing grants, subscription renewals.
- Always verify transactions using `VerificationResult` — never trust `.unverified`.
- Use StoreKit Configuration files (`.storekit`) for local testing without App Store Connect setup.
- Never use StoreKit 1 (original API) in new code.

---

## 7. Subscriptions & Paywalls

### SubscriptionStoreView (built-in paywall)

```swift
import StoreKit

struct PaywallView: View {
    var body: some View {
        SubscriptionStoreView(groupID: "A1B2C3D4") {
            // Marketing content above the subscription options
            VStack(spacing: 12) {
                Image(systemName: "star.fill")
                    .font(.largeTitle)
                    .foregroundStyle(.yellow)
                Text("Unlock Premium")
                    .font(.title.bold())
                Text("Get unlimited access to all features")
                    .foregroundStyle(.secondary)
            }
        }
        .subscriptionStoreControlStyle(.prominentPicker)
        .subscriptionStoreButtonLabel(.multiline)
        .storeButton(.visible, for: .restorePurchases)
        .subscriptionStorePolicyDestination(url: termsURL, for: .termsOfService)
        .subscriptionStorePolicyDestination(url: privacyURL, for: .privacyPolicy)
    }
}
```

### Subscription status checking

```swift
func checkSubscriptionStatus() async -> Bool {
    guard let statuses = try? await Product.SubscriptionInfo
        .status(for: "A1B2C3D4") else {
        return false
    }

    for status in statuses {
        guard let transaction = try? checkVerified(status.transaction) else {
            continue
        }

        switch status.state {
        case .subscribed, .inGracePeriod:
            return true
        case .revoked, .expired:
            continue
        default:
            continue
        }
    }
    return false
}
```

### Introductory and promotional offers

```swift
// Check if user is eligible for introductory offer (free trial)
if let offer = product.subscription?.introductoryOffer {
    let eligible = await product.subscription?
        .isEligibleForIntroOffer ?? false
    if eligible {
        // Show "Start Free Trial" UI
        // offer.period, offer.paymentMode (.freeTrial, .payAsYouGo, .payUpFront)
    }
}

// Promotional offer (server-signed, for win-back)
let promoOffer = Product.PurchaseOption.promotionalOffer(
    offerID: "winback_offer",
    keyID: "KEY_ID",
    nonce: UUID(),
    signature: serverSignature,
    timestamp: timestamp
)
let result = try await product.purchase(options: [promoOffer])
```

### Testing subscriptions

| Environment | Purpose |
|---|---|
| StoreKit Configuration file (`.storekit`) | Local testing, no App Store Connect needed |
| Sandbox | End-to-end testing with App Store Connect products |
| TestFlight | Beta testing with accelerated renewal periods |

Sandbox renewal periods:
| Actual Duration | Sandbox Duration |
|---|---|
| 1 week | 3 minutes |
| 1 month | 5 minutes |
| 1 year | 1 hour |

Rules:
- Use `SubscriptionStoreView` for standard paywall UI — consistent with Apple guidelines.
- Always show restore purchases button (`.storeButton(.visible, for: .restorePurchases)`).
- Include terms of service and privacy policy links.
- Test all subscription states: new purchase, renewal, grace period, billing retry, expiration, refund.
- Server Notifications v2 (App Store Server Notifications) should handle subscription lifecycle events server-side.

---

## 8. SharePlay / Group Activities

### Defining a GroupActivity

```swift
import GroupActivities

struct WatchTogetherActivity: GroupActivity {
    let movieID: String
    let movieTitle: String

    var metadata: GroupActivityMetadata {
        var meta = GroupActivityMetadata()
        meta.title = "Watch \(movieTitle)"
        meta.subtitle = "Watch together in real time"
        meta.type = .watchTogether
        return meta
    }
}
```

### Starting and managing a session

```swift
@Observable
final class SharePlayManager {
    private var groupSession: GroupSession<WatchTogetherActivity>?
    private var messenger: GroupSessionMessenger?
    private(set) var isSharePlaying = false

    func startSharePlay(movieID: String, title: String) async {
        let activity = WatchTogetherActivity(movieID: movieID, movieTitle: title)

        switch await activity.prepareForActivation() {
        case .activationPreferred:
            _ = try? await activity.activate()
        case .activationDisabled:
            break  // SharePlay not available
        case .cancelled:
            break
        @unknown default:
            break
        }
    }

    func configureSession(_ session: GroupSession<WatchTogetherActivity>) {
        groupSession = session
        let messenger = GroupSessionMessenger(session: session)
        self.messenger = messenger
        isSharePlaying = true

        session.join()

        // Listen for incoming messages
        Task {
            for await (message, _) in messenger.messages(of: SyncMessage.self) {
                handleIncomingMessage(message)
            }
        }

        // Listen for session state changes
        Task {
            for await state in session.$state.values {
                if case .invalidated = state {
                    groupSession = nil
                    self.messenger = nil
                    isSharePlaying = false
                }
            }
        }
    }
}
```

### Sending and receiving messages

```swift
struct SyncMessage: Codable {
    enum Action: Codable {
        case play(timestamp: TimeInterval)
        case pause(timestamp: TimeInterval)
        case seek(to: TimeInterval)
    }
    let action: Action
}

extension SharePlayManager {
    func sendPlayAction(at timestamp: TimeInterval) async {
        let message = SyncMessage(action: .play(timestamp: timestamp))
        try? await messenger?.send(message)
    }

    private func handleIncomingMessage(_ message: SyncMessage) {
        switch message.action {
        case .play(let timestamp):
            // Sync playback to timestamp
            break
        case .pause(let timestamp):
            // Pause at timestamp
            break
        case .seek(let position):
            // Seek to position
            break
        }
    }
}
```

### SwiftUI integration

```swift
struct MovieDetailView: View {
    let movie: Movie
    @State private var sharePlayManager = SharePlayManager()

    var body: some View {
        VideoPlayer(player: player)
            .task {
                for await session in WatchTogetherActivity.sessions() {
                    sharePlayManager.configureSession(session)
                }
            }
            .toolbar {
                ShareLink(
                    item: movie,
                    preview: SharePreview(movie.title)
                )
            }
    }
}
```

Rules:
- FaceTime provides built-in session discovery and management; standalone SharePlay requires custom session setup.
- `GroupSessionMessenger` sends Codable messages with reliable delivery by default; use `.unreliable` for high-frequency updates.
- Always iterate `Activity.sessions()` to handle incoming sessions.
- Coordinate media playback using `AVPlaybackCoordinator` for simpler video sync.
- The `.sharePlayActivity` modifier can present SharePlay UI inline.

---

## 9. Declared Age Range (NEW in iOS 26)

Parents can share a child's age range with apps through system settings. The app receives a privacy-preserving range (not the exact date of birth).

```swift
import DeclaredAgeRange

func adjustExperienceForAge() async {
    let ageRange = DeclaredAgeRange.current

    switch ageRange {
    case .under13:
        // Hide social features, disable in-app purchases
        // Apply strict content filtering
        break
    case .age13To16:
        // Moderate content filtering
        // Limit social features
        break
    case .age17Plus, .undeclared:
        // Full experience
        break
    @unknown default:
        // Default to safest experience
        break
    }
}
```

Use cases:
| Age Range | Adjustments |
|---|---|
| Under 13 | Disable purchases, hide social, strict content filter |
| 13-16 | Moderate filtering, limited social, parental controls visible |
| 17+ / Undeclared | Full experience |

Rules:
- This is a privacy-preserving signal — never request or store exact birth dates.
- Default to the safest experience if age range is `.undeclared`.
- Combine with App Store age rating for comprehensive age-appropriate design.

---

## 10. Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|---|---|---|
| Custom URL scheme for web content | Not verifiable, hijackable by other apps | Use universal links for web-accessible content |
| Store SIWA credentials in UserDefaults | Insecure, accessible to jailbroken device tools | Use Keychain Services |
| Expect fullName/email on repeat SIWA auth | Only provided on first authorization | Cache server-side on first auth |
| Use StoreKit 1 in new code | Deprecated, callback-based, harder to maintain | Use StoreKit 2 async/await API |
| Skip server-side receipt verification | Client-side verification can be bypassed | Always verify server-side for subscriptions |
| Omit Transaction.updates listener | Miss external purchases, family sharing, renewals | Start listener at app launch |
| Aggressive paywall before demonstrating value | Violates App Store Review Guidelines | Show value first, then offer subscription |
| Forget transaction.finish() | Unfinished transactions retry and accumulate | Always call finish() after content delivery |
| Hardcode product IDs | Cannot update products without app update | Fetch product IDs from your server |
| Exceed App Clip 15 MB budget | App Clip will not pass review | Measure with Xcode Archive thinning report |
| Skip credential state check at launch | User may have revoked Apple ID authorization | Check `getCredentialState(forUserID:)` on launch |
| Ignore AASA caching delays | Universal links may not work immediately | Allow 24-48 hours for Apple CDN propagation |
