# Java Concurrency — Traps, Bugs & Senior Interview Questions

---

## Trap 1 — Race Condition on Compound Operations

Any "check-then-act" or "read-modify-write" that isn't atomic is a race condition.

```java
public class RaceCondition {
    private Map<String, Integer> inventory = new ConcurrentHashMap<>();

    // BUG: check-then-act is NOT atomic even with ConcurrentHashMap
    public boolean sell(String item) {
        Integer stock = inventory.get(item);  // read
        if (stock != null && stock > 0) {     // check
            inventory.put(item, stock - 1);   // act
            // Two threads both read stock=1, both sell, stock goes to -1
            return true;
        }
        return false;
    }

    // FIX: use atomic compute()
    public boolean sellFixed(String item) {
        int[] sold = {0};
        inventory.compute(item, (k, stock) -> {
            if (stock != null && stock > 0) {
                sold[0] = 1;
                return stock - 1;
            }
            return stock;
        }); // entire lambda runs atomically
        return sold[0] == 1;
    }
}

```

---

## Trap 2 — Deadlock

Two threads each hold a lock the other needs. Both wait forever.

```java
public class Deadlock {
    private final Object lockA = new Object();
    private final Object lockB = new Object();

    public void method1() {
        synchronized (lockA) {
            synchronized (lockB) { /* work */ } // Thread1: A then B
        }
    }

    public void method2() {
        synchronized (lockB) {
            synchronized (lockA) { /* work */ } // Thread2: B then A — DEADLOCK
        }
    }
}

// Fix 1: Consistent lock ordering — always A before B everywhere
// Fix 2: tryLock with timeout
public boolean transfer(Account from, Account to, double amount)
        throws InterruptedException {
    while (true) {
        if (from.lock.tryLock(50, TimeUnit.MILLISECONDS)) {
            try {
                if (to.lock.tryLock(50, TimeUnit.MILLISECONDS)) {
                    try {
                        from.debit(amount); to.credit(amount); return true;
                    } finally { to.lock.unlock(); }
                }
            } finally { from.lock.unlock(); }
        }
        Thread.sleep(10); // back off and retry
    }
}
```

---

## Trap 3 — Livelock

Threads keep responding to each other but make no actual progress.

```java
// Both workers see the other is active, both yield, forever — no work gets done
public void work(Worker other) {
    while (active) {
        if (other.isActive()) {
            Thread.yield(); // keeps yielding, never works
            continue;
        }
        active = false; // this line never executes
    }
}
// Fix: randomized backoff — retry after random delay to break the sync dance
```

---

## Trap 4 — Starvation

A thread can never acquire a lock because other threads always get it first.

```java
ReentrantLock unfairLock = new ReentrantLock();       // default: unfair, random winner
ReentrantLock fairLock   = new ReentrantLock(true);   // fair: FIFO order, no starvation
// Caveat: fair locks have lower throughput — use only when starvation is observed
```

---

## Trap 5 — Visibility Bug Without volatile

```java
public class VisibilityBug {
    private boolean stopRequested = false; // no volatile

    public void start() {
        new Thread(() -> {
            while (!stopRequested) { doWork(); } // JVM may cache as 'true forever'
        }).start();
    }

    public void stop() {
        stopRequested = true; // this write may never reach the worker thread's cache
    }
}
// Worker thread may run forever even after stop() is called
// On server JVMs with aggressive JIT — almost certainly loops forever

// Fix:
private volatile boolean stopRequested = false;
// OR:
private final AtomicBoolean stopRequested = new AtomicBoolean(false);
```

---

## Trap 6 — Synchronized on Wrong Object

```java
public class WrongLock {
    private Integer count = 0;

    public void increment() {
        synchronized (count) { // WRONG — Integer is autoboxed to new object on count++
            count++;            // lock object changes every time — no mutual exclusion
        }
    }
}

public class AlsoWrong {
    private static int sharedCount = 0;

    public synchronized void increment() {
        sharedCount++; // synchronized on 'this' — different instance per thread
                       // if each thread has its own instance: no mutual exclusion!
    }
    // Fix: synchronized (AlsoWrong.class) for static shared state
}

//Summary of Rules
//    Never synchronize on a non-final field.
//    Never synchronize on "boxed" types like Integer, Long, or String (because of object pooling and immutability).
//    Always make your lock objects final.
```

---

## Trap 7 — Double-Checked Locking (Classic Interview Question)

