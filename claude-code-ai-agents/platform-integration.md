# Platform Integration -- Cross-Skill Orchestration

How the `claude-code-ai-agents` skill composes with the four domain skills in this repo: `ios-26-app`, `tvos-26-app`, `web-frontend`, and `architect`. Covers decision routing, MCP cross-cutting workflows, agent-per-platform decomposition, shared code patterns, and end-to-end orchestration pipelines.

This file assumes familiarity with the Agent SDK (`query()`, `SDKMessage`, `SDKOptions`). If you need the SDK reference, read `agent-sdk-orchestration.md` first. If you need MCP server configuration details, read `mcp-consuming.md`.

---

## 1. Integration with ios-26-app

The `ios-26-app` skill covers iOS 26 / iPadOS 26 apps with SwiftUI, Liquid Glass, modern Swift 6.2, `@Observable`, SwiftData, and the full Apple Intelligence stack (Foundation Models, Writing Tools API, Image Playground, SpeechAnalyzer, Translation, Smart Reply, App Intents + Siri).

### Foundation Models -- on-device vs Claude API

iOS 26 introduces Foundation Models framework: an on-device LLM accessible via `LanguageModelSession`. The critical design decision is **when to use on-device inference vs Claude API calls**. These are complementary, not competing -- the on-device model handles latency-sensitive, privacy-critical, and offline tasks; Claude API handles complex reasoning.

| Scenario | Foundation Models (on-device) | Claude API (cloud) |
|----------|-------------------------------|-------------------|
| Text classification / sentiment | Best -- low latency, offline, private | Overkill |
| Short summarization (<500 words) | Best -- fast, private, no network cost | Overkill |
| Complex multi-step reasoning | Too limited -- small model | Required |
| Code generation / review | Not capable | Required |
| Image understanding / analysis | Limited to Apple Vision framework | Better quality with Claude Vision |
| Privacy-critical data (health, finance) | Best -- stays on device | Risk -- leaves device |
| Offline operation | Works -- no network required | Impossible -- requires network |
| Structured data extraction (`@Generable`) | Good for simple schemas | Better for complex nested schemas |
| Multi-turn conversation | Limited context | Full 200K context window |
| Tool calling with external APIs | On-device tool calling (iOS 26) | Full MCP tool ecosystem |

#### Foundation Models Swift example

```swift
import FoundationModels

// Simple one-shot
let session = LanguageModelSession()
let response = try await session.respond(to: "Classify this review as positive, neutral, or negative: \(reviewText)")

// Structured output with @Generable
@Generable
struct SentimentResult {
    let sentiment: String // "positive", "neutral", "negative"
    let confidence: Double
    @Guide(description: "Key phrases that drove the classification")
    let keyPhrases: [String]
}

let result: SentimentResult = try await session.respond(to: "Analyze: \(reviewText)", generating: SentimentResult.self)
```

#### Claude API in iOS (Anthropic Swift SDK)

```swift
import AnthropicSwiftSDK

let client = AnthropicClient(apiKey: ProcessInfo.processInfo.environment["ANTHROPIC_API_KEY"]!)

let message = try await client.messages.create(
    model: .claude4Sonnet,
    maxTokens: 1024,
    messages: [.user("Analyze this codebase structure and suggest architectural improvements:\n\n\(codeSnippet)")]
)
```

#### Hybrid pattern: on-device triage, cloud escalation

The highest-value pattern is using Foundation Models as a fast local triage layer that decides whether a request needs cloud escalation:

```swift
import FoundationModels
import AnthropicSwiftSDK

@Observable
final class HybridAIManager {
    private let localSession = LanguageModelSession()
    private let cloudClient = AnthropicClient(apiKey: ProcessInfo.processInfo.environment["ANTHROPIC_API_KEY"]!)

    @Generable
    struct TriageResult {
        let needsCloudEscalation: Bool
        @Guide(description: "Brief reason for escalation decision")
        let reason: String
    }

    func process(_ userQuery: String) async throws -> String {
        // Step 1: On-device triage (< 200ms, offline-capable)
        let triage: TriageResult = try await localSession.respond(
            to: "Does this query require complex reasoning, code generation, or multi-step analysis? Query: \(userQuery)",
            generating: TriageResult.self
        )

        if triage.needsCloudEscalation {
            // Step 2: Cloud escalation for complex tasks
            let message = try await cloudClient.messages.create(
                model: .claude4Sonnet, maxTokens: 2048,
                messages: [.user(userQuery)]
            )
            return message.content.first(where: { $0.type == .text })?.text ?? ""
        } else {
            // Step 2: On-device response for simple tasks
            let response = try await localSession.respond(to: userQuery)
            return response.text
        }
    }
}
```

### Apple Intelligence APIs + Claude API composition

| Apple Intelligence API | What it does on-device | Claude API augmentation |
|------------------------|------------------------|-------------------------|
| **Writing Tools** (`.writingToolsBehavior`, `UIWritingToolsCoordinator`) | System-wide text rewriting, proofreading, summarization | Domain-specific suggestions beyond system defaults -- legal tone, medical precision, brand voice |
| **Image Playground / ImageCreator** | On-device image generation (3 styles: Animation, Illustration, Sketch) | Claude Vision for analyzing generated images, describing them for accessibility, or evaluating quality |
| **SpeechAnalyzer** | On-device speech-to-text transcription (long-form, auto-language) | Send transcription to Claude for summarization, key point extraction, action item generation |
| **Translation** (TranslationSession) | On-device text translation (50+ languages) | Claude for nuanced translation requiring cultural context or domain expertise |
| **Smart Reply** (UISmartReplyConfiguration) | Suggested replies for messaging/email apps | Claude for composing longer, context-aware responses beyond simple suggestions |
| **Visual Intelligence** (App Intents integration) | Screen-based and camera-based visual search | Claude Vision for deeper analysis of what Visual Intelligence surfaces |

### Agent-assisted iOS development workflow

When building iOS features using agents, the workflow connects `architect` (for planning), `ios-26-app` (for platform conventions), and `claude-code-ai-agents` (for orchestration):

