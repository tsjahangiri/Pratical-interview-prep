# Java Concurrency — Part 1: Thread Basics

---

## What Is a Thread and Why Do We Need It?

Imagine a restaurant. One waiter doing everything — taking orders, cooking, washing dishes, serving — is painfully slow. Multiple waiters working at the same time on different tasks is **concurrency**.

In Java:
- Your **program** = the restaurant
- **Threads** = the waiters
- **Shared memory (heap)** = the kitchen they all use

Without concurrency:
- UI freezes while downloading a file
- A server handles one request at a time — everyone else waits in line
- Your 8-core CPU does the work of one core

Every Java program already has one thread — the **main thread**. You create more threads to do work in parallel.

---

## How Threads Share and Own Memory

Each thread has its **own private area**:
- **Stack** — local variables, method calls, parameters (only that thread sees this)
- **Program counter** — which line of code it is currently executing

All threads **share together**:
- **Heap** — every object you create with `new`
- **Static fields** — class-level variables
- **Open files, sockets** — system resources

This shared heap is what makes concurrency powerful — and dangerous. Two threads reading the same data is fine. Two threads writing the same data without coordination = bugs.

---

## Creating Threads — 3 Ways

### Way 1 — Extend Thread

```java
public class WorkerThread extends Thread {

    private String name;

    public WorkerThread(String name) {
        this.name = name;
    }

    @Override
    public void run() {
        // Everything here runs inside the new thread
        for (int i = 1; i <= 3; i++) {
            System.out.println(name + " → step " + i
                + " [thread: " + Thread.currentThread().getName() + "]");
            try {
                Thread.sleep(500); // pause 500ms to simulate work
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return;
            }
        }
    }
}

// Usage:
WorkerThread t1 = new WorkerThread("Downloader");
WorkerThread t2 = new WorkerThread("Uploader");

t1.start(); // ← creates a NEW thread, runs run() inside it
t2.start(); // ← starts simultaneously

// WRONG:
// t1.run(); ← this does NOT create a new thread — runs on current thread like a normal method call
```

Output (order varies — both run at the same time):
```
Downloader → step 1 [thread: Thread-0]
Uploader   → step 1 [thread: Thread-1]
Downloader → step 2 [thread: Thread-0]
Uploader   → step 2 [thread: Thread-1]
...
```

**The golden rule:** Always call `start()`, never `run()` directly. Calling `run()` just executes the method on the current thread — no new thread is created.

---

### Way 2 — Implement Runnable (Preferred)

`Runnable` separates the **task** from the **thread**. Your class stays free to extend something else.

```java
public class ReportTask implements Runnable {

    private String reportName;

    public ReportTask(String reportName) {
        this.reportName = reportName;
    }

    @Override
    public void run() {
        System.out.println("Generating report: " + reportName);
        // ... report logic
        System.out.println("Report done: " + reportName);
    }
}

// Wrap in Thread to run it
Thread t = new Thread(new ReportTask("Q4-Sales"));
t.start();

// Or with a lambda — since Runnable is a functional interface (one method)
Thread t2 = new Thread(() -> System.out.println("Running in background"));
t2.start();
```

Runnable is preferred over extending Thread because:
- Java only allows extending one class — don't waste it on Thread
- It cleanly separates "what to do" from "how to run it"
- Works naturally with ExecutorService (covered in Part 2)

---

### Way 3 — Callable + Future (Gets a Result Back)

`Runnable.run()` returns nothing and cannot throw checked exceptions. When you need a result from a thread, use `Callable`.

```java
import java.util.concurrent.*;

Callable<Integer> task = () -> {
    System.out.println("Computing sum...");
    Thread.sleep(2000); // simulate heavy computation
    return 100 + 200;   // returns a result
};

ExecutorService executor = Executors.newSingleThreadExecutor();

Future<Integer> future = executor.submit(task);
// Future = a "ticket" — you hand in the task and get a receipt
// The task runs in a background thread immediately

System.out.println("Task submitted. Doing other work...");
Thread.sleep(500); // main thread keeps doing its own work

Integer result = future.get(); // BLOCKS here until the task finishes
System.out.println("Result: " + result); // 300

executor.shutdown();
```

**Future methods:**

```java
future.get();                           // block until done, return result
future.get(5, TimeUnit.SECONDS);        // block max 5 seconds, throw TimeoutException if slow
future.isDone();                        // check without blocking — true if finished
future.cancel(true);                    // try to cancel the task
```

---

## Thread Lifecycle — The 6 States

A thread moves through these states during its life:

