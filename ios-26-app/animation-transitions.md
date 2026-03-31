# Animation, Transitions, and Haptics

## Table of contents
1. Animation principles
2. Spring animations (default)
3. iOS 26 animation additions
4. View transitions
5. Navigation transitions
6. Sheet and modal transitions
7. Matched geometry and hero transitions
8. Symbol effects
9. Haptic feedback
10. Reduce Motion compliance
11. Performance guidelines

---

## 1. Animation principles

### Apple's core motion principles (unchanged since iOS 7)
- **Responsiveness**: Animations acknowledge user input instantly. Never delay visible feedback.
- **Continuity**: Motion maintains context — where things come from and where they go.
- **Naturalness**: Physics-based motion (springs, momentum) feels real. Avoid linear timing.
- **Purposefulness**: Every animation serves a function — orientation, feedback, or delight. Never animate for decoration.
- **Interruptibility**: Users should be able to interrupt any animation with a new gesture or action.

### iOS 26 amplifies these
- All standard navigation transitions are now **always-interactive** and **interruptible**
- Liquid Glass elements **materialize** rather than fade (gradual light modulation)
- Spring animations are the system default everywhere
- Content backswipe works from **anywhere** in the content area, not just the leading edge

## 2. Spring animations (default)

Springs are the default and recommended animation type. They maintain velocity continuity when interrupted, preventing jarring jumps.

### SwiftUI spring API (iOS 17+)
```swift
// Duration + bounce parameterization (preferred)
withAnimation(.spring(duration: 0.5, bounce: 0.3)) {
    isExpanded.toggle()
}

// Presets
withAnimation(.bouncy) { }                    // duration: 0.5, bounce: 0.3
withAnimation(.bouncy(duration: 0.4)) { }     // custom duration
withAnimation(.smooth) { }                     // duration: 0.5, bounce: 0
withAnimation(.smooth(duration: 0.3)) { }     // quick, no bounce
withAnimation(.snappy) { }                     // duration: 0.4, bounce: 0.15
withAnimation(.snappy(extraBounce: 0.1)) { }  // extra springiness

// Default spring
withAnimation(.spring()) { }  // response: 0.55, dampingFraction: 0.825

// View modifier
.animation(.spring(duration: 0.4, bounce: 0.2), value: someState)
```

### Spring parameter guide
| Parameter | Effect |
|-----------|--------|
| `duration` | Time to settle. 0.3–0.5 is typical. |
| `bounce` | 0 = critically damped (no overshoot). 0.3 = noticeably bouncy. 0.5 = very bouncy. Negative = overdamped. |

### Classic spring API (still valid)
```swift
.spring(response: 0.55, dampingFraction: 0.825, blendDuration: 0)
// response = natural period; dampingFraction: 1.0 = critical, <1.0 = underdamped (bouncy), >1.0 = overdamped
```

### When NOT to use springs
- **Progress indicators**: Use linear timing for determinate progress bars
- **Looping animations**: Use `.linear` with `.repeatForever()`
- **Color/opacity fades**: Use `.easeInOut` for simple cross-fades
- **Text transitions**: Use `.easeIn` for appearing text

## 3. iOS 26 animation additions

### flushUpdates (UIKit)
Automatically applies pending Observable/constraint updates during animation, eliminating `layoutIfNeeded()` calls:
```swift
UIView.animate(options: [.flushUpdates]) {
    // Constraint changes, Observable property changes
    // are automatically flushed and animated
}
```

### Always-interactive navigation
Standard push/pop transitions in `NavigationStack` are now always-interactive:
- Swipe back from **anywhere** in the content area (not just the leading 20pt edge)
- Interact with both the current and previous view during transition
- Tap back button multiple times rapidly — transitions chain smoothly
- No code changes needed — this is automatic in iOS 26

**Gesture conflict warning**: The always-interactive backswipe from anywhere in the content area may conflict with horizontal gestures in your views (carousels, swipeable cards, horizontal `ScrollView`s). The system prioritizes the navigation gesture. Test these interactions carefully — if critical horizontal gestures are being swallowed, consider restructuring the interaction rather than disabling navigation.

