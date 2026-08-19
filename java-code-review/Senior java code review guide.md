# Senior Java / Spring Boot — Live Code Review Interview Playbook

> A training guide for passing **live code-review interviews** at senior/staff level.
> Assumes Java 17+, Spring Boot 3.x, production microservice context.
> The goal is not just to *find* bugs, but to **READ → QUESTION → REASON → PRIORITIZE → EXPLAIN → FIX → DISCUSS TRADE-OFFS**.

---

## How to use this guide

Read Parts 1–10 once to internalize the framework and the topic catalog. Then drill Part 11 (the fully-worked `ReportGenerator` review) until you can reproduce the *reasoning*, not memorize the answers. Part 13 is a fresh exercise with a spoiler-gated answer key — attempt it cold. Part 15 (how to speak) and Part 18 (the 90-second checklist) are what you actually run in the room.

---

# PART 1 — The Senior Code-Review Framework

## The mindset shift

A junior reads code top-to-bottom and reacts to syntax. A senior reads code as a **system under load, under failure, and under change**. Before you comment on a single line, you answer three framing questions out loud:

1. **What is this code supposed to do?** (business intent — ask if unclear)
2. **What are the invariants and constraints?** (money must balance, IDs unique, this runs on 20 pods, etc.)
3. **What is the blast radius if it's wrong?** (a formatting nit vs. a lost financial transaction)

Everything after that is prioritized by blast radius, not by how odd the line looks.

## The two-pass method (what to actually do in 30–60 min)

**Pass 1 — Orient (5 min).** Skim for structure. Identify entry points, I/O boundaries (DB, network, files, queues), shared mutable state, and transaction boundaries. Say what the code *does* back to the interviewer. This buys you time and demonstrates you read for intent, not pattern-match.

**Pass 2 — Scan by risk, high-to-low.** Run the checklist below in *severity order* so that if you run out of time you've already covered the things that page someone at 3 a.m.

## The C-R-E-A-M-S mental checklist

A compressible acronym you can run live. Six buckets, each with a one-word trigger:

- **C — Correctness & Concurrency**: Does it do the right thing? Is there shared mutable state, a race, a visibility problem, a missed `volatile`/lock? Are transaction boundaries correct?
- **R — Resources & Reliability**: Streams/connections/threads closed? Retries idempotent? Timeouts set? What happens when the DB/queue/API is down or slow?
- **E — Edge cases & Errors**: Nulls, empty collections, zero/negative, overflow, huge inputs. Are exceptions handled or swallowed? Is failure observable?
- **A — API & Abstraction**: HTTP semantics, status codes, DTO vs entity leakage, validation, versioning, SOLID violations, leaky abstractions.
- **M — Money, Memory & Measurement**: `double` for money (never), unbounded collections/result sets, O(n²), object churn. And the discipline: *measure before optimizing*.
- **S — Security & Storage**: Injection, secrets/PII in logs, missing authz, and the data layer — N+1, missing indexes, long transactions, locking.

If you can only remember one thing: **shared state, transaction boundaries, resource leaks, and money-as-double** are where senior candidates separate from the pack. Those four are almost always present and almost always high-severity.

## The 20-point catalog (reference, not a live script)

Correctness · Business logic · Edge cases · Null handling · Exception handling · Resource management · Concurrency/thread safety · Performance · Memory · DB interaction · Transactions · API design · Security · Logging/observability · Testing · Maintainability · SOLID · Scalability · Reliability/fault tolerance · Distributed-system concerns.

Don't recite this. Fold it into C-R-E-A-M-S. The catalog is what you *cover*; the acronym is how you *remember to cover it*.

---

# PART 2 — Java Topics You Must Be Able to Review

For each: the concept, a broken sample, what a senior *notices*, the fix, and the trap.

## Immutability & records
**Concept:** Immutable objects are inherently thread-safe and cache-friendly. `record` gives you final fields, canonical constructor, `equals`/`hashCode`/`toString` for free.
**Broken:**
```java
record Order(String id, List<Item> items) {}   // items is mutable — record is a lie
```
**Senior notices:** The record *looks* immutable but exposes a mutable `List`. A caller can mutate `order.items()`.
**Fix:** Defensive copy in a compact constructor: `items = List.copyOf(items);` and it's already unmodifiable on the way out.
**Trap:** Records are only *shallowly* immutable. Interviewers love handing you a record holding a `Date`, array, or `List`.

## equals / hashCode
**Concept:** Contract — equal objects must have equal hashCodes; both must be consistent and use the same fields.
**Broken:**
```java
class Money {
    BigDecimal amount;
    public boolean equals(Object o){ /* compares amount */ }
    // no hashCode override
}
```
**Senior notices:** Overriding `equals` without `hashCode` breaks every hash-based collection — the object vanishes from `HashSet`/`HashMap`.
**Fix:** Override both, or use a record. For `BigDecimal`, note that `equals` treats `2.0` and `2.00` as *unequal* (scale-sensitive) — use `compareTo` for value equality.
**Trap:** Mutable fields used in `hashCode` → object gets lost after mutation while in a set.

## String vs StringBuilder vs StringBuffer
**Concept:** `String` is immutable; concatenation in a loop is O(n²). `StringBuilder` is not synchronized (fast, single-thread); `StringBuffer` is synchronized (rarely needed).
**Broken:** `s += x` inside a loop. (See Part 11 — this is exactly BUG 2.)
**Senior notices:** Loop concatenation allocates a new char array each iteration.
**Fix:** `StringBuilder` sized to an estimate. But — *the compiler already rewrites a single-expression concat into StringBuilder*; the anti-pattern is specifically the **loop**.
**Trap:** Being told "use StringBuffer for thread safety" when nothing is shared — that's just slower for no reason.

## HashMap internals
**Concept:** Array of buckets, hash → index, collisions chained as linked list, treeified to a red-black tree at ≥8 entries per bucket (with capacity ≥64). Load factor 0.75 triggers resize/rehash.
**Senior notices:** A poor/constant `hashCode` degrades to O(n) per bucket. Mutating a key after insertion corrupts lookups.
**Trap:** "What thread-safety does HashMap give you?" — none. Concurrent writes can even spin the CPU (historically infinite loop on resize in Java 7).

## ConcurrentHashMap
**Concept:** Thread-safe, lock-striped, non-blocking reads. But **compound actions are not atomic**.
**Broken:**
```java
if (!map.containsKey(k)) map.put(k, v);   // check-then-act race
```
**Fix:** `map.putIfAbsent(k, v)` or `map.computeIfAbsent(k, key -> v)`.
**Trap:** People think using `ConcurrentHashMap` makes *their whole method* thread-safe. It only makes each *single* operation atomic.

