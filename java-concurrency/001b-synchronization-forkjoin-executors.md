# Java Concurrency — Part 2: Synchronization, Fork/Join & Executors

---

## Part 1 — Synchronization Tools

These tools solve the three JMM problems: visibility, atomicity, and ordering.

---

### synchronized — The Simplest Lock

`synchronized` is Java's built-in locking mechanism. It gives you two things at once:
1. **Mutual exclusion** — only one thread can be inside the block at a time
2. **Visibility** — on entry: reads fresh values from RAM. On exit: flushes writes to RAM.

Think of it as a bathroom with one key. The first thread takes the key (acquires the lock), does its work, and returns the key (releases the lock). Everyone else waits outside.

```java
public class BankAccount {
    private double balance = 1000.0;

    // WITHOUT synchronized — the race condition:
    // Thread A reads balance = 1000
    // Thread B reads balance = 1000
    // Thread A writes balance = 800 (withdrew 200)
    // Thread B writes balance = 800 (withdrew 200)
    // Net result: 400 withdrawn but balance only dropped by 200. Money created from nothing.

    public synchronized void withdraw(double amount) {
        // Only ONE thread executes this at a time
        if (balance < amount) throw new IllegalStateException("Insufficient funds");
        balance -= amount;
    }

    public synchronized void deposit(double amount) {
        balance += amount;
    }

    public synchronized double getBalance() {
        return balance; // synchronized getter ensures you always see the latest value
    }
}
```

**Which object is the lock? This matters:**

```java
public class LockExamples {

    // Lock = this instance
    public synchronized void instanceMethod() { }

    // Lock = LockExamples.class — completely different lock from above!
    public static synchronized void staticMethod() { }

    // Lock = this instance (same as synchronized instance method)
    public void blockOnThis() {
        synchronized (this) { }
    }

    // Lock = a dedicated private object — best practice for non-public locking
    private final Object lock = new Object();
    public void blockOnPrivateLock() {
        synchronized (lock) { }
    }
}
```

**Reentrancy — a thread can re-enter its own lock:**

```java
public class Reentrant {
    public synchronized void outer() {
        System.out.println("outer");
        inner(); // same thread already holds the lock — Java allows re-entry, no deadlock
    }

    public synchronized void inner() {
        System.out.println("inner"); // executes fine
    }
}
```

---

### volatile — Visibility Without Locking

`volatile` is lighter than `synchronized`. It only fixes **visibility** — NOT atomicity, NOT mutual exclusion.

When a field is `volatile`:
- Every write goes directly to main memory — no CPU cache buffering
- Every read comes directly from main memory — never from a stale CPU cache

```java
public class ServiceRunner {
    private volatile boolean running = true; // the volatile keyword is the fix

    public void start() {
        Thread worker = new Thread(() -> {
            while (running) {       // reads from main memory every iteration — sees updates
                handleRequest();
            }
            System.out.println("Service stopped cleanly");
        });
        worker.start();
    }

    public void stop() {
        running = false; // goes directly to main memory — worker thread sees it immediately
    }
}
```

**Where volatile is NOT enough — atomicity:**

```java
private volatile int count = 0;

public void increment() {
    count++; // BROKEN even with volatile
             // count++ is really: READ count → ADD 1 → WRITE count
             // Two threads both READ 5, both ADD 1, both WRITE 6
             // One increment is lost — race condition
}
// Fix: use AtomicInteger or synchronized
```

**When volatile is the right choice:**
- A boolean flag written by one thread, read by many (stop signal, init flag)
- A reference that is fully replaced (not mutated internally)
- You have a single writer and many readers

---

### Atomic Classes — Lock-Free Thread Safety

`java.util.concurrent.atomic` provides thread-safe operations without locks, using CPU-level **Compare-And-Swap (CAS)**.

CAS logic: "Change the value to NEW only if it is currently EXPECTED. If someone else changed it first — retry."

```java
import java.util.concurrent.atomic.*;

// AtomicInteger — thread-safe counter
AtomicInteger counter = new AtomicInteger(0);

counter.incrementAndGet();      // atomic count++, returns new value
counter.decrementAndGet();      // atomic count--
counter.addAndGet(5);           // atomic count += 5
counter.getAndSet(0);           // atomically read the value then set it to 0
counter.compareAndSet(10, 20);  // set to 20 only if current value is 10 — returns true/false
```

**Real example — thread-safe request counter:**