### Scroll edge effects
Content scrolling under floating glass bars can create harsh visual collisions. Use the new scroll edge effect to blur content edges:
```swift
ScrollView {
    // content
}
.scrollEdgeEffectStyle(.soft, for: .top)  // Blurs content at top edge under glass nav bar
```

### Liquid Glass materialization
Glass elements don't fade in/out. They appear by gradually increasing lensing intensity:
```swift
// The system handles materialization automatically for .glassEffect()
// No manual animation code needed for glass appearance/disappearance
```

### Glass morphing
Glass elements morph between positions using matched IDs:
```swift
@Namespace private var ns

Button("Open") { }
    .glassEffectID("control", in: ns)
    .glassEffect()

// When transitioning to another view, matching glass morphs smoothly
ToolbarItem {
    Button("Close") { }
        .glassEffectID("control", in: ns)
}
```

## 4. View transitions

### Built-in transitions
```swift
.transition(.opacity)           // Fade
.transition(.slide)             // Slide from leading edge
.transition(.move(edge: .bottom)) // Slide from specific edge
.transition(.scale)             // Scale from center
.transition(.scale(scale: 0.5, anchor: .topLeading)) // Custom anchor
.transition(.push(from: .bottom))  // Push with opacity
.transition(.blurReplace)       // iOS 17: blur during replacement
.transition(.symbolEffect)      // iOS 17: for SF Symbol transitions
```

### Combined transitions
```swift
.transition(.opacity.combined(with: .scale(scale: 0.8)))
.transition(.asymmetric(
    insertion: .move(edge: .trailing).combined(with: .opacity),
    removal: .move(edge: .leading).combined(with: .opacity)
))
```

### Custom transitions
```swift
struct SlideAndFade: Transition {
    func body(content: Content, phase: TransitionPhase) -> some View {
        content
            .opacity(phase.isIdentity ? 1 : 0)
            .offset(x: phase == .willAppear ? 50 : phase == .didDisappear ? -50 : 0)
    }
}
```

### ContentTransition (text and symbol changes)
```swift
Text("\(count)")
    .contentTransition(.numericText())  // Animates digit changes

Image(systemName: isPlaying ? "pause.fill" : "play.fill")
    .contentTransition(.symbolEffect(.replace))
```

## 5. Navigation transitions

### Zoom transition (iOS 18, enhanced in iOS 26)
Creates a spatial relationship between list item and detail view:
```swift
@Namespace private var namespace

NavigationLink(value: item) {
    ItemCard(item: item)
        .matchedTransitionSource(id: item.id, in: namespace)
}

// In destination:
.navigationTransition(.zoom(sourceID: item.id, in: namespace))
```

In iOS 26, zoom transitions automatically fall back to standard slide when Reduce Motion is enabled.

**Always-interactive side effects**: During always-interactive navigation, both the current and previous views are simultaneously interactive. Avoid triggering side effects in `.onAppear`/`.onDisappear` that assume exclusive view presence — both views may be visible and receiving touches at the same time.

### Standard push (enhanced in iOS 26)
No code changes needed — push transitions are now always-interactive and support backswipe from anywhere in the content area.

### Custom navigation transitions (UIKit)
```swift
class CustomTransition: NSObject, UINavigationControllerDelegate {
    func navigationController(
        _ nav: UINavigationController,
        animationControllerFor operation: UINavigationController.Operation,
        from fromVC: UIViewController,
        to toVC: UIViewController
    ) -> UIViewControllerAnimatedTransitioning? {
        return CustomAnimator(operation: operation)
    }
}
```

## 6. Sheet and modal transitions

### Standard sheet
```swift
.sheet(isPresented: $show) {
    DetailView()
        .presentationDetents([.medium, .large])
        .presentationDragIndicator(.visible)
        .presentationCornerRadius(20)
        .presentationBackgroundInteraction(.enabled(upThrough: .medium))
}
```

### Sheet with zoom from source (iOS 26)
```swift
@Namespace private var ns

Button("Show Details") { show = true }
    .matchedTransitionSource(id: "source", in: ns)

.sheet(isPresented: $show) {
    DetailView()
        .navigationTransition(.zoom(sourceID: "source", in: ns))
}
```

In iOS 26, sheets feature Liquid Glass backgrounds. Remove custom `.presentationBackground()` to let glass render properly.