```
NEW
 │
 │  start()
 ↓
RUNNABLE ←─────────────────────────────────────┐
 │                                             │
 │  gets CPU                                  │ lock released / notified / timeout
 ↓                                             │
RUNNING                                        │
 │                                             │
 ├──→ waiting for synchronized lock ──→ BLOCKED
 │
 ├──→ wait() / join() (no timeout) ──→ WAITING
 │
 ├──→ sleep(ms) / join(ms) / wait(ms) ──→ TIMED_WAITING
 │
 └──→ run() finishes ──→ TERMINATED
```

| State | What is happening |
|---|---|
| **NEW** | Thread object created, `start()` not called yet |
| **RUNNABLE** | Ready to run, waiting for the OS to give it CPU time |
| **RUNNING** | Actually executing on CPU right now |
| **BLOCKED** | Stuck waiting to get a `synchronized` lock someone else holds |
| **WAITING** | Suspended indefinitely — waiting for `notify()`, `join()`, etc. |
| **TIMED_WAITING** | Suspended for a set time — `sleep(ms)`, `join(ms)`, `wait(ms)` |
| **TERMINATED** | `run()` has finished — thread is dead, cannot be restarted |

```java
Thread t = new Thread(() -> {
    try { Thread.sleep(3000); } catch (InterruptedException e) {}
});

System.out.println(t.getState()); // NEW

t.start();
Thread.sleep(100); // let it start
System.out.println(t.getState()); // TIMED_WAITING (inside sleep)

t.join();
System.out.println(t.getState()); // TERMINATED
```

---

## Key Thread Properties

### Thread Name
```java
Thread t = new Thread(() -> {
    System.out.println("I am thread: " + Thread.currentThread().getName());
});
t.setName("DatabasePoller");
t.start();
// I am thread: DatabasePoller
```

### Daemon Threads

A **daemon thread** is a background thread that the JVM kills automatically when all normal (non-daemon) threads finish. Use it for housekeeping tasks — log flushing, cache cleanup, heartbeats.

```java
Thread cleanup = new Thread(() -> {
    while (true) {
        try {
            purgeOldSessions();
            Thread.sleep(60_000); // run every minute
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return;
        }
    }
});

cleanup.setDaemon(true); // MUST be set before start()
cleanup.start();

// When main thread exits, JVM kills this daemon thread automatically
// Do NOT use daemon threads for work that must finish (writing files, DB writes)
```

---

## The 5 Core Thread Methods

---

### 1. sleep() — Pause This Thread

`Thread.sleep(ms)` pauses the **currently running thread** for at least the given milliseconds. It gives up the CPU but **holds onto any locks** it has.

```java
public class Ticker {
    public static void main(String[] args) throws InterruptedException {
        for (int i = 5; i >= 1; i--) {
            System.out.println("Tick: " + i);
            Thread.sleep(1000); // pause 1 second before next tick
        }
        System.out.println("Done!");
    }
}
```

Two threads using sleep independently — they do not affect each other:

```java
Thread fast = new Thread(() -> {
    try {
        for (int i = 0; i < 5; i++) {
            System.out.println("Fast: " + i);
            Thread.sleep(200); // wakes every 200ms
        }
    } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
});

Thread slow = new Thread(() -> {
    try {
        for (int i = 0; i < 3; i++) {
            System.out.println("Slow: " + i);
            Thread.sleep(700); // wakes every 700ms
        }
    } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
});

fast.start();
slow.start();

// Output interleaved — both running concurrently:
// Fast: 0
// Slow: 0
// Fast: 1
// Fast: 2
// Fast: 3
// Slow: 1
// Fast: 4
// Slow: 2
```

**What sleep() does NOT do:**
- Does NOT release synchronized locks
- Does NOT guarantee exact wake-up time (OS decides)
- Does NOT create any visibility guarantee between threads

---

### 2. join() — Wait for Another Thread to Finish

`join()` makes the **calling thread wait** until the target thread dies. It is how you say: "I need your result before I can continue."

```java
public class JoinDemo {
    public static void main(String[] args) throws InterruptedException {

        Thread worker = new Thread(() -> {
            System.out.println("Worker: processing data...");
            try { Thread.sleep(3000); } catch (InterruptedException e) {}
            System.out.println("Worker: finished!");
        });

        worker.start();
        System.out.println("Main: waiting for worker...");

        worker.join(); // Main thread STOPS here until worker finishes

        System.out.println("Main: worker is done — I can continue now");
    }
}
```

Guaranteed output order (join makes this deterministic):
```
Worker: processing data...
Main: waiting for worker...
(3 seconds pass)
Worker: finished!
Main: worker is done — I can continue now
```

**Without join()** — main might print "continuing" before worker finishes:
```java
worker.start();
// NO join() here
System.out.println("Main: continuing..."); // prints immediately, worker still running
```

