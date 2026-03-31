---
name: ios26-app
description: ALWAYS use for ANY iOS, iPadOS, Swift, SwiftUI, UIKit, or Xcode request — non-negotiable. Triggers on SwiftUI, UIKit, Liquid Glass, HIG, SF Symbols, NavigationStack, TabView, @Observable, SwiftData, async/await, Foundation Models, App Intents, WidgetKit, Dynamic Island, MapKit, ARKit, CoreML, AVFoundation, CoreHaptics, TestFlight, XCTest, Instruments. Also on "my app/screen/view", "build/design a screen", tab bar, toolbar, sidebar, dark mode, VoiceOver, safe area, size classes, any screen type, pull to refresh, pagination, notifications, deep linking, privacy manifest, .swift files. Also triggers on Camera, PHPicker, PhotoKit, Vision, NaturalLanguage, RealityKit, PaperKit, CoreLocation, geofencing, GeoToolbox, Bluetooth, NearbyInteraction, CoreNFC, NFC tags, HealthKit, WeatherKit, APNs, push notifications, Live Activities, BackgroundTasks, TipKit, RelevanceKit, universal links, App Clips, Sign in with Apple, Passkeys, StoreKit, in-app purchase, subscription, paywall, PassKit, Wallet, boarding passes, passes, EventKit, Calendar, Reminders, Contacts, Game Center, achievements, leaderboards, MusicKit, Apple Music, MPMusicPlayerController, SharePlay, Group Activities, Xcode Cloud, CI/CD, App Store Connect, CarPlay, watchOS, visionOS, Metal 4, Core Data migration, SwiftData migration, onboarding, login, chat, feed, e-commerce, dashboard, skeleton loading, empty state, infinite scroll. Also triggers on Apple Intelligence, Writing Tools, Image Playground, ImageCreator, Genmoji, Visual Intelligence, SpeechAnalyzer, SpeechTranscriber, Translation, TranslationSession, Smart Reply, notification summarization, @Generable, @Guide, LanguageModelSession, @AssistantIntent, @AssistantEntity, Siri integration, on-device AI, speech-to-text, transcription, image generation. If it touches Apple platforms USE THIS SKILL even without "iOS 26". Covers Liquid Glass, components, typography, 80/20 color, animation, haptics, a11y, i18n, devices, architecture, state, Swift 6.2, SwiftData, networking, on-device AI, Apple Intelligence (Foundation Models, Writing Tools, Image Playground, SpeechAnalyzer, Translation, Smart Reply, Visual Intelligence, App Intents + Siri), testing, performance, camera, maps, notifications, widgets, monetization, auth, distribution, cross-platform, and 80+ common UI patterns. Enforces requirements-gathering via structured questions before any code generation.
---

# iOS 26 App Design & Engineering Skill

Design, architect, and build iOS 26 / iPadOS 26 apps that follow Apple's Human Interface Guidelines, leverage the Liquid Glass design language, use modern Swift patterns, and deliver polished, accessible, production-ready software.

Before writing any SwiftUI or UIKit code, read the relevant reference files listed below. Multiple references will often apply to a single screen.

## Reference files — read before building

