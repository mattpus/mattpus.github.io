# Agent-Readiness on iOS: Structure Your App So Siri Can Act Across Features

Siri's capabilities are expanding from "launch an action" to "operate with context," including deeper in-app and cross-app actions powered by the App Intents framework. Apple positions App Intents as the integration surface for Siri, Spotlight, widgets/controls, and Shortcuts, and notes that personal context understanding, on-screen awareness, and in-app actions are being rolled out via future software updates.

Apple (and the broader Apple ecosystem press) has also highlighted that iOS 18.2 introduces developer APIs to make on-screen content available to Siri and Apple Intelligence, so apps can be "ready" when those features ship more broadly.

Below is a practical, implementation-focused guide to make your app agent-ready: expose your core actions via App Intents, model your core objects as App Entities, and provide on-screen awareness hooks so Siri can act on "what the user is looking at."

---

## Design Your "Agent Surface Area": Verbs + Nouns, Not Screens

A Siri/agent integration works best when you think in:

- **Verbs**: actions a user wants done (create, update, send, open, track, pay, save) via AppIntent
- **Nouns**: domain objects the verb applies to (project, invoice, album, transaction) via AppEntity

This aligns with Apple's guidance that App Intents make actions and content discoverable across Siri/Spotlight/Shortcuts and that App Entities help the system resolve parameters and understand app concepts.

**Tip**: Start with the top 5 high-frequency tasks that are "atomic" (finish in one step, minimal UI), then expand once the intent/entity plumbing is solid. This matches how App Shortcuts are meant to surface "key app functionalities" that people use to complete tasks quickly.

---

## Make Core Features Callable with AppIntent

Here's a compact example: a finance app intent that logs an expense. It demonstrates parameters, safe defaults, and returning a result that can be shown in system UI.

```swift
import AppIntents

struct LogExpenseIntent: AppIntent {
    static var title: LocalizedStringResource = "Log Expense"
    static var description = 
        IntentDescription("Adds an expense entry with amount, category, and optional note.")
    
    @Parameter(title: "Amount")
    var amount: Double
    
    @Parameter(title: "Category")
    var category: CategoryEntity
    
    @Parameter(title: "Note")
    var note: String?
    
    // Safe default example: if note is omitted, proceed without extra prompts.
    func perform() async throws -> some IntentResult {
        // Persist expense in your storage layer
        try await ExpenseStore.shared.add(amount: amount, categoryID: category.id, note: note)
        return .result()
    }
}
```

Why this structure matters:

- App Intents are the mechanism to make app actions discoverable to Siri and other system surfaces
- Clear parameterization improves how Shortcuts/Siri can run actions without extra clarification, especially when paired with App Shortcuts

**Tip (safe defaults)**: Prefer non-destructive defaults and avoid actions that irreversibly modify data without friction; Apple's design guidance for voice/shortcut experiences emphasizes keeping interactions simple and using defaults/alternatives when appropriate.

---

## Declare Your App's Nouns as AppEntity (and Make Them Queryable)

To let Siri resolve "Category" from natural language (and to show rich UI in Shortcuts), declare it as an AppEntity and provide an EntityQuery. Apple explicitly calls out that App Entities let the system introspect types, resolve intent parameters, and speed up interactions with fewer verbal back-and-forths.

Also note Apple's guidance that entity identifiers should be unique and persistent because the system may save them in a shortcut.

```swift
import AppIntents

struct CategoryEntity: AppEntity {
    static var typeDisplayRepresentation: TypeDisplayRepresentation = "Category"
    static var defaultQuery = CategoryQuery()
    
    // Must be stable over time (saved in shortcuts).
    var id: String
    
    @Property(title: "Name")
    var name: String
    
    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(name)")
    }
}

struct CategoryQuery: EntityQuery {
    func entities(for identifiers: [CategoryEntity.ID]) async throws -> [CategoryEntity] {
        try await CategoryStore.shared.fetch(ids: identifiers)
    }
    
    func suggestedEntities() async throws -> [CategoryEntity] {
        // Keep this list short; show "top" or "recent" categories.
        try await CategoryStore.shared.suggested()
    }
    
    func defaultResult() async -> CategoryEntity? {
        try? await CategoryStore.shared.mostUsed()
    }
}
```

Why this matters for agent readiness:

- Entities + queries are how Siri and the system map user language to your in-app objects (parameter resolution)
- Rich displayRepresentation improves the system UI for disambiguation and Shortcuts tiles

**Tip**: Model conceptual entities too (e.g., "Current Invoice", "Most Recent Project")—Apple mentions entities can represent concepts like "the current photo" to reduce clarifying questions.

---

## Boost Discoverability with App Shortcuts (Your "Distribution Channel")

App Shortcuts make your intent available immediately after install across Siri, Spotlight, and the Shortcuts app—no "Add to Siri" flow required.

They also support preconfigured parameters so users can run "one phrase → one action," reducing Siri follow-ups.

