# Notifications, WidgetKit, Live Activities, BackgroundTasks, and TipKit

## Table of contents
1. Push Notifications (APNs)
2. Local Notifications
3. Notification Extensions
4. WidgetKit
5. Live Activities & Dynamic Island
6. BackgroundTasks Framework
7. RelevanceKit (NEW in iOS 26)
8. TipKit (Feature Discovery)
9. Anti-Patterns

---

## 1. Push Notifications (APNs)

### Requesting authorization

```swift
import UserNotifications

let center = UNUserNotificationCenter.current()
let granted = try await center.requestAuthorization(options: [.alert, .badge, .sound])
```

Available options:

| Option | Purpose |
|--------|---------|
| `.alert` | Display banner/lock screen alerts |
| `.badge` | Update app icon badge number |
| `.sound` | Play notification sound |
| `.criticalAlert` | Bypass Do Not Disturb (requires Apple entitlement) |
| `.provisional` | Deliver quietly without prompting — appears in notification center only |
| `.providesAppNotificationSettings` | Show "Configure in App" button in system Settings |

### Provisional authorization

Use `.provisional` to deliver notifications silently without an upfront permission prompt. The user sees them in the notification center and can promote or dismiss.

```swift
let granted = try await center.requestAuthorization(options: [.alert, .badge, .sound, .provisional])
```

### Device token registration

In SwiftUI, use `UIApplicationDelegateAdaptor` to receive the device token.

```swift
@main
struct MyApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) var appDelegate

    var body: some Scene {
        WindowGroup { ContentView() }
    }
}

class AppDelegate: NSObject, UIApplicationDelegate {
    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]? = nil
    ) -> Bool {
        UNUserNotificationCenter.current().delegate = self
        application.registerForRemoteNotifications()
        return true
    }

    func application(
        _ application: UIApplication,
        didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data
    ) {
        let token = deviceToken.map { String(format: "%02.2hhx", $0) }.joined()
        // Send token to your server
        print("APNs token: \(token)")
    }

    func application(
        _ application: UIApplication,
        didFailToRegisterForRemoteNotificationsWithError error: Error
    ) {
        print("Failed to register: \(error.localizedDescription)")
    }
}
```

### APNs payload structure

```json
{
    "aps": {
        "alert": {
            "title": "New Message",
            "subtitle": "From John",
            "body": "Hey, are you free tonight?"
        },
        "badge": 3,
        "sound": "default",
        "content-available": 1,
        "mutable-content": 1,
        "category": "MESSAGE_CATEGORY",
        "thread-id": "conversation-42"
    },
    "custom-key": "custom-value"
}
```

| Field | Purpose |
|-------|---------|
| `content-available` | Triggers silent/background notification (set to 1) |
| `mutable-content` | Allows Notification Service Extension to modify content (set to 1) |
| `category` | Maps to UNNotificationCategory for action buttons |
| `thread-id` | Groups notifications visually in notification center |

### Token-based authentication (.p8 key)

Recommended over certificate-based authentication. A single .p8 key works for all your apps and never expires.

1. Generate in Apple Developer portal: Certificates, Identifiers & Profiles > Keys > Add
2. Download the .p8 file (available only once)
3. Use Key ID + Team ID + .p8 to sign JWT tokens server-side
4. Send to `api.push.apple.com` (production) or `api.sandbox.push.apple.com`

### Categories and actions

```swift
let replyAction = UNTextInputNotificationAction(
    identifier: "REPLY_ACTION",
    title: "Reply",
    options: [],
    textInputButtonTitle: "Send",
    textInputPlaceholder: "Type a message..."
)

let deleteAction = UNNotificationAction(
    identifier: "DELETE_ACTION",
    title: "Delete",
    options: [.destructive, .authenticationRequired]
)

let messageCategory = UNNotificationCategory(
    identifier: "MESSAGE_CATEGORY",
    actions: [replyAction, deleteAction],
    intentIdentifiers: [],
    options: [.customDismissAction]
)

center.setNotificationCategories([messageCategory])
```

