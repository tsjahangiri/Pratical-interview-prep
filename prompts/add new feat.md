# Interview — Adding a New POST (Create) Endpoint to an Existing Service

For when the interviewer says *"now add a POST endpoint to create a [thing]."* This is
an **extension of an existing repo**, not a greenfield build, so most of the analysis is
already done. What's different — and what the interviewer is actually testing — is that a
POST **writes** data, which pulls in concerns GET endpoints never had: request-body
validation, `@Transactional` (not readOnly), duplicate/conflict handling, the trust
boundary on the request body, and the created-resource response contract.

**How this maps to the full workflow:** compressed Analyse-&-Lock → Step 3 on only the
layers a create touches → a trimmed Step 4. Under time pressure you can run it as one
flowing conversation — the value isn't the ceremony, it's not skipping the write-specific
concerns.

---

## PHASE 1 — Compressed analyse & lock (recon what changes, ask only what's open)

```
We're adding a new endpoint to the existing service: POST [resource] to create a
[thing]. Before any code:

1. Recon only the files this touches — the target entity, its repository, the existing
   controller, the DTO/mapper pattern already in use, and the exception handler. Don't
   dump them; tell me in a few lines what conventions I must follow (DTO shape,
   validation style, exception types, response pattern).

2. Then ask me — as quick choices — only the decisions this POST genuinely needs:
   - request body: which fields are required vs optional?
   - uniqueness/conflict: is there a business rule that makes a create a duplicate
     (e.g. one X per Y)? what status on violation — 409?
   - response: return 201 + created resource, or 201 + Location header only?
   - any field that needs server-side defaulting or is forbidden in the request
     (id, audit fields, status, version)?
   - who owns the created resource — is it always the authenticated user?

Wait for my answers. Then give me the locked mini-design (endpoint signature, request
DTO, response DTO, validation rules, status codes, service method shape) as text. No
code until I approve.
```

---

## PHASE 2 — Build only the layers a create touches (Step 3, write-specific gates)

```
Approved. Build it layer by layer, stopping after each for my review, touching only
what a create needs. Apply the write-specific gates:

- ENTITY (only if needed): unique constraint for the business rule so duplicates fail
  at the DB, not just in app code. Add the matching index if the uniqueness check
  queries it.

- REQUEST DTO (new): Bean Validation (@NotNull/@NotBlank/@Size/@Positive) on required
  fields; forbid client-supplied id / audit / status / version fields (ignore or
  reject them — the server controls these).

- MAPPER: inbound map from request DTO → entity, ignoring generated fields
  (id, audit timestamps, version). Follow existing MapStruct conventions.

- SERVICE: @Transactional (NOT readOnly — this writes). Check the uniqueness rule and
  throw a conflict exception BEFORE saving; also wrap the save so a
  DataIntegrityViolationException becomes a clean domain conflict exception (this
  covers the race where two requests pass the app-level check simultaneously). Set the
  owner from the passed-in authenticated user, never from the request. Return the
  persisted entity.

- CONTROLLER: POST mapping, @Valid on the body, 201 with the created resource (or 201 +
  Location header per the locked design), auth via @AuthenticationPrincipal, owner
  taken from the principal — never from the request body. Versioned path, consistent
  with existing endpoints.

- EXCEPTION: map the conflict to 409 and validation failures to 400, both in the
  EXISTING error-response shape. Generic messages — no constraint/table/class names.

- TESTS: follow existing patterns; covered in Phase 3.

Follow the existing repo's conventions exactly. Real compilable code, current layer
only, one conventional-commit message per layer. Start with whichever layer is first
and stop for my review.
```

---

## PHASE 3 — Create-specific hardening (trimmed Step 4)

```
Now the create-specific hardening:

1. Confirm an invalid/missing required field returns 400 in the SAME error shape as the
   rest of the API — not a 500, not a raw validation dump.

2. Confirm the duplicate-create path returns 409 with a generic message (no
   constraint/table names leaked), and that the DB unique constraint — not just an app
   check — is the real guard. Confirm the race (two simultaneous identical creates) is
   handled by catching the DB integrity violation, not only the pre-check.

3. Tests:
   - happy-path create → assert 201 AND the resource was actually persisted (read it
     back / assert DB state)
   - validation failure → 400
   - duplicate → 409
   - unauthorized → 401
   - CRITICAL: a user can only create a resource owned by themselves — attempting to
     set another owner via the request body must not succeed.

Run the suite; report status. Keep changes minimal; anything out of scope goes under
"Potential follow-up".
```

---

## What the interviewer is really testing with a POST (be ready to say these out loud)

**1. Idempotency / duplicate handling — the most likely follow-up.**
"What happens if this POST is sent twice?"
- Naive answer: creates two rows.
- Senior answer: a **unique constraint at the DB level** makes the second fail; the
  service returns **409**; and you can discuss the **race condition** — two simultaneous
  requests both pass the app-level existence check, and only the DB constraint (caught as
  `DataIntegrityViolationException` → 409) actually saves you. The app check is a nicety;
  the DB constraint is the guarantee.

**2. Trust boundary on the request body — a security question in disguise.**
"What stops me creating a resource owned by another user, or setting my own id/status?"
- Owner comes from the **authenticated principal**, never the body.
- `id`, audit fields, `version`, and `status` are **server-controlled** — ignored or
  rejected if a client sends them.
- This is why the data-isolation test extends to creates: you can't spoof ownership.

**3. The right status codes and response contract.**
- **201 Created**, not 200. Return the created resource (with its server-generated id) or
  a `Location` header — state which and why.
- **400** for validation, **409** for conflict, **401** for unauthenticated — all in the
  same error shape as the rest of the API.

**4. Transaction correctness.**
- `@Transactional` (not `readOnly`). If the create involves multiple writes, they commit
  or roll back together. Domain exceptions extend `RuntimeException` so a thrown conflict
  rolls the transaction back.

---

## Quick reference — GET vs POST concerns

| Concern | GET (what you already did) | POST (new) |
|---|---|---|
| Transaction | `readOnly = true` | `@Transactional` (writes) |
| Body validation | none | `@Valid` + Bean Validation on required fields |
| Success status | 200 | **201** (+ resource or Location) |
| Conflict | n/a | **409** via DB unique constraint, race-safe |
| Trust boundary | owner-from-principal on read | owner-from-principal on write; forbid client id/status |
| Failure to guard | leaks another user's data | lets a user spoof ownership / create duplicates |

---

## The standing caution

For a single endpoint under time pressure, run this as one flowing conversation, not
four rigid sends — but do **not** skip the write-specific concerns (validation,
transaction, conflict, ownership), because those are precisely what an interviewer probes
on a POST and precisely what GET endpoints let you ignore. Read every gate answer
yourself: "what happens on a double-submit?" and "what stops me spoofing the owner?" are
the questions behind this whole flow. If Claude's answer doesn't convince you, dig in
before you move on — it's your name on the submission, and your voice in the follow-up.
```