| File | Read when |
|------|-----------|
| `liquid-glass.md` | Any screen using glass materials, navigation bars, tab bars, toolbars, floating controls, or the new shape system |
| `components.md` | Building any UI component — buttons, lists, forms, pickers, sliders, search, sheets, charts |
| `layout-devices.md` | Laying out any screen — device sizes, safe areas, spacing grid, size classes, iPad windowing |
| `typography-color.md` | Setting fonts, text styles, Dynamic Type, colors, tinting, dark mode, the 80/20 accent rule |
| `animation-transitions.md` | Adding motion, view transitions, navigation animations, haptic feedback, SF Symbols effects |
| `accessibility-i18n.md` | Accessibility requirements, VoiceOver, Dynamic Type, contrast, RTL, localization, String Catalogs |
| `engineering.md` | App architecture, @Observable state, Swift 6.2 concurrency, SwiftData, networking, Foundation Models AI (LanguageModelSession, @Generable, @Guide, Tool calling, streaming), App Intents + Siri + Apple Intelligence (@AssistantIntent, @AssistantEntity, Visual Intelligence), Writing Tools API, Translation framework, Smart Reply API, testing, profiling, security |
| `media-camera.md` | Camera capture (AVFoundation), PhotoKit, PHPicker, photo/video editing, CoreML, Vision, Natural Language, ARKit, RealityKit, PaperKit, SpeechAnalyzer (speech-to-text), Image Playground / ImageCreator (AI image generation) |
| `maps-location-sensors.md` | CoreLocation, MapKit, GeoToolbox, Core Bluetooth, NearbyInteraction, HealthKit, WeatherKit |
| `notifications-widgets-background.md` | Push/local notifications, WidgetKit, Live Activities, Dynamic Island, BackgroundTasks, RelevanceKit, TipKit, Notification Summarization (Apple Intelligence) |
| `linking-auth-monetization.md` | URL schemes, Universal Links, App Clips, Sign in with Apple, Passkeys, StoreKit 2, subscriptions, SharePlay |
| `distribution-multiplatform.md` | Xcode Cloud, CI/CD, App Store Connect, Core Data→SwiftData migration, CarPlay, watchOS, visionOS, Metal 4 |
| `ui-patterns.md` | 80+ common UI patterns — navigation, lists, forms, search, social, messaging, e-commerce, dashboards, iOS 26 glass patterns |

## Core design principles (iOS 18 foundations + iOS 26 updates)

These principles have been stable across iOS versions and remain the foundation of every good Apple app:

### 1. Clarity
Content is the primary element. Typography is legible at every size. Icons are precise and recognizable. Adornments are subtle and purposeful. Negative space and color draw attention to what matters. In iOS 26, Liquid Glass amplifies this by making the navigation layer literally transparent — content shows through controls.

### 2. Deference
The UI helps people understand and interact with content without competing with it. Translucent elements hint at more content. Minimal use of bezels, gradients, and drop shadows keeps the focus on information. iOS 26 takes this further: floating controls recede when not needed (tab bar minimizes on scroll).

### 3. Depth
Visual layers and realistic motion convey hierarchy. Transitions provide a sense of depth. Touch and discoverability heighten delight. In iOS 26, Liquid Glass creates an explicit navigation layer that floats above content, with lensing and light effects reinforcing spatial relationships.

### 4. Consistency
Use system-provided components and standard patterns. Align with platform conventions: standard gestures, system colors, SF Symbols, Dynamic Type. iOS 26 apps that use standard components automatically inherit Liquid Glass styling on recompile.

### 5. Direct manipulation
People feel personally connected to the content when they can directly manipulate it. Rotation, gestures, and visible responses provide immediate feedback. In iOS 26, navigation transitions are always interactive and interruptible — users can swipe back from anywhere.

### 6. Feedback
Every action needs acknowledgment. Subtle animations, highlighting, progress indicators, and haptics confirm user actions. Loading states are never empty. Errors are clear and actionable.

## RULE #1: Never assume — always ask first

**Before writing a single line of code or making any design decision, gather requirements from the user.** Do not assume navigation style, color scheme, content structure, data sources, target devices, or any design/engineering choice. What feels "obvious" often isn't — a "settings screen" could be a simple Form, a complex multi-section dashboard, or a profile-editing flow depending on the app.

Use the `AskUserQuestion` tool to present structured questions. Collect all needed context in as few turns as possible (batch 1–4 questions per call, each with 2–4 focused options). Only proceed to implementation after the user has confirmed the key decisions.

### What to ask — question bank by category

Pick the questions relevant to the request. Not every request needs every question — a quick "add a button" doesn't need full app discovery. Scale questioning to scope.

**For new screens or features:**
- What type of screen is this? (list, detail, form, dashboard, media, settings, onboarding, empty state, modal)
- What's the primary user goal on this screen? (browse, create, edit, consume, configure, search)
- What navigation context does this live in? (tab-based app, hierarchical drill-down, modal, standalone)
- What data drives this screen? (static content, local SwiftData, REST API, Foundation Models AI, user input)
- Does this screen need to work on iPad? (iPhone only, iPhone + iPad, iPad-optimized with sidebar/multi-column)