```
Step 1: /architect Phase 1
  Orchestrator reads ios-26-app/SKILL.md to identify relevant reference files.
  For a navigation feature: reads liquid-glass.md, components.md, layout-devices.md.
  For AI integration: reads engineering.md (Foundation Models section).
  GitHub MCP: reads related issues for context.
  Figma MCP: gets design specs for the feature.
  Output: ## Context section in docs/plans/<slug>.md

Step 2: /architect Phase 2
  Designs SwiftUI view hierarchy using ios-26-app patterns:
    - @Observable view models (never ObservableObject)
    - Liquid Glass for navigation layer
    - Dynamic Type + accessibility sizing
    - Swift 6.2 structured concurrency
  Cross-cutting concerns: accessibility, concurrency, memory (from phase-2-plan.md checklist).
  Output: ## Plan section approved via ExitPlanMode

Step 3: Parallel worker agents (Agent SDK)
  Orchestrator decomposes into isolated workers:
    Worker 1 (views): SwiftUI views following ios-26-app/components.md patterns
    Worker 2 (models): @Observable view models + SwiftData @Model entities
    Worker 3 (networking): URLSession async/await + Zod-equivalent validation
    Worker 4 (AI layer): Foundation Models integration per engineering.md
  Each worker gets: ios-26-app SKILL.md context + specific reference files + plan section

Step 4: Test agent
  Uses Bash to run: xcodebuild test -scheme <AppScheme> -destination 'platform=iOS Simulator,name=iPhone 16'
  Validates: accessibility audit (performAccessibilityAudit()), unit tests (@Test + #expect), UI tests

Step 5: Review + PR
  GitHub MCP: creates PR with implementation details
  Figma MCP: verifies implementation matches design (export_node_as_image comparison)
```

#### iOS worker agent with SDK

```typescript
import { query } from "@anthropic-ai/claude-code";

async function iosViewWorker(
  viewSpec: { name: string; type: string; planSection: string },
  projectDir: string
): Promise<void> {
  for await (const msg of query({
    prompt: `
      Implement the SwiftUI view "${viewSpec.name}" (${viewSpec.type}).

      Plan section:
      ${viewSpec.planSection}

      Requirements:
      - Read ios-26-app/SKILL.md for current conventions
      - Read ios-26-app/liquid-glass.md if this involves navigation chrome
      - Read ios-26-app/components.md for component patterns
      - Use @Observable (never ObservableObject)
      - Use .glassEffect() for navigation-layer elements only
      - Support Dynamic Type (all 12 size categories)
      - Include #Preview block
      - Target iOS 26 (@available(iOS 26.0, *))
      - Follow the 80/20 color rule (ios-26-app/typography-color.md)

      Write the view to the path specified in the plan section.
    `,
    options: {
      maxTurns: 15,
      cwd: projectDir,
      allowedTools: ["Read", "Write", "Edit", "Glob", "Grep"],
      model: "claude-sonnet-4-6",
      appendSystemPrompt: "You are implementing iOS 26 SwiftUI views. Follow ios-26-app conventions exactly.",
    },
  })) {
    // Stream processing
  }
}
```

---

## 2. Integration with tvos-26-app

The `tvos-26-app` skill covers Apple TV apps with focus-driven interaction, Siri Remote input, 10-foot UI, Top Shelf extensions, parallax effects, and tvOS-specific Liquid Glass.

### Key constraint: NO Apple Intelligence on tvOS

tvOS lacks Foundation Models, Apple Intelligence, Writing Tools, Image Playground, SpeechAnalyzer, and all on-device AI frameworks. Every AI capability on Apple TV must come from the Claude API via network. This is the single most important architectural difference from iOS when building cross-platform agent systems.

```
iOS app:  Foundation Models (on-device) + Claude API (cloud) -- hybrid
tvOS app: Claude API (cloud) only -- all AI is network-dependent
```

Consequence for agent design: any agent building a cross-platform feature with AI must generate **two code paths** -- one that uses Foundation Models for iOS and one that uses the Claude API for tvOS. The agent cannot assume on-device AI availability.

### What IS shared between iOS and tvOS

Both platforms share these foundations (agents can generate shared code):
- Swift 6.2 with strict concurrency (`@Sendable`, actor isolation)
- `@Observable` / `@State` / `@Binding` / `@Environment` state management
- SwiftData `@Model` + `@Query` with model inheritance
- `async/await`, `TaskGroup`, structured concurrency
- URLSession async/await networking
- Liquid Glass APIs (`.glassEffect()`, `.buttonStyle(.glass)`)
- SF Symbols 6 with all rendering modes
- NavigationStack / NavigationSplitView
- Swift Charts (2D)

### What tvOS does NOT have (agents must never generate)

- Camera / PhotoKit / Vision / ARKit / RealityKit
- CoreLocation / MapKit / HealthKit / NearbyInteraction
- WidgetKit / Live Activities / Dynamic Island
- UIFeedbackGenerator / Haptics (no Taptic Engine on Apple TV)
- Multi-touch gestures / Drag and drop
- Notifications / badges
- SpeechAnalyzer / Image Playground / Writing Tools API

An agent generating code for tvOS must have these exclusions in its system prompt or `appendSystemPrompt`. A PreToolUse hook can enforce this:

```bash
#!/bin/bash
# .claude/hooks/pre-tool-use/tvos-framework-guard.sh
# Blocks writes containing iOS-only framework imports in tvOS targets

if [[ "$TOOL_NAME" == "Write" || "$TOOL_NAME" == "Edit" ]]; then
  BLOCKED_IMPORTS="FoundationModels|HealthKit|CoreLocation|ARKit|WidgetKit|SpeechAnalyzer|ImagePlayground"
  if echo "$TOOL_INPUT" | grep -qE "import ($BLOCKED_IMPORTS)" && echo "$TOOL_INPUT" | grep -q "tvOS"; then
    echo "BLOCKED: Attempted to import iOS-only framework in tvOS target" >&2
    exit 2
  fi
fi
exit 0
```

### Agent-assisted tvOS development workflow

```
Step 1: /architect Phase 1
  Reads tvos-26-app/SKILL.md to identify reference files.
  For a content browse screen: reads references/focus-system.md, references/layout-10ft.md,
    references/navigation-patterns.md.
  For media playback: reads references/media-playback.md.
  Maps focus flow graph before any code.

Step 2: /architect Phase 2
  Designs focus navigation graph with explicit focus paths:
    - Where does focus start? (.prefersDefaultFocus())
    - How does focus flow between shelves? (.focusSection())
    - Are there focus traps? (UIFocusGuide for gaps)
  Layout uses tvOS constraints: 1920x1080pt, 60pt safe area, 29pt minimum body text.

Step 3: Parallel worker agents
  Worker 1 (views): tvOS SwiftUI views with focus flow per references/tvos-swiftui.md
  Worker 2 (focus): Focus system integration per references/focus-system.md
  Worker 3 (data): Shared data layer (@Observable, SwiftData)
  Worker 4 (media): AVPlayerViewController customization per references/media-playback.md

Step 4: Test agent
  Uses Bash: xcodebuild test -scheme <TVScheme> -destination 'platform=tvOS Simulator,name=Apple TV'
  Focus testing: verifies focus never gets stuck, .onExitCommand handled everywhere
```

### Cross-platform iOS + tvOS orchestration

When building features for both platforms, use hierarchical sub-orchestrators:

```typescript
import { query } from "@anthropic-ai/claude-code";

async function crossPlatformOrchestrator(
  featureSpec: string,
  projectDir: string
): Promise<void> {
  // Phase 1: Shared data layer (runs first -- both platforms depend on it)
  await runWorker({
    name: "shared-data",
    prompt: `
      Implement the shared data layer for: ${featureSpec}
      - SwiftData @Model entities (shared between iOS and tvOS)
      - @Observable manager class with async/await networking
      - Use Swift 6.2 @Sendable for cross-isolation safety
      - Write to Sources/Shared/Models/ and Sources/Shared/Managers/
    `,
    tools: ["Read", "Write", "Edit", "Glob", "Grep"],
    cwd: projectDir,
  });

  // Phase 2: Platform-specific UI (runs in parallel -- independent)
  await Promise.all([
    runWorker({
      name: "ios-ui",
      prompt: `
        Implement the iOS 26 UI for: ${featureSpec}
        - Read ios-26-app/SKILL.md for conventions
        - Use Liquid Glass for navigation chrome
        - Use Foundation Models for on-device AI where appropriate
        - Import shared models from Sources/Shared/
        - Write to Sources/iOS/Features/<FeatureName>/
      `,
      tools: ["Read", "Write", "Edit", "Glob", "Grep"],
      cwd: projectDir,
    }),
    runWorker({
      name: "tvos-ui",
      prompt: `
        Implement the tvOS 26 UI for: ${featureSpec}
        - Read tvos-26-app/SKILL.md for conventions
        - Design focus flow before coding (references/focus-system.md)
        - Use 1920x1080pt layout with 60pt safe area (references/layout-10ft.md)
        - NO Foundation Models, NO HealthKit, NO Camera -- tvOS only
        - Import shared models from Sources/Shared/
        - Write to Sources/tvOS/Features/<FeatureName>/
      `,
      tools: ["Read", "Write", "Edit", "Glob", "Grep"],
      cwd: projectDir,
    }),
  ]);
}

async function runWorker(config: {
  name: string; prompt: string; tools: string[]; cwd: string;
}): Promise<void> {
  for await (const msg of query({
    prompt: config.prompt,
    options: {
      maxTurns: 20,
      cwd: config.cwd,
      allowedTools: config.tools,
      model: "claude-sonnet-4-6",
      appendSystemPrompt: `You are the "${config.name}" worker agent.`,
    },
  })) {
    if (msg.type === "result" && msg.subtype === "error_max_turns") {
      console.warn(`[${config.name}] Hit max turns`);
    }
  }
}
```

---

## 3. Integration with web-frontend

The `web-frontend` skill covers Next.js 15+, React 19+, Tailwind CSS v4+, and TypeScript. Server Components by default, `use()` for promises, `useActionState`/`useFormStatus` for forms, CSS-first `@theme` configuration.

### Claude API in Next.js -- three patterns

#### Pattern 1: Route Handler (API endpoint)

For client components that need to call Claude:

```typescript
// app/api/chat/route.ts
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic(); // reads ANTHROPIC_API_KEY from env

export async function POST(req: Request) {
  const { message } = await req.json();

  const response = await client.messages.create({
    model: "claude-sonnet-4-6",
    max_tokens: 1024,
    messages: [{ role: "user", content: message }],
  });

  const text = response.content[0].type === "text" ? response.content[0].text : "";
  return Response.json({ content: text });
}
```

#### Pattern 2: Streaming with Vercel AI SDK

For real-time chat UIs:

```typescript
// app/api/chat/route.ts
import { createAnthropic } from "@ai-sdk/anthropic";
import { streamText } from "ai";

const anthropic = createAnthropic(); // reads ANTHROPIC_API_KEY from env

export async function POST(req: Request) {
  const { messages } = await req.json();
  const result = streamText({
    model: anthropic("claude-sonnet-4-6"),
    messages,
  });
  return result.toDataStreamResponse();
}
```

```typescript
// app/chat/page.tsx
"use client";
import { useChat } from "@ai-sdk/react";

export default function ChatPage() {
  const { messages, input, handleInputChange, handleSubmit, isLoading } = useChat({
    api: "/api/chat",
  });

  return (
    <div className="flex flex-col gap-4 p-4">
      <div className="flex-1 space-y-4 overflow-y-auto">
        {messages.map((m) => (
          <div
            key={m.id}
            className={m.role === "user" ? "ml-auto max-w-prose text-right" : "max-w-prose"}
          >
            {m.content}
          </div>
        ))}
      </div>
      <form onSubmit={handleSubmit} className="flex gap-2">
        <input
          value={input}
          onChange={handleInputChange}
          disabled={isLoading}
          className="flex-1 rounded-lg border px-4 py-2"
          placeholder="Ask anything..."
        />
        <button type="submit" disabled={isLoading} className="rounded-lg bg-primary px-4 py-2 text-white">
          Send
        </button>
      </form>
    </div>
  );
}
```

#### Pattern 3: Server Components with direct AI calls (no 'use client')

For pages that render AI-generated content at request time without client JS:

```typescript
// app/summary/page.tsx  (Server Component -- zero client JS)
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

export default async function SummaryPage({
  searchParams,
}: {
  searchParams: Promise<{ url: string }>;
}) {
  const { url } = await searchParams;
  const pageContent = await fetch(url).then((r) => r.text());

  const summary = await client.messages.create({
    model: "claude-haiku-4-5",
    max_tokens: 512,
    messages: [{ role: "user", content: `Summarize this web page concisely:\n\n${pageContent.slice(0, 8000)}` }],
  });

  const text = summary.content[0].type === "text" ? summary.content[0].text : "";

  return (
    <article className="prose mx-auto max-w-2xl py-8">
      <h1>Summary</h1>
      <p>{text}</p>
    </article>
  );
}
```

#### Pattern 4: Server Actions calling Claude

For form submissions that trigger AI analysis:

```typescript
// app/actions/analyze.ts
"use server";
import Anthropic from "@anthropic-ai/sdk";
import { z } from "zod";

const client = new Anthropic();

const AnalyzeInput = z.object({
  code: z.string().min(1).max(50000),
  language: z.enum(["typescript", "python", "swift", "go"]),
});

export async function analyzeCode(formData: FormData) {
  const input = AnalyzeInput.parse({
    code: formData.get("code"),
    language: formData.get("language"),
  });

  const response = await client.messages.create({
    model: "claude-sonnet-4-6",
    max_tokens: 2048,
    messages: [{
      role: "user",
      content: `Review this ${input.language} code for security vulnerabilities, performance issues, and best practice violations. Be specific with line numbers.\n\n\`\`\`${input.language}\n${input.code}\n\`\`\``,
    }],
  });

  return response.content[0].type === "text" ? response.content[0].text : "";
}
```

### Figma MCP to Tailwind v4 design tokens

One of the highest-value cross-skill workflows: extract Figma styles and generate Tailwind v4 `@theme` CSS. This bridges `mcp-consuming.md` (Figma MCP) with `web-frontend/modern-css.md` (`@theme` syntax).

```typescript
import { query } from "@anthropic-ai/claude-code";

