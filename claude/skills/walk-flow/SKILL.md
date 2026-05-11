---
name: walk-flow
description: Walk through a feature or API call layer-by-layer (HTTP route → service → repository → DB / external API / UI). Each layer gets a short prose summary plus concrete file_path:line references. Pause after each layer and wait for the user to say "go" / "next" / "yes" before moving deeper. Use when the user asks to understand how a feature works end-to-end, trace a request, or build a mental model of a flow.
argument-hint: "<feature, endpoint, or behavior to trace>"
---

# Walk a feature flow, one layer at a time

Goal: help the user build a layered mental model of how a feature works end-to-end. The user explicitly does **not** want a single mega-dump — they want to digest one layer, optionally read code themselves, then prompt for the next.

## Operating rules

1. **One layer per turn.** After describing a layer, **stop**. End your turn with a short prompt like *"Tell me when you've looked, and I'll give you Layer N+1."* or *"Ready for Layer N+1?"*. Do **not** preemptively describe the next layer in the same response.

2. **Wait for explicit confirmation** before continuing. Phrases that mean "go": "yes", "go", "next", "continue", "ok". If the user asks a clarifying question instead, answer it and **stay on the current layer** — do not advance.

3. **Anchor every claim with a file:line reference.** When you mention a function, route, or behavior, format the location as `agent/src/elara/services/engine.py:78` so the user can click straight to it. If you can't pin a line, give the file path and the function name.

4. **Layers are not fixed at four.** Adapt to the flow:
   - A simple CRUD endpoint may have 3 layers (route → repository → DB).
   - An LLM-streaming endpoint may have 5+ (route → orchestration generator → LLM client → SSE pubsub → DB persistence → UI consumer).
   - A frontend feature may go: page → hook → API client → backend route → service → repo. Walk forward through whichever layers actually exist.

5. **Each layer summary is short** — 4–8 sentences max for the body. If there are multiple substeps inside one layer (e.g. orchestration steps), use a tight numbered list with brief items.

6. **Don't narrate your search.** The user doesn't want "Let me check… Now I'll grep…". Run reads and greps quietly, then deliver the layer summary directly.

## Layer-by-layer template

For each layer, structure the response as:

> **Layer N — <short name>**
>
> 2–5 sentences explaining the layer's role and what it hands off to the next.
>
> Then either a short bulleted list of concrete sub-steps, or 2–3 file:line anchors:
>
> - `path/to/file.py:LINE` — what happens here
> - `path/to/other.py:LINE` — and here
>
> Optionally one short Insight box if there's a non-obvious detail.
>
> End with a one-line prompt to continue.

## Phase 1 — Identify the entry point

When the skill is invoked, your **first** turn should identify and describe **only Layer 1** — the entry point.

Pick the entry point based on what the user asked about:
- For an API endpoint or backend behavior → the route handler (look in `routes/`, `api/`, controllers).
- For a UI feature → the React component or page that initiates the action (look in `frontend/src/pages/`, `frontend/src/components/`).
- For a background job → the scheduler / cron / queue subscriber.
- For a CLI command → the command entry function.

If it's ambiguous (e.g. "explain how authentication works"), **ask one focused clarifying question** before starting Layer 1 — e.g. "Do you want to walk through the login flow specifically, or session validation on subsequent requests?". Don't ask multiple questions at once.

## Phase 2 — Subsequent layers

After the user signals to continue, identify the **next layer down** by examining what Layer N hands off to. Common transitions:

- **Route → Service**: route returns a `StreamingResponse` / `EventSourceResponse` / awaits a service function.
- **Service → Repository / DB**: service calls `repo.X()` or opens a DB session.
- **Service → External API**: service calls an SDK like `anthropic.AsyncAnthropic`, `boto3`, etc.
- **Service → Pubsub / Queue**: service publishes to an in-memory bus, Redis, SQS, etc.
- **Frontend Component → API Client**: component calls a function from `lib/api.ts` or similar.

Inspect the actual handoff in the previous layer's code to determine where to go next — don't guess from architecture-shaped naming.

## Phase 3 — Termination

The walkthrough is complete when:
- The flow reaches its terminus (DB write returns, external API responds, response is rendered, SSE stream closes).
- The user says "stop", "that's enough", "done", or otherwise signals they have what they need.
- You've covered all layers and are about to repeat yourself.

When complete, give a one-paragraph end-to-end recap connecting all layers, e.g. *"So the flow is: frontend `chat-input` (Layer 1) → POST `/chat` route (Layer 2) → `chat_stream` generator (Layer 3) → Anthropic streaming SDK (Layer 4) → SSE delta events back to the frontend (Layer 5) → assistant message persisted (Layer 6). The orchestrating piece is `chat_stream`, which also fires the extraction pipeline as a follow-up."*

## Insight boxes (optional, sparing)

If a layer has a non-obvious detail worth highlighting (a lazy evaluation pattern, a race condition, a surprising shared resource, an unusual SDK behavior), include **one** short insight in this format:

```
★ Insight ─────────────────────────────────────
[2–3 sentences on the non-obvious detail]
─────────────────────────────────────────────────
```

Use sparingly — at most one per layer, often zero. They're for genuinely surprising behavior, not generic programming tips.

## Style — what to avoid

- ❌ Don't dump multiple layers in one response.
- ❌ Don't describe a layer without file:line anchors.
- ❌ Don't narrate your tool use ("Let me read engine.py…").
- ❌ Don't describe the same code at two layers — each layer is a new vertical slice.
- ❌ Don't add an Insight box just to look thorough — only when the detail is actually surprising.

## Style — what to do

- ✅ Quiet research, then deliver the layer cleanly.
- ✅ Stop at the end of each layer and wait.
- ✅ When the user asks a clarifying question mid-walk, answer it on the current layer; stay put until they say "next".
- ✅ Adapt depth to the user's apparent expertise — they may already know what FastAPI is; don't over-explain framework basics.
