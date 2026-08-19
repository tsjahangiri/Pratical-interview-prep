# Part 2 — Java Code Review (Practice Workbook)

**54 exercises — 27 topics × 2.** Java 17+, realistic production code. Review each cold; write findings before opening the answer key.

> Some exercises contain multiple bugs, some contain code that *looks* wrong but is correct, and some contain misleading comments. Verify everything yourself.

**Legend:** 🔴 Critical · 🟠 High · 🟡 Medium · 🟢 Low · ⚪ Nit

---

# EXERCISES

## Topic 1 — OOP

### Exercise 1 — Shopping cart internals

**Scenario**
`ShoppingCart` is used across the checkout flow. Callers read the items to render the cart.

**Code**
```java
public class ShoppingCart {
    private final List<Item> items = new ArrayList<>();
    private BigDecimal total = BigDecimal.ZERO;

    public void add(Item item) {
        items.add(item);
        total = total.add(item.getPrice());
    }

    public List<Item> getItems() {
        return items;
    }

    public BigDecimal getTotal() {
        return total;
    }
}
```

**Your Task**
Review the encapsulation and object design.

---

### Exercise 2 — Read-only list subtype

**Scenario**
`ImmutableAuditLog` is meant to be an append-then-frozen list handed to reporting code that must not modify it.

**Code**
```java
public class ImmutableAuditLog extends ArrayList<String> {
    private boolean frozen = false;

    public void freeze() { this.frozen = true; }

    @Override
    public boolean add(String s) {
        if (frozen) throw new UnsupportedOperationException("frozen");
        return super.add(s);
    }
}
```

**Your Task**
Review the design against OOP principles (substitutability in particular).

---

## Topic 2 — SOLID

### Exercise 1 — Shipping cost calculator

**Scenario**
`ShippingCalculator` computes cost by carrier. New carriers are onboarded regularly.

**Code**
```java
public BigDecimal cost(Shipment s) {
    if (s.getCarrier().equals("DHL")) {
        return s.getWeight().multiply(new BigDecimal("2.5"));
    } else if (s.getCarrier().equals("UPS")) {
        return s.getWeight().multiply(new BigDecimal("2.7")).add(new BigDecimal("1"));
    } else if (s.getCarrier().equals("FEDEX")) {
        return s.getWeight().multiply(new BigDecimal("3.0"));
    }
    throw new IllegalArgumentException("Unknown carrier");
}
```

**Your Task**
Review against SOLID (focus on extensibility).

---

### Exercise 2 — Repository interface

**Scenario**
All persistence goes through one interface. A new read-only, cache-backed implementation is being added for a reporting service.

**Code**
```java
public interface DataRepository<T> {
    T findById(Long id);
    List<T> findAll();
    T save(T entity);
    void delete(Long id);
    void bulkImport(List<T> entities);
    void reindex();
    void purgeCache();
}

// The new read-only implementation:
public class CachedReadRepository<T> implements DataRepository<T> {
    public T findById(Long id) { /* ... */ return null; }
    public List<T> findAll() { /* ... */ return List.of(); }
    public T save(T e) { throw new UnsupportedOperationException(); }
    public void delete(Long id) { throw new UnsupportedOperationException(); }
    public void bulkImport(List<T> e) { throw new UnsupportedOperationException(); }
    public void reindex() { throw new UnsupportedOperationException(); }
    public void purgeCache() { /* ... */ }
}
```

**Your Task**
Review against SOLID.

---

## Topic 3 — Immutability

### Exercise 1 — "Immutable" money range

**Scenario**
`PriceBand` is treated as immutable and shared across threads and cached.

**Code**
```java
public final class PriceBand {
    private final List<BigDecimal> tiers;

    public PriceBand(List<BigDecimal> tiers) {
        this.tiers = tiers;
    }

    public List<BigDecimal> getTiers() {
        return tiers;
    }
}
```

**Your Task**
Review whether this class is actually immutable.

---

### Exercise 2 — Frozen config

**Scenario**
`AppConfig` is built once at startup and read from everywhere.

**Code**
```java
public final class AppConfig {
    private final Map<String, String> settings;

    public AppConfig(Map<String, String> settings) {
        this.settings = new HashMap<>(settings); // defensive copy on the way in
    }

    public Map<String, String> getSettings() {
        return settings;
    }
}
```

**Your Task**
Review the immutability. State whether the defensive copy fully protects it.

---

## Topic 4 — Java Records

### Exercise 1 — Order record with items

**Scenario**
`OrderSnapshot` is a record passed to an audit system that assumes snapshots never change after creation.

**Code**
```java
public record OrderSnapshot(Long id, List<String> lineItems) {}

// usage:
List<String> items = new ArrayList<>(List.of("A", "B"));
OrderSnapshot snap = new OrderSnapshot(1L, items);
items.add("C"); // later, elsewhere
```

**Your Task**
Review whether the record provides the immutability the audit system assumes.

---

### Exercise 2 — Signature record

**Scenario**
`Signature` records a signed payload. Two signatures are considered equal if they carry the same bytes. They're used as `HashMap` keys.

**Code**
```java
public record Signature(String algorithm, byte[] data) {}

// usage:
Map<Signature, String> seen = new HashMap<>();
seen.put(new Signature("SHA256", payload), "first");
boolean dup = seen.containsKey(new Signature("SHA256", payload)); // expected true
```

**Your Task**
Review whether equality/hashing behaves as the scenario expects.

---

## Topic 5 — equals / hashCode

### Exercise 1 — Product value object

**Scenario**
`Product` is placed in `HashSet`s and used as `Map` keys throughout the catalog service.

**Code**
```java
public class Product {
    private final String sku;
    private final String name;

    public Product(String sku, String name) {
        this.sku = sku;
        this.name = name;
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Product p)) return false;
        return sku.equals(p.sku);
    }
}
```

**Your Task**
Review the equals/hashCode implementation.

---

### Exercise 2 — Entity identity

**Scenario**
`Customer` is a JPA entity. New (unsaved) customers are added to a `Set` before the batch is persisted, then compared.

**Code**
```java
@Entity
public class Customer {
    @Id @GeneratedValue
    private Long id;
    private String email;

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Customer c)) return false;
        return Objects.equals(id, c.id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}
```

**Your Task**
Review equality for entities, especially before persistence.

---

## Topic 6 — String

### Exercise 1 — Status comparison

**Scenario**
`OrderService` branches on an order's status string, which arrives from an external API.

**Code**
```java
public boolean isActive(Order order) {
    String status = order.getStatus();
    if (status == "ACTIVE") {
        return true;
    }
    return false;
}
```

**Your Task**
Review the string comparison.

---

### Exercise 2 — Normalizing input

**Scenario**
`UserService` normalizes an email before saving it.

**Code**
```java
public String normalize(String email) {
    email.trim();
    email.toLowerCase();
    return email;
}
```

**Your Task**
Review whether the normalization does what it appears to.

---

## Topic 7 — StringBuilder

### Exercise 1 — Shared report builder

**Scenario**
`ReportAggregator` collects log lines from multiple worker threads into one report. The builder is a shared field.

**Code**
```java
@Service
public class ReportAggregator {
    private final StringBuilder report = new StringBuilder();

    public void append(String line) {
        report.append(line).append('\n');
    }

    public String build() {
        return report.toString();
    }
}
```

**Your Task**
Review for thread safety.

---

### Exercise 2 — Local join

**Scenario**
`CsvService.toRow` builds a single CSV row. The endpoint is called concurrently.

**Code**
```java
public String toRow(List<String> fields) {
    StringBuilder sb = new StringBuilder();
    for (int i = 0; i < fields.size(); i++) {
        if (i > 0) sb.append(',');
        sb.append(fields.get(i));
    }
    return sb.toString();
}
```

**Your Task**
Review for thread safety and correctness. State clearly whether the `StringBuilder` is a problem here.

---

## Topic 8 — StringBuffer

### Exercise 1 — Legacy message builder

**Scenario**
`MessageFormatter.format` builds a short message. It runs single-threaded inside a request handler.

**Code**
```java
public String format(String name, int count) {
    StringBuffer sb = new StringBuffer();
    sb.append("Hello ").append(name).append(", you have ")
      .append(count).append(" messages");
    return sb.toString();
}
```

**Your Task**
Review this. Is `StringBuffer` appropriate here?

---

### Exercise 2 — "Thread-safe" accumulator

**Scenario**
`Tracker` appends two related tokens per event from many threads. A previous author chose `StringBuffer` "so it's thread-safe."

**Code**
```java
public class Tracker {
    // StringBuffer is synchronized, so this is thread-safe
    private final StringBuffer log = new StringBuffer();

    public void record(String user, String action) {
        log.append(user);
        log.append(":");
        log.append(action);
        log.append(";");
    }
}
```

**Your Task**
Review whether the "thread-safe" claim holds for the intended output.

---

## Topic 9 — Java Collections

### Exercise 1 — Removing while iterating

**Scenario**
`CartService.removeExpired` drops expired items from a cart during checkout.

**Code**
```java
public void removeExpired(List<Item> items) {
    for (Item item : items) {
        if (item.isExpired()) {
            items.remove(item);
        }
    }
}
```

**Your Task**
Review the iteration/removal logic.

---

### Exercise 2 — Building a mutable list

**Scenario**
`TagService.defaultTags` returns a starter set of tags that the caller will then add to.

**Code**
```java
public List<String> defaultTags() {
    List<String> tags = Arrays.asList("new", "unreviewed");
    return tags;
}

// caller:
List<String> tags = tagService.defaultTags();
tags.add("priority");
```

**Your Task**
Review whether the caller's usage will work.

---

## Topic 10 — HashMap internals / misuse

### Exercise 1 — Mutable key

**Scenario**
`SessionRegistry` keys active sessions by a `UserKey` object. `UserKey`'s fields can change while a user upgrades their plan.

**Code**
```java
public class UserKey {
    private String userId;
    private String plan;
    // equals & hashCode use BOTH userId and plan
}

Map<UserKey, Session> sessions = new HashMap<>();
UserKey key = new UserKey("u1", "FREE");
sessions.put(key, session);
key.setPlan("PREMIUM");     // later
Session s = sessions.get(key);   // expected: the same session
```

**Your Task**
Review the use of this type as a map key.

---

### Exercise 2 — Assuming iteration order