### Full screen cover
```swift
.fullScreenCover(isPresented: $show) {
    FullScreenView()
        .navigationTransition(.zoom(sourceID: "item", in: ns))
}
```

## 7. Matched geometry and hero transitions

### matchedGeometryEffect (SwiftUI)
```swift
@Namespace private var animation

// Source
if !isExpanded {
    Image("photo")
        .matchedGeometryEffect(id: "hero", in: animation)
        .frame(width: 100, height: 100)
}

// Destination
if isExpanded {
    Image("photo")
        .matchedGeometryEffect(id: "hero", in: animation)
        .frame(maxWidth: .infinity, maxHeight: 300)
}

// Animate the toggle
withAnimation(.spring(duration: 0.5, bounce: 0.2)) {
    isExpanded.toggle()
}
```

### PhaseAnimator (iOS 17+)
Multi-step animations that cycle through phases:
```swift
PhaseAnimator([false, true]) { phase in
    Image(systemName: "heart.fill")
        .scaleEffect(phase ? 1.2 : 1.0)
        .foregroundStyle(phase ? .red : .pink)
} animation: { _ in
    .spring(duration: 0.3, bounce: 0.4)
}
```

### KeyframeAnimator (iOS 17+)
Time-based multi-property animations:
```swift
KeyframeAnimator(initialValue: AnimationState()) { state in
    Circle()
        .offset(x: state.x, y: state.y)
        .scaleEffect(state.scale)
} keyframes: { _ in
    KeyframeTrack(\.y) {
        SpringKeyframe(-100, duration: 0.3, spring: .bouncy)
        SpringKeyframe(0, duration: 0.5, spring: .bouncy)
    }
    KeyframeTrack(\.scale) {
        LinearKeyframe(1.5, duration: 0.2)
        SpringKeyframe(1.0, duration: 0.4, spring: .smooth)
    }
}
```

## 8. Symbol effects

### Discrete effects (trigger once)
```swift
.symbolEffect(.bounce, value: triggerValue)       // Bounces on change
.symbolEffect(.pulse, value: triggerValue)         // Pulses once
.symbolEffect(.breathe, value: triggerValue)       // Breathes once
```

### Continuous effects (while active)
```swift
.symbolEffect(.pulse, isActive: isLoading)         // Continuous pulse
.symbolEffect(.variableColor.iterative, isActive: isSearching)
.symbolEffect(.breathe, isActive: isListening)
.symbolEffect(.wiggle, isActive: hasError)
.symbolEffect(.rotate, isActive: isSyncing)
```

### Draw effects (NEW iOS 26)
```swift
.symbolEffect(.drawOn, isActive: isComplete)   // Calligraphic stroke appearance
.symbolEffect(.drawOff, isActive: isDismissed) // Stroke disappearance
```

### Replace transition
```swift
Image(systemName: isBookmarked ? "bookmark.fill" : "bookmark")
    .contentTransition(.symbolEffect(.replace))
```

## 9. Haptic feedback

### UIFeedbackGenerator types

#### Impact — physical collision simulation
```swift
let impact = UIImpactFeedbackGenerator(style: .medium)
impact.prepare()    // Call just before triggering
impact.impactOccurred()

// Styles: .light, .medium, .heavy, .rigid, .soft
// Custom intensity:
impact.impactOccurred(intensity: 0.7) // 0–1
```

#### Selection — discrete state changes
```swift
let selection = UISelectionFeedbackGenerator()
selection.prepare()
selection.selectionChanged()
// Use for: picker scrolling, segmented control changes, toggle flips
```

#### Notification — task outcomes
```swift
let notification = UINotificationFeedbackGenerator()
notification.prepare()
notification.notificationOccurred(.success)  // or .warning, .error
```

### SwiftUI sensory feedback (iOS 17+)
```swift
// Triggered by value change
.sensoryFeedback(.success, trigger: taskComplete)
.sensoryFeedback(.error, trigger: errorOccurred)
.sensoryFeedback(.selection, trigger: selectedItem)
.sensoryFeedback(.impact(weight: .heavy, intensity: 0.9), trigger: counter)
.sensoryFeedback(.increase, trigger: volume)
.sensoryFeedback(.decrease, trigger: brightness)
.sensoryFeedback(.start, trigger: isRecording)
.sensoryFeedback(.stop, trigger: isRecording)
.sensoryFeedback(.alignment, trigger: snappedToGuide)
.sensoryFeedback(.levelChange, trigger: currentLevel)
```

