# 🧠 When to Run AI On‑Device vs Escalate to the Cloud

A very practical (and very opinionated) decision playbook for iOS developers who love building

I'm an iOS developer who's genuinely obsessed with the craft of great apps — the kind that feel instant, respectful, and surprisingly smart. Lately, AI features have become the fastest way to delight users… and the fastest way to break trust if we're sloppy about privacy, latency, or cost.

Apple Intelligence pushes us toward a hybrid mindset: do as much as you can on the device, and only "go up" to the cloud when you truly need more capability. Apple's own architecture reflects that: on‑device first, and for heavier requests, Private Cloud Compute (PCC) — a privacy-focused cloud layer built to extend Apple's security model into the data center. [security.apple.com], [security.apple.com]

And when you want third‑party power (like ChatGPT) you can offer it — but only with explicit user consent and clear controls, exactly how Apple models it in the system experience. [support.apple.com], [help.openai.com]

Let's turn that into a decision playbook you can actually use.

---

## The TL;DR Ladder (my default mental model)

**1) Start On‑Device (fast, private, predictable cost)**  
Apple's Foundation Models framework gives you access to Apple's on‑device language model for tasks like summarization, extraction, classification, rewriting, and even tool calling + structured output.  
It requires that the user has Apple Intelligence enabled on their device. [developer.apple.com]

**2) Escalate to PCC when "bigger brain" is needed (still privacy‑first)**  
Apple designed Private Cloud Compute for "more sophisticated tasks" that need "more complex foundation models in the cloud," while meeting requirements like stateless processing, no privileged runtime access, and verifiable transparency.  
PCC uses custom Apple silicon and a hardened OS, and is built so user data sent to PCC is not accessible to anyone other than the user — "not even to Apple." [security.apple.com], [security.apple.com] [security.apple.com]

**3) Offer ChatGPT (or another external model) only when users explicitly opt in**  
Apple's ChatGPT integration is an "Extension" users enable, and Siri asks for confirmation by default before sending a request to ChatGPT.  
Users can connect a free or paid account, and paid accounts can use advanced capabilities more often. [support.apple.com], [help.openai.com] [support.apple.com]

---

## What "On‑Device" really means (and why I love it)

With Foundation Models, you're working with an on-device LLM built to do "language understanding, structured output, and tool calling," and Apple explicitly calls out strengths like summarization, entity extraction, text understanding, refinement, dialog, and creative content. [developer.apple.com]

A tiny "shape" of what this feels like:

```swift
import FoundationModels

let session = LanguageModelSession()
let response = try await session.respond(to: "Summarize this note in 3 bullets.")
print(response.content)
```

LanguageModelSession is the core object representing a session with the on-device model. [developer.apple.com]

### When I choose on-device by default

I reach for on-device first when:

- Latency must feel instant (typing assistance, smart UI, autofill-ish moments)
- The user might be offline
- The data is sensitive (health, finance, private messages, internal work notes)
- The feature is high volume (a lot of small requests that would kill a cloud bill)

And yes — there's a cost: not money per request, but battery/thermal budget and local performance constraints. That's your trade: you "pay" with device resources instead of server invoices.

---

## Private Cloud Compute (PCC): the "cloud, but make it Apple‑private"

When Apple Intelligence can't do something fully on device, PCC is the designed escalation path: "Whenever possible" tasks are local, but "more sophisticated tasks require additional processing power… in the cloud." [security.apple.com]

What makes PCC different (and the reason users might trust it more than a typical AI API):

- **Stateless computation**: user data isn't accessible after the response is returned [security.apple.com], [security.apple.com]
- **No privileged runtime access**: PCC is designed so Apple staff can't bypass privacy guarantees [security.apple.com], [security.apple.com]
- **Verifiable transparency**: researchers can verify PCC's privacy/security properties [security.apple.com], [security.apple.com]
- **Custom Apple silicon + hardened OS** built for privacy [security.apple.com]

**My mental rule:**  
If the request needs more capability but still wants Apple's privacy posture, PCC is the "cloud tier" that aligns with Apple's trust story.

---

## ChatGPT (external escalation): powerful, but consent must be visible

Apple's own UX pattern is the right one to copy:

- Users choose to enable ChatGPT integration. [support.apple.com]
- Siri asks for confirmation by default before using ChatGPT. [help.openai.com], [support.apple.com]
- Users can connect an account, and a paid account can unlock more frequent access to advanced capabilities. [support.apple.com]

This is the core product insight: **privacy delegation should be explicit, not implicit**.

So if your app offers "Use a stronger model," make it a clear, user-driven action — not a silent fallback.

---

## Cost talk (the part nobody wants to do, but we must)

Let's break it down plainly.

### 1) On‑device costs

**What you save:**
- No per‑request inference bill
- No GPU servers
- No bandwidth for requests/responses

**What you "pay":**
- Battery/thermal impact
- App size / model packaging constraints (depending on approach)
- More QA across devices
- Feature availability constraints (Apple Intelligence must be enabled) [developer.apple.com]

On-device can reduce dependency on data centers, and industry commentary increasingly points out that cloud inference has hidden dollar + resource costs that on-device can sometimes avoid. [qualcomm.com]

### 2) Cloud costs (your own servers or third-party APIs)

