# Java Concurrency — Core Concepts (Senior Developer Guide)

---

## What is Concurrency and Why Does It Exist?

Imagine a restaurant kitchen. One chef doing everything — taking orders, cooking, washing dishes, serving — is slow. Multiple chefs working at the same time on different tasks is concurrency. Your program is the kitchen. Threads are the chefs.

Concurrency lets your program do multiple things at the same time (or appear to). Without it:
- Your UI freezes while a file downloads
- Your server handles one request at a time — everyone else waits
- Your CPU sits idle while waiting for a database response

Java gives you tools to create and manage threads. But multiple chefs in a kitchen can bump into each other, reach for the same knife at the same time, or shout conflicting instructions. That is where concurrency bugs come from.

---

## Part 1 — Thread Basics

---

### What is a Thread?

A **thread** is the smallest unit of execution in a program. Every Java program starts with one thread — the **main thread**. When you create additional threads, they run concurrently alongside it.

Think of threads as separate workers each with their own to-do list, but all sharing the same office (memory/heap).

```
Main Thread          Thread A             Thread B
───────────          ──────────           ──────────
starts program  →    created by main      created by main
runs main()          runs task A          runs task B
                     ↕ (share heap)       ↕ (share heap)
```

Each thread has its own:
- **Stack** — its own call stack, local variables, method frames
- **Program counter** — tracks which instruction it is currently executing

Threads share:
- **Heap** — all objects created with `new`
- **Static fields** — class-level variables
- **Open files, sockets** — OS-level resources

---

### Creating Threads — 3 Ways

#### Way 1: Extend Thread

```java
public class MyThread extends Thread {
    private String taskName;

    public MyThread(String taskName) {
        this.taskName = taskName;
    }

    @Override
    public void run() {
        // This is what the thread will do
        for (int i = 1; i <= 5; i++) {
            System.out.println(taskName + " - step " + i);
            try {
                Thread.sleep(500); // pause 500ms between steps
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return;
            }
        }
    }
}

// How to use it:
MyThread t1 = new MyThread("Download");
MyThread t2 = new MyThread("Upload");

t1.start(); // starts the thread — calls run() in a NEW thread
t2.start(); // starts simultaneously

// Output (order will vary — both run at the same time):
// Download - step 1
// Upload - step 1
// Download - step 2
// Upload - step 2
```

**Critical distinction:**
```java
t1.run();   // WRONG — runs on the CURRENT thread, no new thread created
t1.start(); // CORRECT — creates a NEW thread, runs run() inside it
```

#### Way 2: Implement Runnable (Preferred)

```java
// Runnable is just a task description — it does not "know" it is a thread
// Preferred because your class can still extend something else
public class DownloadTask implements Runnable {
    private String url;

    public DownloadTask(String url) { this.url = url; }

    @Override
    public void run() {
        System.out.println("Downloading: " + url);
        // download logic
        System.out.println("Done: " + url);
    }
}

Thread thread = new Thread(new DownloadTask("https://example.com/file.zip"));
thread.start();

// Same thing with a lambda (Runnable is a functional interface):
Thread thread2 = new Thread(() -> System.out.println("Running in new thread"));
thread2.start();
```

#### Way 3: Callable (Returns a Result)

`Runnable.run()` cannot return a value or throw checked exceptions. `Callable` solves both.

```java
Callable<Integer> task = () -> {
    System.out.println("Computing...");
    Thread.sleep(1000);
    return 42; // returns a result
};

ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(task); // submit and get a "receipt"

System.out.println("Doing other work while task runs...");

Integer result = future.get(); // BLOCKS here until task finishes
System.out.println("Result: " + result); // 42

executor.shutdown();
```

---

### Thread Lifecycle — The 6 States

```
NEW ──── start() ──→ RUNNABLE ──→ RUNNING
                         ↑            │
                         │       (waiting for lock)
                         │            ↓
                         │        BLOCKED
                         │
                         │       (sleep / wait / join)
                         │            ↓
                         └──── WAITING / TIMED_WAITING
                                      │
                              (condition met / timeout)
                                      ↓
                               TERMINATED
```

| State | Meaning |
|---|---|
| **NEW** | Created but `start()` not called yet |
| **RUNNABLE** | Ready to run, waiting for CPU time |
| **RUNNING** | Actually executing on CPU |
| **BLOCKED** | Waiting to acquire a `synchronized` lock |
| **WAITING** | Waiting indefinitely for `notify()`, `join()`, etc. |
| **TIMED_WAITING** | Waiting with a timeout — `sleep(ms)`, `join(ms)` |
| **TERMINATED** | `run()` finished — thread is dead |

