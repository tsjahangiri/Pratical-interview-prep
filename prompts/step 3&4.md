# Interview Take-Home — Step 3 (Build) & Step 4 (Harden)

Continues from the Analyse-&-Lock flow. By now you have an **approved locked design**.
Step 3 builds it layer by layer with a decision gate after each. Step 4 audits the
finished whole for cross-cutting properties. Both are written for speed under interview
pressure.

**The rule that splits them:** *Step 3 gets everything decidable while building a single
layer. Step 4 gets everything only verifiable against the finished whole.* Indexes need
queries → Step 3 (repository gate). Per-endpoint validation needs the entry point → Step 3
(controller gate). "Is the error shape consistent everywhere / can any user see another's
data / is logging safe across the flow" → Step 4.

---

## STEP 3 — Build layer by layer, with a decision gate after each

```
Approved. Build the locked design now under these rules — the per-layer GATES matter
as much as the code.

PACE & GATES
- Build ONE layer at a time: entities → repositories → dtos/mapper → service →
  controller → exception handling → config → tests.
- After each layer, STOP. Give me: (a) the files, (b) the LAYER GATE for that layer
  answered explicitly (below), (c) nothing else. Wait for my "next".
- Do NOT jump ahead or dump the whole app. If a later layer forces an earlier change,
  say so explicitly instead of silently editing.

LAYER GATES — answer these at the layer where they belong, not later:

  ▸ ENTITY layer gate:
    - Relationship modelling correct (join entity for relationship-attributes)?
    - EnumType.STRING? LocalDate vs OffsetDateTime correct?
    - @Version on mutable entities? Unique constraints for business rules?
    - Explicit FetchType.LAZY on @ManyToOne/@OneToOne? cascade parent→children only?

  ▸ REPOSITORY layer gate  ← decide INDEXES here:
    - List each query you've written and the WHERE/ORDER BY columns it uses.
    - From those, state exactly which indexes are needed and why: single-column for
      single-column filters, composite (most-selective-first) for combined filters.
      Distinguish indexes needed NOW from forward-looking ones, and say which is which.
      Add them to the entity @Table(indexes=...) and show the change.
    - Confirm: record projection + explicit countQuery for lists? JOIN FETCH
      (mandatory) vs LEFT JOIN FETCH (optional) correct? (field = :p OR :p IS NULL)
      for nullable filters? Flag the TupleBackedMap pitfall if records are used.

  ▸ DTO/MAPPER layer gate:
    - List DTO ≠ detail DTO (no over-fetch)? Mapper covers null/empty/format edges?

  ▸ SERVICE layer gate:
    - Minimal dependencies? readOnly on reads? Fallible repo calls wrapped →
      DataPersistenceException? No framework exceptions leaking upward?

  ▸ CONTROLLER layer gate  ← decide REQUEST VALIDATION here:
    - For every endpoint, state the validation applied and WHERE it lives:
      * path variables (e.g. UUID format — how is a malformed one handled?)
      * pagination (page ≥ 0, a max-page-size cap — set it in config, not ad hoc)
      * filter/query params (invalid enum value → handled gracefully, not a 500)
      * request bodies (Bean Validation @NotNull/@NotBlank/@Size/@Valid at this
        boundary; business validation stays in the service — don't duplicate)
    - Thin controller? Versioned path? Auth via @AuthenticationPrincipal? Filters as
      a record?

  ▸ EXCEPTION layer gate:
    - Right status per type (404/409/400/401/500)? (Message-leak audit is Step 4 —
      just wire the types and statuses correctly here.)

  ▸ CONFIG layer gate:
    - open-in-view:false? profiles present? secrets from env? max-page-size cap set?
      cache keys isolate per user?

  ▸ TEST layer gate:
    - Follows existing BaseIntegrationTest/TestFactory patterns? Real PostgreSQL via
      Testcontainers (not H2)? (Full coverage incl. data-isolation is expanded in Step 4.)

QUALITY BAR (apply silently — don't lecture me)
- Constructor injection; thin controllers; no business logic/DB access in controllers;
  no HTTP/JSON in services. Follow the EXISTING repo's conventions over personal taste.

OUTPUT DISCIPLINE
- Real compilable code in the correct package. Only the current layer's files.
  Don't reprint unchanged files. One conventional-commit message per layer.

Start with the ENTITY layer. Answer its gate, then stop and wait for me.
```

### Your per-layer review checklist (keep beside you — don't send it)

Read each layer against this before you say "next":

- **Entities** — relationship modelling? enum-as-string? LocalDate vs OffsetDateTime? @Version? unique constraints? lazy @ManyToOne?
- **Repositories** — index decision justified by real queries? projection + countQuery? JOIN FETCH vs LEFT JOIN FETCH? ID type matches?
- **DTOs / mapper** — list DTO ≠ detail DTO? mapper edge cases (null/empty/format)?
- **Service** — minimal deps? readOnly? fallible calls → DataPersistenceException?
- **Controller** — validation placed correctly? thin? versioned? auth via principal? page-size cap?
- **Exceptions** — right status per type?
- **Config** — open-in-view:false? profiles? env secrets? per-user cache keys?

If anything's off, say exactly what and why, and have it revised before moving on. This per-layer reading *is* how you end up able to defend the submission live.

---

## STEP 4 — Cross-cutting hardening & audit pass