---

**Real power of join() — run tasks in parallel then combine:**

```java
public class ParallelWork {
    static int resultA, resultB, resultC;

    public static void main(String[] args) throws InterruptedException {

        // Three tasks that can run independently at the same time
        Thread t1 = new Thread(() -> { resultA = computeA(); }); // takes 3s
        Thread t2 = new Thread(() -> { resultB = computeB(); }); // takes 2s
        Thread t3 = new Thread(() -> { resultC = computeC(); }); // takes 1s

        long start = System.currentTimeMillis();

        t1.start(); t2.start(); t3.start(); // all 3 START simultaneously

        t1.join(); t2.join(); t3.join();    // wait for ALL 3 to finish

        long elapsed = System.currentTimeMillis() - start;
        // elapsed ≈ 3000ms (longest task) NOT 6000ms (sequential)

        int total = resultA + resultB + resultC;
        System.out.println("Total: " + total + " in " + elapsed + "ms");
    }
}
```

**join() with timeout — do not wait forever:**

```java
worker.join(5000); // wait at most 5 seconds

if (worker.isAlive()) {
    System.out.println("Worker is too slow — giving up");
    worker.interrupt(); // tell the worker to stop
} else {
    System.out.println("Worker finished in time — great");
}
```

---

### 3. interrupt() — Signal a Thread to Stop

Java has no "kill" switch for threads. Instead, you **politely signal** a thread via `interrupt()`. The thread itself decides when and how to react.

Two scenarios:

**Scenario A — thread is running normally:**
```java
Thread worker = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) { // check the flag each loop
        processNextItem();
    }
    System.out.println("Worker: got interrupt signal, stopping cleanly");
    // do any cleanup here before returning
});

worker.start();
Thread.sleep(3000);  // let it work for 3 seconds
worker.interrupt();  // set the interrupted flag — worker checks it on next loop
```

**Scenario B — thread is sleeping or waiting:**
```java
Thread worker = new Thread(() -> {
    try {
        while (true) {
            processNextItem();
            Thread.sleep(1000);   // interrupted HERE — throws InterruptedException
        }
    } catch (InterruptedException e) {
        // sleep() woke up early AND cleared the interrupted flag automatically
        System.out.println("Worker: woken up by interrupt, stopping");
        Thread.currentThread().interrupt(); // restore the flag — good practice
    }
});

worker.start();
Thread.sleep(2500);
worker.interrupt(); // wakes the sleeping thread immediately
```

**Three interrupt-related methods — know the difference:**

```java
thread.interrupt();                     // sets the interrupted flag on that thread
Thread.currentThread().isInterrupted(); // reads the flag — does NOT clear it
Thread.interrupted();                   // reads AND clears the flag (static method)
```

Why restore the flag with `Thread.currentThread().interrupt()` after catching `InterruptedException`? Because `InterruptedException` automatically clears the flag. If you swallow it silently, the caller (e.g. an ExecutorService shutting down) checks the flag to know if the thread should stop — finds `false` — and cannot shut it down cleanly.

---

### 4. yield() — Politely Give Up the CPU

`Thread.yield()` hints to the OS scheduler: "I am willing to pause and let other threads run." The OS may completely ignore it. Used rarely in modern code — ExecutorService and thread pools handle scheduling automatically.

```java
public void run() {
    for (int i = 0; i < 1000; i++) {
        doWork(i);
        Thread.yield(); // hint: let another thread run if one is waiting
    }
}
```

---

### 5. wait(), notify(), notifyAll() — Threads Talking to Each Other

These three methods let threads **coordinate through a shared object**. One thread waits for a condition. Another thread creates the condition and wakes the waiting thread.

Rules you must know:
- They only work **inside a `synchronized` block** on the same object
- `wait()` **releases the lock** while waiting (unlike `sleep()` which holds it)
- Always call `wait()` inside a **while loop**, never an `if` — because of spurious wakeups

```java
public class OrderQueue {
    private final Queue<String> orders = new LinkedList<>();

    // Chef ADDS an order (producer)
    public synchronized void addOrder(String order) throws InterruptedException {
        while (orders.size() >= 5) {
            System.out.println("Queue full — chef waiting...");
            wait(); // release lock, suspend, wait for space
        }
        orders.add(order);
        System.out.println("Order added: " + order + " | Queue size: " + orders.size());
        notifyAll(); // wake up any waiting waiters
    }

    // Waiter TAKES an order (consumer)
    public synchronized String takeOrder() throws InterruptedException {
        while (orders.isEmpty()) {
            System.out.println("No orders — waiter waiting...");
            wait(); // release lock, suspend, wait for an order
        }
        String order = orders.poll();
        System.out.println("Order taken: " + order + " | Queue size: " + orders.size());
        notifyAll(); // wake up any waiting chefs
        return order;
    }
}

// Running it:
OrderQueue queue = new OrderQueue();

// Chef thread — produces orders
Thread chef = new Thread(() -> {
    try {
        String[] menu = {"Pizza", "Pasta", "Salad", "Soup", "Steak", "Sushi"};
        for (String dish : menu) {
            queue.addOrder(dish);
            Thread.sleep(300);
        }
    } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
});

// Waiter thread — consumes orders
Thread waiter = new Thread(() -> {
    try {
        for (int i = 0; i < 6; i++) {
            String order = queue.takeOrder();
            Thread.sleep(800); // waiter is slower than chef
        }
    } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
});

chef.start();
waiter.start();
```