### Handling notifications

```swift
extension AppDelegate: UNUserNotificationCenterDelegate {
    // Called when user taps a notification
    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        didReceive response: UNNotificationResponse
    ) async {
        let userInfo = response.notification.request.content.userInfo

        switch response.actionIdentifier {
        case "REPLY_ACTION":
            if let textResponse = response as? UNTextInputNotificationResponse {
                print("User replied: \(textResponse.userText)")
            }
        case "DELETE_ACTION":
            print("User chose delete")
        case UNNotificationDefaultActionIdentifier:
            print("User tapped the notification body")
        case UNNotificationDismissActionIdentifier:
            print("User dismissed (requires .customDismissAction)")
        default:
            break
        }
    }

    // Called when notification arrives while app is in foreground
    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        willPresent notification: UNNotification
    ) async -> UNNotificationPresentationOptions {
        return [.banner, .badge, .sound]
    }
}
```

---

## 2. Local Notifications

### Building notification content

```swift
let content = UNMutableNotificationContent()
content.title = "Daily Reminder"
content.body = "Time to review your photos!"
content.sound = .default
content.badge = 1
content.categoryIdentifier = "REMINDER_CATEGORY"
content.threadIdentifier = "daily-reminders"

// Attach an image
if let imageURL = Bundle.main.url(forResource: "reminder", withExtension: "png") {
    let attachment = try UNNotificationAttachment(identifier: "image", url: imageURL)
    content.attachments = [attachment]
}
```

### Triggers

| Trigger | Use case | Example |
|---------|----------|---------|
| `UNTimeIntervalNotificationTrigger` | Fire after elapsed time | 60 seconds from now |
| `UNCalendarNotificationTrigger` | Fire at specific date/time | Every day at 9 AM |
| `UNLocationNotificationTrigger` | Fire on region entry/exit | Arriving at the office |

```swift
// After 60 seconds (non-repeating)
let timeTrigger = UNTimeIntervalNotificationTrigger(timeInterval: 60, repeats: false)

// Daily at 9:00 AM
var dateComponents = DateComponents()
dateComponents.hour = 9
dateComponents.minute = 0
let calendarTrigger = UNCalendarNotificationTrigger(dateMatching: dateComponents, repeats: true)

// Location-based
let center = CLLocationCoordinate2D(latitude: 37.3346, longitude: -122.0090)
let region = CLCircularRegion(center: center, radius: 100, identifier: "apple-park")
region.notifyOnEntry = true
region.notifyOnExit = false
let locationTrigger = UNLocationNotificationTrigger(region: region, repeats: false)
```

### Scheduling

```swift
let request = UNNotificationRequest(
    identifier: "daily-reminder-9am",
    content: content,
    trigger: calendarTrigger
)

try await UNUserNotificationCenter.current().add(request)
```

### Managing pending notifications

```swift
let center = UNUserNotificationCenter.current()

// List all pending
let pending = await center.pendingNotificationRequests()

// Remove specific
center.removePendingNotificationRequests(withIdentifiers: ["daily-reminder-9am"])

// Remove all
center.removeAllPendingNotificationRequests()

// Remove delivered notifications from notification center
center.removeDeliveredNotifications(withIdentifiers: ["daily-reminder-9am"])
```

### Grouped notifications

Set `threadIdentifier` on content to group notifications visually. The system collapses groups automatically.

```swift
content.threadIdentifier = "conversation-42"  // All messages in this thread group together
```

### Complete example: daily reminder

```swift
func scheduleDailyReminder() async throws {
    let center = UNUserNotificationCenter.current()
    let granted = try await center.requestAuthorization(options: [.alert, .sound])
    guard granted else { return }

    let content = UNMutableNotificationContent()
    content.title = "Photo Review"
    content.body = "You have unreviewed photos. Swipe through a few!"
    content.sound = .default

    var components = DateComponents()
    components.hour = 19  // 7 PM daily
    components.minute = 0

    let trigger = UNCalendarNotificationTrigger(dateMatching: components, repeats: true)
    let request = UNNotificationRequest(
        identifier: "daily-photo-reminder",
        content: content,
        trigger: trigger
    )

    try await center.add(request)
}
```