**Scenario**
`MenuService` builds a display order of menu sections from a map populated in a fixed sequence.

**Code**
```java
public List<String> sectionsInOrder() {
    Map<String, Section> map = new HashMap<>();
    map.put("starters", ...);
    map.put("mains", ...);
    map.put("desserts", ...);
    return new ArrayList<>(map.keySet()); // must come out starters, mains, desserts
}
```

**Your Task**
Review whether the ordering requirement is met.

---

## Topic 11 — ConcurrentHashMap

### Exercise 1 — First-writer registration

**Scenario**
`DeviceRegistry` registers a device the first time it's seen. Called concurrently by many ingestion threads.

**Code**
```java
private final ConcurrentHashMap<String, Device> devices = new ConcurrentHashMap<>();

public void register(String id, Device device) {
    if (!devices.containsKey(id)) {
        devices.put(id, device);
        auditService.log("registered " + id);
    }
}
```

**Your Task**
Review the concurrency correctness.

---

### Exercise 2 — Per-key counter

**Scenario**
`HitCounter` counts hits per endpoint. Called concurrently by request threads.

**Code**
```java
private final ConcurrentHashMap<String, Integer> hits = new ConcurrentHashMap<>();

public void hit(String endpoint) {
    Integer current = hits.get(endpoint);
    if (current == null) current = 0;
    hits.put(endpoint, current + 1);
}
```

**Your Task**
Review the counting logic under concurrency.

---

## Topic 12 — List selection

### Exercise 1 — Positional access

**Scenario**
`PlaylistService` shuffles then reads tracks by index in a tight loop. Playlists can hold tens of thousands of tracks.

**Code**
```java
public long totalDuration(LinkedList<Track> tracks) {
    long total = 0;
    for (int i = 0; i < tracks.size(); i++) {
        total += tracks.get(i).getDurationMs();
    }
    return total;
}
```

**Your Task**
Review the data-structure choice and performance.

---

### Exercise 2 — Work queue

**Scenario**
`TaskBuffer` holds pending tasks; new tasks are prepended (highest priority first) and tasks are consumed from the front. It handles a high, steady throughput.

**Code**
```java
private final List<Task> buffer = new ArrayList<>();

public void addFront(Task t) {
    buffer.add(0, t);
}

public Task next() {
    return buffer.remove(0);
}
```

**Your Task**
Review the data-structure choice for this access pattern.

---

## Topic 13 — Set selection

### Exercise 1 — Sorted unique report

**Scenario**
`RegionService.uniqueRegions` returns the distinct regions seen, and the UI displays them in alphabetical order.

**Code**
```java
public Set<String> uniqueRegions(List<Order> orders) {
    Set<String> regions = new HashSet<>();
    for (Order o : orders) {
        regions.add(o.getRegion());
    }
    return regions; // UI renders these expecting alphabetical order
}
```

**Your Task**
Review whether the ordering expectation is met.

---

### Exercise 2 — Deduplicating by a comparator

**Scenario**
`EventService.distinctEvents` deduplicates events. Two events are "the same event" only if id AND timestamp match; the code wants unique events sorted by timestamp.

**Code**
```java
public Set<Event> distinctEvents(List<Event> events) {
    Set<Event> set = new TreeSet<>(Comparator.comparing(Event::getTimestamp));
    set.addAll(events);
    return set;
}
```

**Your Task**
Review whether deduplication is correct.

---

## Topic 14 — Map selection

### Exercise 1 — Recently-used cache

**Scenario**
`ThumbnailCache` should keep only the 100 most-recently-used thumbnails in memory, evicting the least recently used.

**Code**
```java
public class ThumbnailCache {
    private final Map<String, byte[]> cache = new HashMap<>();

    public byte[] get(String key) {
        return cache.get(key);
    }

    public void put(String key, byte[] data) {
        if (cache.size() >= 100) {
            cache.remove(cache.keySet().iterator().next());
        }
        cache.put(key, data);
    }
}
```

**Your Task**
Review whether this implements an LRU cache correctly.

---

### Exercise 2 — Synchronized map iteration

**Scenario**
`MetricsSnapshot` exposes a live view of counters shared across threads. Reads iterate the map to build a snapshot.

**Code**
```java
private final Map<String, Long> counters =
        Collections.synchronizedMap(new HashMap<>());

public Map<String, Long> snapshot() {
    Map<String, Long> copy = new HashMap<>();
    for (Map.Entry<String, Long> e : counters.entrySet()) {
        copy.put(e.getKey(), e.getValue());
    }
    return copy;
}
```

**Your Task**
Review the concurrency correctness of the iteration.

---

## Topic 15 — Java Streams

### Exercise 1 — Parallel enrichment

**Scenario**
`EnrichmentService` enriches a list of orders by calling a remote service per order, collecting results.

**Code**
```java
public List<EnrichedOrder> enrich(List<Order> orders) {
    List<EnrichedOrder> results = new ArrayList<>();
    orders.parallelStream().forEach(o -> {
        EnrichedOrder e = remoteClient.enrich(o); // blocking HTTP
        results.add(e);
    });
    return results;
}
```

**Your Task**
Review the stream usage for correctness and appropriateness.

---

### Exercise 2 — Indexing by key

**Scenario**
`UserService.byEmail` builds a lookup map from a list of users. Emails are expected to be unique, but the data occasionally contains duplicates.

**Code**
```java
public Map<String, User> byEmail(List<User> users) {
    return users.stream()
                .collect(Collectors.toMap(User::getEmail, u -> u));
}
```

**Your Task**
Review the collector behavior.

---

## Topic 16 — Optional

### Exercise 1 — Unwrapping a lookup

**Scenario**
`AccountService.balanceOf` returns a balance for an account that may not exist.

**Code**
```java
public BigDecimal balanceOf(Long accountId) {
    Optional<Account> account = accountRepository.findById(accountId);
    return account.get().getBalance();
}
```

**Your Task**
Review the Optional handling.

---

### Exercise 2 — Default from a fallback source

**Scenario**
`ConfigService.timeout` returns a configured timeout or falls back to loading a default from a remote config service (a slow call).

**Code**
```java
public Duration timeout() {
    return localConfig.getTimeout()          // Optional<Duration>
            .orElse(remoteConfig.loadDefaultTimeout()); // remote network call
}
```

**Your Task**
Review the fallback logic, including its performance implications.

---

## Topic 17 — Generics

### Exercise 1 — Raw type processor

**Scenario**
`EventBus.publish` dispatches events to handlers stored in a raw collection (legacy code).

**Code**
```java
private List handlers = new ArrayList();

public void addHandler(Handler h) {
    handlers.add(h);
}

public void publish(Event e) {
    for (Object o : handlers) {
        Handler h = (Handler) o;
        h.handle(e);
    }
}
```

**Your Task**
Review the use of generics/raw types.

---

### Exercise 2 — Summing numbers

**Scenario**
`MathUtil.sum` should accept a list of any `Number` subtype (Integers, Doubles, BigDecimals-as-Number, etc.).

**Code**
```java
public static double sum(List<Number> numbers) {
    double total = 0;
    for (Number n : numbers) {
        total += n.doubleValue();
    }
    return total;
}

// caller wants to do:
List<Integer> ints = List.of(1, 2, 3);
double s = MathUtil.sum(ints); // does not compile
```

**Your Task**
Review the generic signature and API usability.

---

## Topic 18 — Checked vs unchecked exceptions

### Exercise 1 — Wrapping and losing the cause

**Scenario**
`FileImportService` reads a file and wraps failures for the caller.

**Code**
```java
public List<Record> importFile(Path path) {
    try {
        return parse(Files.readAllLines(path));
    } catch (IOException e) {
        throw new RuntimeException("Import failed");
    }
}
```

**Your Task**
Review the exception translation.

---

### Exercise 2 — Catching everything

**Scenario**
`JobRunner.runSafely` is meant to keep the scheduler alive even if a job fails.

**Code**
```java
public void runSafely(Job job) {
    try {
        job.execute();
    } catch (Throwable t) {
        log.warn("Job failed, continuing", t);
    }
}
```

**Your Task**
Review the scope of what is caught.

---

## Topic 19 — Try-with-resources

### Exercise 1 — Manual close in finally

**Scenario**
`DataExporter` streams rows to a file. A previous author used a manual finally block.

**Code**
```java
public void export(Path path, List<String> rows) throws IOException {
    Writer w = null;
    try {
        w = Files.newBufferedWriter(path);
        for (String r : rows) {
            w.write(r);
            w.write('\n');
        }
    } finally {
        w.close();
    }
}
```

**Your Task**
Review the resource handling and exception behavior.

---

### Exercise 2 — Reading a response body

**Scenario**
`ApiClient.readBody` reads the body of an HTTP response using try-with-resources.

**Code**
```java
public String readBody(HttpResponse<InputStream> response) throws IOException {
    try (InputStream in = response.body();
         BufferedReader reader = new BufferedReader(new InputStreamReader(in, StandardCharsets.UTF_8))) {
        return reader.lines().collect(Collectors.joining("\n"));
    }
}
```

**Your Task**
Review the resource handling. State clearly whether there is a real problem here.

---

## Topic 20 — Java Date/Time API

### Exercise 1 — Scheduling across regions

**Scenario**
`ReminderService` stores the moment a reminder should fire. The system serves users in multiple time zones, and reminders are compared against "now" on servers in UTC.

**Code**
```java
public class Reminder {
    private LocalDateTime fireAt;   // set from user input

    public boolean isDue() {
        return LocalDateTime.now().isAfter(fireAt);
    }
}
```

**Your Task**
Review the date/time modeling for a multi-region system.

---

### Exercise 2 — Same time tomorrow

**Scenario**
`BillingService.nextRun` schedules the next daily billing run for "the same wall-clock time tomorrow" in a given zone. The zone observes daylight saving time.

**Code**
```java
public ZonedDateTime nextRun(ZonedDateTime current) {
    return current.plus(Duration.ofHours(24));
}
```

**Your Task**
Review the time arithmetic across DST boundaries.

---

## Topic 21 — BigDecimal

### Exercise 1 — Comparing amounts

**Scenario**
`PaymentValidator` checks whether a payment exactly matches an invoice total. Amounts come from different sources with different scales (e.g. `100.0` vs `100.00`).

