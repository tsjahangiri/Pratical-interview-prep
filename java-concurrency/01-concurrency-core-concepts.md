# Java Concurrency — Core Concepts (Senior Developer Guide)

---

## Concept 1 — Threads & the Java Memory Model (JMM)

The JMM is the foundation everything else builds on. Without it, concurrency bugs look like magic.

**The problem:** CPUs and the JVM cache values in registers, reorder instructions, and write to local CPU caches instead of main memory. In a single thread this is invisible. Across threads, it causes chaos.

**Three core JMM problems:**

| Problem | Description | Example |
|---|---|---|
| **Visibility** | One thread's write is invisible to another | `running = false` never seen |
| **Atomicity** | Operation appears as one step but isn't | `count++` is actually 3 ops |
| **Ordering** | Instructions execute out of written order | Init before flag set gets reordered |

```java
public class VisibilityDemo {
    private boolean running = true; // no volatile

    public void stop() { running = false; }      // Thread A writes

    public void run() {
        while (running) { doWork(); }             // Thread B may NEVER see the update
        System.out.println("Stopped");            // may never print
    }
}
```

**Happens-Before relationships (formal JMM guarantees):**
- `thread.start()` happens-before any code inside that thread
- Releasing a lock happens-before acquiring the same lock
- Volatile write happens-before volatile read of the same variable
- `Thread.join()` — all thread actions happen-before `join()` returns

---

## Concept 2 — synchronized

Provides two guarantees simultaneously: **mutual exclusion** and **visibility**.

```java
public class SafeCounter {
    private int count = 0;

    public synchronized void increment() { count++; } // locks on 'this'

    public void decrement() {
        synchronized (this) { count--; }              // equivalent block form
    }

    public static synchronized void staticMethod() {  // locks on SafeCounter.class
        // different lock from instance methods!
    }
}
```

**What synchronized does internally:**
```
Thread acquires lock
  → flushes working memory
  → reads fresh values from main memory
  → executes critical section
  → writes changes to main memory
Thread releases lock
```

**Reentrant nature — same thread can re-enter:**
```java
public synchronized void methodA() {
    methodB(); // same thread already holds lock — no deadlock, Java locks are reentrant
}
public synchronized void methodB() { ... }
```

---

## Concept 3 — volatile

Solves **visibility only** — does NOT provide atomicity.

```java
public class VolatileDemo {
    private volatile boolean flag = false;
    private volatile int value = 0;

    public void writer() {           // Thread A
        value = 42;                  // write 1
        flag = true;                 // volatile write — flushes everything above
    }

    public void reader() {           // Thread B
        if (flag) {                  // volatile read
            System.out.println(value); // guaranteed to see 42
        }
    }
}
```

**Where volatile fails — non-atomic compound operations:**
```java
private volatile int count = 0;
public void increment() {
    count++; // NOT atomic even with volatile!
    // = read count, add 1, write count — three ops, race condition possible
}
```

**When volatile is the right tool:**
- Single writer, multiple readers
- Boolean flags (stop signals, init flags)
- References replaced atomically (not mutated)

---

## Concept 4 — Atomic Classes

Lock-free thread-safe operations using CPU-level **Compare-And-Swap (CAS)**.

```java
public class AtomicDemo {
    private AtomicInteger count = new AtomicInteger(0);
    private AtomicReference<String> state = new AtomicReference<>("IDLE");

    public void increment() { count.incrementAndGet(); }

    public void compareAndUpdate() {
        boolean updated = state.compareAndSet("IDLE", "RUNNING");
        // Only updates if current value matches expected — returns false if not
    }

    public int getAndReset() {
        return count.getAndSet(0); // atomically reads and resets
    }
}
```

**How CAS works:**
```
1. Read current value (e.g. 5)
2. Compute new value (6)
3. CAS: "set to 6 ONLY IF current is still 5"
4. If another thread changed it — CAS fails, retry
```

**ABA Problem:**
```java
// Thread A reads "A", Thread B changes A→B→A, Thread A CAS succeeds wrongly
// Fix: AtomicStampedReference — tracks version number alongside value
AtomicStampedReference<String> ref = new AtomicStampedReference<>("A", 0);
int[] stamp = new int[1];
String val = ref.get(stamp);
ref.compareAndSet(val, "B", stamp[0], stamp[0] + 1); // CAS with version
```

---

## Concept 5 — Locks (ReentrantLock & ReadWriteLock)

`ReentrantLock` gives everything `synchronized` does plus fine-grained control.