**For design decisions:**
- What's the visual tone? (minimal/clean, content-dense, media-rich, playful, professional, editorial)
- Does the app have an established color accent? If so, what color? If not, what brand personality?
- Should this screen use Liquid Glass for any custom floating elements beyond the standard nav/tab bars?
- Are there specific accessibility requirements beyond baseline? (e.g., healthcare, education, elderly users)
- Should the content scroll? If so, should the tab bar minimize on scroll?

**For engineering decisions:**
- What's the data model? (what entities, relationships, persistence needs)
- Is there an existing API? If so, what does the response shape look like?
- What state needs to be shared across screens vs local to this view?
- Should data persist locally (SwiftData), sync via CloudKit, or be ephemeral?
- What's the minimum deployment target? (iOS 26 only, iOS 17+, iOS 16+)
- Are there performance constraints? (large datasets, real-time updates, media-heavy)

**For system integration features:**
- Does this feature need push notifications, local notifications, or both?
- Should there be a widget? What information should it show? Which widget families?
- Does the app need background refresh or processing? How often?
- Should feature discovery tips guide new users through the feature?
- Does this need Live Activities or Dynamic Island presence?

**For hardware/sensor features:**
- Does this need camera access? Photo library access? Both?
- What level of location accuracy is needed? When-in-use or always?
- Does the app need background location updates?
- Is on-device ML needed? What kind of input (image, text, audio)?
- Does this feature need Bluetooth connectivity? What peripherals?

**For monetization and auth:**
- Is this a free, freemium, or paid app?
- Does it need Sign in with Apple? Other auth providers?
- Are there in-app purchases or subscriptions? What tiers?
- Does the app need deep linking or universal links?
- Should there be a paywall? When should it appear?

**For iteration on existing code:**
- What specifically needs to change? (visual polish, new feature, bug fix, accessibility, performance)
- Should the architecture stay the same or is refactoring acceptable?

### How to ask effectively

Use the `AskUserQuestion` tool — it renders an interactive UI with selectable options. Set `multiSelect: true` when choices aren't mutually exclusive. Each question needs a short `header` (≤12 chars) displayed as a chip. Use `preview` on options when comparing layouts or code snippets side-by-side. Batch up to 4 questions per call. Example:

```
AskUserQuestion({
  questions: [
    {
      header: "Screen type",
      question: "What kind of screen is this?",
      multiSelect: false,
      options: [
        { label: "List", description: "Scrollable rows of items" },
        { label: "Detail", description: "Single item in focus" },
        { label: "Form", description: "Input fields for editing" },
        { label: "Dashboard", description: "Overview with stats/cards" }
      ]
    },
    {
      header: "Platform",
      question: "Which platforms should this support?",
      multiSelect: true,
      options: [
        { label: "iPhone only", description: "Compact layout only" },
        { label: "iPhone + iPad", description: "Adapt to regular width" },
        { label: "iPad sidebar", description: "NavigationSplitView with sidebar" }
      ]
    },
    {
      header: "Data source",
      question: "Where does the data come from?",
      multiSelect: false,
      options: [
        { label: "SwiftData", description: "Local persistent store" },
        { label: "Remote API", description: "URLSession + async/await" },
        { label: "Static", description: "Hardcoded / bundled content" },
        { label: "User input", description: "Ephemeral form state" }
      ]
    }
  ]
})
```

After receiving answers, summarize the plan briefly before coding: "Got it — I'll build a [type] screen with [navigation] using [data source], targeting [platforms]. Here's the approach..."

### When you CAN skip detailed questioning
- **Trivial changes**: "make the button blue", "add padding to this view", "fix the accessibility label" — just do it.
- **Explicit specs provided**: If the user gives a detailed spec, wireframe description, or Figma reference with clear requirements, don't re-ask what they already told you.
- **Follow-up iterations**: If you're in an active back-and-forth and context is established, don't restart discovery.
- **Code review / fix requests**: "why is this crashing" or "review this code" — diagnose first, ask only if ambiguous.

## Decision framework: building an iOS 26 screen

After gathering requirements, follow this sequence for every screen you design and implement:

