# Part 3 — Concurrency & Multithreading (Practice Workbook)

**56 exercises — 28 topics × 2.** Java 17+, realistic production concurrency. Every exercise has a *different* root cause — none is fixed by "just add `synchronized`." Review cold; write findings before the answer key.

> Multiple bugs per exercise are common. Some code *looks* unsafe but is correct. Some comments about thread-safety are wrong. Verify everything.

**Legend:** 🔴 Critical · 🟠 High · 🟡 Medium · 🟢 Low · ⚪ Nit

---

# EXERCISES

## Topic 1 — Thread

### Exercise 1 — Kicking off background work

**Scenario**
`ImportService` starts a long import "in the background" so the request returns quickly.

**Code**
```java
public void startImport(ImportJob job) {
    Thread worker = new Thread(() -> process(job));
    worker.run();
    log.info("Import started in background");
}
```

**Your Task**
Review whether this achieves background execution.

---

### Exercise 2 — One thread per request

**Scenario**
`WebhookController` handles inbound webhooks; each one triggers some processing that the author wanted off the request thread. Traffic spikes to thousands of webhooks/second.

**Code**
```java
@PostMapping("/webhook")
public ResponseEntity<Void> receive(@RequestBody Payload p) {
    new Thread(() -> handler.process(p)).start();
    return ResponseEntity.accepted().build();
}
```

**Your Task**
Review the threading approach under load.

---

## Topic 2 — Runnable

### Exercise 1 — Fire-and-forget task

**Scenario**
`ReindexService` submits a reindex task to a thread. If it fails, the team expects to see it.

**Code**
```java
public void reindexAsync() {
    Runnable task = () -> {
        searchIndex.rebuild(); // can throw
    };
    new Thread(task).start();
}
```

**Your Task**
Review error visibility for this task.

---

### Exercise 2 — Reused runnable with state

**Scenario**
`BatchCounter` is a `Runnable` submitted to several threads to process shards in parallel; it tallies processed rows.

**Code**
```java
public class BatchCounter implements Runnable {
    private int processed = 0;
    private final List<Row> shard;

    public BatchCounter(List<Row> shard) { this.shard = shard; }

    @Override
    public void run() {
        for (Row r : shard) {
            handle(r);
            processed++;
        }
    }
    public int getProcessed() { return processed; }
}

// usage: same instance handed to multiple threads
BatchCounter c = new BatchCounter(rows);
for (int i = 0; i < 4; i++) new Thread(c).start();
```

**Your Task**
Review this for correctness under parallel execution.

---

## Topic 3 — Callable

### Exercise 1 — Result never retrieved

**Scenario**
`PriceService` submits a pricing `Callable` that can throw a `PricingException`. The team reports "sometimes prices just silently don't update, no errors in logs."

**Code**
```java
public void updatePriceAsync(Product p) {
    executor.submit((Callable<Void>) () -> {
        BigDecimal price = computePrice(p); // may throw PricingException
        p.setPrice(price);
        return null;
    });
}
```

**Your Task**
Review why failures might be invisible.

---

### Exercise 2 — Parallel calls that aren't parallel

**Scenario**
`AggregatorService` fans out to several downstream services and combines results. It should run them concurrently.

**Code**
```java
public Report aggregate(List<String> sources) {
    List<Data> all = new ArrayList<>();
    for (String s : sources) {
        Future<Data> f = executor.submit(() -> fetch(s)); // blocking fetch
        all.add(f.get()); // wait for this one before submitting the next
    }
    return combine(all);
}
```

**Your Task**
Review whether the fan-out actually runs concurrently.

---

## Topic 4 — Executor

### Exercise 1 — The "async" executor

**Scenario**
`EmailService` uses an injected `Executor` to send emails off the request thread. The `Executor` bean is defined elsewhere.

**Code**
```java
@Configuration
class ExecutorConfig {
    @Bean
    Executor emailExecutor() {
        return Runnable::run; // "simple executor"
    }
}

@Service
class EmailService {
    private final Executor executor;
    EmailService(Executor executor) { this.executor = executor; }

    public void sendAsync(Email e) {
        executor.execute(() -> smtp.send(e)); // expected to be off-thread
    }
}
```

**Your Task**
Review whether `sendAsync` runs off the caller's thread.

---

### Exercise 2 — Cached pool for slow calls

**Scenario**
`ThumbnailService` resizes images by calling a slow external service, using a cached thread pool. Under a traffic burst, thousands of requests arrive at once.

**Code**
```java
private final ExecutorService pool = Executors.newCachedThreadPool();

public Future<byte[]> resize(Image img) {
    return pool.submit(() -> slowExternalResize(img)); // ~2s each
}
```

**Your Task**
Review the pool choice under burst load.

---

## Topic 5 — ExecutorService

### Exercise 1 — Submitted task that vanishes

**Scenario**
`AuditWriter.write` persists audit records off the request path. Occasionally records go missing with no trace.

**Code**
```java
private final ExecutorService pool = Executors.newFixedThreadPool(4);

public void write(AuditRecord record) {
    pool.submit(() -> auditRepository.save(record)); // save() can throw on DB hiccup
}
```

**Your Task**
Review why failures could be silent.

---

### Exercise 2 — Nested submit on a single-thread pool

**Scenario**
`OrderPipeline` uses a single-thread executor to serialize order processing. Processing an order also validates it, which is submitted to the same executor and waited on.

**Code**
```java
private final ExecutorService pool = Executors.newSingleThreadExecutor();

public void process(Order o) {
    pool.submit(() -> {
        Future<Boolean> valid = pool.submit(() -> validate(o)); // same pool
        if (valid.get()) {   // wait for validation
            persist(o);
        }
        return null;
    });
}
```

**Your Task**
Review the interaction between the nested submissions.

---

## Topic 6 — Future

### Exercise 1 — Waiting forever

**Scenario**
`QuoteService` calls a downstream pricing engine via a `Future` and returns the quote. The engine occasionally hangs.

**Code**
```java
public Quote getQuote(Request r) throws Exception {
    Future<Quote> f = executor.submit(() -> engine.price(r));
    return f.get(); // block for the result
}
```

**Your Task**
Review the blocking behavior and failure modes.

---

### Exercise 2 — Fire-and-forget futures

**Scenario**
`BatchProcessor.process` submits all items and returns. The caller then reports success to the user.

**Code**
```java
public void process(List<Item> items) {
    for (Item item : items) {
        executor.submit(() -> handle(item)); // returns Future, discarded
    }
    // method returns immediately; caller tells the user "all processed"
}
```

**Your Task**
Review whether "all processed" is a safe claim, and whether failures surface.

---

## Topic 7 — CompletableFuture

### Exercise 1 — Lost exception in a chain

**Scenario**
`ProfileService` loads a profile asynchronously and caches it. When loading fails, nothing appears in logs and the cache silently stays empty.

**Code**
```java
public void warmCache(Long userId) {
    CompletableFuture
        .supplyAsync(() -> profileClient.load(userId)) // may throw
        .thenAccept(profile -> cache.put(userId, profile));
    // no return, no error handling
}
```

**Your Task**
Review error handling in this pipeline.

---

### Exercise 2 — Blocking the async stage

**Scenario**
`DashboardService.load` composes several remote calls with `CompletableFuture`. Each remote call is blocking HTTP.

**Code**
```java
public Dashboard load(Long userId) {
    CompletableFuture<Orders> orders =
        CompletableFuture.supplyAsync(() -> ordersClient.get(userId));   // blocking HTTP
    CompletableFuture<Prefs> prefs =
        CompletableFuture.supplyAsync(() -> prefsClient.get(userId));    // blocking HTTP
    return orders.thenCombine(prefs, Dashboard::new).join();
}
```

**Your Task**
Review which threads run the blocking work and the implications.

---

## Topic 8 — synchronized

### Exercise 1 — Locking on a changing reference

**Scenario**
`BufferHolder` guards access to a swappable buffer with a lock object.

**Code**
```java
public class BufferHolder {
    private Object lock = new Object();
    private Buffer buffer = new Buffer();

    public void replaceBuffer() {
        lock = new Object();          // "reset the lock too"
        buffer = new Buffer();
    }

    public void write(byte[] data) {
        synchronized (lock) {
            buffer.append(data);
        }
    }
}
```

**Your Task**
Review whether the synchronization actually provides mutual exclusion.

---

### Exercise 2 — Lazy singleton

**Scenario**
`ConnectionManager` lazily initializes a shared, expensive manager on first use from multiple threads.

**Code**
```java
public class ConnectionManager {
    private static ConnectionManager instance;

    public static ConnectionManager getInstance() {
        if (instance == null) {
            synchronized (ConnectionManager.class) {
                if (instance == null) {
                    instance = new ConnectionManager(); // expensive init
                }
            }
        }
        return instance;
    }
}
```

**Your Task**
Review the lazy-initialization correctness.

---

## Topic 9 — volatile

### Exercise 1 — Volatile counter

**Scenario**
`RequestMeter` counts requests across many threads and exposes the count.

**Code**
```java
public class RequestMeter {
    private volatile long count = 0;

    public void increment() {
        count++;
    }
    public long get() { return count; }
}
```

**Your Task**
Review whether `volatile` makes this correct.

---

### Exercise 2 — Volatile reference to a mutable holder

**Scenario**
`ConfigCache` holds the current config in a `volatile` reference. Reloading updates fields on the existing config object.

**Code**
```java
public class ConfigCache {
    private volatile Config config = new Config();

    public void reload(Map<String, String> newValues) {
        config.getValues().clear();
        config.getValues().putAll(newValues); // mutate the existing object's map
    }

    public String get(String key) {
        return config.getValues().get(key);
    }
}
```

**Your Task**
Review whether `volatile` guarantees readers see the reloaded values safely.

---

## Topic 10 — Atomic classes

### Exercise 1 — Two atomics, one invariant

**Scenario**
`Stats` tracks total and count to compute a running average across threads.

**Code**
```java
public class Stats {
    private final AtomicLong total = new AtomicLong();
    private final AtomicLong count = new AtomicLong();

    public void record(long value) {
        total.addAndGet(value);
        count.incrementAndGet();
    }

    public double average() {
        return (double) total.get() / count.get();
    }
}
```

**Your Task**
Review the consistency of `average()` under concurrency.

---

### Exercise 2 — Atomic used non-atomically

**Scenario**
`SequenceGenerator` hands out increasing IDs to concurrent callers.

**Code**
```java
private final AtomicInteger seq = new AtomicInteger(0);

public int next() {
    int current = seq.get();
    seq.set(current + 1);
    return current;
}
```

**Your Task**
Review whether IDs are guaranteed unique.

---

## Topic 11 — Lock

### Exercise 1 — Lock without finally

**Scenario**
`InventoryService.decrement` guards stock changes with an explicit `Lock`.

**Code**
```java
private final Lock lock = new ReentrantLock();

public void decrement(String sku, int qty) {
    lock.lock();
    Stock s = stock.get(sku);
    if (s.available() < qty) {
        throw new InsufficientStockException(sku); // throws while holding the lock
    }
    s.reduce(qty);
    lock.unlock();
}
```

**Your Task**
Review the lock handling across all code paths.

---

### Exercise 2 — Writes locked, reads not

**Scenario**
`PositionTracker` stores a mutable position updated under a lock; readers fetch it without locking "for speed."

**Code**
```java
private final Lock lock = new ReentrantLock();
private Position position = new Position(0, 0);

public void update(int x, int y) {
    lock.lock();
    try {
        position = new Position(x, y);
    } finally {
        lock.unlock();
    }
}

public Position current() {
    return position; // read without the lock
}
```