async function syncFigmaToTailwind(figmaFileKey: string, outputPath: string): Promise<void> {
  for await (const msg of query({
    prompt: `
      Sync design tokens from Figma file "${figmaFileKey}" to Tailwind v4 @theme CSS.

      Steps:
      1. Use mcp__figma__get_styles to extract all styles from the file
      2. Use mcp__figma__get_components to identify the component library structure
      3. Read the existing CSS file at "${outputPath}" to preserve manual additions
      4. Map Figma tokens to Tailwind v4 @theme:
         - Color styles -> --color-* (use oklch values for perceptual uniformity)
         - Text styles -> --text-*, --font-*, --leading-*, --tracking-*
         - Effect styles (shadows) -> --shadow-*
         - Spacing from auto-layout -> --spacing-* if systematic
      5. Generate @theme block following web-frontend/modern-css.md conventions
      6. Write to "${outputPath}", preserving any @variant or @utility blocks

      Use oklch() color format per web-frontend/typography-color.md guidance.
      Follow the Tailwind v4 CSS-first config pattern -- no tailwind.config.js.
    `,
    options: {
      maxTurns: 15,
      allowedTools: [
        "Read", "Write", "Edit",
        "mcp__figma__get_styles",
        "mcp__figma__get_components",
        "mcp__figma__get_node",
      ],
      model: "claude-sonnet-4-6",
    },
  })) {
    // Stream processing
  }
}
```

Generated output example:

```css
/* src/styles/tokens.css -- auto-generated from Figma, do not edit manually */
@import "tailwindcss";

@theme {
  --color-primary: oklch(0.65 0.25 265);
  --color-primary-light: oklch(0.80 0.15 265);
  --color-secondary: oklch(0.55 0.20 145);
  --color-surface: oklch(0.98 0.005 265);
  --color-surface-dark: oklch(0.15 0.02 265);

  --font-sans: "Inter", ui-sans-serif, system-ui, sans-serif;
  --font-heading: "Cal Sans", var(--font-sans);

  --text-xs: 0.75rem;
  --text-sm: 0.875rem;
  --text-base: 1rem;
  --text-lg: 1.125rem;
  --text-xl: 1.25rem;
  --text-2xl: 1.5rem;

  --shadow-sm: 0 1px 2px oklch(0 0 0 / 0.05);
  --shadow-md: 0 4px 6px oklch(0 0 0 / 0.07), 0 2px 4px oklch(0 0 0 / 0.06);
}
```

### Vercel MCP deployment workflow

Integrates `web-frontend` testing with `mcp-consuming.md` Vercel tools for automated deploy:

```typescript
import { query } from "@anthropic-ai/claude-code";

async function webDeployPipeline(projectDir: string, owner: string, repo: string): Promise<void> {
  for await (const msg of query({
    prompt: `
      Run the full web-frontend deploy pipeline.

      Steps:
      1. Run tests: Bash("cd ${projectDir} && npm test -- --run")
         If tests fail, STOP and report failures.
      2. Run build: Bash("cd ${projectDir} && npm run build")
         If build fails, STOP and report errors.
      3. Run Lighthouse CI: Bash("cd ${projectDir} && npx @lhci/cli autorun") (optional, skip if not configured)
      4. Deploy to Vercel:
         a. mcp__vercel__list_projects to find project ID
         b. mcp__vercel__create_deployment with target "preview"
         c. Wait, then mcp__vercel__list_deployments to check status
         d. If ERROR: mcp__vercel__get_deployment_logs, report failure
         e. If READY: report preview URL
      5. Post deployment URL to GitHub:
         a. mcp__github__list_pull_requests for the current branch on ${owner}/${repo}
         b. mcp__github__add_issue_comment with the preview URL and build status

      Follow web-frontend/performance.md Core Web Vitals thresholds:
      LCP < 2.5s, CLS < 0.1, INP < 200ms.
    `,
    options: {
      maxTurns: 25,
      cwd: projectDir,
      allowedTools: [
        "Bash", "Read",
        "mcp__vercel__list_projects",
        "mcp__vercel__create_deployment",
        "mcp__vercel__list_deployments",
        "mcp__vercel__get_deployment_logs",
        "mcp__github__list_pull_requests",
        "mcp__github__add_issue_comment",
      ],
      model: "claude-sonnet-4-6",
    },
  })) {
    // Stream processing
  }
}
```

### Agent-assisted web development workflow

```
Step 1: /architect Phase 1
  Reads web-frontend/SKILL.md to identify reference files.
  For a dashboard page: reads components-ui-patterns.md, layout-responsive.md, engineering.md.
  For a form: reads forms-validation.md, auth-security.md.
  GitHub MCP: reads related issues.
  Figma MCP: gets design specs.

Step 2: /architect Phase 2
  Designs component hierarchy using web-frontend patterns:
    - Server Components by default ('use client' only when needed)
    - Tailwind v4 @theme tokens (not arbitrary values)
    - Zod validation at every boundary
    - useActionState + useFormStatus for form states
  Cross-cutting: accessibility (WCAG AA), performance (Core Web Vitals), security (CSP, CSRF).

Step 3: Parallel worker agents
  Worker 1 (pages): Server Components, layout.tsx, page.tsx per nextjs.md
  Worker 2 (components): Client Components with Tailwind v4 per modern-css.md
  Worker 3 (API): Route handlers + Server Actions with Zod per forms-validation.md
  Worker 4 (tests): Vitest units + Playwright E2E per testing.md

Step 4: Test + deploy agent
  npm test -> npm build -> Vercel MCP deploy -> GitHub MCP PR comment
```

---

## 4. Integration with architect

The `architect` skill provides a 3-phase workflow: **Context** (Phase 1) -> **Plan** (Phase 2) -> **Execute** (Phase 3). It is the primary human-in-the-loop mechanism and the natural entry point for any non-trivial feature work.

### Phase 1 (Context) + agents

Phase 1 gathers requirements, discovers patterns, and maps the blast radius. Agent integration:

- **Parallel Explore subagents**: Phase 1 says "use parallel Explore agents for independent areas (max 3 concurrent)." With the Agent SDK, these are `query()` calls with `allowedTools: ["Read", "Glob", "Grep"]` running in `Promise.all()`.
- **GitHub MCP for context**: Read related issues (`mcp__github__create_issue`... no -- `mcp__github__get_file_contents` to read issue templates or use `Bash("gh issue view <number>")` for issue context).
- **Domain skill routing**: Phase 1 explicitly routes to domain skills based on platform:
  - iOS/iPadOS/Swift/SwiftUI -> read `ios-26-app/SKILL.md`, note relevant reference files
  - tvOS/Apple TV -> read `tvos-26-app/SKILL.md`, note relevant reference files
  - Web -> read `web-frontend/SKILL.md`, note relevant reference files
  - Cross-platform -> read multiple skills, note shared vs platform-specific concerns

```typescript
// Programmatic Phase 1 context gathering with parallel agents
async function gatherContext(task: string, platform: "ios" | "tvos" | "web" | "cross") {
  const skillRefs: Record<string, string[]> = {
    ios: ["ios-26-app/SKILL.md"],
    tvos: ["tvos-26-app/SKILL.md"],
    web: ["web-frontend/SKILL.md"],
    cross: ["ios-26-app/SKILL.md", "tvos-26-app/SKILL.md", "web-frontend/SKILL.md"],
  };

  // Parallel exploration agents
  const [codePatterns, issueContext, designContext] = await Promise.all([
    // Agent 1: Search codebase for existing patterns
    collectText(query({
      prompt: `Search the codebase for patterns related to: ${task}. Find naming conventions, file organization, reusable utilities. Read 2-3 representative examples in full.`,
      options: { maxTurns: 10, allowedTools: ["Read", "Glob", "Grep"], model: "claude-haiku-4-5" },
    })),
    // Agent 2: GitHub issue context
    collectText(query({
      prompt: `Use GitHub MCP to find issues related to: ${task}. Read their descriptions and comments for requirements context.`,
      options: { maxTurns: 5, allowedTools: ["Bash", "mcp__github__list_commits"], model: "claude-haiku-4-5" },
    })),
    // Agent 3: Read domain skill references
    collectText(query({
      prompt: `Read these skill reference files and summarize the conventions relevant to "${task}": ${skillRefs[platform].join(", ")}`,
      options: { maxTurns: 5, allowedTools: ["Read", "Glob"], model: "claude-haiku-4-5" },
    })),
  ]);

  return { codePatterns, issueContext, designContext };
}