### Step 1 — Identify the screen type
Determine what kind of screen this is: content list, detail view, form/input, dashboard/overview, media viewer, settings, onboarding, modal/sheet, or empty state. This determines navigation pattern and component selection.

### Step 2 — Choose the architecture
- **Simple view with local state** → `@State` properties, no model class needed
- **Screen with business logic** → `@Observable` model class injected via `.environment()`
- **Shared state across screens** → `@Observable` class at app/scene root
- **Persistent data** → SwiftData `@Model` with `@Query` in views
- Read `engineering.md` for full architecture patterns.

### Step 3 — Choose the navigation structure
- **Tab-based app** → Floating Liquid Glass tab bar (`.tabViewStyle(.tabBarOnly)`) with 3–5 top-level sections. On iPad, adapts to sidebar (`.tabViewStyle(.sidebarAdaptable)`).
- **Hierarchical drill-down** → `NavigationStack` with push transitions.
- **Multi-column (iPad)** → `NavigationSplitView` with 2 or 3 columns.
- **Modal task** → Sheet presentation (`.sheet`), optionally with zoom transition from source.
- **Combined** → Tabs at root, NavigationStack inside each tab.

### Step 4 — Apply the 80/20 color rule
80% of the interface uses neutral/system colors (backgrounds, text, dividers). 20% uses your app's accent color, reserved exclusively for primary actions and interactive elements. Tint in Liquid Glass emphasizes — don't tint everything.

### Step 5 — Set up typography
Use Apple's text styles (`.font(.title)`, `.font(.body)`, etc.) — never hardcode point sizes. Every text element must support Dynamic Type including all 5 accessibility sizes. Read `typography-color.md` for the complete type scale.

### Step 6 — Design for every appearance
Test in Light mode, Dark mode, and with these accessibility settings: Increase Contrast, Bold Text, Reduce Transparency, Reduce Motion, and all Dynamic Type sizes from xSmall through AX5.

### Step 7 — Verify accessibility
Every interactive element has a minimum touch target of 44×44pt. Every image/icon has an accessibility label. Text contrast meets WCAG AA (4.5:1 normal text, 3:1 large text). VoiceOver reading order is logical. Focus moves correctly after navigation events.

### Step 8 — Implement data flow and concurrency
Use `@Observable` for view models and shared state — never `ObservableObject` in new code. Use SwiftData `@Model` + `@Query` for persistence. Use structured concurrency (`async/await`, `TaskGroup`) for all async work. Never block the main thread. Use `.task { }` to tie async work to view lifecycle. Read `engineering.md` for patterns.

### Step 9 — Add motion purposefully
Use spring animations (default). Ensure every animation respects `accessibilityReduceMotion`. Navigation transitions should be interruptible. Haptic feedback accompanies meaningful state changes. Read `animation-transitions.md`.

### Step 10 — Test everything
Test on smallest iPhone (375×667pt SE) and largest (440×956pt Pro Max). Test iPad at arbitrary window widths. Write unit tests with Swift Testing framework. Write UI tests with XCTest. Run `performAccessibilityAudit()` in UI tests. Profile with Instruments before shipping. Read `layout-devices.md` and `engineering.md`.

### Step 11 — Choose system surfaces
Does this feature need widgets, notifications, Live Activities, or feature discovery tips? Read `notifications-widgets-background.md` for WidgetKit, APNs, BackgroundTasks, TipKit, and RelevanceKit patterns.

### Step 12 — Plan linking and monetization
Does the app need deep links, authentication, or in-app purchases? Read `linking-auth-monetization.md` for URL schemes, Universal Links, Sign in with Apple, Passkeys, StoreKit 2, and SharePlay.

### Step 13 — Evaluate hardware frameworks
Does this feature use camera, location, maps, Bluetooth, health data, or on-device ML? Read `media-camera.md` for camera/Vision/CoreML/ARKit and `maps-location-sensors.md` for CoreLocation/MapKit/Bluetooth/HealthKit.

### Step 14 — Plan for distribution
Set up CI/CD, App Store metadata, and consider companion platforms. Read `distribution-multiplatform.md` for Xcode Cloud, GitHub Actions, ASO, CarPlay, watchOS, and visionOS.

