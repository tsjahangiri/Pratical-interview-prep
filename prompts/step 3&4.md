# Interview Take-Home — Step 3 (Build) & Step 4 (Harden)

Continues from the fast Analyse-&-Lock flow. By now you have an **approved locked design**. Step 3 builds it layer by layer with a review gate after each. Step 4 is the senior-signal hardening pass. Both are written for speed under interview pressure.

> **Use:** After you've said "approved" to the locked design, send Step 3 once. Review each layer as it comes, using the mini-checklist. When the happy path works end to end, send Step 4.

---

## STEP 3 — Build layer by layer, with a review gate after each

```
Approved. Build the locked design now, but under these rules — they matter as much
as the code:

PACE & GATES
- Build ONE layer at a time in the agreed order: entities → repositories →
  dtos/mapper → service → controller → exception handling → config → tests.
- After each layer, STOP. Give me: (a) the files for that layer, (b) a 2–3 bullet
  "what to check" note calling out any non-obvious decision, (c) nothing else.
  Then wait for my "next" before starting the following layer.
- Do NOT jump ahead. Do NOT dump the whole app. If a later layer forces a change to
  an earlier one, say so explicitly rather than silently editing.

QUALITY BAR (apply silently, per the system prompt — don't lecture me about it)
- Constructor injection; thin controllers; no business logic or DB access in
  controllers; no HTTP/JSON concerns in services.
- Deliberate @Transactional boundaries (readOnly on reads).
- Repositories: record projections + explicit countQuery for lists; single-query
  JOIN FETCH / LEFT JOIN FETCH for detail; (field = :p OR :p IS NULL) for nullable
  filters. Flag the TupleBackedMap pitfall if records are involved.
- Explicit FetchType.LAZY on @ManyToOne/@OneToOne; cascade parent→children only.
- Follow the EXISTING repo's naming, DTO shape, exception style, and test patterns
  if one exists. Consistency beats your personal preference.

OUTPUT DISCIPLINE
- Real, compilable code in the correct package. No pseudo-code, no "// TODO fill in".
- Only the files for the current layer. Don't re-print unchanged files.
- One conventional-commit message per layer (feat:/refactor:/test:/chore:).

Start with the ENTITY layer. Stop after it and wait for me.
```

### Your per-layer review checklist (keep this next to you — don't send it)

Read each layer against this before you say "next":

- **Entities** — right relationship modelling? enum-as-string? LocalDate vs OffsetDateTime correct? @Version on mutable entities? unique constraints + indexes present? lazy on @ManyToOne?
- **Repositories** — projection for the list? explicit countQuery? JOIN FETCH vs LEFT JOIN FETCH correct (mandatory vs optional)? ID type matches?
- **DTOs / mapper** — list DTO ≠ detail DTO (no over-fetch)? mapper handles the null/empty/format edge cases?
- **Service** — minimal dependencies? readOnly on reads? fallible calls wrapped → DataPersistenceException? no leaking of framework exceptions?
- **Controller** — thin? versioned path? auth via @AuthenticationPrincipal? filters as a record? page-size cap?
- **Exceptions** — right status per type? **client messages generic (no SQL/table/class names)?**
- **Config** — open-in-view:false? profiles? secrets from env? cache keys isolate per user?

If anything's off, say exactly what and why, and have it revised before moving on. This per-layer reading *is* how you end up able to defend the submission.

---

## STEP 4 — Senior-signal hardening pass

> Send this once the happy path runs end to end. These are the details interviewers actually probe — they're worth more per line than any extra feature.

```
Happy path works. Now a hardening pass — implement these explicitly, one small commit
each, and after each tell me in one line what changed and why:

SECURITY & CORRECTNESS
1. Error messages: audit every exception the client can trigger. Client-facing
   messages must be GENERIC — no class names, SQL, table names, constraint names, or
   internal IDs. Map each to the right status (404/409/400/401/500). Show me the
   before/after of any message that was leaking internals.
2. Data isolation: add the test that proves one authenticated user can NEVER see
   another user's data through any endpoint. This is the single most important
   security test — make it explicit and unmistakable.
3. Auth edge cases (if the API is secured): tests for valid / wrong-password /
   wrong-user / no-credentials, and confirm auth failures are generic (no user
   enumeration).
4. Graceful DB failure: confirm every fallible repository call surfaces as a
   controlled DataPersistenceException → 500 with a generic message, never a raw
   stack trace or framework exception to the client.

OBSERVABILITY
5. Logging discipline: add SLF4J logging at the right levels (DEBUG flow / INFO
   success / WARN expected-failure & security events / ERROR unexpected). Verify we
   NEVER log passwords, usernames/emails, request-parameter values, or framework/DB
   exception messages. Safe to log: public UUIDs, pagination state, result counts.

VALIDATION
6. Run the full test suite. Report build + test status. If anything fails, root-cause
   it and fix — don't just make the test pass.

Keep each change minimal and self-contained. Don't refactor unrelated code. If you
spot a genuine improvement outside this scope, list it under "Potential follow-up" —
don't implement it.
```

---

## After Step 4 — the finishing moves (quick prompts you can fire as needed)

- **Runnable proof:**
  ```
  Add a docker-compose (db + app) and a @Profile("dev") data seeder that loads varied
  realistic data (multiple users, every enum value, edge cases like null dates and
  empty collections), idempotent, never active in prod/test. Give me the exact
  commands to run it and hit the endpoints in Postman.
  ```
- **README:**
  ```
  Write a professional README: tech stack, architecture + design-decision rationale,
  setup, run (docker + local), full endpoint reference with request/response examples
  and curl, auth instructions, filtering/pagination, error catalogue, logging strategy,
  test instructions (with the Testcontainers-over-H2 rationale), and a Future
  Enhancements section that names what we DELIBERATELY excluded and why.
  ```
- **Interview rehearsal (do this yourself, with Claude as sparring partner):**
  ```
  Act as the interviewer. For each significant decision in this codebase — ID strategy,
  relationship modelling, projections, JOIN FETCH, AuthenticatedUser, generic errors,
  Testcontainers, what we deliberately DIDN'T build — ask me one probing question at a
  time and pressure-test my answer. Don't accept vague answers.
  ```

---

## Why the build phase is structured this way

- **The review gate is the point, not a delay.** Stopping after each layer is what forces you to actually read the code — which is the only thing that lets you defend it live. A submission you didn't review is one the follow-up interview will expose in minutes.
- **Hardening is separated from building on purpose.** Generic errors, data-isolation, safe logging, graceful failure — these are low-line-count, high-signal changes. Interviewers probe them precisely because most candidates skip them. Doing them as a distinct, deliberate pass makes sure they don't get lost in feature work.
- **"Potential follow-up, don't implement" keeps scope honest.** Naming what you deliberately left out — and why — reads as more senior than cramming in extra features. It shows judgement, which is what's actually being evaluated.
- **The rehearsal step is the real deliverable.** The code is an artifact; your reasoning is what's scored. Using Claude to interrogate you before the interview is the highest-leverage thing in this whole workflow — do not skip it.

## The standing caution

Every review gate you rush and every hardening item you accept without reading is a question you won't be able to answer live. Speed comes from tapping and approving quickly *because you understood* — not from skipping the read. And confirm the take-home's rules on AI assistance before you rely on any of this.
