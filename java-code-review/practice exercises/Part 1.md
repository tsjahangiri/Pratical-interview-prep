# Part 1 — Senior Code-Review Mindset (Practice Workbook)

**40 exercises — 20 topics × 2.** Java 17+, Spring Boot 3.x, PostgreSQL-style DB, multi-instance production assumptions.

> **How to use this file.** Review each exercise cold — do *not* scroll to the answer key. Write your findings (severity, why, fix, trade-offs) before checking. Some exercises contain multiple bugs; some contain code that *looks* wrong but is correct; some contain misleading comments. Treat every comment as unverified.

**Legend:** 🔴 Critical · 🟠 High · 🟡 Medium · 🟢 Low · ⚪ Nit

---

# EXERCISES

## Topic 1 — Correctness

### Exercise 1 — Order total with discount

**Scenario**
`OrderService` computes the final price of an order. It's called from the checkout endpoint. Discounts are stored as whole-number percentages (e.g. `10` means 10% off).

**Code**
```java
public double finalPrice(double subtotal, int discountPercent) {
    double discount = subtotal * discountPercent / 100;
    return subtotal - discount * discountPercent;
}
```

**Your Task**
Perform a senior-level code review. Identify correctness, reliability, and any other relevant issues.

---

### Exercise 2 — Average order value report

**Scenario**
`ReportService` computes the average order value for a daily report. `amounts` holds each order total in cents. This runs in a nightly batch over ~10,000 orders/day.

**Code**
```java
public int averageOrderValue(List<Integer> amounts) {
    int sum = 0;
    for (int a : amounts) {
        sum += a;
    }
    return sum / amounts.size();
}
```

**Your Task**
Review for correctness and edge cases.

---

## Topic 2 — Business logic

### Exercise 1 — Refund processing

**Scenario**
`PaymentService.refund` issues a refund against a prior payment. Called from an admin endpoint. A payment may be partially refunded more than once.

**Code**
```java
public void refund(Payment payment, BigDecimal amount) {
    if (amount.compareTo(payment.getAmount()) > 0) {
        throw new IllegalArgumentException("Refund exceeds payment");
    }
    gateway.refund(payment.getTransactionId(), amount);
    payment.setStatus(Status.REFUNDED);
    paymentRepository.save(payment);
}
```

**Your Task**
Review the business logic for correctness and safety.

---

### Exercise 2 — Loyalty tier assignment

**Scenario**
`CustomerService` assigns a loyalty tier based on annual spend. Tiers: BRONZE < 1000, SILVER 1000–4999, GOLD 5000+.

**Code**
```java
public Tier tierFor(BigDecimal annualSpend) {
    if (annualSpend.compareTo(new BigDecimal("1000")) > 0) {
        return Tier.SILVER;
    } else if (annualSpend.compareTo(new BigDecimal("5000")) > 0) {
        return Tier.GOLD;
    }
    return Tier.BRONZE;
}
```

**Your Task**
Review for business-logic correctness.

---

## Topic 3 — Edge cases

### Exercise 1 — Percentile latency

**Scenario**
`MetricsService` returns the median latency from a list of samples collected over a minute. `samples` can occasionally be empty when no traffic arrived.

**Code**
```java
public long median(List<Long> samples) {
    Collections.sort(samples);
    return samples.get(samples.size() / 2);
}
```

**Your Task**
Review for edge cases and any side effects.

---

### Exercise 2 — Subscription renewal date

**Scenario**
`SubscriptionService` computes the next renewal date by adding one month to the current start date. Subscriptions can start on any calendar day.

**Code**
```java
public LocalDate nextRenewal(LocalDate start) {
    return start.plusDays(30);
}
```

**Your Task**
Review for edge cases and correctness against the stated intent ("add one month").

---

## Topic 4 — Null handling

### Exercise 1 — Shipping address lookup

**Scenario**
`ShippingService` builds a label from a customer's default address. Not all customers have set a default address.

**Code**
```java
public String label(Customer customer) {
    Address a = customer.getDefaultAddress();
    return a.getStreet().toUpperCase() + ", " + a.getCity().toUpperCase();
}
```

**Your Task**
Review for null-safety and correctness.

---

### Exercise 2 — Config value with default

**Scenario**
`FeatureService` reads a feature flag from a config map. The map is populated at startup from environment variables.

**Code**
```java
public boolean isEnabled(Map<String, String> config, String key) {
    // config may not contain the key, so we guard with Optional
    return Optional.ofNullable(config.get(key))
                   .map(Boolean::parseBoolean)
                   .orElse(false);
}
```

**Your Task**
Review for null-safety. State clearly whether this is correct.

---

## Topic 5 — Exception handling

### Exercise 1 — Notification dispatch

**Scenario**
`NotificationService.sendAll` sends emails to a list of recipients. It's important that one bad address doesn't stop the rest.

**Code**
```java
public void sendAll(List<String> recipients, String body) {
    for (String r : recipients) {
        try {
            emailClient.send(r, body);
        } catch (Exception e) {
            // continue with the next recipient
        }
    }
}
```

**Your Task**
Review the exception-handling strategy.

---

### Exercise 2 — Import worker interrupt

**Scenario**
`ImportService` runs on a background worker thread and polls a queue. A graceful shutdown interrupts the thread.

**Code**
```java
public void run() {
    while (true) {
        try {
            Task task = queue.take();
            process(task);
        } catch (InterruptedException e) {
            log.warn("Interrupted, retrying");
        }
    }
}
```

**Your Task**
Review the exception handling and shutdown behavior.

---

## Topic 6 — Resource management

### Exercise 1 — CSV export

**Scenario**
`ExportService` writes a report to disk. Called from a scheduled job a few times per hour.

**Code**
```java
public void export(String path, List<String> rows) throws IOException {
    BufferedWriter w = new BufferedWriter(new FileWriter(path));
    for (String row : rows) {
        w.write(row);
        w.newLine();
    }
    w.close();
}
```

**Your Task**
Review resource handling and reliability.

---

### Exercise 2 — JDBC lookup

**Scenario**
`LegacyLookupService` runs a raw JDBC query against a connection obtained from a pooled `DataSource`. Called on a hot path, ~100 req/s.

**Code**
```java
public String findName(long id) throws SQLException {
    try (Connection c = dataSource.getConnection()) {
        PreparedStatement ps = c.prepareStatement("SELECT name FROM users WHERE id = ?");
        ps.setLong(1, id);
        ResultSet rs = ps.executeQuery();
        if (rs.next()) {
            return rs.getString("name");
        }
        return null;
    }
}
```

**Your Task**
Review resource handling. Note anything that is *not* actually a leak as well.

---

## Topic 7 — Concurrency / thread safety

### Exercise 1 — Request counter in a service

**Scenario**
`ApiService` is a Spring `@Service` (singleton) that tracks how many requests it has served, exposed on a metrics endpoint. Called concurrently by many REST threads.

**Code**
```java
@Service
public class ApiService {
    private long requestCount = 0;

    public Result handle(Request req) {
        requestCount++;
        return doWork(req);
    }

    public long getRequestCount() {
        return requestCount;
    }
}
```

**Your Task**
Review for thread safety.

---

### Exercise 2 — Shared formatter

**Scenario**
`AuditService` formats timestamps for audit log lines. It's invoked from many concurrent request threads. A previous author left a note.

**Code**
```java
@Service
public class AuditService {
    // Shared formatter reused for performance
    private static final DateTimeFormatter FMT =
            DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss");

    public String line(String event, Instant at) {
        String ts = FMT.format(at.atZone(ZoneOffset.UTC));
        return ts + " " + event;
    }
}
```

**Your Task**
Review for thread safety. State clearly whether the shared formatter is a bug.

---

## Topic 8 — Performance

### Exercise 1 — Membership check in a loop

**Scenario**
`InventoryService` filters incoming SKUs against a blocklist. The blocklist has ~50,000 entries; requests arrive with up to a few thousand SKUs each.

**Code**
```java
public List<String> allowed(List<String> skus, List<String> blocklist) {
    List<String> result = new ArrayList<>();
    for (String sku : skus) {
        if (!blocklist.contains(sku)) {
            result.add(sku);
        }
    }
    return result;
}
```

**Your Task**
Review for performance. Reason about scale.

---

### Exercise 2 — Validation regex

**Scenario**
`ValidationService.isValidEmail` validates email addresses. Called on every field of every incoming request; the endpoint handles ~100 req/s.

**Code**
```java
public boolean isValidEmail(String email) {
    Pattern pattern = Pattern.compile("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+$");
    return pattern.matcher(email).matches();
}
```

**Your Task**
Review for performance and correctness.

---

## Topic 9 — Memory usage

