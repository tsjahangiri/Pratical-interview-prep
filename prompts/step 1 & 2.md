# Interview Take-Home — Fast Analyse & Lock (revised Step 1 + slim Step 2)

Optimised for **speed under interview time pressure**. Step 1 analyses, then asks you the open design questions **one at a time as quick choices** (your interface can render these as tappable buttons, so you answer in seconds instead of typing). It then produces the locked design automatically. Step 2 shrinks to a one-line approval.

> **Use:** Paste your repository-first system prompt once at the top. Then send Step 1 with the problem attached. Answer the questions it asks. Approve. Build.

---

## STEP 1 — Analyse, then ask me the decisions one by one

```
You are the Senior/Staff Spring Boot engineer from the system prompt above. This is
a TIME-CONSTRAINED interview take-home. Be efficient. I direct, you implement — but
right now, analysis and design only. No implementation code yet.

I've attached the problem statement[ and existing repo]. Do this in order:

PART A — Show me a tight analysis I can scan in ~20 seconds:
- PURPOSE (1 sentence)
- ENDPOINTS (compact table: METHOD | PATH | auth? | 1-line purpose)
- DOMAIN MODEL (entities + relationships; explicitly flag any attribute that
  belongs to a RELATIONSHIP rather than an entity — those need a join entity)
- EXPLICIT CONSTRAINTS (db, auth, api style, error format — only what the doc dictates)
- CORE vs OPTIONAL (must-have vs bonus/future)
Keep it tight. Tables and short bullets. No essays, no restating the problem back.

PART B — Then STOP and ask me the open design decisions ONE AT A TIME, as a short
question with 2–4 concrete options. Rules for how you ask:
- Ask only decisions that genuinely change the implementation AND are not settled by
  the document. Skip anything the doc already answers.
- For each question: give the options, mark your RECOMMENDED default, and add a
  half-line reason. Keep each question to one screen.
- Ask them one at a time (or in one small batched set of tappable choices if my
  interface supports it) — do NOT make me write paragraphs.
- Typical decisions to ask about (only if unresolved): ID strategy (UUID / Long /
  hybrid); free-text field → enum or string; a field's placement (entity-level vs
  relationship-level); how to model a specific relationship; auth requirement on a
  borderline endpoint; whether an optional feature is in scope for this build.
- If the document actually leaves NOTHING open, say so and skip to Part C using your
  recommended defaults, listing the defaults you chose.

End your turn after asking. Wait for my answers. Do not proceed to design or code
until I've answered.

PART C — After I answer, produce the LOCKED DESIGN as text (still no implementation):
- Entities: fields, types, nullability, and a half-line rationale ONLY for non-obvious
  choices (enum-as-string, LocalDate vs OffsetDateTime, @Version, unique constraints,
  indexes). Skip rationale for the obvious.
- Relationships: owning side, mappedBy, fetch types, cascade direction.
- Endpoints: final versioned paths, auth, request/response DTO shapes.
- Build order: entities → repos → dtos/mapper → service → controller → exceptions →
  config → tests.
Then ask me to approve. On "approved", we build one layer at a time and I review each
layer before the next.

If there's an existing repo, prepend a 3-line "ALREADY EXISTS" note (built / reusable /
missing) before Part A. Never dump file contents.
```

---

## STEP 2 — Slim approval (usually one line)

> After the locked design appears, you normally just approve. Only reach for corrections if something's off.

```
Approved — build in the stated order, one layer at a time, and stop after each layer
for my review.
```

or, if you need a tweak:

```
Two changes before we build:
- [correction 1]
- [correction 2]
Then produce the revised locked design and wait for my approval.
```

---

## Why this version is faster

- **You answer by choosing, not writing.** Part B asks the open questions as small option sets — your interface can render them as tappable buttons, so a decision that took a paragraph in the old Step 2 now takes a tap. That's the main time saver.
- **It only asks what's genuinely open.** Anything the document already settles is skipped, so you're not confirming obvious things.
- **It carries recommended defaults.** If you're truly out of time, you can reply "use all your recommended defaults" and it proceeds — but you still saw the decisions, so you can defend them.
- **Design still gets locked as text before code.** The speed-up doesn't sacrifice the one guardrail that matters most: a reviewed schema before any layer is built.

## The one caution

Speed here comes from *choosing quickly*, not from *skipping the thinking*. Even as buttons, read each question against your own reading of the problem — the recommended default is Claude's opinion, not a fact, and it's your name on the submission. The decisions you tap through are exactly the ones an interviewer will ask you to justify. Fast is good; understanding what you tapped is non-negotiable.