## Collection selection
- `ArrayList` — random access, cache-friendly; O(n) middle inserts.
- `LinkedList` — almost never the right answer in interviews.
- `HashSet`/`HashMap` — O(1) average, no order.
- `TreeMap`/`TreeSet` — sorted, O(log n).
- `LinkedHashMap` — insertion order; also the classic LRU cache (`removeEldestEntry`).
**Trap:** Choosing `LinkedList` "because lots of inserts" without noting random-access cost, or picking `Vector`/`Hashtable` (legacy, synchronized).

## Streams
**Concept:** Declarative pipelines. Lazy until a terminal op.
**Broken:**
```java
list.stream().forEach(x -> results.add(x));      // side effects into shared list
list.parallelStream().forEach(...);              // shared mutable state → race
```
**Senior notices:** Side-effecting `forEach`, especially parallel; `peek` used for logic; a stream where a plain loop is clearer.
**Fix:** `collect(Collectors.toList())`; keep pipelines side-effect-free.
**Trap:** `parallelStream()` presented as a free speedup — it uses the common ForkJoinPool, blocks siblings on I/O, and only helps CPU-bound work on large N.

## Optional
**Concept:** A return-type tool to signal absence. **Not** for fields or parameters.
**Broken:** `optional.get()` without `isPresent`; `Optional<X>` as a class field; `Optional` in JPA entities.
**Fix:** `orElseThrow`, `map`, `orElseGet`.
**Trap:** `orElse(expensiveCall())` always evaluates the argument — use `orElseGet(() -> ...)`.

## Exceptions & try-with-resources
**Concept:** Don't swallow; don't use exceptions for control flow; close resources deterministically.
**Broken:**
```java
try { ... } catch (Exception e) { e.printStackTrace(); }   // swallow + no context
FileWriter fw = new FileWriter(f); fw.write(s);            // never closed (Part 11 BUG 5)
```
**Fix:** `try (var w = Files.newBufferedWriter(path)) { ... }`; log with context and rethrow/wrap.
**Trap:** Catching `Exception`/`Throwable` broadly and hiding `InterruptedException` (must restore the interrupt flag).

## BigDecimal / money
**Concept:** `double`/`float` cannot represent decimal fractions exactly — money math drifts.
**Broken:** `double amount` (Part 11 — flagged by *no comment*, which is the point).
**Fix:** `BigDecimal` constructed from `String` (`new BigDecimal("150.50")`, never from a double), with an explicit `RoundingMode` and scale.
**Trap:** `BigDecimal.equals` is scale-sensitive; use `compareTo`.

## Date/Time API
**Concept:** `java.time` (immutable, thread-safe) replaces `Date`/`Calendar`/`SimpleDateFormat`.
**Broken:** `SimpleDateFormat` as a shared field across threads (not thread-safe → silent data corruption).
**Fix:** `DateTimeFormatter` (immutable) as a `static final` constant.
**Trap:** See Part 11 — the *naive* fix of making a `SimpleDateFormat` static introduces a concurrency bug. `DateTimeFormatter` is the correct static.

## enums, interfaces vs abstract classes, composition vs inheritance, DIP
- **Enums:** singletons with behavior; great for strategy tables and finite state.
- **Interface vs abstract class:** interface for capability/multiple inheritance of type; abstract class for shared state + template methods. Default methods blur this — note diamond ambiguity.
- **Composition over inheritance:** favor delegating to collaborators; inheritance couples you to the parent's implementation and breaks encapsulation.
- **Dependency Inversion:** depend on abstractions; in Spring, inject interfaces via the constructor, not `new` concretes inside methods.

## Code smells to name on sight
God class, long method, feature envy, primitive obsession (money-as-double is one), shotgun surgery, leaky abstraction (entity returned from controller), boolean-parameter flags, deep nesting, magic numbers (the literal `100` thread pool in Part 11).

---

# PART 3 — Concurrency & Multithreading (how to spot bugs in review)

Concurrency bugs are the highest-signal thing you can catch. For each, the pattern is: **broken → what you notice → why it's dangerous → fix → what you say.**

## Race condition (check-then-act / read-modify-write)
```java
private int counter = 0;
public void inc() { counter++; }   // not atomic: read, add, write
```
**Notice:** shared mutable field mutated without synchronization from multiple threads.
**Dangerous:** lost updates; the value is nondeterministic under load.
**Fix:** `AtomicInteger.incrementAndGet()`, or `synchronized`, or a `LongAdder` under high contention.
**Say:** *"`counter++` is a read-modify-write, so with concurrent callers we lose updates. I'd use an `AtomicInteger` here, or `LongAdder` if this is a hot counter with high contention."*

## Visibility (missing volatile)
```java
private boolean running = true;      // writer thread flips it; reader may never see the change
while (running) { ... }
```
**Notice:** a flag written by one thread and read by another with no memory barrier.
**Fix:** `volatile boolean running`, or an `AtomicBoolean`.
**Say:** *"Without `volatile`, there's no happens-before edge, so the reader can cache `running` forever — the loop may never terminate."*

## Atomicity vs. visibility
`volatile` gives visibility, **not** atomicity. `volatile int x; x++` is still a race. Atomicity needs `Atomic*` or locks.

## synchronized / Lock / ReentrantLock
**Notice:** synchronizing on `this` or a public object (external code can lock you); locking too coarsely (contention) or too finely (races). `ReentrantLock` acquired without a `try/finally { unlock() }`.
**Fix:** private final lock object; `lock.lock(); try { ... } finally { lock.unlock(); }`.
**Say:** *"This locks on `this`, which any external caller can also synchronize on — I'd use a private final lock to avoid unexpected contention or deadlock."*

## Deadlock / starvation / contention
**Notice:** two locks acquired in different orders on different paths → classic deadlock.
**Fix:** global lock ordering; `tryLock` with timeout; reduce critical-section size.
**Say:** *"Path A takes lock X then Y; path B takes Y then X — that's a lock-ordering deadlock. I'd enforce a single acquisition order or use `tryLock` with a timeout."*