async function collectText(gen: AsyncGenerator<any>): Promise<string> {
  const texts: string[] = [];
  for await (const msg of gen) {
    if (msg.type === "assistant") {
      texts.push(...msg.message.content.filter((b: any) => b.type === "text").map((b: any) => b.text));
    }
  }
  return texts.join("\n");
}
```

### Phase 2 (Plan) + agents

Phase 2 designs the step-by-step plan. The plan doc (`docs/plans/YYYY-MM/<slug>.md`) IS the spec that drives agentic execution:

- **Plan doc = TaskSpec**: The `<slug>.md` file from architect Phase 2 contains everything a `TaskSpec` (from `agentic-workflows.md` section 1) needs -- inputs, outputs, steps, constraints, acceptance criteria. An orchestrator can parse the plan doc and decompose it into worker tasks.
- **Cross-cutting concerns checklist**: Phase 2's checklist (security, performance, accessibility, concurrency, memory, API contracts, CI/CD, docs, cross-platform, i18n) maps directly to the agent security and governance patterns in `security-trust.md`.
- **15-step decision framework**: SKILL.md's 15-step decision framework maps to Phase 2's design process. Step 15 ("Map to existing skills") explicitly says to check domain skills before building infrastructure.
- **Domain reference reading**: Phase 2 says "if Phase 1 identified domain skill references, read them NOW to inform step design." This is when agents should load platform-specific reference files -- not eagerly at startup (context budget).

### Phase 3 (Execute) + agents

Phase 3 implements the approved plan. This is where the Agent SDK shines:

- **Convert plan to workers**: Each plan step (from `<slug>.md`) becomes a worker agent task. Steps that touch <= 3-4 files are ideal worker units.
- **Sequential dependency ordering**: Phase 3 follows the dependency chain from Phase 2. Steps that modify shared interfaces run before steps that consume them. The orchestrator respects this ordering.
- **Verification per step**: Each plan step has a `**Verify**` field. The test agent runs this after each worker completes.
- **Scope expansion protocol**: If a worker discovers the plan is insufficient, it stops. The orchestrator re-enters plan mode (Phase 2 scope expansion), gets approval, then resumes.
- **Context management**: Phase transitions use `/clear` to free context. With agents, this translates to separate `query()` calls -- each worker starts with a fresh context window.

```typescript
import { query } from "@anthropic-ai/claude-code";

interface PlanStep {
  number: number;
  description: string;
  files: string[];
  pattern: string;
  changes: string[];
  verify: string;
}