```java
Thread t = new Thread(() -> {
    try { Thread.sleep(2000); } catch (InterruptedException e) {}
});

System.out.println(t.getState()); // NEW
t.start();
System.out.println(t.getState()); // RUNNABLE or TIMED_WAITING
t.join();
System.out.println(t.getState()); // TERMINATED
```

---

### Thread Properties

**Naming:**
```java
Thread t = new Thread(() -> {
    System.out.println("I am: " + Thread.currentThread().getName());
});
t.setName("Worker-1");
t.start(); // I am: Worker-1
```

**Priority:**
```java
thread.setPriority(Thread.MAX_PRIORITY); // 10 — hint to scheduler, not guaranteed
thread.setPriority(Thread.MIN_PRIORITY); // 1
thread.setPriority(Thread.NORM_PRIORITY);// 5 — default
```

**Daemon threads:**
```java
Thread daemon = new Thread(() -> {
    while (true) { cleanUpOldFiles(); Thread.sleep(60000); }
});
daemon.setDaemon(true); // must be set BEFORE start()
daemon.start();
// JVM exits when all non-daemon threads finish — daemon threads are killed automatically
// Never use daemon threads for critical work — they can be killed mid-operation
```

**currentThread():**
```java
// Static method — gives you the thread currently running this line of code
Thread me = Thread.currentThread();
System.out.println(me.getName());    // thread name
System.out.println(me.isDaemon());   // is it a daemon?
System.out.println(me.isAlive());    // is it still running?
```

---

### Key Thread Methods

---

#### sleep() — Pause the Current Thread

`Thread.sleep(ms)` pauses the **currently running thread** for at least the given milliseconds. The thread gives up CPU time but holds any locks it currently has.

```java
public void countdown() throws InterruptedException {
    for (int i = 5; i >= 1; i--) {
        System.out.println("T-minus: " + i);
        Thread.sleep(1000); // pause 1 second
    }
    System.out.println("Liftoff!");
}

Thread t = new Thread(() -> {
    try {
        countdown();
    } catch (InterruptedException e) {
        System.out.println("Countdown aborted!");
        Thread.currentThread().interrupt(); // restore the interrupted flag
    }
});
t.start();
```

**What sleep() does NOT do:**
- Does NOT release locks the thread is holding
- Does NOT guarantee exact wake-up time (OS may delay it)
- Does NOT create a happens-before relationship between threads (do not use it as synchronization)

---

#### join() — Wait for Another Thread to Finish

`join()` makes the **calling thread wait** until the target thread finishes. It is how you say: "I need your result before I can continue."

```java
public class JoinExample {
    public static void main(String[] args) throws InterruptedException {

        Thread worker = new Thread(() -> {
            System.out.println("Worker: starting...");
            try { Thread.sleep(3000); } catch (InterruptedException e) {}
            System.out.println("Worker: done!");
        });

        worker.start();

        System.out.println("Main: waiting for worker...");
        worker.join(); // Main thread BLOCKS here until worker finishes
        System.out.println("Main: continuing after worker");
    }
}

// Guaranteed output order:
// Worker: starting...
// Main: waiting for worker...
// Worker: done!
// Main: continuing after worker
```

**join() with timeout — never wait forever:**
```java
worker.join(5000); // wait at most 5 seconds

if (worker.isAlive()) {
    System.out.println("Worker too slow — moving on");
    worker.interrupt();
} else {
    System.out.println("Worker finished in time");
}
```

**Real-world use case — parallel tasks then combine:**
```java
int[] results = new int[3];

Thread t1 = new Thread(() -> { results[0] = computePartA(); });
Thread t2 = new Thread(() -> { results[1] = computePartB(); });
Thread t3 = new Thread(() -> { results[2] = computePartC(); });

t1.start(); t2.start(); t3.start(); // all 3 run in parallel

t1.join(); t2.join(); t3.join();    // wait for ALL to finish

// Now safe — all threads have written their results
int total = results[0] + results[1] + results[2];
```

---

#### interrupt() — Signal a Thread to Stop

Java has no "kill thread" switch. Instead, you politely signal a thread that it should stop, and the thread decides how to react.