## ExecutorService / thread pools / bounded queues
```java
ExecutorService ex = Executors.newFixedThreadPool(100);   // magic number, never shut down
```
**Notice:** unbounded default queues (`newFixedThreadPool` uses an *unbounded* `LinkedBlockingQueue` → OOM under backpressure); pool sized by magic number; never `shutdown()` (Part 11 BUG 3); exceptions from `submit` swallowed by ignored `Future`s.
**Fix:** a `ThreadPoolExecutor` with an explicit bounded queue and a `RejectedExecutionHandler`; size by workload (CPU-bound ≈ cores; I/O-bound higher, measured); `shutdown()` + `awaitTermination()`; in Spring, a managed `ThreadPoolTaskExecutor` bean.
**Say:** *"`newFixedThreadPool` hides an unbounded queue, so under load we accumulate tasks until OOM instead of applying backpressure. I'd use a bounded `ThreadPoolExecutor` with an explicit rejection policy, and I'd size it after measuring whether this is CPU- or I/O-bound."*

## Future / CompletableFuture
**Notice:** `future.get()` with no timeout (blocks forever); exceptions lost because `Future` result never read; blocking calls on the common pool.
**Fix:** `get(timeout, unit)`; `CompletableFuture` with `exceptionally`/`handle`; a dedicated executor for blocking work.

## Callable vs Runnable
`Runnable` can't return a value or throw checked exceptions; `Callable<V>` can. If a task swallows its result/exception, prefer `Callable` + collected `Future`s so failures surface.

## Graceful shutdown & cancellation
**Notice:** no `shutdown()`/`awaitTermination()`; ignoring `InterruptedException`; long tasks that never check `Thread.interrupted()`.
**Fix:** shutdown → awaitTermination(timeout) → shutdownNow; restore the interrupt flag: `Thread.currentThread().interrupt();`.

## Parallel streams / common ForkJoinPool
**Notice:** `parallelStream()` doing blocking I/O — it starves the shared common pool used by the whole JVM.
**Say:** *"Parallel streams run on the shared common ForkJoinPool. Blocking I/O here can starve unrelated parallel work across the app. For I/O I'd use a dedicated executor."*

---

# PART 4 — Spring Boot Code Review

## Dependency Injection
**Field injection is a smell:**
```java
@Autowired private OrderService service;   // field injection
```
**Senior notices:** hides required dependencies, breaks immutability, can't be `final`, makes unit testing harder (needs reflection), and masks too-many-dependencies (a God bean).
**Fix:** constructor injection (Spring auto-wires a single constructor; no `@Autowired` needed):
```java
private final OrderService service;
OrderController(OrderService service) { this.service = service; }
```
**Circular dependencies:** two constructor-injected beans referencing each other fail at startup — that's a *design* signal to extract a third collaborator, not to paper over with `@Lazy`.
**Scopes:** default singleton beans holding request/mutable state is a classic bug — a singleton with an instance field mutated per request is a race across all users. **Bean lifecycle:** heavy work in a constructor vs `@PostConstruct`.

## REST
- **HTTP semantics:** GET must be safe/idempotent and take no body; POST creates (201 + Location); PUT/DELETE idempotent; PATCH partial. Returning 200 for a creation, or 200 with an error body, is wrong.
- **Status codes:** 400 vs 404 vs 409 vs 422; never 200-with-error-payload; 500 only for genuine server faults.
- **Validation:** `@Valid` on the DTO + Bean Validation annotations; validate at the boundary, not deep in services.
- **DTOs:** never accept or return JPA entities directly — leaks the schema, invites over-posting, and drags lazy associations into serialization (LazyInitializationException / N+1 at render time).
- **Error handling:** centralize with `@RestControllerAdvice` + `@ExceptionHandler`; return a consistent problem shape (RFC 7807 `ProblemDetail`), never a stack trace.
- **Pagination:** unbounded `findAll()` behind an endpoint is a production incident waiting for the table to grow; require `Pageable` with a max page size.
- **Idempotency:** POST that can be retried (payments!) needs an idempotency key.
- **AuthN/AuthZ:** authentication ≠ authorization; check ownership (`@PreAuthorize` / method security), not just "is logged in." Watch for IDOR (any user can fetch `/orders/{id}`).
- **Versioning:** URI (`/v1`) or header; breaking changes need a version.

**Say (entity leak):** *"The controller returns the `Order` entity directly. Beyond coupling the API to the schema, if `items` is `LAZY` we'll either hit a `LazyInitializationException` during serialization or trigger N+1 lazy loads. I'd map to a DTO inside the transaction."*

## Spring Transactions (highest-value Spring topic)
- **`@Transactional` only works through the Spring proxy.** A `private` method, or a method called via `this.method()` **self-invocation**, bypasses the proxy → **no transaction at all**. This is the #1 hidden Spring bug.
```java
@Service
class Svc {
    public void outer() { inner(); }            // self-call — proxy NOT applied
    @Transactional public void inner() { ... }  // silently non-transactional here
}
```
- **Propagation:** `REQUIRED` (default, join existing), `REQUIRES_NEW` (suspend + independent tx — use for audit/outbox rows that must survive a rollback), `NESTED` (savepoint). Misusing `REQUIRES_NEW` in a loop = connection exhaustion.
- **Isolation:** default is DB default (usually READ_COMMITTED). Higher isolation → more locking. Know the anomalies (dirty/non-repeatable/phantom reads).
- **Rollback rules:** Spring rolls back on unchecked exceptions **only** by default. A checked exception does **not** trigger rollback unless you set `rollbackFor`. Catching an exception inside the method also silently commits.
- **`readOnly = true`:** a hint (flush mode, some driver optimizations); not a guarantee.
- **Boundaries:** transactions should be short. `@Transactional` on a method that also makes a REST call holds a DB connection open across network latency → pool exhaustion. Never do I/O inside a transaction.
- **Lost update / locking:** two reads + writes without a version column overwrite each other. Fix with **optimistic locking** (`@Version`) for low-contention, **pessimistic** (`SELECT ... FOR UPDATE`, `@Lock(PESSIMISTIC_WRITE)`) for high-contention hot rows.

**Say (self-invocation):** *"`outer()` calls `inner()` on `this`, so the `@Transactional` on `inner` never fires — the proxy is bypassed. Either move `inner` to another bean or make `outer` transactional. This is silent, which makes it dangerous."*

---

# PART 5 — JPA / Hibernate Code Review

## N+1 queries
**Broken:**
```java
List<Order> orders = orderRepo.findAll();
for (Order o : orders) o.getItems().size();   // 1 query + N lazy loads
```
**Detect:** a loop touching a lazy association; `show_sql`/`p6spy` reveals the flood; the classic "list endpoint got slow as data grew."
**Fix:** `JOIN FETCH` (`@Query("select o from Order o join fetch o.items")`), `@EntityGraph`, or a projection/DTO query. For collections, beware fetch-join + pagination (Hibernate paginates in memory — use two queries or `@BatchSize`).
**Say:** *"This is a textbook N+1: one query for orders, then one per order for items. I'd use a fetch join or an entity graph. If I need pagination too, I'd split it because a collection fetch join can't paginate at the DB level."*