You pay for:
- Inference (often token‑based or compute‑time based)
- Traffic (uploads/downloads)
- Storage/logging (if you keep data — which is also a privacy risk)
- Reliability engineering (rate limits, retries, fallbacks, incidents)

Also, sending data back and forth costs bandwidth and can become expensive at scale; this is one of the classic "cloud AI" tradeoffs compared to edge/on-device processing. [edgeir.com]

### A simple way to estimate cloud spend (use this in planning)

I like this back‑of‑napkin formula:

```
Monthly cost ≈ (requests/day × 30) × (avg tokens/request ÷ 1000) × ($/1K tokens)
+ bandwidth + baseline infra + monitoring
```

Even if you don't know exact pricing yet, this gives you a "scale smell test."

### 3) PCC costs

PCC is part of Apple's platform design for Apple Intelligence: it exists to run "more complex foundation models in the cloud" without giving up the security model.  
From a developer perspective, the "cost" you feel most is latency + dependency on network availability, not necessarily a line-item cloud bill (though you still pay indirectly via product constraints like availability and user expectations). [security.apple.com], [security.apple.com]

### 4) ChatGPT costs

Here, the cost can shift to the user:
- Users can connect free or paid accounts, and paid accounts can access advanced capabilities more often.
- For you, the product "cost" is building consent UX and clear data-handling rules. [support.apple.com]

---

## The Decision Tree I actually use

Here are the forcing functions — if you answer these honestly, the architecture usually picks itself.

### ✅ Choose On‑Device when:

- You need instant interaction
- The feature should work offline
- You're handling sensitive personal context
- The task is small/structured (extract, tag, summarize, rewrite)
- You expect high volume and want predictable costs

Foundation Models is built for exactly these kinds of language tasks (summarization, extraction, refinement, etc.). [developer.apple.com]

### ☁️ Choose PCC when:

- The task is too complex for on-device constraints
- You still want Apple's privacy posture (stateless processing, no privileged access, verifiable transparency) [security.apple.com], [security.apple.com]
- You can tolerate network latency

### 🌍 Choose External Cloud (e.g., ChatGPT) when:

- Users explicitly ask for capabilities beyond Apple's models
- You need broad world knowledge, specialized domain depth, or specific generation styles
- You can present clear opt-in and data boundaries

Apple's own integration makes consent explicit (confirmation by default). [help.openai.com], [support.apple.com]

---

## Concrete scenarios (from "things I've built or want to build")

### Scenario A: A note‑taking app

**On-device:**
- Summarize the current note into bullets
- Extract action items
- Auto-tag the note for search

This matches what Foundation Models is meant to excel at (summarization, extraction, tagging, text understanding). [developer.apple.com]

**Escalate (PCC):**
- "Find themes across 200 notes and draft a weekly review"
- "Compare these documents and reconcile conflicts"

That's the kind of "more sophisticated" request Apple designed PCC to support. [security.apple.com], [security.apple.com]

**Optional external (ChatGPT):**
- "Help me research this topic" or "Draft a creative essay in a specific voice"
- Make it an explicit "Ask ChatGPT" mode with clear UI

This matches Apple's pattern: user chooses to enable it and confirm requests by default. [support.apple.com], [help.openai.com]

### Scenario B: Email or messaging "smart replies"

**On-device only (most of the time):**
- Suggest 2–3 short replies
- Rewrite tone (more formal / more friendly)
- Extract dates or commitments

High volume + privacy sensitive = on-device wins.

### Scenario C: Photo / camera intelligence

**On-device:**
- Quick caption suggestions
- Categorize albums ("receipts", "travel", "family")

**ChatGPT (opt-in):**
- Deep interpretation + creative outputs where the user wants "more"
- Apple's own docs describe using ChatGPT with visual intelligence once enabled. [support.apple.com]

### Scenario D: Health, finance, and anything regulated

My rule here is blunt: **default to on-device whenever possible**, because sending raw sensitive text to third parties becomes a compliance and trust minefield.

If you must escalate:
- Prefer privacy-preserving layers (PCC-style guarantees are explicitly designed around statelessness and restricted access). [security.apple.com], [security.apple.com]
- If using external models, require explicit opt-in and isolate what you send.

---

## Practical implementation tips (the "save yourself pain later" section)

### 1) Build a progressive disclosure UX
Start with on-device results fast. Then offer:
- "Improve with Advanced AI" (PCC / external) as an explicit action

### 2) Send the minimum necessary input
Even in privacy-first systems, smaller prompts are cheaper, faster, and safer.

### 3) Cache intelligently
- Cache summaries and tags for stable content
- Store "model outputs" with versioning so you can refresh later if needed

### 4) Design graceful failure paths
- If network is down: keep on-device functionality usable
- If external AI is disabled: don't degrade the core product

### 5) Measure what matters
- Latency p50/p95
- Battery impact (energy log)
- User trust signals (opt-in rate, disable rate, "why is this slow?" feedback)

---

## My closing take

If you remember one thing: **AI architecture is product architecture**.

On-device gives you immediacy and privacy. PCC is Apple's attempt to bring the same trust model to cloud-scale requests. And external models can be incredible — as long as you treat them like what they are: a user-authorized delegation of privacy and data handling. [security.apple.com], [security.apple.com], [support.apple.com], [help.openai.com]

If you map your features to that ladder — device first, cloud when necessary, third-party with permission — you'll build AI that feels useful without feeling creepy.