```java
public class RequestTracker {
    private final AtomicInteger totalRequests  = new AtomicInteger(0);
    private final AtomicInteger activeRequests = new AtomicInteger(0);

    public void onRequestStart() {
        totalRequests.incrementAndGet();
        activeRequests.incrementAndGet();
    }

    public void onRequestEnd() {
        activeRequests.decrementAndGet();
    }

    public void printStats() {
        System.out.println("Total: "  + totalRequests.get());
        System.out.println("Active: " + activeRequests.get());
    }
}
```

**AtomicBoolean — perfect for one-time initialization:**

```java
public class ExpensiveService {
    private final AtomicBoolean initialized = new AtomicBoolean(false);

    public void ensureInitialized() {
        // compareAndSet: "set to true ONLY IF currently false"
        // First thread: false → true → enters block
        // All other threads: already true → skip
        if (initialized.compareAndSet(false, true)) {
            performExpensiveStartup(); // guaranteed to run exactly once
        }
    }
}
```

**LongAdder — faster than AtomicLong under high contention:**

```java
// AtomicLong: all threads fight over one value → CAS retries under load
// LongAdder: each thread updates its own cell → sum() aggregates them on demand

LongAdder hitCounter = new LongAdder();

// In each of your request handler threads:
hitCounter.increment(); // each thread mostly touches its own cell

// When you need the total:
long total = hitCounter.sum();
// Under heavy load: LongAdder is significantly faster than AtomicLong
```

---

### ReentrantLock — synchronized With Extra Powers

`ReentrantLock` gives you everything `synchronized` does, plus:
- **tryLock()** — try to get the lock without blocking
- **tryLock(timeout)** — try for a limited time then give up
- **lockInterruptibly()** — waiting thread can be interrupted
- **Fair mode** — threads get the lock in arrival order (prevents starvation)

```java
import java.util.concurrent.locks.*;

public class ResourceManager {
    private final ReentrantLock lock = new ReentrantLock();
    private String resource = "free";

    // Basic usage — always unlock in finally
    public void useResource() {
        lock.lock();
        try {
            resource = "busy";
            doWork();
        } finally {
            resource = "free";
            lock.unlock(); // ALWAYS in finally — never skip even if exception thrown
        }
    }

    // Non-blocking attempt — great when you have alternative work
    public boolean tryUseResource() {
        if (lock.tryLock()) {           // returns immediately: true if got lock, false if not
            try {
                doWork();
                return true;
            } finally {
                lock.unlock();
            }
        }
        System.out.println("Resource busy — doing something else");
        return false;
    }

    // Try with timeout — prevents waiting forever
    public boolean tryUseWithTimeout() throws InterruptedException {
        if (lock.tryLock(2, TimeUnit.SECONDS)) { // wait at most 2 seconds
            try {
                doWork();
                return true;
            } finally {
                lock.unlock();
            }
        }
        System.out.println("Gave up waiting");
        return false;
    }
}
```

**ReadWriteLock — read in parallel, write exclusively:**

This is a major performance win when reads are very frequent and writes are rare (caches, config stores, lookup tables).

```java
public class ProductCatalog {
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock  = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    private final Map<String, Product> catalog = new HashMap<>();

    // Multiple threads can read at the SAME TIME — no blocking between readers
    public Product findProduct(String id) {
        readLock.lock();
        try {
            return catalog.get(id);
        } finally {
            readLock.unlock();
        }
    }

    // Writing is exclusive — blocks ALL readers and all other writers
    public void addProduct(Product p) {
        writeLock.lock();
        try {
            catalog.put(p.getId(), p);
        } finally {
            writeLock.unlock();
        }
    }
}

// With regular synchronized: even reads block each other
// With ReadWriteLock: 100 threads can read catalog simultaneously
//                     only updateProduct() pauses everyone
```

---

## Part 2 — Thread Coordination Tools

---

### CountDownLatch — "Wait Until N Things Have Happened"

Think of it as a rocket countdown. You set the count. Various threads call `countDown()`. Anyone calling `await()` is blocked until the count reaches zero.

**Key characteristic:** Single-use — once it reaches zero, it stays there. Cannot be reset.