> Send once the happy path runs end to end. These span every endpoint and are only verifiable against the finished whole — the highest-signal, lowest-line-count changes, and exactly what gets probed.

```
Happy path works and each layer passed its gate. Now audit the WHOLE system for the
cross-cutting properties below. One small commit each; after each, tell me in one line
what changed and why.

1. ERROR-SHAPE CONSISTENCY & LEAK AUDIT (whole surface):
   - Enumerate every error a client can trigger across ALL endpoints.
   - Confirm they all return the SAME error response shape (same JSON structure).
   - Confirm every client-facing message is GENERIC — no class names, SQL, table names,
     constraint names, or internal IDs. Show before/after for anything that leaked.
   - Confirm each maps to the correct status (400/401/404/409/500). Flag any endpoint
     where an invalid param currently produces a 500 instead of a 400.

2. DATA ISOLATION (the single most important security test):
   - Add an explicit, unmistakable test proving one authenticated user can NEVER see
     another user's data through ANY endpoint (list, detail, filtered, paginated).
   - If any endpoint derives the owner from a request param instead of the
     authenticated principal, that's a vulnerability — flag and fix it.

3. AUTH SURFACE AUDIT (if secured):
   - Tests for valid / wrong-password / wrong-user / no-credentials across the
     protected endpoints.
   - Confirm auth failures are generic (no user enumeration) and that no protected
     endpoint is accidentally reachable unauthenticated.

4. GRACEFUL FAILURE (whole surface):
   - Confirm EVERY fallible repository call surfaces as a controlled
     DataPersistenceException → generic 500, never a raw stack trace or framework
     exception to the client. Name any path still missing this.

5. LOGGING DISCIPLINE (cross-cutting):
   - Add SLF4J logging at correct levels: DEBUG flow / INFO success / WARN
     expected-failure & security events / ERROR unexpected (stack trace to log only).
   - Audit that we NEVER log passwords, usernames/emails, request-param values, or
     framework/DB exception messages. Safe: public UUIDs, pagination state, counts.

6. FULL VALIDATION RUN:
   - Run the complete build + test suite. Report status. On failure, root-cause and
     fix — don't just make the test pass. Confirm the final git diff contains only
     intended changes (no debug code, secrets, dead code, stray formatting).

Keep each change minimal and self-contained. Don't refactor unrelated code. Genuine
improvements outside this scope go under "Potential follow-up" — don't implement them.
```

---

## Step 3 / Step 4 ownership (reference)

| Concern | Owner | Why there |
|---|---|---|
| Relationship modelling, enum-as-string, @Version, unique constraints | Step 3 — entity gate | Schema-level, decided when the entity is written |
| **Database indexes** | Step 3 — repository gate | Only justifiable once the queries exist |
| Projections, countQuery, JOIN FETCH vs LEFT JOIN FETCH | Step 3 — repository gate | Part of writing the query correctly |
| Transaction boundaries, DataPersistenceException wrapping | Step 3 — service gate | Part of writing the service correctly |
| **Per-endpoint request validation** (param bounds, UUID, page cap, enum handling) | Step 3 — controller gate | Part of writing the entry point correctly |
| Exception→status wiring | Step 3 — exception gate | Structural, per-type |
| open-in-view, profiles, secrets, cache keys | Step 3 — config gate | Config-level |
| **Error-shape consistency + leak audit** | Step 4 | Only checkable across the whole surface |
| **Data-isolation** | Step 4 | Cross-cutting security property |
| **Auth surface audit** | Step 4 | Spans all protected endpoints |
| **Graceful-failure completeness** | Step 4 | Verified against every path at once |
| **Logging discipline** | Step 4 | Cuts across all layers |

---

## After Step 4 — finishing moves (fire as needed)

- **Runnable proof:**
  ```
  Add a docker-compose (db + app) and a @Profile("dev") data seeder that loads varied
  realistic data (multiple users, every enum value, edge cases like null dates and
  empty collections), idempotent, never active in prod/test. Give me the exact commands
  to run it and hit the endpoints in Postman.
  ```
- **README:**
  ```
  Write a professional README: tech stack, architecture + design-decision rationale,
  setup, run (docker + local), full endpoint reference with request/response examples
  and curl, auth instructions, filtering/pagination, error catalogue, logging strategy,
  test instructions (with the Testcontainers-over-H2 rationale), and a Future
  Enhancements section that names what we DELIBERATELY excluded and why.
  ```
- **Interview rehearsal (Claude as sparring partner — the highest-value step):**
  ```
  Act as the interviewer. For each significant decision in this codebase — ID strategy,
  relationship modelling, indexes, projections, JOIN FETCH, request validation
  placement, AuthenticatedUser, generic errors, data-isolation, Testcontainers, and
  what we deliberately DIDN'T build — ask me one probing question at a time and
  pressure-test my answer. Don't accept vague answers.
  ```

---

## The standing caution

The gates and audits only earn their keep if you **read and answer them yourself** at each stop, not tap "next" on autopilot. The exact things they surface — "why these two indexes and not that one?", "where do you validate this and why there?", "prove a user can't see another user's data", "show an endpoint where an invalid param returns 500 instead of 400" — are the literal follow-up questions. If a gate or audit answer doesn't convince you, dig in before moving on. Speed comes from deciding quickly *because you understood*, never from skipping the read. And confirm the take-home's rules on AI assistance before you rely on any of this.
