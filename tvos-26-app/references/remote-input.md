# Remote & Input — Complete Reference

## Table of contents
1. Siri Remote hardware
2. SwiftUI remote commands
3. UIKit press handling
4. Game controller support
5. Text input strategies
6. Siri voice commands
7. Rules and anti-patterns

---

## 1. Siri Remote hardware

### Siri Remote (3rd gen, 2022+)
| Control | Type | Notes |
|---------|------|-------|
| **Clickpad** | Touch-sensitive + clickable | Swipe for direction, click to select. Outer ring for scrubbing. |
| **Back button** | Physical | Returns to previous screen. Maps to `.onExitCommand`. |
| **Home/TV button** | Physical | Holds for Control Center. Single press: Home. |
| **Play/Pause** | Physical | Maps to `.onPlayPauseCommand`. |
| **Volume** | IR buttons | Controls TV volume. Not programmable. |
| **Siri/Mic** | Physical | Hold to activate Siri. |
| **Power** | Physical | Turns TV on/off. Not programmable. |

### Siri Remote (2nd gen, 2021)
Same as 3rd gen but with aluminum body. Touch surface behavior is identical.

### Siri Remote (1st gen, 2015-2019)
Glass touch surface instead of clickpad. Swipe-only navigation (no click ring). Accelerometer and gyroscope included (useful for games). Menu button instead of Back button.

## 2. SwiftUI remote commands

### .onPlayPauseCommand — Play/Pause button

```swift
struct PlayerControlView: View {
    @State private var isPlaying = false

    var body: some View {
        ContentView()
            .onPlayPauseCommand {
                isPlaying.toggle()
            }
    }
}
```

### .onExitCommand — Back/Menu button

```swift
struct ModalView: View {
    @Environment(\.dismiss) var dismiss
    @State private var showingOverlay = false

    var body: some View {
        ZStack {
            MainContent()

            if showingOverlay {
                OverlayView()
            }
        }
        .onExitCommand {
            if showingOverlay {
                // Dismiss overlay first
                showingOverlay = false
            } else {
                // Default back behavior
                dismiss()
            }
        }
    }
}
```

**Critical**: If you don't handle `.onExitCommand`, the system performs default back navigation. If you handle it but do nothing, the back button appears "broken." Always either dismiss something or call `dismiss()`.

### .onMoveCommand — Directional swipe/click

```swift
struct CarouselView: View {
    @State private var selectedIndex = 0
    let items: [Item]

    var body: some View {
        HStack {
            ForEach(items.indices, id: \.self) { index in
                ItemCard(item: items[index], isSelected: index == selectedIndex)
            }
        }
        .onMoveCommand { direction in
            switch direction {
            case .left:
                selectedIndex = max(0, selectedIndex - 1)
            case .right:
                selectedIndex = min(items.count - 1, selectedIndex + 1)
            case .up, .down:
                break  // Let focus engine handle vertical
            @unknown default:
                break
            }
        }
    }
}
```

### Long press gesture

```swift
// Long press on clickpad (activates context menu or custom action)
Button {
    // Normal click action
    playItem(item)
} label: {
    PosterCard(item: item)
}
.contextMenu {
    Button("Add to List") { addToList(item) }
    Button("Share") { share(item) }
    Button("Mark as Watched") { markWatched(item) }
}
// Context menu appears on long press automatically
```

### Swipe gestures (for custom handling)

```swift
// The clickpad surface supports gesture recognition
// However, most navigation should be handled by the focus engine
// Use custom gesture handling sparingly

struct SwipeableView: View {
    var body: some View {
        ContentView()
            .gesture(
                DragGesture(minimumDistance: 20)
                    .onEnded { value in
                        let horizontal = value.translation.width
                        let vertical = value.translation.height

                        if abs(horizontal) > abs(vertical) {
                            if horizontal > 0 {
                                // Swiped right
                            } else {
                                // Swiped left
                            }
                        }
                    }
            )
    }
}
```

## 3. UIKit press handling

```swift
class CustomViewController: UIViewController {

    // Handle button presses
    override func pressesBegan(_ presses: Set<UIPress>, with event: UIPressesEvent?) {
        for press in presses {
            switch press.type {
            case .select:
                handleSelect()        // Clickpad click
            case .menu:
                handleBack()          // Back button
            case .playPause:
                handlePlayPause()     // Play/Pause
            case .upArrow:
                handleUp()
            case .downArrow:
                handleDown()
            case .leftArrow:
                handleLeft()
            case .rightArrow:
                handleRight()
            case .pageUp:
                handlePageUp()        // Clickpad ring swipe up
            case .pageDown:
                handlePageDown()      // Clickpad ring swipe down
            @unknown default:
                super.pressesBegan(presses, with: event)
            }
        }
    }

    override func pressesEnded(_ presses: Set<UIPress>, with event: UIPressesEvent?) {
        // Handle release
        super.pressesEnded(presses, with: event)
    }
}
```

### Gesture recognizers for touch surface