### Exercise 1 — In-memory result cache

**Scenario**
`PricingService` caches computed prices keyed by a request signature to avoid recomputation. The service runs for weeks between deploys and sees a wide variety of keys.

**Code**
```java
@Service
public class PricingService {
    private final Map<String, BigDecimal> cache = new HashMap<>();

    public BigDecimal price(PriceRequest req) {
        String key = req.signature();
        return cache.computeIfAbsent(key, k -> compute(req));
    }
}
```

**Your Task**
Review for memory behavior at scale.

---

### Exercise 2 — Report from full table load

**Scenario**
`CustomerReportService` builds a CSV of all customers' names. The `customers` table has grown to several million rows.

**Code**
```java
public String namesCsv() {
    List<Customer> all = customerRepository.findAll();
    return all.stream()
              .map(Customer::getName)
              .collect(Collectors.joining("\n"));
}
```

**Your Task**
Review for memory and scalability.

---

## Topic 10 — Database interaction

### Exercise 1 — Bulk status update

**Scenario**
`OrderService.markShipped` updates the status of many orders after a warehouse batch completes. A batch can contain tens of thousands of order IDs.

**Code**
```java
public void markShipped(List<Long> orderIds) {
    for (Long id : orderIds) {
        Order o = orderRepository.findById(id).orElseThrow();
        o.setStatus(Status.SHIPPED);
        orderRepository.save(o);
    }
}
```

**Your Task**
Review the database-interaction pattern and reason about scale.

---

### Exercise 2 — Search by name

**Scenario**
`ProductService.search` finds products by a name fragment supplied by the client.

**Code**
```java
public List<Product> search(String name) {
    String sql = "SELECT * FROM products WHERE name LIKE '%" + name + "%'";
    return jdbcTemplate.query(sql, productRowMapper);
}
```

**Your Task**
Review the database interaction.

---

## Topic 11 — Transactions

### Exercise 1 — Transfer with external call

**Scenario**
`AccountService.transfer` moves money between two internal accounts and notifies an external fraud-check service. `@Transactional` wraps the whole method. The fraud service typically responds in 200ms but can take up to 30s under load.

**Code**
```java
@Transactional
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    Account from = accountRepository.findById(fromId).orElseThrow();
    Account to = accountRepository.findById(toId).orElseThrow();

    from.setBalance(from.getBalance().subtract(amount));
    to.setBalance(to.getBalance().add(amount));
    accountRepository.save(from);
    accountRepository.save(to);

    fraudClient.report(fromId, toId, amount); // synchronous HTTP call
}
```

**Your Task**
Review the transaction boundaries and reliability.

---

### Exercise 2 — Self-invoked transactional method

**Scenario**
`InvoiceService` processes a batch of invoices. Each invoice should be persisted in its own transaction so one failure doesn't roll back the others.

**Code**
```java
@Service
public class InvoiceService {

    public void processBatch(List<Invoice> invoices) {
        for (Invoice inv : invoices) {
            processOne(inv);
        }
    }

    @Transactional
    public void processOne(Invoice inv) {
        invoiceRepository.save(inv);
        ledgerRepository.save(inv.toLedgerEntry());
    }
}
```

**Your Task**
Review the transactional behavior.

---

## Topic 12 — API design

### Exercise 1 — Customer controller

**Scenario**
`CustomerController` exposes read and delete operations. `Customer` is a JPA entity with a lazy `orders` collection.

**Code**
```java
@RestController
@RequestMapping("/customers")
public class CustomerController {

    @GetMapping("/{id}")
    public Customer get(@PathVariable Long id) {
        return customerService.findById(id);
    }

    @GetMapping("/delete/{id}")
    public String delete(@PathVariable Long id) {
        customerService.delete(id);
        return "deleted";
    }
}
```

**Your Task**
Review the API design.

---

### Exercise 2 — List endpoint

**Scenario**
`ProductController` returns products for a catalog page. The catalog has grown to ~2 million products.

**Code**
```java
@GetMapping("/products")
public List<ProductDto> list() {
    return productService.findAll()
                         .stream()
                         .map(ProductDto::from)
                         .toList();
}
```

**Your Task**
Review the API design and reason about scale.

---

## Topic 13 — Security

### Exercise 1 — Login query

**Scenario**
`AuthService.authenticate` checks a username/password against the database.

**Code**
```java
public boolean authenticate(String username, String password) {
    String sql = "SELECT count(*) FROM users WHERE username = '" + username
               + "' AND password = '" + password + "'";
    Integer n = jdbcTemplate.queryForObject(sql, Integer.class);
    return n != null && n > 0;
}
```

**Your Task**
Review for security. There is more than one issue.

---

### Exercise 2 — Payment logging

**Scenario**
`PaymentService.charge` logs the request for debugging before calling the gateway. Logs are shipped to a central aggregator accessible to many teams.

**Code**
```java
public ChargeResult charge(CardDetails card, BigDecimal amount) {
    log.info("Charging {} to card {} exp {} cvv {}",
             amount, card.getNumber(), card.getExpiry(), card.getCvv());
    return gateway.charge(card, amount);
}
```

**Your Task**
Review for security and observability.

---

## Topic 14 — Logging and observability

### Exercise 1 — Debug logging in a hot loop

**Scenario**
`PricingEngine` processes large batches of line items. The log level in production is INFO.

**Code**
```java
public void priceAll(List<LineItem> items) {
    for (LineItem item : items) {
        log.debug("Pricing item " + item.getId() + " with data " + item.toString());
        item.setPrice(compute(item));
    }
}
```

**Your Task**
Review the logging. State clearly whether the log level protects this.

---

### Exercise 2 — Silent failure

**Scenario**
`SyncService` pushes updates to a downstream system. Operators complain they "never know when sync breaks."

**Code**
```java
public void sync(List<Update> updates) {
    for (Update u : updates) {
        try {
            downstream.push(u);
        } catch (Exception e) {
            // swallow: we'll pick it up on the next run
            continue;
        }
    }
}
```

**Your Task**
Review for observability.

---

## Topic 15 — Testing

### Exercise 1 — A test that always passes

**Scenario**
This unit test is meant to verify discount calculation.

**Code**
```java
@Test
void appliesDiscount() {
    OrderService service = new OrderService();
    double result = service.finalPrice(100.0, 10);
    assertNotNull(result);
}
```

**Your Task**
Review the test quality.

---

### Exercise 2 — Repository test with mocks

**Scenario**
`OrderQueryService` runs a custom JPQL query to find overdue orders. The test is meant to verify that query.

**Code**
```java
@Test
void findsOverdueOrders() {
    OrderRepository repo = mock(OrderRepository.class);
    when(repo.findOverdue(any())).thenReturn(List.of(new Order(1L), new Order(2L)));

    OrderQueryService service = new OrderQueryService(repo);
    List<Order> overdue = service.overdue();

    assertEquals(2, overdue.size());
}
```

**Your Task**
Review whether this test actually verifies what the scenario claims.

---

## Topic 16 — Maintainability

### Exercise 1 — Status handling

**Scenario**
`OrderProcessor.handle` reacts to an order's status. New statuses are added a few times per year.

**Code**
```java
public void handle(Order order) {
    if (order.getStatus() == 1) {
        // new
        reserveStock(order);
        charge(order);
    } else if (order.getStatus() == 2) {
        // paid
        ship(order);
    } else if (order.getStatus() == 3) {
        // shipped
        notifyCustomer(order);
    } else if (order.getStatus() == 4) {
        refund(order);
    }
}
```

**Your Task**
Review for maintainability.

---

### Exercise 2 — Boolean flags

**Scenario**
`EmailService.send` is called from ~15 places across the codebase.

**Code**
```java
public void send(String to, String body, boolean html, boolean urgent, boolean bcc) {
    // ...
}

// a typical call site:
emailService.send(user.getEmail(), body, true, false, true);
```

**Your Task**
Review for maintainability and readability.

---

## Topic 17 — SOLID / design principles

### Exercise 1 — A service doing everything

**Scenario**
`OrderService.placeOrder` is the single entry point for checkout.

**Code**
```java
@Service
public class OrderService {
    public void placeOrder(OrderRequest req) {
        // validate
        if (req.getItems().isEmpty()) throw new IllegalArgumentException();
        // persist
        Order order = orderRepository.save(toEntity(req));
        // build & send email inline
        String html = "<html><body>Order " + order.getId() + " confirmed</body></html>";
        smtp.connect();
        smtp.send(req.getEmail(), html);
        smtp.disconnect();
        // call shipping REST API inline
        RestTemplate rt = new RestTemplate();
        rt.postForObject("http://shipping/api/ship", order, Void.class);
    }
}
```

**Your Task**
Review against SOLID principles.

