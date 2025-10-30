# Instruction Design for iOS AI: Writing Prompts That Reflect Your App's Architecture

Lately I've been treating prompts the same way I treat public APIs: versioned, testable, and slightly boring on purpose. When you're shipping an AI feature in an iOS app, "prompt vibes" don't scale—contracts do.

Apple's Foundation Models framework is basically begging us to think this way: it's Swift-native access to an on-device language model with structured output (guided generation) and tool calling (the model can invoke code you expose). And on-device models reward clarity: short, explicit instructions beat essay prompts every time.

So here's how I design instructions that mirror the app's architecture: App Intents define what the app can do system-wide, and Foundation Models instructions define how the model should reason about doing it inside the app. App Intents are also how your actions and content become discoverable via Siri/Shortcuts/Spotlight—especially with Apple Intelligence enhancements.

---

## Treat Prompts as API Contracts (and Name Things Like You Mean It)

If your app already has a clean architecture (feature modules, domain use-cases, DTOs), you can reuse that mental model:

- **App Intents** = public endpoints (your system-facing actions + entities)
- **Tools (Foundation Models)** = internal endpoints (your model-facing functions for data + side effects)
- **Guided generation types** = response DTOs with schema guarantees

And then your "prompt" becomes a contract that says:

*"Here are the allowed calls, the expected output shape, and the error policy."*

### My "contract checklist"

✅ Reference real names from your code: intent names, entity names, tool names  
✅ Say what tools exist and what they're for  
✅ Specify formatting (structured type preferred)  
✅ Define guardrails & failures (what to do when data is missing / tool errors)  
✅ Keep it modular so you can swap a section without rewriting everything

---

## Anchor the Model to Your App Intents and Entities

App Intents are the most "architectural" vocabulary you have, because they define actions and entities the system can reason about. If you're integrating with Siri and Apple Intelligence, Apple even provides assistant schemas via macros like `AppIntent(schema:)`, `AppEntity(schema:)`, and `AppEnum(schema:)` so your intents/entities match patterns the models expect.

Here's Apple's own example style for a schema-conforming intent (Photos domain), just to show what I mean by "names as contracts":

```swift
@AppIntent(schema: .photos.openAsset)
struct OpenAssetIntent: OpenIntent {
    var target: AssetEntity
    
    @MainActor func perform() async throws -> some IntentResult { 
        .result() 
    }
}
```

### What I do in practice

Even if your AI feature is in-app (not Siri), I still reuse those names in instructions:

- "Use OpenAssetIntent when user asks to open a photo."
- "Entities: AssetEntity, fields: id, title."
- "If you need data, call the tool fetchAssetMetadata."

This makes prompts less "English essay" and more "protocol definition".

---

## Compose Instructions Like Modules (Not One Mega Prompt)

Apple explicitly warns that on-device prompting needs to be concise and specific, because the on-device model is smaller with tighter context constraints.

So instead of a single wall-of-text instruction, I write instruction modules:

### A) Role / scope module

```
You are the in-app assistant for BreadBox.
Your job: help the user complete tasks using the app's tools and intents.
Stay within the app's capabilities; do not invent data.
```

### B) Tooling module ("what you can call")

```
Available tools:
- searchBreadDatabase(searchTerm: String, limit: Int) -> [RecipeSummary]
- saveRecipe(name: String, url: URL) -> SaveResult

Use tools when information is needed or an action must occur.
```

Tool calling is designed exactly for this: you provide tools, and the model decides when to call them to ground responses and perform actions.

### C) Output module ("what you must return")

If I can, I skip free-form text and require a structured DTO via guided generation, because it gives strong format guarantees.

```
Return a DraftPlan object.
If a tool fails, return DraftPlan with status="needs_input" and ask one question.
```

### D) Safety + guardrails module ("what you must not do")

This is where I encode practical policies:

- Don't claim you saved something unless the saveRecipe tool succeeded.
- If user requests something outside capabilities, say so and offer an alternative.