```java
public class ServiceStartup {
    public static void main(String[] args) throws InterruptedException {

        int serviceCount = 3;
        CountDownLatch allReady = new CountDownLatch(serviceCount);

        // Start each service in its own thread
        String[] services = {"Database", "Cache", "MessageQueue"};
        for (String service : services) {
            new Thread(() -> {
                try {
                    System.out.println(service + ": starting...");
                    Thread.sleep((long)(Math.random() * 2000)); // simulate startup time
                    System.out.println(service + ": READY");
                    allReady.countDown(); // this service is ready
                } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            }).start();
        }

        System.out.println("Main: waiting for all services...");
        allReady.await(); // BLOCKS until all 3 services have called countDown()
        System.out.println("Main: all services ready — starting application!");
    }
}

// Output:
// Main: waiting for all services...
// Database: starting...
// Cache: starting...
// MessageQueue: starting...
// Cache: READY
// Database: READY
// MessageQueue: READY
// Main: all services ready — starting application!
```

**await() with timeout:**
```java
boolean allStarted = allReady.await(10, TimeUnit.SECONDS);
if (!allStarted) {
    System.out.println("Services took too long — aborting startup");
}
```

---

### CyclicBarrier — "Everyone Meet at This Checkpoint, Then Continue Together"

A CyclicBarrier makes N threads wait until ALL of them reach the same point. Then they all continue. Unlike CountDownLatch — it **automatically resets** after each use, so you can reuse it for multiple phases.

```java
public class ParallelDataPipeline {
    public static void main(String[] args) throws Exception {

        int workers = 3;
        // Optional action runs once when all threads arrive at the barrier
        CyclicBarrier barrier = new CyclicBarrier(workers, () ->
            System.out.println("\n>>> All workers at checkpoint — next phase starting <<<\n")
        );

        for (int i = 1; i <= workers; i++) {
            final int id = i;
            new Thread(() -> {
                try {
                    // --- Phase 1: Load data ---
                    System.out.println("Worker " + id + ": loading data...");
                    Thread.sleep((long)(Math.random() * 1000));
                    System.out.println("Worker " + id + ": load done, waiting");

                    barrier.await(); // WAIT — no one moves to phase 2 until ALL arrive here

                    // --- Phase 2: Process data ---
                    System.out.println("Worker " + id + ": processing data...");
                    Thread.sleep((long)(Math.random() * 1000));
                    System.out.println("Worker " + id + ": process done, waiting");

                    barrier.await(); // barrier RESETS automatically — reused for phase 2

                    // --- Phase 3: Save results ---
                    System.out.println("Worker " + id + ": saving results...");

                } catch (Exception e) { Thread.currentThread().interrupt(); }
            }).start();
        }
    }
}
```

**CountDownLatch vs CyclicBarrier:**

| | CountDownLatch | CyclicBarrier |
|---|---|---|
| Reusable? | No — single use | Yes — auto-resets after each cycle |
| Who calls count? | Any thread calls `countDown()` | The waiting threads themselves call `await()` |
| Use case | Wait for N events to complete | Synchronize N threads at repeated checkpoints |
| Action on completion | Nothing | Optional `Runnable` runs when all arrive |

---

### Semaphore — "Only N Threads Allowed In at a Time"

A Semaphore is a bouncer with a headcount. It holds N permits. Threads `acquire()` a permit to enter and `release()` it when done. If no permits are available — the thread waits.

```java
public class ApiRateLimiter {
    // Only 5 concurrent API calls allowed at a time
    private final Semaphore permits = new Semaphore(5);

    public String callExternalApi(String request) throws InterruptedException {
        permits.acquire(); // block if 5 calls already in flight
        try {
            return sendHttpRequest(request); // only 5 of these run concurrently
        } finally {
            permits.release(); // always release — even if exception thrown
        }
    }
}

// Demonstration — 10 threads want to call the API, only 5 allowed at once
ApiRateLimiter limiter = new ApiRateLimiter();

for (int i = 1; i <= 10; i++) {
    final int id = i;
    new Thread(() -> {
        try {
            System.out.println("Thread " + id + ": waiting for permit...");
            String result = limiter.callExternalApi("request-" + id);
            System.out.println("Thread " + id + ": got response");
        } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }).start();
}

// First 5 threads get permits immediately
// Threads 6-10 wait — each gets a permit as soon as one of the first 5 releases
```

**tryAcquire() — don't wait, just check:**
```java
if (permits.tryAcquire()) {       // returns immediately: true if got permit
    try { doWork(); }
    finally { permits.release(); }
} else {
    System.out.println("System at capacity — try later");
}
```

---

## Part 3 — Fork/Join Framework