**Your Task**
Review whether unlocked reads are safe here.

---

## Topic 12 — ReentrantLock

### Exercise 1 — Read-to-write upgrade

**Scenario**
`CacheService` uses a `ReentrantReadWriteLock`. On a miss it upgrades from read to write to populate the cache.

**Code**
```java
private final ReentrantReadWriteLock rw = new ReentrantReadWriteLock();

public Value get(String key) {
    rw.readLock().lock();
    try {
        Value v = cache.get(key);
        if (v == null) {
            rw.writeLock().lock(); // acquire write while holding read
            try {
                v = load(key);
                cache.put(key, v);
            } finally {
                rw.writeLock().unlock();
            }
        }
        return v;
    } finally {
        rw.readLock().unlock();
    }
}
```

**Your Task**
Review the lock-acquisition sequence.

---

### Exercise 2 — Ignored tryLock result

**Scenario**
`JobScheduler.runOnce` should skip a run if another thread already holds the scheduling lock.

**Code**
```java
private final ReentrantLock lock = new ReentrantLock();

public void runOnce(Job job) {
    lock.tryLock(); // attempt to acquire
    try {
        job.execute();
    } finally {
        lock.unlock();
    }
}
```

**Your Task**
Review the use of `tryLock`.

---

## Topic 13 — ConcurrentHashMap

### Exercise 1 — Heavy work inside compute

**Scenario**
`RateProvider` caches exchange rates. On a miss it fetches from a remote service inside `computeIfAbsent`. Called by many threads for many currencies.

**Code**
```java
private final ConcurrentHashMap<String, BigDecimal> rates = new ConcurrentHashMap<>();

public BigDecimal rate(String currency) {
    return rates.computeIfAbsent(currency, c -> remoteRateClient.fetch(c)); // slow network call
}
```

**Your Task**
Review the use of `computeIfAbsent` here.

---

### Exercise 2 — Null values

**Scenario**
`UserCache` caches user lookups. A lookup can return "no such user."

**Code**
```java
private final ConcurrentHashMap<Long, User> cache = new ConcurrentHashMap<>();

public User get(Long id) {
    return cache.computeIfAbsent(id, key -> userRepository.findById(key).orElse(null));
}
```

**Your Task**
Review this caching approach.

---

## Topic 14 — Race conditions

### Exercise 1 — Lazy field init

**Scenario**
`TemplateEngine` lazily compiles a template on first use, storing it in a field. The engine is shared across request threads.

**Code**
```java
public class TemplateEngine {
    private Template compiled;

    public String render(Model m) {
        if (compiled == null) {
            compiled = compile(loadSource()); // expensive
        }
        return compiled.apply(m);
    }
}
```

**Your Task**
Review for race conditions.

---

### Exercise 2 — Reserve then commit

**Scenario**
`SeatBooking.book` checks a seat is free, then marks it booked. Many users try the same popular event concurrently.

**Code**
```java
public boolean book(String seatId, String userId) {
    Seat seat = seats.get(seatId);
    if (seat.getStatus() == Status.FREE) {   // check
        seat.setStatus(Status.BOOKED);       // act
        seat.setBookedBy(userId);
        return true;
    }
    return false;
}
```

**Your Task**
Review for race conditions and the resulting business impact.

---

## Topic 15 — Visibility

### Exercise 1 — Stop flag

**Scenario**
`PollingWorker` loops until asked to stop. `stop()` is called from a different thread (a shutdown hook).

**Code**
```java
public class PollingWorker implements Runnable {
    private boolean running = true;

    public void stop() { running = false; }

    @Override
    public void run() {
        while (running) {
            poll();
        }
    }
}
```

**Your Task**
Review whether `stop()` reliably ends the loop.

---

### Exercise 2 — Publishing a built object

**Scenario**
`ConfigLoader` builds a config on a background thread and publishes it to a shared field other threads read.

**Code**
```java
public class ConfigLoader {
    private Config config; // not volatile, not final

    public void loadInBackground() {
        new Thread(() -> {
            Config c = new Config();
            c.setTimeout(30);
            c.setRetries(3);
            config = c; // publish
        }).start();
    }

    public Config getConfig() { return config; } // other threads read
}
```

**Your Task**
Review the publication of the built object.

---

## Topic 16 — Atomicity

### Exercise 1 — Bounded add

**Scenario**
`ConnectionPool.tryAdd` adds a connection only if under capacity. Called concurrently as connections are created.

**Code**
```java
private final List<Conn> conns = Collections.synchronizedList(new ArrayList<>());
private static final int MAX = 50;

public boolean tryAdd(Conn c) {
    if (conns.size() < MAX) {   // atomic call
        conns.add(c);           // atomic call
        return true;
    }
    return false;
}
```

**Your Task**
Review whether the capacity bound holds under concurrency.

---

### Exercise 2 — Non-volatile long

**Scenario**
`PriceTicker` stores the latest price as a plain `long` (price in cents), written by a market-data thread and read by request threads.

**Code**
```java
public class PriceTicker {
    private long priceCents; // written by one thread, read by many

    public void update(long p) { priceCents = p; }
    public long latest() { return priceCents; }
}
```

**Your Task**
Review the safety of reading/writing this field across threads (consider 32-bit JVMs).

---

## Topic 17 — Deadlocks

### Exercise 1 — Transfer lock ordering

**Scenario**
`AccountService.transfer` locks both accounts to move money. Called concurrently for many account pairs, including A→B and B→A simultaneously.

**Code**
```java
public void transfer(Account from, Account to, BigDecimal amt) {
    synchronized (from) {
        synchronized (to) {
            from.debit(amt);
            to.credit(amt);
        }
    }
}
```

**Your Task**
Review for deadlock potential.

---

### Exercise 2 — Non-reentrant re-acquire

**Scenario**
`ResourceGuard` protects a critical section with a `Semaphore(1)` used as a mutex. `process` calls `helper`, which also guards with the same semaphore.

**Code**
```java
private final Semaphore mutex = new Semaphore(1);

public void process() throws InterruptedException {
    mutex.acquire();
    try {
        helper();
    } finally {
        mutex.release();
    }
}

private void helper() throws InterruptedException {
    mutex.acquire();   // same thread re-acquires
    try {
        doWork();
    } finally {
        mutex.release();
    }
}
```

**Your Task**
Review what happens when `process` runs.

---

## Topic 18 — Starvation

### Exercise 1 — One slow task blocks the rest

**Scenario**
`ScheduledJobs` runs periodic jobs, including a critical heartbeat every second, on a single-thread scheduled executor. One job occasionally runs for minutes.

**Code**
```java
private final ScheduledExecutorService scheduler =
        Executors.newSingleThreadScheduledExecutor();

public void init() {
    scheduler.scheduleAtFixedRate(this::heartbeat, 0, 1, TimeUnit.SECONDS);
    scheduler.scheduleAtFixedRate(this::generateHeavyReport, 0, 5, TimeUnit.MINUTES);
}
```

**Your Task**
Review whether the heartbeat can be starved.

---

### Exercise 2 — Readers starving the writer

**Scenario**
`PriceBook` is read constantly by pricing threads and updated occasionally by a market-data thread, guarded by a `ReadWriteLock`.

**Code**
```java
private final ReadWriteLock rw = new ReentrantReadWriteLock(); // default (non-fair)

public BigDecimal read(String sym) {
    rw.readLock().lock();
    try { return book.get(sym); } finally { rw.readLock().unlock(); }
}

public void update(String sym, BigDecimal price) {
    rw.writeLock().lock();
    try { book.put(sym, price); } finally { rw.writeLock().unlock(); }
}
```

**Your Task**
Review whether the writer can be starved under heavy read load, and how you'd reason about it.

---

## Topic 19 — Thread contention

### Exercise 1 — One lock for everything

**Scenario**
`SessionStore` guards all session operations with a single method-level lock. It's on the hot path at high request rates.

**Code**
```java
public class SessionStore {
    private final Map<String, Session> sessions = new HashMap<>();

    public synchronized Session get(String id) { return sessions.get(id); }
    public synchronized void put(String id, Session s) { sessions.put(id, s); }
    public synchronized void touch(String id) { sessions.get(id).touch(); }
}
```

**Your Task**
Review the locking granularity and its effect at scale.

---

### Exercise 2 — Shared Random

**Scenario**
`TokenService` generates random tokens on a very hot path across many threads.

**Code**
```java
public class TokenService {
    private final Random random = new Random();

    public String token() {
        byte[] bytes = new byte[16];
        random.nextBytes(bytes); // called from many threads
        return encode(bytes);
    }
}
```

**Your Task**
Review the concurrency and contention characteristics (assume the randomness quality itself is acceptable for this use).

---

## Topic 20 — Thread pools

### Exercise 1 — Fixed pool, unbounded queue

**Scenario**
`IngestService` submits incoming events to a fixed pool. During spikes, producers vastly outpace the 8 worker threads.

**Code**
```java
private final ExecutorService pool = Executors.newFixedThreadPool(8);

public void ingest(Event e) {
    pool.submit(() -> process(e)); // process() ~50ms
}
```

**Your Task**
Review the pool's behavior under sustained overload.

---

### Exercise 2 — Cached pool for long tasks

**Scenario**
`BackupService` runs long-running (minutes) backup tasks. It uses a cached thread pool, and backups can be triggered in large batches.

**Code**
```java
private final ExecutorService pool = Executors.newCachedThreadPool();

public void backupAll(List<Volume> volumes) {
    for (Volume v : volumes) {           // could be hundreds
        pool.submit(() -> backup(v));    // each runs for minutes
    }
}
```

**Your Task**
Review the pool choice for this workload.

---

## Topic 21 — Bounded queues

### Exercise 1 — Unhandled rejection

**Scenario**
`MetricsPipeline` uses a `ThreadPoolExecutor` with a bounded queue to apply backpressure. Under overload, submissions start failing and callers crash.

**Code**
```java
private final ThreadPoolExecutor pool = new ThreadPoolExecutor(
        4, 4, 0L, TimeUnit.MILLISECONDS,
        new ArrayBlockingQueue<>(100)); // default AbortPolicy

public void record(Metric m) {
    pool.submit(() -> sink.write(m)); // throws RejectedExecutionException when full
}
```

**Your Task**
Review the rejection behavior and how the caller experiences overload.

---

### Exercise 2 — "Bounded" queue that isn't

**Scenario**
`OrderQueue` is a producer/consumer buffer meant to bound memory use between a fast producer and slower consumers.

**Code**
```java
private final BlockingQueue<Order> queue = new LinkedBlockingQueue<>(); // no capacity arg

public void submit(Order o) throws InterruptedException {
    queue.put(o); // producer
}

public Order take() throws InterruptedException {
    return queue.take(); // consumer
}
```

**Your Task**
Review whether this actually bounds memory.

---

## Topic 22 — CPU-bound workloads

### Exercise 1 — Oversized pool for computation

**Scenario**
`ImageProcessor` runs pure CPU-bound transforms (no I/O). The pool was sized "big for throughput." The host has 8 cores.

**Code**
```java
private final ExecutorService pool = Executors.newFixedThreadPool(200);

public void processAll(List<Image> images) {
    for (Image img : images) {
        pool.submit(() -> cpuIntensiveTransform(img)); // pure CPU
    }
}
```

**Your Task**
Review the pool sizing for a CPU-bound workload.

---

### Exercise 2 — No isolation between CPU work and latency-sensitive work

**Scenario**
`AnalyticsService` runs heavy CPU-bound aggregations on the same executor that also serves quick, latency-sensitive lookups.