```java
// BROKEN — missing volatile
public class Singleton {
    private static Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {                    // check 1 — no lock
            synchronized (Singleton.class) {
                if (instance == null) {            // check 2 — with lock
                    instance = new Singleton();    // 3 ops: allocate, init, assign
                    // JVM can reorder: assign reference BEFORE init completes
                    // Another thread sees non-null but uninitialized object!
                }
            }
        }
        return instance;
    }
}

// FIX 1: volatile prevents reordering
private static volatile Singleton instance;

// FIX 2 (better): Initialization-on-demand holder — no volatile, no sync needed
public class Singleton {
    private Singleton() {}
    private static class Holder {
        static final Singleton INSTANCE = new Singleton(); // JVM class loading is thread-safe
    }
    public static Singleton getInstance() { return Holder.INSTANCE; }
}
```

---

## Trap 8 — ConcurrentModificationException

```java
public class ConcurrentMod {
    private List<String> list = new ArrayList<>(Arrays.asList("a","b","c"));

    public void removeWhileIterating() {
        for (String s : list) {
            if (s.equals("b")) list.remove(s); // ConcurrentModificationException!
            // Even in single-threaded code — iterator detects structural modification
        }
    }

    // Fix 1: Iterator.remove()
    public void fix1() {
        Iterator<String> it = list.iterator();
        while (it.hasNext()) {
            if (it.next().equals("b")) it.remove(); // safe
        }
    }

    // Fix 2: removeIf (Java 8+)
    public void fix2() { list.removeIf(s -> s.equals("b")); }

    // Fix 3: CopyOnWriteArrayList for concurrent access
    private List<String> safeList = new CopyOnWriteArrayList<>(list);
    // iterates over a snapshot — never throws, but writes are expensive
}
```

---

## Trap 9 — ThreadLocal Memory Leak in Thread Pools

```java
public class ThreadLocalLeak {
    private static final ThreadLocal<UserContext> context = new ThreadLocal<>();

    public void handleRequest(User user) {
        context.set(new UserContext(user));
        try {
            processRequest();
        } finally {
            context.remove(); // CRITICAL — if forgotten in a thread pool:
                              // thread returns to pool WITH old UserContext
                              // next request on same thread sees previous user's data!
        }
    }
}
// Thread pool threads never die — ThreadLocal persists across requests
// Causes data leaks between different users' requests
```

---

## Trap 10 — sleep() as Fake Synchronization

```java
public class SleepSync {
    private int value = 0;

    public void writer() throws InterruptedException {
        value = 42;
        Thread.sleep(100); // assuming reader will run after — NOT a guarantee
    }

    public void reader() throws InterruptedException {
        Thread.sleep(50);
        System.out.println(value); // might print 0 — sleep() creates NO happens-before!
    }
}
// "Works" in testing, fails randomly in production under load
```

---

## Trap 11 — Executor Shutdown Mishandling

```java
public class ExecutorShutdown {

    public void wrong1() {
        ExecutorService pool = Executors.newFixedThreadPool(4);
        pool.submit(() -> longRunningTask());
        pool.shutdown(); // signals shutdown but doesn't wait — program may exit first
    }

    public void wrong2() {
        ExecutorService pool = Executors.newFixedThreadPool(4);
        // Never shuts down — threads keep JVM alive forever
        // Common cause of hanging integration tests
    }

    public void correct() throws InterruptedException {
        ExecutorService pool = Executors.newFixedThreadPool(4);
        pool.submit(() -> longRunningTask());
        pool.shutdown();
        if (!pool.awaitTermination(30, TimeUnit.SECONDS)) {
            pool.shutdownNow(); // force-cancel remaining tasks
        }
    }
}
```

---

## Trap 12 — Exception Swallowed in Submitted Task

```java
public class SilentFailure {
    ExecutorService pool = Executors.newFixedThreadPool(2);

    public void wrong() {
        pool.execute(() -> {
            throw new RuntimeException("Something broke");
            // SILENTLY SWALLOWED — no stack trace, no log, nothing
            // Thread dies quietly, pool replaces it
        });
    }

    public void correct() {
        Future<?> future = pool.submit(() -> { throw new RuntimeException("broke"); });
        try {
            future.get(); // exception re-thrown here as ExecutionException
        } catch (ExecutionException e) {
            System.err.println("Task failed: " + e.getCause());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    // Or: set an uncaught exception handler on the thread factory
    public void withHandler() {
        ThreadFactory factory = r -> {
            Thread t = new Thread(r);
            t.setUncaughtExceptionHandler((thread, ex) ->
                System.err.println("Uncaught in thread: " + ex));
            return t;
        };
        ExecutorService handledPool = Executors.newFixedThreadPool(2, factory);
    }
}
```

---

## Trap 13 — Interruption Handling Done Wrong

```java
public class InterruptionTrap {

    // WRONG — swallowing InterruptedException, losing the interrupted status
    public void wrong() {
        try { Thread.sleep(1000); }
        catch (InterruptedException e) { /* swallowed — thread pool shutdown breaks */ }
    }

    // CORRECT — restore interrupted status
    public void correct() {
        try { Thread.sleep(1000); }
        catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // restore the flag
            return; // stop — someone wants this thread to stop
        }
    }

    // CORRECT — propagate up
    public void alsoCorrect() throws InterruptedException {
        Thread.sleep(1000); // let the caller decide how to handle it
    }
}
```

