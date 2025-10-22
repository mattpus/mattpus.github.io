# Best iOS Use Cases for On-Device Foundation Models (and When Not to Use Them)

Hot takes + battle scars from shipping "AI in the loop" features as an iOS dev who can't stop building.

Apple quietly made a huge product shift: AI is becoming a system capability. As devs, we're no longer stitching together "LLM calls" — we're designing native experiences that can run locally first. The new Foundation Models framework gives apps access to an on-device language model optimized for language understanding, structured output, and even tool calling.

And Apple is also baking AI into the OS with Writing Tools, Genmoji, and Image Playground — many of which "just work" if you're using standard text controls.

Below are the highest-value tasks that actually fit small on-device models — plus what I route to Private Cloud Compute (PCC) or "ChatGPT-style" cloud models instead.

---

## On-Device Isn't a Constraint — It's a Product Superpower

When inference happens on the phone, three things change:

- Latency becomes UI/UX (responses feel like autocomplete, not a network request)
- Privacy becomes a feature you can market
- Cost drops from "per token" to basically "free at scale"

The Foundation Models framework is explicitly positioned for "intelligent tasks specific to your use case," and it's strongest at summarization, entity extraction, refinement, and text understanding — which is exactly what most apps need.

---

## The Best iOS Use Cases for Small On-Device Models

### 1) Notification / Email Summarization (a.k.a. attention triage)

**Why it fits on-device:** short inputs, high privacy, and the output must be fast + consistent. The on-device model excels at summarization and text understanding.

**My experience:** I prototyped an "Inbox Triage" screen that summarized long threads into:
- 1-line gist
- action items
- deadline/date mentions

The breakthrough wasn't the summary — it was structured extraction. Once I had reliable fields, I could build UI people trusted.

**Hot take:** If your summary can't fit in one screen, it's not a summary — it's procrastination.

### 2) Inline Rewrite / Proofread (ship Writing Tools-like moments)

Apple's Writing Tools let users proofread, rewrite, summarize, or compose directly inside supported text views, powered by system LLMs and Apple Intelligence. UITextView and UITextField even support it automatically, with customization hooks if needed.

**My experience:** The most used AI feature I ever shipped wasn't "chat." It was:
- "Make this more concise"
- "Fix tone: friendly → professional"
- "Rewrite but keep my voice"

**Hot take:** "Rewrite" beats "chat" because it's in-context — users never leave the flow.

### 3) Structured Extraction (the ROI king)

This is where on-device models shine: turn messy text into reliable fields (title, date, entities, tags). Apple calls out entity extraction and structured output, and also offers guided generation that can generate Swift data structures with guarantees.

**My experience:** I built "clip → task" from screenshots/notes:

```
input: "Lunch with Sam next Thu 12:30 at De Pijp"
output: { title, datetime, location, attendees }
```

Once it's structured, everything else is deterministic:
- create calendar event
- prefill reminders
- show a confirmation UI

**Hot take:** Free-form text is a liability. Structured output is the product.

### 4) Short Suggestions (micro-copilot > chatbot)

Think:
- suggested replies
- smart titles
- tag suggestions
- "next step" nudges

The Foundation Models framework is built for "tasks specific to your use case," and supports tool calling so the model can ask your code for context (local DB, app state) instead of hallucinating.

**My experience:** For a journaling prototype, the best "AI" was a single line:
- "Want to add a takeaway?"
- "Tag this as: work / health / relationships?"

People used it because it felt like UX polish — not a robot.

### 5) Intent Classification (routing + automation)

On-device classification is perfect for:
- command routing ("create task" vs "search")
- safety filters
- lightweight personalization

And it plays great with tool calling ("if intent == addToCart → call my AddToCart tool").

**Hot take:** The fastest AI app is often just a great router.

### 6) Lightweight Image / Emoji Generation (fun, safe, local)

Apple's Image Playground lets you generate images from a text description, optional source image, and a chosen style — via a system UI sheet or programmatically. Apple also emphasizes that images are generated on device, so you don't host your own models.

And Genmoji is designed to create new emoji experiences, integrating well with standard text systems.

**My experience:** The best use isn't "make art." It's:
- stickers
- reaction images
- visual summaries
- delight moments in messaging, notes, or habit apps

**Hot take:** "Silly" features drive retention because they create shareable identity.

---

## What I Don't Run On-Device (and Where I Route It Instead)

### Better suited to Private Cloud Compute (PCC)

When you need more horsepower than the device can comfortably provide, Apple's approach is:
- do as much as possible on device
- offload "more sophisticated tasks" to PCC for additional compute

PCC is designed around strict privacy properties like stateless computation (data not accessible after the response is returned) and enforceable guarantees.

I route to PCC-style compute for:
- long documents + multi-step transforms
- heavier reasoning that requires larger models
- "bigger" image tasks (more complex generations, more iterations)

### Better suited to "ChatGPT-style" cloud models (with permission)

When the task needs:
- broad world knowledge
- deep creative ideation
- rich, open-ended brainstorming

I ask the user explicitly, then route to a cloud model. My rule: no silent sharing.

**Hot take:** Cloud LLMs are amazing coworkers. On-device models are amazing product features.

---

## Why Now

Because Apple is giving us:

- a Foundation Models framework that exposes the on-device LLM for app-specific tasks, including tool calling and guided generation for structured output
- system-level experiences like Writing Tools, Genmoji, and Image Playground, many of which integrate naturally with standard controls and run locally

This is the moment to stop building "AI tabs" and start building AI-native interactions.

---

## If You're Building This Quarter, Here's My Practical Starter Pack

1. Pick one high-frequency surface area (compose screen, inbox, editor)
2. Add rewrite/proofread + structured extraction
3. Use tool calling to ground outputs in your data model
4. Only escalate to PCC/cloud when the user benefits and understands why

The magic isn't in the AI — it's in making intelligence feel native to your product.