Fork/Join is designed for a specific category of problem: **tasks that can be split into smaller sub-tasks, solved independently in parallel, then combined**.

Think of it like splitting a large pizza order:
1. **Fork** — split the work into smaller pieces
2. **Solve** — each piece is processed by a different thread
3. **Join** — combine all the results

```
[Sum of 1 million numbers]
         │
    ─────┴─────
    │          │
[Left half] [Right half]      ← these run on different threads
    │          │
  [sum A]    [sum B]
    │          │
    └────┬─────┘
       [A + B]                ← combined
```

---

### RecursiveTask — Fork/Join That Returns a Value

```java
import java.util.concurrent.*;

public class ParallelArraySum extends RecursiveTask<Long> {

    private static final int THRESHOLD = 10_000; // below this size, just compute directly
    private final int[] array;
    private final int start, end;

    public ParallelArraySum(int[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end   = end;
    }

    @Override
    protected Long compute() {
        int size = end - start;

        // BASE CASE: small enough — compute directly without splitting further
        if (size <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) sum += array[i];
            return sum;
        }

        // RECURSIVE CASE: too big — split into two halves and solve in parallel
        int mid = start + size / 2;

        ParallelArraySum leftTask  = new ParallelArraySum(array, start, mid);
        ParallelArraySum rightTask = new ParallelArraySum(array, mid,   end);

        leftTask.fork();                  // FORK: submit left to thread pool (runs asynchronously)
        long rightResult = rightTask.compute(); // compute right half on THIS thread (no overhead)
        long leftResult  = leftTask.join();     // JOIN: block until left half is done

        return leftResult + rightResult;  // COMBINE the two halves
    }
}

// How to use it:
int[] data = new int[1_000_000];
Arrays.fill(data, 1); // all 1s — expected sum = 1,000,000

ForkJoinPool pool = ForkJoinPool.commonPool(); // uses all available CPU cores
long sum = pool.invoke(new ParallelArraySum(data, 0, data.length));

System.out.println("Sum: " + sum);           // 1000000
System.out.println("Cores used: " + pool.getParallelism()); // e.g. 7 on 8-core CPU
```

**The fork/join pattern to memorize:**

```java
leftTask.fork();                    // 1. FORK left — send to pool (async)
long rightResult = right.compute(); // 2. COMPUTE right — do it yourself (no overhead)
long leftResult  = leftTask.join(); // 3. JOIN left — wait and get result
return leftResult + rightResult;    // 4. COMBINE
```

Why compute right yourself instead of forking both? Forking has overhead. Doing one side yourself reuses the current thread efficiently.

---

### RecursiveAction — Fork/Join With No Return Value

Use `RecursiveAction` when you modify data in-place and do not need to return a value.

```java
// Fills a large array in parallel
public class ParallelFill extends RecursiveAction {

    private static final int THRESHOLD = 5_000;
    private final int[] array;
    private final int start, end;
    private final int fillValue;

    public ParallelFill(int[] array, int start, int end, int fillValue) {
        this.array     = array;
        this.start     = start;
        this.end       = end;
        this.fillValue = fillValue;
    }

    @Override
    protected void compute() {
        if (end - start <= THRESHOLD) {
            // BASE CASE: small enough — fill directly
            Arrays.fill(array, start, end, fillValue);
            return;
        }

        // RECURSIVE CASE: split into two halves
        int mid = start + (end - start) / 2;

        ParallelFill left  = new ParallelFill(array, start, mid, fillValue);
        ParallelFill right = new ParallelFill(array, mid,   end, fillValue);

        invokeAll(left, right); // shorthand: submits both and waits for both to finish
    }
}

// Usage:
int[] data = new int[1_000_000];
ForkJoinPool.commonPool().invoke(new ParallelFill(data, 0, data.length, 42));
System.out.println(data[500_000]); // 42
```

**RecursiveTask vs RecursiveAction:**

| | RecursiveTask\<V\> | RecursiveAction |
|---|---|---|
| Returns a value? | Yes — generic type V | No — void |
| Use case | Parallel sum, search, merge sort | Parallel fill, sort in-place, update |
| combine step | `return left + right` | No combine needed |

---

### ForkJoinPool — The Engine Behind Fork/Join

```java
// commonPool — shared across the JVM, uses (CPU cores - 1) threads by default
ForkJoinPool commonPool = ForkJoinPool.commonPool();
System.out.println("Parallelism: " + commonPool.getParallelism()); // e.g. 7 on 8-core

// Custom pool — when you need isolation or specific thread count
ForkJoinPool customPool = new ForkJoinPool(4); // exactly 4 threads
long result = customPool.invoke(myTask);
customPool.shutdown();
```