## Lazy vs eager / LazyInitializationException
- `@ManyToOne` defaults to **EAGER** (often undesirable — set `LAZY`); `@OneToMany` defaults to LAZY.
- Accessing a lazy association after the session closes (e.g., in the controller/serializer) → `LazyInitializationException`. **Fix by mapping to a DTO inside the transaction**, *not* by switching to EAGER (which reintroduces N+1 and over-fetching everywhere).

**Say (the trap):** *"I'd resist just flipping this to EAGER — that fixes the exception but over-fetches on every query and can bring back N+1. The right fix is to load what this use case needs, inside the transaction, into a DTO."*

## Cascade / orphanRemoval
`CascadeType.ALL` including `REMOVE` on a `@ManyToOne` can delete shared parents. `orphanRemoval = true` deletes children removed from the collection — correct for true composition, dangerous for shared references.

## Projections & pagination
Prefer interface/DTO projections for read endpoints (fetch only needed columns). `OFFSET` pagination degrades on deep pages — mention keyset/seek pagination for large tables.

## When a suspicious JPA pattern is actually fine
- **EAGER on a small, always-needed reference** (e.g., an `enum`-like lookup) is reasonable.
- **`findAll()`** on a bounded config/reference table (currencies, countries) is fine.
- **A lazy association not accessed** in a code path costs nothing.
Say so explicitly — recognizing a *non*-issue is senior signal.

---

# PART 6 — Database & Performance Review

Targets: N+1, missing indexes, full scans, inefficient joins, redundant DB calls, unbounded result sets, missing pagination, **connection-pool exhaustion** (I/O inside transactions, leaked connections, `REQUIRES_NEW` in loops), long transactions, locking, O(n²) in-memory joins, excessive object creation.

**The senior discipline: reason, don't guess.** The correct answer is often:

> *"This might be slow, but I wouldn't change it blind. I'd measure — check the query plan with `EXPLAIN ANALYZE`, profile under representative load, confirm it's actually the bottleneck. Premature optimization here could add complexity for no measured gain."*