async function executeArchitectPlan(
  steps: PlanStep[],
  projectDir: string
): Promise<void> {
  for (const step of steps) {
    console.log(`[Step ${step.number}] ${step.description}`);

    // Execute the step
    for await (const msg of query({
      prompt: `
        Execute step ${step.number} of the approved plan.

        Description: ${step.description}
        Files: ${step.files.join(", ")}
        Pattern to follow: ${step.pattern}
        Changes:
        ${step.changes.map((c) => `- ${c}`).join("\n")}

        After making changes, verify: ${step.verify}

        If verification fails, fix the issue. If you cannot fix it in 2 attempts,
        STOP and report the failure -- do not proceed to the next step.
      `,
      options: {
        maxTurns: 15,
        cwd: projectDir,
        allowedTools: ["Read", "Write", "Edit", "Glob", "Grep", "Bash"],
        model: "claude-sonnet-4-6",
      },
    })) {
      if (msg.type === "result" && msg.subtype !== "success") {
        throw new Error(`Step ${step.number} failed: ${msg.subtype}`);
      }
    }

    // Commit after each step (per Phase 3 rules)
    for await (const msg of query({
      prompt: `Run: git add -A && git commit -m "${step.description}"`,
      options: {
        maxTurns: 3,
        cwd: projectDir,
        allowedTools: ["Bash"],
        model: "claude-haiku-4-5",
      },
    })) {
      // commit complete
    }
  }
}
```

### Full architect + agents pipeline

```
/architect Phase 1 -> Context gathered, domain skill refs identified
  |
  v  (user approves ## Context, /clear)
/architect Phase 2 -> Plan designed with cross-cutting concerns
  |
  v  (user approves ## Plan via ExitPlanMode, /clear)
Convert <slug>.md steps to worker tasks
  |
  v
Orchestrator runs workers (Agent SDK query() calls):
  Sequential for dependent steps, parallel for independent steps
  Each worker loads only its relevant platform skill refs
  |
  v
Test agent validates each step's verify criteria
  |
  v
Final commit + GitHub MCP creates PR
```

---

## 5. MCP Servers -- Cross-Cutting Integration

MCP servers bridge all skills. Configuration lives in `mcp.json` (see `mcp-consuming.md`). The integration patterns below show how each MCP server connects to multiple domain skills simultaneously.

### Figma MCP across platforms

```
mcp__figma__get_file / get_node / get_components / get_styles
    |
    +-- web-frontend: Extract tokens -> Tailwind v4 @theme CSS
    |   Tools: mcp__figma__get_styles -> generate @theme {} block
    |   Reference: web-frontend/modern-css.md, web-frontend/typography-color.md
    |
    +-- ios-26-app: Extract component specs -> SwiftUI views
    |   Tools: mcp__figma__get_node -> generate SwiftUI with Liquid Glass
    |   Reference: ios-26-app/liquid-glass.md, ios-26-app/components.md
    |   Maps: Figma auto-layout -> HStack/VStack, Figma text styles -> Dynamic Type sizes,
    |          Figma fills -> SwiftUI Color assets, Figma shadows -> .shadow()
    |
    +-- tvos-26-app: Extract component specs -> tvOS SwiftUI views
        Tools: mcp__figma__get_node -> generate tvOS views with focus states
        Reference: tvos-26-app/references/layout-10ft.md (1920x1080, 60pt safe area)
        Maps: Figma poster cards -> TVPosterView, Figma shelves -> horizontal ScrollView,
               Figma auto-layout -> focusSection() boundaries
```

When building for multiple platforms from the same Figma file, the orchestrator extracts once and fans out:

```typescript
import { query } from "@anthropic-ai/claude-code";

async function figmaToAllPlatforms(
  figmaFileKey: string,
  componentNodeId: string,
  projectDir: string
): Promise<void> {
  // Step 1: Extract component spec once (shared)
  const componentSpec = await collectText(query({
    prompt: `Extract the full component spec from Figma node "${componentNodeId}" in file "${figmaFileKey}". Use mcp__figma__get_node and mcp__figma__export_node_as_image. Output a structured description: layout, colors, typography, spacing, shadows, states.`,
    options: {
      maxTurns: 10,
      allowedTools: ["mcp__figma__get_node", "mcp__figma__get_styles", "mcp__figma__export_node_as_image"],
      model: "claude-sonnet-4-6",
    },
  }));

  // Step 2: Generate platform implementations in parallel
  await Promise.all([
    // Web: React + Tailwind v4
    query({
      prompt: `Generate a React component from this Figma spec:\n${componentSpec}\n\nUse Tailwind v4 utilities, Server Component by default, semantic HTML, WCAG AA contrast. Write to ${projectDir}/src/components/`,
      options: {
        maxTurns: 10, cwd: projectDir,
        allowedTools: ["Read", "Write", "Edit"],
        model: "claude-sonnet-4-6",
        appendSystemPrompt: "Follow web-frontend/SKILL.md conventions.",
      },
    }),
    // iOS: SwiftUI + Liquid Glass
    query({
      prompt: `Generate a SwiftUI view from this Figma spec:\n${componentSpec}\n\nUse @Observable, Liquid Glass where appropriate, Dynamic Type, #Preview. Write to ${projectDir}/Sources/iOS/Components/`,
      options: {
        maxTurns: 10, cwd: projectDir,
        allowedTools: ["Read", "Write", "Edit"],
        model: "claude-sonnet-4-6",
        appendSystemPrompt: "Follow ios-26-app/SKILL.md conventions.",
      },
    }),
    // tvOS: SwiftUI + Focus
    query({
      prompt: `Generate a tvOS SwiftUI view from this Figma spec:\n${componentSpec}\n\nDesign focus flow, use 29pt+ body text, 60pt safe area. Write to ${projectDir}/Sources/tvOS/Components/`,
      options: {
        maxTurns: 10, cwd: projectDir,
        allowedTools: ["Read", "Write", "Edit"],
        model: "claude-sonnet-4-6",
        appendSystemPrompt: "Follow tvos-26-app/SKILL.md conventions. NO iOS-only frameworks.",
      },
    }),
  ].map(async (gen) => { for await (const _ of gen) {} }));
}
```

### GitHub MCP across all skills

Every platform benefits from GitHub MCP integration. Common patterns:

| Workflow | Tools used | Which skills benefit |
|----------|-----------|---------------------|
| Read issue for requirements | `mcp__github__get_file_contents`, `Bash("gh issue view")` | All -- Phase 1 context |
| Create PR after implementation | `mcp__github__create_pull_request`, `mcp__github__push_files` | All -- Phase 3 completion |
| CI failure triage | `mcp__github__list_commits`, `mcp__github__create_issue` | All -- background agent |
| Post deployment URL | `mcp__github__add_issue_comment` | web-frontend (Vercel deploy) |
| Search for patterns | `mcp__github__search_repositories` | All -- discovery |

### Vercel MCP for web-frontend

Natural deployment target for Next.js projects. The deploy-on-green workflow (`mcp-consuming.md` section 6) is the canonical cross-skill pattern:

```
web-frontend agent -> npm test -> npm build -> mcp__vercel__create_deployment -> verify health
                                                                                      |
                                                                mcp__vercel__get_deployment_logs (on failure)
                                                                                      |
                                                          mcp__github__add_issue_comment (deploy URL or failure)
```

Privilege scoping for Vercel tools (from `mcp-consuming.md`):

```typescript
// Read-only monitoring -- safest, use for status checks
const monitorTools = ["mcp__vercel__list_projects", "mcp__vercel__list_deployments", "mcp__vercel__get_deployment_logs"];

// Deploy capability -- moderate privilege, for CI/CD agents
const deployTools = [...monitorTools, "mcp__vercel__create_deployment"];

// Full management -- high privilege, require human approval gate
const fullTools = [...deployTools, "mcp__vercel__create_env", "mcp__vercel__list_domains"];
```

### Stripe MCP for monetization (iOS + web)

Both `ios-26-app` (StoreKit 2) and `web-frontend` (Stripe Checkout) deal with monetization. The Stripe MCP creates products/prices that map to both platforms:

```typescript
// Create Stripe products that map to both StoreKit and web checkout
async function setupCrossPlatformPricing(pricingSpecPath: string): Promise<void> {
  for await (const msg of query({
    prompt: `
      Read the pricing spec at "${pricingSpecPath}" and create Stripe products/prices.
      After creation, output a mapping file that includes:
      1. Stripe product/price IDs (for web-frontend Checkout integration)
      2. Equivalent StoreKit 2 product identifiers (for ios-26-app IAP configuration)

      CRITICAL: Verify the Stripe key starts with sk_test_. STOP if live key detected.
    `,
    options: {
      maxTurns: 20,
      allowedTools: [
        "Read", "Write",
        "mcp__stripe__create_product",
        "mcp__stripe__create_price",
      ],
      model: "claude-sonnet-4-6",
    },
  })) {
    // Stream
  }
}
```

---

## 6. Cross-Skill Orchestration -- End-to-End Example

Building a "Wishlist" feature for an app with iOS, tvOS, and web clients. All five skills working together.

### Prerequisites

```jsonc
// .claude/mcp.json -- all servers needed
{
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic-ai/github-mcp-server@1.2.0"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    },
    "figma": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic-ai/figma-mcp-server@1.0.3"],
      "env": { "FIGMA_ACCESS_TOKEN": "${FIGMA_ACCESS_TOKEN}" }
    },
    "vercel": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic-ai/vercel-mcp-server@1.1.0"],
      "env": { "VERCEL_TOKEN": "${VERCEL_TOKEN}" }
    }
  }
}
```

### The pipeline

```
User: "Add wishlist feature -- users can save items and sync across iOS, tvOS, and web"

=== PHASE 1: /architect Context ===