```java
Thread worker = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) { // check the flag in the loop
        doWork();
    }
    System.out.println("Stopping cleanly");
});

worker.start();
Thread.sleep(3000);
worker.interrupt(); // sets the interrupted flag — worker sees it on next loop check
```

**How interrupt interacts with sleep() and wait():**
```java
Thread worker = new Thread(() -> {
    try {
        while (true) {
            doWork();
            Thread.sleep(1000); // if interrupted while sleeping...
        }
    } catch (InterruptedException e) {
        // sleep() woke up early AND cleared the interrupted flag
        System.out.println("Interrupted while sleeping — stopping");
        Thread.currentThread().interrupt(); // restore the flag before returning
    }
});
```

**Three interrupt-related methods:**
```java
thread.interrupt();                          // sets the interrupted flag
Thread.currentThread().isInterrupted();      // checks flag, does NOT clear it
Thread.interrupted();                        // checks flag AND clears it (static)
```

---

#### yield() — Suggest CPU Handover

A gentle hint to the scheduler: "I am happy to let other threads run." The scheduler may completely ignore it. Rarely needed in modern code since ExecutorService handles scheduling.

```java
public void run() {
    for (int i = 0; i < 1000; i++) {
        doWork(i);
        Thread.yield(); // politely offer CPU — scheduler may ignore
    }
}
```

---

### wait(), notify(), notifyAll() — Thread Communication

These allow threads to communicate through a shared object. A thread can say "I am waiting for a condition" and another thread can say "the condition is now true — wake up."

They only work **inside a synchronized block** on the same object.

```java
public class MessageBox {
    private String message = null;
    private boolean hasMessage = false;

    // Producer: puts a message in the box
    public synchronized void put(String msg) throws InterruptedException {
        while (hasMessage) {
            wait(); // box is full — release lock and wait
                    // KEY: wait() releases the lock while waiting
                    //      sleep() does NOT release the lock
        }
        this.message = msg;
        this.hasMessage = true;
        notifyAll(); // wake up any waiting consumers
    }

    // Consumer: takes a message from the box
    public synchronized String take() throws InterruptedException {
        while (!hasMessage) {
            wait(); // box is empty — release lock and wait
        }
        String msg = this.message;
        this.hasMessage = false;
        notifyAll(); // wake up any waiting producers
        return msg;
    }
}
```

**Always use wait() in a while loop, never an if:**
```java
// WRONG — spurious wakeup skips straight to work even if condition is false
synchronized (lock) {
    if (!condition) wait();
    doWork(); // condition might still be false!
}

// CORRECT — spurious wakeup re-checks condition
synchronized (lock) {
    while (!condition) wait(); // loops back and checks again
    doWork();
}
```

| Method | What it does |
|---|---|
| `wait()` | Releases lock, suspends current thread, waits for notify |
| `notify()` | Wakes ONE waiting thread (which one is undefined) |
| `notifyAll()` | Wakes ALL waiting threads — they compete for the lock |

---

## Part 2 — The Java Memory Model (JMM)

Before any synchronization tool makes sense, you need to understand WHY they exist.

---

### The Problem: CPU Caching and Reordering

Modern CPUs do not read from RAM on every access — they cache values in registers and CPU caches (L1/L2/L3). The JVM also reorders instructions for performance. In a single thread this is invisible. Across threads, it causes invisible bugs.

```
CPU Core 1 (Thread A)          CPU Core 2 (Thread B)
┌─────────────────┐            ┌─────────────────┐
│  L1 Cache       │            │  L1 Cache       │
│  running=false  │            │  running=true   │  ← stale cached value
└─────────────────┘            └─────────────────┘
        ↕                               ↕
             Main Memory (RAM)
             running = false  ← Thread A's write may be stuck in L1 cache
```

**Three categories of bugs this causes:**

| Problem | Description | Naive example |
|---|---|---|
| **Visibility** | Thread A's write is invisible to Thread B | `stop = true` never seen |
| **Atomicity** | Multi-step operation interrupted mid-way | `count++` loses increments |
| **Ordering** | JVM reorders instructions unexpectedly | `value=42` reordered after `ready=true` |

---

### Happens-Before — The JMM's Guarantee

If action A **happens-before** action B, everything A wrote is **guaranteed visible** to B. It is not about clock time — it is a formal visibility guarantee.