---

## 3. Notification Extensions

### Notification Service Extension

Intercepts and modifies push notification content before display. Requires `mutable-content: 1` in the APNs payload.

Use cases:
- Download and attach images/media to the notification
- Decrypt end-to-end encrypted payloads
- Augment content with local data

```swift
// NotificationService.swift (in Service Extension target)
class NotificationService: UNNotificationServiceExtension {
    var contentHandler: ((UNNotificationContent) -> Void)?
    var bestAttemptContent: UNMutableNotificationContent?

    override func didReceive(
        _ request: UNNotificationRequest,
        withContentHandler contentHandler: @escaping (UNNotificationContent) -> Void
    ) {
        self.contentHandler = contentHandler
        bestAttemptContent = request.content.mutableCopy() as? UNMutableNotificationContent

        guard let content = bestAttemptContent,
              let imageURLString = content.userInfo["image-url"] as? String,
              let imageURL = URL(string: imageURLString) else {
            contentHandler(request.content)
            return
        }

        // Download image and attach
        let task = URLSession.shared.downloadTask(with: imageURL) { url, _, error in
            defer { contentHandler(content) }
            guard let url, error == nil else { return }

            let dest = FileManager.default.temporaryDirectory
                .appendingPathComponent(UUID().uuidString + ".jpg")
            try? FileManager.default.moveItem(at: url, to: dest)

            if let attachment = try? UNNotificationAttachment(identifier: "image", url: dest) {
                content.attachments = [attachment]
            }
        }
        task.resume()
    }

    // Called if didReceive takes longer than 30 seconds
    override func serviceExtensionTimeWillExpire() {
        if let contentHandler, let content = bestAttemptContent {
            content.title += " (loading media...)"
            contentHandler(content)
        }
    }
}
```

### Notification Content Extension

Provides a custom UI for the expanded notification view. The user sees it when long-pressing or 3D-touching a notification.

- Create a Notification Content Extension target
- Implement a UIViewController conforming to `UNNotificationContentExtension`
- Declare the category in the extension's Info.plist under `UNNotificationExtensionCategory`

```swift
class NotificationViewController: UIViewController, UNNotificationContentExtension {
    @IBOutlet weak var imageView: UIImageView!
    @IBOutlet weak var detailLabel: UILabel!

    func didReceive(_ notification: UNNotification) {
        let content = notification.request.content
        detailLabel.text = content.body

        if let attachment = content.attachments.first,
           attachment.url.startAccessingSecurityScopedResource() {
            defer { attachment.url.stopAccessingSecurityScopedResource() }
            if let data = try? Data(contentsOf: attachment.url) {
                imageView.image = UIImage(data: data)
            }
        }
    }
}
```

### Communication notifications

For messaging apps, use `SenderImageProvider` to display contact photos in notifications. Requires the Communication Notifications capability and integration with `INSendMessageIntent`.

### Live Activity push updates

Live Activities can be updated remotely via APNs using ActivityKit push tokens. See Section 5.

---

## 4. WidgetKit

### Basic widget structure