Orchestrator reads:
  - ios-26-app/SKILL.md -> identifies: engineering.md (SwiftData, @Observable),
    components.md (lists, buttons), liquid-glass.md (navigation)
  - tvos-26-app/SKILL.md -> identifies: references/focus-system.md,
    references/layout-10ft.md, references/navigation-patterns.md
  - web-frontend/SKILL.md -> identifies: engineering.md (React Query, Zustand),
    nextjs.md (Server Actions), forms-validation.md (Zod)
  - claude-code-ai-agents/SKILL.md -> agentic-workflows.md (pipeline design),
    agent-sdk-orchestration.md (worker pattern)

Parallel Explore agents (3 concurrent):
  Agent 1: Search codebase for existing data models, API patterns, sync logic
  Agent 2: GitHub MCP -- read wishlist issue, related PRs, prior discussion
  Agent 3: Figma MCP -- get wishlist designs (mcp__figma__get_components)

Output: ## Context section in docs/plans/2026-04/add-wishlist.md
Gate: User approves context -> /clear

=== PHASE 2: /architect Plan ===

Read domain skill references identified in Phase 1:
  - ios-26-app/engineering.md (SwiftData sync, @Observable patterns)
  - tvos-26-app/references/focus-system.md (focus flow for wishlist grid)
  - web-frontend/nextjs.md (Server Actions for mutations)

Design steps:
  Step 1: Shared API layer (app/api/wishlist/route.ts -- Zod validation)
  Step 2: Shared data models (Sources/Shared/Models/WishlistItem.swift -- SwiftData @Model)
  Step 3: iOS views (Sources/iOS/Features/Wishlist/ -- SwiftUI, Liquid Glass, @Observable)
  Step 4: tvOS views (Sources/tvOS/Features/Wishlist/ -- focus grid, 10ft layout)
  Step 5: Web pages (app/wishlist/ -- RSC, Tailwind v4, React Query)
  Step 6: CloudKit sync (Sources/Shared/Sync/ -- CKRecord mapping)
  Step 7: Tests (all platforms)
  Step 8: Figma verification (compare implementation to design)

Cross-cutting concerns:
  - Accessibility: iOS (Dynamic Type, VoiceOver), tvOS (focus states, VoiceOver),
    web (WCAG AA, keyboard nav, screen reader)
  - Concurrency: Swift 6.2 @Sendable for shared models, async/await everywhere
  - Performance: iOS (lazy loading), tvOS (focus performance), web (Core Web Vitals)
  - Security: Zod on API, input validation, no PII in logs

Output: ## Plan section in docs/plans/2026-04/add-wishlist.md
Gate: User approves plan via ExitPlanMode -> /clear

=== PHASE 3: Parallel Execution ===

Step 1 (API): Worker agent
  allowedTools: [Read, Write, Edit, Glob, Grep]
  appendSystemPrompt: "Follow web-frontend/SKILL.md. Validate with Zod."
  Output: app/api/wishlist/route.ts (GET, POST, DELETE)

Step 2 (Swift models): Worker agent
  allowedTools: [Read, Write, Edit, Glob, Grep]
  appendSystemPrompt: "Follow ios-26-app/engineering.md. SwiftData @Model, @Sendable."
  Output: Sources/Shared/Models/WishlistItem.swift

Steps 3-5 run in parallel (independent, different directories):

  Step 3 (iOS UI): Worker agent
    appendSystemPrompt: "Follow ios-26-app/SKILL.md. Liquid Glass, @Observable, Dynamic Type."
    Output: Sources/iOS/Features/Wishlist/WishlistView.swift
            Sources/iOS/Features/Wishlist/WishlistManager.swift

  Step 4 (tvOS UI): Worker agent
    appendSystemPrompt: "Follow tvos-26-app/SKILL.md. Focus flow, 29pt+ text, no iOS-only frameworks."
    Output: Sources/tvOS/Features/Wishlist/WishlistGridView.swift
            Sources/tvOS/Features/Wishlist/WishlistTVManager.swift

  Step 5 (Web UI): Worker agent
    appendSystemPrompt: "Follow web-frontend/SKILL.md. RSC default, Tailwind v4, Zod."
    Output: app/wishlist/page.tsx (Server Component)
            app/wishlist/WishlistCard.tsx (Client Component for interactions)
            app/wishlist/actions.ts (Server Actions)

Step 6 (Sync): Worker agent (depends on Step 2)
  Output: Sources/Shared/Sync/WishlistSyncManager.swift