**wait() vs sleep() — critical difference:**

| | `wait()` | `sleep()` |
|---|---|---|
| Releases lock? | **YES** — other threads can enter synchronized | **NO** — holds lock while paused |
| Where called | Inside `synchronized` block only | Anywhere |
| Woken by | `notify()` / `notifyAll()` | Timeout expiring or `interrupt()` |
| Purpose | Thread coordination | Simple pause |

---

## The Java Memory Model — Why Synchronization Exists

This is the foundation of all concurrency bugs. You must understand it to reason about any concurrent code.

### The Problem: CPU Caching

Modern CPUs do not read from RAM on every access. They keep copies of values in fast local caches (L1, L2). The JVM also reorders instructions for optimization.

In a single thread: invisible. Everything looks sequential.
Across threads: chaos.

```
Thread A (CPU Core 1)           Thread B (CPU Core 2)
┌──────────────────┐            ┌──────────────────┐
│ L1 Cache         │            │ L1 Cache         │
│ running = false  │            │ running = true ← │ stale! Thread B never saw the update
└──────────────────┘            └──────────────────┘
         ↕                               ↕
              Shared RAM
              running = false  ← Thread A wrote here, but Thread B cached the old value
```

```java
// Thread A writes false, Thread B never sees it — loops forever
public class VisibilityBug {
    private boolean running = true; // no volatile — stored in CPU cache

    public void stop()  { running = false; }   // Thread A

    public void loop()  {
        while (running) { doWork(); }           // Thread B — may cache running=true forever
    }
}
```

### Three Categories of Bugs

| Bug Type | Description | Classic example |
|---|---|---|
| **Visibility** | One thread's write is invisible to another | Flag written but never seen |
| **Atomicity** | An operation that looks single-step but isn't | `count++` = 3 separate steps |
| **Ordering** | JVM reorders instructions in surprising ways | `value=42` moved after `ready=true` |

### Happens-Before — The JMM's Guarantee

If action A **happens-before** action B, everything A wrote is guaranteed visible to B. These are the official happens-before rules:

```java
// Rule 1: thread.start()
sharedValue = 42;
thread.start(); // everything before start() IS visible inside the new thread

// Rule 2: thread.join()
thread.join();  // everything the thread wrote IS visible after join() returns

// Rule 3: synchronized — unlock → next lock on same monitor
synchronized(lock) { data = 99; }   // Thread A writes then unlocks
synchronized(lock) { use(data); }   // Thread B locks → guaranteed sees data = 99

// Rule 4: volatile write → volatile read
volatile boolean flag = false;
flag = true;     // Thread A — goes directly to RAM
if (flag) { }   // Thread B — reads directly from RAM, sees true
```

This is why `synchronized` and `volatile` are not just about "one thread at a time" — they also enforce **memory visibility**.

---

## Quick Reference Card

```
CREATING THREADS
  extends Thread        → override run(), call start()
  implements Runnable   → implement run(), wrap in Thread, call start()
  Callable + Future     → returns a result, use with ExecutorService

THREAD LIFECYCLE
  NEW → RUNNABLE → RUNNING → BLOCKED / WAITING / TIMED_WAITING → TERMINATED

KEY METHODS
  sleep(ms)             → pause current thread, holds locks
  join()                → calling thread waits for target thread to die
  join(ms)              → same but with a timeout
  interrupt()           → sets interrupted flag, wakes sleeping threads
  isInterrupted()       → check flag without clearing
  yield()               → hint to give up CPU (often ignored)

WAIT / NOTIFY (inside synchronized only)
  wait()                → release lock, suspend, wait for notify
  notify()              → wake one waiting thread
  notifyAll()           → wake all waiting threads

JMM RULES
  After start()         → parent writes visible in child thread
  After join()          → child writes visible in parent thread
  After unlock→lock     → writes inside lock visible to next lock holder
  After volatile write  → visible to next volatile read of same variable
```