**Code**
```java
public boolean matches(BigDecimal payment, BigDecimal invoiceTotal) {
    return payment.equals(invoiceTotal);
}
```

**Your Task**
Review the comparison for correctness.

---

### Exercise 2 — Splitting a bill

**Scenario**
`BillingService.perPersonShare` divides a total evenly among people.

**Code**
```java
public BigDecimal perPersonShare(BigDecimal total, int people) {
    return total.divide(new BigDecimal(people));
}
```

**Your Task**
Review the division for correctness and robustness.

---

## Topic 22 — Enums

### Exercise 1 — Persisting order status

**Scenario**
`Order` stores its status as a JPA-mapped enum. New statuses are inserted into the enum over time (sometimes alphabetically, in the middle).

**Code**
```java
public enum OrderStatus { NEW, PAID, SHIPPED, CANCELLED }

@Entity
public class Order {
    @Enumerated(EnumType.ORDINAL)
    private OrderStatus status;
}
```

**Your Task**
Review how the enum is persisted, considering future changes.

---

### Exercise 2 — Enum with counters

**Scenario**
`Currency` is an enum; the team added a per-currency usage counter that increments each time the currency is used.

**Code**
```java
public enum Currency {
    USD, EUR, GBP;

    private int usageCount = 0;

    public void use() {
        usageCount++;
    }

    public int getUsageCount() {
        return usageCount;
    }
}
```

**Your Task**
Review this design (state, sharing, concurrency).

---

## Topic 23 — Interfaces vs abstract classes

### Exercise 1 — Base class constructor

**Scenario**
`Report` is an abstract base; subclasses supply a title used to build a header at construction time.

**Code**
```java
public abstract class Report {
    private final String header;

    protected Report() {
        this.header = "=== " + title() + " ===";
    }

    protected abstract String title();
    public String getHeader() { return header; }
}

public class SalesReport extends Report {
    private final String region;
    public SalesReport(String region) { this.region = region; }
    @Override protected String title() { return "Sales for " + region; }
}
```

**Your Task**
Review the initialization design.

---

### Exercise 2 — Shared behavior via default methods

**Scenario**
`AuditableEntity` is an interface giving entities audit behavior. The team wants shared "last modified" tracking across all entities.

**Code**
```java
public interface Auditable {
    // holds the last-modified timestamp for every implementer
    default void touch() {
        this.lastModified = Instant.now(); // wants to store state
    }
    Instant getLastModified();
}
```

**Your Task**
Review whether an interface is the right tool for this requirement.

---

## Topic 24 — Composition vs inheritance

### Exercise 1 — Instrumented set

**Scenario**
`CountingSet` wraps a set and counts how many elements have ever been added, including via `addAll`.

**Code**
```java
public class CountingSet<E> extends HashSet<E> {
    private int addCount = 0;

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }

    public int getAddCount() { return addCount; }
}
```

**Your Task**
Review the inheritance-based design and the counting correctness.

---

### Exercise 2 — Premium account by inheritance

**Scenario**
`PremiumAccount` reuses `StandardAccount`'s logic but adds a monthly bonus and should never allow the account to go negative.

**Code**
```java
public class StandardAccount {
    protected BigDecimal balance = BigDecimal.ZERO;
    public void withdraw(BigDecimal amt) {
        balance = balance.subtract(amt); // allows negative (overdraft)
    }
    public void deposit(BigDecimal amt) { balance = balance.add(amt); }
}

public class PremiumAccount extends StandardAccount {
    public void applyBonus(BigDecimal bonus) { deposit(bonus); }
    // inherits withdraw as-is
}
```

**Your Task**
Review the reuse strategy and whether the invariant holds.

---

## Topic 25 — Dependency inversion

### Exercise 1 — Concrete dependency type

**Scenario**
`OrderService` depends on the repository. The team wants to unit-test it with a fake and later add a caching layer.

**Code**
```java
@Service
public class OrderService {
    private final JpaOrderRepository repository; // concrete class

    public OrderService(JpaOrderRepository repository) {
        this.repository = repository;
    }
}
```

**Your Task**
Review against dependency inversion.

---

### Exercise 2 — Hidden static dependency

**Scenario**
`CheckoutService.checkout` charges a card via a gateway.

**Code**
```java
@Service
public class CheckoutService {
    public void checkout(Order order) {
        ChargeResult result = PaymentGateway.getInstance()
                                             .charge(order.getCard(), order.getTotal());
        if (!result.isSuccess()) throw new PaymentFailedException();
    }
}
```

**Your Task**
Review the dependency structure and testability.

---

## Topic 26 — Clean Code

### Exercise 1 — Nested eligibility check

**Scenario**
`LoanService.isEligible` decides whether an applicant qualifies.

**Code**
```java
public boolean isEligible(Applicant a) {
    if (a != null) {
        if (a.getAge() >= 18) {
            if (a.getCreditScore() > 650) {
                if (a.getIncome().compareTo(new BigDecimal("30000")) >= 0) {
                    if (!a.hasActiveDefault()) {
                        return true;
                    }
                }
            }
        }
    }
    return false;
}
```

**Your Task**
Review for readability/maintainability. Note any correctness concern too.

---

### Exercise 2 — Comment vs behavior

**Scenario**
`SearchService.findMatches` is documented for its callers.

**Code**
```java
/**
 * Returns matching products, or an empty list if none are found.
 * Never returns null.
 */
public List<Product> findMatches(String query) {
    List<Product> results = repository.search(query);
    if (results.isEmpty()) {
        return null;
    }
    return results;
}
```

**Your Task**
Review the code against its documentation.

---

## Topic 27 — Code smells

### Exercise 1 — Address data clump

**Scenario**
`ShippingService.createLabel` and several sibling methods pass the same address fields around individually.

**Code**
```java
public Label createLabel(String name, String street, String city,
                         String zip, String country) {
    validate(street, city, zip, country);
    return new Label(name, format(city, street, country, zip));
}

// a caller:
createLabel(customer.getName(),
            customer.getStreet(), customer.getCity(),
            customer.getCountry(), customer.getZip());
```

**Your Task**
Review for code smells and any resulting bug.

---

### Exercise 2 — Reaching through objects

**Scenario**
`NotificationService.notifyManager` sends a message to an employee's department manager.

**Code**
```java
public void notifyManager(Employee employee, String message) {
    String email = employee.getDepartment().getManager().getContact().getEmail();
    emailClient.send(email, message);
}
```

**Your Task**
Review for code smells and robustness.

---

---

# ANSWER KEY

## Topic 1 · Exercise 1 — Shopping cart internals
**Findings**
- 🟠 **DEFINITE BUG — Encapsulation.** `getItems()` returns the internal `List` by reference; a caller can `getItems().clear()` or add items, mutating cart state and desyncing `total` (which is only updated in `add`). Broken invariant.
- 🟢 Money in `BigDecimal` is good; keep it.

**Fix:** return `Collections.unmodifiableList(items)` or `List.copyOf(items)`; keep `total` derived only through `add`.
**What a strong senior would say:** *"The cart leaks its internal list, so callers can mutate items directly and the cached `total` silently goes stale. I'd return an unmodifiable view so the only way to change items is through `add`, which keeps the total consistent."*
**Interviewer is testing:** encapsulation and invariant protection.
**Expected:** leaked mutable collection. **Bonus:** the total/items invariant desync. **Mention first:** the leak.

## Topic 1 · Exercise 2 — Read-only list subtype
**Findings**
- 🟠 **DESIGN CONCERN — Liskov violation.** Extending `ArrayList` and throwing from `add` when frozen breaks substitutability: code expecting a `List` can get an `UnsupportedOperationException` from a normal-looking list. Also, only `add` is overridden — `addAll`, `set`, `remove`, `clear`, and the iterator's `remove` all still mutate a "frozen" log. Inheritance exposes dozens of mutators you'd have to override.
- 🟡 Prefer **composition**: wrap a list and expose only append/read, or return `List.copyOf` on freeze.

**Fix:** don't extend `ArrayList`; hold a private list, expose `append`/read methods, and return an unmodifiable snapshot when frozen.
**What a strong senior would say:** *"Subclassing ArrayList to make it read-only is a Liskov trap — callers treating it as a List will hit surprise exceptions, and you've only plugged one of many mutation paths. I'd use composition: wrap a list, expose just append and read, and hand out an unmodifiable view once frozen."*
**Interviewer is testing:** LSP and composition-over-inheritance.
**Expected:** LSP + incomplete override. **Bonus:** iterator.remove/set/clear paths. **Mention first:** the substitutability break.

## Topic 2 · Exercise 1 — Shipping cost calculator
**Findings**
- 🟡 **DESIGN CONCERN — OCP.** Every new carrier edits this method (and its tests), risking regressions in existing carriers. String-typed carriers are also fragile.
- 🟢 Money math in `BigDecimal` is fine.

**Fix:** a `CarrierRateStrategy` per carrier resolved from a map/enum, or carrier rate config in data; the method becomes `strategies.get(carrier).cost(shipment)`.
**What a strong senior would say:** *"This works but violates open/closed — each carrier onboarding means editing this branch and re-testing all carriers. I'd model carriers as strategies (or config rows) keyed by carrier, so adding one is additive and isolated. Given carriers change 'regularly,' that pays off quickly."*
**Interviewer is testing:** OCP and pragmatic extensibility (not over-engineering a 3-branch method unless change is frequent — which the scenario states).
**Mention first:** the extensibility cost, tied to "onboarded regularly."

