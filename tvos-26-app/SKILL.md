---
name: tvos-26-app
description: ALWAYS use for ANY tvOS, Apple TV, or TV-specific request — non-negotiable. Triggers on tvOS, Apple TV, TV app, focus system, Siri Remote, Top Shelf, TVUIKit, TVServices, 10-foot UI, .onPlayPauseCommand, .onExitCommand, .onMoveCommand, @FocusState (tvOS context), focusable(), focusSection(), parallax, TVPosterView, .buttonStyle(.card), AVPlayerViewController (tvOS), transport controls, .focusEffectDisabled, shelf layout, hero banner (TV), carousel (TV), living room, remote gestures, game controller, GCController, MFi controller, tvOS SwiftUI, TV navigation, sidebar (TV), overscan, safe area (TV), 1920x1080, full-screen browsing, tvOS Liquid Glass, TV tab bar, TV media playback, passthrough audio, TrueHD, DTS, Auro3D, Top Shelf extension, TVTopShelfContentProvider, layered images, .lsr files, Apple TV 4K, tvOS deployment target. If it touches Apple TV platform USE THIS SKILL. Covers focus engine, remote input, 10-foot layout, TV navigation, Liquid Glass on TV, media playback, Top Shelf, parallax, tvOS-specific SwiftUI, and TV engineering patterns. Enforces requirements-gathering via structured questions before any code generation.
---

# tvOS 26 App Design & Engineering Skill

Design, architect, and build tvOS 26 apps for Apple TV that follow Apple's Human Interface Guidelines for TV, leverage Liquid Glass on the big screen, master the focus-driven interaction model, and deliver cinematic, 10-foot UI experiences.

Before writing any tvOS code, read the relevant reference files listed below. Multiple references will often apply to a single screen.

## Reference files — read before building

| File | Read when |
|------|-----------|
| `references/focus-system.md` | Any screen with focusable elements — @FocusState, focusable(), focusSection(), focus guides, custom focus behavior, focus debugging |
| `references/remote-input.md` | Handling Siri Remote gestures, button presses, play/pause, menu/back, game controllers, keyboard input |
| `references/layout-10ft.md` | Any screen layout — 1920x1080 logical points, 60pt safe area insets, overscan, typography scale, card/poster sizing, spacing |
| `references/navigation-patterns.md` | Tab bars (top, not bottom), NavigationStack, NavigationSplitView sidebar, full-screen browsing, shelf rows, hero banners |
| `references/media-playback.md` | AVPlayerViewController on TV, transport bar customization, info panels, interstitials, PiP, background audio, passthrough audio |
| `references/liquid-glass-tv.md` | Liquid Glass on Apple TV — variants, shapes, tinting, focus-driven illumination, GlassEffectContainer, glass buttons |
| `references/top-shelf.md` | Top Shelf extensions, TVTopShelfContentProvider, sectioned vs inset content, deep linking from Top Shelf |
| `references/parallax-effects.md` | Parallax poster effects, layered images (.lsr), TVPosterView, custom SwiftUI parallax, focus-driven motion |
| `references/tvos-swiftui.md` | tvOS-only modifiers, behaviors that differ from iOS, ButtonStyle differences, sheets on TV, searchable on TV |
| `references/engineering-tv.md` | Architecture, @Observable, Swift 6.2, SwiftData, CloudKit sync, concurrency, networking, TV Provider SSO, multi-user profiles, what iOS frameworks are NOT available on tvOS |
| `references/accessibility-tv.md` | VoiceOver on tvOS, Dynamic Type, Closed Captions, Audio Descriptions, Reduce Motion/Transparency/Contrast, accessibility testing checklist |
| `references/companion-handoff.md` | Handoff (NSUserActivity), Continuity Keyboard, QR code sign-in, Universal Links (tvOS+iOS), CloudKit cross-device sync |

## Core design principles for TV

These principles are specific to the 10-foot, focus-driven TV experience:

### 1. Cinematic presence
The TV is the largest Apple screen. Content should feel immersive and theatrical. Use full-bleed imagery, generous whitespace, and bold typography. Let the content breathe — every pixel matters at 10 feet.

### 2. Focus is everything
There is no touch. The focus system IS the interaction model. Every navigable element must have clear, visible focus states. Focus movement must be predictable and never get "stuck." The user should always know where they are and where they can go.

### 3. Lean-back simplicity
TV is a lean-back, shared experience. Minimize text input, reduce cognitive load, keep navigation shallow. Users hold a remote at arm's length — every action should be achievable with minimal button presses.