```java
public class LockDemo {
    private final ReentrantLock lock = new ReentrantLock();
    private int count = 0;

    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock(); // ALWAYS in finally — never skip this
        }
    }

    public boolean tryIncrement() throws InterruptedException {
        if (lock.tryLock(100, TimeUnit.MILLISECONDS)) { // non-blocking with timeout
            try { count++; return true; }
            finally { lock.unlock(); }
        }
        return false;
    }
}
```

**ReadWriteLock — multiple readers, exclusive writer:**
```java
public class CachedData {
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock  = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();
    private Map<String, String> cache = new HashMap<>();

    public String get(String key) {
        readLock.lock(); // multiple threads hold this simultaneously
        try { return cache.get(key); }
        finally { readLock.unlock(); }
    }

    public void put(String key, String value) {
        writeLock.lock(); // exclusive — blocks all readers and writers
        try { cache.put(key, value); }
        finally { writeLock.unlock(); }
    }
}
```

---

## Concept 6 — Thread Coordination

**CountDownLatch — wait for N events (one-time use):**
```java
CountDownLatch ready = new CountDownLatch(3);
// workers call ready.countDown() when ready
ready.await(); // main thread blocks until count reaches 0
```

**CyclicBarrier — all threads meet at a checkpoint (reusable):**
```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("Phase done"));
// all 3 threads must reach barrier.await() before any continues
// automatically resets for next phase
```

**Semaphore — controlling access to N resources:**
```java
public class ConnectionPool {
    private final Semaphore permits = new Semaphore(10); // max 10 connections

    public Connection acquire() throws InterruptedException {
        permits.acquire(); // blocks if 10 already in use
        return createConnection();
    }

    public void release(Connection conn) {
        closeConnection(conn);
        permits.release(); // unblocks a waiting thread
    }
}
```

---

## Concept 7 — ExecutorService & Thread Pools

Never create raw threads in production. Use the executor framework.

```java
public class ExecutorDemo {

    public void fixedPool() throws Exception {
        ExecutorService pool = Executors.newFixedThreadPool(4);

        Future<String> future = pool.submit(() -> {
            Thread.sleep(1000);
            return "result";
        });

        String result = future.get(5, TimeUnit.SECONDS); // blocks with timeout

        pool.shutdown();
        pool.awaitTermination(10, TimeUnit.SECONDS);
    }

    public void completableFuture() {
        CompletableFuture<String> cf = CompletableFuture
            .supplyAsync(() -> fetchFromDB())
            .thenApply(data -> transform(data))
            .thenCombine(
                CompletableFuture.supplyAsync(() -> fetchFromCache()),
                (db, cache) -> merge(db, cache)
            )
            .exceptionally(ex -> "fallback");

        String result = cf.join();
    }
}
```

**Thread pool types:**

| Pool Type | Use Case | Risk |
|---|---|---|
| `newFixedThreadPool(n)` | CPU-bound tasks | Unbounded queue — OOM under load |
| `newCachedThreadPool()` | Short-lived async tasks | Unlimited threads — can overwhelm |
| `newSingleThreadExecutor()` | Sequential background work | Single point of failure |
| `newScheduledThreadPool(n)` | Periodic/delayed tasks | — |
| `ForkJoinPool` | Divide-and-conquer recursion | Complex to tune |

---

## Concept 8 — Concurrent Collections

```java
// NEVER use HashMap from multiple threads — can corrupt internal structure

// ConcurrentHashMap — best for most cases
Map<String, Integer> safeMap = new ConcurrentHashMap<>();
safeMap.putIfAbsent("key", 1);                     // atomic check-then-act
safeMap.computeIfAbsent("key", k -> compute(k));   // atomic compute if missing
safeMap.merge("key", 1, Integer::sum);             // atomic read-modify-write

// synchronizedMap — locks ENTIRE map for every operation (worse throughput)
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
```

**BlockingQueue — the producer-consumer backbone:**
```java
public class ProducerConsumer {
    private final BlockingQueue<Task> queue = new LinkedBlockingQueue<>(100);

    public void produce(Task task) throws InterruptedException {
        queue.put(task);   // blocks if full
    }

    public void consume() throws InterruptedException {
        Task task = queue.take(); // blocks if empty
        process(task);
    }
}
```

---

## The Concurrency Mental Model

Ask these 3 questions about every shared variable:

```
1. VISIBILITY  → Can all threads see the latest value?
                 Fix: volatile, synchronized, Atomic

2. ATOMICITY   → Does the operation complete as one unit?
                 Fix: synchronized, Atomic, Lock

3. ORDERING    → Do operations happen in the expected sequence?
                 Fix: volatile, synchronized, happens-before

If YES to all three — your shared state is safe.
```