```swift
import WidgetKit
import SwiftUI

struct PhotoStatsEntry: TimelineEntry {
    let date: Date
    let reviewedCount: Int
    let deletedCount: Int
}

struct PhotoStatsProvider: TimelineProvider {
    func placeholder(in context: Context) -> PhotoStatsEntry {
        PhotoStatsEntry(date: .now, reviewedCount: 0, deletedCount: 0)
    }

    func getSnapshot(in context: Context, completion: @escaping (PhotoStatsEntry) -> Void) {
        let entry = PhotoStatsEntry(date: .now, reviewedCount: 42, deletedCount: 15)
        completion(entry)
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<PhotoStatsEntry>) -> Void) {
        // Read from shared App Group storage
        let defaults = UserDefaults(suiteName: "group.com.app.swipetoclear")
        let reviewed = defaults?.integer(forKey: "reviewedCount") ?? 0
        let deleted = defaults?.integer(forKey: "deletedCount") ?? 0

        let entry = PhotoStatsEntry(date: .now, reviewedCount: reviewed, deletedCount: deleted)
        let nextUpdate = Calendar.current.date(byAdding: .hour, value: 1, to: .now)!
        let timeline = Timeline(entries: [entry], policy: .after(nextUpdate))
        completion(timeline)
    }
}

struct PhotoStatsWidgetView: View {
    var entry: PhotoStatsEntry

    var body: some View {
        VStack(alignment: .leading) {
            Text("Photo Review")
                .font(.headline)
            Text("\(entry.reviewedCount) reviewed")
            Text("\(entry.deletedCount) deleted")
                .foregroundStyle(.secondary)
        }
        .containerBackground(.fill.tertiary, for: .widget)
    }
}

struct PhotoStatsWidget: Widget {
    let kind = "PhotoStatsWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: PhotoStatsProvider()) { entry in
            PhotoStatsWidgetView(entry: entry)
        }
        .configurationDisplayName("Photo Stats")
        .description("See your review progress.")
        .supportedFamilies([.systemSmall, .systemMedium, .accessoryRectangular])
    }
}
```

### Widget families

| Family | Size | Platform |
|--------|------|----------|
| `.systemSmall` | Square, tap-only | iOS, iPadOS |
| `.systemMedium` | Wide rectangle | iOS, iPadOS |
| `.systemLarge` | Tall rectangle | iOS, iPadOS |
| `.systemExtraLarge` | Extra wide | iPadOS only |
| `.accessoryCircular` | Small circle | Lock Screen, Watch |
| `.accessoryRectangular` | Small rectangle | Lock Screen, Watch |
| `.accessoryInline` | Single line of text | Lock Screen, Watch |

### Timeline reload policies

| Policy | Behavior |
|--------|----------|
| `.atEnd` | Reload after the last entry's date passes |
| `.after(date)` | Reload after a specific date |
| `.never` | Only reload when explicitly requested |

### Shared data via App Groups

Both the app and widget extension must be in the same App Group.

```swift
// In the main app — write data
let defaults = UserDefaults(suiteName: "group.com.app.swipetoclear")
defaults?.set(reviewedCount, forKey: "reviewedCount")

// Trigger widget reload
WidgetCenter.shared.reloadTimelines(ofKind: "PhotoStatsWidget")
```

### Interactive widgets (iOS 17+)

`Button` and `Toggle` work inside widget views. Actions are dispatched via `AppIntent`.

```swift
struct MarkReviewedIntent: AppIntent {
    static var title: LocalizedStringResource = "Mark as Reviewed"

    func perform() async throws -> some IntentResult {
        // Update shared data
        return .result()
    }
}

struct InteractiveWidgetView: View {
    var entry: PhotoStatsEntry

    var body: some View {
        Button(intent: MarkReviewedIntent()) {
            Label("Review", systemImage: "checkmark.circle")
        }
        .containerBackground(.fill.tertiary, for: .widget)
    }
}
```

### AppIntentConfiguration (iOS 17+, replaces IntentConfiguration)

```swift
struct SelectAlbumIntent: WidgetConfigurationIntent {
    static var title: LocalizedStringResource = "Select Album"

    @Parameter(title: "Album")
    var albumName: String?
}

struct ConfigurableWidget: Widget {
    var body: some WidgetConfiguration {
        AppIntentConfiguration(
            kind: "ConfigurableWidget",
            intent: SelectAlbumIntent.self,
            provider: ConfigurableProvider()
        ) { entry in
            ConfigurableWidgetView(entry: entry)
        }
        .configurationDisplayName("Album Stats")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}
```

### iOS 26 changes