## Topic 2 · Exercise 2 — Repository interface
**Findings**
- 🟠 **DESIGN CONCERN — ISP violation.** One fat interface forces the read-only implementation to stub `save`/`delete`/`bulkImport`/`reindex` with `UnsupportedOperationException` — a Liskov landmine (callers of `DataRepository` can't rely on `save` working). Split into role interfaces: `ReadRepository` (findById/findAll), `WriteRepository`, `IndexMaintenance`, `CacheControl`; implementations compose only what they support.

**Fix:** segregate interfaces; `CachedReadRepository implements ReadRepository, CacheControl`.
**What a strong senior would say:** *"The read-only impl throwing on half the methods is the classic sign of a fat interface — it's forced to pretend to be something it isn't, and any code holding the interface can call `save` and blow up at runtime. I'd segregate into read/write/maintenance interfaces so each implementation only exposes what it truly supports."*
**Interviewer is testing:** ISP and the ISP↔LSP link.
**Mention first:** the forced `UnsupportedOperationException` stubs.

## Topic 3 · Exercise 1 — "Immutable" money range
**Findings**
- 🟠 **DEFINITE BUG — Immutability.** `final` on the field only prevents reassigning the reference. The constructor stores the caller's list directly and `getTiers()` returns it, so external code can mutate the list before or after construction — the object isn't immutable, and it's being shared across threads/cached (unsafe publication of mutations).

**Fix:** `this.tiers = List.copyOf(tiers)` in the constructor (immutable copy); `getTiers()` then safely returns it.
**What a strong senior would say:** *"`final` here only freezes the reference, not the list contents. Since we take the caller's list and hand it back out, it's mutable in both directions — and this is cached and shared across threads, so that's a real hazard. `List.copyOf` on the way in makes it genuinely immutable."*
**Interviewer is testing:** deep vs shallow immutability; defensive copy.
**Mention first:** the shared-mutable-list escape.

## Topic 3 · Exercise 2 — Frozen config
**Findings**
- 🟠 **DEFINITE BUG — Immutability (half-done).** The defensive copy on the way *in* is correct and blocks the constructor-aliasing attack — but `getSettings()` returns the internal map by reference, so callers can still mutate the "frozen" config. The inbound copy alone does **not** fully protect it.

**Fix:** return `Collections.unmodifiableMap(settings)` (or `Map.copyOf` on the way in and return it), so both directions are protected.
**What a strong senior would say:** *"The inbound copy is right and stops external aliasing at construction, but the getter hands out the live map, so it's still mutable. I'd either wrap the getter in `unmodifiableMap` or store a `Map.copyOf`. Defensive copying has to cover both the boundary in and the boundary out."*
**Interviewer is testing:** whether you notice the copy is only half the protection (avoids the false-positive of "copy present → safe").
**Mention first:** the leaking getter.

## Topic 4 · Exercise 1 — Order record with items
**Findings**
- 🟠 **DEFINITE BUG — Shallow immutability.** Records are only shallowly immutable. The `lineItems` list is stored by reference, so `items.add("C")` later mutates the "snapshot" the audit system assumed frozen.

**Fix:** compact constructor with a defensive copy: `public OrderSnapshot { lineItems = List.copyOf(lineItems); }` (and it's unmodifiable on read).
**What a strong senior would say:** *"A record gives us immutability of the *references*, not of a mutable list inside it — so this snapshot changes when the caller later mutates the original list. I'd add a compact constructor doing `List.copyOf`, which both copies and makes it unmodifiable."*
**Interviewer is testing:** the common misconception that records are deeply immutable.
**Mention first:** the shared list.

## Topic 4 · Exercise 2 — Signature record
**Findings**
- 🟠 **DEFINITE BUG — Arrays in records break value equality.** A record's generated `equals`/`hashCode` use `Arrays`-by-reference for array components (identity, `==`), not contents. So two `Signature`s with equal byte contents are **not** equal and hash differently — `containsKey` returns `false`, defeating dedup/keys.

**Fix:** don't use a raw array component for value semantics — override `equals`/`hashCode` with `Arrays.equals`/`Arrays.hashCode` (and defensively clone the array in a compact constructor), or model the payload as a `List<Byte>`/wrapper with proper value semantics.
**What a strong senior would say:** *"The subtle one: records give arrays *reference* equality, so two signatures with identical bytes won't be equal or hash the same — the map lookup silently misses. I'd override equals/hashCode using `Arrays.equals`/`Arrays.hashCode` and clone the array defensively, since arrays are also mutable."*
**Interviewer is testing:** deep knowledge of record-generated equals with array components — a genuine trap.
**Mention first:** array reference-equality.

## Topic 5 · Exercise 1 — Product value object
**Findings**
- 🔴 **DEFINITE BUG — Missing `hashCode`.** `equals` is overridden (by `sku`) but `hashCode` is not, breaking the contract. In `HashSet`/`HashMap`, two products with the same `sku` land in different buckets, so lookups/dedup fail unpredictably.
- 🟡 Also verify `equals` intent: equality by `sku` only means two products with the same sku but different names are "equal" — usually right for a value object keyed on sku, but worth confirming.

**Fix:** add `@Override public int hashCode() { return Objects.hash(sku); }` consistent with `equals`.
**What a strong senior would say:** *"Overriding equals without hashCode breaks the contract — in any hash-based collection these products get lost. I'd add a matching hashCode over the same field, `sku`. I'd also confirm that sku-only equality is the intended value semantics."*
**Interviewer is testing:** the equals/hashCode contract reflex.
**Mention first:** missing hashCode.

## Topic 5 · Exercise 2 — Entity identity
**Findings**
- 🟠 **DEFINITE BUG (context-dependent) — Entity equality on generated id.** Before persistence, `id` is `null` for all new customers, so **all unsaved customers are equal** and share a hashCode → a `Set` collapses them to one, and after they're persisted their identity changes (id assigned), which also breaks any set they're already in. This is the well-known Hibernate identity trap.

**Fix:** use a stable **business key** (e.g. `email`) for `equals`/`hashCode`, or a UUID assigned at construction; avoid a DB-generated id in equality. If you must use id, don't put transient entities in hash structures.
**What a strong senior would say:** *"Basing equality on a generated id is a classic entity trap: before flush every new entity has a null id, so they're all 'equal' and collapse in a Set, and once persisted their hashCode changes underneath any set they're in. I'd key equals/hashCode on a stable natural key like email, or a client-assigned UUID."*
**Interviewer is testing:** JPA identity semantics — strong senior signal.
**Mention first:** null-id collapse before persistence.

## Topic 6 · Exercise 1 — Status comparison
**Findings**
- 🔴 **DEFINITE BUG — `==` on strings.** `status == "ACTIVE"` compares references, not contents. For strings from an external API (not interned), this is almost always `false`, so `isActive` silently returns wrong results. Use `.equals` (ideally `"ACTIVE".equals(status)` to null-guard, or `equalsIgnoreCase`).

**Fix:** `return "ACTIVE".equals(status);`
**What a strong senior would say:** *"`==` compares references — for a status string coming off an external API it won't be interned, so this is effectively always false. I'd use `\"ACTIVE\".equals(status)`, which also null-guards. Honestly I'd model status as an enum to avoid stringly-typed comparisons entirely."*
**Interviewer is testing:** the `==`-vs-`equals` fundamental and null-safety.
**Mention first:** reference comparison.

## Topic 6 · Exercise 2 — Normalizing input
**Findings**
- 🔴 **DEFINITE BUG — Ignoring return values.** `String` is immutable; `trim()` and `toLowerCase()` return **new** strings and don't mutate `email`. The method returns the original untouched. Chain and return the result.

**Fix:** `return email.trim().toLowerCase();` (guard null; consider `Locale.ROOT` for `toLowerCase`).
**What a strong senior would say:** *"Strings are immutable, so `trim()` and `toLowerCase()` produce new strings that are thrown away — the method returns the input unchanged. I'd chain and return them, null-guard, and use `toLowerCase(Locale.ROOT)` to avoid locale surprises like the Turkish-i."*
**Interviewer is testing:** String immutability understanding.
**Mention first:** discarded return values.

## Topic 7 · Exercise 1 — Shared report builder
**Findings**
- 🔴 **DEFINITE BUG — Concurrency.** `StringBuilder` is **not** thread-safe; a shared field appended from multiple worker threads causes data races — lost/interleaved/corrupted content, and even `ArrayIndexOutOfBounds`/`NPE` internally. In a singleton service this shared mutable field is the problem.

**Fix:** avoid shared mutable state — have each worker produce its own string and merge, or use a thread-safe/concurrent collection (e.g. a `ConcurrentLinkedQueue<String>` then join), or synchronize (worst option). Consider whether aggregation even needs to be a shared field.
**What a strong senior would say:** *"A shared `StringBuilder` mutated by multiple threads is a data race — output gets interleaved or corrupted, and you can even get internal exceptions. I'd have each worker build locally and collect results into a concurrent queue to join at the end, rather than sharing one builder. `StringBuffer` would be safe per-append but still wouldn't give coherent multi-append output."*
**Interviewer is testing:** StringBuilder thread-unsafety; designing away shared state.
**Mention first:** the shared-builder race.

## Topic 7 · Exercise 2 — Local join
**Findings**
- ⚪ **NOT A BUG.** The `StringBuilder` is a **local variable** — each call gets its own, never shared across threads — so concurrent calls are perfectly safe. This is the correct, idiomatic choice (faster than `StringBuffer`). Don't flag it.
- 🟢 NIT: `String.join(",", fields)` is cleaner, but the builder is fine.

**What a strong senior would say:** *"This is fine — the builder is local to each invocation, so there's no sharing and no thread-safety issue even under concurrent calls. I wouldn't change it for safety; if anything I'd use `String.join` for readability, but that's cosmetic."*
**Interviewer is testing:** not flagging a local `StringBuilder` as a concurrency bug (false-positive resistance).
**Mention first:** state it's correct and why (local = not shared).

## Topic 8 · Exercise 1 — Legacy message builder
**Findings**
- 🟢 **NIT / minor perf.** `StringBuffer` is synchronized; in a single-threaded local build the locking is unnecessary overhead. Prefer `StringBuilder` (or plain concatenation, which the compiler optimizes for a single expression). Not a correctness issue.

**Fix:** `StringBuilder` (or `return "Hello " + name + ", you have " + count + " messages";`).
**What a strong senior would say:** *"Functionally correct — this is a nit. `StringBuffer` pays for synchronization we don't need in a local single-threaded build; `StringBuilder` (or even simple concatenation, which javac turns into one builder) is the idiomatic choice. Not something I'd hold up a PR for."*
**Interviewer is testing:** knowing StringBuffer vs StringBuilder and *correctly rating it a nit* (prioritization).
**Mention first:** flag it as a nit, don't over-weight it.

## Topic 8 · Exercise 2 — "Thread-safe" accumulator
**Findings**
- 🟠 **DEFINITE BUG — Atomicity vs per-op safety (misleading comment).** Each individual `append` is synchronized, but the **sequence** of four appends is not atomic. Under concurrency, tokens from different events interleave (e.g. `user1:` then `user2` slips in), so the logical `user:action;` records get scrambled even though no low-level corruption occurs. The comment's "so this is thread-safe" is wrong at the semantic level.

**Fix:** build the whole record first and append once (`log.append(user + ":" + action + ";")` — still interleaves at record boundaries only), or synchronize the whole `record` method, or better, collect structured entries in a concurrent queue rather than a shared string.
**What a strong senior would say:** *"The comment conflates two things — `StringBuffer` makes each append atomic, but not the group of four, so records from different threads interleave and you get garbled `user:action` pairs. If we must share, I'd append the full token in one call or synchronize the method; ideally I'd store structured entries in a concurrent queue instead of a shared buffer."*
**Interviewer is testing:** the atomicity-of-compound-operations concept and skepticism of the comment.
**Mention first:** compound-operation interleaving.

## Topic 9 · Exercise 1 — Removing while iterating
**Findings**
- 🔴 **DEFINITE BUG — ConcurrentModificationException.** Structurally modifying `items` (via `List.remove`) during the enhanced-for iteration throws `CME` (or silently skips elements in edge cases). 

**Fix:** `items.removeIf(Item::isExpired);` (or iterate with an explicit `Iterator` and call `it.remove()`).
**What a strong senior would say:** *"Removing from the list you're iterating with a for-each throws ConcurrentModificationException — the fail-fast iterator detects the structural change. I'd use `removeIf`, which is both correct and clearer, or an explicit iterator's `remove` if I needed more control."*
**Interviewer is testing:** the CME/iterator-invalidation fundamental and the modern `removeIf` fix.
**Mention first:** the CME.

## Topic 9 · Exercise 2 — Building a mutable list
**Findings**
- 🟠 **DEFINITE BUG — Fixed-size list.** `Arrays.asList(...)` returns a **fixed-size** list backed by an array; the caller's `add("priority")` throws `UnsupportedOperationException`. (`List.of(...)` would be fully immutable — same failure.)

**Fix:** return a resizable copy: `return new ArrayList<>(Arrays.asList("new", "unreviewed"));` (or `new ArrayList<>(List.of(...))`).
**What a strong senior would say:** *"`Arrays.asList` gives a fixed-size view, so the caller's `add` throws `UnsupportedOperationException`. Since the contract is 'starter set the caller extends,' I'd return a `new ArrayList<>(...)`. If I wanted it read-only, I'd document that and return `List.of`, but then the caller code is wrong."*
**Interviewer is testing:** knowing the fixed-size/immutable list gotchas and matching the contract.
**Mention first:** the fixed-size add failure.

## Topic 10 · Exercise 1 — Mutable key
**Findings**
- 🟠 **DEFINITE BUG — Mutating a key after insertion.** Since `hashCode` depends on `plan`, changing `plan` to `PREMIUM` after `put` changes the key's hashCode, so `get(key)` looks in the wrong bucket and returns `null` — the entry becomes unreachable (a subtle leak).
- 🟡 Design: keys should be immutable.

**Fix:** make `UserKey` immutable, or key by an immutable/stable field only (e.g. `userId`), storing `plan` inside the value.
**What a strong senior would say:** *"The moment we mutate a field that participates in hashCode after inserting, the entry is effectively lost — it hashes to a different bucket, so lookups miss and it leaks. Map keys must be immutable. I'd either freeze `UserKey` or key on the stable `userId` and move `plan` into the value."*
**Interviewer is testing:** understanding how HashMap uses hashCode at insert vs lookup.
**Mention first:** mutable-key unreachability.

## Topic 10 · Exercise 2 — Assuming iteration order
**Findings**
- 🟠 **DEFINITE BUG — HashMap has no order guarantee.** `keySet()` order for a `HashMap` is unspecified and can differ from insertion order (and across JVM versions/resizes). The required `starters, mains, desserts` order is not guaranteed.

**Fix:** use `LinkedHashMap` (insertion order) — or, if the order is truly a domain concept, an explicit ordered list / `TreeMap` with a defined comparator.
**What a strong senior would say:** *"HashMap makes no ordering promise, so relying on keys coming out in insertion order is a latent bug that can flip on a resize or JVM change. Since order is a requirement here, I'd use `LinkedHashMap` to preserve insertion order explicitly."*
**Interviewer is testing:** knowing which Map guarantees order.
**Mention first:** unordered iteration.

## Topic 11 · Exercise 1 — First-writer registration
**Findings**
- 🟠 **DEFINITE BUG — Check-then-act race.** `containsKey` then `put` is not atomic; two threads can both see "absent" and both `put` and both log "registered," double-registering and double-auditing. `ConcurrentHashMap` makes each op atomic, not the pair.

**Fix:** `computeIfAbsent` (atomic) or `putIfAbsent` and act only if it returned null:
```java
devices.computeIfAbsent(id, k -> { auditService.log("registered " + k); return device; });
```
(Note: keep the audit side-effect's placement in mind — `computeIfAbsent`'s function runs at most once per absent key, which is exactly what we want here.)
**What a strong senior would say:** *"This is check-then-act — two ingestion threads can both pass `containsKey` and both register and audit the same device. `ConcurrentHashMap` only makes single operations atomic. I'd use `computeIfAbsent`, which runs the mapping (and the audit) exactly once for a new key."*
**Interviewer is testing:** compound-operation atomicity on concurrent maps.
**Mention first:** the check-then-act race.

## Topic 11 · Exercise 2 — Per-key counter
**Findings**
- 🟠 **DEFINITE BUG — Read-modify-write race.** `get` then `put(current+1)` is a lost-update race across threads even on a `ConcurrentHashMap` — two threads read the same value and both write back the same increment, dropping counts.

**Fix:** atomic combinators — `hits.merge(endpoint, 1, Integer::sum)` or `hits.computeIfAbsent(endpoint, k -> new AtomicInteger()).incrementAndGet()` (or a `LongAdder` value).
**What a strong senior would say:** *"Using the concurrent map doesn't help when we do read-modify-write across two calls — that's a lost-update race and counts will be undercounted under load. I'd use `merge(endpoint, 1, Integer::sum)`, which performs the update atomically, or hold `AtomicInteger`/`LongAdder` values."*
**Interviewer is testing:** recognizing that CHM value-updates still need atomic combinators.
**Mention first:** the lost-update.

## Topic 12 · Exercise 1 — Positional access
**Findings**
- 🟠 **PERFORMANCE CONCERN — Wrong list type.** `LinkedList.get(i)` is O(n) (traverses from an end), so indexed iteration is O(n²). With tens of thousands of tracks this is a real hotspot. Either use an enhanced-for / iterator (O(n) on a LinkedList), or better, use `ArrayList` for index access.

**Fix:** iterate with for-each (`for (Track t : tracks)`), or accept/use `List<Track>` backed by `ArrayList`.
**What a strong senior would say:** *"Indexed `get` on a LinkedList is linear, so this loop is O(n²) — at tens of thousands of tracks it'll show up. The minimal fix is to iterate with an enhanced-for so it's a single O(n) traversal; the better fix is to not use LinkedList for index-accessed data at all — ArrayList is almost always the right default."*
**Interviewer is testing:** cost model of LinkedList vs ArrayList.
**Mention first:** O(n²) from LinkedList indexing.

## Topic 12 · Exercise 2 — Work queue
**Findings**
- 🟡 **PERFORMANCE CONCERN — Wrong structure for the access pattern.** `ArrayList.add(0, t)` and `remove(0)` are O(n) (shift every element) — on a high-throughput front-insert/front-remove queue that's O(n) per operation. Use `ArrayDeque` (`addFirst`/`pollFirst`, amortized O(1)), or a proper queue.

**Fix:** `ArrayDeque<Task>` with `addFirst`/`pollFirst`; if this is a priority scenario, a `PriorityQueue`; if concurrent, a concurrent deque/blocking queue.
**What a strong senior would say:** *"Both operations here are front-of-list on an ArrayList, which shifts the whole array each time — O(n) per op at high throughput. `ArrayDeque` gives amortized O(1) at both ends. If 'highest priority first' is really a priority order rather than LIFO/FIFO, a `PriorityQueue` is more honest."*
**Interviewer is testing:** matching structure to access pattern; ArrayDeque awareness.
**Mention first:** O(n) front operations.

## Topic 13 · Exercise 1 — Sorted unique report
**Findings**
- 🟠 **DEFINITE BUG (vs requirement) — HashSet is unordered.** The UI expects alphabetical order, but `HashSet` iteration order is unspecified. Requirement unmet.

**Fix:** return a `TreeSet<>()` (natural alphabetical order), or sort at the boundary. Use `LinkedHashSet` only if *insertion* order were wanted (it isn't here).
**What a strong senior would say:** *"HashSet gives dedup but no order, and the UI needs alphabetical — so this is a requirements bug. I'd use a `TreeSet` for natural ordering, or sort the result explicitly if I want to keep the set type. I'd confirm the ordering is really the service's responsibility versus the UI's."*
**Interviewer is testing:** Set ordering guarantees and reading the requirement.
**Mention first:** unordered output vs the alphabetical requirement.

## Topic 13 · Exercise 2 — Deduplicating by a comparator
**Findings**
- 🔴 **DEFINITE BUG — TreeSet dedups by the comparator, not equals.** `TreeSet` treats elements as equal when the **comparator** returns 0. Comparing only by `timestamp` means two *different* events sharing a timestamp are considered duplicates and one is silently dropped — data loss. The requirement (same only if id AND timestamp) isn't met.

**Fix:** make the comparator consistent with the intended equality: `Comparator.comparing(Event::getTimestamp).thenComparing(Event::getId)` (so distinct ids at the same time are kept), or dedup with a `HashSet`/`LinkedHashSet` using proper `equals`/`hashCode` and sort afterward.
**What a strong senior would say:** *"TreeSet decides equality via the comparator, so comparing on timestamp alone silently drops distinct events that happen to share a timestamp — that's data loss, not dedup. I'd make the comparator include id as a tiebreaker so it matches the real equality rule, or dedup with a hash set on proper equals/hashCode and sort separately."*
**Interviewer is testing:** the TreeSet-comparator-vs-equals contract — a real senior trap.
**Mention first:** silent drop of same-timestamp events.

## Topic 14 · Exercise 1 — Recently-used cache
**Findings**
- 🟠 **DEFINITE BUG — Not actually LRU.** Two problems: (1) `get` doesn't record recency, so nothing tracks "recently used"; (2) eviction removes `keySet().iterator().next()`, which for a `HashMap` is an arbitrary entry, not the least-recently-used. It's a random-eviction fixed-size map, not LRU. Also not thread-safe if shared.

**Fix:** `LinkedHashMap` in access-order mode with `removeEldestEntry`:
```java
new LinkedHashMap<String, byte[]>(16, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry<String, byte[]> e) { return size() > 100; }
};
```
or use Caffeine (`maximumSize(100)`), which is also thread-safe.
**What a strong senior would say:** *"This isn't LRU — `get` doesn't update recency and eviction picks an arbitrary HashMap entry. I'd use an access-order `LinkedHashMap` with `removeEldestEntry`, or Caffeine if it's shared across threads since Caffeine handles concurrency and gives real LRU/other policies."*
**Interviewer is testing:** knowing how to build/pick an LRU; access-order LinkedHashMap.
**Mention first:** it's not LRU at all.

## Topic 14 · Exercise 2 — Synchronized map iteration
**Findings**
- 🟠 **DEFINITE BUG — Iteration not synchronized.** `Collections.synchronizedMap` synchronizes individual method calls, but **iterating** the `entrySet` must be done inside a `synchronized (counters) { ... }` block, or a concurrent write during iteration throws `ConcurrentModificationException`.

**Fix:** wrap the loop in `synchronized (counters) { for (...) {...} }`, or switch to `ConcurrentHashMap`, whose iterators are weakly-consistent and don't throw (better for a live snapshot).
**What a strong senior would say:** *"`synchronizedMap` only guards single operations — iteration is a compound action, so a concurrent update during this loop can throw ConcurrentModificationException. Either I synchronize on the map around the loop, or, since this is a live snapshot read under concurrent writes, I'd use `ConcurrentHashMap` whose iterators are weakly consistent and won't throw."*
**Interviewer is testing:** the synchronizedMap iteration caveat and CHM's iterator semantics.
**Mention first:** unsynchronized iteration.

## Topic 15 · Exercise 1 — Parallel enrichment
**Findings**
- 🔴 **DEFINITE BUG — Race on shared list.** `parallelStream().forEach(results::add)` mutates a non-thread-safe `ArrayList` from multiple threads → lost/corrupted elements, possible internal exceptions.
- 🟠 **Appropriateness — blocking I/O on the common pool.** `parallelStream` runs on the shared common ForkJoinPool; doing blocking HTTP there starves the whole JVM's parallel work and gives you only `cores`-worth of concurrency anyway (wrong tool for I/O).

**Fix:** don't side-effect into a shared list — `orders.stream().map(remoteClient::enrich).collect(toList())`; for concurrency on I/O use a dedicated executor with `CompletableFuture` (bounded), not parallel streams.
**What a strong senior would say:** *"Two issues: collecting into a shared ArrayList from a parallel stream is a data race — I'd use `map(...).collect(toList())` instead of `forEach` with a side effect. And parallel streams are the wrong tool for blocking I/O anyway — they run on the shared common pool and can starve the whole app. For parallel HTTP I'd use a dedicated bounded executor with CompletableFutures."*
**Interviewer is testing:** stream side-effect safety + parallelStream/common-pool pitfalls.
**Mention first:** the shared-list race, then the I/O-on-common-pool point.

## Topic 15 · Exercise 2 — Indexing by key
**Findings**
- 🟠 **DEFINITE BUG — `toMap` throws on duplicate keys.** `Collectors.toMap` with no merge function throws `IllegalStateException: Duplicate key` when two users share an email — and the scenario says duplicates happen. Crashes the request.

**Fix:** supply a merge function to decide the winner: `toMap(User::getEmail, u -> u, (a, b) -> a)` (keep first) — or `(a,b) -> b` (keep last), per requirement. If duplicates are truly invalid, catch and report rather than 500.
**What a strong senior would say:** *"`toMap` without a merge function throws on duplicate keys, and we're told duplicates occur — so this will blow up in production. I'd add a merge function encoding the intended resolution, keep-first or keep-last, or if duplicate emails are a data-quality error, handle them explicitly rather than letting it throw."*
**Interviewer is testing:** the `toMap` duplicate-key gotcha and requirement-driven merge.
**Mention first:** the duplicate-key exception.

## Topic 16 · Exercise 1 — Unwrapping a lookup
**Findings**
- 🔴 **DEFINITE BUG — `Optional.get()` without presence check.** If the account doesn't exist, `findById` returns empty and `.get()` throws `NoSuchElementException` (opaque 500). The Optional's whole point is being ignored.

**Fix:** `return accountRepository.findById(accountId).map(Account::getBalance).orElseThrow(() -> new AccountNotFoundException(accountId));`
**What a strong senior would say:** *"Calling `.get()` straight away defeats the Optional and throws a confusing NoSuchElementException on a missing account. I'd map to the balance and `orElseThrow` a domain-specific not-found exception that maps to a 404, so callers get a meaningful error."*
**Interviewer is testing:** idiomatic Optional handling.
**Mention first:** unguarded `get()`.

## Topic 16 · Exercise 2 — Default from a fallback source
**Findings**
- 🟠 **DEFINITE BUG — `orElse` always evaluates its argument.** `orElse(remoteConfig.loadDefaultTimeout())` invokes the **remote network call on every invocation**, even when the local timeout is present — the argument is evaluated eagerly regardless. Wasteful and adds latency/failure surface to the happy path.

**Fix:** `orElseGet(() -> remoteConfig.loadDefaultTimeout())` — the supplier runs only when the Optional is empty.
**What a strong senior would say:** *"`orElse` evaluates its argument eagerly, so we make the remote default call every time even when the local value exists — that's needless latency and a failure dependency on the hot path. `orElseGet` with a supplier defers it to the empty case. This one bites people constantly."*
**Interviewer is testing:** the `orElse` vs `orElseGet` eager-evaluation trap.
**Mention first:** the always-executed remote call.

## Topic 17 · Exercise 1 — Raw type processor
**Findings**
- 🟡 **DESIGN/Safety — Raw types.** `List handlers = new ArrayList()` is a raw type: no compile-time type safety, `addHandler` could be bypassed to insert non-`Handler` objects elsewhere, and the cast `(Handler) o` risks `ClassCastException` at runtime. Parameterize.

**Fix:** `private final List<Handler> handlers = new ArrayList<>();` — drops the cast and gives compile-time safety.
**What a strong senior would say:** *"Raw types opt out of generic type checking, so nothing stops a non-Handler ending up in this list and the runtime cast turning into a ClassCastException. I'd type it as `List<Handler>`, which removes the cast and moves the guarantee to compile time. Raw types are legacy — no reason to keep them here."*
**Interviewer is testing:** understanding raw types and their runtime risk.
**Mention first:** the raw type / unchecked cast.

## Topic 17 · Exercise 2 — Summing numbers
**Findings**
- 🟡 **DESIGN — Invariant generics limit the API.** `List<Number>` does **not** accept `List<Integer>` (generics are invariant), so callers with typed lists can't use it — the scenario's call won't compile. Use a bounded wildcard.

**Fix:** `public static double sum(List<? extends Number> numbers)` — now `List<Integer>`, `List<Double>`, etc. all work (PECS: producer-extends).
**What a strong senior would say:** *"Generics are invariant, so `List<Number>` won't accept a `List<Integer>` — the API is unusable for typed lists. By PECS, this list is a producer of Numbers, so I'd widen the parameter to `List<? extends Number>`, which accepts any Number subtype list."*
**Interviewer is testing:** wildcards / PECS and API ergonomics.
**Mention first:** the invariance blocking callers.

## Topic 18 · Exercise 1 — Wrapping and losing the cause
**Findings**
- 🟠 **DEFINITE BUG — Lost exception cause.** `throw new RuntimeException("Import failed")` discards the original `IOException` — no stack trace of the real failure, making production debugging painful. Always chain the cause.

**Fix:** `throw new ImportException("Import failed: " + path, e);` (custom unchecked type, cause preserved).
**What a strong senior would say:** *"Wrapping is fine, but dropping the cause is the bug — we lose the original IOException and its stack trace, so on-call sees 'Import failed' with no reason. I'd pass `e` as the cause and use a specific exception type rather than bare RuntimeException, so callers can catch it meaningfully."*
**Interviewer is testing:** exception chaining discipline.
**Mention first:** the dropped cause.

## Topic 18 · Exercise 2 — Catching everything
**Findings**
- 🟠 **DEFINITE BUG — Catching `Throwable`.** This swallows `Error`s too — `OutOfMemoryError`, `StackOverflowError`, `NoClassDefFoundError` — which are usually unrecoverable and should not be caught and "continued" past; doing so can leave the JVM in a corrupt state while pretending it's fine. Catch `Exception` (or specific types) instead.
- 🟢 Also rethrow/handle `InterruptedException` specially if it can occur.

**Fix:** `catch (Exception e)` (not `Throwable`); let `Error`s propagate.
**What a strong senior would say:** *"Catching `Throwable` scoops up Errors like OOM and StackOverflow and pretends the job just failed normally — but the JVM may be in an unrecoverable state, so continuing is dangerous. I'd narrow this to `Exception`. Keeping the scheduler alive across *application* failures is the goal; swallowing VM errors isn't."*
**Interviewer is testing:** the Throwable/Error distinction.
**Mention first:** catching Error.

## Topic 19 · Exercise 1 — Manual close in finally
**Findings**
- 🟠 **DEFINITE BUG — NPE risk + suppressed exception.** If `Files.newBufferedWriter` throws, `w` stays null and `finally` does `null.close()` → NPE that masks the real error. Also, if a `write` throws *and* `close()` throws, the close exception replaces the original (no suppression). Manual try/finally is error-prone here.

**Fix:** try-with-resources handles null-safety and suppresses secondary exceptions correctly:
```java
try (Writer w = Files.newBufferedWriter(path)) {
    for (String r : rows) { w.write(r); w.write('\n'); }
}
```
**What a strong senior would say:** *"Two problems with the manual finally: if opening the writer fails, `w` is null and `close()` NPEs, hiding the real cause; and if both the write and the close throw, the close exception overwrites the original. Try-with-resources fixes both — it null-guards and adds the close failure as a *suppressed* exception rather than masking the primary one."*
**Interviewer is testing:** why TWR exists — suppressed exceptions and null-safety.
**Mention first:** the NPE-in-finally / masked exception.

## Topic 19 · Exercise 2 — Reading a response body
**Findings**
- ⚪ **NOT A BUG.** This is correct: both resources are in try-with-resources and close in reverse order; closing the `BufferedReader` closes the underlying `InputStreamReader` and `InputStream`, so there's no double-close problem and no leak. Charset is explicit. Nothing to fix.
- 🟢 NIT: declaring `in` separately is slightly redundant since the reader would close it, but it's harmless and arguably clearer.

**What a strong senior would say:** *"This one's fine — both are in try-with-resources, they close in reverse order, and closing the reader chains down to the stream, so there's no leak or double-close. Charset is pinned to UTF-8. I wouldn't change it."*
**Interviewer is testing:** not inventing a leak where TWR already handles it (false-positive resistance).
**Mention first:** state it's correct.

## Topic 20 · Exercise 1 — Scheduling across regions
**Findings**
- 🟠 **DEFINITE BUG — `LocalDateTime` has no time zone.** Storing a zone-less `LocalDateTime` for a global "moment" and comparing to `LocalDateTime.now()` (server local, here UTC) means the reminder fires at the wrong absolute instant for users in other zones, and the semantics depend on server time zone. A point in time should be an `Instant` (UTC), with the user's zone captured separately if needed for display.

**Fix:** store `Instant fireAt` (absolute), compare with `Instant.now()`; convert to/from the user's `ZoneId` at the input/display boundary.
**What a strong senior would say:** *"`LocalDateTime` is a wall-clock value with no zone, so comparing it to server-now conflates different instants across regions — reminders fire at the wrong absolute time. For 'a moment that should happen,' I'd model it as an `Instant` in UTC and only apply the user's zone at input and display. If the requirement is 'same wall-clock time regardless of zone,' that's a different model I'd clarify."*
**Interviewer is testing:** Instant vs LocalDateTime vs ZonedDateTime modeling.
**Good clarifying question:** *"Should this fire at a fixed global instant, or at a wall-clock time in the user's zone?"* — **Why it matters:** picks Instant vs zoned scheduling.
**Mention first:** zone-less instant.

## Topic 20 · Exercise 2 — Same time tomorrow
**Findings**
- 🟠 **DEFINITE BUG — Duration vs Period across DST.** `Duration.ofHours(24)` adds exactly 24 hours of elapsed time. On a DST transition day (23 or 25 hours of wall-clock), the result lands an hour off the intended same-wall-clock-time. For "same local time tomorrow," add a **`Period`/calendar day**: `current.plusDays(1)`, which `ZonedDateTime` resolves with DST rules.

**Fix:** `return current.plusDays(1);`
**What a strong senior would say:** *"`Duration.ofHours(24)` is 24 elapsed hours, but a DST day isn't 24 wall-clock hours, so billing would drift by an hour twice a year. Since we want the same local time tomorrow, I'd use `plusDays(1)`, which is calendar-based and lets `ZonedDateTime` apply the DST offset correctly."*
**Interviewer is testing:** Duration (exact time) vs Period (calendar) and DST awareness.
**Mention first:** the DST hour drift.

## Topic 21 · Exercise 1 — Comparing amounts
**Findings**
- 🔴 **DEFINITE BUG — `BigDecimal.equals` is scale-sensitive.** `equals` returns false for `100.0` vs `100.00` (same value, different scale). Since amounts come from sources with differing scales, exact-match payments will be wrongly rejected. Use `compareTo(...) == 0` for numeric equality.

**Fix:** `return payment.compareTo(invoiceTotal) == 0;`
**What a strong senior would say:** *"`BigDecimal.equals` compares scale as well as value, so `100.0` and `100.00` are 'not equal' even though the amounts match — and we're explicitly getting different scales. That means valid payments get rejected. I'd compare with `compareTo == 0`, which is value-based. This is the single most common BigDecimal mistake."*
**Interviewer is testing:** the equals-vs-compareTo scale trap.
**Mention first:** scale-sensitive equals.

## Topic 21 · Exercise 2 — Splitting a bill
**Findings**
- 🔴 **DEFINITE BUG — Non-terminating division throws.** `total.divide(new BigDecimal(people))` with no scale/RoundingMode throws `ArithmeticException` whenever the result is a non-terminating decimal (e.g. €100 / 3). Must specify scale and `RoundingMode`.
- 🟡 **Edge cases:** `people == 0` → ArithmeticException; and even/odd rounding means the sum of shares may not equal the total (need a remainder-allocation strategy).

**Fix:** `total.divide(BigDecimal.valueOf(people), 2, RoundingMode.HALF_UP);` guard `people > 0`; consider distributing the rounding remainder to one payer so shares sum to the total.
**What a strong senior would say:** *"Dividing without a scale and RoundingMode throws on any non-terminating result like 100/3 — a guaranteed production exception. I'd divide with scale 2 and an explicit RoundingMode, guard zero people, and for money I'd also handle the rounding remainder so the shares actually add back up to the total."*
**Interviewer is testing:** BigDecimal division requiring rounding; money-split remainder awareness.
**Mention first:** the divide-without-rounding exception.

## Topic 22 · Exercise 1 — Persisting order status
**Findings**
- 🔴 **DEFINITE BUG — `@Enumerated(ORDINAL)`.** Persisting by ordinal stores 0,1,2,3. Inserting a new value in the middle (or reordering) shifts every subsequent ordinal, so existing rows now map to the **wrong** status — silent, catastrophic data corruption that's hard to detect.

**Fix:** `@Enumerated(EnumType.STRING)` (stores the name; safe against reordering, only breaks on rename which is explicit). Trade-off: slightly more storage and rename requires a data migration.
**What a strong senior would say:** *"ORDINAL persists the position, so the moment someone inserts a status in the middle of the enum, every existing row silently shifts to the wrong status — a nasty, invisible data-corruption bug. I'd switch to `EnumType.STRING`, which is stable against reordering. The cost is a bit more storage and that renames need a migration, which is a fair trade for correctness."*
**Interviewer is testing:** the ORDINAL-vs-STRING persistence trap — strong senior signal.
**Mention first:** ordinal reorder corruption.

## Topic 22 · Exercise 2 — Enum with counters
**Findings**
- 🟠 **DEFINITE BUG — Mutable enum = shared global state.** Enum constants are singletons, so `usageCount` is process-wide mutable state; `use()` from multiple threads is a race (non-atomic `++`), and the counter is shared across every unrelated caller and survives for the JVM's life. Enums should be immutable.

**Fix:** move usage counting out of the enum into a dedicated (thread-safe) metrics component keyed by currency (`Map<Currency, LongAdder>` / Micrometer counter). Keep the enum stateless.
**What a strong senior would say:** *"Enum constants are singletons, so this counter is global mutable state — every caller shares it, it never resets, and `++` across threads is a race. Enums should be immutable value types. I'd track usage in a separate metrics component keyed by currency, ideally with `LongAdder` or the metrics library, and leave the enum stateless."*
**Interviewer is testing:** enum singleton semantics and why mutable enums are wrong.
**Mention first:** global shared mutable state.

## Topic 23 · Exercise 1 — Base class constructor
**Findings**
- 🟠 **DEFINITE BUG — Overridable method called from a constructor.** `Report()`'s constructor calls `title()`, which is overridden in `SalesReport` and reads `region`. But the superclass constructor runs **before** the subclass field `region` is initialized, so `title()` sees `region == null` → header becomes `"Sales for null"` (or NPE). Calling overridable methods during construction is a known hazard.

**Fix:** don't call overridable methods in the constructor — compute `header` lazily (on first `getHeader()`), or pass the title in via the constructor (`protected Report(String title)`), or make `title()` not depend on subclass state.
**What a strong senior would say:** *"The base constructor calls the overridable `title()` before the subclass has set `region`, so the header captures a null region — the classic 'don't call overridable methods from a constructor' trap. I'd either compute the header lazily after construction, or require subclasses to pass the title up through the constructor so there's no dependency on not-yet-initialized state."*
**Interviewer is testing:** initialization order and the overridable-method-in-constructor pitfall.
**Mention first:** the init-order NPE/null.

## Topic 23 · Exercise 2 — Shared behavior via default methods
**Findings**
- 🟠 **DESIGN CONCERN / won't compile — Interfaces can't hold state.** The default `touch()` assigns `this.lastModified`, but interfaces have no instance fields, so this doesn't compile — you can't store per-instance state in an interface. An interface is the wrong tool for shared *stateful* behavior.

**Fix:** use an **abstract base class** (`abstract class AuditableEntity { protected Instant lastModified; public void touch(){...} }`) for shared state, or **composition** (an `AuditInfo` field/embeddable each entity holds and delegates to). In JPA specifically, a `@MappedSuperclass` or `@Embeddable` with `@PreUpdate` is idiomatic.
**What a strong senior would say:** *"This can't work — interfaces can't hold instance fields, so a default method can't store `lastModified`. Shared *state* wants an abstract base class or composition, not an interface. In a JPA context I'd use a `@MappedSuperclass` (or an embeddable `AuditInfo`) with lifecycle callbacks, which is the standard way to share audit fields."*
**Interviewer is testing:** the interface-vs-abstract-class boundary (state → not an interface).
**Mention first:** interfaces can't store state.

## Topic 24 · Exercise 1 — Instrumented set
**Findings**
- 🟠 **DEFINITE BUG — Self-use fragility (Effective Java classic).** `HashSet.addAll` internally calls `add`, so each element is counted **twice** — once in the overridden `addAll` (+= size) and again via the overridden `add`. The count is wrong. This is the textbook danger of extending a class whose self-use behavior isn't part of its contract.

**Fix:** **composition** — wrap a `Set` and forward, incrementing the counter only in your own `add`/`addAll` (or implement `addAll` as a loop over your own `add`):
```java
public class CountingSet<E> {
    private final Set<E> delegate;
    private int addCount = 0;
    public boolean add(E e){ addCount++; return delegate.add(e); }
    public boolean addAll(Collection<? extends E> c){ boolean m=false; for (E e:c) m|=add(e); return m; }
}
```
**What a strong senior would say:** *"This double-counts on `addAll` because HashSet's `addAll` calls `add` internally, and we've overridden both — the classic fragile-base-class problem from Effective Java. The fix is composition: wrap a Set and forward, so we're not coupled to the superclass's self-use, and count in one place."*
**Interviewer is testing:** composition-over-inheritance with the canonical example.
**Mention first:** the double-count from self-use.

## Topic 24 · Exercise 2 — Premium account by inheritance
**Findings**
- 🟠 **DEFINITE BUG — Inheriting behavior that violates the new invariant.** `PremiumAccount` inherits `withdraw`, which allows negatives, so the "never negative" invariant is not enforced — inheritance pulled in behavior that contradicts the subtype's rules. Overriding `withdraw` to add a guard is possible but couples you to the parent (and to `protected balance` being directly mutable).
- 🟡 `protected balance` breaks encapsulation — subclasses (or the parent) can corrupt it directly.

**Fix:** prefer **composition/delegation** (a `PremiumAccount` holds an account and enforces its own rules), or redesign the hierarchy so the overdraft policy is a strategy; make `balance` private with controlled mutators enforcing invariants.
**What a strong senior would say:** *"Reusing `StandardAccount` by inheritance drags in the overdraft-allowing `withdraw`, so the premium account's 'never negative' rule isn't actually enforced — and the `protected balance` lets anyone bypass the rules. I'd model the overdraft policy explicitly (composition or a strategy) rather than inherit behavior that contradicts the subtype, and make balance private behind guarded operations."*
**Interviewer is testing:** inheritance pulling in unwanted behavior; encapsulation of invariants.
**Mention first:** unenforced invariant via inherited withdraw.

## Topic 25 · Exercise 1 — Concrete dependency type
**Findings**
- 🟡 **DESIGN CONCERN — Depending on a concrete class.** Depending on `JpaOrderRepository` (concrete) instead of an `OrderRepository` interface blocks substituting a fake in unit tests and adding a caching/decorator implementation later (both stated goals). Depend on the abstraction.

**Fix:** `private final OrderRepository repository;` (interface); the JPA impl (or a caching decorator) is wired by Spring.
**What a strong senior would say:** *"Since we want to unit-test with a fake and later add caching, depending on the concrete `JpaOrderRepository` works against both — I'd inject the `OrderRepository` interface instead. Then a fake in tests and a caching decorator in prod are just different implementations, no change to this class."*
**Interviewer is testing:** DIP and its testability/extensibility payoff.
**Mention first:** concrete-vs-interface dependency.

## Topic 25 · Exercise 2 — Hidden static dependency
**Findings**
- 🟠 **DESIGN CONCERN — Hidden static singleton dependency.** `PaymentGateway.getInstance()` is a hard, hidden dependency: not visible in the constructor, impossible to substitute in tests (can't inject a fake/mock), and tightly coupled to a specific implementation. Also global mutable singleton state.

**Fix:** inject a `PaymentGateway` interface via the constructor; Spring provides the implementation. Tests pass a mock.
**What a strong senior would say:** *"The static `getInstance()` is an invisible dependency — you can't tell from the constructor that this class needs a gateway, and you can't swap it in a test, so `checkout` is basically untestable without hitting the real gateway. I'd inject a `PaymentGateway` interface through the constructor and let the container wire it."*
**Interviewer is testing:** recognizing static/singleton access as a DIP + testability violation.
**Mention first:** the hidden static dependency.

## Topic 26 · Exercise 1 — Nested eligibility check
**Findings**
- 🟡 **MAINTAINABILITY — Arrow anti-pattern.** Five levels of nesting obscure the logic; guard clauses flatten it and make each rule explicit. Not a correctness bug as written, but hard to modify safely.
- 🟢 **POTENTIAL — null field access.** `a.getIncome()` could NPE if income is null; guard clauses make such checks visible.

**Fix (guard clauses):**
```java
if (a == null) return false;
if (a.getAge() < 18) return false;
if (a.getCreditScore() <= 650) return false;
if (a.getIncome() == null || a.getIncome().compareTo(new BigDecimal("30000")) < 0) return false;
return !a.hasActiveDefault();
```
**What a strong senior would say:** *"This is correct but the deep nesting makes each rule hard to see and change — I'd invert to guard clauses that return early, which reads as a clear list of disqualifiers. Flattening also surfaces edge cases like a null income, which is easy to miss buried five levels in."*
**Interviewer is testing:** readability refactoring (guard clauses) without breaking behavior; this is a legit Clean-Code topic so style focus is appropriate.
**Mention first:** the nesting, then the null-income edge.

## Topic 26 · Exercise 2 — Comment vs behavior
**Findings**
- 🟠 **DEFINITE BUG — Code contradicts its contract.** The Javadoc promises "empty list if none / never null," but the code returns `null` on empty. Callers trusting the doc will do `for (Product p : findMatches(...))` and NPE. The comment is the misleading part — the code is wrong relative to its stated contract.

**Fix:** honor the contract — just `return results;` (the repository's empty list), removing the null branch entirely.
**What a strong senior would say:** *"The doc says it never returns null and gives an empty list, but the code returns null on no matches — so every caller who trusted the contract NPEs. This is why I don't trust comments over code: here the comment states the *right* behavior and the code violates it. I'd delete the null branch and just return the (already empty) list."*
**Interviewer is testing:** verifying code against its documentation; null-vs-empty-collection contract.
**Mention first:** the doc/behavior contradiction (null return).

## Topic 27 · Exercise 1 — Address data clump
**Findings**
- 🔴 **DEFINITE BUG — Argument transposition (hidden by the smell).** The caller passes `getCountry()` and `getZip()` in the `zip, country` parameter positions — swapped — because five same-typed String params are easy to misorder. The data clump *causes* the bug. Silent data corruption on labels.
- 🟡 **Smell — data clump / primitive obsession.** The repeated `street, city, zip, country` group should be an `Address` value object; passing it as one object makes transposition impossible and cleans up every signature.

**Fix:** introduce `Address` (record); methods take `Address`; the caller passes `customer.getAddress()`.
**What a strong senior would say:** *"There's an actual bug here — the caller has country and zip swapped — and it's a direct consequence of the smell: five String parameters in a row are trivially misordered and the compiler can't help. I'd extract an `Address` value object so the fields travel together and transposition becomes impossible. The refactor fixes the class of bug, not just this instance."*
**Interviewer is testing:** connecting a code smell to a concrete bug; data-clump refactor.
**Mention first:** the swapped arguments (🔴), then the clump.

## Topic 27 · Exercise 2 — Reaching through objects
**Findings**
- 🟡 **DESIGN/Robustness — Law of Demeter / train wreck + NPE chain.** `employee.getDepartment().getManager().getContact().getEmail()` couples this method to four classes' internal structure and NPEs if any link is null (no department, no manager, no contact). Fragile and hard to test.
- 🟢 No null handling for "employee has no manager" (a real domain case).

**Fix:** add a behavior method that encapsulates the traversal (`employee.managerEmail()` returning `Optional<String>`), or fetch the manager's contact through a service; handle the no-manager case explicitly.
**What a strong senior would say:** *"This train wreck reaches through four objects, so it breaks if any link is null and couples us to the whole object graph's shape. I'd push the navigation behind a method like `employee.managerEmail()` returning an Optional, and handle 'no manager' explicitly rather than NPE. It also makes the method testable without building four nested stubs."*
**Interviewer is testing:** Law of Demeter and null-chain robustness.
**Mention first:** the fragile chain / NPE risk.

---

# Senior Java Review Checklist — Part 2 (Language-level)

**Value types & identity:** equals⇔hashCode always paired · entity identity on a stable business key, never a null generated id · `BigDecimal` uses `compareTo` (not `equals`) and division needs scale+RoundingMode · records are only *shallow* immutable (defensive-copy collections; arrays break value equality).
**Immutability:** `final` freezes the reference, not the contents · defensive-copy on the way **in and out** · leaked getters break immutability.
**Strings:** `.equals` not `==` · Strings are immutable (use the return value of `trim`/`toLowerCase`) · loop concatenation is O(n²).
**Builders:** local `StringBuilder` is thread-safe (don't flag it) · shared `StringBuilder` is a race · `StringBuffer` makes single ops atomic, not *sequences*.
**Collections:** don't mutate while iterating (`removeIf`) · `Arrays.asList`/`List.of` are fixed/immutable · HashMap has no order (use LinkedHashMap/TreeMap) · mutable map keys go unreachable · LinkedList index access is O(n²) · front ops want ArrayDeque · TreeSet dedups by **comparator**, not equals · HashSet is unordered.
**Concurrency (language):** CHM makes single ops atomic — compound ops need `computeIfAbsent`/`merge` · `synchronizedMap` iteration must be externally synchronized · mutable enums = global shared state.
**Streams:** no side effects into shared collections · `parallelStream` runs on the common pool (bad for blocking I/O) · `toMap` throws on duplicate keys.
**Optional:** never blind `.get()` · `orElse` evaluates eagerly — use `orElseGet` for expensive/side-effecting defaults · not for fields/params.
**Generics:** avoid raw types · use `? extends`/PECS so APIs accept subtypes.
**Exceptions:** chain the cause when wrapping · catch `Exception`, not `Throwable`/`Error`.
**Resources:** try-with-resources over manual finally (null-safe + suppressed exceptions) — but don't invent a leak where TWR already closes the chain.
**Date/Time:** `Instant` for a moment, `LocalDateTime` has no zone · `Period`/`plusDays` for calendar days, `Duration` for elapsed time (DST!).
**Enums:** persist by `STRING`, never `ORDINAL`.
**OOP/SOLID/design:** protect invariants (no leaked internals, no overridable calls in constructors) · LSP (no surprise `UnsupportedOperationException`) · ISP (segregate fat interfaces) · DIP (inject interfaces, avoid `new`/static singletons) · composition over inheritance (fragile base class / self-use) · interfaces can't hold state → abstract class/composition.
**Clean code / smells:** guard clauses over deep nesting · verify code against its comments/contract (null vs empty) · data clumps → value objects (and they prevent argument transposition) · Law of Demeter (no train wrecks).

**Always:** name at least one *not-a-bug* (local StringBuilder, correct TWR, inbound defensive copy) to show judgment; separate DEFINITE BUG from DESIGN/NIT; lead with 🔴/🟠.

---

*End of Part 2. Say **"next"** for Part 3 — Concurrency & Multithreading (28 topics × 2 = 56 exercises).*