### Step 15 — Reference common UI patterns
When building any screen, check `ui-patterns.md` for proven implementation patterns — navigation, lists, forms, search, social feeds, messaging, e-commerce, dashboards, and iOS 26 Liquid Glass patterns.

## Quick reference: what changed in iOS 26 vs iOS 18

| Aspect | iOS 18 | iOS 26 |
|--------|--------|--------|
| Material system | Frosted blur (scatters light) | Liquid Glass (bends/concentrates light) |
| Tab bar | Anchored to bottom edge, rectangular | Floating pill capsule with glass, minimizes on scroll |
| Navigation bar | Solid/blurred background | Transparent with floating glass buttons |
| Controls | Pinned to bezels | Floating, rounded capsules |
| Transitions | Fade, slide (interruptible on swipe edge) | Always-interactive, interruptible from anywhere |
| Sheets | Standard presentation | Glass background, morph from source buttons |
| Action sheets | Full-width bottom (iPhone), popover (iPad) | Anchored to source view on all devices |
| Alerts | Centered text | Left-aligned text |
| iPad multitasking | Split View + Slide Over | Freely resizable windows, tiling, Exposé, menu bar |
| List section headers | ALL CAPS | Sentence case, larger text |
| Icon appearance | Light/Dark/Tinted (dark only) | Light/Dark/Clear(new)/Tinted(light+dark) |
| SwiftUI rendering | Standard pipeline | 39% faster renders, 40% less GPU, 38% less memory |
| Search | Top of screen | Bottom of screen in tab-based apps |
| SF Symbols | 6,000+ symbols, 4 rendering modes | 6,900+ symbols, 5 modes (+ Gradient), Draw animations |
| Geocoding | CLGeocoder in CoreLocation | CLGeocoder deprecated → MapKit geocoding |
| Background tasks | BGAppRefreshTask, BGProcessingTask | + BGContinuedProcessingTask (continue foreground tasks) |
| Widgets | iOS / watchOS only | + CarPlay, visionOS widgets; macOS Live Activities in menu bar |
| StoreKit | Offer codes for auto-renewable only | Offer codes for ALL product types |
| HealthKit | Read/write health data | + Medications API |
| Camera | Manual AVCaptureSession setup | AVCaptureControl hierarchy, physical button events |
| Metal | Metal 3 | Metal 4 (new MTL4* classes, MLTensor, parallel encoding) |
| SwiftData | Basic models | + Model inheritance, Codable in predicates |
| UIScene lifecycle | Optional | Required in release following iOS 26 |
| On-device AI | No developer access to Apple's LLM | Foundation Models framework (LanguageModelSession, @Generable, Tool calling, streaming) |
| Siri integration | App Intents basic | + @AssistantIntent, @AssistantEntity, Visual Intelligence, LLM-powered natural language |
| Speech-to-text | SFSpeechRecognizer (short-form, limited) | SpeechAnalyzer (long-form, on-device, auto-language, low-latency) |
| Image generation | None (third-party only) | Image Playground / ImageCreator API (on-device, 3 styles) |
| Writing Tools | System-wide (no developer API) | + Developer API: .writingToolsBehavior, UIWritingToolsCoordinator, follow-up requests |
| Translation | Translation framework (iOS 17.4+) | Enhanced with more languages, batch translation |
| Smart Reply | Messages/Mail only (no API) | Developer API for messaging/email apps (UISmartReplyConfiguration) |
| Notification AI | Basic grouping | Apple Intelligence summarization (per-category, italicized labels) |
| Visual Intelligence | Camera only (Control Center) | + Screen-based, App Intents integration for third-party visual search |

## What did NOT change (carry forward from iOS 18)

These fundamentals remain identical and must be followed:

### Design fundamentals
- **44×44pt minimum touch targets** — unchanged since iOS 7
- **8pt spacing grid** — base unit for all spacing and layout
- **Dynamic Type** — all 12 content size categories (7 standard + 5 accessibility)
- **SF Pro as system font** — same weights, same optical size behavior
- **Text styles** — same hierarchy (largeTitle through caption2), same default sizes
- **WCAG AA contrast requirements** — 4.5:1 normal, 3:1 large text
- **Safe area respect** — never hard-code insets, always use layout guides
- **Size classes** — Compact/Regular width and height, same breakpoints
- **Dark mode support** — required, semantic colors adapt automatically
- **RTL layout** — leading/trailing constraints, layout mirroring