- **Clear glass presentation**: Widgets on the home screen render with the Liquid Glass material. Use `.containerBackground` and avoid opaque backgrounds.
- **Custom tint color**: Apply accent tinting to widget glass appearance.
- **CarPlay widgets**: WidgetKit now supports CarPlay — add the CarPlay widget family to `supportedFamilies`.
- **visionOS 26**: Compatible iPhone/iPad widgets auto-appear on visionOS without modification.
- **APNs push updates on all platforms**: Widgets can be refreshed via push notifications on iOS, watchOS, macOS, and visionOS.

---

## 5. Live Activities & Dynamic Island

### Define ActivityAttributes

```swift
import ActivityKit

struct PhotoReviewAttributes: ActivityAttributes {
    // Static data — set once when activity starts
    let albumName: String
    let totalPhotos: Int

    // Dynamic data — updated throughout the activity
    struct ContentState: Codable, Hashable {
        let reviewedCount: Int
        let deletedCount: Int
        let currentPhotoIndex: Int
    }
}
```

### Starting a Live Activity

```swift
func startReviewActivity(albumName: String, totalPhotos: Int) throws -> Activity<PhotoReviewAttributes> {
    let attributes = PhotoReviewAttributes(
        albumName: albumName,
        totalPhotos: totalPhotos
    )

    let initialState = PhotoReviewAttributes.ContentState(
        reviewedCount: 0,
        deletedCount: 0,
        currentPhotoIndex: 0
    )

    let content = ActivityContent(
        state: initialState,
        staleDate: Calendar.current.date(byAdding: .minute, value: 30, to: .now)
    )

    return try Activity.request(
        attributes: attributes,
        content: content,
        pushType: .token  // Enable remote push updates
    )
}
```

### Updating a Live Activity

```swift
func updateReviewActivity(_ activity: Activity<PhotoReviewAttributes>, reviewed: Int, deleted: Int, index: Int) async {
    let updatedState = PhotoReviewAttributes.ContentState(
        reviewedCount: reviewed,
        deletedCount: deleted,
        currentPhotoIndex: index
    )

    let content = ActivityContent(
        state: updatedState,
        staleDate: Calendar.current.date(byAdding: .minute, value: 30, to: .now)
    )

    await activity.update(content)
}
```

### Ending a Live Activity

```swift
func endReviewActivity(_ activity: Activity<PhotoReviewAttributes>, reviewed: Int, deleted: Int) async {
    let finalState = PhotoReviewAttributes.ContentState(
        reviewedCount: reviewed,
        deletedCount: deleted,
        currentPhotoIndex: 0
    )

    let content = ActivityContent(state: finalState, staleDate: nil)

    await activity.end(content, dismissalPolicy: .default)
    // .default — stays briefly on Lock Screen, then dismisses
    // .immediate — removes instantly
    // .after(date) — stays until specified date
}
```

### Push token for remote updates

```swift
func observePushToken(_ activity: Activity<PhotoReviewAttributes>) {
    Task {
        for await token in activity.pushTokenUpdates {
            let tokenString = token.map { String(format: "%02x", $0) }.joined()
            // Send tokenString to your server for remote updates
        }
    }
}
```

### Dynamic Island layouts

```swift
struct PhotoReviewLiveActivity: Widget {
    var body: some WidgetConfiguration {
        ActivityConfiguration(for: PhotoReviewAttributes.self) { context in
            // Lock Screen / banner presentation
            HStack {
                VStack(alignment: .leading) {
                    Text(context.attributes.albumName)
                        .font(.headline)
                    Text("\(context.state.reviewedCount)/\(context.attributes.totalPhotos) reviewed")
                }
                Spacer()
                Text("\(context.state.deletedCount) deleted")
                    .foregroundStyle(.red)
            }
            .padding()
            .activityBackgroundTint(.black.opacity(0.8))

        } dynamicIsland: { context in
            DynamicIsland {
                // Expanded regions (long press)
                DynamicIslandExpandedRegion(.leading) {
                    Label(context.attributes.albumName, systemImage: "photo.on.rectangle")
                }
                DynamicIslandExpandedRegion(.trailing) {
                    Text("\(context.state.deletedCount)")
                        .font(.title2)
                        .foregroundStyle(.red)
                }
                DynamicIslandExpandedRegion(.center) {
                    Text("Reviewing Photos")
                        .font(.caption)
                }
                DynamicIslandExpandedRegion(.bottom) {
                    ProgressView(
                        value: Double(context.state.reviewedCount),
                        total: Double(context.attributes.totalPhotos)
                    )
                }
            } compactLeading: {
                // Always-visible left pill
                Image(systemName: "photo.circle.fill")
            } compactTrailing: {
                // Always-visible right pill
                Text("\(context.state.reviewedCount)/\(context.attributes.totalPhotos)")
                    .font(.caption2)
            } minimal: {
                // When multiple activities compete
                Image(systemName: "photo.circle.fill")
            }
        }
    }
}
```