### 4. Content-forward hierarchy
The TV app, Apple Music, and Apple Fitness+ all follow the same pattern: hero content at the top, horizontal shelf rows below, minimal chrome. Navigation recedes. Content dominates. Follow this pattern.

### 5. Spatial depth
Parallax on focus, scale transitions, shadows, and Liquid Glass create a layered spatial experience. Focused elements lift toward the viewer. Unfocused elements settle back. This depth hierarchy is fundamental to TV UI.

### 6. Shared viewing
Multiple people watch TV together. Avoid personal/private UI patterns. Design for the "across the room" viewer — large text, high contrast, clear iconography, no fine detail that requires leaning in.

## RULE #1: Never assume — always ask first

**Before writing a single line of code or making any design decision, gather requirements from the user.** TV apps have unique constraints — focus navigation, remote input, overscan, 10-foot readability — that make assumptions especially dangerous.

Use the `AskUserQuestion` tool to present structured questions. Collect all needed context in as few turns as possible (batch 1–4 questions per call, each with 2–4 focused options). Only proceed to implementation after the user has confirmed the key decisions.

### What to ask — question bank for TV

Pick the questions relevant to the request. Scale questioning to scope.

**For new screens or features:**
- What type of TV screen is this? (content shelf/browse, detail/hero, media player, settings, search, onboarding, empty state)
- What's the primary user goal? (browse content, watch media, configure settings, discover new content, search)
- What's the content layout? (horizontal shelves, grid, single hero, split-view with sidebar, full-screen media)
- What data drives this screen? (local SwiftData, REST API, static content, Top Shelf extension)
- Does this need Top Shelf support? (sectioned shelves, inset banners, none)

**For focus/navigation decisions:**
- How should focus flow on this screen? (left-right shelves, grid movement, sidebar toggle, custom)
- Does any element need initial/preferred focus? (hero banner, first item, search field)
- Are there focus traps to avoid? (modal overlays, sidebar transitions, nested scrollable regions)
- Should the screen respond to specific remote buttons? (play/pause, menu/back custom behavior)

**For media playback:**
- What type of media? (video streaming, live TV, audio, short-form clips)
- Does the player need custom transport controls? (skip intro, audio/subtitle selection, info panel)
- Should playback support PiP or background audio?
- Are there interstitials/ad breaks to handle?

**For design decisions:**
- What's the visual tone? (cinematic/dark, clean/minimal, content-dense, branded)
- Should this use Liquid Glass for any custom floating elements?
- Are there parallax poster effects needed? (layered images, focus-driven depth)
- Does this need to match an existing iOS companion app's visual language?

**For accessibility:**
- Does this need VoiceOver support? (section headers, custom labels, grouped elements)
- Does the app play video? (Closed Captions and Audio Descriptions required in most markets)
- Are there dynamic content changes that need VoiceOver announcements?

**For companion app / cross-device:**
- Does this need Handoff support? (continue watching from TV → iPhone or vice versa)
- How will users sign in? (QR code, TV Provider SSO, on-screen keyboard — QR is preferred)
- Should watchlist/preferences sync between TV and iPhone via CloudKit?
- Does this need Universal Links that work on both tvOS and iOS?

**For iteration on existing code:**
- What specifically needs to change? (focus behavior, layout, performance, new feature)
- Should the architecture stay the same or is refactoring acceptable?

### When you CAN skip detailed questioning
- **Trivial changes**: "make the button bigger", "fix focus order", "add padding" — just do it.
- **Explicit specs provided**: User gives detailed wireframe/spec — don't re-ask.
- **Follow-up iterations**: Active back-and-forth with established context.
- **Code review / fix requests**: Diagnose first, ask only if ambiguous.

## Decision framework: building a tvOS 26 screen

After gathering requirements, follow this sequence:

### Step 1 — Identify the TV screen type
- **Content shelf/browse** — horizontal scrolling shelves with posters (Apple TV app pattern)
- **Hero detail** — full-screen backdrop with metadata, actions, related content
- **Media player** — AVPlayerViewController with custom transport controls
- **Settings/preferences** — form-based, simple focus flow
- **Search** — full-screen search with keyboard and results
- **Sidebar browse** — NavigationSplitView with category sidebar

### Step 2 — Design the focus flow
Map out how focus moves through the screen. Draw the focus graph mentally:
- What gets focus first? (`.prefersDefaultFocus()`)
- How does focus flow between sections? (`.focusSection()`)
- Are there gaps where focus could get lost? (`UIFocusGuide`)
- Does any section need focus isolation? (`.focusScope()`)