---

## Senior Interview Questions & Model Answers

---

### Q1: "What's the difference between volatile and synchronized?"

| | volatile | synchronized |
|---|---|---|
| Visibility | ✅ Yes | ✅ Yes |
| Atomicity | ❌ No | ✅ Yes |
| Mutual exclusion | ❌ No | ✅ Yes |
| Performance | Faster | Slower (lock overhead) |
| Use case | Flags, single writes | Compound operations |

---

### Q2: "100 threads increment a counter. What's the safest approach?"

```java
// Option 1 — synchronized (simplest)
private int count = 0;
public synchronized void increment() { count++; }

// Option 2 — AtomicInteger (faster, lock-free)
private AtomicInteger count = new AtomicInteger(0);
public void increment() { count.incrementAndGet(); }

// Option 3 — LongAdder (fastest under high contention)
private LongAdder count = new LongAdder();
public void increment() { count.increment(); }
public long getCount() { return count.sum(); }
```

**Follow-up:** "Why is `LongAdder` faster than `AtomicInteger` under high contention?"
→ LongAdder stripes the counter across per-thread cells, reducing CAS contention.
→ On read (`sum()`), it aggregates all cells. Under low contention: same as AtomicInteger.

---

### Q3: "How do you detect and fix a deadlock in production?"

**Detection:**
```bash
jstack <pid>      # JDK thread dump — look for "BLOCKED" and "waiting to lock" cycles
kill -3 <pid>     # Linux signal for thread dump
```

**Programmatic detection:**
```java
ThreadMXBean bean = ManagementFactory.getThreadMXBean();
long[] deadlocked = bean.findDeadlockedThreads();
if (deadlocked != null) {
    ThreadInfo[] info = bean.getThreadInfo(deadlocked);
    // log and alert
}
```

**Prevention:** consistent lock ordering, `tryLock` with timeout, avoid nested locks.

---

### Q4: "Explain happens-before and give three examples."

> "Happens-before is the JMM's formal guarantee that if action A happens-before action B, all effects of A are visible to B. It's about visibility guarantees, not wall-clock time."

1. `thread.start()` — all writes before `start()` are visible inside the new thread
2. `lock.unlock()` — all writes inside a synchronized block are visible to the next thread acquiring the same lock
3. Volatile write — visible to all subsequent volatile reads of the same variable

---

### Q5: "What is a thread pool's rejection policy and when would you customize it?"

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    4, 8,                          // core and max threads
    60, TimeUnit.SECONDS,          // idle thread timeout
    new ArrayBlockingQueue<>(100), // bounded queue
    new ThreadPoolExecutor.CallerRunsPolicy() // custom rejection policy
);
// CallerRunsPolicy: calling thread runs the task — natural backpressure
// AbortPolicy:      throws RejectedExecutionException (default)
// DiscardPolicy:    silently drops the task
// DiscardOldestPolicy: drops oldest queued task, retries submission
```

**Senior answer:** "I'd use `CallerRunsPolicy` — it creates natural backpressure, slowing producers when the system is overloaded instead of losing work silently."

---

### Q6: "What's wrong with this code?" (Classic whiteboard)

```java
public class LazyInit {
    private static Resource resource;

    public static Resource get() {
        if (resource == null) {
            resource = new Resource(); // two threads can both enter here
        }
        return resource;
    }
}
```

**Expected answer:**
- Race condition — two threads both see `null`, both create `Resource`, one overwrites the other
- Missing `volatile` — publication unsafe, other threads may see partially initialized object
- Missing `synchronized` — no mutual exclusion
- Fix: initialization-on-demand holder, or double-checked locking with `volatile`

---

### Q7: "Why must you call Thread.currentThread().interrupt() after catching InterruptedException?"

`InterruptedException` clears the thread's interrupted flag when thrown. If you catch it and don't restore the flag, the thread pool's management code checks `Thread.isInterrupted()` to decide if the thread should stop — finds `false` — and keeps the thread alive instead of shutting down cleanly. Swallowing the interrupt breaks cooperative thread cancellation.

---

## The Concurrency Mental Model

```
Ask these 3 questions about every shared variable:

1. VISIBILITY  → Can all threads see the latest value?
                 Fix: volatile, synchronized, Atomic

2. ATOMICITY   → Does the operation complete as one unit?
                 Fix: synchronized, Atomic, Lock

3. ORDERING    → Do operations happen in the expected sequence?
                 Fix: volatile, synchronized, happens-before

If YES to all three — your shared state is safe.
```