### Stale date handling

Set `staleDate` on `ActivityContent`. When the date passes without an update, the system may display a visual indicator that the content is outdated. Always provide a stale date and update before it elapses.

### iOS 26 changes

- **CarPlay Dashboard**: Live Activities started on iPhone automatically appear on CarPlay Dashboard.
- **macOS Tahoe**: Paired iPhone Live Activities appear in the macOS menu bar.
- **Scheduling API**: Third-party apps can now schedule Live Activity start/end times in advance.

---

## 6. BackgroundTasks Framework

### Task types

| Type | Duration | Purpose |
|------|----------|---------|
| `BGAppRefreshTaskRequest` | ~30 seconds | Periodic data refresh |
| `BGProcessingTaskRequest` | Minutes | Heavy work (ML, cleanup) |
| `BGContinuedProcessingTaskRequest` (iOS 26) | Extended | Continue foreground work in background |

### Registration

Register handlers at app launch, before the first scene is created.

```swift
import BackgroundTasks

@main
struct MyApp: App {
    init() {
        BGTaskScheduler.shared.register(
            forTaskWithIdentifier: "com.app.refresh",
            using: nil
        ) { task in
            self.handleAppRefresh(task as! BGAppRefreshTask)
        }

        BGTaskScheduler.shared.register(
            forTaskWithIdentifier: "com.app.processing",
            using: nil
        ) { task in
            self.handleProcessing(task as! BGProcessingTask)
        }
    }

    var body: some Scene {
        WindowGroup { ContentView() }
    }

    func handleAppRefresh(_ task: BGAppRefreshTask) {
        // Schedule the next refresh
        scheduleAppRefresh()

        let operation = Task {
            // Perform lightweight fetch
            let newData = try? await fetchLatestData()
            task.setTaskCompleted(success: newData != nil)
        }

        task.expirationHandler = {
            operation.cancel()
            task.setTaskCompleted(success: false)
        }
    }

    func handleProcessing(_ task: BGProcessingTask) {
        let operation = Task {
            await performHeavyWork()
            task.setTaskCompleted(success: true)
        }

        task.expirationHandler = {
            operation.cancel()
            task.setTaskCompleted(success: false)
        }
    }
}
```

### Info.plist

Add the permitted identifiers array:

```xml
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array>
    <string>com.app.refresh</string>
    <string>com.app.processing</string>
</array>
```

### Scheduling

```swift
func scheduleAppRefresh() {
    let request = BGAppRefreshTaskRequest(identifier: "com.app.refresh")
    request.earliestBeginDate = Date(timeIntervalSinceNow: 15 * 60)  // 15 min from now
    try? BGTaskScheduler.shared.submit(request)
}

func scheduleProcessing() {
    let request = BGProcessingTaskRequest(identifier: "com.app.processing")
    request.earliestBeginDate = Date(timeIntervalSinceNow: 60 * 60)  // 1 hour
    request.requiresNetworkConnectivity = true
    request.requiresExternalPower = false
    try? BGTaskScheduler.shared.submit(request)
}
```

### Testing in Xcode

Pause the debugger and run:

```
e -l objc -- (void)[[BGTaskScheduler sharedScheduler] _simulateLaunchForTaskWithIdentifier:@"com.app.refresh"]
```

### iOS 26: BGContinuedProcessingTask