Step 7 (Tests): Parallel test agents
  iOS: xcodebuild test (Swift Testing @Test + #expect)
  tvOS: xcodebuild test (focus navigation tests)
  Web: npm test (Vitest + Playwright)

Step 8 (Verification): Agent compares Figma designs to implementation
  mcp__figma__export_node_as_image for each wishlist screen
  Visual comparison report

=== POST-EXECUTION ===

GitHub MCP: mcp__github__create_pull_request
  Title: "feat: add cross-platform wishlist with sync"
  Body: links to plan doc, test results, Figma comparison

Vercel MCP: mcp__vercel__create_deployment (preview for web)
  Posts preview URL to PR via mcp__github__add_issue_comment
```

### Orchestrator code (simplified)

```typescript
import { query, type SDKMessage } from "@anthropic-ai/claude-code";

interface WorkerConfig {
  name: string;
  prompt: string;
  tools: string[];
  systemPromptSuffix: string;
  dependsOn?: string[];
}

async function wishlistPipeline(projectDir: string): Promise<void> {
  const workers: WorkerConfig[] = [
    {
      name: "api",
      prompt: "Implement wishlist API route at app/api/wishlist/route.ts...",
      tools: ["Read", "Write", "Edit", "Glob", "Grep"],
      systemPromptSuffix: "Follow web-frontend/SKILL.md. Validate all inputs with Zod.",
    },
    {
      name: "swift-models",
      prompt: "Implement WishlistItem SwiftData model at Sources/Shared/Models/...",
      tools: ["Read", "Write", "Edit", "Glob", "Grep"],
      systemPromptSuffix: "Follow ios-26-app/engineering.md. Use @Sendable, @Model.",
    },
    {
      name: "ios-ui",
      prompt: "Implement iOS wishlist views at Sources/iOS/Features/Wishlist/...",
      tools: ["Read", "Write", "Edit", "Glob", "Grep"],
      systemPromptSuffix: "Follow ios-26-app/SKILL.md. Liquid Glass, @Observable.",
      dependsOn: ["swift-models"],
    },
    {
      name: "tvos-ui",
      prompt: "Implement tvOS wishlist views at Sources/tvOS/Features/Wishlist/...",
      tools: ["Read", "Write", "Edit", "Glob", "Grep"],
      systemPromptSuffix: "Follow tvos-26-app/SKILL.md. Focus flow, 10ft layout.",
      dependsOn: ["swift-models"],
    },
    {
      name: "web-ui",
      prompt: "Implement web wishlist pages at app/wishlist/...",
      tools: ["Read", "Write", "Edit", "Glob", "Grep"],
      systemPromptSuffix: "Follow web-frontend/SKILL.md. RSC, Tailwind v4.",
      dependsOn: ["api"],
    },
  ];

  // Topological execution: run workers in dependency order
  const completed = new Set<string>();

  while (completed.size < workers.length) {
    const ready = workers.filter(
      (w) => !completed.has(w.name) && (w.dependsOn ?? []).every((d) => completed.has(d))
    );

    if (ready.length === 0) throw new Error("Circular dependency detected");

    // Run all ready workers in parallel
    await Promise.all(
      ready.map(async (worker) => {
        console.log(`[${worker.name}] Starting...`);
        for await (const msg of query({
          prompt: worker.prompt,
          options: {
            maxTurns: 20,
            cwd: projectDir,
            allowedTools: worker.tools,
            model: "claude-sonnet-4-6",
            appendSystemPrompt: worker.systemPromptSuffix,
          },
        })) {
          if (msg.type === "result" && msg.subtype !== "success") {
            console.error(`[${worker.name}] Failed: ${msg.subtype}`);
          }
        }
        completed.add(worker.name);
        console.log(`[${worker.name}] Complete`);
      })
    );
  }

  // Final: test + deploy
  await runTests(projectDir);
  await createPR(projectDir);
}

async function runTests(projectDir: string): Promise<void> {
  for await (const msg of query({
    prompt: `Run all test suites:
      1. Bash("cd ${projectDir} && xcodebuild test -scheme App-iOS -destination 'platform=iOS Simulator,name=iPhone 16'")
      2. Bash("cd ${projectDir} && xcodebuild test -scheme App-tvOS -destination 'platform=tvOS Simulator,name=Apple TV'")
      3. Bash("cd ${projectDir} && npm test -- --run")
      Report pass/fail for each suite.`,
    options: {
      maxTurns: 10,
      cwd: projectDir,
      allowedTools: ["Bash", "Read"],
      model: "claude-sonnet-4-6",
    },
  })) {
    // Stream
  }
}

async function createPR(projectDir: string): Promise<void> {
  for await (const msg of query({
    prompt: `Create a PR for the wishlist feature:
      1. Bash("git add -A && git commit -m 'feat: add cross-platform wishlist with sync'")
      2. Bash("git push -u origin HEAD")
      3. mcp__github__create_pull_request with title, body linking to plan doc, test results
      4. mcp__vercel__create_deployment for web preview
      5. mcp__github__add_issue_comment with preview URL on the PR`,
    options: {
      maxTurns: 15,
      cwd: projectDir,
      allowedTools: [
        "Bash",
        "mcp__github__create_pull_request",
        "mcp__github__add_issue_comment",
        "mcp__vercel__create_deployment",
        "mcp__vercel__list_deployments",
      ],
      model: "claude-sonnet-4-6",
    },
  })) {
    // Stream
  }
}
```

---

## 7. Skill Routing Decision Tree

When a task arrives, determine which skills to load:

```
Task arrives
  |
  +-- Involves iOS/iPadOS/Swift/SwiftUI?
  |     YES -> Load ios-26-app
  |     + Involves on-device AI (Foundation Models, Writing Tools, SpeechAnalyzer)?
  |         YES -> Read ios-26-app/engineering.md (AI sections)
  |
  +-- Involves tvOS/Apple TV?
  |     YES -> Load tvos-26-app
  |     NOTE: NO Apple Intelligence on tvOS -- all AI = Claude API
  |
  +-- Involves web/React/Next.js/Tailwind?
  |     YES -> Load web-frontend
  |
  +-- Involves agent orchestration, MCP, Claude API integration?
  |     YES -> Load claude-code-ai-agents
  |
  +-- Non-trivial (multi-file, new feature, architectural)?
  |     YES -> Load architect, start with /architect Phase 1
  |
  +-- Cross-platform (iOS + tvOS, iOS + web, all three)?
        YES -> Load all relevant platform skills
        + Load claude-code-ai-agents for orchestration
        + Load architect for planning
        Use hierarchical sub-orchestrators per platform
```

### Context budget management

Loading multiple skills consumes context. Budget carefully:

| What | Approximate context cost | Strategy |
|------|-------------------------|----------|
| SKILL.md (each) | 3-5% of window | Always load for relevant platform |
| Reference file (each) | 2-4% of window | Load lazily -- only when the step needs it |
| Plan doc (<slug>.md) | 1-3% of window | Always load in Phase 3 |
| MCP tool schemas | 1-2% per server | Auto-loaded on first tool call |
| Conversation history | 10-40% (grows) | Use /clear between phases, fresh query() per worker |

Rule: keep total loaded context under 60% of window capacity. Reserve 40% for reasoning, tool results, and the actual work. Signs of exhaustion: model contradicts earlier instructions, ignores conventions, gives terse responses.

---

## 8. Anti-Patterns for Cross-Skill Integration

1. **Loading all skill references eagerly** -- Read only the reference files needed for the current step. A view worker does not need `testing.md`; a test worker does not need `liquid-glass.md`.

2. **Using one agent for all platforms** -- A single agent trying to write SwiftUI, tvOS focus code, and React components will exhaust context and confuse conventions. Use one worker per platform.

3. **Assuming tvOS has iOS capabilities** -- No Foundation Models, no Camera, no Location, no Haptics, no Widgets on tvOS. Every cross-platform agent must have tvOS exclusions in its system prompt.

4. **Skipping the architect skill for multi-platform features** -- Multi-platform work is inherently non-trivial. Always use `/architect` Phase 1-2 before spawning platform workers. The plan doc prevents duplicate work and conflicting implementations.

5. **Sharing mutable state between parallel workers** -- Each worker gets its own directory, its own output files, and no shared mutable state with siblings. The orchestrator aggregates results after all workers complete.

6. **Using Claude API where Foundation Models suffice on iOS** -- Simple classification, short summarization, and structured extraction with `@Generable` should use the on-device model for lower latency and offline capability. Reserve Claude API for complex reasoning.

7. **Hardcoding MCP tool names without namespace** -- Always use `mcp__<server>__<tool>` format in `allowedTools`. Unnamespaced names cause silent tool resolution failures.

8. **Giving deploy agents broad MCP permissions** -- A deploy agent needs `create_deployment` and `list_deployments`, not `create_env` (which can overwrite production secrets). Scope `allowedTools` to the minimum set.

9. **Running Figma MCP `get_file` on large design systems** -- Use `get_components` + `get_styles` + targeted `get_node` calls. A full `get_file` on a 200-component design system returns 20MB+ of JSON and destroys the context window.

10. **Not version-pinning MCP server packages** -- Always pin: `@anthropic-ai/github-mcp-server@1.2.0`, not `@anthropic-ai/github-mcp-server`. Unpinned versions can change tool schemas between runs, breaking `allowedTools` lists and agent prompts.