**Work-stealing — why ForkJoinPool is efficient:**

Each thread has its own queue of tasks. When a thread runs out of tasks, it **steals** from the back of another thread's queue. This keeps all threads busy automatically.

```
Thread 1 queue: [Task-A, Task-B, Task-C] ← Thread 1 takes from front
Thread 2 queue: [Task-D, Task-E]         ← Thread 2 finishes, steals Task-C from Thread 1
Thread 3 queue: []                        ← Thread 3 steals Task-E from Thread 2
```

No thread sits idle waiting — work flows automatically to where there is capacity.

---

## Part 4 — ExecutorService & Thread Pools

Creating a `new Thread()` for every task is expensive (threads are heavy OS resources) and uncontrolled (you might accidentally create thousands). `ExecutorService` gives you a managed pool of reusable threads.

---

### The Core Idea

```java
// Think of ExecutorService as a team manager with N workers
// You give the manager tasks
// Workers pick them up from a queue and execute them
// Workers are reused — not killed and recreated per task

ExecutorService pool = Executors.newFixedThreadPool(4); // 4 worker threads

// submit() — hand in a task and get a receipt (Future)
Future<String> future = pool.submit(() -> {
    Thread.sleep(2000);
    return "report generated";
});

System.out.println("Task submitted, doing other things...");

String result = future.get(); // block until the task finishes, get the result
System.out.println("Got: " + result);

// execute() — fire and forget (no result needed)
pool.execute(() -> System.out.println("Background cleanup"));

// ALWAYS shut down — otherwise the JVM will NOT exit
pool.shutdown();                          // no new tasks — waits for current ones to finish
pool.awaitTermination(30, TimeUnit.SECONDS); // give running tasks up to 30s
```

---

### Thread Pool Types

```java
// Fixed pool — best for CPU-bound work, predictable resource use
ExecutorService fixed = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors() // one thread per CPU core
);

// Cached pool — creates threads on demand, reuses idle ones (idle timeout 60s)
// Good for: many short-lived async tasks
// Risk: no upper bound on threads — can overwhelm system under load
ExecutorService cached = Executors.newCachedThreadPool();

// Single-thread — tasks run sequentially one at a time in submission order
ExecutorService single = Executors.newSingleThreadExecutor();

// Scheduled — run tasks after a delay, or repeatedly on a schedule
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

scheduler.schedule(() -> System.out.println("Runs once after 5s"),
    5, TimeUnit.SECONDS);

scheduler.scheduleAtFixedRate(() -> System.out.println("Runs every 10s"),
    0,  // initial delay
    10, // period
    TimeUnit.SECONDS);
```

---

### CompletableFuture — Async Pipelines Without Blocking

`CompletableFuture` is the modern way to chain asynchronous operations. Instead of blocking and waiting at each step, you define a pipeline and each stage kicks off automatically when the previous one completes.

```java
// Sequential blocking approach — main thread blocks 3 times
User user    = fetchUser(userId);     // blocks 1s
Orders orders = fetchOrders(user);    // blocks 1s
String html  = renderPage(orders);    // blocks 1s
// Total: 3 seconds, main thread idle the whole time

// CompletableFuture pipeline — main thread is free
CompletableFuture
    .supplyAsync(() -> fetchUser(userId))       // runs on background thread
    .thenApply(user -> fetchOrders(user))       // runs when fetch finishes
    .thenApply(orders -> renderPage(orders))    // runs when fetchOrders finishes
    .thenAccept(html -> sendResponse(html))     // runs when render finishes
    .exceptionally(ex -> {                      // runs if ANY step throws
        log.error("Failed", ex);
        sendErrorResponse();
        return null;
    });

// Main thread continues here immediately — pipeline runs in background
```

**Running two things in parallel then combining:**

```java
CompletableFuture<User>    userFuture    = CompletableFuture.supplyAsync(() -> fetchUser(id));
CompletableFuture<Account> accountFuture = CompletableFuture.supplyAsync(() -> fetchAccount(id));
// Both run in parallel — not waiting for each other

CompletableFuture.allOf(userFuture, accountFuture)   // wait for BOTH
    .thenRun(() -> {
        User    user    = userFuture.join();    // already done — no blocking
        Account account = accountFuture.join();
        buildDashboard(user, account);
    });
```