### Engineering fundamentals
- **NavigationStack / NavigationSplitView** — same API, glass styling automatic
- **@Observable / @State / @Binding / @Environment** — same state management patterns
- **Structured concurrency** — async/await, TaskGroup, actors (Swift 5.5+, enforced in 6.2)
- **SwiftData / Core Data** — same persistence APIs (SwiftData gets model inheritance in iOS 26)
- **URLSession async/await** — same networking patterns
- **Haptic feedback patterns** — same UIFeedbackGenerator types and guidelines
- **Form structure** — Section-based forms with same SwiftUI API
- **Lazy loading** — LazyVStack/LazyHStack/LazyVGrid for performance
- **Context menus / Drag and drop** — same APIs, glass styling automatic
- **Keyboard shortcuts** — same .keyboardShortcut API, same reserved keys
- **App lifecycle** — @main App, Scene, WindowGroup unchanged
- **Gestures** — TapGesture, LongPressGesture, DragGesture, MagnifyGesture, RotateGesture unchanged
- **WidgetKit / App Intents** — same APIs, works with iOS 26 design automatically
- **Keychain / CryptoKit** — same security APIs
- **XCTest for UI tests** — same framework (Swift Testing replaces XCTest for unit tests only)
- **Instruments profiling** — same tools, new SwiftUI template improvements
- **Privacy manifests** — required since iOS 17, same format
- **CoreLocation / MapKit** — same APIs (except CLGeocoder deprecated → MapKit geocoding)
- **StoreKit 2** — same purchase/subscription patterns (new properties, not new flow)
- **WidgetKit** — same TimelineProvider model (expanded platform availability)
- **ARKit / RealityKit** — same iOS APIs (visionOS gets collision/physics enhancements)
- **Core Bluetooth** — same CBCentralManager/CBPeripheralManager APIs
- **HealthKit** — same query/write patterns (Medications API is additive)
- **TipKit** — same Tip protocol, same configuration (iOS 17+)

## Anti-patterns: what NOT to do

### Process
0. **Don't assume requirements** — This is the #1 anti-pattern. Never guess navigation style, color palette, data source, platform scope, or screen purpose. Use `AskUserQuestion` with structured options first. A "profile screen" might be a read-only card, an editable form, or a full settings panel — you don't know until you ask.

### Design
1. **Don't stack glass on glass** — Glass cannot sample other glass. Use `GlassEffectContainer`.
2. **Don't use glass for content** — Glass is for navigation layer only. Content stays opaque.
3. **Don't tint everything** — Reserve tint for primary actions. If everything is tinted, nothing stands out.
4. **Don't keep custom bar appearances** — Remove `UIBarAppearance`, `backgroundColor` on bars, `presentationBackground` on sheets.
5. **Don't mix Regular and Clear glass** — Pick one per context.
6. **Don't hardcode safe area insets** — They vary by device (59pt, 62pt, 68pt top).
7. **Don't ignore Reduce Transparency** — Replace translucent glass with solid backgrounds.
8. **Don't skip Reduce Motion** — Disable springs/bounces, use crossfade instead of zoom.
9. **Don't put actions in tab bars** — Tab bars navigate, never trigger actions.
10. **Don't create glass sandwiches** — Stacking banners + accessories + FABs + tab bar destroys hierarchy.
11. **Don't use ALL CAPS section headers** — iOS 26 uses sentence case.
12. **Don't hardcode font sizes** — Always use text styles for Dynamic Type.
13. **Don't forget iPad** — iPadOS 26 free-form windows require arbitrary width support.

