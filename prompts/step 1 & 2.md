# Interview Take-Home — Step 1 & Step 2 Prompts

Two prompts to run in sequence at the start of an interview take-home. Step 1 sets the frame and extracts a structured analysis **with no code**. Step 2 is your critical reaction to that analysis — it locks the design before a single entity is written.

> **How to use:** Paste your repository-first system prompt first (once, at the top of the conversation). Then send Step 1 with the problem attached. Read the response critically. Then send Step 2, filled in with your reactions.

---

## STEP 1 — Frame & Analyse (no code yet)

> Paste this, and attach the problem statement (PDF/text) and — if it's an existing codebase — point Claude at the repo.

```
You are acting as the Senior/Staff Spring Boot engineer defined in the system
prompt above. This is an interview take-home. I will direct the implementation;
your job right now is analysis only.

I have attached the problem statement[ and existing repository]. Do NOT write any
implementation code, entities, or config yet. Do not scaffold. Produce a structured
analysis I can review and correct before we design anything.

Give me exactly these sections, concisely:

1. PURPOSE — what this service is, in 1–2 sentences.

2. ENDPOINTS — every required endpoint as a table: METHOD | PATH | purpose |
   auth required? | request shape | response shape. Include status codes the
   requirement implies.

3. DOMAIN MODEL — the real-world entities and how they relate. For each
   relationship state the cardinality (1:1, 1:N, M:N) and, critically, flag any
   attribute that belongs to a RELATIONSHIP rather than an entity (e.g. a status
   or date that describes an assignment, not the user or the item) — these
   usually need a dedicated join entity, not a field on either side.

4. EXPLICIT TECH CONSTRAINTS — database, auth mechanism, API style, error-handling
   format, and anything else the document dictates. Quote or cite where the
   requirement states each one.

5. CORE vs OPTIONAL — separate must-have functionality from anything the document
   marks as optional / bonus / future (pagination, filtering, caching,
   performance). Do not conflate them.

6. OPEN DESIGN QUESTIONS — the 1–4 decisions that genuinely shape the
   implementation and are NOT settled by the document. Typical ones: ID strategy
   (UUID / Long / hybrid); whether a free-text field should be an enum for
   consistency and filtering; whether a field is entity-level or relationship-level;
   how a relationship should be modelled. For each, give your recommended default
   and the one-line reason — but frame it as a question for me, not a decision.

7. RISKS / AMBIGUITIES — anything under-specified in the document that a candidate
   could get wrong, and any requirement that will be hard to verify or is likely
   to be probed in a follow-up interview.

Rules for this step:
- Analysis only. No code, no file scaffolding, no package layout yet.
- Separate what the document EXPLICITLY states from your reasonable assumptions —
  label assumptions as assumptions.
- Do not invent requirements. If something isn't specified, put it in section 6 or 7.
- Be concise. Tables and short bullets, not essays.

If you are working from an existing repository as well, first do a quick recon
(build file, config/profiles, existing entities/repos/services, test conventions)
and add a short section 0: WHAT ALREADY EXISTS — what's built, what's reusable, and
what's missing for these requirements. Do not dump file contents.
```

---

## STEP 2 — Interrogate & Lock the Design

> This is your *reaction*. Read Step 1's output against your own reading of the problem, then fill the brackets. Delete lines that don't apply. The point is to catch divergence and settle every open question **before** any code exists.

```
Good. Before we write anything, let's pressure-test this analysis and lock the
design. Address each point, then produce the final agreed design.

CORRECTIONS — where your analysis diverges from the requirement or from my reading:
- [e.g. "You modelled User→Item as direct M:N, but the status and dates describe
   the assignment, so it needs a dedicated join entity. Confirm and revise."]
- [e.g. "Endpoint X should be authenticated — the response includes user-specific
   status, which is impossible without knowing who is asking. Agree?"]
- [e.g. "You listed <field> as required; the document shows it as an example, not a
   mandatory field. Make it optional/nullable."]
- [if none: "No corrections — your analysis matches my reading."]

DECISIONS on your open questions (section 6):
- ID strategy: [UUID only / Long only / hybrid Long+UUID] — because [reason].
- [Field] as enum vs free text: [enum / string] — because [reason].
- [Field] placement: [entity-level / relationship-level].
- Relationship modelling: [your call].
- [any other open question]: [decision].

CHALLENGE — for each of these, explain the trade-off and the alternative you'd
reject, so I can be sure the choice is deliberate (I'll need to defend it live):
- [decision A]
- [decision B]

SCOPE for this build: implement the CORE requirements first. For the optional items
[list], do NOT implement yet — just tell me, in one line each, where the design
should leave a clean seam so we can add them later without rework.

Now produce the LOCKED DESIGN:
- Final entity list with each entity's fields, types, nullability, and the specific
  rationale for any non-obvious type/constraint choice (enum-as-string, LocalDate vs
  OffsetDateTime, @Version, unique constraints, indexes).
- Final relationship map (owning side, mappedBy, fetch types, cascade direction).
- Final endpoint list with paths (versioned), auth, and request/response DTO shapes.
- The build order we'll follow (entities → repos → dtos/mapper → service →
  controller → exceptions → config → tests).

Still no implementation code — I want the locked design as text I can approve.
Once I say "approved", we build one layer at a time and I review each layer before
the next.
```

---

## Why these two steps matter (and how to run them well)

- **Send them separately.** Step 2 depends on what Step 1 returns — you cannot pre-write your corrections. If you paste both at once you lose the chance to catch a bad assumption before it's baked into the design.

- **Read Step 1 against your own reading, not in a vacuum.** The value isn't Claude's analysis — it's the *gap* between Claude's analysis and yours. That gap is where you learn the problem and where you catch errors an interviewer would catch.

- **The "CHALLENGE" block is the interview rehearsal.** Every decision you lock here is a decision you'll be asked to justify live. If Claude's explanation of a trade-off doesn't convince you, you're not ready to defend it — push until it does, or change the decision.

- **Keep the "no code yet" guardrail.** The single most common failure mode is letting the model scaffold an entire app before the schema is right. A wrong relationship or ID choice at this stage costs you a rewrite across every layer.

- **"Locked design as text" is your checkpoint artifact.** You approve it explicitly. From that point, every subsequent step builds against an agreed contract, and you review each layer against it — which is exactly the discipline that makes the final submission defensible.

- **Caveat:** confirm the take-home's rules on AI assistance before you start. And remember the goal of these two steps is that *you* end up understanding the design deeply enough to extend it live — not just to produce it.