**Useful CompletableFuture methods:**

```java
.supplyAsync(supplier)              // start async task that returns a value
.runAsync(runnable)                 // start async task with no return value
.thenApply(fn)                      // transform the result (like map)
.thenAccept(consumer)               // consume the result (terminal)
.thenCompose(fn)                    // chain another CompletableFuture (like flatMap)
.thenCombine(other, biFunction)     // combine result of two futures
.allOf(future1, future2, ...)       // wait for all to complete
.anyOf(future1, future2, ...)       // wait for any one to complete first
.exceptionally(fn)                  // handle exception, provide fallback
.join()                             // block until done (unchecked exception)
.get()                              // block until done (checked exception)
```

---

## Part 5 — Concurrent Collections

Standard collections (`HashMap`, `ArrayList`) are not thread-safe. Using them from multiple threads corrupts their internal structure.

---

### ConcurrentHashMap

The go-to thread-safe map. Reads never block. Writes lock only a small segment (not the whole map).

```java
Map<String, Integer> wordCount = new ConcurrentHashMap<>();

// WRONG approach even with ConcurrentHashMap — get-then-put is not atomic:
Integer count = wordCount.get(word);
wordCount.put(word, count == null ? 1 : count + 1); // race condition between get and put

// CORRECT — use atomic compound operations:
wordCount.merge(word, 1, Integer::sum);              // word frequency counter — atomic
wordCount.putIfAbsent("key", 0);                     // add only if not already there — atomic
wordCount.computeIfAbsent("key", k -> loadFromDB(k));// compute and add if absent — atomic
wordCount.compute("key", (k, v) -> v == null ? 1 : v + 1); // full atomic read-modify-write
```

---

### BlockingQueue — Producer-Consumer Backbone

A queue where `put()` blocks if the queue is full and `take()` blocks if it is empty. This gives you automatic backpressure with zero extra synchronization code.

```java
public class WorkPipeline {
    // Bounded queue — max 50 items. Producer slows down naturally when full.
    private final BlockingQueue<Task> queue = new LinkedBlockingQueue<>(50);

    // Producer thread
    public void produce(Task task) throws InterruptedException {
        queue.put(task);   // blocks if queue is full — producer naturally slows down
    }

    // Consumer thread
    public Task consume() throws InterruptedException {
        return queue.take(); // blocks if queue is empty — consumer waits for work
    }
}

// Real-world usage with multiple producers and consumers
BlockingQueue<String> queue = new LinkedBlockingQueue<>(100);

// 3 producer threads
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        try {
            while (true) { queue.put(generateWork()); }
        } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }).start();
}

// 2 consumer threads
for (int i = 0; i < 2; i++) {
    new Thread(() -> {
        try {
            while (true) { processWork(queue.take()); }
        } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }).start();
}
```

---

### CopyOnWriteArrayList — Thread-Safe List for Rare Writes

Every write operation makes a **full copy** of the internal array. Reads never block because they always operate on a stable snapshot. Very fast for reading, expensive for writing.

```java
// Perfect for: event listeners, subscribers, config snapshots — things rarely changed
List<EventListener> listeners = new CopyOnWriteArrayList<>();

// Adding a listener (rare) — creates a new internal array copy
listeners.add(new MyListener());

// Iterating (frequent) — safe even if another thread adds a listener concurrently
for (EventListener listener : listeners) {
    listener.onEvent(event); // iterates over a snapshot — never ConcurrentModificationException
}
```

---

## Full Concurrency Toolkit — One-Page Reference

```
PROBLEM                             TOOL
────────────────────────────────────────────────────────────────
Protect shared state                synchronized / ReentrantLock
Flag visible across threads         volatile
Thread-safe counter                 AtomicInteger / LongAdder
One-time state transition           AtomicReference.compareAndSet
Multiple readers, rare writes       ReadWriteLock
Wait for N services/events to start CountDownLatch
Sync N threads at phase boundary    CyclicBarrier
Limit concurrent access             Semaphore
Divide-and-conquer computation      ForkJoinPool + RecursiveTask
Parallel void operations            ForkJoinPool + RecursiveAction
Manage a pool of reusable threads   ExecutorService
Chain async operations              CompletableFuture
Thread-safe high-perf map           ConcurrentHashMap
Producer-consumer with backpressure BlockingQueue
Thread-safe read-heavy list         CopyOnWriteArrayList
```