(Also: Apple's docs call out that the model may not be suitable for things like code generation and certain reasoning-heavy tasks; I treat that as a product constraint and push those parts into deterministic code or tools.)

---

## Encode Formatting + Error Policy Like You Would for Networking

If prompts are contracts, failures should be predictable.

### My default error policy pattern

1. Try to solve with guided generation (structured response)
2. If information is missing, tool call to fetch it (ground in source of truth)
3. If a tool fails, ask a single clarifying question and return a partial draft

This mirrors the tool-calling lifecycle Apple describes: you present tools, the model generates tool arguments, your code runs, returns output, and then the model produces a final response based on tool output.

---

## Guided Generation: Schema-First Output (My Favorite Reliability Lever)

When the model returns raw text, you end up parsing it and praying. Guided generation flips that: you define a Swift type, and the framework uses constrained sampling to strongly enforce the expected structure.

### A "draft transfer" style DTO

```swift
import FoundationModels

@Generable(description: "A draft plan for a money transfer inside the app")
struct TransferDraft {
    enum Status: String, Codable { 
        case ready, needsInput, blocked 
    }
    
    var status: Status
    
    @Guide(description: "Amount in minor units (e.g. cents). Must be > 0", .range(1...10_000_000))
    var amountMinor: Int
    
    @Guide(description: "ISO currency code like EUR or USD")
    var currency: String
    
    @Guide(description: "Short label identifying the payee, or empty if unknown")
    var payeeHint: String
    
    @Guide(description: "One question to ask if status is needsInput")
    var question: String?
}
```

A small detail I love: Apple recommends keeping descriptions short because long descriptions cost context and can add latency.

### Asking for structured output

```swift
let session = LanguageModelSession()
let prompt = """
User request: "Send 25 euros to Alex for dinner"
Create a TransferDraft. If payee is ambiguous, set status=needsInput and ask one question.
"""

let draft = try await session.respond(to: prompt, generating: TransferDraft.self)
```

Guided generation is also handy for "UI-ready" objects (cards, lists, next actions) rather than blobs of prose.

My take: once you start returning DTOs, you naturally design prompts like endpoints. It becomes boring—and boring is dependable.

---

## Tool-Calling Patterns That Feel "Composable"

Tool calling is how you connect the model to your real data and your side effects—fetching database rows, calling services, flipping toggles, saving content, etc.

Apple's docs lay out the phases clearly (tools list → prompt → model generates args → your tool runs → output returned → final response). That's basically dependency injection, but for an LLM.

### Pattern A: "Read tool" + "Write tool" split

I separate tools into:

- **read-only tools** (safe, repeatable)
- **side-effect tools** (save, delete, send)

Then I encode a policy:
```
Never call side-effect tools unless the user clearly confirmed.
Prefer read-only tools to gather missing data.
```

### Pattern B: "Guided generation + tool calling" loop (guided plan, then execute)

1. Generate a structured plan (TransferDraft)
2. If ready, call tools to execute steps
3. Return final result card

This reduces hallucinations because the plan is schema-constrained, and execution is tool-grounded.

### A Tool that looks like an internal API endpoint

Apple's tool example style looks like this (note the Tool conformance and Arguments being generable/guided):

```swift
struct PayeeLookupTool: Tool {
    let name = "lookupPayee"
    let description = "Finds a payee in the user's saved contacts inside the app."
    
    @Generable
    struct Arguments {
        @Guide(description: "Name fragment to search for")
        var query: String
        
        @Guide(description: "Max results", .range(1...5))
        var limit: Int
    }
    
    func call(arguments: Arguments) async throws -> [String] {
        // return list of payee IDs or display names
        []
    }
}
```

My take: tools are where you should put anything you'd normally put behind a repository/service. The model shouldn't "know" balances, IDs, or business rules. It should ask your code.

---

## "Guided Generation" Prompt Patterns I Actually Reuse

### Pattern 1: "Draft + question"

```
Create a TransferDraft.
If any required field is unknown, set status=needsInput and ask exactly one question.
Do not assume payee identity.
```

### Pattern 2: "Selectable options"

Define a DTO like:

`OptionsCard { title, options[], nextAction }`

Then enforce:

- options count 3–5
- each option label ≤ 24 chars

Guides support constraints like ranges/counts; using constraints is a quiet reliability boost.

### Pattern 3: "Explain + cite tool output"

```
If you used a tool, include a short 'Based on:' line referencing tool result IDs.
If no tool output exists, say you don't know and ask to fetch data.
```

---

## Why This Matters Right Now (My Timing Take)

- Foundation Models is positioned for structured output and tool calling, which means instruction quality directly affects reliability
- App Intents are how your actions/entities become discoverable and usable across Siri/Shortcuts/Spotlight, and Apple Intelligence enhancements lean on that metadata
- Assistant schemas + macros exist specifically to make intents/entities match what Apple Intelligence expects

My take: good instructions are the new glue code. If you invest in prompt-as-contract + schema output + tool grounding, you get "agent-like" behavior that's composable instead of chaotic.

---

## A Starter "Instruction Pack" Template (Copy/Paste Friendly)

Here's a compact template I'd actually ship (fill in your app-specific names):

```
[ROLE]
You are the in-app assistant for <AppName>.
Help the user complete tasks using the provided tools and intents.
Do not invent app data.

[CAPABILITIES]
Primary intents (names are exact):
- <IntentA>: <one-line description>
- <IntentB>: <one-line description>

Key entities:
- <EntityX>: fields <...>
- <EntityY>: fields <...>

[TOOLS]
You may call these tools when needed:
- <toolName>(<args...>) -> <result type>

Rules:
- Use tools to fetch real data or perform actions.
- Never claim an action happened unless the tool succeeded.

[OUTPUT]
Return a <GenerableType> object.
If required info is missing, set status="needsInput" and ask exactly one question.
Keep responses concise.

[ERROR POLICY]
If a tool fails:
- set status="needsInput"
- include errorCode="<toolName>.failed"
- ask one question or propose one retry
```

The magic isn't in the AI—it's in making intelligence feel like a natural extension of your app's existing architecture.