```java
// Rule 1: thread.start()
int x = 5;
thread.start();
// x=5 is guaranteed visible inside the new thread

// Rule 2: thread.join()
thread.join();
// Everything the thread wrote is visible after join() returns

// Rule 3: synchronized — unlock happens-before next lock of same monitor
synchronized(lock) { data = 10; }   // Thread A writes and unlocks
synchronized(lock) { use(data); }   // Thread B locks — guaranteed sees data=10

// Rule 4: volatile write happens-before volatile read
volatile boolean flag = false;
flag = true;   // Thread A
if (flag) { } // Thread B — guaranteed to see true
```

---

## Part 3 — Synchronization Tools

---

### synchronized — Mutual Exclusion + Visibility

`synchronized` gives two guarantees at once:
1. **Mutual exclusion** — only one thread inside the block at a time
2. **Visibility** — on entry reads fresh values; on exit flushes writes to main memory

```java
public class BankAccount {
    private double balance = 1000.0;

    // Without synchronized — two threads can both read balance=1000,
    // both subtract 200, both write 800 — only 200 deducted instead of 400
    public synchronized void withdraw(double amount) {
        if (balance < amount) throw new IllegalStateException("Insufficient funds");
        balance -= amount; // only ONE thread executes this at a time
    }

    public synchronized double getBalance() {
        return balance; // synchronized getter ensures fresh value
    }
}
```

**Which object is the lock:**
```java
public synchronized void instanceMethod() {
    // Lock: 'this' instance
}

public static synchronized void staticMethod() {
    // Lock: MyClass.class — DIFFERENT from instance lock
}

synchronized (this)         { } // same as synchronized instance method
synchronized (someObject)   { } // custom lock object
synchronized (MyClass.class){ } // same as static synchronized
```

**Reentrancy — same thread can re-enter its own lock:**
```java
public synchronized void outer() {
    inner(); // same thread, already holds the lock — no deadlock, Java allows this
}
public synchronized void inner() { }
```

---

### volatile — Visibility Without Locking

`volatile` is lighter than `synchronized`. It only solves visibility — NOT atomicity, NOT mutual exclusion.

Marking a field `volatile` means:
- Every write goes directly to main memory (bypasses CPU cache)
- Every read comes directly from main memory (never reads from cache)

```java
public class ServerLoop {
    private volatile boolean running = true;

    public void start() {
        new Thread(() -> {
            while (running) {        // reads from main memory every time
                handleNextRequest();
            }
            System.out.println("Server stopped");
        }).start();
    }

    public void stop() {
        running = false; // goes directly to main memory — other thread sees it immediately
    }
}
```

**Where volatile fails — non-atomic compound operations:**
```java
private volatile int count = 0;

public void increment() {
    count++; // NOT atomic even with volatile
             // = READ count (5) + ADD 1 + WRITE count (6)
             // Two threads: both read 5, both write 6 — one increment lost
}
// Fix: use AtomicInteger or synchronized
```

**When to use volatile:**
- A flag one thread writes and others read
- A reference that is replaced (not mutated) atomically
- Single writer, multiple readers of a primitive

---

### Atomic Classes — Lock-Free Thread Safety

`java.util.concurrent.atomic` uses CPU-level **Compare-And-Swap (CAS)**: "Set value to NEW only if it is currently EXPECTED — if someone changed it, retry."

```java
// AtomicInteger — thread-safe int operations
AtomicInteger counter = new AtomicInteger(0);

counter.incrementAndGet();          // atomic count++, returns new value
counter.decrementAndGet();          // atomic count--
counter.addAndGet(5);               // atomic count += 5
counter.getAndSet(0);               // atomic read then reset to 0
counter.compareAndSet(5, 10);       // set to 10 only if currently 5

// AtomicBoolean — thread-safe boolean, great for one-time init
AtomicBoolean initialized = new AtomicBoolean(false);

public void initOnce() {
    if (initialized.compareAndSet(false, true)) {
        // Only ONE thread will EVER enter this block
        // First thread: false→true, enters
        // All others: already true, skips
        doExpensiveInit();
    }
}

// AtomicReference — thread-safe object reference swap
AtomicReference<String> status = new AtomicReference<>("IDLE");
boolean started = status.compareAndSet("IDLE", "RUNNING"); // atomic state machine
```

**LongAdder — fastest counter under high contention:**
```java
LongAdder hits = new LongAdder();

// Each thread updates its own internal cell — no fighting over one value
hits.increment();        // in request handler threads
long total = hits.sum(); // aggregates all cells when you