**Code**
```java
private final ExecutorService shared = Executors.newFixedThreadPool(8);

public Future<Report> heavyAggregate(Query q) {
    return shared.submit(() -> crunchMillionsOfRows(q)); // seconds of CPU
}

public Future<Item> quickLookup(Long id) {
    return shared.submit(() -> lookup(id)); // must be fast
}
```

**Your Task**
Review the sharing of one pool between these two workloads.

---

## Topic 23 — I/O-bound workloads

### Exercise 1 — Pool sized to cores for I/O

**Scenario**
`CrawlerService` fetches thousands of URLs (each mostly network wait, ~500ms). The pool is sized to the core count for "efficiency." Host has 8 cores.

**Code**
```java
private final ExecutorService pool =
        Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());

public void crawl(List<String> urls) {
    for (String url : urls) {
        pool.submit(() -> httpClient.get(url)); // mostly waiting on network
    }
}
```

**Your Task**
Review the pool sizing for an I/O-bound workload.

---

### Exercise 2 — One slow dependency ties up the pool

**Scenario**
`EnrichmentService` calls three downstream services (A, B, C) on a shared fixed pool. When service C degrades, the team notices calls to A and B also start failing.

**Code**
```java
private final ExecutorService pool = Executors.newFixedThreadPool(20);

public CompletableFuture<Void> enrich(Order o) {
    return CompletableFuture.allOf(
        CompletableFuture.runAsync(() -> callA(o), pool),
        CompletableFuture.runAsync(() -> callB(o), pool),
        CompletableFuture.runAsync(() -> callC(o), pool)); // C can hang for 30s
}
```

**Your Task**
Review why a single slow dependency affects the others.

---

## Topic 24 — Graceful executor shutdown

### Exercise 1 — shutdownNow and assume done

**Scenario**
`ExportManager.stop` is called on application shutdown. In-flight exports must complete so files aren't left half-written.

**Code**
```java
private final ExecutorService pool = Executors.newFixedThreadPool(4);

public void stop() {
    pool.shutdownNow(); // stop the pool
    log.info("All exports finished");
}
```

**Your Task**
Review the shutdown behavior against the requirement.

---

### Exercise 2 — No await on context close

**Scenario**
`NotificationDispatcher` submits sends to a pool. On redeploy, users report missing notifications that were "in flight."

**Code**
```java
@Service
public class NotificationDispatcher {
    private final ExecutorService pool = Executors.newFixedThreadPool(8);

    public void dispatch(Notification n) {
        pool.submit(() -> send(n));
    }
    // no @PreDestroy / shutdown logic anywhere
}
```

**Your Task**
Review the lifecycle handling on shutdown/redeploy.

---

## Topic 25 — Task cancellation

### Exercise 1 — Cancel that does nothing

**Scenario**
`SearchService.searchWithDeadline` cancels a long search when it exceeds a deadline.

**Code**
```java
public List<Hit> searchWithDeadline(Query q) throws Exception {
    Future<List<Hit>> f = executor.submit(() -> bruteForceSearch(q));
    try {
        return f.get(2, TimeUnit.SECONDS);
    } catch (TimeoutException e) {
        f.cancel(true); // ask it to stop
        return List.of();
    }
}

// the task:
private List<Hit> bruteForceSearch(Query q) {
    List<Hit> hits = new ArrayList<>();
    for (Document d : hugeCorpus) {          // long CPU loop
        if (matches(d, q)) hits.add(toHit(d));
    }
    return hits;
}
```

**Your Task**
Review whether `cancel(true)` actually stops the work.

---

### Exercise 2 — Swallowed interrupt during blocking wait

**Scenario**
`QueueWorker` blocks on a queue and is expected to stop promptly when cancelled/interrupted.

**Code**
```java
public void run() {
    while (true) {
        try {
            Task t = queue.take();  // blocking, interruptible
            handle(t);
        } catch (InterruptedException e) {
            // ignore and keep going
        }
    }
}
```

**Your Task**
Review whether this worker can be cancelled.

---

## Topic 26 — Timeout handling

### Exercise 1 — Timed get without cancel

**Scenario**
`ReportService.render` gives up on a slow render after 5s and returns a fallback, then handles the next request.

**Code**
```java
public Report render(ReportRequest r) {
    Future<Report> f = executor.submit(() -> heavyRender(r));
    try {
        return f.get(5, TimeUnit.SECONDS);
    } catch (TimeoutException e) {
        return Report.fallback(); // f is left running
    } catch (Exception e) {
        return Report.fallback();
    }
}
```

**Your Task**
Review what happens to the timed-out task under sustained load.

---

### Exercise 2 — Retry with per-attempt timeout only

**Scenario**
`PaymentClient.charge` retries a downstream call up to 3 times, each attempt with a 30s timeout. The overall request has a 20s SLA to the user.

**Code**
```java
public ChargeResult charge(Charge c) throws Exception {
    for (int attempt = 0; attempt < 3; attempt++) {
        Future<ChargeResult> f = executor.submit(() -> gateway.charge(c));
        try {
            return f.get(30, TimeUnit.SECONDS);
        } catch (TimeoutException e) {
            f.cancel(true);
            // retry
        }
    }
    throw new TimeoutException("charge failed after retries");
}
```

**Your Task**
Review the timeout strategy against the SLA.

---

## Topic 27 — Common ForkJoinPool

### Exercise 1 — Blocking the common pool

**Scenario**
`BulkLoader.load` processes a large list, calling a blocking DB fetch for each element, parallelized with a parallel stream.

**Code**
```java
public List<Entity> load(List<Long> ids) {
    return ids.parallelStream()
              .map(id -> blockingDbClient.fetch(id)) // blocking JDBC, ~100ms each
              .collect(Collectors.toList());
}
```

**Your Task**
Review the effect on the common ForkJoinPool and the wider application.

---

### Exercise 2 — Parallelism in a constrained container

**Scenario**
`Aggregator` relies on parallel streams to speed up a CPU-bound aggregation. It's deployed in Kubernetes pods each limited to 1 CPU.

**Code**
```java
public long aggregate(List<long[]> partitions) {
    // "runs in parallel across cores for speed"
    return partitions.parallelStream()
                     .mapToLong(this::sumPartition)
                     .sum();
}
```

**Your Task**
Review whether this delivers the parallel speedup the comment claims, given the deployment.

---

## Topic 28 — Parallel streams

### Exercise 1 — Non-associative reduction

**Scenario**
`ScoreService.combined` reduces per-stage scores into a single figure. It was switched to a parallel stream for speed.

**Code**
```java
public double combined(List<Double> stageScores) {
    return stageScores.parallelStream()
                      .reduce(100.0, (acc, s) -> acc - s); // subtract each stage penalty
}
```

**Your Task**
Review whether the parallel reduction produces a correct, deterministic result.

---

### Exercise 2 — Parallel sum of a large array

**Scenario**
`StatisticsService.total` sums a very large `List<Long>` (tens of millions) of independent values. It uses a parallel stream. A reviewer left a note.

**Code**
```java
public long total(List<Long> values) {
    // not sure this is thread-safe?
    return values.parallelStream()
                 .mapToLong(Long::longValue)
                 .sum();
}
```

**Your Task**
Review whether the parallel stream is correct and appropriate here. State clearly whether the reviewer's note points to a real problem.

---

---

# ANSWER KEY

## Topic 1 · Exercise 1 — Kicking off background work
**Findings**
- 🔴 **DEFINITE BUG — `run()` vs `start()`.** `worker.run()` executes the Runnable **on the calling (request) thread** synchronously — no new thread is started. The request blocks for the whole import; the "started in background" log is a lie.

**Fix:** `worker.start()` — but better, submit to a managed executor rather than a raw thread (see Topic 20).
**What a strong senior would say:** *"`run()` just calls the method on the current thread — you need `start()` to actually spawn one. As written the request blocks for the entire import. And I wouldn't hand-roll a raw Thread for this anyway; I'd submit to a bounded managed executor."*
**Interviewer is testing:** the most basic threading fundamental — run vs start.
**Mention first:** run-doesn't-start-a-thread.

## Topic 1 · Exercise 2 — One thread per request
**Findings**
- 🔴 **DEFINITE BUG — Unbounded thread creation.** `new Thread().start()` per webhook at thousands/sec creates thousands of unmanaged threads → memory (each ~1MB stack), scheduler thrash, and eventually `OutOfMemoryError: unable to create native thread`. No backpressure, no reuse.
- 🟠 No `UncaughtExceptionHandler` — a failing task dies silently.

**Fix:** a bounded `ThreadPoolExecutor` (or Spring `@Async` with a configured executor) with a bounded queue + rejection policy; consider a queue/broker for true spikes.
**What a strong senior would say:** *"Thread-per-request is a classic capacity bug — at thousands per second we exhaust native threads and OOM, with no backpressure. I'd use a bounded pool so concurrency is capped and excess work either queues or is shed via a rejection policy. For genuine spikes I'd push webhooks onto a queue and consume at a controlled rate."*
**Interviewer is testing:** why unbounded thread creation fails; bounded pools + backpressure.
**Mention first:** unbounded threads → OOM.

## Topic 2 · Exercise 1 — Fire-and-forget task
**Findings**
- 🟠 **DEFINITE BUG — Lost exception.** A `Runnable` thrown on a raw `Thread` goes to the thread's `UncaughtExceptionHandler` (default: prints to stderr, easily lost in prod) and nothing else observes it — no metric, no alert, no retry. The team's "we'll see it" expectation fails.

**Fix:** submit a `Callable`/task to an executor and inspect the `Future`, or set a proper `UncaughtExceptionHandler`/use `CompletableFuture` with `exceptionally`, and emit a metric/log on failure.
**What a strong senior would say:** *"An exception in a raw thread's Runnable just hits the default uncaught handler and effectively vanishes in production. If failures matter, I'd run this on an executor and check the Future (or a CompletableFuture with exceptionally), logging and emitting a metric on failure so it's actually visible."*
**Interviewer is testing:** how exceptions escape raw threads and how to make them observable.
**Mention first:** the silently-lost exception.

## Topic 2 · Exercise 2 — Reused runnable with state
**Findings**
- 🔴 **DEFINITE BUG — Shared mutable state.** One `BatchCounter` instance is run by 4 threads, all iterating the **same** `shard` and mutating the **same** `processed` field. `processed++` races (lost updates), and all four process the identical shard (not four shards) — logic and counting both wrong.
- 🟡 Design: the intent (parallel shards) requires *separate* instances per shard, or a stateless task + external aggregation.

**Fix:** one `BatchCounter` per shard (`new BatchCounter(shardI)`), and aggregate counts via `AtomicLong`/returned values; don't share a mutable Runnable.
**What a strong senior would say:** *"Sharing one Runnable instance across four threads means they all process the same shard and race on `processed` — so both the work and the tally are wrong. I'd create one task per shard and aggregate the counts through an AtomicLong or by collecting returned results, keeping the task free of shared mutable state."*
**Interviewer is testing:** unsafe sharing of a stateful Runnable.
**Mention first:** shared instance → same shard + counter race.

## Topic 3 · Exercise 1 — Result never retrieved
**Findings**
- 🟠 **DEFINITE BUG — Swallowed Callable exception.** `submit` captures any exception in the `Future`, but the `Future` is discarded, so a `PricingException` is never thrown or logged — exactly the "silently doesn't update, no errors" symptom. The `Callable` return type doesn't help if nobody calls `get()`.