```swift
// UITapGestureRecognizer — clickpad tap
let tap = UITapGestureRecognizer(target: self, action: #selector(handleTap))
tap.allowedPressTypes = [NSNumber(value: UIPress.PressType.select.rawValue)]
view.addGestureRecognizer(tap)

// UISwipeGestureRecognizer — directional swipes
for direction: UISwipeGestureRecognizer.Direction in [.left, .right, .up, .down] {
    let swipe = UISwipeGestureRecognizer(target: self, action: #selector(handleSwipe))
    swipe.direction = direction
    view.addGestureRecognizer(swipe)
}

// UIPanGestureRecognizer — continuous touch tracking
let pan = UIPanGestureRecognizer(target: self, action: #selector(handlePan))
view.addGestureRecognizer(pan)
```

## 4. Game controller support

```swift
import GameController

class GameControllerManager {

    func setupControllers() {
        // Listen for controller connections
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(controllerConnected),
            name: .GCControllerDidConnect,
            object: nil
        )

        // Siri Remote as micro gamepad
        if let remote = GCController.controllers().first(where: { $0.microGamepad != nil }) {
            setupRemoteGamepad(remote)
        }
    }

    func setupRemoteGamepad(_ controller: GCController) {
        guard let gamepad = controller.microGamepad else { return }

        // D-pad (touch surface)
        gamepad.dpad.valueChangedHandler = { _, xValue, yValue in
            // xValue: -1.0 (left) to 1.0 (right)
            // yValue: -1.0 (down) to 1.0 (up)
        }

        // A button (clickpad press)
        gamepad.buttonA.pressedChangedHandler = { _, _, pressed in
            if pressed { /* select */ }
        }

        // X button (play/pause)
        gamepad.buttonX.pressedChangedHandler = { _, _, pressed in
            if pressed { /* play/pause */ }
        }
    }

    // Extended gamepad (MFi, DualSense, Xbox)
    func setupExtendedGamepad(_ controller: GCController) {
        guard let gamepad = controller.extendedGamepad else { return }

        gamepad.leftThumbstick.valueChangedHandler = { _, xValue, yValue in
            // Analog stick
        }

        gamepad.buttonA.pressedChangedHandler = { _, _, pressed in
            // A button
        }

        gamepad.buttonB.pressedChangedHandler = { _, _, pressed in
            // B button
        }

        // Triggers, bumpers, etc.
        gamepad.leftTrigger.pressedChangedHandler = { _, _, pressed in }
        gamepad.rightTrigger.pressedChangedHandler = { _, _, pressed in }
    }
}
```

## 5. Text input strategies

Text input on tvOS is challenging — no hardware keyboard (usually). Minimize text entry whenever possible.

### On-screen keyboard (default)
```swift
// Standard TextField shows the tvOS on-screen keyboard
TextField("Search", text: $query)
    .keyboardType(.default)

// Tip: Use .searchable() for search — provides better TV UX
NavigationStack {
    ContentGrid()
}
.searchable(text: $query, prompt: "Movies, Shows, People")
```

### Reduce text input — alternatives
```swift
// 1. Use selection lists instead of text fields
Picker("Genre", selection: $genre) {
    ForEach(Genre.allCases) { genre in
        Text(genre.name).tag(genre)
    }
}

// 2. Sign in on companion device (Remote app on iPhone)
// The system keyboard allows iPhone/iPad as input device

// 3. Siri dictation — user holds Siri button and speaks
// Handled automatically by the system when a text field has focus

// 4. QR code sign-in — show a QR code for phone-based auth
// Common pattern for streaming apps (Netflix, Disney+)
```

## 6. Siri voice commands

```swift
// Siri on Apple TV handles:
// - Text dictation in text fields
// - "Play [content name]"
// - "Search for [query]"
// - "Open [app name]"
// - "What did they just say?" (rewind 15s + captions)

// To support Siri deep links, implement:
// 1. Universal Links
// 2. NSUserActivity for content indexing
// 3. SiriKit intents (INPlayMediaIntent for media apps)

import Intents

class IntentHandler: INExtension, INPlayMediaIntentHandling {
    func handle(intent: INPlayMediaIntent) async -> INPlayMediaIntentResponse {
        guard let mediaItem = intent.mediaItems?.first else {
            return INPlayMediaIntentResponse(code: .failure, userActivity: nil)
        }
        // Resolve and play the media item
        return INPlayMediaIntentResponse(code: .success, userActivity: nil)
    }
}
```

## 7. Rules and anti-patterns

### DO:
- Handle `.onExitCommand` on every screen with custom overlays or modals
- Use `.onPlayPauseCommand` in media-related views
- Support game controllers if your app has game-like interactions
- Minimize text input — prefer selection, dictation, or companion device
- Test with both Siri Remote generations (click vs glass touch)
- Provide audio feedback where haptic feedback would be used on iOS

### DON'T:
- Don't override the Home/TV button — it's system-reserved
- Don't assume specific remote hardware — support all generations
- Don't require precision touch input (like drawing or drag-and-drop)
- Don't ignore the Play/Pause button — users expect it to work globally
- Don't require complex multi-gesture sequences — keep interactions simple
- Don't use tap/swipe gestures when the focus engine handles navigation fine
- Don't forget that the 1st gen remote has an accelerometer — some users play games with it