### Step 3 — Apply TV layout constraints
- 1920x1080 logical points, 60pt safe area insets all edges
- Body text minimum 29pt, titles 48-76pt
- Focus targets minimum 66x66pt
- Poster cards: ~250x375pt (2:3), ~350x197pt (16:9)
- Horizontal shelf spacing: 40-50pt between items
- Read `references/layout-10ft.md` for complete sizing

### Step 4 — Choose the navigation structure
- **Tab-based** → TabView (renders at TOP of screen on tvOS)
- **Hierarchical** → NavigationStack (back via Menu button)
- **Sidebar** → NavigationSplitView (sidebar slides from left)
- **Full-screen browse** → Hero + shelf rows (Apple TV app pattern)
- Read `references/navigation-patterns.md` for patterns

### Step 5 — Apply Liquid Glass (tvOS 26)
- Glass is for navigation chrome only (tab bar, toolbars, floating controls)
- Focus-driven illumination replaces touch-driven (no fingertip, focus state triggers)
- Same API as iOS: `.glassEffect()`, `.buttonStyle(.glass)`, `GlassEffectContainer`
- Read `references/liquid-glass-tv.md` for TV-specific rules

### Step 6 — Add focus feedback and parallax
- Every focusable element needs visible focus state (scale, shadow, highlight)
- Use parallax for poster imagery (layered images or custom SwiftUI)
- Focus scale: 1.05-1.1x
- Read `references/parallax-effects.md` and `references/focus-system.md`

### Step 7 — Handle remote input
- Map `.onPlayPauseCommand`, `.onExitCommand`, `.onMoveCommand` as needed
- Context menus via long press on clickpad
- Never assume touch gestures — everything is focus + click
- Read `references/remote-input.md`

### Step 8 — Implement architecture
- `@Observable` for view models, `.environment()` for injection
- `.task { }` for async work, structured concurrency
- SwiftData if persistence needed
- Read `references/engineering-tv.md` for what's available (and what's NOT) on tvOS

### Step 9 — Add motion purposefully
- Spring animations for focus transitions (default)
- Respect `accessibilityReduceMotion`
- Parallax on focus, scale transitions, shadow changes
- Haptics are NOT available on tvOS (no Taptic Engine) — use sound effects sparingly instead

### Step 10 — Test on actual hardware
- Test with physical Siri Remote (not just keyboard in simulator)
- Verify focus navigation doesn't get stuck
- Check 60pt safe area on multiple TV brands
- Verify overscan behavior
- Test with game controllers if supported
- Profile with Instruments

## Quick reference: tvOS 26 vs iOS 26

| Aspect | iOS 26 | tvOS 26 |
|--------|--------|---------|
| Input | Touch + gestures | Focus engine + Siri Remote |
| Screen | 375-440pt wide, 2-3x | 1920x1080pt, 1x |
| Tab bar | Bottom, floating pill | Top of screen, glass |
| Navigation back | Swipe edge, back button | Menu button on remote |
| Haptics | UIFeedbackGenerator | Not available |
| Sheets | Half-sheet / full | Always full-screen |
| Search | Inline .searchable() | Full-screen keyboard |
| Typography | Body 17pt | Body 29pt |
| Safe area | Device-specific notch/island | 60pt all edges (overscan) |
| Liquid Glass | Touch-driven illumination | Focus-driven illumination |
| Parallax | Not standard | Core interaction pattern |
| Camera/AR | Full support | Not available |
| Location | Full CoreLocation | Not available |
| Widgets | WidgetKit | Not available |
| AI/Foundation Models | Full Apple Intelligence | Not available |
| HealthKit/Sensors | Available | Not available |
| Background audio | Supported | Supported |
| PiP | Supported | Supported (tvOS 14+) |
| Metal | Metal 4 | Metal 4 (Apple TV 4K 3rd gen) |
| Top Shelf | N/A | TVTopShelfContentProvider |

## What carries over from iOS 26 (shared foundations)

- Swift 6.2 with strict concurrency
- `@Observable`, `@State`, `@Binding`, `@Environment`
- SwiftData `@Model` + `@Query` with model inheritance
- `async/await`, `TaskGroup`, structured concurrency
- URLSession async/await networking
- Liquid Glass APIs (`.glassEffect()`, `.buttonStyle(.glass)`)
- SF Symbols 6 with all rendering modes
- SwiftUI performance improvements (39% faster renders)
- `ContentUnavailableView`
- Swift Charts (2D)
- NavigationStack / NavigationSplitView
- Same SwiftUI state management patterns

## What is NOT available on tvOS

These iOS/iPadOS frameworks and features do **not exist** on Apple TV:

- **Apple Intelligence / Foundation Models** — No Neural Engine capable enough
- **Camera / PhotoKit / Vision** — No camera hardware
- **CoreLocation / MapKit** — No GPS
- **HealthKit / NearbyInteraction** — No sensors
- **ARKit / RealityKit** — No AR
- **WidgetKit / Live Activities / Dynamic Island** — No home screen widgets
- **PaperKit** — No document scanning
- **SpeechAnalyzer** — No on-device speech processing
- **Image Playground / ImageCreator** — No AI image generation
- **Writing Tools API** — No text editing context
- **UIFeedbackGenerator / Haptics** — No Taptic Engine
- **Multi-touch gestures** — No touchscreen
- **Drag and drop** — No direct manipulation
- **Notifications / badges** — No notification center on TV

## Anti-patterns: what NOT to do on tvOS

### Process
0. **Don't assume requirements** — TV UI has unique constraints. Always ask first.

### Design
1. **Don't use small text** — Body under 29pt is unreadable at 10 feet. Never hardcode small sizes.
2. **Don't ignore overscan** — Content in outer 60pt gets clipped on many TVs. Use safe area guides.
3. **Don't forget focus states** — Every focusable element MUST have visible focus feedback. Unfocused state must also be clear.
4. **Don't create focus traps** — Test that focus can always escape any section. Use `UIFocusGuide` for gaps.
5. **Don't use iOS tab bar patterns** — tvOS tabs are at the TOP, not bottom.
6. **Don't overcrowd shelves** — Maximum 5-7 visible items per horizontal row at standard poster sizes.
7. **Don't assume touch** — No tap, swipe, pinch, rotate, or drag gestures on the screen.
8. **Don't use haptics** — tvOS has no Taptic Engine. Use focus sound effects or visual feedback instead.
9. **Don't require text input** — Minimize keyboard usage. Use Siri dictation, selection lists, or sign-in on companion device.
10. **Don't build personal/private UI** — TV is shared. Avoid patterns that assume single-user viewing.
11. **Don't use thin fonts** — Room lighting varies. Use medium/semibold weights for readability.
12. **Don't place critical content at edges** — Overscan zones are dangerous. Keep key content well within safe areas.

### Engineering
13. **Don't use iOS-only frameworks** — No Camera, Location, Health, AR, Widgets, Apple Intelligence on tvOS.
14. **Don't ignore Menu button** — Always handle `.onExitCommand` to prevent unexpected behavior.
15. **Don't use custom navigation that breaks focus** — Use `NavigationStack`, not custom view swapping.
16. **Don't test only in simulator** — Simulator keyboard ≠ Siri Remote. Always test on hardware.
17. **Don't use ObservableObject** — Use `@Observable` for all new code.
18. **Don't block the main thread** — Same as iOS: all async work in `.task { }`.
19. **Don't use array indices as ForEach IDs** — Use stable identifiers.
20. **Don't use GeometryReader when focus-based sizing works** — Prefer `containerRelativeFrame` and focus scaling.
21. **Don't skip VoiceOver and caption support** — Required in most markets. Test with VoiceOver on hardware, mark section headers, and include captions in video content. Read `references/accessibility-tv.md`.
22. **Don't require on-screen keyboard for sign-in** — Implement QR code sign-in or TV Provider SSO. Typing email/password via the on-screen keyboard is painful and causes abandonment. Read `references/companion-handoff.md`.

## Code generation guidelines

**Pre-check: Have you gathered requirements?** If ambiguous — stop and ask. See RULE #1.

When generating code for tvOS 26 apps:

### UI layer
1. Target tvOS 26 (`@available(tvOS 26.0, *)`).
2. Use `.glassEffect()` for navigation-layer elements only.
3. Design all focus flow upfront — map focus movement before coding.
4. Use `.focusSection()` to isolate horizontal shelves from vertical scroll.
5. Apply `.prefersDefaultFocus()` for initial focus on key elements.
6. Scale focused elements 1.05-1.1x with spring animation.
7. Respect 60pt safe area insets on all edges.
8. Use body text ≥ 29pt, titles ≥ 48pt.
9. Supply `.accessibilityLabel()` on every interactive element.
10. Handle `.onExitCommand` and `.onPlayPauseCommand` where appropriate.

### Architecture layer
11. Use `@Observable` classes for all shared state.
12. Inject via `.environment()` at root, consume via `@Environment`.
13. Use `.task { }` for async work tied to view lifecycle.
14. Use SwiftData `@Model` + `@Query` for persistent data.
15. Handle errors with typed error enums conforming to `LocalizedError`.
16. Use `Sendable` types for data crossing isolation boundaries.
17. Check framework availability before importing — many iOS frameworks don't exist on tvOS.