**Fix:** keep the `Future` and call `get()` (handling the `ExecutionException`), or use `CompletableFuture.supplyAsync(...).exceptionally(...)`, logging/metric on failure.
**What a strong senior would say:** *"`submit` swallows the exception into a Future that we throw away, so pricing failures disappear — which matches the symptom exactly. I'd retain the Future and check it, or switch to a CompletableFuture with an `exceptionally` handler that logs and records a metric, so a failed price update is visible."*
**Interviewer is testing:** where submitted-task exceptions go and how to surface them.
**Mention first:** discarded Future hides the exception.

## Topic 3 · Exercise 2 — Parallel calls that aren't parallel
**Findings**
- 🟠 **DEFINITE BUG — Accidental serialization.** Calling `f.get()` immediately after each `submit`, inside the loop, waits for each fetch before submitting the next — so the fetches run strictly sequentially. Total latency = sum of all fetches, not the max. No concurrency achieved.

**Fix:** submit all first, collect the `Future`s, then `get()` each:
```java
List<Future<Data>> futures = sources.stream().map(s -> executor.submit(() -> fetch(s))).toList();
for (Future<Data> f : futures) all.add(f.get());
```
(or `CompletableFuture.allOf`).
**What a strong senior would say:** *"The `get()` is inside the submit loop, so each fetch completes before the next starts — it's sequential despite the executor. I'd submit all tasks first, collect the futures, then join them, so latency is the slowest call rather than the sum. `CompletableFuture.allOf` expresses that cleanly."*
**Interviewer is testing:** the submit-then-get ordering that defeats parallelism.
**Mention first:** in-loop get serializes everything.

## Topic 4 · Exercise 1 — The "async" executor
**Findings**
- 🟠 **DEFINITE BUG — Synchronous executor.** `Runnable::run` is an `Executor` that runs the command **inline on the caller's thread**. `sendAsync` therefore blocks the request thread on SMTP — no async at all. (Legitimate for tests, wrong here.)

**Fix:** provide a real pool-backed executor (`ThreadPoolExecutor` / Spring `ThreadPoolTaskExecutor`) as the bean.
**What a strong senior would say:** *"`Runnable::run` is a same-thread executor — it just calls the task inline, so `sendAsync` runs SMTP on the request thread. The abstraction hides it, which is why it's sneaky. I'd wire a real bounded pool executor as the bean; the direct executor is fine for unit tests but not production."*
**Interviewer is testing:** understanding that `Executor` is an interface and a direct executor isn't async.
**Mention first:** the same-thread executor.

## Topic 4 · Exercise 2 — Cached pool for slow calls
**Findings**
- 🔴 **DEFINITE BUG — Unbounded thread growth.** `newCachedThreadPool` creates a new thread for every task when all existing threads are busy, with a `SynchronousQueue` (no queueing). With ~2s tasks and a burst of thousands, it spawns thousands of threads → native-thread OOM / severe context-switch thrash. It also amplifies load on the already-slow external service.

**Fix:** a bounded `ThreadPoolExecutor` (fixed max threads + bounded queue + rejection/backpressure); size for the external service's capacity; add timeouts/circuit breaker.
**What a strong senior would say:** *"Cached pools are dangerous for slow tasks — they add a thread per task under load with no cap, so a burst of thousands of 2-second calls spawns thousands of threads and OOMs, while hammering the slow dependency harder. I'd use a bounded pool sized to what the downstream can take, with a bounded queue and a rejection policy, plus a timeout and breaker."*
**Interviewer is testing:** the newCachedThreadPool failure mode for long/slow tasks.
**Mention first:** unbounded thread explosion.

## Topic 5 · Exercise 1 — Submitted task that vanishes
**Findings**
- 🟠 **DEFINITE BUG — Swallowed failure on submit.** Like Topic 3 Ex1 but for a `Runnable`: `submit` puts any exception into the ignored `Future`, so a DB hiccup during `save` throws, is captured, and is never surfaced — the audit record is lost silently (matches "records go missing with no trace").

**Fix:** don't fire-and-forget critical writes — check the Future / use `CompletableFuture.exceptionally`, log + metric on failure, and consider a retry or durable queue for audit durability.
**What a strong senior would say:** *"`submit` swallows the save failure into a Future nobody reads, so a transient DB error just drops the audit record silently. For something that must not be lost, I'd surface failures (check the future, log, metric), and I'd back audit writes with a retry or a durable queue rather than a best-effort thread pool."*
**Interviewer is testing:** submit-swallows-exceptions again, now framed as data loss + durability.
**Mention first:** silent drop of the audit record.

## Topic 5 · Exercise 2 — Nested submit on a single-thread pool
**Findings**
- 🔴 **DEFINITE BUG — Thread-pool starvation deadlock.** The outer task occupies the single worker thread, then submits `validate` to the **same** single-thread pool and blocks on `valid.get()`. The validation task can never run (no free thread) → permanent deadlock. Every `process` call hangs.

**Fix:** don't block on a task submitted to the same bounded pool. Either run validation inline (`validate(o)` directly), or use a separate executor for the nested work, or restructure to avoid the dependency.
**What a strong senior would say:** *"This is a pool-starvation deadlock: the only thread is busy running the outer task and waiting on a sub-task it submitted to the same single-thread pool, which can't get a thread — so it hangs forever. The fix is to not submit-and-block within the same bounded pool; validate inline or use a distinct executor. This class of deadlock also bites bounded pools, not just single-thread ones."*
**Interviewer is testing:** thread-pool starvation deadlock via self-dependency — advanced.
**Mention first:** the starvation deadlock.

## Topic 6 · Exercise 1 — Waiting forever
**Findings**
- 🟠 **DEFINITE BUG — Unbounded `get()`.** `f.get()` with no timeout blocks the request thread indefinitely when the engine hangs; enough hung requests exhaust the request-thread pool → cascading outage.

**Fix:** `f.get(timeout, unit)`; on `TimeoutException`, cancel the future and return a fallback/error (and note the cancel caveat — Topic 26).
**What a strong senior would say:** *"An unbounded `get()` means a hung engine ties up the request thread forever, and enough of those take down the service. I'd use a bounded `get` with a timeout, cancel on timeout, and return a fallback or error. Timeouts are the first line of resilience for anything that can hang."*
**Interviewer is testing:** always-timeout-your-gets; cascading failure.
**Mention first:** no-timeout blocking.

## Topic 6 · Exercise 2 — Fire-and-forget futures
**Findings**
- 🟠 **DEFINITE BUG — False completion claim + swallowed failures.** The method returns as soon as tasks are *submitted*, not *completed*, so the caller's "all processed" is wrong — items may still be running or may have failed (Futures discarded → exceptions lost). Under failure, some items silently never processed.

**Fix:** collect the Futures and wait (`for (Future f: futures) f.get();`) or `CompletableFuture.allOf(...).join()`, handling exceptions; only then report completion. If truly async, report "accepted," not "processed."
**What a strong senior would say:** *"Returning after submission isn't the same as completion — the caller reports 'all processed' while tasks may still be running or already failed silently. I'd collect the futures and await them, handling failures, before claiming success; or if this is meant to be async, the honest response is 'accepted,' with completion tracked separately."*
**Interviewer is testing:** submission vs completion semantics; discarded futures.
**Mention first:** completion claimed before tasks finish.

## Topic 7 · Exercise 1 — Lost exception in a chain
**Findings**
- 🟠 **DEFINITE BUG — No exceptionally/handle.** If `supplyAsync`'s supplier throws, the exception completes the future exceptionally, `thenAccept` is skipped, and since nothing observes the stage, the failure is swallowed (no log, cache stays empty — matches the symptom). Dangling CompletableFutures with no terminal error handling silently drop exceptions.

**Fix:** add `.exceptionally(ex -> { log.error("warm failed", ex); return null; })` or `.whenComplete((v, ex) -> ...)`; consider returning the future so callers can react.
**What a strong senior would say:** *"There's no exceptionally or whenComplete, so a load failure completes the future exceptionally, skips thenAccept, and is never observed — the exception is lost and the cache silently stays empty. I'd add an exceptionally/whenComplete that logs and emits a metric, and I'd return the future rather than firing and forgetting."*
**Interviewer is testing:** CompletableFuture exception propagation and terminal handling.
**Mention first:** swallowed exceptional completion.

## Topic 7 · Exercise 2 — Blocking the async stage
**Findings**
- 🟠 **DEFINITE BUG — Blocking on the common pool.** `supplyAsync` without an explicit executor runs on the **common ForkJoinPool**; both suppliers do blocking HTTP there. Under load this starves the shared common pool (used by parallel streams and other async work across the JVM), and the pool's small size limits real concurrency.
- 🟢 `join()` at the end blocks the caller — acceptable here since it's the aggregation point, but the blocking work should be on a dedicated pool.

**Fix:** pass a dedicated bounded executor to each `supplyAsync(..., ioExecutor)` sized for I/O; keep blocking work off the common pool.
**What a strong senior would say:** *"`supplyAsync` with no executor uses the common ForkJoinPool, and we're doing blocking HTTP on it — that starves the pool the whole app shares for parallel work, and caps our concurrency at its small size. I'd supply a dedicated I/O executor to each async call so the blocking work is isolated and appropriately sized."*
**Interviewer is testing:** knowing the default executor for CompletableFuture and why blocking it is bad.
**Mention first:** blocking the common pool.

## Topic 8 · Exercise 1 — Locking on a changing reference
**Findings**
- 🔴 **DEFINITE BUG — Non-final, reassigned lock.** `lock` is reassigned in `replaceBuffer`. A thread in `write` synchronizes on the *old* lock object while another calls `replaceBuffer` and swaps to a *new* one — subsequent writers lock a different monitor, so two threads can be in the critical section simultaneously on `buffer`. Locks must be `final`.
- 🟠 `buffer` reassignment while writers hold the old lock also races on the buffer reference.

**Fix:** `private final Object lock = new Object();` (never reassign the lock); guard both `buffer` access and replacement under the same final lock.
**What a strong senior would say:** *"The lock isn't final and gets swapped in replaceBuffer, so different threads can end up synchronizing on different monitor objects — which means no mutual exclusion at all. The lock must be a final object that's never reassigned, and buffer replacement has to happen under that same lock."*
**Interviewer is testing:** locks must be final/stable; reassigning a monitor breaks exclusion.
**Mention first:** the reassigned lock.

## Topic 8 · Exercise 2 — Lazy singleton
**Findings**
- 🟠 **DEFINITE BUG — Double-checked locking without `volatile`.** `instance` isn't `volatile`, so the DCL pattern is broken: due to reordering, a thread can observe a **non-null but partially-constructed** `instance` and use it before its constructor's writes are visible. Classic broken DCL.

**Fix:** make `instance` `volatile` (correct DCL), or better, use the **initialization-on-demand holder idiom** (static nested holder class) or an `enum` singleton — both are simpler and correct without volatile.
**What a strong senior would say:** *"This is double-checked locking without a volatile field, which is the textbook-broken version — a thread can see a non-null reference to a half-initialized object because the construction and the reference write can be reordered. I'd mark `instance` volatile, but honestly I'd switch to the holder-class idiom, which gives lazy, thread-safe init with no locking or volatile to get wrong."*
**Interviewer is testing:** the broken-DCL classic and the safer idioms.
**Mention first:** DCL missing volatile → partially-constructed object.

## Topic 9 · Exercise 1 — Volatile counter
**Findings**
- 🟠 **DEFINITE BUG — Volatile ≠ atomic.** `volatile` guarantees visibility of reads/writes but **not** atomicity of `count++` (read-modify-write), so concurrent increments still lose updates. The count under-reports.