```swift
import AppIntents

struct FinanceShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(
            intent: LogExpenseIntent(
                amount: 0, 
                category: CategoryEntity(id: "food", name: "Food"), 
                note: nil
            ),
            phrases: [
                "Log an expense in \(.applicationName)",
                "Add expense in \(.applicationName)"
            ],
            shortTitle: "Log Expense",
            systemImageName: "plus.circle"
        )
    }
}
```

Why this matters now:

- Apple notes Siri will suggest your actions and can take actions "in and across apps" as App Intents and Apple Intelligence evolve
- App Shortcuts metadata (phrases, titles) is part of how the system improves discovery and learns which actions to suggest

**Tip**: Keep phrases user-centric ("Start a...", "Log...", "Open...") and avoid feature jargon; Apple's HIG stresses predictability and simplicity for voice interactions.

---

## Prepare for On-Screen Awareness: Attach Visible Content via NSUserActivity

On-screen awareness depends on your app explicitly providing "what's on screen" to Siri/Apple Intelligence via App Intents + NSUserActivity. Apple's documentation describes the pattern: create an AppEntity, generate an EntityIdentifier, and set it on the active NSUserActivity using appEntityIdentifier.

Apple also states that Siri/Apple Intelligence can retrieve on-screen content when the user asks about it, and can send content to third-party services only if the user explicitly requests it.

Here's a SwiftUI example similar to Apple's documented approach:

```swift
import SwiftUI
import AppIntents

struct InvoiceDetailView: View {
    let invoice: InvoiceEntity
    
    var body: some View {
        InvoiceUI(invoice: invoice)
            .userActivity("com.example.myapp.viewingInvoice", element: invoice) { invoice, activity in
                activity.title = "Viewing invoice"
                activity.appEntityIdentifier = EntityIdentifier(for: invoice)
            }
    }
}
```

If your entity represents file-like content, Apple also recommends making the entity Transferable so Siri/Apple Intelligence can attach it when the user explicitly asks to share/send it.

**Tip (privacy & safety)**: Treat on-screen exposure as a capability gate: only attach entities when content is truly visible, and ensure the attached entity contains the minimum data needed for the task—this aligns with Apple's "explicit request" model for sending content outward.

---

## Intent Versioning: Evolve Without Breaking People's Shortcuts

Because App Intents and App Entities can be saved inside user shortcuts, stability matters. Apple notes entity identifiers must be persistent since the system may store them.

A robust pattern for intent evolution is:

- Avoid breaking changes to existing intent semantics/parameters where possible
- If you must change behavior significantly, create a new intent (LogExpenseIntentV2) and keep the old one available (possibly deprecated in code, but still functional)
- Keep entity IDs stable even if display names change (IDs are what shortcuts persist)

Example pattern:

```swift
import AppIntents

@available(iOS, deprecated: 18.0, message: "Use LogExpenseIntentV2")
struct LogExpenseIntentV1: AppIntent { /* old fields & behavior */ }

struct LogExpenseIntentV2: AppIntent {
    static var title: LocalizedStringResource = "Log Expense"
    
    @Parameter(title: "Amount") 
    var amount: Double
    
    @Parameter(title: "Category") 
    var category: CategoryEntity
    
    @Parameter(title: "Merchant") 
    var merchant: String?   // new capability
    
    func perform() async throws -> some IntentResult { 
        .result() 
    }
}
```

**Tip**: When adding new parameters, prefer making them optional first (or provide defaults via App Shortcuts preconfiguration), because Apple highlights preconfigured parameters as a way to reduce clarifying steps.

---

## A Practical Checklist for "Agent-Ready" Apps

### Core Actions
- ✅ Top tasks expressed as AppIntent verbs
- ✅ Parameters modeled to minimize follow-up questions (use App Shortcuts where common presets exist)

### Core Objects
- ✅ Key nouns expressed as AppEntity with rich displayRepresentation
- ✅ Query types implemented for resolution and suggestions

### Discoverability
- ✅ App Shortcuts for the "top 3–10" user tasks, available immediately post-install

### On-Screen Awareness
- ✅ Visible content bound to NSUserActivity.appEntityIdentifier with EntityIdentifier(for:)
- ✅ Entities made Transferable where share/send workflows are expected

### Safety & Evolution
- ✅ Non-destructive defaults; avoid irreversible operations without appropriate friction
- ✅ Versioning plan so existing user shortcuts keep working (stable IDs, additive changes, new intents for breaking updates)

---

## Why This Matters (In Plain Terms)

As Siri gains more context and the ability to act within and across apps, your App Intents + Entities become your app's "agent interface." Apple explicitly frames App Intents as the integration point for Siri and other system experiences, including the forthcoming on-screen awareness and personal context features.

And Apple's rollout of on-screen awareness APIs in iOS 18.2 (with ecosystem reporting highlighting the same) signals that "agent readiness" is not speculative—you can implement the hooks now so your app participates when the broader capability set becomes available.

The magic isn't in predicting the future—it's in building the right abstractions today so your app can evolve with the platform tomorrow.