Allows foreground-started work (video export, image processing, sensor analysis) to continue after the app is backgrounded. The system shows a progress indicator to the user.

```swift
BGTaskScheduler.shared.register(
    forTaskWithIdentifier: "com.app.continued",
    using: nil
) { task in
    guard let continuedTask = task as? BGContinuedProcessingTask else { return }
    // Continue the work that started in foreground
    Task {
        await finishExportingVideo()
        continuedTask.setTaskCompleted(success: true)
    }

    continuedTask.expirationHandler = {
        continuedTask.setTaskCompleted(success: false)
    }
}

// Request when starting foreground work
let request = BGContinuedProcessingTaskRequest(identifier: "com.app.continued")
try? BGTaskScheduler.shared.submit(request)
```

### iOS 26: Background GPU access

On supported devices, background tasks can access the GPU for ML inference or rendering.

```swift
// Add the background GPU capability in Xcode
// Check availability at runtime:
if BGTaskScheduler.shared.supportedResources.contains(.gpu) {
    // Schedule GPU-enabled background work
}
```

---

## 7. RelevanceKit (NEW in iOS 26)

RelevanceKit uses on-device intelligence to help widgets appear at the right time in the Smart Stack. No server-side logic needed.

Key concepts:
- **POI signals**: Location-based relevance — a coffee shop widget surfaces near cafes
- **Routine signals**: Time-of-day patterns — a workout widget surfaces at gym time
- **Activity signals**: watchOS 26 surfaces widgets based on detected physical activity

RelevanceKit runs entirely on-device. No user data leaves the device for relevance scoring.

```swift
import RelevanceKit

// Associate relevance metadata with your widget
// The system uses these hints when ranking widgets in the Smart Stack
```

Widgets using RelevanceKit appear proactively on:
- iOS 26 Smart Stack
- watchOS 26 Smart Stack
- Home Screen suggestions

---

## 8. Notification Summarization (Apple Intelligence)

Apple Intelligence can summarize notification stacks on-device. Your app's notifications are automatically eligible, but you can optimize how they summarize.

### Best practices for summarization-friendly notifications

```swift
// 1. Use descriptive, complete notification titles and bodies
let content = UNMutableNotificationContent()
content.title = "John sent you a message"        // Good: who + what
content.body = "Hey, are we still on for dinner tonight at 8?"  // Full context
// Bad: title = "New message", body = "1 unread" — nothing to summarize

// 2. Use Communication notifications for messaging apps
content.contentType = .communication  // tells the system this is a message

// 3. Set thread identifiers to group related notifications
content.threadIdentifier = "chat-\(conversationID)"

// 4. Use SenderIdentity for communication notifications
let sender = INPerson(personHandle: handle, nameComponents: nameComponents,
                       displayName: "John Doe", image: avatar)
let messageIntent = INSendMessageIntent(recipients: nil, outgoingMessageType: .text,
                                         content: content.body, speakableGroupName: nil,
                                         conversationIdentifier: conversationID,
                                         serviceName: nil, sender: sender)
let interaction = INInteraction(intent: messageIntent, response: nil)
interaction.direction = .incoming
try? await interaction.donate()
content = try content.updating(from: messageIntent)
```

### Notification categories for Apple Intelligence

| Category | Summarization behavior |
|----------|----------------------|
| `.communication` | Summarizes by sender/conversation, Smart Reply enabled |
| `.news` | Italicized summary with "Summarized by Apple Intelligence" label |
| `.entertainment` | Grouped and summarized by show/topic |
| Default | Generic summarization based on title + body content |

### What NOT to do
- Don't use generic titles like "Update" or "Alert" — the AI needs meaningful text
- Don't put critical info only in the subtitle — title + body are primary inputs
- Don't rely on notification summaries for accuracy — users can disable them per-app
- Don't include time-sensitive info only in the summary — keep it in the original notification too

---

## 9. TipKit (Feature Discovery)

### Defining a Tip