**Fix:** `AtomicLong.incrementAndGet()`, or `LongAdder` for a hot counter (better under contention).
**What a strong senior would say:** *"`volatile` fixes visibility but not atomicity — `count++` is still read-modify-write, so we lose increments under concurrency. I'd use `AtomicLong`, or `LongAdder` if it's very hot, since it reduces contention by striping."*
**Interviewer is testing:** the visibility-vs-atomicity distinction — the single most common volatile misconception.
**Mention first:** volatile doesn't make ++ atomic.

## Topic 9 · Exercise 2 — Volatile reference to a mutable holder
**Findings**
- 🟠 **DEFINITE BUG — Volatile publishes the reference, not deep mutations.** `config` is `volatile`, but `reload` mutates the **existing** object's map (`clear`/`putAll`) without changing the reference. Volatile provides no happens-before for those element writes, so readers can see a torn/partial map, and `clear`+`putAll` on a plain `HashMap` concurrently with readers is itself unsafe (can throw / corrupt).

**Fix:** publish a **new immutable** config on reload and assign the volatile reference (`config = new Config(Map.copyOf(newValues))`); readers then safely see either the old or the new whole object via the volatile read.
**What a strong senior would say:** *"Volatile only gives a happens-before on the reference itself, not on mutations to the object it points to — and here we mutate the existing map in place, so readers can see a half-updated or corrupted map, and the concurrent clear/putAll on a HashMap is unsafe on its own. The clean pattern is copy-on-write: build a new immutable Config and swap the volatile reference atomically."*
**Interviewer is testing:** the depth of volatile's guarantee; copy-on-write publication.
**Mention first:** in-place mutation defeats volatile.

## Topic 10 · Exercise 1 — Two atomics, one invariant
**Findings**
- 🟠 **DEFINITE BUG — Compound invariant across two atomics.** Each atomic is individually safe, but `total` and `count` are updated in two separate steps, so `average()` can read `total` after an update and `count` before it (or vice-versa), yielding an inconsistent/wrong average — the pair isn't atomic together.

**Fix:** guard both under one lock for the read/update, or hold them in a single immutable snapshot updated atomically (e.g. an `AtomicReference<long[2]>`/record updated via `updateAndGet`), or accept the approximation if exact consistency isn't required (clarify).
**What a strong senior would say:** *"Both atomics are individually correct, but the invariant spans the two — `average` can observe an updated total against a stale count, giving a wrong value. If we need a consistent snapshot I'd update them together under a lock or as a single AtomicReference to an immutable (total,count) pair. If a slightly-off running average is acceptable, I'd document that instead of adding synchronization."*
**Interviewer is testing:** that combining atomics doesn't make a multi-variable invariant atomic.
**Mention first:** cross-field inconsistency.

## Topic 10 · Exercise 2 — Atomic used non-atomically
**Findings**
- 🔴 **DEFINITE BUG — get/set instead of atomic increment.** `get()` then `set(current+1)` is a read-modify-write across two calls — two threads can read the same `current` and both return it, producing **duplicate IDs**. The `AtomicInteger` type buys nothing when used this way.

**Fix:** `return seq.getAndIncrement();` (single atomic op).
**What a strong senior would say:** *"Using get then set defeats the whole point of AtomicInteger — it's a two-step read-modify-write, so two threads can hand out the same id. `getAndIncrement()` does it as one atomic operation and guarantees uniqueness."*
**Interviewer is testing:** using atomic combinators, not get/set.
**Mention first:** duplicate IDs from get/set.

## Topic 11 · Exercise 1 — Lock without finally
**Findings**
- 🔴 **DEFINITE BUG — Lock leaked on exception.** When stock is insufficient, the method throws **while holding the lock** and `unlock()` is never reached → the lock is never released, so every subsequent caller for any SKU blocks forever (effective deadlock/hang). Explicit locks must always unlock in `finally`.

**Fix:**
```java
lock.lock();
try {
    Stock s = stock.get(sku);
    if (s.available() < qty) throw new InsufficientStockException(sku);
    s.reduce(qty);
} finally {
    lock.unlock();
}
```
**What a strong senior would say:** *"The exception path throws before `unlock`, so the lock is never released and the whole service wedges on the next call. Explicit locks must be released in a finally that immediately follows the `lock()` — that's the non-negotiable idiom. This one's critical: it's a latent full hang."*
**Interviewer is testing:** the lock/try-finally invariant.
**Mention first:** lock never released on the throw path.

## Topic 11 · Exercise 2 — Writes locked, reads not
**Findings**
- 🟠 **DEFINITE BUG — Asymmetric locking / visibility.** Writes are guarded, but `current()` reads `position` with no lock and the field isn't `volatile`, so readers may see a **stale** reference (no happens-before with the writer). Since `Position` is replaced (not mutated) the read is at least not torn, but visibility is still not guaranteed.

**Fix:** make `position` `volatile` (sufficient here because it's an immutable object swapped by reference), or read under the same lock. Volatile is the lighter correct choice given immutable Position.
**What a strong senior would say:** *"Locking only the writes leaves readers with no happens-before, so they can see a stale position indefinitely. Because Position is immutable and we swap the reference, marking the field `volatile` is enough to publish updates safely — cheaper than locking the read. If Position were mutated in place, I'd need the lock on reads too."*
**Interviewer is testing:** that safe publication needs *both* sides synchronized (or volatile); immutable-object + volatile pattern.
**Mention first:** unsynchronized read → stale value.

## Topic 12 · Exercise 1 — Read-to-write upgrade
**Findings**
- 🔴 **DEFINITE BUG — Lock upgrade deadlock.** `ReentrantReadWriteLock` does **not** support upgrading a held read lock to a write lock — acquiring the write lock while holding the read lock blocks forever (the write lock waits for all readers, including this thread, to release). If two threads try to upgrade simultaneously, guaranteed deadlock.

**Fix:** release the read lock before acquiring the write lock, then re-check under the write lock (double-check on the write side):
```java
rw.readLock().lock();
try { v = cache.get(key); } finally { rw.readLock().unlock(); }
if (v == null) {
    rw.writeLock().lock();
    try { v = cache.get(key); if (v == null) { v = load(key); cache.put(key, v); } }
    finally { rw.writeLock().unlock(); }
}
```
(Downgrading write→read is allowed; upgrading read→write is not.)
**What a strong senior would say:** *"ReentrantReadWriteLock doesn't allow upgrading read to write — the write acquisition waits for all readers to release, including the very thread holding the read, so it self-deadlocks. I'd drop the read lock, take the write lock, and re-check under it. Downgrade is supported, upgrade isn't — that asymmetry trips people up."*
**Interviewer is testing:** the no-upgrade rule of ReadWriteLock — advanced.
**Mention first:** upgrade deadlock.

## Topic 12 · Exercise 2 — Ignored tryLock result
**Findings**
- 🔴 **DEFINITE BUG — tryLock return value ignored.** `tryLock()` returns a `boolean` indicating whether the lock was acquired; ignoring it means the code enters the critical section **even when it didn't get the lock** (so "skip if busy" doesn't happen — concurrent runs proceed). Worse, the `finally { unlock() }` then calls `unlock()` on a lock this thread may not hold → `IllegalMonitorStateException`.

**Fix:**
```java
if (lock.tryLock()) {
    try { job.execute(); } finally { lock.unlock(); }
} else {
    log.debug("Already running, skipping");
}
```
**What a strong senior would say:** *"The tryLock result is discarded, so the 'skip if another thread has it' logic never happens — everyone runs — and the finally unlocks a lock we might not own, throwing IllegalMonitorStateException. I'd branch on the boolean and only enter (and unlock) if we actually acquired it."*
**Interviewer is testing:** tryLock semantics and unlock-only-if-acquired.
**Mention first:** ignored boolean → both the logic bug and the illegal unlock.

## Topic 13 · Exercise 1 — Heavy work inside compute
**Findings**
- 🟠 **DEFINITE BUG — Long/blocking work inside `computeIfAbsent`.** The mapping function runs while holding the bucket lock for that key. A slow remote fetch holds that lock, blocking other threads that hash to the same bin and hurting map-wide throughput; the docs explicitly say the function should be **short and must not attempt to update the same map**. It can also stack up many concurrent identical fetches for *different* keys (no dedup benefit across keys).

**Fix:** don't do slow I/O inside `computeIfAbsent`. Fetch outside and `putIfAbsent`, or use a caching library (Caffeine) with async loading and proper single-flight semantics; guard against the thundering-herd separately.
**What a strong senior would say:** *"Doing a network fetch inside `computeIfAbsent` holds the per-bin lock for the duration, which serializes and blocks other operations on that bin and violates the contract that the function be short and non-reentrant. I'd move the fetch outside and use `putIfAbsent`, or better use Caffeine's async loading cache, which handles single-flight and eviction correctly."*
**Interviewer is testing:** the CHM computeIfAbsent locking contract — subtle senior knowledge.
**Mention first:** blocking work under the bin lock.

## Topic 13 · Exercise 2 — Null values
**Findings**
- 🔴 **DEFINITE BUG — CHM forbids nulls.** `ConcurrentHashMap` does not permit null keys or values; when the user isn't found, `orElse(null)` makes the mapping function return `null`, which throws `NullPointerException` inside `computeIfAbsent`. (Also, even if allowed, caching "absent" as null defeats negative caching semantics.)

**Fix:** don't store null — either don't cache misses, or cache an `Optional<User>`/sentinel: `cache.computeIfAbsent(id, k -> userRepository.findById(k).orElse(NOT_FOUND))` or store `Optional` values. Consider Caffeine for proper negative caching.
**What a strong senior would say:** *"ConcurrentHashMap disallows null values, so a not-found user makes the mapping return null and throws NPE — this crashes rather than caches a miss. If we want to cache misses I'd store an `Optional` or a sentinel value; otherwise skip caching absences. This is a common trap migrating from HashMap, which does allow nulls."*
**Interviewer is testing:** CHM null-prohibition and negative-caching design.
**Mention first:** NPE on null value.

## Topic 14 · Exercise 1 — Lazy field init
**Findings**
- 🟠 **DEFINITE BUG — Race on lazy init.** Multiple threads can see `compiled == null` and each run the expensive `compile()`, and worse, a thread may read a partially/unsafely published `compiled` written by another (no happens-before). Redundant work plus a possible visibility bug.
- 🟡 If `Template` is immutable, redundant compiles are wasteful but harmless in result; the visibility gap is the real correctness issue.

**Fix:** initialize eagerly in the constructor if always needed; or use the **holder idiom** / a `volatile` field with double-checked locking; or precompute once at startup. For "compile once," an `AtomicReference` with `updateAndGet` or holder class is clean.
**What a strong senior would say:** *"Unsynchronized lazy init lets several threads compile in parallel and, more importantly, one thread can see a non-null but not-safely-published template. If it's always needed I'd just compile eagerly; if it must be lazy, I'd use the holder idiom or a volatile double-checked field so it's compiled once and safely published."*
**Interviewer is testing:** lazy-init race + safe publication; choosing the right idiom.
**Mention first:** duplicate compile + unsafe publication.

## Topic 14 · Exercise 2 — Reserve then commit
**Findings**
- 🔴 **DEFINITE BUG — Check-then-act (double booking).** Two users can both read `FREE`, both set `BOOKED`, and both get `true` — the seat is sold twice. TOCTOU race with direct financial/customer impact.
- 🟡 In a multi-instance deployment, in-JVM synchronization wouldn't even suffice — needs DB-level enforcement.