### Core Haptics (custom patterns)
```swift
import CoreHaptics

let engine = try CHHapticEngine()
try engine.start()

let sharpTap = CHHapticEvent(
    eventType: .hapticTransient,
    parameters: [
        CHHapticEventParameter(parameterID: .hapticIntensity, value: 1.0),
        CHHapticEventParameter(parameterID: .hapticSharpness, value: 1.0)
    ],
    relativeTime: 0
)

let pattern = try CHHapticPattern(events: [sharpTap], parameters: [])
let player = try engine.makePlayer(with: pattern)
try player.start(atTime: CHHapticTimeImmediate)
```

Two parameters: **hapticIntensity** (0–1, vibration strength) and **hapticSharpness** (0 = rounded/dull, 1 = precise/sharp). Two event types: `.hapticTransient` (short tap) and `.hapticContinuous` (sustained, max 30 seconds).

### Haptic design principles
- **Causality**: Every haptic must have a clear visual cause
- **Harmony**: The haptic should feel the way the visual looks (light tap for light visual, heavy for heavy)
- **Utility**: Provide information, don't just vibrate for fun
- **Restraint**: Too many haptics desensitize. Use them for meaningful moments:
  - Toggle state changes
  - Successful actions (save, send, complete)
  - Errors and warnings
  - Picker/stepper value changes
  - Pull-to-refresh threshold
  - Snap-to-position in drag operations

## 10. Reduce Motion compliance

### Required: respect the user's preference
```swift
@Environment(\.accessibilityReduceMotion) var reduceMotion

// Option 1: conditional animation
.animation(reduceMotion ? nil : .spring(), value: state)

// Option 2: alternative animation
.animation(reduceMotion ? .easeInOut(duration: 0.2) : .spring(duration: 0.5, bounce: 0.3), value: state)

// Option 3: conditional transition
.transition(reduceMotion ? .opacity : .move(edge: .bottom).combined(with: .opacity))
```

### What changes with Reduce Motion
- Spring bounce is eliminated — use critically damped or simple easing
- Zoom navigation transitions fall back to standard slide (automatic)
- Parallax/gyroscope effects are disabled
- Symbol effects are simplified (bounce → opacity change)
- Liquid Glass elastic properties are disabled
- Materialization animations become simple opacity fades

### What should NOT change with Reduce Motion
- Loading spinners (they indicate state, not decoration)
- Progress bars
- Essential state changes (toggle flip, checkbox check)
- Navigation pushes/pops (simplified but still present)

## 11. Performance guidelines

### Animation performance rules
- Target **60fps minimum** (120fps on ProMotion devices)
- Never trigger layout passes inside animation blocks
- Use `.drawingGroup()` for complex composited animations
- Prefer `.opacity` and `.offset` animations — they're GPU-accelerated
- Avoid animating `.frame()` changes on complex views — animate `.offset()` instead
- Use `TimelineView` for frame-synchronized continuous animations
- Profile with Core Animation Instruments — watch for offscreen rendering

### Energy efficiency
- Consider battery impact — minimize continuous animations, prefer `.animation()` over `TimelineView` when possible
- Avoid running animations when the app is not visible (background or covered by another app)
- Continuous symbol effects (`.pulse`, `.variableColor`) drain battery — disable when the triggering condition ends
- Test animation-heavy screens with Instruments Energy Log to verify acceptable power consumption

### Liquid Glass performance
iOS 26's Liquid Glass uses pre-compiled Metal shaders, so glass rendering has minimal CPU overhead. However:
- Don't create dozens of independent glass elements — group them in `GlassEffectContainer`
- Glass elements over rapidly changing content (video, heavy scroll) have higher GPU cost
- Test on oldest supported hardware (iPhone SE, iPad 10th gen)

### Animation frame budget
| Refresh rate | Budget per frame |
|-------------|-----------------|
| 60Hz | 16.67ms |
| 80Hz (ProMotion adaptive) | 12.5ms |
| 120Hz (ProMotion max) | 8.33ms |