```swift
import TipKit

struct SwipeFeatureTip: Tip {
    var title: Text {
        Text("Swipe to Decide")
    }

    var message: Text? {
        Text("Swipe right to keep a photo, left to mark for deletion.")
    }

    var image: Image? {
        Image(systemName: "hand.draw")
    }

    var actions: [Action] {
        Action(id: "learn-more", title: "Learn More")
    }
}
```

### Displaying tips

```swift
struct CardStackView: View {
    let swipeTip = SwipeFeatureTip()

    var body: some View {
        VStack {
            // Inline tip
            TipView(swipeTip) { action in
                if action.id == "learn-more" {
                    showTutorial = true
                }
            }

            // Or as a popover
            PhotoCard()
                .popoverTip(swipeTip)
        }
    }
}
```

### Eligibility rules

```swift
struct UndoFeatureTip: Tip {
    // Parameter-based rule
    @Parameter
    static var hasSwipedOnce: Bool = false

    // Event-based rule
    static let swipeEvent = Event(id: "swipe-performed")

    var rules: [Rule] {
        #Rule(Self.$hasSwipedOnce) { $0 == true }
        #Rule(Self.swipeEvent, by: .count) { $0.donations.count >= 3 }
    }

    var title: Text {
        Text("Undo Mistakes")
    }

    var message: Text? {
        Text("Tap the undo button to bring back the last photo.")
    }
}

// Donate events when they occur
UndoFeatureTip.swipeEvent.donate()

// Set parameters when conditions change
UndoFeatureTip.hasSwipedOnce = true
```

### Configuration at launch

```swift
@main
struct MyApp: App {
    init() {
        try? Tips.configure([
            .displayFrequency(.daily),
            .datastoreLocation(.applicationDefault)
        ])
    }

    var body: some Scene {
        WindowGroup { ContentView() }
    }
}
```

| Frequency | Behavior |
|-----------|----------|
| `.immediate` | Show eligible tips right away |
| `.daily` | Show at most one tip per day |
| `.monthly` | Show at most one tip per month |
| `.hourly` | Show at most one tip per hour |

### Invalidation

```swift
let tip = SwipeFeatureTip()

// User performed the action — dismiss permanently
tip.invalidate(reason: .actionPerformed)

// Tip no longer relevant
tip.invalidate(reason: .displayCountExceeded)
```

### TipGroup

Show tips sequentially — only one at a time.

```swift
TipGroup {
    SwipeFeatureTip()
    UndoFeatureTip()
    DeleteFeatureTip()
}
```

### Testing and reset

```swift
// Reset all tip state (useful during development)
try? Tips.resetDatastore()

// Show all tips regardless of rules (for testing)
try? Tips.configure([.displayFrequency(.immediate)])
Tips.showAllTipsForTesting()

// Hide all tips
Tips.hideAllTipsForTesting()
```

---

## 10. Anti-Patterns

| Anti-Pattern | Why it fails | Do instead |
|---|---|---|
| Sending excessive push notifications | Users disable all notifications | Batch, throttle, let users choose frequency |
| Forgetting App Groups for widget data | Widget shows stale/empty data | Set up App Group, use shared UserDefaults or FileManager container |
| Updating widgets too frequently | System throttles reloads (~40-70/day budget) | Use timeline entries with future dates, reload only on meaningful changes |
| Using background tasks for real-time updates | BGTasks are system-scheduled, not immediate | Use push notifications for timely updates |
| Showing TipKit tips on first launch | Overwhelms new users who are still exploring | Gate tips with event rules (e.g., after 3 uses) |
| Ignoring notification permission denial | Users get no value from notification features | Detect denial, show in-app guidance with deep link to Settings |
| Skipping staleDate on Live Activities | Stale content erodes user trust | Always set a staleDate and update before it elapses |
| Registering BGTaskScheduler in viewDidLoad | Misses the registration window | Register in app init, before first scene creation |
| Forgetting mutable-content: 1 in APNs | Notification Service Extension never fires | Always include mutable-content: 1 when using Service Extension |
| Using opaque widget backgrounds on iOS 26 | Breaks Liquid Glass appearance | Use .containerBackground with translucent fills |