**Fix:** make the check-and-set atomic: a conditional DB `UPDATE ... SET status='BOOKED', booked_by=? WHERE seat_id=? AND status='FREE'` and treat "0 rows updated" as "already taken" — atomic and correct across instances. In-memory, a per-seat lock or `AtomicReference`/`compareAndSet` on status; but the DB approach is the right one for a distributed system.
**What a strong senior would say:** *"This is a check-then-act double-booking race — two requests both see FREE and both book. Since we run multiple instances, in-JVM locking isn't enough; I'd push the atomicity to the database with a conditional update guarded by `status = 'FREE'` and treat zero rows affected as 'already booked.' That's atomic and correct across pods."*
**Interviewer is testing:** TOCTOU + recognizing distributed atomicity needs the DB.
**Mention first:** double-booking race, then the multi-instance point.

## Topic 15 · Exercise 1 — Stop flag
**Findings**
- 🟠 **DEFINITE BUG — Visibility of the stop flag.** `running` isn't `volatile`; the worker thread may cache it and **never observe** `stop()`'s write → the loop runs forever (or until unrelated memory effects happen to flush it). Pure visibility bug.

**Fix:** `private volatile boolean running = true;` (or an `AtomicBoolean`).
**What a strong senior would say:** *"Without volatile there's no happens-before between `stop()` and the loop's read, so the worker can cache `running=true` and never see the change — the loop won't terminate. Marking it volatile establishes visibility. This is the canonical case where volatile alone is exactly the right fix."*
**Interviewer is testing:** the classic non-volatile stop-flag visibility bug.
**Mention first:** missing volatile on the flag.

## Topic 15 · Exercise 2 — Publishing a built object
**Findings**
- 🟠 **DEFINITE BUG — Unsafe publication.** `config` is neither `volatile` nor `final`, and it's written by the background thread and read by others. Without a happens-before edge, readers can see `null`, or a **partially-constructed** `Config` (fields `timeout`/`retries` not yet visible even though the reference is). Unsafe publication of a mutable object.

**Fix:** make `config` `volatile` (publishes the fully-constructed object safely, since the writes happen-before the volatile write), or make `Config` immutable with `final` fields and publish via a `volatile`/`final` reference, or use a `CompletableFuture`/`CountDownLatch` for coordination.
**What a strong senior would say:** *"This is unsafe publication — the reference is written by one thread and read by others with no happens-before, so readers can see null or a Config whose fields aren't visible yet. I'd make the field volatile so the object is safely published after construction, and ideally make Config immutable with final fields so it's safe to share regardless."*
**Interviewer is testing:** safe publication and partially-constructed-object visibility.
**Mention first:** unsafe publication.

## Topic 16 · Exercise 1 — Bounded add
**Findings**
- 🟠 **DEFINITE BUG — Compound check-then-act on a synchronized list.** `size()` and `add()` are each atomic, but the sequence isn't; N threads can all see `size() < MAX` and all `add`, overshooting `MAX`. `Collections.synchronizedList` protects individual calls, not the pair.

**Fix:** synchronize the whole check-and-add on the list's monitor (`synchronized (conns) { if (conns.size() < MAX) conns.add(c); }`), or use a bounded structure/semaphore (`Semaphore(MAX)` acquired before add).
**What a strong senior would say:** *"The bound leaks because `size()` and `add()` are individually atomic but not together — many threads pass the size check and all add, blowing past MAX. I'd make the check-and-add one atomic block synchronized on the list, or gate additions with a `Semaphore(MAX)`, which is cleaner for a capacity limit."*
**Interviewer is testing:** compound-operation atomicity on synchronized collections.
**Mention first:** the bound overshoot.

## Topic 16 · Exercise 2 — Non-volatile long
**Findings**
- 🟡 **POTENTIAL BUG — Non-atomic 64-bit access + visibility.** The JLS permits non-`volatile` `long`/`double` reads/writes to be split into two 32-bit halves, so on a 32-bit JVM a reader can observe a **torn** value (high half of one write, low half of another). And with no volatile there's no visibility guarantee even on 64-bit JVMs — readers may see stale prices. For a price feed both matter.

**Fix:** `private volatile long priceCents;` (volatile longs are guaranteed atomic and visible), or `AtomicLong`.
**What a strong senior would say:** *"A plain long can be read/written in two halves per the JLS, so on a 32-bit VM a reader can see a torn price, and without volatile there's no visibility guarantee anywhere. For a price others read, I'd make it `volatile` — that guarantees 64-bit atomicity and visibility — or use AtomicLong. On modern 64-bit VMs tearing is unlikely in practice, but I wouldn't rely on it, and the visibility issue stands regardless."*
**Interviewer is testing:** the 64-bit non-atomicity rule + visibility; calibrated severity.
**Mention first:** torn/stale reads; volatile fix.

## Topic 17 · Exercise 1 — Transfer lock ordering
**Findings**
- 🔴 **DEFINITE BUG — Lock-ordering deadlock.** Concurrent `transfer(A,B)` and `transfer(B,A)` acquire `A` then `B` vs `B` then `A` — each holds one and waits for the other → deadlock. Any two account pairs transferred in opposite directions can hang.

**Fix:** impose a **global lock ordering** (e.g. lock the lower account id first), so all threads acquire in the same order:
```java
Account first = from.getId() < to.getId() ? from : to;
Account second = first == from ? to : from;
synchronized (first) { synchronized (second) { ... } }
```
(handle equal ids). Or use a single coarser lock / DB-level ordering for a distributed system.
**What a strong senior would say:** *"This is the textbook lock-ordering deadlock — A→B and B→A grab the two monitors in opposite orders and wait on each other. The fix is a consistent global order, e.g. always lock the lower account id first, so no cycle can form. In a multi-instance system I'd enforce ordering at the database instead."*
**Interviewer is testing:** recognizing and breaking lock-ordering deadlock.
**Mention first:** opposite-order acquisition.

## Topic 17 · Exercise 2 — Non-reentrant re-acquire
**Findings**
- 🔴 **DEFINITE BUG — Self-deadlock via non-reentrant Semaphore.** A `Semaphore(1)` is **not reentrant**: the thread that holds the single permit calls `helper()`, which tries to `acquire()` again — but the only permit is held by the same thread, so it blocks forever waiting on itself. `process()` deadlocks immediately.

**Fix:** use a reentrant lock (`ReentrantLock`, which allows the owner to re-acquire) if re-entry is intended; or restructure so the permit isn't re-acquired (don't guard `helper` separately, or pass a "lock already held" contract). Choose the primitive based on whether reentrancy is a requirement.
**What a strong senior would say:** *"A Semaphore isn't reentrant, so when the permit-holder calls `helper` and acquires again, it blocks on itself — instant self-deadlock. If re-entry is legitimate I'd use a ReentrantLock, which lets the owning thread re-acquire; otherwise I'd restructure so the inner method assumes the lock is already held rather than re-acquiring it."*
**Interviewer is testing:** reentrancy — Semaphore vs ReentrantLock.
**Mention first:** non-reentrant self-deadlock.

## Topic 18 · Exercise 1 — One slow task blocks the rest
**Findings**
- 🟠 **DEFINITE BUG — Starvation on a single-thread scheduler.** `newSingleThreadScheduledExecutor` runs all scheduled tasks on one thread. A multi-minute `generateHeavyReport` occupies that thread, so the 1-second `heartbeat` can't run and is **starved/delayed** for the whole report duration — heartbeats miss, possibly triggering false failure detection.

**Fix:** give the heartbeat its own scheduler (isolate critical periodic tasks), or use a multi-threaded `ScheduledThreadPoolExecutor` sized so long jobs don't block short ones; better, run the heavy report on a separate worker pool and keep the scheduler for triggering only.
**What a strong senior would say:** *"With a single scheduler thread, the multi-minute report monopolizes it and starves the one-second heartbeat, which could trip health checks. I'd isolate the heartbeat on its own scheduler, or use a scheduled pool with enough threads, and offload the heavy report to a separate executor so scheduling stays responsive."*
**Interviewer is testing:** starvation from a shared single-thread scheduler; task isolation.
**Mention first:** heartbeat starvation.