**When "I'd profile it first" is right:** when the concern is *performance* and the fix adds complexity (caching, denormalization, parallelism, a rewrite). **When it's a cop-out:** when it's a *correctness* bug (N+1 that's obviously O(n) calls, a missing index on a known-hot WHERE column, `double` for money, a missing `LIMIT` on a growing table) — those you call out directly, no profiling needed.

Frame the distinction: *"Correctness and obvious algorithmic problems I'll flag now. For anything where the fix trades simplicity for speed, I'd want a measurement before committing."*

---

# PART 7 — Microservices / Distributed Systems Review

## Retries without idempotency (dangerous)
```java
// retry 3x on failure — but the call is a payment charge
retryTemplate.execute(ctx -> paymentClient.charge(order));
```
**Notice:** retrying a non-idempotent operation double-charges. Retries need an **idempotency key** so the server dedups.
**Say:** *"Blind retries on a non-idempotent charge risk double-charging on a network blip where the request succeeded but the response was lost. I'd pass an idempotency key and let the provider dedup."*

## Timeouts & circuit breakers
**Notice:** REST/HTTP client with **no connect/read timeout** → a slow dependency exhausts your threads and cascades. No circuit breaker → repeated hammering of a dead service.
**Fix:** explicit timeouts everywhere; Resilience4j circuit breaker + bulkhead + fallback.

## Kafka: duplicates, ordering, offsets
- **At-least-once delivery** means **consumers must be idempotent** — process the same message twice safely (dedup by key/event id).
- **Auto-commit before processing** loses messages on crash; **commit after** processing risks reprocessing (acceptable if idempotent).
- **Ordering** is per-partition only; keys route related events to the same partition.
- **Poison messages** need a **dead-letter topic** and bounded retries, or one bad record blocks the partition forever.
- **Producing inside a DB transaction** isn't atomic with the DB → use the **Outbox pattern** (write event to an outbox table in the same tx, relay to Kafka separately).

## Dual-write / distributed transaction
**Broken:** save to DB, then publish to Kafka. A crash between the two leaves them inconsistent. There is no distributed transaction across DB + broker — **Outbox** or **Saga**, not 2PC.

## Partial failure
**Notice:** a loop calling N services with no per-item error isolation — one failure aborts the batch. Decide: fail-fast vs. collect-and-continue with a report.

---

# PART 8 — Security Review (Java/Spring)

- **SQL injection:** string-concatenated queries / `@Query` with concatenation, or native queries built from input. **Fix:** parameter binding (`?`/`:named`), never concatenate.
- **Command injection / SSRF:** `Runtime.exec` with user input; fetching a URL supplied by the user (SSRF — can hit internal metadata endpoints). Allowlist, validate, isolate.
- **Path traversal:** `new File(baseDir, userFilename)` with `../` in input. Canonicalize and verify the resolved path stays under the base.
- **Insecure deserialization:** native Java deserialization of untrusted input → RCE. Avoid; use JSON with strict typing.
- **Sensitive data in logs:** logging full request bodies, tokens, PII, card numbers. **Fix:** mask; never log secrets; `printStackTrace` to stdout is both an observability *and* a leak smell.
- **Exposed secrets:** credentials in code/`application.yml` committed to VCS. Externalize (env/secret manager).
- **Missing authorization / IDOR:** endpoint authenticates but doesn't verify the resource belongs to the caller.
- **Broken input validation:** trusting client-supplied fields (price, role, isAdmin — over-posting/mass-assignment). Use explicit DTOs; never bind entities from request bodies.

**Say:** *"This builds SQL by concatenating the `name` param — that's an injection vector. I'd switch to a bound parameter. And separately, we're logging the full auth header, which puts a bearer token in our logs."*

---

# PART 9 — Testing Review

What senior reviewers expect to see (and flag when missing):
- **Behavior over implementation:** tests assert outcomes, not that a private method was called. Over-mocking internal calls → brittle tests that pass while the system is broken.
- **Real assertions:** `assertTrue(result != null)` is weak; assert the actual value. A test with no assertion (only a mock verify, or nothing) is theater.
- **Isolation & determinism:** no shared mutable static state between tests; no reliance on wall-clock/`now()` (inject a `Clock`); no test ordering dependence — sources of **flaky tests**.
- **Edge cases:** empty/null/boundary/huge inputs, not just the happy path.
- **The right test level:** unit tests for logic (mock collaborators with Mockito); **`@DataJpaTest`** for repository queries (mocking the repo can't catch a broken JPQL/N+1); **Testcontainers** for real DB behavior; `@SpringBootTest` sparingly (slow).
- **Concurrency tests:** race conditions rarely surface in single-threaded tests — flag their absence for concurrent code.

**Say:** *"These service tests mock the repository, which is fine for logic, but nothing exercises the actual query — a broken JPQL or an N+1 would pass CI. I'd add a `@DataJpaTest` (ideally Testcontainers) for the repository layer."*

---

# PART 10 — Production-Readiness Interrogation

For **every** sample, ask these aloud — they generate 80% of senior-level findings:

- DB down / slow? Queue down? External API times out or 500s?
- Request retried → can the operation run **twice**? Can data be **lost** or **duplicated**?
- Under high load / with 10M records — does memory stay bounded? Is there pagination/streaming?
- A worker thread throws — is it observed, or silently swallowed?
- Can resources (connections, threads, file handles, streams) **leak**?
- Can we **observe** the failure — metrics, structured logs with correlation id, traces, alerts?
- Is it **idempotent**? Does it degrade gracefully (timeout, fallback, circuit breaker)?

*"The code compiles"* is the floor. *"Here's what happens when the third dependency is slow at 2 a.m. and someone retries"* is the bar.

---

# PART 11 — Deep Review of `ReportGenerator` (independently verified)

**Ground rule applied:** I did not trust the `// BUG N` comments. Each is verified against the code, and I flag issues the comments *missed* — because in a real interview the comments are noise, and the misses are where you earn senior signal.

## Verdict table

| # | Claim / Finding | Real? | Severity | One-line reason |
|---|---|---|---|---|
| BUG 1 | `new SimpleDateFormat` per item | ✅ Real (inefficiency) | MEDIUM | Object churn per row; but the *naive* static fix is a concurrency bug — see below |
| BUG 2 | String `+=` concatenation in loop | ✅ Real | MEDIUM | O(n²) allocation; use `StringBuilder` |
| BUG 3 | ExecutorService never shut down | ✅ Real | HIGH | Non-daemon threads leak; JVM won't exit; repeated calls exhaust threads |
| BUG 4 | Timestamp filename collision | ✅ Real (under concurrency) | MEDIUM–HIGH | 100 threads → same millisecond → files silently overwrite each other |
| BUG 5 | Unclosed `FileWriter` | ✅ Real | HIGH | File-handle leak + unflushed data can be lost |
| **MISS A** | `double amount` for money | ✅ Real (**not flagged**) | **HIGH** | Financial precision loss — the worst bug, and no comment mentions it |
| **MISS B** | Task exceptions swallowed | ✅ Real (**not flagged**) | MEDIUM | `submit` returns `Future`s that are ignored; failures vanish |
| **MISS C** | No `awaitTermination` / completion signal | ✅ Real (**not flagged**) | MEDIUM | Caller can't know when export finished or whether it succeeded |
| MISS D | CSV: no escaping, no header, `double.toString` format | ✅ Real (**not flagged**) | LOW–MED | Commas/newlines in fields corrupt the file; locale-dependent formatting |
| MISS E | Magic number `100`; `printStackTrace` | ✅ Real (**not flagged**) | LOW | Unmeasured pool size; log to a logger, not stdout |
| **NOT-A-BUG 1** | "`generateReport` is called from many threads — is that a race?" | ❌ Not a bug | — | It only touches locals + its argument; no shared mutable state → thread-safe |
| NOT-A-BUG 2 | Indexed `for` loop instead of for-each | ❌ Not a bug | NIT | Style only; correctness identical |

## Issue-by-issue

### BUG 1 — `new SimpleDateFormat("yyyy-MM-dd")` per iteration ✅ (with a critical nuance)
**Is it a bug?** It's a real *inefficiency* — you allocate and configure a formatter for every transaction. It is **not** a thread-safety bug *as written*, because each formatter is local to one iteration on one thread.
**Impact/severity:** MEDIUM (object churn, GC pressure at scale). Not correctness.
**The nuance that earns points:** The obvious "fix" — hoist it to a `static final SimpleDateFormat` — is **worse**. `SimpleDateFormat` is **not thread-safe**, and this method runs on a 100-thread pool. A shared `SimpleDateFormat` would silently corrupt formatted dates under concurrency.
**Correct fix:**
```java
private static final DateTimeFormatter DATE_FMT = DateTimeFormatter.ofPattern("yyyy-MM-dd");
// per row:  DATE_FMT.format(t.getDate())   // after migrating Date -> LocalDate
```
`DateTimeFormatter` is immutable and thread-safe, so a single static instance is both fast and correct.
**Say:** *"Creating a `SimpleDateFormat` per row is wasteful, but I want to be careful: the naive fix of a static `SimpleDateFormat` would be a concurrency bug, since it's not thread-safe and this runs on a thread pool. The right move is a static `DateTimeFormatter` from `java.time`."*

### BUG 2 — String concatenation in a loop ✅
`report += ...` builds a new `String` (and backing array) each iteration → **O(n²)**. The compiler optimizes a *single* concatenation expression into a `StringBuilder`, but **not** an accumulating loop.
**Fix:**
```java
StringBuilder sb = new StringBuilder(transactions.size() * 40);
for (Transaction t : transactions) {
    sb.append(t.getId()).append(',')
      .append(t.getAmount()).append(',')
      .append(DATE_FMT.format(t.getDate())).append('\n');
}
return sb.toString();
```
**Severity:** MEDIUM (performance; degrades badly on large batches). **Say:** *"Loop concatenation is O(n²) here — I'd use a pre-sized `StringBuilder`."*

### BUG 3 — ExecutorService never shut down ✅ (HIGH)
```java
ExecutorService executor = Executors.newFixedThreadPool(100);
// ... submit tasks ... method returns. No shutdown().
```
**Impact:** `newFixedThreadPool` creates **non-daemon** threads. Without `shutdown()`, they never terminate → the JVM won't exit (the comment is accurate). Worse in production: **every call to `exportReports` creates a brand-new 100-thread pool that is never reclaimed** → thread leak → eventual `OutOfMemoryError: unable to create new native thread`. Also, `newFixedThreadPool`'s **unbounded** queue means a flood of batches accumulates in memory instead of applying backpressure.
**Fix:** a shared, managed, bounded executor; shut it down; wait for completion:
```java
ExecutorService executor = new ThreadPoolExecutor(
        cores, cores, 60L, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(1000),
        new ThreadPoolExecutor.CallerRunsPolicy());
try {
    List<Future<?>> futures = new ArrayList<>();
    for (List<Transaction> batch : batches) {
        futures.add(executor.submit(() -> saveToFile(generateReport(batch))));
    }
    for (Future<?> f : futures) f.get();   // surfaces exceptions (MISS B/C)
} finally {
    executor.shutdown();
    executor.awaitTermination(1, TimeUnit.MINUTES);
}
```
In Spring, inject a `ThreadPoolTaskExecutor` bean instead of creating pools inside a method.
**Severity:** HIGH. **Say:** *"This leaks the entire thread pool on every invocation and never shuts down. I'd use a shared bounded executor, collect the futures so failures surface, and shut it down in a finally block."*

### BUG 4 — Timestamp filename collision ✅ (MEDIUM–HIGH under concurrency)
`"report_" + System.currentTimeMillis() + ".txt"` — with 100 threads finishing near-simultaneously, **multiple batches produce the same millisecond**, so files **silently overwrite** each other and reports are lost. This is only a bug *because* of the concurrent context — worth stating that dependency.
**Fix:** deterministic unique names — include a batch index or a `UUID`/atomic counter: `"report_" + batchId + "_" + counter.incrementAndGet() + ".txt"`.
**Say:** *"Under the 100-thread pool, `currentTimeMillis` collides and reports overwrite each other. I'd derive the filename from the batch identity or an atomic sequence."*

### BUG 5 — Unclosed `FileWriter` ✅ (HIGH)
```java
FileWriter fw = new FileWriter(...);
fw.write(report);   // never closed
```
**Impact:** leaks a file descriptor on every call (FD exhaustion under load), and buffered/OS data may **not be flushed** → truncated/empty files. `printStackTrace` on failure also loses the error into stdout.
**Fix:** try-with-resources + buffering + a logger:
```java
Path path = Path.of("report_" + id + ".txt");
try (BufferedWriter w = Files.newBufferedWriter(path)) {
    w.write(report);
} catch (IOException e) {
    log.error("Failed to write report {}", path, e);
    throw new UncheckedIOException(e);   // don't swallow
}
```
**Severity:** HIGH. **Say:** *"The `FileWriter` is never closed — that's an FD leak and risks unflushed data. Try-with-resources fixes both, and I'd log through the logger and not swallow the failure."*

### MISS A — `double amount` for money ✅ (HIGH — and no comment flagged it)
The `Transaction.amount` is a `double` (`150.50`). **This is the most serious defect in the file** and none of the `// BUG` comments mention it — exactly the kind of thing that separates candidates.
**Impact:** `double` can't represent decimal fractions exactly; summing many transactions drifts, and `150.50` may serialize as `150.5` or, for large values, scientific notation.
**Fix:** `BigDecimal amount` (constructed from a `String`), with explicit scale/`RoundingMode`; format for output with `NumberFormat`/`DecimalFormat` and an explicit `Locale`.
**Say:** *"The bigger issue the comments don't flag is `double` for money — that's a precision bug for financial data. I'd move `amount` to `BigDecimal` and control formatting and locale explicitly."*

### MISS B & C — Swallowed task failures / no completion signal ✅ (MEDIUM)
`executor.submit(...)` returns `Future`s that are discarded, so any exception inside a task (an `IOException`, an NPE) is **silently swallowed** and never logged. And `exportReports` returns immediately with **no way to know when — or whether — the export finished**. Both are fixed by collecting the `Future`s and calling `get()` (see BUG 3 fix).
**Say:** *"We're ignoring the returned futures, so a failing batch disappears with no trace, and the caller gets no completion or success signal. I'd collect and await the futures."*

### MISS D & E — CSV robustness, magic number, logging ✅ (LOW–MEDIUM)
No CSV escaping (an id or amount containing a comma/newline corrupts the row), no header row, and `double.toString()` formatting is locale/precision-inconsistent. Pool size `100` is an unmeasured magic number. `printStackTrace` should be a logger.

### NOT-A-BUG 1 — "Is `generateReport` a race because many threads call it?" ❌
Explicitly call this out as **safe**: `generateReport` reads only its `transactions` argument and local variables (`sb`, `t`) — **no shared mutable state** — so concurrent invocation is fine. (Once you migrate to a static `DateTimeFormatter`, it stays safe because that type is thread-safe.) Recognizing a non-issue is senior signal.
**Say:** *"To be clear, `generateReport` itself is thread-safe — it only touches its argument and locals, no shared state — so running it across the pool is fine."*

### NOT-A-BUG 2 — indexed `for` loop ❌ (NIT)
Using `for (int i...)` instead of for-each is pure style; correctness is identical. Mention only in passing, if at all.

## How to deliver this review in the room (priority order)
1. **Lead with HIGH:** thread-pool leak (BUG 3), unclosed writer (BUG 5), and `double`-for-money (MISS A). "These three are what I'd block the PR on."
2. **Then MEDIUM:** timestamp collisions, swallowed failures / no completion signal, O(n²) concatenation, per-row formatter (with the static-`SimpleDateFormat` warning).
3. **Then LOW/NIT:** CSV escaping/header, magic number, logger vs `printStackTrace`, loop style.
4. **Explicitly name the non-bugs** (generateReport thread-safety) — it shows you don't cry wolf.

That ordering — blockers first, non-issues acknowledged — is the whole game.

---

# PART 12 — Think Beyond the Sample (categories companies actually give you)

Expect any of these. For each, the *first question* to ask yourself is in brackets:

- **Java collections/streams** [side effects? unbounded growth? wrong collection?]
- **Files/resources** [closed? buffered? try-with-resources?]
- **Concurrency** [shared mutable state? atomicity vs visibility? pool lifecycle?]
- **Spring controllers/services** [field injection? entity leak? validation? status codes?]
- **`@Transactional`** [self-invocation? checked-exception rollback? I/O inside tx? boundaries?]
- **JPA entities/repos** [N+1? lazy access after session? cascade/orphanRemoval? missing `@Version`?]
- **Kafka consumers/producers** [idempotent? commit timing? DLQ? dual-write/outbox?]
- **REST clients** [timeouts? retries on non-idempotent ops? circuit breaker?]
- **Production** [observable? bounded memory? graceful degradation? resource leaks?]

---

# PART 13 — Interview Simulation (attempt before reading the key)

> **Review this as if it were a production pull request.** Don't assume every comment is accurate. Some issues are obvious, some subtle; at least one is a Spring/transaction/concurrency issue; and at least one thing that *looks* wrong is actually fine. Write your findings with severity before scrolling to the answer key.

```java
@Service
public class WalletService {

    @Autowired
    private WalletRepository walletRepo;          // Wallet has: id, BigDecimal balance, @Version long version
    @Autowired
    private LedgerRepository ledgerRepo;
    @Autowired
    private KafkaTemplate<String, String> kafka;

    private final Object lock = new Object();     // thread-safety for balance updates

    // Transfers `amount` from one wallet to another and emits an event.
    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        synchronized (lock) {                     // ensure no concurrent transfer corrupts balances
            Wallet from = walletRepo.findById(fromId).get();
            Wallet to   = walletRepo.findById(toId).get();

            from.setBalance(from.getBalance().subtract(amount));
            to.setBalance(to.getBalance().add(amount));

            walletRepo.save(from);
            walletRepo.save(to);

            // publish AFTER the balance changes so downstream sees the transfer
            kafka.send("transfers", fromId + "->" + toId + ":" + amount);
        }
    }

    // Returns all wallets whose balance is below the threshold.
    public List<Wallet> lowBalanceWallets(BigDecimal threshold) {
        List<Wallet> all = walletRepo.findAll();  // reference-style admin report, run rarely
        List<Wallet> result = new ArrayList<>();
        for (Wallet w : all) {
            if (w.getBalance().compareTo(threshold) < 0) {   // uses compareTo, not equals — good
                result.add(w);
            }
        }
        return result;
    }

    // Retries the transfer up to 3 times on any failure.
    public void transferWithRetry(Long fromId, Long toId, BigDecimal amount) {
        for (int i = 0; i < 3; i++) {
            try {
                transfer(fromId, toId, amount);
                return;
            } catch (Exception e) {
                // transient failure — retry
            }
        }
    }
}
```

Note what you'd block on, what's a nit, and what's a false alarm. Then check yourself:

<details>
<summary><b>▶ ANSWER KEY — don't open until you've written your review</b></summary>

**CRITICAL**
- **Retry on a non-idempotent transfer (`transferWithRetry`).** If `transfer` succeeds but the response/commit path throws, or a Kafka send fails after the DB commit, retrying **re-runs the whole transfer** and double-debits. Retries must be idempotent (idempotency key / dedup) — and the current loop also **swallows every exception silently**, so a genuinely failed transfer looks successful after 3 silent failures. HIGH/CRITICAL.
- **Dual-write / `kafka.send` inside `@Transactional`.** The DB commit and the Kafka publish are **not atomic**. If the tx rolls back after the send, downstream saw a transfer that never happened; if the send fails after commit, the event is lost. The comment "publish AFTER the balance changes" describes intent but **the ordering guarantee it implies doesn't exist**. Fix: **Outbox pattern** (write an outbox row in the same tx; relay to Kafka separately). CRITICAL.

**HIGH**
- **`@Transactional` + `synchronized` is the wrong concurrency model and actively broken.** The lock is released when the method returns, but the transaction **commits *after* the method returns** (proxy boundary). So thread B can enter the synchronized block and read balances **before thread A's commit is visible** — the mutual exclusion doesn't cover the commit, so it doesn't prevent lost updates. Also, a JVM-level lock does nothing across **multiple instances/pods**. The right tool is the DB: **optimistic locking via the existing `@Version`** (retry on `OptimisticLockException`) or `SELECT ... FOR UPDATE`. Remove the `synchronized` block.
- **`.findById(...).get()` with no presence check.** A missing wallet throws `NoSuchElementException` (opaque 500) instead of a clean 404/validation error. Use `orElseThrow(() -> new WalletNotFoundException(id))`.
- **No overdraft/negative-balance or null/negative-amount validation.** `transfer` will happily drive a balance negative or process a negative/zero `amount`. Business-invariant check missing.

**MEDIUM**
- **`transferWithRetry` bypasses `@Transactional` via self-invocation... actually worse:** it calls `transfer` (a proxied public method on the *same bean*) — so **the `@Transactional` on `transfer` is bypassed** (self-invocation through `this`), meaning each "transfer" may run **without a transaction**, so the two `save` calls aren't atomic. This is the subtle Spring trap. Fix: move `transfer` to a separate bean, or drive retries from a caller/`@Retryable` on a proxied entry point.
- **No wait/verification that Kafka send succeeded** (fire-and-forget `send` returns a future that's ignored) — even with an outbox this pattern hides failures.

**LOW / NIT**
- Field injection throughout — prefer constructor injection with `final` fields.
- Event payload is a hand-built string — use a typed serializer/schema.

**NOT A BUG (say so explicitly)**
- **`compareTo` instead of `equals` on `BigDecimal`** — the inline comment is correct; this is the *right* choice (`equals` is scale-sensitive). Don't "fix" it.
- **`findAll()` in `lowBalanceWallets`** — flagged intent says it's a rarely-run admin report on a bounded set. On a *bounded* table this is acceptable; you'd note "if wallets can grow large, push the predicate into a query (`WHERE balance < :t`) and paginate," but it's **not automatically a bug**. Ask about table size rather than asserting.
- The linear loop filter itself is fine for a small set (though a query is better if the table is large).

**Ideal delivery:** "I'd block on three things — the non-idempotent retry that can double-transfer, the DB/Kafka dual write, and the `synchronized`+`@Transactional` mismatch which doesn't actually prevent lost updates and won't work across pods. Then the `.get()` and missing validation. The `compareTo` is correct as-is, and `findAll` is fine *if* this table stays small — I'd confirm that."
</details>

---

# PART 14 — Evaluation Rubric (grade yourself, or I will in a live session)

Score 0–10 on each; a senior hire is ~7+ on the top rows:

1. Bug detection 2. Technical correctness 3. **Prioritization** 4. Java 5. Spring 6. **Concurrency** 7. DB/JPA 8. Security 9. **Production thinking** 10. Communication 11. **Trade-off reasoning** 12. **Senior judgment**

After a review I'll tell you: what you correctly caught · what you missed · what you *incorrectly* flagged (false positives cost you) · which observations were too junior (formatting while a transaction bug sits there) · which showed senior thinking (naming a non-bug, asking about table size, deferring to measurement) · what you should have said instead · and how the interviewer likely read you.

**The four failure modes that sink senior candidates:** (a) missing the transaction/concurrency bug while polishing style; (b) crying wolf on non-bugs; (c) proposing a rewrite before understanding intent; (d) asserting "this is slow" with no measurement plan.

---

# PART 15 — How to Speak Like a Senior in the Room

Templates. Fill in the specifics; keep the shape.

**Raising a concern (calibrated, not alarmist):**
> *"I see a potential scalability issue here — we create a fixed 100-thread pool on every invocation. Before changing it I'd confirm whether this is CPU- or I/O-bound and what the downstream can absorb. In Spring I'd prefer a managed, bounded executor with proper lifecycle."*

**Asking for clarification (shows you review for intent):**
> *"What's the expected size of this list in production? My concern about `findAll` only matters if this table grows — on a bounded reference table it's fine."*

**Challenging an assumption / a misleading comment:**
> *"The comment says publishing after the balance change guarantees downstream consistency, but the DB commit and the Kafka send aren't atomic, so that guarantee doesn't actually hold. I'd reach for an outbox."*

**Identifying a definite bug (state impact + severity):**
> *"This is a clear one: the `FileWriter` is never closed — file-descriptor leak plus possible unflushed data. I'd block on it; the fix is try-with-resources."*

**Explaining severity / prioritizing:**
> *"There are a few things here, but I'd focus on two blockers first — the thread-pool leak and `double` for money — and treat the loop style as a nit we can leave."*

**Proposing a fix (with the trade-off named):**
> *"I'd hoist the formatter to a static field — but as a `DateTimeFormatter`, not `SimpleDateFormat`, because the latter isn't thread-safe and this runs on a pool."*

**Discussing trade-offs:**
> *"Optimistic locking is cheaper if contention is low; if this is a hot row with frequent conflicts, pessimistic locking or a queue avoids constant retries. I'd pick based on the expected contention."*

**Saying something is NOT a bug (high-value):**
> *"`compareTo` on `BigDecimal` here is actually correct — `equals` is scale-sensitive, so I'd leave this as is."*

**Admitting uncertainty professionally:**
> *"I'm not 100% sure of the default propagation behavior in this exact setup — my read is it joins the existing transaction, but I'd verify before relying on it."*

**Deferring to measurement (when it's genuinely a perf question):**
> *"This could be slow, but I wouldn't optimize blind — I'd profile under representative load and check the query plan first, so we don't add complexity for an unmeasured gain."*

**Offering an alternative:**
> *"Another approach would be a projection query that returns only the columns this endpoint needs, which sidesteps the lazy-loading question entirely."*

The meta-pattern: **observation → why it matters → severity → fix → trade-off**, in one or two sentences, then move on. Don't monologue.

---

# PART 16 — Prioritization

Classify every finding; only spend interview minutes proportional to severity.

| Level | Examples | Mention in a timed interview? |
|---|---|---|
| **CRITICAL** | Data loss/corruption, double-charge, security hole, dual-write, `double` for money | Always — lead with these |
| **HIGH** | Resource/thread leak, unclosed stream, N+1 on a hot path, missing tx boundary | Always |
| **MEDIUM** | O(n²) on large N, swallowed exceptions, missing pagination on a growing table | Yes, briefly |
| **LOW** | Magic numbers, `printStackTrace` vs logger, weak logging | Mention in passing |
| **NIT/STYLE** | Loop style, naming, formatting | Only if time; ideally batch as "a few nits" |

For each finding, be ready to state **business impact · production impact · likelihood · severity**. The cardinal sin: **10 minutes on formatting while a transaction or concurrency bug goes unmentioned.** Budget: spend your first third on CRITICAL/HIGH before touching anything below.

---

# PART 17 — Senior Java Code Review Interview Traps

- **Trusting the comments.** `// BUG 1` may be wrong, incomplete, or a distraction from the real (unflagged) bug. Verify independently.
- **Assuming every odd line is a bug.** False positives cost credibility. If it's fine, say it's fine.
- **Confusing style with correctness.** A loop-index vs for-each is not a bug.
- **Overengineering / immediate rewrite.** "I'd rewrite this whole thing" before understanding intent reads as junior. Propose the *minimal* correct fix.
- **Ignoring business requirements.** Ask what the code must guarantee before judging it.
- **Ignoring concurrency & transaction boundaries.** The two most-missed high-severity areas.
- **Blindly flipping LAZY→EAGER.** Fixes one exception, spreads N+1 everywhere.
- **Blindly adding `synchronized`.** Often the wrong tool (see Part 13 — doesn't cover the commit, doesn't cross pods).
- **Blindly increasing thread pools / adding retries.** Bigger pools can worsen contention; retries on non-idempotent ops cause duplicates.
- **Retrying non-idempotent operations.** Double-charge / double-transfer.
- **Swallowing exceptions.** Empty catch, `printStackTrace`, ignored `Future`s.
- **Logging sensitive data.** Tokens/PII/card numbers in logs.
- **"This is slow" with no evidence.** Always pair with a measurement plan.
- **Suggesting microservices / big architecture unnecessarily.**
- **Only reading syntax; never asking "what happens when X fails?"**
- **Not asking clarifying questions** about scale, contention, and intent.

---

# PART 18 — Rapid-Review Checklist (memorize this)

Run **C-R-E-A-M-S** top-to-bottom, HIGH severity first:

1. **Correctness / business invariants** — does it do the intended thing; are invariants (money balances, uniqueness) preserved?
2. **Nulls / edge cases** — empty, zero, negative, huge, overflow; `.get()`/`.orElseThrow`.
3. **Exceptions** — swallowed? checked-exception rollback? `InterruptedException` restored?
4. **Resources** — streams/connections/threads closed (try-with-resources)? executor shut down?
5. **Collections** — right type; unbounded growth; side-effecting streams.
6. **Concurrency** — shared mutable state; atomicity vs visibility; pool lifecycle & bounded queue; deadlock.
7. **Performance** — O(n²); object churn; *measure before optimizing*.
8. **Database** — N+1; missing index; full scan; large/unbounded result sets; pagination.
9. **Transactions** — boundaries short; no I/O inside tx; self-invocation; rollback rules; `@Version`/locking.
10. **Security** — injection; secrets/PII in logs; missing authz/IDOR; input validation.
11. **API** — HTTP semantics & status codes; DTO vs entity leak; validation; versioning; idempotency.
12. **Testing** — real assertions; right level (`@DataJpaTest`/Testcontainers); flakiness; concurrency tests.
13. **Observability** — structured logs + correlation id; metrics; is the failure visible?
14. **Scalability** — 10M records? high load? bounded memory?
15. **Reliability** — DB/queue/API down or slow? retries idempotent? timeouts, circuit breakers, DLQ, outbox?
16. **Maintainability** — SOLID; DI style; naming; magic numbers; leaky abstractions.

**And always:** lead with CRITICAL/HIGH · name at least one *non*-bug · pair perf claims with "I'd measure" · ask about scale and contention before asserting · state each finding as *observation → why → severity → fix → trade-off*.

---

*Next step: attempt Part 13 cold, then ask me to run a live simulation with fresh code — I'll withhold the bugs, wait for your review, and grade you on the Part 14 rubric.*