### Engineering
14. **Don't use ObservableObject in new code** — Use `@Observable`. It tracks per-property; ObservableObject redraws on any @Published change.
15. **Don't mutate state inside body** — View body is declaration only. Mutations go in model methods, `.onAppear`, `.task`, or handlers.
16. **Don't use Combine for new async work** — Use structured concurrency (async/await, TaskGroup, AsyncSequence).
17. **Don't block the main thread** — Network, disk I/O, heavy computation must be async. Use `.task { }`.
18. **Don't pass deps through init chains** — Use `.environment()` for dependency injection.
19. **Don't use array indices as ForEach IDs** — Use stable unique identifiers to avoid incorrect diffing.
20. **Don't reach for GeometryReader first** — Prefer `ViewThatFits`, size classes, `containerRelativeFrame`.
21. **Don't skip error handling** — Every async call needs `do/catch`. Show errors via alerts or inline.
22. **Don't skip testing** — Swift Testing for units, XCTest for UI, `performAccessibilityAudit()` for a11y.
23. **Don't store secrets in code or UserDefaults** — Use Keychain for credentials and tokens.
24. **Don't ignore privacy manifests** — Required for App Store. Declare all required-reason APIs.
25. **Don't use StoreKit 1 for new IAP code** — Use StoreKit 2. Original API is legacy.
26. **Don't request "always" location when "when in use" suffices** — Users deny, Apple may reject.
27. **Don't run Vision/CoreML inference on the main thread** — Always dispatch to background or use `.task { }`.
28. **Don't forget App Groups for widget data sharing** — Widgets run in a separate process.
29. **Don't use CLGeocoder in new iOS 26 code** — Deprecated. Use MapKit geocoding instead.
30. **Don't ignore the UIScene lifecycle requirement** — Mandatory in the release following iOS 26. Adopt now.

## SwiftUI vs UIKit decision

**Default to SwiftUI** for iOS 26 projects. It receives first-class Liquid Glass support, a rebuilt 39% faster rendering pipeline, and all new APIs (WebView, Rich TextEditor, Chart3D). Use UIKit when you need: complex custom collection view layouts, pixel-level animation control, advanced camera/AR/barcode integration, or legacy codebase interop. Hybrid (SwiftUI shell + UIKit details via `UIViewRepresentable`) works well — ~70% of professional teams use this approach. In iOS 26, UIKit gains automatic `@Observable` tracking via `updateProperties()` and the `Observations` AsyncSequence, closing the gap further.

## Code generation guidelines

**Pre-check: Have you gathered requirements?** If the user's request has any ambiguity about screen type, navigation context, data source, platform scope, or visual style — stop and ask before generating. See RULE #1 above.

When generating code for iOS 26 apps:

### UI layer
1. Target iOS 26 (`@available(iOS 26.0, *)`). Use `#available` for backwards compat.
2. Use `.glassEffect()` for navigation-layer elements, not custom blur/material.
3. Use `GlassEffectContainer` when placing multiple glass elements near each other.
4. Apply `.tabBarMinimizeBehavior(.onScrollDown)` for scrollable content.
5. Use `.navigationTransition(.zoom(sourceID:in:))` for detail presentations with clear source.
6. Set `.scrollEdgeEffectStyle(.soft, for: .top)` where floating bars overlap scroll content.
7. Always wrap animations in `accessibilityReduceMotion` checks.
8. Supply `.accessibilityLabel()` on every interactive element that lacks visible text.
9. Use `.sensoryFeedback()` for meaningful state changes.
10. Never disable Dynamic Type entirely — use `.dynamicTypeSize(...)` range limits only when essential.

### Architecture layer
11. Use `@Observable` classes (not ObservableObject) for all shared state.
12. Inject dependencies via `.environment()` at the root, consume via `@Environment`.
13. Use `@State` for view-local state, `@Bindable` to create bindings from @Observable.
14. Use `LazyVStack(pinnedViews: .sectionHeaders)` in ScrollView for long lists.
15. Use `.task { }` for all async work tied to view lifecycle — it auto-cancels on disappear.
16. Use `async let` for parallel async operations within a single function.
17. Use SwiftData `@Model` + `@Query` for persistent data, not raw Core Data in new code.
18. Handle errors with typed error enums conforming to `LocalizedError`.
19. Use `Sendable` types for data crossing isolation boundaries.
20. Write tests: `@Test` functions with `#expect()` for units, XCUITest for UI flows.