## Topic 18 · Exercise 2 — Readers starving the writer
**Findings**
- 🟡 **POTENTIAL BUG — Writer starvation under a non-fair ReadWriteLock.** With continuous overlapping readers and a non-fair `ReentrantReadWriteLock`, new readers can keep acquiring while a waiting writer never gets a turn — the price update is starved and prices go stale. Whether it actually starves depends on read intensity (it's a *potential* issue to reason about, not a guaranteed bug).

**Fix:** construct the lock in **fair** mode (`new ReentrantReadWriteLock(true)`) so a waiting writer eventually blocks new readers — trade-off: lower read throughput. Or use `StampedLock` (optimistic reads) for read-heavy workloads. Measure read intensity before deciding.
**What a strong senior would say:** *"Under a heavy, continuous read load a non-fair ReadWriteLock can starve the writer, so updates stall and reads serve stale prices. I'd consider a fair lock so waiting writers get in — at some read-throughput cost — or `StampedLock` for a read-heavy path. I'd want to confirm the read pattern actually causes starvation before trading away read performance, so this is a 'measure, then tune fairness' call."*
**Interviewer is testing:** writer starvation, fairness trade-offs, and calibrated ("it depends") reasoning.
**Good clarifying question:** *"How read-heavy is this, and how stale can prices get?"* — **Why it matters:** decides whether fairness/StampedLock is warranted.
**Mention first:** writer-starvation potential + fairness trade-off.

## Topic 19 · Exercise 1 — One lock for everything
**Findings**
- 🟡 **PERFORMANCE CONCERN — Coarse single lock (contention).** Every operation serializes on the one intrinsic lock, so at high request rates threads queue behind each other even for unrelated sessions — a throughput bottleneck. Correctness is fine; scalability isn't.
- 🟢 `touch` also does `get().touch()` under the lock — fine, but note NPE if absent.

**Fix:** use `ConcurrentHashMap<String, Session>` (lock-striped, concurrent reads/writes) so operations on different keys don't contend; per-session state mutations handled within the session or via atomic map ops.
**What a strong senior would say:** *"It's correct but the single lock serializes every session operation, so it won't scale on a hot path — unrelated sessions block each other. I'd back it with a ConcurrentHashMap, which stripes locking so different keys proceed concurrently, and handle per-session mutation atomically. I'd confirm with a quick benchmark that lock contention is the actual bottleneck first."*
**Interviewer is testing:** lock granularity and choosing concurrent collections over coarse locks.
**Mention first:** coarse-lock contention.

## Topic 19 · Exercise 2 — Shared Random
**Findings**
- 🟡 **PERFORMANCE CONCERN — Contention on shared `Random`.** `Random` is thread-safe but uses a single `AtomicLong` seed updated via CAS on every call; under heavy multi-thread use this becomes a contention hotspot (CAS retries) that serializes token generation. Correctness is fine.

**Fix:** `ThreadLocalRandom.current().nextBytes(...)` (per-thread state, no contention). If cryptographic strength is required (tokens!), use a `SecureRandom` per thread or a pool — note the scenario said randomness quality is acceptable, so `ThreadLocalRandom` fits the contention question, but for security tokens I'd flag that `Random` is not cryptographically secure.
**What a strong senior would say:** *"A shared `Random` is thread-safe but its single CAS-updated seed becomes a contention point on a hot path, effectively serializing token generation. `ThreadLocalRandom` removes the sharing. Separately — and outside the scope you set — `Random` isn't cryptographically secure, so for real tokens I'd want SecureRandom regardless."*
**Interviewer is testing:** shared-Random contention and ThreadLocalRandom; bonus security awareness.
**Mention first:** contention on the shared seed.

## Topic 20 · Exercise 1 — Fixed pool, unbounded queue
**Findings**
- 🟠 **DEFINITE BUG — Unbounded queue → OOM under overload.** `newFixedThreadPool` uses an **unbounded** `LinkedBlockingQueue`, so during sustained overload tasks pile up in the queue without bound → growing memory and latency, eventually OOM. No backpressure or shedding.

**Fix:** a `ThreadPoolExecutor` with a **bounded** queue + a rejection policy (`CallerRunsPolicy` for backpressure, or reject+shed with metrics); size the queue and pool to the sustainable rate. For real spikes, front with a broker.
**What a strong senior would say:** *"`newFixedThreadPool` hides an unbounded queue, so under sustained overload we accumulate tasks until we OOM instead of pushing back. I'd use an explicit ThreadPoolExecutor with a bounded queue and a rejection policy — CallerRuns to slow producers, or shed with a metric — so overload degrades gracefully rather than fatally."*
**Interviewer is testing:** the hidden-unbounded-queue trap and backpressure.
**Mention first:** unbounded queue OOM.

## Topic 20 · Exercise 2 — Cached pool for long tasks
**Findings**
- 🟠 **DEFINITE BUG — Thread explosion.** `newCachedThreadPool` spawns a new thread per task when none are idle; hundreds of multi-minute backups submitted together create hundreds of concurrent threads → native-thread OOM / thrash, and possibly overwhelms the backup target. Cached pools suit many *short* tasks, not few *long* ones.

**Fix:** a fixed/bounded pool sized to how many concurrent backups the system and target can sustain, with a bounded queue for the rest.
**What a strong senior would say:** *"Cached pools are for bursts of short tasks — for hundreds of minutes-long backups they'll create hundreds of threads at once and exhaust resources while hammering the backup target. I'd use a bounded pool sized to the safe concurrency level and queue the remainder."*
**Interviewer is testing:** matching pool type to task duration/count (the mirror image of Ex1).
**Mention first:** thread explosion from cached pool.

## Topic 21 · Exercise 1 — Unhandled rejection
**Findings**
- 🟠 **DEFINITE BUG — Unhandled `RejectedExecutionException`.** The bounded queue is right for backpressure, but the default `AbortPolicy` throws `RejectedExecutionException` when full, and `record` doesn't handle it — so under overload the caller (likely a request thread) gets an unhandled exception and crashes. Backpressure without a rejection strategy just moves the failure.

**Fix:** choose an intentional `RejectedExecutionHandler` — `CallerRunsPolicy` (throttles producers), or catch the rejection and drop-with-metric for droppable metrics, or block with a bounded offer/timeout. Decide based on whether metrics are droppable (usually yes).
**What a strong senior would say:** *"Bounding the queue is correct, but the default policy aborts with an exception the caller doesn't handle, so overload turns into crashes. For droppable metrics I'd catch the rejection and increment a 'dropped' counter, or use CallerRuns to throttle producers. The point is to choose an explicit overload behavior, not let it throw into the caller."*
**Interviewer is testing:** rejection policies and intentional overload handling.
**Mention first:** unhandled rejection under load.

## Topic 21 · Exercise 2 — "Bounded" queue that isn't
**Findings**
- 🟠 **DEFINITE BUG — Unbounded queue.** `new LinkedBlockingQueue<>()` with no capacity argument defaults to `Integer.MAX_VALUE` — effectively unbounded. `put` never blocks, so a fast producer outrunning consumers grows the queue until OOM. The intended memory bound and backpressure don't exist.

**Fix:** give it a capacity: `new LinkedBlockingQueue<>(1000)` (or `ArrayBlockingQueue<>(1000)`), so `put` blocks when full and applies real backpressure to the producer.
**What a strong senior would say:** *"The no-arg LinkedBlockingQueue is bounded only by Integer.MAX_VALUE, so it's effectively unbounded — `put` never blocks and a fast producer just grows the heap until OOM. The whole point was backpressure, so I'd set an explicit capacity, which makes `put` block and slows the producer to the consumer's rate."*
**Interviewer is testing:** knowing the default LinkedBlockingQueue capacity; backpressure.
**Mention first:** the missing capacity.

## Topic 22 · Exercise 1 — Oversized pool for computation
**Findings**
- 🟡 **PERFORMANCE CONCERN — Too many threads for CPU work.** 200 threads for pure CPU work on 8 cores doesn't increase throughput — only ~8 can run at once; the extra 192 add context-switching, cache thrash, and memory overhead, often making it *slower*. CPU-bound pools should be ~`availableProcessors` (± a little).

**Fix:** size ≈ number of cores (`Runtime.getRuntime().availableProcessors()`, maybe +1); measure to tune. Respect container CPU limits (see Topic 27 Ex2).
**What a strong senior would say:** *"For pure CPU work, only as many threads as cores can actually make progress — 200 on 8 cores just adds context-switch and cache overhead and can be slower. I'd size around the core count and measure. I'd also make sure we read the *container's* CPU limit, not the host's, in a Kubernetes setup."*
**Interviewer is testing:** CPU-bound pool sizing (~cores).
**Mention first:** oversized pool → thrash, no speedup.

## Topic 22 · Exercise 2 — No isolation between CPU work and latency-sensitive work
**Findings**
- 🟠 **DEFINITE BUG — No bulkhead between workloads.** Heavy multi-second CPU aggregations and fast lookups share one 8-thread pool. When several aggregations run, they occupy all threads and the latency-sensitive `quickLookup` queues behind them → its latency spikes. Different SLAs on one pool = interference.

**Fix:** **separate pools (bulkheads)** — a small dedicated pool for heavy aggregations (bounded), and a separate pool for quick lookups, so heavy work can't starve latency-sensitive work.
**What a strong senior would say:** *"These have different SLAs but share a pool, so a few heavy aggregations occupy all threads and the quick lookups queue behind them — latency spikes. I'd bulkhead them into separate pools so the heavy CPU work can't starve the latency-sensitive path, and bound the aggregation pool so it can't take over the machine either."*
**Interviewer is testing:** the bulkhead pattern / workload isolation.
**Mention first:** shared pool → latency interference.

## Topic 23 · Exercise 1 — Pool sized to cores for I/O
**Findings**
- 🟡 **PERFORMANCE CONCERN — Under-sized pool for I/O.** With 8 threads for tasks that spend ~500ms mostly *waiting* on the network, throughput is capped at ~8 concurrent fetches while cores sit idle. I/O-bound pools should have **more** threads than cores (they're mostly blocked, not computing).

**Fix:** size larger, guided by Little's Law / the wait-to-compute ratio (e.g. `cores * (1 + wait/compute)`), or use non-blocking/async I/O to get high concurrency without many threads. Bound it to what the target can absorb, and measure.
**What a strong senior would say:** *"Sizing to cores is right for CPU work but wrong for I/O — these threads are mostly parked on the network, so we're leaving throughput on the table. I'd raise the thread count based on the wait-to-service ratio, or move to non-blocking I/O for high fan-out. Either way I'd cap it to what the target endpoint can handle and validate with load testing."*
**Interviewer is testing:** I/O-bound sizing (> cores) and the reasoning behind it.
**Mention first:** under-sized I/O pool.

## Topic 23 · Exercise 2 — One slow dependency ties up the pool
**Findings**
- 🟠 **DEFINITE BUG — Head-of-line blocking / no bulkhead.** All three downstreams share one 20-thread pool. When C hangs (30s), its tasks occupy threads for the full timeout; enough C calls exhaust the shared pool, so A and B calls can't get a thread and also fail — the classic cascading failure from a shared pool + a slow dependency.

**Fix:** **per-dependency bulkheads** (separate bounded pools per downstream, e.g. Resilience4j Bulkhead), plus per-call timeouts and a circuit breaker on C so it fails fast instead of holding threads.
**What a strong senior would say:** *"They share one pool, so when C hangs its calls hold threads for the whole timeout and starve A and B — a cascading failure. I'd give each dependency its own bounded pool (a bulkhead) with per-call timeouts and a circuit breaker on C, so a sick dependency is contained and fails fast rather than taking the others down."*
**Interviewer is testing:** bulkheading downstream calls; cascading-failure containment.
**Mention first:** shared pool → cross-dependency cascade.

## Topic 24 · Exercise 1 — shutdownNow and assume done
**Findings**
- 🔴 **DEFINITE BUG — `shutdownNow` interrupts in-flight work + drops the queue.** `shutdownNow` attempts to **interrupt** running tasks and returns the queued (never-run) tasks, so in-flight exports are cut off mid-write (half-written files — violating the requirement) and queued exports are silently dropped. The log falsely claims completion.

**Fix:** graceful shutdown sequence:
```java
pool.shutdown(); // stop accepting new, let running/queued finish
if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {
    pool.shutdownNow(); // escalate only after grace period
}
```
Make export tasks write atomically (temp file + rename) so an interrupt can't leave partial files.
**What a strong senior would say:** *"`shutdownNow` interrupts running exports and discards queued ones, so we get half-written files and dropped work — the opposite of the requirement, and the 'finished' log is wrong. I'd call `shutdown()` then `awaitTermination` with a grace period, only escalating to `shutdownNow` if it overruns. I'd also make exports write to a temp file and rename, so even a forced stop can't leave a corrupt file."*
**Interviewer is testing:** shutdown() vs shutdownNow() semantics and graceful drain.
**Mention first:** shutdownNow interrupts/drops in-flight work.

## Topic 24 · Exercise 2 — No await on context close
**Findings**
- 🟠 **DEFINITE BUG — No lifecycle shutdown.** The pool is never shut down. On redeploy the process is killed while tasks are in flight, so queued/running notifications are lost (matches the symptom). There's no `@PreDestroy` to drain, so Spring can't stop it gracefully, and the pool would also leak if the bean were recreated.

**Fix:** add a `@PreDestroy` that does `shutdown()` + `awaitTermination(...)` (escalating to `shutdownNow`), or use Spring's `ThreadPoolTaskExecutor` with `setWaitForTasksToCompleteOnShutdown(true)` + `setAwaitTerminationSeconds(...)`, which integrates with the context lifecycle. For guaranteed delivery, back sends with a durable queue/retry.
**What a strong senior would say:** *"There's no shutdown hook, so on redeploy in-flight and queued notifications die with the process. I'd add a `@PreDestroy` that drains with shutdown + awaitTermination, or use Spring's ThreadPoolTaskExecutor configured to wait for tasks on shutdown. If these truly can't be lost, best-effort threads aren't enough — I'd use a durable queue with retries."*
**Interviewer is testing:** integrating executor lifecycle with the app/container shutdown; durability.
**Mention first:** no graceful drain on shutdown.

## Topic 25 · Exercise 1 — Cancel that does nothing
**Findings**
- 🟠 **DEFINITE BUG — Task ignores interruption.** `cancel(true)` sets the interrupt flag on the worker thread, but `bruteForceSearch`'s CPU loop never checks `Thread.interrupted()` (and does no interruptible blocking call), so it runs to completion regardless — the "cancel" frees the caller (which already timed out) but the work keeps consuming a thread/CPU. Under load these zombie searches pile up.

**Fix:** make the task **cooperatively cancellable** — check `Thread.currentThread().isInterrupted()` periodically in the loop and bail out:
```java
for (Document d : hugeCorpus) {
    if (Thread.currentThread().isInterrupted()) return List.of();
    if (matches(d, q)) hits.add(toHit(d));
}
```
**What a strong senior would say:** *"Cancellation in Java is cooperative — `cancel(true)` just interrupts the thread, but this CPU loop never checks the interrupt flag, so it ignores the cancel and runs to the end, holding a thread the caller already gave up on. I'd have the loop poll `isInterrupted()` and stop. Otherwise timed-out searches accumulate and exhaust the pool."*
**Interviewer is testing:** cooperative cancellation; interrupt is a request, not a kill.
**Mention first:** task ignores the interrupt.

## Topic 25 · Exercise 2 — Swallowed interrupt during blocking wait
**Findings**
- 🟠 **DEFINITE BUG — Swallowed interrupt makes the worker uncancellable.** `queue.take()` throws `InterruptedException` on interrupt (and clears the interrupt flag). The catch ignores it and loops, so the worker never stops and the interrupt status is lost — cancellation/shutdown can't end it. `while(true)` compounds it.

**Fix:** on `InterruptedException`, restore the flag and exit the loop:
```java
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    break; // stop promptly
}
```
(or loop on `!Thread.currentThread().isInterrupted()`).
**What a strong senior would say:** *"Swallowing InterruptedException and continuing makes this worker impossible to cancel or shut down cleanly — and it also erases the interrupt status. I'd restore the flag with `Thread.currentThread().interrupt()` and break out of the loop, so an interrupt actually stops it. This is the standard interrupt-handling contract."*
**Interviewer is testing:** the InterruptedException contract (restore + stop).
**Mention first:** swallowed interrupt → uncancellable.

## Topic 26 · Exercise 1 — Timed get without cancel
**Findings**
- 🟠 **DEFINITE BUG — Orphaned task after timeout.** On `TimeoutException` the code returns a fallback but never `cancel`s the future, so `heavyRender` keeps running and holding a pool thread. Under sustained load, timed-out-but-still-running renders accumulate and exhaust the executor — the timeout protects the caller's latency but not the system's resources.

**Fix:** `f.cancel(true)` on timeout (and make `heavyRender` responsive to interruption — see Topic 25), so the abandoned work is actually stopped and the thread reclaimed.
**What a strong senior would say:** *"The timeout frees the caller but leaves the render running and holding a thread, so under load the pool fills with zombie renders. I'd cancel the future on timeout and make the render interruptible so the thread is actually reclaimed. A timeout without cancellation only masks the latency; it doesn't bound resource use."*
**Interviewer is testing:** timeout must be paired with cancellation + interruptible task.
**Mention first:** orphaned task on timeout.

## Topic 26 · Exercise 2 — Retry with per-attempt timeout only
**Findings**
- 🟠 **DEFINITE BUG — No overall deadline (SLA breach).** Each attempt allows 30s and there are 3 attempts, so worst case is ~90s — but the user SLA is 20s. Per-attempt timeouts without an **overall budget** blow the end-to-end deadline; the user is long gone before it gives up.

**Fix:** track a total deadline and shrink each attempt's timeout to the remaining budget (`remaining = deadline - now`), stopping when the budget is exhausted; cap the per-attempt timeout at the remaining time. Consider fewer retries and backoff within the budget.
**What a strong senior would say:** *"The timeouts are per-attempt, but there's no overall budget — three 30-second attempts can take 90 seconds against a 20-second SLA. I'd compute a total deadline up front and cap each attempt to the time remaining, aborting when the budget's gone. Retries should live inside the deadline, not extend it."*
**Interviewer is testing:** overall-deadline vs per-attempt timeout; SLA-aware retries.
**Mention first:** no end-to-end deadline → SLA breach.

## Topic 27 · Exercise 1 — Blocking the common pool
**Findings**
- 🟠 **DEFINITE BUG — Blocking JDBC on the common ForkJoinPool.** `parallelStream` runs on the shared common pool; each `fetch` is blocking JDBC (~100ms). This ties up the common pool's few workers in blocking waits, starving every other parallel-stream/CompletableFuture user in the JVM, and gives poor concurrency (pool sized to cores) for an I/O workload.

**Fix:** don't use parallel streams for blocking I/O. Use a dedicated bounded executor with `CompletableFuture`/`ExecutorService`, sized for I/O; or a batch/`IN`-query to fetch all ids at once (often the best fix — one query beats N parallel ones).
**What a strong senior would say:** *"Parallel streams run on the common ForkJoinPool, so doing blocking JDBC here parks its workers and starves the rest of the app's parallel work, with only cores-worth of concurrency anyway. I'd move this to a dedicated I/O executor with CompletableFutures — or, better, replace N fetches with a single batched `IN` query, which sidesteps the whole concurrency question."*
**Interviewer is testing:** why blocking the common pool is harmful; batching as the real fix.
**Mention first:** blocking JDBC on the common pool.

## Topic 27 · Exercise 2 — Parallelism in a constrained container
**Findings**
- 🟠 **DEFINITE BUG (vs expectation) — Common-pool parallelism = cores − 1.** The common ForkJoinPool's parallelism defaults to `availableProcessors() - 1`. In a 1-CPU pod that's **0/1 worker**, so `parallelStream` runs essentially **serially** (mostly on the calling thread) — the comment's "parallel across cores" is false in this deployment. No speedup, plus the parallel overhead.

**Fix:** if the pod really has 1 CPU, don't expect parallel speedup — keep it sequential (simpler), or give the pod more CPU and pass an explicit sized `ForkJoinPool`/executor if parallelism is desired; verify `availableProcessors()` reflects the cgroup limit (it does on modern JVMs). Match the concurrency model to the CPU allocation.
**What a strong senior would say:** *"The common pool's parallelism is processors minus one, so in a 1-vCPU pod it's basically serial — the 'parallel across cores' comment doesn't hold, and we pay stream overhead for no gain. Either accept it's sequential and simplify, or, if we need real parallelism, allocate more CPU and use an explicit sized pool. I'd confirm the JVM sees the cgroup CPU limit, which it does on current versions."*
**Interviewer is testing:** container CPU limits vs parallelism assumptions — very real in Kubernetes.
**Mention first:** ~serial execution in a 1-CPU pod.

## Topic 28 · Exercise 1 — Non-associative reduction
**Findings**
- 🔴 **DEFINITE BUG — Non-associative reducer in parallel.** `reduce(100.0, (acc, s) -> acc - s)` uses subtraction, which is **not associative**. Parallel reduction splits and combines sub-results in arbitrary groupings, so the result is **nondeterministic and wrong** (it effectively does `100 - a - b ...` only for a left-fold; in parallel it computes things like `(100-a) - (100-b) ...` shapes). `reduce` requires an associative accumulator (and the combiner and identity must be consistent).

**Fix:** reformulate as associative — sum the penalties, then subtract once: `100.0 - stageScores.parallelStream().mapToDouble(Double::doubleValue).sum();` (addition is associative). Or keep it sequential if the fold is inherently ordered.
**What a strong senior would say:** *"Parallel `reduce` requires an associative operation, and subtraction isn't — so the result depends on how the stream splits and is nondeterministically wrong. I'd rewrite it as an associative reduction: sum the penalties in parallel and subtract from 100 once. When an operation is inherently a left-fold, parallelizing it is simply incorrect."*
**Interviewer is testing:** the associativity requirement of parallel reduce — a real correctness trap.
**Mention first:** non-associative reducer → wrong parallel result.

## Topic 28 · Exercise 2 — Parallel sum of a large array
**Findings**
- ⚪ **NOT A BUG.** This is correct and appropriate. `mapToLong(...).sum()` is an associative, stateless reduction with no shared mutable state; parallelizing an independent sum over tens of millions of elements on a good source is exactly the right use of a parallel stream. The reviewer's "not sure this is thread-safe?" note is a **false alarm** — there's no shared state to be unsafe.
- 🟢 Only caveats (not bugs): a `List<Long>` boxes values (memory/cache cost) and doesn't split as well as a primitive array; on a CPU-constrained container the speedup may be limited (see Topic 27 Ex2). Neither is a correctness issue.

**What a strong senior would say:** *"This is fine — it's a stateless, associative sum with no shared mutable state, so the parallel stream is thread-safe and well-suited to tens of millions of independent values; I'd tell the reviewer their note is a false alarm. If I wanted to optimize, I'd store primitives in a `long[]` to avoid boxing and split better, and I'd remember the speedup depends on available CPUs — but there's no bug here."*
**Interviewer is testing:** *not* flagging correct parallel code; distinguishing a real thread-safety issue from a non-issue (and handling the misleading comment).
**Mention first:** state it's correct and the comment is a false alarm.

---

# Senior Concurrency Review Checklist — Part 3

Run these mentally on any concurrent code, **critical first**:

**Threads & tasks:** `start()` not `run()` · never thread-per-request (bounded pool) · raw-thread exceptions vanish (use executor + observe the Future) · don't share a stateful Runnable.
**Futures & CompletableFuture:** always `get(timeout)` · a discarded Future swallows the exception *and* means "submitted," not "done" · CompletableFuture needs `exceptionally`/`whenComplete` or exceptions are lost · `supplyAsync` with no executor runs on the **common pool** — never block it.
**Executors & pools:** `Runnable::run` is a same-thread executor (not async) · `newFixedThreadPool` = unbounded queue (OOM) · `newCachedThreadPool` = unbounded threads for long tasks (OOM) · bound the queue **and** set a rejection policy · `LinkedBlockingQueue()` with no arg is unbounded · never submit-and-block on the same bounded pool (starvation deadlock).
**Sizing & isolation:** CPU-bound ≈ cores; I/O-bound > cores (wait/compute ratio) · bulkhead different SLAs / different dependencies into separate pools · a slow dependency on a shared pool causes cascading failure · common-pool parallelism = cores−1 (≈ serial in a 1-CPU pod).
**Locks:** locks must be `final` (don't reassign a monitor) · always `unlock()` in `finally` · lock reads too (or use `volatile`/immutable-swap) · ReadWriteLock can't upgrade read→write (deadlock) · always check `tryLock()`'s return · `Semaphore`/plain lock aren't reentrant → self-deadlock · consistent global lock ordering to avoid deadlock.
**volatile / atomics:** `volatile` = visibility, not atomicity (`++` still races) · volatile publishes the *reference*, not deep mutations (copy-on-write) · use atomic combinators (`getAndIncrement`, `merge`), not get-then-set · combining two atomics doesn't make a multi-field invariant atomic · non-volatile `long`/`double` can tear on 32-bit.
**Races & publication:** check-then-act on shared state (lazy init, capacity bound, double-booking) needs atomicity — often at the **DB** for multi-instance · unsafe publication → make the field `volatile`/`final` or the object immutable · stop-flags must be `volatile`.
**ConcurrentHashMap:** compound ops need `computeIfAbsent`/`merge` (Part 2) · keep the `computeIfAbsent` function short & non-blocking · CHM forbids null keys/values.
**Cancellation & timeouts:** cancellation is cooperative — tasks must check `isInterrupted()` · restore the interrupt flag and stop on `InterruptedException` · pair every timeout with `cancel(true)` and an interruptible task · give retries an **overall deadline**, not just per-attempt timeouts.
**Parallel streams:** no side effects into shared collections (Part 2) · `reduce` must be **associative** · runs on the common pool · verify it's actually beneficial — and **don't flag a correct, stateless, associative parallel sum**.

**Always:** name at least one *not-a-bug* (correct parallel sum, correct volatile stop-flag) to prove judgment; separate a guaranteed deadlock/race (🔴) from a contention/starvation *potential* (🟡 — "depends on load, I'd measure"); when correctness hinges on "is this called from one thread / how many instances / how read-heavy," **ask** before asserting.

---

*End of Part 3. Say **"next"** for Part 4 — Spring Boot Code Review (32 topics × 2 = 64 exercises).*