---

### Exercise 2 — Hard-wired dependency

**Scenario**
`ReportService` generates a PDF. The team wants to swap the PDF library later without touching this class.

**Code**
```java
@Service
public class ReportService {
    public byte[] generate(ReportData data) {
        ItextPdfGenerator generator = new ItextPdfGenerator();
        return generator.render(data);
    }
}
```

**Your Task**
Review against SOLID (especially dependency inversion).

---

## Topic 18 — Scalability

### Exercise 1 — In-memory rate limiter

**Scenario**
`RateLimiter` caps each user to 100 requests/minute. The service runs on 6 pods behind a load balancer that does not use sticky sessions.

**Code**
```java
@Service
public class RateLimiter {
    private final Map<String, Integer> counts = new ConcurrentHashMap<>();

    public boolean allow(String userId) {
        int c = counts.merge(userId, 1, Integer::sum);
        return c <= 100;
    }
}
```

**Your Task**
Review for scalability and correctness across instances.

---

### Exercise 2 — Aggregate computed in memory

**Scenario**
`SalesService.totalRevenue` returns the sum of all order amounts for a dashboard tile refreshed every 10 seconds. The `orders` table has millions of rows.

**Code**
```java
public BigDecimal totalRevenue() {
    return orderRepository.findAll().stream()
            .map(Order::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
}
```

**Your Task**
Review for scalability.

---

## Topic 19 — Reliability / fault tolerance

### Exercise 1 — External client with no timeout

**Scenario**
`GeoService` calls a third-party geocoding API on the request path. The dependency is usually fast but occasionally hangs.

**Code**
```java
@Service
public class GeoService {
    private final RestTemplate restTemplate = new RestTemplate();

    public GeoResult geocode(String address) {
        return restTemplate.getForObject(
            "https://geo.example.com/v1?q=" + address, GeoResult.class);
    }
}
```

**Your Task**
Review for reliability and fault tolerance.

---

### Exercise 2 — Retry loop

**Scenario**
`PaymentService.chargeWithRetry` retries a charge on failure. The charge call is not known to be idempotent.

**Code**
```java
public ChargeResult chargeWithRetry(CardDetails card, BigDecimal amount) {
    for (int i = 0; i < 5; i++) {
        try {
            return gateway.charge(card, amount);
        } catch (GatewayException e) {
            // transient; try again immediately
        }
    }
    throw new IllegalStateException("charge failed");
}
```

**Your Task**
Review the retry strategy for reliability and correctness.

---

## Topic 20 — Distributed-system concerns

### Exercise 1 — DB write then event publish

**Scenario**
`OrderService.place` persists an order and publishes an `OrderPlaced` event so downstream services (inventory, email) react. Runs on multiple instances; Kafka is the broker.

**Code**
```java
@Transactional
public void place(Order order) {
    orderRepository.save(order);
    kafkaTemplate.send("orders", new OrderPlaced(order.getId()));
}
```

**Your Task**
Review for distributed-system correctness.

---

### Exercise 2 — Event consumer

**Scenario**
`InventoryConsumer` listens to `OrderPlaced` and decrements stock. The topic is configured for at-least-once delivery.

**Code**
```java
@KafkaListener(topics = "orders")
public void onOrderPlaced(OrderPlaced event) {
    Product p = productRepository.findById(event.getProductId()).orElseThrow();
    p.setStock(p.getStock() - 1);
    productRepository.save(p);
}
```

**Your Task**
Review for distributed-system correctness (delivery semantics, idempotency, concurrency).

---

---

# ANSWER KEY

Classifications used: **DEFINITE BUG** · **POTENTIAL BUG** (depends on requirements) · **DESIGN CONCERN** · **PERFORMANCE CONCERN** · **NIT** · **NOT A BUG**.

## Topic 1 · Exercise 1 — Order total with discount

**Findings**
- 🔴 **DEFINITE BUG — Correctness.** `subtotal - discount * discountPercent` multiplies the discount amount by the percent *again*. For subtotal 100, discount 10%: `discount = 10`, result `= 100 - 10*10 = 0`. It should be `subtotal - discount`. Operator-precedence/logic error that silently corrupts every price.
- 🟡 **PERFORMANCE/Correctness — money as `double`.** Prices in `double` accumulate rounding error and can serialize oddly. Use `BigDecimal`.
- 🟢 **Edge case.** No guard for `discountPercent` outside 0–100 (negative → price increase; >100 → negative price).

**Fix**
```java
public BigDecimal finalPrice(BigDecimal subtotal, int discountPercent) {
    if (discountPercent < 0 || discountPercent > 100)
        throw new IllegalArgumentException("discountPercent must be 0..100");
    BigDecimal discount = subtotal.multiply(BigDecimal.valueOf(discountPercent))
                                  .divide(BigDecimal.valueOf(100));
    return subtotal.subtract(discount);
}
```
**What a strong senior would say:** *"The core bug is the double-multiply — we subtract discount×percent instead of discount, so a 10% discount zeroes a €100 order. Beyond that I'd move money off `double` to `BigDecimal` and validate the percent range. This is the kind of thing a single unit test with a known value would have caught."*
**What the interviewer is testing:** careful reading of arithmetic; money handling instinct; whether you add a test.
**Expected findings:** the multiply bug. **Bonus:** `double`-for-money, range validation, "a test asserting the exact value catches this."
**Mention first:** the correctness bug (🔴).

## Topic 1 · Exercise 2 — Average order value

**Findings**
- 🔴 **DEFINITE BUG — Edge case.** Empty list → `sum / 0` → `ArithmeticException`. A no-traffic day crashes the batch.
- 🟡 **POTENTIAL BUG — Overflow.** `int sum` over 10k orders in cents: 10,000 × (say) €500 = 500,000,000 cents fits, but a larger day or higher-value orders can exceed `Integer.MAX_VALUE` (~2.1B). Use `long`.
- 🟢 **Integer division** truncates the average (acceptable if cents and truncation is intended — clarify).

**Fix:** guard empty (`return 0` or throw a domain exception), accumulate in `long`.
**What a strong senior would say:** *"Two things: empty input divides by zero, and I'd accumulate in `long` to avoid overflow on big days. The integer truncation may be fine since we're in cents, but I'd confirm the rounding expectation."*
**Interviewer is testing:** empty-collection reflex, overflow awareness.
**Expected:** div-by-zero. **Bonus:** overflow, rounding clarification. **Mention first:** div-by-zero.

## Topic 2 · Exercise 1 — Refund processing

