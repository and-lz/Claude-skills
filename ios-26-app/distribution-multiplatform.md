# iOS 26 Distribution, CI/CD & Cross-Platform

Comprehensive reference for CI/CD pipelines, App Store distribution, companion platforms (CarPlay, watchOS, visionOS), Core Data migration, and Metal 4.

## Table of contents

1. [Xcode Cloud](#1-xcode-cloud)
2. [CI/CD patterns (non-Xcode Cloud)](#2-cicd-patterns-non-xcode-cloud)
3. [App Store Connect & ASO](#3-app-store-connect--aso)
4. [Core Data → SwiftData migration](#4-core-data--swiftdata-migration)
5. [CarPlay](#5-carplay)
6. [watchOS companion apps](#6-watchos-companion-apps)
7. [visionOS cross-platform](#7-visionos-cross-platform)
8. [Metal 4 (NEW in iOS 26)](#8-metal-4-new-in-ios-26)
9. [Anti-patterns](#9-anti-patterns)

---

## 1. Xcode Cloud

### Workflow setup

Workflows are configured in Xcode or App Store Connect:

| Component | Options |
|-----------|---------|
| **Start conditions** | Branch push, pull request, tag, schedule |
| **Actions** | Build, test, analyze, archive |
| **Post-actions** | Notify (Slack, email), deploy to TestFlight, deploy to App Store |

### Custom scripts

Place scripts in `ci_scripts/` at project root:

```bash
#!/bin/bash
# ci_scripts/ci_post_clone.sh
# Runs after repo clone — install deps, generate files

# Install tools
brew install xcodegen swiftlint

# Generate Xcode project
xcodegen generate

# Download secrets (from environment)
echo "$API_CONFIG" > Config/api_keys.plist
```

Available scripts:
- `ci_post_clone.sh` — after clone, before build
- `ci_pre_xcodebuild.sh` — before build starts
- `ci_post_xcodebuild.sh` — after build completes

### Environment variables

| Variable | Description |
|----------|-------------|
| `CI_WORKSPACE` | Path to workspace |
| `CI_PRODUCT` | Product name |
| `CI_BRANCH` | Current branch |
| `CI_COMMIT` | Current commit SHA |
| `CI_PULL_REQUEST_NUMBER` | PR number (if PR trigger) |
| `CI_TAG` | Tag name (if tag trigger) |

### Secrets

Store sensitive values in Xcode Cloud environment settings. Reference as `$SECRET_NAME` in scripts.

### Pricing

- 25 compute hours/month free with Apple Developer Program
- Additional hours: pay-per-use

### iOS 26

All App Store submissions must use Xcode 26 SDK by April 2026.

---

## 2. CI/CD patterns (non-Xcode Cloud)

### GitHub Actions

```yaml
name: iOS CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: macos-15
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_26.app

      - name: Build
        run: |
          xcodebuild -project App.xcodeproj \
            -scheme App \
            -destination 'platform=iOS Simulator,name=iPhone 16' \
            -configuration Debug \
            build 2>&1 | xcbeautify

      - name: Test
        run: |
          xcodebuild -project App.xcodeproj \
            -scheme App \
            -destination 'platform=iOS Simulator,name=iPhone 16' \
            test 2>&1 | xcbeautify
```

### Fastlane

```ruby
# Fastfile
default_platform(:ios)

platform :ios do
  desc "Run tests"
  lane :test do
    run_tests(scheme: "App")
  end

  desc "Build and upload to TestFlight"
  lane :beta do
    match(type: "appstore")          # Code signing
    gym(scheme: "App")               # Build .ipa
    pilot(skip_waiting_for_build_processing: true) # Upload
  end

  desc "Deploy to App Store"
  lane :release do
    match(type: "appstore")
    gym(scheme: "App")
    deliver(force: true)             # Upload metadata + binary
  end
end
```

### Code signing in CI

| Approach | Pros | Cons |
|----------|------|------|
| Fastlane match | Git-based, team sharing, encrypted | Requires Git repo for certs |
| Manual (P12 + provisioning) | Simple, no dependencies | Manual rotation |
| Xcode automatic | Zero config | Requires Apple ID in CI |

### Useful tools

- `xcbeautify` — readable xcodebuild output
- `xcresulttool` — extract data from .xcresult bundles
- `swift-snapshot-testing` — visual regression tests

---

## 3. App Store Connect & ASO

### Metadata limits

| Field | Limit |
|-------|-------|
| App name | 30 characters |
| Subtitle | 30 characters |
| Keywords | 100 characters (comma-separated) |
| Description | 4,000 characters |
| Promotional text | 170 characters (updatable without review) |
| What's New | 4,000 characters |

### Screenshot requirements

| Device | Size (points) | Scale |
|--------|---------------|-------|
| iPhone 6.7" (16 Pro Max) | 430×932 | @3x |
| iPhone 6.5" (legacy) | 414×896 | @3x |
| iPhone 5.5" (legacy) | 414×736 | @3x |
| iPad Pro 12.9" | 1024×1366 | @2x |
| iPad Pro 11" | 834×1194 | @2x |

- Up to 10 screenshots per device size
- First 3 screenshots most impactful (visible before "See More")

### Common App Review rejections

| Guideline | Issue | Fix |
|-----------|-------|-----|
| **4.0 Design** | Crashes, broken features, placeholder content | Test thoroughly on real devices |
| **2.1 Performance** | Incomplete functionality, demo modes | Ship complete features only |
| **5.1.1 Privacy** | Missing purpose strings, undisclosed data collection | Add all Info.plist strings, complete privacy manifest |
| **3.1.1 IAP** | Digital goods not using Apple IAP | Use StoreKit for all digital content |
| **4.2 Minimum Functionality** | App is too simple or web wrapper | Add native value beyond website |
| **1.2 User-Generated Content** | No reporting/blocking mechanism | Add content moderation features |

### Distribution strategies

- **Phased release**: 1% → 2% → 5% → 10% → 20% → 50% → 100% over 7 days
- **Custom product pages**: up to 35 variants for different audiences/campaigns
- **A/B testing**: test screenshots, icons, descriptions
- **In-app events**: time-limited events shown on App Store (challenges, launches)

### iOS 26 changes

- **Mobile TestFlight feedback**: review screenshots, crash logs on iPhone/iPad via App Store Connect app
- **Apple-hosted background assets**: files up to 200GB, independent of app updates
- **100+ new subscription/monetization metrics**
- **Failed builds retained** with error/warning details
- **Accessibility Nutrition Labels** (new)
- Privacy Nutrition Labels (still required)

---

## 4. Core Data → SwiftData migration

### When to migrate

| Strategy | When to use |
|----------|-------------|
| **New features only** | Add SwiftData for new models, keep Core Data for existing |
| **Incremental** | Migrate model-by-model over releases |
| **Full migration** | Rewrite persistence layer (major version) |

### Side-by-side coexistence

Both can run simultaneously against separate or shared stores:

```swift
// SwiftData container
let modelContainer = try ModelContainer(for: NewFeatureModel.self)

// Core Data stack (existing)
let persistentContainer = NSPersistentContainer(name: "LegacyModel")
persistentContainer.loadPersistentStores { _, error in }
```

### Entity-to-@Model mapping

```swift
// Core Data entity (before)
class CDTask: NSManagedObject {
    @NSManaged var title: String
    @NSManaged var isCompleted: Bool
    @NSManaged var createdAt: Date
    @NSManaged var project: CDProject?
}

// SwiftData model (after)
@Model
class SDTask {
    var title: String
    var isCompleted: Bool
    var createdAt: Date
    @Relationship var project: SDProject?

    init(title: String, isCompleted: Bool = false, createdAt: Date = .now) {
        self.title = title
        self.isCompleted = isCompleted
        self.createdAt = createdAt
    }
}
```

### Schema migration

```swift
enum TaskSchemaV1: VersionedSchema {
    static var versionIdentifier = Schema.Version(1, 0, 0)
    static var models: [any PersistentModel.Type] { [SDTask.self] }

    @Model class SDTask {
        var title: String
        var isCompleted: Bool
    }
}

enum TaskSchemaV2: VersionedSchema {
    static var versionIdentifier = Schema.Version(2, 0, 0)
    static var models: [any PersistentModel.Type] { [SDTask.self] }

    @Model class SDTask {
        var title: String
        var isCompleted: Bool
        var priority: Int = 0 // new field
    }
}

enum TaskMigrationPlan: SchemaMigrationPlan {
    static var schemas: [any VersionedSchema.Type] { [TaskSchemaV1.self, TaskSchemaV2.self] }
    static var stages: [MigrationStage] {
        [.lightweight(fromVersion: TaskSchemaV1.self, toVersion: TaskSchemaV2.self)]
    }
}
```

### iOS 26

**SwiftData model inheritance** — `@Model` classes can form inheritance hierarchies, enabling shared properties and polymorphic queries.

### Migration checklist

1. Map all Core Data entities to @Model classes
2. Test with copy of production database
3. Verify data integrity after migration
4. Keep Core Data stack as rollback until validated
5. Handle both stores during transition period

---

## 5. CarPlay

### Entitlement

Requires `com.apple.developer.carplay-*` capability — must apply to Apple per category.

### Categories

| Category | Template types | Approval |
|----------|---------------|----------|
| Navigation | CPMapTemplate | Required |
| Audio | CPNowPlayingTemplate, CPListTemplate | Required |
| Communication | CPListTemplate, CPContactTemplate | Required |
| EV charging | CPPointOfInterestTemplate | Required |
| Parking | CPPointOfInterestTemplate | Required |
| Quick food ordering | CPListTemplate | Required |
| Driving task | CPListTemplate, CPGridTemplate | Required |

### Template types

```swift
// Tab-based root navigation
let tabBar = CPTabBarTemplate(templates: [listTemplate, mapTemplate])

// List template
let item = CPListItem(text: "Song Title", detailText: "Artist")
item.handler = { item, completion in
    playSong()
    completion()
}
let section = CPListSection(items: [item])
let listTemplate = CPListTemplate(title: "Library", sections: [section])
```

### Scene setup

```swift
class CarPlaySceneDelegate: UIResponder, CPTemplateApplicationSceneDelegate {
    var interfaceController: CPInterfaceController?

    func templateApplicationScene(_ scene: CPTemplateApplicationScene,
                                   didConnect interfaceController: CPInterfaceController) {
        self.interfaceController = interfaceController
        interfaceController.setRootTemplate(tabBarTemplate, animated: true)
    }
}
```

### iOS 26 changes

- **Liquid Glass** design throughout CarPlay
- **Widgets and Live Activities** on CarPlay Dashboard — auto-show when started on iPhone
- **Multi-touch gesture** support for mapping apps
- **Smart Display Zoom**: configurable display scale
- **List template**: customizable image rows, disable elements, multi-line support

---

## 6. watchOS companion apps

### WatchConnectivity

```swift
import WatchConnectivity

class ConnectivityManager: NSObject, WCSessionDelegate {
    static let shared = ConnectivityManager()

    func activate() {
        guard WCSession.isSupported() else { return }
        WCSession.default.delegate = self
        WCSession.default.activate()
    }

    // Real-time (requires reachability)
    func sendMessage(_ data: [String: Any]) {
        guard WCSession.default.isReachable else {
            WCSession.default.transferUserInfo(data)
            return
        }
        WCSession.default.sendMessage(data, replyHandler: nil)
    }

    // Background transfer (queued, FIFO)
    func transferData(_ data: [String: Any]) {
        WCSession.default.transferUserInfo(data)
    }

    // Latest-only (replaces previous)
    func updateContext(_ data: [String: Any]) {
        try? WCSession.default.updateApplicationContext(data)
    }

    // Required delegate methods
    func session(_ session: WCSession, activationDidCompleteWith state: WCSessionActivationState, error: Error?) {}
    func sessionDidBecomeInactive(_ session: WCSession) {}
    func sessionDidDeactivate(_ session: WCSession) { WCSession.default.activate() }
}
```

### Communication patterns

| Method | Delivery | When |
|--------|----------|------|
| `sendMessage` | Immediate | Both apps active, reachable |
| `transferUserInfo` | Background, queued | Guaranteed delivery, FIFO order |
| `updateApplicationContext` | Background, latest-only | Only most recent value matters |
| `transferFile` | Background | Large files, images |
| `transferCurrentComplicationUserInfo` | High priority | Complication updates |

### watchOS SwiftUI specifics

- `NavigationStack` — same as iOS
- `TabView` — vertical paging by default on watchOS
- `.digitalCrownRotation($value)` — Digital Crown integration
- `.focusable()` — enable Digital Crown interaction

### iOS 26 watchOS changes

- **Liquid Glass** throughout (Smart Stack, Control Center, Photos face)
- **Notes app** on Apple Watch
- **Workout Buddy**: AI-powered spoken motivation
- **Control Widget API**: custom controls for Control Center, Action Button, Smart Stack
- **MapKit on watchOS**: search nearby POIs, get routes, display route overlays
- **RelevanceKit**: Smart Stack widget surfacing based on routine/location
- **Widget push updates** via APNs

---

## 7. visionOS cross-platform

### Conditional compilation

```swift
#if os(visionOS)
import RealityKit
// visionOS-specific code
#elseif os(iOS)
import ARKit
// iOS-specific code
#endif
```

### Compatibility modes

| Mode | Description | Code changes |
|------|-------------|-------------|
| **Compatible** | iPhone/iPad app in flat window | Zero — automatic |
| **Designed for visionOS** | Optimized spatial experience | Moderate — add spatial features |

### Scene types

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView() // 2D window
        }

        #if os(visionOS)
        ImmersiveSpace(id: "immersive") {
            ImmersiveView() // Full 3D environment
        }

        WindowGroup(id: "volume", for: UUID.self) { $id in
            VolumeView() // Bounded 3D content
        }
        .windowStyle(.volumetric)
        #endif
    }
}
```

### Spatial layout

- Depth: `.offset(z: 50)` — push content forward/back
- Ornaments: `.ornament(attachmentAnchor: .scene(.bottom)) { controls }`
- Hover effects: `.hoverEffect()` — highlight on gaze

### Input differences

| iOS | visionOS |
|-----|----------|
| Touch/tap | Eye tracking + hand pinch |
| Swipe gestures | Spatial tap gesture |
| Long press | Long gaze + pinch |
| Pinch to zoom | Not applicable (use resize) |

### Shared code strategy

- Abstract platform-specific code behind protocols
- Use `#if os()` for minimal divergences
- Share view models and data layer entirely
- Split only the view layer where needed

### iOS 26 / visionOS 26 changes

- **Spatial widgets**: snap to walls/tables, blend into environment
- **90Hz hand tracking** for immersive apps (no code changes)
- **Direct ARKit data access**: anchoring, collision, physics for real-world objects
- **Widgets now available** in visionOS apps (compatible iPhone/iPad widgets auto-available)
- **PlayStation VR2 Sense controller** support

---

## 8. Metal 4 (NEW in iOS 26)

### Overview

Complete new GPU framework alongside Metal 3 (coexist, not replace). Designed for modern GPU architectures with explicit control.

### Key classes

| Class | Purpose |
|-------|---------|
| `MTL4CommandAllocator` | Direct control of command buffer memory |
| `MTL4RenderCommandEncoder` | Attachment map for logical→physical color attachments |
| `MTLTensor` | New resource type for ML workflows |
| `MTL4MachineLearningCommandEncoder` | Run ML networks on GPU timeline alongside draws |

### Features

- **Native parallel encoding** — multiple threads encode to same command buffer
- **Residency sets** — specify resources to make GPU-resident
- **Placement sparse resources** — efficient streaming of large assets
- **Barriers** — explicit synchronization between passes
- **MetalFX Frame Interpolation** — temporal upscaling for higher framerates
- **Ray tracing denoiser** — built-in noise reduction

### ML integration

```swift
// Run ML inference alongside rendering on GPU timeline
let mlEncoder = commandBuffer.makeMachineLearningCommandEncoder()
mlEncoder?.encode(neuralNetwork, input: inputTensor, output: outputTensor)
mlEncoder?.endEncoding()
```

### Requirements

- A14 Bionic or newer (iPhone 12+)
- M1 or newer (iPad Pro/Air, Mac)

---

## 9. Anti-patterns

| Anti-pattern | Why | Instead |
|-------------|-----|---------|
| Skip migration testing on prod-size data | Lightweight migration can fail on edge cases | Test with copy of real production database |
| Hardcode CI secrets in scripts/source | Exposed in logs, version control | Use environment variables / secret management |
| Ignore App Store review guidelines | Rejection delays launch by days/weeks | Read 4.0, 2.1, 5.1.1, 3.1.1 before submitting |
| Ship CarPlay without physical testing | Simulator misses timing, layout, interaction issues | Test on CarPlay-equipped vehicle or aftermarket unit |
| Create watch app without clear value | Apple rejects trivial/empty watch apps | Identify specific wrist-only use case first |
| Assume visionOS = ARKit on iOS | Volumetric vs screen-based, different input model | Study spatial design guidelines separately |
| Use `sendMessage` without checking reachability | Silent failure when counterpart app not active | Check `isReachable`, fall back to `transferUserInfo` |
| Build "Designed for visionOS" without testing | Spatial interactions feel wrong without iteration | Use visionOS Simulator or device extensively |
| Force-push to main in CI | Overwrites team members' work | Use merge commits or rebase with care |
| Skip phased release for major updates | No rollback if critical bug ships | Use 1%→100% over 7 days, monitor crash rate |