**Findings**
- 🔴 **DEFINITE BUG — Business logic.** No tracking of cumulative refunds. The check compares each refund to the *original* amount, so a payment of €100 can be refunded €80 + €80 = €160. Must track already-refunded total and validate `alreadyRefunded + amount <= payment.getAmount()`.
- 🟠 **DEFINITE BUG — State/atomicity.** `setStatus(REFUNDED)` on a *partial* refund is wrong (it's `PARTIALLY_REFUNDED`), and the gateway call happens before persistence with no transaction — if `save` fails after the gateway refunds, the money left but the record says otherwise.
- 🟠 **Reliability — no idempotency.** A retry re-refunds. Needs an idempotency key / refund ledger row.

**Fix:** sum prior refunds; set status by remaining balance; wrap in `@Transactional`; record each refund as its own row with an idempotency key.
**What a strong senior would say:** *"The dangerous one is that we validate against the original amount, not the remaining balance — so a payment can be over-refunded across multiple partial refunds. I'd track refunds in a ledger, validate against the remaining balance, make it idempotent, and set status from the remaining amount rather than hard-coding REFUNDED."*
**Interviewer is testing:** whether you model state over time, not just one call; money safety; idempotency.
**Expected:** over-refund. **Bonus:** partial-status, gateway/DB ordering, idempotency. **Mention first:** over-refund (🔴), then gateway-before-persist ordering.

## Topic 2 · Exercise 2 — Loyalty tier

**Findings**
- 🔴 **DEFINITE BUG — Business logic (unreachable branch).** The first `if` catches everything `> 1000`, so `annualSpend = 6000` returns SILVER — the GOLD branch is **unreachable**. Order the conditions high-to-low, or check GOLD first.
- 🟡 **Boundary — off-by-one.** Spec says SILVER is 1000–4999, i.e. **≥ 1000**, but code uses `> 1000`, so exactly 1000 falls to BRONZE. Use `>=`.

**Fix:**
```java
if (annualSpend.compareTo(new BigDecimal("5000")) >= 0) return Tier.GOLD;
if (annualSpend.compareTo(new BigDecimal("1000")) >= 0) return Tier.SILVER;
return Tier.BRONZE;
```
**What a strong senior would say:** *"The GOLD branch is dead code — the first condition swallows everything over 1000. And the boundaries use strict `>` where the spec is inclusive, so 1000 and 5000 land in the wrong tier. I'd reorder high-to-low and switch to `>=`, and I'd add boundary tests at exactly 1000 and 5000."*
**Interviewer is testing:** branch-order reasoning, boundary conditions.
**Expected:** unreachable GOLD. **Bonus:** inclusive-boundary off-by-one, boundary tests. **Mention first:** unreachable branch.

## Topic 3 · Exercise 1 — Median latency

**Findings**
- 🔴 **DEFINITE BUG — Edge case.** Empty `samples` → `get(0)` on an empty list → `IndexOutOfBoundsException`. Guard for empty (return 0 / `OptionalLong`).
- 🟠 **DESIGN CONCERN — mutating the caller's list.** `Collections.sort(samples)` sorts the *input* in place — a surprising side effect if the caller reuses it (and it fails if the list is immutable). Copy first.
- 🟡 **Correctness — "median" definition.** For even sizes this returns the upper-middle element, not the average of the two middles. Clarify whether that's acceptable.

**Fix:** copy, guard empty, define even-size behavior.
**What a strong senior would say:** *"Empty input throws, and I'd also flag that we're sorting the caller's list in place — that's a side effect that bites when the list is shared or immutable. I'd sort a copy. And I'd confirm the median definition for even counts before calling it correct."*
**Interviewer is testing:** empty-collection reflex *and* spotting the hidden mutation (senior signal).
**Expected:** empty crash. **Bonus:** in-place mutation, even-size semantics. **Mention first:** empty crash, then the mutation.

## Topic 3 · Exercise 2 — Renewal date

**Findings**
- 🟠 **POTENTIAL BUG — Edge case vs intent.** Scenario says "add one month" but code adds 30 days. These diverge every month (Jan 31 + 30 = Mar 2; drifts over a year). If the requirement is calendar-month, use `plusMonths(1)`.
- 🟡 **Month-end semantics.** `plusMonths(1)` on Jan 31 yields Feb 28/29 (java.time clamps) — usually the desired billing behavior; worth stating explicitly.

**Fix:** `return start.plusMonths(1);` (and document month-end clamping).
**What a strong senior would say:** *"This is a requirements question, not just a bug — 30 days isn't a month. If billing is monthly, `plusMonths(1)` is correct and it handles month-end clamping for us. I'd confirm the intended semantics before declaring it wrong."*
**Interviewer is testing:** reading code against *intent*, and asking rather than assuming.
**Good clarifying question:** *"Is 'one month' a calendar month or a fixed 30-day cycle?"* — **Why it matters:** it changes every renewal date and revenue recognition.
**Mention first:** the intent mismatch, framed as a question.

## Topic 4 · Exercise 1 — Shipping label

**Findings**
- 🔴 **DEFINITE BUG — Null handling.** `getDefaultAddress()` can be null (scenario says so) → NPE. Also `getStreet()`/`getCity()` may be null.
- 🟡 **API/UX.** NPE surfaces as a 500; should be a clear domain error / 400.

**Fix:** guard the address and required fields; throw a meaningful exception or return an `Optional`/error.
**What a strong senior would say:** *"The scenario tells us not every customer has a default address, so `getDefaultAddress()` returns null and this NPEs. I'd guard it and return a domain-specific error rather than letting it bubble as a 500 — and I'd null-check the street/city too if those are optional."*
**Interviewer is testing:** does the candidate use the stated context (null is explicitly possible).
**Expected:** address NPE. **Bonus:** field-level nulls, error-mapping. **Mention first:** the NPE.

## Topic 4 · Exercise 2 — Config with default

**Findings**
- ⚪ **NOT A BUG.** The comment implies risk, but `Optional.ofNullable(config.get(key))...orElse(false)` is correct null-handling: missing key → `null` → `Optional.empty` → `false`. No NPE.
- 🟢 **NIT.** `Optional` here is slightly heavier than `Boolean.parseBoolean(config.getOrDefault(key, "false"))`, but both are fine; not worth a code-review comment.
- 🟢 **POTENTIAL edge.** `parseBoolean` treats anything non-"true" as false (e.g. `"1"`, `"yes"` → false). Only relevant if the config uses those forms — clarify.

**What a strong senior would say:** *"This one's actually fine — the Optional chain safely handles a missing key and returns false. I wouldn't change it. The only thing I'd sanity-check is whether the config might use values like '1' or 'yes', since `parseBoolean` only accepts 'true'."*
**Interviewer is testing:** discipline **not** to flag correct code — false-positive resistance.
**Mention first:** state plainly that it's correct, then the minor parseBoolean caveat.

## Topic 5 · Exercise 1 — Notification dispatch

**Findings**
- 🟠 **DEFINITE BUG — Observability.** The catch is empty: failures are invisible. "Continue on error" is a valid *strategy*, but you must log the failure (with recipient + cause) and ideally collect a failed list / emit a metric.
- 🟡 **Exception scope.** Catching `Exception` swallows programming errors (NPE) the same as transient send failures; consider catching the specific mail exception and letting unexpected ones surface — or at least log everything.
- 🟢 No dead-letter / retry for failed recipients — depends on requirements.

**Fix:** log each failure with context; return/collect failures; consider a retry or DLQ.
**What a strong senior would say:** *"Continuing past a bad recipient is a reasonable requirement, but swallowing silently means we never learn about failures. I'd log recipient and cause, emit a metric, and return the list of failures so the caller can retry or alert. I'd also be wary of catching bare `Exception` — that hides bugs, not just bad addresses."*
**Interviewer is testing:** distinguishing "continue-on-error is fine" from "silent is not."
**Expected:** silent swallow. **Bonus:** exception scoping, failure reporting/metrics. **Mention first:** the silent swallow.

## Topic 5 · Exercise 2 — Worker interrupt

**Findings**
- 🔴 **DEFINITE BUG — Concurrency/shutdown.** Catching `InterruptedException` and looping without restoring the interrupt flag (`Thread.currentThread().interrupt()`) or breaking means the worker **ignores shutdown and loops forever**. The `while(true)` with no exit makes it worse.
- 🟠 **DESIGN.** No exit condition; `process(task)` exceptions aren't handled (a runtime exception kills the loop entirely — the opposite failure mode).

**Fix:**
```java
while (!Thread.currentThread().isInterrupted()) {
    try {
        Task task = queue.take();
        process(task);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt(); // restore + exit
        break;
    } catch (Exception e) {
        log.error("Task failed", e); // keep processing
    }
}
```
**What a strong senior would say:** *"Swallowing `InterruptedException` and continuing defeats graceful shutdown — the thread will never stop when the pool asks it to. I'd restore the interrupt flag and break. Separately, a runtime exception in `process` currently kills the whole loop, so I'd catch those per-iteration and log."*
**Interviewer is testing:** the single most-missed concurrency idiom — interrupt handling.
**Expected:** interrupt not restored / infinite loop. **Bonus:** per-task exception isolation. **Mention first:** interrupt/shutdown bug.

## Topic 6 · Exercise 1 — CSV export

**Findings**
- 🟠 **DEFINITE BUG — Resource management.** `close()` is called on the happy path only. If a `write` throws, the writer never closes → file-descriptor leak and possibly a partially-written file. Use try-with-resources so it closes on exception too.
- 🟢 Charset unspecified (platform default) — pass `StandardCharsets.UTF_8`.

**Fix:**
```java
try (BufferedWriter w = Files.newBufferedWriter(Path.of(path), StandardCharsets.UTF_8)) {
    for (String row : rows) { w.write(row); w.newLine(); }
}
```
**What a strong senior would say:** *"The close is outside a finally, so any write failure leaks the handle. Try-with-resources fixes that cleanly. I'd also pin the charset to UTF-8 rather than rely on the platform default."*
**Interviewer is testing:** try-with-resources reflex; noticing close-on-happy-path-only.
**Mention first:** the leak.

## Topic 6 · Exercise 2 — JDBC lookup

**Findings**
- 🟠 **DEFINITE BUG — Resource management.** The `Connection` is closed (try-with-resources), **but the `PreparedStatement` and `ResultSet` are not**. On a 100 req/s hot path, statement/cursor leaks accumulate (some pools/drivers close children with the connection, but relying on that is fragile). Wrap the statement (and rs) in try-with-resources too.
- ⚪ **NOT A BUG — the Connection handling.** Closing a pooled connection *returns it to the pool* — that's correct, not a real close. Say so, so you're not seen as misunderstanding pooling.

**Fix:**
```java
try (Connection c = dataSource.getConnection();
     PreparedStatement ps = c.prepareStatement("SELECT name FROM users WHERE id = ?")) {
    ps.setLong(1, id);
    try (ResultSet rs = ps.executeQuery()) {
        return rs.next() ? rs.getString("name") : null;
    }
}
```
**What a strong senior would say:** *"The connection is handled correctly — closing a pooled connection just returns it to the pool. The gap is the `PreparedStatement` and `ResultSet`: they're not in try-with-resources, so on a hot path we can leak cursors. I'd wrap those too."*
**Interviewer is testing:** whether you know pooled-connection semantics *and* still catch the statement/rs leak (both a false-positive trap and a real bug in one).
**Expected:** stmt/rs leak. **Bonus:** explicitly calling the pooled-connection-close a non-issue. **Mention first:** stmt/rs leak.

## Topic 7 · Exercise 1 — Request counter

**Findings**
- 🔴 **DEFINITE BUG — Concurrency.** A singleton with a mutable `long requestCount++` from many request threads: `++` is read-modify-write, so updates are lost, and `long` reads/writes aren't guaranteed atomic on all JVMs. The count is wrong under load.
- 🟢 Visibility of `getRequestCount()` also unsynchronized.

**Fix:** `AtomicLong` (`incrementAndGet()`, `get()`), or `LongAdder` for a hot counter with high write contention.
**What a strong senior would say:** *"This is a shared-mutable-state race in a singleton — `count++` loses updates concurrently, and a plain `long` isn't safely published to readers. I'd use `AtomicLong`, or `LongAdder` if this is a very hot counter, since it scales better under contention."*
**Interviewer is testing:** recognizing that Spring singletons are shared across threads; atomicity vs visibility.
**Expected:** the race. **Bonus:** `LongAdder` vs `AtomicLong` trade-off; visibility of the getter. **Mention first:** the race.

## Topic 7 · Exercise 2 — Shared formatter

**Findings**
- ⚪ **NOT A BUG.** `DateTimeFormatter` is **immutable and thread-safe**; sharing one `static final` instance across threads is the *recommended* pattern. The comment is right, and there's nothing to fix. (Contrast with `SimpleDateFormat`, which is *not* thread-safe — that would be a bug.)
- 🟢 NIT: none material.

**What a strong senior would say:** *"This is correct and actually best practice — `DateTimeFormatter` is immutable, so a shared static instance is thread-safe and efficient. If this had been `SimpleDateFormat` I'd flag it immediately, but java.time formatters are safe to share. No change."*
**Interviewer is testing:** knowing the java.time-vs-legacy distinction and resisting a false positive.
**Mention first:** state it's correct; name the `SimpleDateFormat` contrast to prove you know *why*.

## Topic 8 · Exercise 1 — Membership check in a loop

**Findings**
- 🟠 **PERFORMANCE CONCERN (real at this scale).** `List.contains` is O(n); inside a loop over the input it's O(n×m). With a 50k blocklist and a few thousand SKUs that's ~10⁸ comparisons per request at 100 req/s — a CPU sink. Convert the blocklist to a `HashSet` once for O(1) lookups.

**Fix:**
```java
Set<String> blocked = new HashSet<>(blocklist);   // ideally build once, not per request
return skus.stream().filter(s -> !blocked.contains(s)).toList();
```
**What a strong senior would say:** *"`List.contains` is linear, so this is O(n·m) — at 50k × a few thousand per request it's a real hotspot, not a micro-optimization. I'd back the blocklist with a `HashSet`. Better still, build that set once and cache it rather than per call, since the blocklist changes rarely."*
**Interviewer is testing:** algorithmic-complexity reasoning grounded in the given scale (not hand-wavy "it's slow").
**Expected:** O(n·m) → Set. **Bonus:** hoist/cache the set; note it's justified by the stated scale. **Mention first:** the complexity fix.

## Topic 8 · Exercise 2 — Validation regex

**Findings**
- 🟡 **PERFORMANCE CONCERN.** `Pattern.compile` runs on every call. Compiling is relatively expensive; on a hot path, hoist to a `private static final Pattern`.
- 🟢 **Correctness.** The regex is simplistic (rejects some valid addresses, accepts some invalid). Acceptable for a basic check, but don't oversell it as RFC-compliant.

**Fix:** `private static final Pattern EMAIL = Pattern.compile(...);` reused per call.
**What a strong senior would say:** *"The functional concern is that we recompile the pattern every call — I'd make it a static final. The regex itself is a loose approximation; that's usually fine for a sanity check as long as we're not claiming full RFC 5322 validation."*
**Interviewer is testing:** the compile-once pattern; realistic take on email validation.
**Mention first:** hoist the compiled pattern.

## Topic 9 · Exercise 1 — In-memory cache

**Findings**
- 🟠 **DEFINITE BUG — Memory.** An unbounded `HashMap` used as a cache with high key cardinality over a long-running process is a memory leak → eventual OOM. No eviction, no size bound, no TTL.
- 🟡 **Concurrency.** Plain `HashMap` with concurrent `computeIfAbsent` from request threads is unsafe (can corrupt internal state / infinite loop historically). Needs a concurrent structure anyway.

**Fix:** a bounded cache with eviction — Caffeine (`maximumSize`/`expireAfterWrite`) or at minimum a size-capped `ConcurrentHashMap` + eviction policy.
**What a strong senior would say:** *"This grows without bound — with wide key variety over weeks it's an OOM waiting to happen, and the plain `HashMap` isn't even thread-safe for concurrent `computeIfAbsent`. I'd use Caffeine with a max size and a TTL. The right bound depends on the key cardinality and hit rate, which I'd measure."*
**Interviewer is testing:** recognizing unbounded caches as leaks; concurrency of `HashMap`.
**Expected:** unbounded growth. **Bonus:** HashMap-not-threadsafe; eviction policy trade-offs. **Mention first:** unbounded memory.

## Topic 9 · Exercise 2 — Report from full table load

**Findings**
- 🟠 **DEFINITE BUG — Memory/Scalability.** `findAll()` loads millions of `Customer` entities into the persistence context and heap at once → huge allocation, GC pressure, likely OOM. Also hydrates full entities when only `name` is needed.
- 🟡 Should stream from the DB and/or project just the name column.

**Fix:** projection query returning only names + streaming/paging (`Stream<String> streamAllNames()` with `@Query`, or a `Pageable` loop, writing to the output incrementally rather than joining a giant string in memory).
**What a strong senior would say:** *"Loading several million full entities to read one field will blow the heap and thrash GC. I'd project just the name column and stream results to the output rather than materializing everything — cursor-based streaming or paging, writing as we go."*
**Interviewer is testing:** the `findAll()`-on-a-big-table reflex; projection + streaming.
**Expected:** full load OOM. **Bonus:** projection over full entity; incremental write. **Mention first:** the full-table load.

## Topic 10 · Exercise 1 — Bulk status update

**Findings**
- 🟠 **PERFORMANCE/DB CONCERN.** Per-ID `findById` + `save` is 2 queries × N (tens of thousands) → a query storm and huge round-trip cost. Replace with a single bulk `UPDATE ... WHERE id IN (:ids)` (or batched chunks of IN).
- 🟡 **Transaction.** No `@Transactional`; each save may auto-commit separately, so a mid-batch failure leaves a partial update with no atomicity story.

**Fix:**
```java
@Modifying @Query("update Order o set o.status = :s where o.id in :ids")
int markShipped(@Param("s") Status s, @Param("ids") Collection<Long> ids);
```
chunk `ids` (e.g. 1–5k per IN) and wrap in a transaction.
**What a strong senior would say:** *"This is 2N queries for what should be one bulk UPDATE. At tens of thousands of IDs that's a serious DB hit. I'd do a single `UPDATE ... WHERE id IN (...)`, chunked to keep the IN list sane, inside a transaction. Trade-off: a bulk update bypasses entity lifecycle callbacks/optimistic-lock checks, so if we depend on those I'd reconsider."*
**Interviewer is testing:** spotting N-query patterns; knowing bulk update trade-offs.
**Expected:** per-row queries → bulk. **Bonus:** IN-chunking, transaction, bulk-update-bypasses-callbacks caveat. **Mention first:** the query storm.

## Topic 10 · Exercise 2 — Search by name

**Findings**
- 🔴 **DEFINITE BUG — Security (SQL injection).** `name` is concatenated straight into SQL. `'; DROP TABLE products; --` style input executes. Use a bound parameter.
- 🟡 **PERFORMANCE.** Leading-wildcard `LIKE '%x%'` can't use a normal B-tree index → full scan on a large table. Consider trigram/GIN index or full-text search.

**Fix:** `jdbcTemplate.query("... WHERE name LIKE ?", ps -> ..., mapper)` with `"%" + name + "%"` bound as a parameter (value, not concatenated SQL).
**What a strong senior would say:** *"First and foremost this is SQL injection — `name` goes straight into the query string. I'd bind it as a parameter immediately. Secondly, a leading-wildcard LIKE won't use a standard index, so on a large table it's a full scan; if search is important I'd look at a trigram index or Postgres full-text."*
**Interviewer is testing:** injection reflex + index-usage awareness in one exercise.
**Expected:** injection. **Bonus:** leading-wildcard non-sargable. **Mention first:** injection (🔴).

## Topic 11 · Exercise 1 — Transfer with external call

**Findings**
- 🔴 **DEFINITE BUG — Transaction boundary.** A synchronous HTTP call (`fraudClient.report`, up to 30s) sits *inside* `@Transactional`. The DB connection and row locks are held for the entire external round-trip → under load, connection-pool exhaustion and lock contention cascade into an outage.
- 🟠 **Reliability.** If `fraudClient` throws, the whole transfer rolls back — coupling money movement to a reporting call. And no timeout on the client.
- 🟡 **Lost update** risk on balances without `@Version`/locking (see Topic on optimistic locking).

**Fix:** commit the transfer, then report *after* the transaction (e.g. publish an event / call post-commit), with a timeout and its own retry. Keep the transaction to the DB work only.
**What a strong senior would say:** *"The critical issue is doing a 30-second-capable HTTP call inside the transaction — we hold a DB connection and locks across the network, which exhausts the pool under load. Fraud reporting shouldn't gate or roll back the money movement either. I'd move it out of the transaction, ideally to an after-commit event, with its own timeout and retry."*
**Interviewer is testing:** the #1 transaction anti-pattern — I/O inside a tx; decoupling side-effects.
**Expected:** I/O in transaction. **Bonus:** don't-roll-back-on-report, timeout, balance locking. **Mention first:** I/O-in-transaction (🔴).

## Topic 11 · Exercise 2 — Self-invoked @Transactional

**Findings**
- 🔴 **DEFINITE BUG — Spring proxy / self-invocation.** `processBatch` calls `processOne` via `this`, so the Spring proxy is bypassed and `@Transactional` on `processOne` **does nothing** — the two saves aren't in the intended per-invoice transaction. The "isolate failures" goal is not met.
- 🟡 The intent (per-item transaction) also needs `REQUIRES_NEW` semantics if `processBatch` itself were transactional.

**Fix:** move `processOne` to a separate bean and inject it (proxy applies), or self-inject the proxy, or restructure so the transactional entry point is called externally.
**What a strong senior would say:** *"This is the classic self-invocation trap: calling `processOne` on `this` skips the proxy, so `@Transactional` never activates — each invoice isn't actually isolated. I'd extract `processOne` into its own bean so the proxy wraps it, and if the batch method becomes transactional I'd use `REQUIRES_NEW` to truly isolate each invoice."*
**Interviewer is testing:** understanding *how* `@Transactional` works (proxies), not just that it exists.
**Expected:** self-invocation bypass. **Bonus:** REQUIRES_NEW for isolation. **Mention first:** the proxy bypass (🔴).

## Topic 12 · Exercise 1 — Customer controller

**Findings**
- 🔴 **DEFINITE BUG — HTTP semantics.** `DELETE` exposed via `GET /delete/{id}` — a GET must be safe/idempotent and not mutate. Crawlers/prefetch/caches can delete customers. Use `@DeleteMapping("/{id}")`.
- 🟠 **DESIGN — entity leak + lazy loading.** Returning the `Customer` entity with a lazy `orders` collection risks `LazyInitializationException` during serialization or accidental N+1/over-exposure of the schema. Return a DTO.
- 🟢 Delete should return `204 No Content`, not a `200 "deleted"` string.

**Fix:** proper verbs, DTOs, status codes.
**What a strong senior would say:** *"The serious one is deletion behind a GET — that violates HTTP safety and can be triggered by anything that prefetches links. It must be DELETE. I'd also stop returning the entity directly: DTO-map it to avoid lazy-loading surprises and schema leakage, and return 204 on delete."*
**Interviewer is testing:** HTTP method semantics; entity-vs-DTO discipline.
**Expected:** GET-that-deletes. **Bonus:** entity leak/lazy, status codes. **Mention first:** GET-deletes (🔴).

## Topic 12 · Exercise 2 — List endpoint

**Findings**
- 🟠 **DEFINITE BUG — API/Scalability.** Unbounded list of ~2M products serialized in one response → massive payload, memory, and latency; can OOM the service and the client. Must paginate (`Pageable`, max page size) and page at the DB level.
- 🟡 Mapping happens after loading everything into memory — combine with DB paging.

**Fix:** accept `Pageable`, return a `Page<ProductDto>`, enforce a max size, project at the query.
**What a strong senior would say:** *"At 2M rows this endpoint can't return everything — it'll blow memory and time out. I'd make it paginated with a bounded page size and page at the database, not in memory. If clients need bulk export, that's a separate streaming endpoint."*
**Interviewer is testing:** the unbounded-collection endpoint reflex.
**Mention first:** missing pagination.

## Topic 13 · Exercise 1 — Login query

**Findings**
- 🔴 **DEFINITE BUG — SQL injection.** Username/password concatenated into SQL → auth bypass (`' OR '1'='1' --`). Parameterize.
- 🔴 **DEFINITE BUG — Security (plaintext passwords).** Comparing `password` directly implies passwords are stored in plaintext. Must store a salted hash (bcrypt/Argon2) and compare via the hash.
- 🟡 Count-based auth can't easily distinguish/lock accounts; no rate limiting / lockout.

**Fix:** parameterized query fetching the stored hash by username, then `passwordEncoder.matches(raw, hash)`; never store or compare plaintext.
**What a strong senior would say:** *"Two critical issues: this is trivially SQL-injectable for an auth bypass, and it's comparing plaintext passwords, which means they're stored in plaintext. I'd parameterize, and I'd store salted bcrypt/Argon2 hashes and verify with a constant-time match. I'd also add rate limiting to slow brute force."*
**Interviewer is testing:** stacked security failures; do you catch *both* injection and plaintext.
**Expected:** injection + plaintext. **Bonus:** lockout/rate-limit, constant-time compare. **Mention first:** either 🔴; injection usually first.

## Topic 13 · Exercise 2 — Payment logging

**Findings**
- 🔴 **DEFINITE BUG — Security (PII/PCI).** Logging full PAN, expiry, and CVV is a severe compliance violation (PCI-DSS forbids storing CVV at all) and leaks card data to anyone with log access. Remove entirely; at most log a masked last-4 and never the CVV.
- 🟢 Observability: keep a correlation/transaction id instead.

**Fix:** drop card fields from logs; log masked reference + amount + a transaction id.
**What a strong senior would say:** *"This logs full card number and CVV — that's a serious PCI violation; CVV must never be stored or logged, even transiently. I'd strip these immediately and log only a masked last-4 with a correlation id. This is the kind of finding I'd escalate, not just leave as a PR comment."*
**Interviewer is testing:** sensitive-data-in-logs reflex and knowing CVV is special.
**Mention first:** the card/CVV logging (🔴).

## Topic 14 · Exercise 1 — Debug logging in a hot loop

**Findings**
- 🟡 **PERFORMANCE CONCERN.** Even at INFO level, `"Pricing item " + item.getId() + ... + item.toString()` is **eagerly evaluated** before `log.debug` decides to discard it — string building and `toString` run for every item on every batch. Use parameterized logging (`log.debug("Pricing item {} data {}", id, item)`) which defers formatting until the level is enabled.
- 🟢 Even parameterized, per-item debug in huge batches can be noisy — consider sampling.

**Fix:** parameterized SLF4J logging (no `+` concatenation).
**What a strong senior would say:** *"The trap is that the message string is built with `+` before `debug` checks the level, so we pay the concatenation and `toString` cost per item even in production where debug is off. Switching to SLF4J's `{}` placeholders defers that. So the level does *not* fully protect us here as written."*
**Interviewer is testing:** understanding eager arg evaluation vs guarded logging — a subtle, senior detail.
**Mention first:** eager concatenation cost.

## Topic 14 · Exercise 2 — Silent failure

**Findings**
- 🟠 **DEFINITE BUG — Observability.** Comment says "pick it up next run," but nothing records the failure, so operators are blind (matches the complaint). Also "next run" implies the update is retried — is it? If `push` isn't idempotent, silent retry could duplicate.
- 🟡 Needs logging with context, a failure metric/alert, and a defined retry/DLQ strategy.

**Fix:** log each failure with the update id + cause, increment an error metric, and define whether failures are retried (and whether `push` is idempotent).
**What a strong senior would say:** *"The operators' complaint is the design: failures are swallowed with no log or metric, so nobody knows sync broke. I'd log with context and emit a metric that can alert. And I'd clarify the 'next run' assumption — if we reprocess, is `push` idempotent, or do we risk duplicates?"*
**Interviewer is testing:** connecting a silent catch to a real ops symptom; idempotency of retries.
**Good clarifying question:** *"Are these updates reprocessed on the next run, and is `push` idempotent?"* — **Why it matters:** determines whether silent-continue is safe or causes duplicates.
**Mention first:** the invisible failure.

## Topic 15 · Exercise 1 — A test that always passes

**Findings**
- 🟠 **DEFINITE BUG — Testing.** `assertNotNull` on a primitive `double` result is meaningless — the value is autoboxed and can never be null, so the test passes regardless of correctness (including the discount bug from Topic 1). Assert the actual expected value: `assertEquals(90.0, result, 0.001)`.
- 🟢 No boundary/edge cases (0%, 100%, negative).

**Fix:** assert the computed value; add boundary cases.
**What a strong senior would say:** *"This test can't fail — asserting a primitive double is non-null is always true, so it verifies nothing. It would pass even with a completely wrong discount. I'd assert the exact expected total and add boundary cases at 0 and 100 percent."*
**Interviewer is testing:** spotting assertion theater — tests that give false confidence.
**Mention first:** the vacuous assertion.

## Topic 15 · Exercise 2 — Repository test with mocks

**Findings**
- 🟠 **DESIGN CONCERN — Testing the wrong thing.** The scenario wants to verify a *custom JPQL query*, but the test mocks the repository and asserts on the stubbed return — so it tests Mockito, not the query. A broken `findOverdue` JPQL would still pass. Use `@DataJpaTest` (ideally Testcontainers with real Postgres) to exercise the actual query against a database.
- 🟢 The service-level mock test is fine *for service logic* — but it doesn't cover the query.

**Fix:** add a `@DataJpaTest` integration test that inserts overdue/non-overdue rows and asserts the query returns the right ones.
**What a strong senior would say:** *"Mocking the repository means we're asserting on our own stub — the actual JPQL is never executed, so a broken query passes CI. To verify the query I'd use `@DataJpaTest`, ideally with Testcontainers so it runs against real Postgres, seed representative rows, and assert the results. The mock test still has value for service logic, just not for the query."*
**Interviewer is testing:** knowing what each test level actually covers; repository tests need a real DB.
**Mention first:** the query isn't tested.

## Topic 16 · Exercise 1 — Status handling

**Findings**
- 🟡 **MAINTAINABILITY / DESIGN.** Magic-number statuses (`1,2,3,4`) with clarifying comments are a smell; a new status means editing this chain, and there's no default/else for unknown statuses (silent no-op). Use an `enum` for `Status`, and consider a `switch` (exhaustive with enums) or a state/strategy map so adding a status is localized and the compiler flags missing cases.
- 🟢 SRP: the handler both branches and performs the work inline.

**Fix:** `enum OrderStatus`; `switch` on it (Java 17 switch can be exhaustive) or a `Map<Status, Consumer<Order>>`.
**What a strong senior would say:** *"The magic numbers and if-else chain make this fragile — adding a status touches this method with no compiler help, and there's no handling of unexpected values. I'd model status as an enum and use an exhaustive switch so new statuses are caught at compile time; a strategy map is an option if the actions grow."*
**Interviewer is testing:** recognizing OCP/maintainability smells; enum modeling.
**Mention first:** magic numbers + no default, not formatting.

## Topic 16 · Exercise 2 — Boolean flags

**Findings**
- 🟡 **MAINTAINABILITY.** `send(to, body, true, false, true)` is unreadable at call sites — no one can tell what the booleans mean, and it's easy to transpose them (a "boolean parameter" smell). Replace with an options object/builder or named enums (`EmailOptions.builder().html(true).bcc(true).build()`), or distinct methods.
- 🟢 Adding a 4th flag compounds the problem (combinatorial).

**Fix:** parameter object / builder; or `EnumSet<EmailFlag>`.
**What a strong senior would say:** *"Positional booleans are a readability and correctness hazard — `true, false, true` tells the reader nothing and is easy to get out of order. With 15 call sites I'd introduce an options object or builder so each flag is named at the call site. It also makes adding options non-breaking."*
**Interviewer is testing:** the boolean-parameter smell and a clean remedy.
**Mention first:** unreadable/transposable flags.

## Topic 17 · Exercise 1 — Service doing everything

**Findings**
- 🟠 **DESIGN CONCERN — SRP.** `placeOrder` validates, persists, builds+sends email (inline SMTP), and calls a shipping REST API — four responsibilities in one method. Hard to test, change, or reuse; email/HTTP failures are entangled with order placement.
- 🟠 **Reliability.** `new RestTemplate()` per call (no timeout, no reuse); SMTP connect/disconnect inline with no error handling; a shipping-API failure isn't isolated from the DB write (no transaction/outbox story).
- 🟡 **DIP.** Depends on concrete SMTP/RestTemplate rather than abstractions.

**Fix:** split into `OrderService` (persist) + `NotificationService` + `ShippingClient` (injected, with timeouts); decouple side-effects via events/after-commit; wrap persistence in a transaction.
**What a strong senior would say:** *"This method owns four responsibilities, which makes it fragile and untestable — a flaky SMTP or shipping call now sits in the middle of order placement. I'd extract notification and shipping into their own injected collaborators, drive them off an after-commit event so they don't block or roll back the order, and give the HTTP client real timeouts instead of newing a RestTemplate per call."*
**Interviewer is testing:** SRP + coupling of side-effects + reliability, holistically.
**Expected:** SRP split. **Bonus:** events/after-commit, RestTemplate reuse+timeout, transaction boundary. **Mention first:** the entangled side-effects (reliability), then SRP.

## Topic 17 · Exercise 2 — Hard-wired dependency

**Findings**
- 🟡 **DESIGN CONCERN — DIP.** `new ItextPdfGenerator()` inside the method hard-couples `ReportService` to a concrete library, defeating the stated goal of swapping it later and making unit tests unable to substitute a fake. Depend on a `PdfGenerator` interface injected via the constructor.

**Fix:**
```java
public ReportService(PdfGenerator generator) { this.generator = generator; }
public byte[] generate(ReportData data) { return generator.render(data); }
```
**What a strong senior would say:** *"Since the goal is to swap the PDF library later, newing the concrete generator here is exactly backwards — it hard-wires the dependency and blocks testing with a fake. I'd depend on a `PdfGenerator` interface and inject it, so the implementation is a wiring detail, not baked into this class."*
**Interviewer is testing:** dependency inversion and its testability payoff.
**Mention first:** the concrete instantiation vs injection.

## Topic 18 · Exercise 1 — In-memory rate limiter

**Findings**
- 🟠 **DEFINITE BUG — Scalability (distributed state).** The count lives in one pod's heap. Across 6 non-sticky pods, each enforces its own 100/min, so a user effectively gets ~600/min; counts are lost on restart. In-memory limiting can't be correct across instances. Use a shared store (Redis with atomic INCR + expiry, or a token-bucket in Redis / an API-gateway limiter).
- 🟡 **Correctness — no time window.** `counts` never resets, so it's a lifetime counter, not per-minute. Needs a sliding/fixed window with expiry.

**Fix:** Redis-backed counter keyed `user:minute` with `INCR` + TTL, or a gateway/library (Bucket4j + Redis).
**What a strong senior would say:** *"Two problems: the state is per-pod, so with 6 instances and no stickiness the real limit is ~6×, and it's lost on restart — in-memory limiting can't be globally correct. And there's no time window, so it's really a lifetime counter. I'd move this to Redis with atomic increments and a per-window TTL, or push it to the gateway."*
**Interviewer is testing:** reasoning about state across instances — a core distributed/scalability idea.
**Expected:** per-instance state. **Bonus:** missing time window, restart loss, Redis/gateway options. **Mention first:** the cross-instance correctness gap.

## Topic 18 · Exercise 2 — Aggregate in memory

**Findings**
- 🟠 **DEFINITE BUG — Scalability.** Loading millions of orders to sum one column, every 10 seconds, is enormous heap + query cost for a value the database computes trivially. Use a DB aggregate: `SELECT SUM(amount) FROM orders`.
- 🟡 Even that runs every 10s over a huge table — consider a materialized/cached value or incremental maintenance.

**Fix:** `@Query("select coalesce(sum(o.amount),0) from Order o") BigDecimal totalRevenue();` and cache/refresh appropriately.
**What a strong senior would say:** *"This pulls millions of rows into memory to compute a sum the database can do in one query — I'd push it down to a `SUM`. Given it refreshes every 10 seconds on a large table, I'd also cache the result or maintain it incrementally rather than re-aggregating constantly."*
**Interviewer is testing:** push computation to the DB; caching hot aggregates.
**Mention first:** in-memory aggregation of a huge table.

## Topic 19 · Exercise 1 — External client, no timeout

**Findings**
- 🟠 **DEFINITE BUG — Reliability.** A default `RestTemplate` has effectively no connect/read timeout, so a hung geocoding dependency ties up the request thread indefinitely; enough hangs exhaust the thread pool and take the whole service down (cascading failure). Configure explicit connect + read timeouts.
- 🟡 No circuit breaker / fallback for a repeatedly failing dependency; no retry policy.
- 🟢 **Security (minor).** `address` is concatenated into the URL unencoded — encode query params.

**Fix:** build `RestTemplate`/`RestClient` with timeouts; wrap in Resilience4j circuit breaker + timeout + fallback; URL-encode params.
**What a strong senior would say:** *"The killer here is no timeout — a hanging dependency holds request threads until the pool is exhausted and unrelated requests fail too. I'd set explicit connect and read timeouts, then add a circuit breaker with a fallback so a sick dependency degrades gracefully instead of cascading. Minor: the address should be URL-encoded."*
**Interviewer is testing:** timeouts as the first line of resilience; cascading-failure reasoning.
**Expected:** missing timeout. **Bonus:** circuit breaker/fallback, encoding. **Mention first:** missing timeout.

## Topic 19 · Exercise 2 — Retry loop

**Findings**
- 🔴 **DEFINITE BUG — Reliability/Correctness.** Retrying a **non-idempotent** charge risks multiple charges: if the first call succeeded but the response was lost (timeout), the retry charges again. Retries here need an **idempotency key** so the gateway dedups.
- 🟠 **No backoff.** Immediate tight-loop retries hammer a struggling gateway (retry storm); use exponential backoff + jitter and cap attempts.
- 🟢 Catches only `GatewayException` — fine, but ensure truly non-retryable errors (e.g. card declined) aren't retried.

**Fix:** pass a stable idempotency key per logical charge; add exponential backoff + jitter; don't retry business declines.
**What a strong senior would say:** *"Retrying a charge that isn't idempotent can double-charge on a lost response, so before anything I'd require an idempotency key so the gateway dedups. I'd also add exponential backoff with jitter instead of hammering, and make sure we only retry transient failures — a hard decline shouldn't be retried at all."*
**Interviewer is testing:** the idempotency-vs-retry relationship; backoff; retryable vs terminal errors.
**Expected:** non-idempotent retry. **Bonus:** backoff/jitter, terminal-error classification. **Mention first:** double-charge risk (🔴).

## Topic 20 · Exercise 1 — DB write then event publish

**Findings**
- 🔴 **DEFINITE BUG — Distributed (dual write).** `save` (DB) and `kafkaTemplate.send` (broker) are **not atomic**. If the tx commits but the publish fails (or vice-versa), DB and downstream diverge — orders with no event, or events for rolled-back orders. And `send` inside the transaction publishes *before* commit, so a rollback still emits the event.
- 🟠 Fix with the **Transactional Outbox**: write an outbox row in the same DB transaction, and a relay (CDC/Debezium or a poller) publishes to Kafka after commit — at-least-once with atomicity to the DB.

**Fix:** outbox table written in-tx; separate publisher; consumers made idempotent (see next exercise).
**What a strong senior would say:** *"This is a dual-write: the DB commit and the Kafka publish can't be atomic, so we'll eventually get orders without events or events without orders — and publishing inside the transaction means a rollback still emits. The standard fix is a transactional outbox: persist the event in the same transaction, then relay it to Kafka separately, and make consumers idempotent since delivery is at-least-once."*
**Interviewer is testing:** recognizing dual-write inconsistency and naming the outbox pattern.
**Expected:** dual-write. **Bonus:** publish-before-commit nuance, outbox + CDC, downstream idempotency. **Mention first:** the dual-write (🔴).

## Topic 20 · Exercise 2 — Event consumer

**Findings**
- 🔴 **DEFINITE BUG — Idempotency (at-least-once).** At-least-once means duplicates happen; this consumer decrements stock every time, so a redelivered message double-decrements. Needs idempotency — dedup by event id / processed-events table, or an idempotent operation keyed to the event.
- 🟠 **Concurrency — lost update.** `read stock, subtract 1, save` from concurrent consumers/partitions is a read-modify-write race → lost decrements. Use an atomic DB update (`UPDATE product SET stock = stock - 1 WHERE id = ? AND stock > 0`) or optimistic locking.
- 🟡 **Correctness.** No check for stock hitting zero/negative.
- 🟢 `event.getProductId()` vs the `OrderPlaced(order.getId())` published earlier — payload may not even carry productId (cross-exercise inconsistency worth questioning).

**Fix:** atomic conditional `UPDATE`; dedup by event id; guard non-negative stock.
**What a strong senior would say:** *"Two distributed issues: the topic is at-least-once, so a duplicate delivery double-decrements — I'd dedup by event id or make the decrement idempotent. And the read-modify-write on stock races across concurrent consumers, causing lost updates — I'd do it as a single atomic conditional UPDATE guarded by `stock > 0`, or use optimistic locking. I'd also confirm the event actually carries a product id."*
**Interviewer is testing:** at-least-once implications + concurrent update correctness together.
**Expected:** duplicate handling + lost update. **Bonus:** atomic conditional update, non-negative guard, payload consistency. **Mention first:** idempotency (🔴), then the lost update.

---

# Senior Code-Review Master Checklist (Part 1)

Run top-to-bottom, **highest severity first**. Speak each finding as: *observation → why it matters → severity → fix → trade-off.*

1. **Correctness** — arithmetic, operators, branch order, dead/unreachable code.
2. **Business logic** — cumulative state (refunds/limits), boundaries inclusive vs exclusive.
3. **Nulls / edge cases** — empty collection, div-by-zero, missing/optional fields, overflow.
4. **Exceptions** — never swallow silently; restore interrupts; scope `catch`; don't roll back on non-critical side-effects.
5. **Resources** — try-with-resources for streams/statements/rs; close on the failure path; pooled-connection close is *not* a leak.
6. **Collections** — right structure (`Set` for membership); no unbounded growth; no in-place mutation of caller data.
7. **Concurrency** — shared mutable state in singletons; `++` races; atomicity vs visibility; interrupt handling; `HashMap` isn't thread-safe; `DateTimeFormatter` *is*.
8. **Performance** — O(n·m) loops, compile-once patterns, hoist invariants; justify with the given scale, else "I'd measure."
9. **Memory** — unbounded caches (leak), `findAll()` on big tables → project + stream.
10. **Database** — per-row query storms → bulk ops; leading-wildcard LIKE → full scan; bind params.
11. **Transactions** — no I/O inside a tx; self-invocation bypasses the proxy; rollback rules; balance locking.
12. **Spring** — singletons are shared; constructor injection; DTOs not entities from controllers.
13. **JPA/Hibernate** — lazy access after session, N+1, projections (covered deeply in Part 5).
14. **REST/API** — correct verbs (never GET-that-mutates), status codes, pagination, idempotency.
15. **Security** — SQL injection (parameterize), no PII/CVV/passwords in logs, hash passwords, authz/ownership.
16. **Testing** — real assertions (not `assertNotNull` on a primitive), test the query with `@DataJpaTest`, no time/order coupling.
17. **Logging** — parameterized (`{}`) not `+`; guarded hot-path logs; never log secrets.
18. **Observability** — no silent failures; log with context + metric + correlation id.
19. **Scalability** — state across instances (rate limits/counters need shared store); push aggregates to the DB.
20. **Reliability** — timeouts everywhere; circuit breaker + fallback; retries only with idempotency + backoff.
21. **Distributed systems** — dual-write → outbox; at-least-once → idempotent, deduped consumers; atomic conditional updates.
22. **Maintainability** — magic numbers → enums; boolean-param smell → options object; exhaustive switches.
23. **SOLID / design** — SRP (split entangled side-effects); DIP (inject abstractions, don't `new` concretes).
24. **Production readiness** — DB/broker/API down or slow? retried twice? data lost or duplicated? observable? bounded memory?

**Golden rules:** lead with 🔴/🟠; name at least one *not-a-bug* to show judgment; pair every "this is slow" with "I'd measure/profile"; ask a clarifying question when correctness depends on requirements (thread-safety guarantees, idempotency, dataset size) instead of asserting.

---

*End of Part 1. Say **"next"** for Part 2 — Java Code Review (27 topics × 2 = 54 exercises), built to the same standard.*
