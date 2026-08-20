# Part 4 — Spring Boot Code Review (Practice Workbook)

**64 exercises — 32 topics × 2.** Java 17+, Spring Boot 3.x, Spring MVC + Spring Data JPA, PostgreSQL. Review cold; write findings before the answer key.

> Multiple bugs per exercise are common. Some code *looks* wrong but is correct. Some comments are misleading. Verify everything — especially anything involving proxies, transactions, and bean scope.

**Legend:** 🔴 Critical · 🟠 High · 🟡 Medium · 🟢 Low · ⚪ Nit

---

# EXERCISES

## Topic 1 — IoC

### Exercise 1 — Building collaborators by hand

**Scenario**
`OrderService` is a Spring `@Service`. It depends on an email client and a repository.

**Code**
```java
@Service
public class OrderService {
    private final EmailClient emailClient = new SmtpEmailClient("smtp.corp.local", 25);
    private final OrderRepository repository = new JpaOrderRepositoryImpl();

    public void place(Order order) {
        repository.save(order);
        emailClient.send(order.getEmail(), "Order placed");
    }
}
```

**Your Task**
Review how dependencies are obtained and whether this fits the Spring container model.

---

### Exercise 2 — Service locator

**Scenario**
`PaymentProcessor` resolves the right gateway bean at runtime based on the payment method.

**Code**
```java
@Service
public class PaymentProcessor implements ApplicationContextAware {
    private ApplicationContext ctx;

    @Override
    public void setApplicationContext(ApplicationContext c) { this.ctx = c; }

    public void process(Payment p) {
        PaymentGateway gw = (PaymentGateway) ctx.getBean(p.getMethod() + "Gateway");
        gw.charge(p);
    }
}
```

**Your Task**
Review this dependency-resolution approach.

---

## Topic 2 — Dependency Injection

### Exercise 1 — Ambiguous bean

**Scenario**
Two `NotificationSender` beans exist: `EmailSender` and `SmsSender`. `AlertService` injects one.

**Code**
```java
@Component class EmailSender implements NotificationSender { /* ... */ }
@Component class SmsSender implements NotificationSender { /* ... */ }

@Service
public class AlertService {
    private final NotificationSender sender;
    public AlertService(NotificationSender sender) { this.sender = sender; }
}
```

**Your Task**
Review whether this wires up correctly.

---

### Exercise 2 — Injecting into a static field

**Scenario**
`AuditUtil` is a utility whose methods are called statically from many places; it needs the repository.

**Code**
```java
@Component
public class AuditUtil {
    @Autowired
    private static AuditRepository repository;

    public static void record(String event) {
        repository.save(new AuditRecord(event)); // NPE?
    }
}
```

**Your Task**
Review whether the injection works.

---

## Topic 3 — Constructor injection

### Exercise 1 — Two constructors

**Scenario**
`ReportService` has two constructors; one takes an optional cache.

**Code**
```java
@Service
public class ReportService {
    private final ReportRepository repo;
    private final ReportCache cache;

    public ReportService(ReportRepository repo) {
        this(repo, null);
    }
    public ReportService(ReportRepository repo, ReportCache cache) {
        this.repo = repo;
        this.cache = cache;
    }
}
```

**Your Task**
Review whether Spring can construct this bean unambiguously.

---

### Exercise 2 — Constructor injection with final fields

**Scenario**
`InvoiceService` uses constructor injection. A reviewer commented: *"Switch this to `@Autowired` field injection to cut the boilerplate."*

**Code**
```java
@Service
public class InvoiceService {
    private final InvoiceRepository repository;
    private final TaxCalculator taxCalculator;

    public InvoiceService(InvoiceRepository repository, TaxCalculator taxCalculator) {
        this.repository = repository;
        this.taxCalculator = taxCalculator;
    }
}
```

**Your Task**
Review this. Evaluate the reviewer's suggestion — is a change warranted?

---

## Topic 4 — Field injection

### Exercise 1 — Dependency used during construction

**Scenario**
`CacheWarmer` preloads a cache as soon as it's created.

**Code**
```java
@Component
public class CacheWarmer {
    @Autowired
    private ProductRepository productRepository;

    private final Map<Long, Product> cache;

    public CacheWarmer() {
        cache = new HashMap<>();
        for (Product p : productRepository.findAll()) { // used in constructor
            cache.put(p.getId(), p);
        }
    }
}
```

**Your Task**
Review whether this initializes correctly.

---

### Exercise 2 — Field injection and testability

**Scenario**
`ShippingService` uses field injection. The team writes a plain unit test with `new ShippingService()`.

**Code**
```java
@Service
public class ShippingService {
    @Autowired private RateProvider rateProvider;
    @Autowired private AddressValidator addressValidator;

    public BigDecimal quote(Shipment s) {
        addressValidator.validate(s.getAddress());
        return rateProvider.rate(s);
    }
}

// test:
@Test void quotes() {
    ShippingService svc = new ShippingService();
    svc.quote(sampleShipment()); // ?
}
```

**Your Task**
Review the injection style and its impact on testing.

---

## Topic 5 — Bean lifecycle

### Exercise 1 — Work in the constructor

**Scenario**
`ConnectionPoolManager` establishes connections to a downstream system when the app starts.

**Code**
```java
@Component
public class ConnectionPoolManager {
    private final DownstreamClient client;

    public ConnectionPoolManager(DownstreamClient client) {
        this.client = client;
        client.connectAll(); // opens sockets to downstream at construction
    }
}
```

**Your Task**
Review the timing/lifecycle of this initialization.

---

### Exercise 2 — No cleanup

**Scenario**
`SchedulerService` creates its own executor for periodic work.

**Code**
```java
@Service
public class SchedulerService {
    private final ScheduledExecutorService scheduler =
            Executors.newScheduledThreadPool(4);

    @PostConstruct
    public void start() {
        scheduler.scheduleAtFixedRate(this::poll, 0, 10, TimeUnit.SECONDS);
    }
    // no @PreDestroy
}
```

**Your Task**
Review the lifecycle management of the executor.

---

## Topic 6 — Bean scopes

### Exercise 1 — Stateful singleton

**Scenario**
`RequestContextHolder` stores the current user for use during a request. It's a normal `@Service`.

**Code**
```java
@Service
public class CurrentUserService {
    private String currentUser; // set at the start of each request

    public void setCurrentUser(String user) { this.currentUser = user; }
    public String getCurrentUser() { return currentUser; }
}
```

**Your Task**
Review the scope/state design under concurrent requests.

---

### Exercise 2 — Prototype into singleton

**Scenario**
`OrderProcessor` (singleton) needs a fresh `PricingContext` (prototype) per call.

**Code**
```java
@Component
@Scope("prototype")
class PricingContext { /* per-call mutable state */ }

@Service
public class OrderProcessor {
    private final PricingContext context; // injected once

    public OrderProcessor(PricingContext context) { this.context = context; }

    public void process(Order o) {
        context.reset();
        context.apply(o); // expects a fresh context each call
    }
}
```

**Your Task**
Review whether each call gets a fresh prototype instance.

---

## Topic 7 — Circular dependencies

### Exercise 1 — Mutual constructor dependency

**Scenario**
`OrderService` and `InventoryService` each need the other.

**Code**
```java
@Service
public class OrderService {
    private final InventoryService inventory;
    public OrderService(InventoryService inventory) { this.inventory = inventory; }
}

@Service
public class InventoryService {
    private final OrderService orders;
    public InventoryService(OrderService orders) { this.orders = orders; }
}
```

**Your Task**
Review what happens at application startup.

---

### Exercise 2 — "Fixed" with @Lazy

**Scenario**
The same cycle as above, "resolved" so the app starts.

**Code**
```java
@Service
public class OrderService {
    private final InventoryService inventory;
    // Fixed the circular dependency
    public OrderService(@Lazy InventoryService inventory) { this.inventory = inventory; }
}

@Service
public class InventoryService {
    private final OrderService orders;
    public InventoryService(OrderService orders) { this.orders = orders; }
}
```

**Your Task**
Review whether `@Lazy` is a proper fix here.

---

## Topic 8 — Spring configuration

### Exercise 1 — Missing property

**Scenario**
`FeatureConfig` reads a threshold from configuration.

**Code**
```java
@Component
public class FeatureConfig {
    @Value("${feature.max-batch-size}")
    private int maxBatchSize;

    public int getMaxBatchSize() { return maxBatchSize; }
}
```
(`feature.max-batch-size` is not defined in any properties file for the `prod` profile.)

**Your Task**
Review the configuration robustness.

---

### Exercise 2 — Inter-bean method call

**Scenario**
`AppConfig` defines two beans; one uses the other.

**Code**
```java
@Component
public class AppConfig {

    @Bean
    public DataSourceProps props() {
        return new DataSourceProps("jdbc:postgresql://db/app");
    }

    @Bean
    public DataSource dataSource() {
        return build(props()); // calls the @Bean method directly
    }
}
```

**Your Task**
Review whether `dataSource()` and any other consumer share the same `props()` instance.

---

## Topic 9 — Controller design

### Exercise 1 — Business logic in the controller

**Scenario**
`OrderController` handles order creation.

**Code**
```java
@RestController
@RequestMapping("/orders")
public class OrderController {
    private final OrderRepository orderRepository;
    private final EmailClient emailClient;

    @PostMapping
    public Order create(@RequestBody OrderRequest req) {
        if (req.getItems().isEmpty()) throw new IllegalArgumentException();
        Order o = new Order();
        o.setItems(req.getItems());
        o.setTotal(req.getItems().stream()
                      .map(Item::getPrice).reduce(BigDecimal.ZERO, BigDecimal::add));
        o.setStatus("NEW");
        orderRepository.save(o);
        emailClient.send(req.getEmail(), "Order " + o.getId() + " created");
        return o;
    }
}
```

**Your Task**
Review the controller's responsibilities and structure.

---

### Exercise 2 — Controller with shared state

**Scenario**
`UploadController` accepts file uploads and tracks the last upload for a status endpoint.

**Code**
```java
@RestController
public class UploadController {
    private UploadResult lastResult; // instance field on a singleton controller

    @PostMapping("/upload")
    public ResponseEntity<Void> upload(@RequestBody byte[] data) {
        lastResult = storageService.store(data);
        return ResponseEntity.ok().build();
    }

    @GetMapping("/upload/status")
    public UploadResult status() {
        return lastResult;
    }
}
```

**Your Task**
Review the controller design under concurrent requests.

---

## Topic 10 — HTTP methods

### Exercise 1 — PUT that drops fields

**Scenario**
`ProfileController.update` lets a user change part of their profile. The mobile client sends only the fields the user edited.

**Code**
```java
@PutMapping("/profile/{id}")
public Profile update(@PathVariable Long id, @RequestBody Profile incoming) {
    Profile existing = profileService.find(id);
    existing.setName(incoming.getName());
    existing.setBio(incoming.getBio());
    existing.setPhone(incoming.getPhone());
    return profileService.save(existing);
}
```

**Your Task**
Review the HTTP method choice and the field-update semantics.

---

### Exercise 2 — GET with a body

**Scenario**
`SearchController.search` accepts a complex multi-field search criteria object.

**Code**
```java
@GetMapping("/search")
public List<Result> search(@RequestBody SearchCriteria criteria) {
    return searchService.query(criteria);
}
```

**Your Task**
Review the method/semantics choice.

---

## Topic 11 — HTTP status codes

### Exercise 1 — 200 for everything

**Scenario**
`AccountController.get` returns account details.

**Code**
```java
@GetMapping("/accounts/{id}")
public ResponseEntity<?> get(@PathVariable Long id) {
    Optional<Account> account = accountService.find(id);
    if (account.isEmpty()) {
        return ResponseEntity.ok(Map.of("error", "Account not found"));
    }
    return ResponseEntity.ok(account.get());
}
```

**Your Task**
Review the status-code usage.

---

### Exercise 2 — Wrong codes on create/validate

**Scenario**
`UserController.create` registers a user.

**Code**
```java
@PostMapping("/users")
public ResponseEntity<User> create(@RequestBody UserRequest req) {
    if (!req.getEmail().contains("@")) {
        throw new RuntimeException("Invalid email"); // becomes a 500
    }
    User u = userService.register(req);
    return ResponseEntity.ok(u); // 200
}
```

**Your Task**
Review the status codes returned for the success and validation paths.

---

## Topic 12 — Request validation

### Exercise 1 — Annotations that never fire

**Scenario**
`RegistrationController.register` accepts a DTO annotated with Bean Validation constraints.

**Code**
```java
public class RegistrationRequest {
    @NotBlank private String username;
    @Email private String email;
    @Size(min = 8) private String password;
    // getters/setters
}

@PostMapping("/register")
public User register(@RequestBody RegistrationRequest req) {
    return userService.register(req);
}
```

**Your Task**
Review whether the constraints are enforced.

---

### Exercise 2 — Trusting client fields

**Scenario**
`AccountController.create` binds the request body directly to the `Account` entity.

**Code**
```java
@Entity
public class Account {
    @Id @GeneratedValue Long id;
    String email;
    BigDecimal balance;
    String role; // "USER" or "ADMIN"
}

@PostMapping("/accounts")
public Account create(@Valid @RequestBody Account account) {
    return accountRepository.save(account);
}
```

**Your Task**
Review the binding and validation for security implications.

---

## Topic 13 — DTOs

### Exercise 1 — Returning the entity

**Scenario**
`UserController.get` returns a user. `User` is a JPA entity.

**Code**
```java
@Entity
public class User {
    @Id Long id;
    String email;
    String passwordHash;
    String twoFactorSecret;
    @OneToMany(fetch = FetchType.LAZY) List<Order> orders;
}

@GetMapping("/users/{id}")
public User get(@PathVariable Long id) {
    return userRepository.findById(id).orElseThrow();
}
```

**Your Task**
Review returning the entity from the controller.

---

### Exercise 2 — Entity as request body

**Scenario**
`ProductController.update` accepts a product to update. `Product` is a JPA entity with relationships.

**Code**
```java
@PutMapping("/products/{id}")
public Product update(@PathVariable Long id, @RequestBody Product product) {
    product.setId(id);
    return productRepository.save(product); // save == merge for detached entity
}
```

**Your Task**
Review using the entity directly as the request body.

---

## Topic 14 — Error handling

### Exercise 1 — Inconsistent per-controller handling

**Scenario**
Several controllers each wrap calls in try/catch and format errors their own way.

**Code**
```java
@GetMapping("/orders/{id}")
public ResponseEntity<?> get(@PathVariable Long id) {
    try {
        return ResponseEntity.ok(orderService.find(id));
    } catch (Exception e) {
        return ResponseEntity.ok("Error: " + e.getMessage());
    }
}
```

**Your Task**
Review the error-handling approach and its consistency across the API.

---

### Exercise 2 — Global handler leaking internals

**Scenario**
`GlobalExceptionHandler` centralizes error handling.

**Code**
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(Exception.class)
    public ResponseEntity<String> handle(Exception e) {
        return ResponseEntity.status(500)
                .body("Error: " + e.getMessage() + "\n"
                      + Arrays.toString(e.getStackTrace()));
    }
}
```

**Your Task**
Review the global handler for correctness and security.

---

## Topic 15 — Pagination

### Exercise 1 — Client-controlled page size

**Scenario**
`ProductController.list` supports paging via client-supplied `page` and `size`.

**Code**
```java
@GetMapping("/products")
public List<Product> list(@RequestParam int page, @RequestParam int size) {
    return productRepository.findAll(PageRequest.of(page, size)).getContent();
}
```
(Callers can send any `size`.)

**Your Task**
Review the pagination for robustness and safety at scale.

---

### Exercise 2 — Unstable page order

**Scenario**
`OrderController.list` pages through orders. Orders are created constantly.

**Code**
```java
@GetMapping("/orders")
public Page<Order> list(@RequestParam int page) {
    // no explicit sort
    return orderRepository.findAll(PageRequest.of(page, 20));
}
```

**Your Task**
Review whether paging yields a consistent, complete traversal.

---

## Topic 16 — Idempotency

### Exercise 1 — Duplicate submissions

**Scenario**
`PaymentController.pay` creates a payment. A flaky mobile network causes clients to retry, and users sometimes double-tap.

**Code**
```java
@PostMapping("/payments")
public Payment pay(@RequestBody PaymentRequest req) {
    Payment p = new Payment(req.getOrderId(), req.getAmount());
    gateway.charge(req.getCard(), req.getAmount());
    return paymentRepository.save(p);
}
```

**Your Task**
Review the endpoint for idempotency/retry safety.

---

### Exercise 2 — Broken idempotency check

**Scenario**
`OrderController.create` uses an idempotency key to avoid duplicates. Two concurrent requests can carry the same key (double-submit).

**Code**
```java
@PostMapping("/orders")
public Order create(@RequestHeader("Idempotency-Key") String key,
                    @RequestBody OrderRequest req) {
    if (orderRepository.existsByIdempotencyKey(key)) {
        return orderRepository.findByIdempotencyKey(key);
    }
    Order o = toOrder(req);
    o.setIdempotencyKey(key);
    return orderRepository.save(o);
}
```

**Your Task**
Review the idempotency implementation under concurrent duplicate requests.

---

## Topic 17 — Authentication

### Exercise 1 — Open endpoint

**Scenario**
Security config for an internal admin API.

**Code**
```java
@Bean
SecurityFilterChain chain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(auth -> auth
            .requestMatchers("/admin/**").permitAll()   // admin endpoints
            .anyRequest().authenticated())
        .csrf(csrf -> csrf.disable());
    return http.build();
}
```

**Your Task**
Review the authentication configuration.

---

### Exercise 2 — Trusting a header

**Scenario**
`ApiController` identifies the caller from a header set "by the gateway."

**Code**
```java
@GetMapping("/me/orders")
public List<Order> myOrders(@RequestHeader("X-User-Id") Long userId) {
    return orderService.findByUser(userId);
}
```

**Your Task**
Review how the user identity is established.

---

## Topic 18 — Authorization

### Exercise 1 — Any user can read any resource

**Scenario**
`DocumentController.get` returns a document by id. Documents belong to specific users.

**Code**
```java
@GetMapping("/documents/{id}")
public Document get(@PathVariable Long id, Authentication auth) {
    return documentRepository.findById(id).orElseThrow();
}
```

**Your Task**
Review the authorization.

---

### Exercise 2 — Client-supplied role

**Scenario**
`AdminController.deleteUser` restricts deletion to admins.

**Code**
```java
@DeleteMapping("/users/{id}")
public ResponseEntity<Void> delete(@PathVariable Long id,
                                   @RequestParam String role) {
    if ("ADMIN".equals(role)) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
    return ResponseEntity.status(403).build();
}
```

**Your Task**
Review the authorization decision.

---

## Topic 19 — API versioning

### Exercise 1 — Breaking change in place

**Scenario**
`v1` of the orders API returns a `total` field as a number. A change reshapes the response: `total` becomes an object `{amount, currency}`, and `status` codes change from strings to integers — on the same endpoint.

**Code**
```java
// Existing clients depend on: { "id": 1, "total": 42.50, "status": "PAID" }
@GetMapping("/api/orders/{id}")
public OrderResponse get(@PathVariable Long id) {
    Order o = orderService.find(id);
    OrderResponse r = new OrderResponse();
    r.setId(o.getId());
    r.setTotal(new Money(o.getAmount(), o.getCurrency())); // was a plain number
    r.setStatus(o.getStatusCode());                        // was a string
    return r;
}
```

**Your Task**
Review the change for backward compatibility.

---

### Exercise 2 — Adding an optional field

**Scenario**
The same endpoint adds a new optional field `estimatedDelivery` to the response. A reviewer says: *"This is a breaking change — bump to v2."*

**Code**
```java
// Response was: { "id": 1, "total": 42.50, "status": "PAID" }
public class OrderResponse {
    private Long id;
    private BigDecimal total;
    private String status;
    private LocalDate estimatedDelivery; // NEW: null when unknown
    // getters/setters
}
```

**Your Task**
Review whether this change actually breaks existing clients. Evaluate the reviewer's claim.

---

## Topic 20 — @Transactional

### Exercise 1 — Annotation on a private method

**Scenario**
`LedgerService.post` writes two ledger rows that must both succeed or both fail.

**Code**
```java
@Service
public class LedgerService {
    public void record(Entry e) {
        post(e);
    }

    @Transactional
    private void post(Entry e) {
        ledgerRepository.save(e.debitRow());
        ledgerRepository.save(e.creditRow());
    }
}
```

**Your Task**
Review whether the two saves are atomic.

---

### Exercise 2 — No transaction at all

**Scenario**
`TransferService.move` debits one account and credits another.

**Code**
```java
@Service
public class TransferService {
    public void move(Long fromId, Long toId, BigDecimal amt) {
        Account from = accountRepository.findById(fromId).orElseThrow();
        from.setBalance(from.getBalance().subtract(amt));
        accountRepository.save(from);

        Account to = accountRepository.findById(toId).orElseThrow();
        to.setBalance(to.getBalance().add(amt));
        accountRepository.save(to);
    }
}
```

**Your Task**
Review the atomicity of the two updates.

---

## Topic 21 — Transaction boundaries

### Exercise 1 — External call inside the transaction

**Scenario**
`OrderService.checkout` persists an order and calls a payment provider, all under one transaction. The provider can take up to 30s.

**Code**
```java
@Transactional
public void checkout(Order order) {
    orderRepository.save(order);
    inventoryRepository.decrement(order.getItems());
    PaymentResult r = paymentClient.charge(order); // synchronous HTTP, up to 30s
    order.setPaymentRef(r.getRef());
    orderRepository.save(order);
}
```

**Your Task**
Review the transaction boundary and its effect on the connection pool.

---

### Exercise 2 — Split atomic operation

**Scenario**
`WalletService` debits then credits, each in its own transactional method, called in sequence by an untransacted caller.

**Code**
```java
@Service
public class WalletService {
    @Transactional
    public void debit(Long id, BigDecimal amt) {
        Wallet w = walletRepository.findById(id).orElseThrow();
        w.setBalance(w.getBalance().subtract(amt));
    }

    @Transactional
    public void credit(Long id, BigDecimal amt) {
        Wallet w = walletRepository.findById(id).orElseThrow();
        w.setBalance(w.getBalance().add(amt));
    }
}

// caller (not transactional):
walletService.debit(a, amt);
walletService.credit(b, amt);
```

**Your Task**
Review the transaction boundaries for the overall transfer.

---

## Topic 22 — Transaction propagation

### Exercise 1 — Audit rolled back with the main operation

**Scenario**
`OrderService.place` records an audit entry for every attempt — success or failure. The audit must persist even if placing the order fails.

**Code**
```java
@Service
public class OrderService {
    @Transactional
    public void place(Order order) {
        auditService.record("place attempt " + order.getId()); // same transaction
        validate(order); // may throw
        orderRepository.save(order);
    }
}

@Service
class AuditService {
    @Transactional  // default propagation
    public void record(String msg) { auditRepository.save(new Audit(msg)); }
}
```

**Your Task**
Review whether the audit survives a failure in `place`.

---

### Exercise 2 — REQUIRES_NEW in a batch loop

**Scenario**
`ImportService.importAll` processes a large batch; each item is saved in its own new transaction so failures are isolated.

**Code**
```java
@Service
public class ImportService {
    public void importAll(List<Record> records) { // 50,000 records
        for (Record r : records) {
            self.saveOne(r);
        }
    }
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveOne(Record r) {
        recordRepository.save(r);
    }
}
```

**Your Task**
Review the propagation choice at this scale.

---

## Topic 23 — Transaction isolation

### Exercise 1 — Check-then-update under default isolation

**Scenario**
`CouponService.redeem` allows a coupon to be used only once. Under READ_COMMITTED (default), it's used concurrently.

**Code**
```java
@Transactional
public void redeem(String code, Long userId) {
    Coupon c = couponRepository.findByCode(code);
    if (c.isRedeemed()) {
        throw new AlreadyRedeemedException();
    }
    c.setRedeemed(true);
    c.setRedeemedBy(userId);
    couponRepository.save(c);
}
```

**Your Task**
Review whether the "once only" rule holds concurrently under default isolation.

---

### Exercise 2 — SERIALIZABLE everywhere

**Scenario**
To "be safe," every service method is annotated with the highest isolation level. This is a high-throughput API.

**Code**
```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public Order getOrder(Long id) {
    return orderRepository.findById(id).orElseThrow();
}

@Transactional(isolation = Isolation.SERIALIZABLE)
public List<Order> recentOrders(Long userId) {
    return orderRepository.findRecent(userId);
}
```

**Your Task**
Review the blanket use of SERIALIZABLE.

---

## Topic 24 — Rollback behavior

### Exercise 1 — Checked exception, no rollback

**Scenario**
`RegistrationService.register` saves a user, then sends a welcome email; a mail failure should roll back the registration.

**Code**
```java
@Transactional
public void register(User user) throws MailException {
    userRepository.save(user);
    mailService.sendWelcome(user); // throws checked MailException on failure
}
```
(`MailException extends Exception`.)

**Your Task**
Review whether a `MailException` rolls back the saved user.

---

### Exercise 2 — Swallowed exception inside the transaction

**Scenario**
`OrderService.finalizeOrder` updates several rows; if a step fails, the whole thing should roll back.

**Code**
```java
@Transactional
public void finalizeOrder(Long id) {
    Order o = orderRepository.findById(id).orElseThrow();
    o.setStatus(Status.FINAL);
    try {
        shippingRepository.createLabel(o); // may throw
    } catch (Exception e) {
        log.warn("label failed", e); // swallow and continue
    }
    invoiceRepository.create(o);
}
```

**Your Task**
Review the rollback behavior when `createLabel` fails.

---

## Topic 25 — readOnly

### Exercise 1 — Writing under readOnly

**Scenario**
`ProfileService.viewAndTouch` loads a profile (read-optimized) and updates its `lastViewed` timestamp.

**Code**
```java
@Transactional(readOnly = true)
public Profile viewAndTouch(Long id) {
    Profile p = profileRepository.findById(id).orElseThrow();
    p.setLastViewed(Instant.now()); // mutate a managed entity
    return p;
}
```

**Your Task**
Review whether `lastViewed` is persisted.

---

### Exercise 2 — readOnly as a guarantee

**Scenario**
A junior engineer marks a reporting method `readOnly = true` and comments: *"This guarantees nothing in this method can ever modify the database, so it's safe to expose."*

**Code**
```java
@Transactional(readOnly = true)
public Report build(ReportRequest req) {
    List<Row> rows = reportRepository.query(req);
    reportRepository.markAccessed(req.getId()); // a native UPDATE query
    return assemble(rows);
}
```

**Your Task**
Review the `readOnly` usage and evaluate the engineer's claim.

---

## Topic 26 — Transaction timeout

### Exercise 1 — Long batch, no timeout

**Scenario**
`ArchiveService.archiveOldOrders` updates millions of rows in one transaction nightly; occasionally it runs for a very long time holding locks.

**Code**
```java
@Transactional
public void archiveOldOrders() {
    List<Order> old = orderRepository.findOlderThan(cutoff()); // could be millions
    for (Order o : old) {
        o.setArchived(true);
    }
}
```

**Your Task**
Review the transaction duration and locking risk.

---

### Exercise 2 — Timeout that misses the real hang

**Scenario**
`SyncService.sync` sets a transaction timeout, then makes an external HTTP call plus a DB write.

**Code**
```java
@Transactional(timeout = 5) // seconds
public void sync(Long id) {
    Entity e = repository.findById(id).orElseThrow();
    String data = externalHttpClient.fetch(id); // can hang for minutes; no HTTP timeout
    e.setData(data);
}
```

**Your Task**
Review what the transaction timeout actually bounds.

---

## Topic 27 — Self-invocation

### Exercise 1 — Internal call to a transactional method

**Scenario**
`BillingService.runBilling` iterates customers and bills each; each billing should be its own transaction.

**Code**
```java
@Service
public class BillingService {
    public void runBilling(List<Customer> customers) {
        for (Customer c : customers) {
            billCustomer(c); // internal call
        }
    }

    @Transactional
    public void billCustomer(Customer c) {
        charge(c);
        invoiceRepository.save(new Invoice(c));
    }
}
```

**Your Task**
Review whether `billCustomer` runs transactionally.

---

### Exercise 2 — Self-invoked @Async

**Scenario**
`ReportService.generateReport` kicks off an async export via an internal call.

**Code**
```java
@Service
public class ReportService {
    public void generateReport(Request r) {
        prepare(r);
        exportAsync(r); // expected to run on a separate thread
    }

    @Async
    public void exportAsync(Request r) {
        heavyExport(r);
    }
}
```

**Your Task**
Review whether `exportAsync` actually runs asynchronously.

---

## Topic 28 — Spring proxy mechanism

### Exercise 1 — @Transactional on a final method

**Scenario**
`PricingService` has a transactional method marked `final`.

**Code**
```java
@Service
public class PricingService {
    @Transactional
    public final void recalculateAll() {
        // updates many rows
        priceRepository.bulkRecalculate();
    }
}
```

**Your Task**
Review whether the transactional behavior is applied.

---

### Exercise 2 — Transactional call from @PostConstruct

**Scenario**
`ReferenceDataLoader` loads reference data into the DB at startup within a transaction.

**Code**
```java
@Component
public class ReferenceDataLoader {

    @PostConstruct
    public void init() {
        loadDefaults(); // internal call during initialization
    }

    @Transactional
    public void loadDefaults() {
        referenceRepository.saveAll(defaultRows());
    }
}
```

**Your Task**
Review whether `loadDefaults` runs inside a transaction.

---

## Topic 29 — Nested transactions

### Exercise 1 — Inner failure marks the whole tx rollback-only

**Scenario**
`OrderService.place` saves an order and tries to add a recommendation; a recommendation failure should NOT fail the order.

**Code**
```java
@Service
public class OrderService {
    @Transactional
    public void place(Order o) {
        orderRepository.save(o);
        try {
            recommendationService.add(o); // @Transactional (REQUIRED)
        } catch (Exception e) {
            log.warn("recommendation failed, continuing", e);
        }
        orderRepository.save(o.markComplete());
    }
}

@Service
class RecommendationService {
    @Transactional // REQUIRED — joins the caller's transaction
    public void add(Order o) {
        recRepository.save(buildRec(o)); // may throw
    }
}
```

**Your Task**
Review what happens to `place`'s transaction when `add` throws and is caught.

---

### Exercise 2 — NESTED expected to commit independently

**Scenario**
`JobService.run` runs several sub-steps; each sub-step uses `NESTED` propagation, and the author expects each successful sub-step to be permanently committed even if a later step fails.

**Code**
```java
@Transactional
public void run(Job job) {
    for (Step step : job.getSteps()) {
        stepService.execute(step); // @Transactional(propagation = NESTED)
    }
    // if step 5 throws here, author expects steps 1-4 to remain committed
}
```

**Your Task**
Review whether completed `NESTED` sub-steps survive a later failure in the outer transaction.

---

## Topic 30 — Optimistic locking

### Exercise 1 — No version field

**Scenario**
`ProductService.adjustStock` reads a product, changes stock, and saves. Multiple warehouse clients adjust the same product concurrently.

**Code**
```java
@Entity
public class Product {
    @Id Long id;
    int stock;
    // no @Version
}

@Transactional
public void adjustStock(Long id, int delta) {
    Product p = productRepository.findById(id).orElseThrow();
    p.setStock(p.getStock() + delta);
    productRepository.save(p);
}
```

**Your Task**
Review the concurrency safety of the stock update.

---

### Exercise 2 — Version present, conflict unhandled

**Scenario**
`DocumentService.update` uses an entity with `@Version`. Two users edit the same document.

**Code**
```java
@Entity
public class Document {
    @Id Long id;
    @Version Long version;
    String content;
}

@Transactional
public Document update(Long id, String content) {
    Document d = documentRepository.findById(id).orElseThrow();
    d.setContent(content);
    return documentRepository.save(d); // may throw OptimisticLockException on conflict
}
```

**Your Task**
Review how a concurrent-edit conflict is handled from the user's perspective.

---

## Topic 31 — Pessimistic locking

### Exercise 1 — Lock held across an external call

**Scenario**
`InventoryService.reserve` pessimistically locks a stock row, then calls an external allocation service before committing.

**Code**
```java
@Transactional
public void reserve(Long productId, int qty) {
    Stock s = stockRepository.findByIdForUpdate(productId); // @Lock(PESSIMISTIC_WRITE)
    externalAllocator.allocate(productId, qty);             // HTTP, up to 20s
    s.setReserved(s.getReserved() + qty);
}
```

**Your Task**
Review the pessimistic lock's scope and contention impact.

---

### Exercise 2 — FOR UPDATE outside a transaction

**Scenario**
`SeatService.hold` selects a seat "for update" to lock it, then marks it held.

**Code**
```java
@Repository
public interface SeatRepository extends JpaRepository<Seat, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select s from Seat s where s.id = :id")
    Seat findByIdForUpdate(@Param("id") Long id);
}

@Service
public class SeatService {
    // no @Transactional
    public void hold(Long seatId, String userId) {
        Seat s = seatRepository.findByIdForUpdate(seatId);
        s.setHeldBy(userId);
        seatRepository.save(s);
    }
}
```

**Your Task**
Review whether the pessimistic lock actually protects the seat.

---

## Topic 32 — Lost updates

### Exercise 1 — Read-modify-write across two requests

**Scenario**
An edit screen does `GET /articles/{id}` to load, then `PUT /articles/{id}` to save. Two editors open the same article and save minutes apart.

**Code**
```java
@PutMapping("/articles/{id}")
public Article update(@PathVariable Long id, @RequestBody ArticleDto dto) {
    Article a = articleRepository.findById(id).orElseThrow();
    a.setTitle(dto.getTitle());
    a.setBody(dto.getBody());
    return articleRepository.save(a);
}
```

**Your Task**
Review the concurrent-edit behavior across the two requests.

---

### Exercise 2 — Counter increment by load-modify-save

**Scenario**
`ArticleService.incrementViews` bumps a view counter on every page load. Popular articles get thousands of concurrent views.

**Code**
```java
@Transactional
public void incrementViews(Long id) {
    Article a = articleRepository.findById(id).orElseThrow();
    a.setViews(a.getViews() + 1);
    articleRepository.save(a);
}
```

**Your Task**
Review the counter update under high concurrency.

---

---

# ANSWER KEY

## Topic 1 · Exercise 1 — Building collaborators by hand
**Findings**
- 🟠 **DESIGN CONCERN — Defeats IoC.** `new SmtpEmailClient(...)` and `new JpaOrderRepositoryImpl()` hard-wire concrete collaborators, so the container can't inject configured, proxied, or test beans. The repository especially will lack Spring Data/transaction proxying and a managed `EntityManager` — it likely won't even work as a real repository. Untestable (no mocks), config hard-coded.

**Fix:** inject via constructor: `OrderService(EmailClient emailClient, OrderRepository repository)`; let Spring provide the beans.
**What a strong senior would say:** *"This bypasses the container entirely — newing the repository and mail client means no injection, no proxying, hard-coded config, and no way to substitute test doubles. The repository in particular needs Spring to wire it. I'd take both as constructor dependencies."*
**Interviewer is testing:** understanding the whole point of IoC/the container.
**Mention first:** hand-wired dependencies defeat DI.

## Topic 1 · Exercise 2 — Service locator
**Findings**
- 🟡 **DESIGN CONCERN — Service-locator anti-pattern.** Pulling beans from `ApplicationContext` at runtime hides dependencies (not visible in the constructor), couples the class to Spring's API, and is fragile (bean-name string built from `p.getMethod()` → `NoSuchBeanDefinitionException` on any mismatch). Harder to test.
- 🟢 It does work; it's just the wrong pattern for a fixed, known set of gateways.

**Fix:** inject a `Map<String, PaymentGateway>` (Spring populates it by bean name) or a `List<PaymentGateway>` + a strategy that matches on a `supports(method)` method — dependencies explicit, no context lookup.
**What a strong senior would say:** *"This is service-locator — it hides the dependency and couples us to the ApplicationContext, and the string-built bean name fails at runtime for any unknown method. Since the gateways are a known set, I'd inject a `Map<String, PaymentGateway>` keyed by method, or a strategy list, so the wiring is explicit and testable."*
**Interviewer is testing:** recognizing the service-locator anti-pattern and the map/strategy injection idiom.
**Mention first:** hidden dependency via context lookup.

## Topic 2 · Exercise 1 — Ambiguous bean
**Findings**
- 🟠 **DEFINITE BUG — Ambiguous dependency.** Two beans implement `NotificationSender` and the constructor asks for one by type → `NoUniqueBeanDefinitionException` at startup (unless one is `@Primary` or the parameter name happens to match a bean name). The app won't start.

**Fix:** `@Qualifier("emailSender")` on the parameter, mark one `@Primary`, or inject `Map<String, NotificationSender>`/`List<...>` if all are needed and choose explicitly.
**What a strong senior would say:** *"Two candidates of the same type with no qualifier means Spring can't decide — startup fails with NoUniqueBeanDefinitionException. I'd add a `@Qualifier`, or `@Primary` if there's a clear default, or inject all of them as a map/list and select intentionally."*
**Interviewer is testing:** ambiguous-injection resolution.
**Mention first:** startup failure from ambiguity.

## Topic 2 · Exercise 2 — Injecting into a static field
**Findings**
- 🔴 **DEFINITE BUG — `@Autowired` on a `static` field doesn't work.** Spring injects into instance fields, not static ones, so `repository` stays `null` → `record()` NPEs. (Static + injection is fundamentally at odds; utilities shouldn't hold container-managed state.)

**Fix:** make it a normal injected instance bean (constructor injection) and call it as a bean, not statically; or use a setter that stores into a static field from an instance `@PostConstruct` (a known workaround, but the clean fix is to drop the static utility pattern).
**What a strong senior would say:** *"Spring can't inject into a static field, so this stays null and NPEs the first time it's used. Static utilities and DI don't mix. I'd make this an injectable bean with the repository as a constructor dependency and call it as a collaborator rather than a static helper."*
**Interviewer is testing:** knowing static-field injection is unsupported.
**Mention first:** static injection → null → NPE.

## Topic 3 · Exercise 1 — Two constructors
**Findings**
- 🟠 **POTENTIAL BUG — Ambiguous constructor selection.** With two constructors and no `@Autowired` marking which to use, Spring doesn't know which to call. Modern Spring auto-selects a *single* constructor, but with two it needs `@Autowired` on the intended one (or it may fail / not inject the cache). Result: unpredictable wiring (cache likely null).

**Fix:** annotate the intended constructor with `@Autowired` (the two-arg one), or make the cache an `Optional`/`@Nullable` single constructor, or provide the cache as a real bean so one constructor suffices.
**What a strong senior would say:** *"With two constructors Spring can't auto-pick one — I'd put `@Autowired` on the constructor I actually want it to use, the two-arg one, so the cache gets injected. As written the wiring is ambiguous and the cache may end up null."*
**Interviewer is testing:** multiple-constructor injection rules.
**Mention first:** ambiguous constructor.

## Topic 3 · Exercise 2 — Constructor injection with final fields
**Findings**
- ⚪ **NOT A BUG — This is best practice.** Constructor injection with `final` fields is exactly what Spring recommends: dependencies are explicit and immutable, the bean can't exist half-wired, and it's trivially unit-testable with `new InvoiceService(mockRepo, mockCalc)`. The reviewer's suggestion to switch to field injection is **wrong** — field injection is the inferior pattern (hidden deps, non-final, needs reflection to test).

**What a strong senior would say:** *"There's nothing to fix here — constructor injection with final fields is the recommended approach, and I'd push back on the reviewer: field injection would make dependencies implicit, the fields non-final, and testing harder. The 'boilerplate' is worth it, and Lombok's `@RequiredArgsConstructor` removes it if that's the concern."*
**Interviewer is testing:** defending correct code against a bad suggestion (false-positive resistance + articulating *why* constructor injection wins).
**Mention first:** state it's correct and rebut the field-injection suggestion.

## Topic 4 · Exercise 1 — Dependency used during construction
**Findings**
- 🔴 **DEFINITE BUG — Field injected after construction.** With field injection, `productRepository` is set **after** the constructor runs, so it's `null` inside the constructor → NPE at bean creation. The cache-warming logic runs too early.

**Fix:** move the warming into `@PostConstruct` (runs after injection) or, better, use constructor injection so the dependency is available and warm in the constructor / a `@PostConstruct`:
```java
public CacheWarmer(ProductRepository repo) { this.repo = repo; }
@PostConstruct void warm() { repo.findAll().forEach(p -> cache.put(p.getId(), p)); }
```
**What a strong senior would say:** *"Field injection happens after the constructor, so `productRepository` is null when the constructor tries to use it — NPE on startup. I'd either move the warm-up to a `@PostConstruct`, or switch to constructor injection so the dependency is present when I use it. This is a strong argument against field injection generally."*
**Interviewer is testing:** injection timing (constructor vs field vs @PostConstruct).
**Mention first:** null dependency in the constructor.

## Topic 4 · Exercise 2 — Field injection and testability
**Findings**
- 🟠 **DESIGN CONCERN — Untestable without Spring/reflection.** `new ShippingService()` leaves `rateProvider` and `addressValidator` null (nothing wires them), so the test NPEs. Field injection forces tests to use Spring context or reflection to set privates. There's no constructor to pass mocks.

**Fix:** constructor injection with `final` fields; the test becomes `new ShippingService(mockRate, mockValidator)`.
**What a strong senior would say:** *"Field injection is why this test fails — `new` doesn't populate the injected fields, so they're null and it NPEs, and there's no constructor to pass mocks. Constructor injection fixes both the test and the general 'hidden dependencies' problem. That testability gap is the practical case against field injection."*
**Interviewer is testing:** the testability cost of field injection.
**Mention first:** can't inject mocks → NPE in test.

## Topic 5 · Exercise 1 — Work in the constructor
**Findings**
- 🟠 **DEFINITE BUG — Side-effecting I/O in the constructor.** Opening sockets to a downstream during construction runs at bean-creation time, before the context is fully up; if the downstream is unavailable, bean creation fails and the whole app fails to start (and it's hard to retry/observe). Constructors should just assign dependencies.

**Fix:** move `connectAll()` to `@PostConstruct` (or, better, `ApplicationReadyEvent`) so it runs after the context is ready and can be made resilient (retry, health-gated, lazy).
**What a strong senior would say:** *"Opening downstream sockets in the constructor couples app startup to that dependency being up — if it's down, bean creation throws and the app won't boot, with no clean retry. I'd move connection setup to `@PostConstruct` or an ApplicationReadyEvent listener, where I can add retries and health checks and not block context creation."*
**Interviewer is testing:** constructor vs lifecycle-callback for initialization side-effects.
**Mention first:** I/O in constructor couples startup to downstream availability.

## Topic 5 · Exercise 2 — No cleanup
**Findings**
- 🟠 **DEFINITE BUG — No `@PreDestroy` for the executor.** The scheduler is never shut down on context close, so on shutdown/redeploy its non-daemon threads keep running (delaying/preventing JVM exit) and in-flight tasks aren't drained — a lifecycle/resource leak.

**Fix:** add `@PreDestroy` that `shutdown()`s and `awaitTermination()`s the scheduler (escalating to `shutdownNow`), or use a Spring `ThreadPoolTaskScheduler` bean whose lifecycle Spring manages.
**What a strong senior would say:** *"The executor has a `@PostConstruct` to start but no `@PreDestroy` to stop, so it leaks threads on shutdown and won't drain in-flight work. I'd add a `@PreDestroy` doing a graceful shutdown, or use a Spring-managed `ThreadPoolTaskScheduler` so the container handles the lifecycle."*
**Interviewer is testing:** paired lifecycle callbacks; resource cleanup.
**Mention first:** missing @PreDestroy.

## Topic 6 · Exercise 1 — Stateful singleton
**Findings**
- 🔴 **DEFINITE BUG — Request state in a singleton.** `@Service` is a singleton shared across all request threads; storing `currentUser` in an instance field means concurrent requests overwrite each other's user → data bleed / wrong-user actions (a serious security/correctness bug).

**Fix:** don't hold request state in a singleton. Use `SecurityContextHolder` (thread-local, request-scoped), a `@RequestScope` bean, or pass the user through method parameters.
**What a strong senior would say:** *"A `@Service` is a singleton, so this `currentUser` field is shared by every concurrent request — user A can see or act as user B. That's both a correctness and a security bug. Request-scoped state belongs in a request-scoped holder or the SecurityContext, or should be passed explicitly, never stored on a singleton."*
**Interviewer is testing:** singleton scope + shared mutable state across requests.
**Mention first:** cross-request state bleed.

## Topic 6 · Exercise 2 — Prototype into singleton
**Findings**
- 🟠 **DEFINITE BUG — Prototype injected once.** A prototype bean injected into a singleton is created **once** (at the singleton's construction) and reused for every call, so `context` is effectively a shared singleton — `reset()/apply()` on shared mutable state races across concurrent orders. The "fresh per call" expectation fails.

**Fix:** get a fresh instance per call — inject an `ObjectProvider<PricingContext>` (`provider.getObject()` per call), or use a scoped proxy (`@Scope(value="prototype", proxyMode=TARGET_CLASS)`), or a factory/`Provider`. Best: make `PricingContext` a local, not a bean, if it's just per-call state.
**What a strong senior would say:** *"Injecting a prototype into a singleton pins one instance for the singleton's lifetime, so every order shares the same context and races on its mutable state. I'd inject an `ObjectProvider` and call `getObject()` per invocation, or use a scoped proxy — or honestly just `new` the per-call state locally, since it's not really a shared bean."*
**Interviewer is testing:** scoped-bean injection semantics (the classic prototype-in-singleton trap).
**Mention first:** prototype created once, shared.

## Topic 7 · Exercise 1 — Mutual constructor dependency
**Findings**
- 🟠 **DEFINITE BUG — Startup failure.** Two constructor-injected beans depending on each other form an unresolvable cycle → `BeanCurrentlyInCreationException` at startup (Spring can't construct either first). Constructor injection surfaces the cycle immediately (a feature).
- 🟡 The cycle usually signals a design problem (mixed responsibilities).

**Fix:** break the cycle by design — extract the shared behavior into a third collaborator both depend on, use events, or invert one dependency. `@Lazy` is a band-aid (see Ex2), not a real fix.
**What a strong senior would say:** *"This is a constructor-injection cycle, so the context fails to start with BeanCurrentlyInCreationException — which is actually good, because it forces us to confront a design smell. Rather than paper over it, I'd extract the shared logic into a third component or use an event so the two-way dependency disappears."*
**Interviewer is testing:** recognizing circular dependencies and preferring redesign.
**Mention first:** startup failure + design smell.

## Topic 7 · Exercise 2 — "Fixed" with @Lazy
**Findings**
- 🟡 **DESIGN CONCERN — `@Lazy` masks, doesn't fix.** `@Lazy` lets the app start by injecting a proxy that resolves `InventoryService` on first use, but the underlying cyclic design remains — it's fragile (init-order surprises, harder reasoning) and the misleading comment "Fixed the circular dependency" oversells it. The real issue (two beans that each need the other) is untouched.

**Fix:** address the design as in Ex1 (extract a third bean / events / invert). Use `@Lazy` only as a deliberate, documented stopgap.
**What a strong senior would say:** *"`@Lazy` gets the context to start by injecting a proxy, but it hasn't fixed anything — the circular design is still there, just deferred, and the comment claiming it's fixed is misleading. I'd treat this as a temporary unblock and do the real refactor: pull the shared responsibility out so neither service depends on the other."*
**Interviewer is testing:** understanding `@Lazy` as a workaround vs a fix; skepticism of the comment.
**Mention first:** @Lazy hides the design smell.

## Topic 8 · Exercise 1 — Missing property
**Findings**
- 🟠 **DEFINITE BUG — Unresolved placeholder.** `@Value("${feature.max-batch-size}")` with no property defined (and no default) fails placeholder resolution → `IllegalArgumentException` at startup in the `prod` profile (or injects a literal if misconfigured). No safety net.

**Fix:** supply a default (`@Value("${feature.max-batch-size:100}")`), or define the property per profile, or use `@ConfigurationProperties` with validation (`@Validated` + `@Min`) so misconfig is caught explicitly with a clear message.
**What a strong senior would say:** *"With no property and no default, this fails to resolve the placeholder and the app won't start in prod. I'd add a sensible default in the annotation, or move config to a validated `@ConfigurationProperties` class so missing/invalid values fail fast with a clear message rather than a cryptic placeholder error."*
**Interviewer is testing:** `@Value` defaults / config validation.
**Mention first:** unresolved placeholder → startup failure.

## Topic 8 · Exercise 2 — Inter-bean method call
**Findings**
- 🟠 **DEFINITE BUG — `@Bean` method call outside `@Configuration`.** The class is `@Component`, not `@Configuration`, so it's **not** CGLIB-enhanced to intercept inter-bean calls. `props()` called directly inside `dataSource()` runs the plain method → a **new** `DataSourceProps` instance, separate from the container-managed `props()` bean. Any other consumer gets a different instance; the singleton guarantee is broken.

**Fix:** annotate the class `@Configuration` (default `proxyBeanMethods=true`), so `props()` returns the shared singleton; or inject `DataSourceProps` as a method parameter of `dataSource(DataSourceProps props)` (works regardless of proxying and is preferred).
**What a strong senior would say:** *"Because this is `@Component` and not `@Configuration`, Spring doesn't proxy the `@Bean` methods, so calling `props()` inside `dataSource()` creates a second, non-managed instance instead of returning the singleton. I'd either make it `@Configuration`, or better, take `DataSourceProps` as a method parameter so it's injected — that's robust even with `proxyBeanMethods=false`."*
**Interviewer is testing:** the @Configuration CGLIB proxying vs @Component distinction — a subtle senior config trap.
**Mention first:** duplicate bean instance from the direct call.

## Topic 9 · Exercise 1 — Business logic in the controller
**Findings**
- 🟠 **DESIGN CONCERN — Fat controller.** The controller owns validation, entity construction, total calculation, persistence, and email — all business logic that belongs in a `@Service`. Not reusable, hard to test, no transaction boundary around save+side-effects, and it returns the entity (see Topic 13).
- 🟡 No `@Transactional`; the email send isn't decoupled from the DB write.

**Fix:** thin controller that validates the DTO (`@Valid`) and delegates to `orderService.place(req)`; move calculation/persistence into the service under a transaction; return a DTO; decouple email (after-commit event).
**What a strong senior would say:** *"The controller is doing the service's job — validation, pricing, persistence, and email. I'd push all of that into an `OrderService` behind a transaction, keep the controller to request/response mapping and validation, return a DTO, and fire the email off an after-commit event so a mail hiccup doesn't affect the order."*
**Interviewer is testing:** layering / SRP; controllers as thin adapters.
**Mention first:** business logic in the controller.

## Topic 9 · Exercise 2 — Controller with shared state
**Findings**
- 🔴 **DEFINITE BUG — Mutable state on a singleton controller.** `@RestController` is a singleton; `lastResult` is shared across all requests/threads → concurrent uploads overwrite it (race), and the status endpoint returns *some other user's* last upload — a cross-request data leak and a race condition.

**Fix:** don't store per-request/per-user state on the controller. Persist upload results (keyed by user/upload id) in a store and look them up; or return a result id the client polls. Controllers must be stateless.
**What a strong senior would say:** *"Controllers are singletons, so this `lastResult` field is shared by every request — uploads race to overwrite it and the status endpoint leaks whoever wrote last, across users. I'd make the controller stateless: persist each result keyed by an upload id and have status look it up by that id."*
**Interviewer is testing:** stateless-controller principle; shared-state race.
**Mention first:** shared mutable state on a singleton.

## Topic 10 · Exercise 1 — PUT that drops fields
**Findings**
- 🟠 **POTENTIAL BUG — PUT semantics + partial payload.** PUT means "replace the resource." The client sends only edited fields, so unspecified fields on `incoming` are null; while this code only copies name/bio/phone, if any of those are absent from the payload they get overwritten with null → data loss. And the HTTP verb is wrong for a partial update.
- 🟡 The right verb for partial update is `PATCH`; PUT should carry the full representation.

**Fix:** use `PATCH` and apply only present fields (e.g. null-means-unchanged, or JSON Merge Patch), or require PUT to send the complete resource. Validate which semantics the client contract expects.
**What a strong senior would say:** *"There's a semantic mismatch: the client sends a partial payload, but PUT is a full replace, so any omitted field can null out existing data. For partial edits I'd use PATCH and only apply the fields actually present — ideally JSON Merge Patch so 'absent' and 'explicit null' are distinguishable. I'd confirm the client contract before deciding."*
**Interviewer is testing:** PUT vs PATCH semantics and partial-update data loss.
**Good clarifying question:** *"Does the client always send the full resource, or only changed fields?"* — decides PUT vs PATCH.
**Mention first:** partial payload under PUT → data loss.

## Topic 10 · Exercise 2 — GET with a body
**Findings**
- 🟠 **DESIGN CONCERN — GET with a request body.** Many clients, proxies, caches, and the HTTP spec don't reliably support/forward a body on GET — it can be silently dropped, so the search may run with empty criteria. GET bodies are discouraged.

**Fix:** either use query parameters for the criteria (keeps GET's cacheability/idempotency for simple searches), or make it `POST /search` when the criteria are complex (a common, accepted pattern for search). Don't rely on a GET body.
**What a strong senior would say:** *"A body on GET is unreliable — proxies and clients may drop it, so the search could run with no criteria. For simple filters I'd use query params; for a rich criteria object, `POST /search` is the accepted pattern even though it's a read, because it carries a body dependably. I wouldn't ship GET-with-body."*
**Interviewer is testing:** knowing GET bodies are problematic and the POST-search convention.
**Mention first:** unreliable GET body.

## Topic 11 · Exercise 1 — 200 for everything
**Findings**
- 🟠 **DEFINITE BUG — Wrong status for not-found.** Returning `200 OK` with an `{error: ...}` body for a missing account means clients can't distinguish success from failure by status code — every consumer must parse the body. Not found should be `404`.

**Fix:** `return account.map(ResponseEntity::ok).orElseThrow(() -> new NotFoundException(id));` mapped to 404 via `@RestControllerAdvice`, ideally with an RFC 7807 `ProblemDetail` body.
**What a strong senior would say:** *"Returning 200 with an error body defeats HTTP status semantics — clients, caches, and monitoring all key off the status, and now they can't tell a miss from a hit. I'd return 404 for not-found, with a consistent problem-detail body, and let a global handler map the exception."*
**Interviewer is testing:** meaningful status codes; 200-for-errors anti-pattern.
**Mention first:** 200 masking a 404.

## Topic 11 · Exercise 2 — Wrong codes on create/validate
**Findings**
- 🟠 **DEFINITE BUG — 500 for a client error.** A bad email is a client mistake (`400 Bad Request`), but throwing a bare `RuntimeException` yields `500`, blaming the server and polluting error metrics/alerts. And a successful create should return `201 Created` (with a `Location`), not `200`.

**Fix:** validate with Bean Validation (`@Valid`) so invalid input → 400 automatically, or throw a `400`-mapped exception; return `ResponseEntity.created(location).body(u)` for the created user.
**What a strong senior would say:** *"Two status issues: an invalid email is a 400, but a raw RuntimeException makes it a 500 — that misattributes the failure and fires server alerts for user error. And a creation should be 201 with a Location header, not 200. I'd use validation for the 400 and `created(...)` for the success."*
**Interviewer is testing:** 400-vs-500 and 201-for-create.
**Mention first:** 500 on client validation error.

## Topic 12 · Exercise 1 — Annotations that never fire
**Findings**
- 🔴 **DEFINITE BUG — Missing `@Valid`.** Without `@Valid` on the `@RequestBody`, the Bean Validation constraints (`@NotBlank`, `@Email`, `@Size`) are **not** evaluated — invalid registrations (blank username, bad email, short password) are accepted. The annotations are decorative until triggered.

**Fix:** `register(@Valid @RequestBody RegistrationRequest req)`; handle the resulting `MethodArgumentNotValidException` in a global handler to return a 400 with field errors.
**What a strong senior would say:** *"The constraints don't run because there's no `@Valid` on the request body — so all this validation is silently inert and bad data gets through. I'd add `@Valid` and map the validation exception to a 400 with per-field messages in the advice."*
**Interviewer is testing:** knowing `@Valid` is what triggers Bean Validation.
**Mention first:** missing @Valid → no validation.

## Topic 12 · Exercise 2 — Trusting client fields
**Findings**
- 🔴 **DEFINITE BUG — Mass assignment / privilege escalation.** Binding the request directly to the `Account` entity lets a caller set `role: "ADMIN"` and `balance: 1000000` in the JSON — the client controls security- and money-critical fields (over-posting). `@Valid` doesn't help; it only checks constraints, not which fields are allowed.

**Fix:** accept a narrow DTO with only client-settable fields (email); set `role`/`balance` server-side; never bind untrusted input straight to an entity.
**What a strong senior would say:** *"This is mass assignment — the client can send `role: ADMIN` and any balance, because we bind the raw body to the entity. That's a privilege-escalation and financial-tampering hole. I'd use a DTO exposing only the fields a user may set, and set role and balance on the server. `@Valid` doesn't protect against over-posting."*
**Interviewer is testing:** over-posting/mass-assignment awareness; DTO boundary as a security control.
**Mention first:** mass assignment of role/balance.

## Topic 13 · Exercise 1 — Returning the entity
**Findings**
- 🔴 **DEFINITE BUG — Sensitive-field exposure.** Serializing the `User` entity leaks `passwordHash` and `twoFactorSecret` to the client — a serious data exposure.
- 🟠 **Lazy loading.** The lazy `orders` collection is accessed during serialization outside the transaction → `LazyInitializationException` (or, if configured open-in-view, hidden N+1). Plus schema coupling.

**Fix:** map to a `UserDto` exposing only safe fields (id, email); never serialize entities. Assemble the DTO inside the transaction.
**What a strong senior would say:** *"Returning the entity leaks the password hash and 2FA secret straight to the client — that alone I'd block on. It also drags the lazy `orders` into serialization, risking a LazyInitializationException or hidden N+1. I'd return a DTO with just the safe fields, built inside the transaction."*
**Interviewer is testing:** entity-leak (security) + lazy-serialization; DTO discipline.
**Mention first:** leaking secrets (🔴).

## Topic 13 · Exercise 2 — Entity as request body
**Findings**
- 🟠 **DEFINITE BUG — Detached-entity merge + over-posting.** Binding the body to `Product` and calling `save` performs a **merge** of a client-controlled detached entity: fields the client omits become null and can blank out columns; relationships/ids can be tampered with; and any server-managed fields (audit, version, owner) are overwritten by client data. Same mass-assignment risk as Topic 12 Ex2, plus JPA merge semantics.

**Fix:** accept a `ProductUpdateDto` with only editable fields; load the managed entity, copy the allowed fields, save. Don't merge a client-built entity.
**What a strong senior would say:** *"Saving a client-supplied detached entity is a merge, so omitted fields get nulled and the client can overwrite server-managed fields and relationships — over-posting plus dangerous merge semantics. I'd take an update DTO of just the editable fields, load the managed product, apply those fields, and save."*
**Interviewer is testing:** why entities shouldn't be request bodies (merge + over-posting).
**Mention first:** detached merge nulls fields / over-posting.

## Topic 14 · Exercise 1 — Inconsistent per-controller handling
**Findings**
- 🟠 **DESIGN CONCERN — Scattered, inconsistent error handling + 200-for-errors.** Each controller catches `Exception` and formats differently, returning `200` with a string error — clients get inconsistent shapes and can't detect failures by status. Duplicated boilerplate; also swallows/obscures real errors.

**Fix:** remove per-controller try/catch; centralize in `@RestControllerAdvice` with typed `@ExceptionHandler`s returning consistent `ProblemDetail` responses and correct statuses. Let exceptions propagate.
**What a strong senior would say:** *"Every controller reinventing error handling gives inconsistent payloads and returns 200 for failures, so clients can't rely on either the shape or the status. I'd delete the local try/catches and centralize handling in a `@RestControllerAdvice` that maps exception types to proper statuses and a uniform problem-detail body."*
**Interviewer is testing:** centralized error handling with @RestControllerAdvice.
**Mention first:** inconsistent handling + 200-for-errors.

## Topic 14 · Exercise 2 — Global handler leaking internals
**Findings**
- 🔴 **DEFINITE BUG — Information disclosure.** Returning `e.getMessage()` and the full **stack trace** to the client leaks internals (class names, file paths, SQL, library versions) — a security and professionalism problem, and it maps everything to a blanket 500 regardless of cause.
- 🟠 Catch-all `Exception` also turns client errors (validation, not-found) into 500s.

**Fix:** log the full detail server-side (with a correlation id); return a sanitized, generic message + the correlation id to the client; add specific handlers for known exception types mapping to appropriate statuses (400/404/409).
**What a strong senior would say:** *"Sending the message and stack trace to the client leaks internal structure and possibly SQL — I'd never expose that. I'd log the details server-side with a correlation id, return a generic body plus that id, and add typed handlers so validation and not-found get 400/404 instead of everything collapsing to 500."*
**Interviewer is testing:** not leaking internals; per-type status mapping.
**Mention first:** stack-trace leak to client.

## Topic 15 · Exercise 1 — Client-controlled page size
**Findings**
- 🟠 **DEFINITE BUG — Unbounded page size (DoS/memory).** The client sets `size` with no cap, so `size=1000000` loads a million rows into memory in one response → OOM/latency, a trivial DoS. Also unvalidated `page`/`size` (negatives) can error.

**Fix:** clamp `size` to a max (e.g. `Math.min(size, 100)`), default sensibly, validate non-negative; Spring's `@PageableDefault(size=20)` + a configured max page size enforces this centrally.
**What a strong senior would say:** *"Letting the client choose an unbounded page size is a DoS vector — one request can pull a million rows and OOM us. I'd cap the size server-side, default it, and validate the inputs; Spring's Pageable with a max-page-size setting does this cleanly."*
**Interviewer is testing:** bounding client-controlled paging.
**Mention first:** unbounded size → DoS.

## Topic 15 · Exercise 2 — Unstable page order
**Findings**
- 🟠 **DEFINITE BUG — Non-deterministic pagination.** With no explicit `ORDER BY`, the database may return rows in an arbitrary/unstable order, and since orders are created constantly, offset paging can **skip or duplicate** rows across pages (page drift) — an incomplete/inconsistent traversal. Offset pagination on a changing dataset is also fragile.

**Fix:** always page with a **stable, total sort** (e.g. `Sort.by("createdAt").and("id")`), so ordering is deterministic; for deep paging on large, changing data prefer keyset/seek pagination (`WHERE (created_at, id) < (:c, :i)`), which avoids drift.
**What a strong senior would say:** *"There's no explicit sort, so the row order isn't guaranteed and, with constant inserts, offset paging drops and repeats rows between pages. I'd always sort by a stable key like `createdAt, id`. For deep pages on a hot table I'd move to keyset pagination, which is both consistent and faster than large offsets."*
**Interviewer is testing:** stable-sort requirement + keyset pagination awareness.
**Mention first:** missing deterministic order → page drift.

## Topic 16 · Exercise 1 — Duplicate submissions
**Findings**
- 🔴 **DEFINITE BUG — Non-idempotent create + charge.** Retries and double-taps create duplicate payments and **charge the card multiple times** (the `gateway.charge` runs each time). No idempotency key, no dedup, and the charge happens before persistence with no transaction linking them.

**Fix:** require an `Idempotency-Key`; on repeat, return the stored result instead of re-charging; pass the key to the gateway so it dedups; persist the payment and charge outcome atomically. A unique constraint on (orderId, idempotencyKey) enforces it at the DB.
**What a strong senior would say:** *"A retry or double-tap here charges the card again and creates a duplicate payment. Payments must be idempotent: I'd take an idempotency key, short-circuit to the stored result on repeats, pass the key to the gateway for its own dedup, and enforce uniqueness at the DB. Charging before persisting with no transaction is a second problem."*
**Interviewer is testing:** idempotency for payment endpoints.
**Mention first:** double-charge on retry (🔴).

## Topic 16 · Exercise 2 — Broken idempotency check
**Findings**
- 🟠 **DEFINITE BUG — Check-then-act race in the idempotency logic.** `existsByIdempotencyKey` then `save` isn't atomic; two concurrent requests with the same key both see "not exists" and both insert → duplicate orders. The idempotency mechanism has a TOCTOU hole.

**Fix:** enforce a **unique constraint** on `idempotency_key` at the DB and rely on it: attempt the insert, and on the unique-violation return the existing row (catch `DataIntegrityViolationException`). That makes first-writer-wins atomic across threads and instances. (A distributed lock on the key is an alternative but heavier.)
**What a strong senior would say:** *"The exists-then-save is a check-then-act race — two concurrent duplicates both pass the check and both insert. I'd put a unique constraint on the idempotency key and let the database arbitrate: insert, and on a unique violation return the already-stored order. That's atomic and works across instances, unlike an in-app check."*
**Interviewer is testing:** recognizing the TOCTOU in idempotency and enforcing it at the DB.
**Mention first:** the concurrent double-insert race.

## Topic 17 · Exercise 1 — Open endpoint
**Findings**
- 🔴 **DEFINITE BUG — Admin endpoints `permitAll`.** `/admin/**` is open to everyone with no authentication — anyone can hit the admin API. The `.anyRequest().authenticated()` protects the rest, but the explicit `permitAll` on admin is a critical misconfiguration.
- 🟡 `csrf.disable()` is fine for a stateless token API but should be a conscious decision (flag it for confirmation, not assume).

**Fix:** `requestMatchers("/admin/**").hasRole("ADMIN")` (authenticated + authorized); keep the rest authenticated.
**What a strong senior would say:** *"This permits all access to `/admin/**` — the admin API is wide open. That's a critical misconfiguration; it should require an ADMIN role. I'd also confirm the CSRF disable is intentional for a stateless API rather than an accidental blanket-off."*
**Interviewer is testing:** reading security config; spotting an open sensitive matcher.
**Mention first:** admin permitAll (🔴).

## Topic 17 · Exercise 2 — Trusting a header
**Findings**
- 🔴 **DEFINITE BUG — Client-controllable identity.** `X-User-Id` is trusted as the caller's identity, but unless the gateway strips/overwrites inbound copies **and** the app is unreachable except through it, a client can send their own `X-User-Id` and impersonate anyone (broken authentication → full account takeover / IDOR).

**Fix:** derive identity from an authenticated principal — a verified JWT/session (`SecurityContextHolder` / `@AuthenticationPrincipal`), not a plain header. If a gateway injects identity, it must be a **signed** token the app verifies, and the network must guarantee requests can't bypass the gateway. Never trust an unauthenticated header as identity.
**What a strong senior would say:** *"Trusting `X-User-Id` means anyone who can reach this service can set it and act as any user — that's an authentication bypass. Identity has to come from something the app verifies: a validated JWT or session principal. If the gateway asserts identity, it must sign it and we must verify the signature, and traffic must be forced through the gateway. As written it's an impersonation hole."*
**Interviewer is testing:** not trusting client-supplied identity; verified principals.
**Mention first:** header impersonation (🔴).

## Topic 18 · Exercise 1 — Any user can read any resource
**Findings**
- 🔴 **DEFINITE BUG — IDOR (missing ownership check).** The endpoint authenticates (`Authentication` present) but never verifies the document **belongs to** the caller — any logged-in user can fetch any document by guessing/enumerating ids. Authentication without authorization.

**Fix:** check ownership: load the doc and verify `doc.getOwnerId().equals(currentUserId)` (throw 403/404 otherwise), or scope the query (`findByIdAndOwner(id, currentUser)`), or use method security (`@PostAuthorize("returnObject.owner == authentication.name")`).
**What a strong senior would say:** *"This is an IDOR — we check that the user is logged in but not that the document is theirs, so anyone can read anyone's documents by id. I'd scope the lookup to the current user (`findByIdAndOwner`) or verify ownership and return 404/403 otherwise. Authentication isn't authorization."*
**Interviewer is testing:** IDOR / object-level authorization.
**Mention first:** missing ownership check (🔴).

## Topic 18 · Exercise 2 — Client-supplied role
**Findings**
- 🔴 **DEFINITE BUG — Authorization from a client parameter.** The `role` comes from a request param the client fully controls, so anyone can pass `role=ADMIN` and delete users. The authorization decision must be based on the **authenticated** principal's roles, not request input.

**Fix:** remove the `role` param; enforce via Spring Security — `@PreAuthorize("hasRole('ADMIN')")` on the method (or config-level matcher), which checks the authenticated user's granted authorities.
**What a strong senior would say:** *"The authorization is based on a request parameter the caller sets, so `role=ADMIN` gives anyone delete rights — a trivial privilege escalation. The decision must use the authenticated principal's authorities: `@PreAuthorize(\"hasRole('ADMIN')\")`, not a client-supplied value."*
**Interviewer is testing:** authorization must come from the security context, not input.
**Mention first:** client-controlled role (🔴).

## Topic 19 · Exercise 1 — Breaking change in place
**Findings**
- 🟠 **DEFINITE BUG — Backward-incompatible change on a live endpoint.** Changing `total` from a number to an object and `status` from string to integer **breaks existing clients** that parse the old shape — deserialization errors or silent misreads in production, with no migration path. Breaking changes need a new version.

**Fix:** introduce `/api/v2/orders/{id}` with the new shape and keep `v1` serving the old contract during a deprecation window (or use content negotiation). Communicate the deprecation timeline. Don't reshape a published response in place.
**What a strong senior would say:** *"Reshaping `total` and retyping `status` on the existing endpoint breaks every current client — this is exactly what versioning is for. I'd ship the new shape under `/v2`, keep `/v1` alive through a deprecation window, and coordinate the client migration. Breaking changes to a live contract are a no-go without a version."*
**Interviewer is testing:** recognizing breaking changes and versioning strategy.
**Mention first:** breaking the published contract.

## Topic 19 · Exercise 2 — Adding an optional field
**Findings**
- ⚪ **NOT A BUG — Additive, backward-compatible change.** Adding a new **optional** field to a JSON response does **not** break well-behaved clients: existing consumers ignore unknown fields (and Jackson clients default to `FAIL_ON_UNKNOWN_PROPERTIES=false` in Spring Boot). No existing field changed type or was removed. The reviewer's "bump to v2" is **over-engineering**; a version bump per additive field would explode the version surface.
- 🟢 Caveat (not a bug): if a specific consumer is known to reject unknown fields (strict schema/codegen), coordinate — but that's the exception, not the rule.

**What a strong senior would say:** *"Adding an optional field is a backward-compatible, additive change — existing clients ignore unknown properties, and nothing existing changed. I'd push back on bumping to v2; versioning per new field would be unmanageable. The only caveat is if we know a consumer strictly rejects unknown fields, in which case we'd coordinate — but by default this is safe."*
**Interviewer is testing:** distinguishing breaking from non-breaking changes; resisting unnecessary versioning (false positive).
**Mention first:** state it's non-breaking and rebut the v2 suggestion.

## Topic 20 · Exercise 1 — Annotation on a private method
**Findings**
- 🔴 **DEFINITE BUG — `@Transactional` on a private method.** Spring's proxy can only apply transactional advice to public methods invoked through the proxy. `post` is `private` **and** self-invoked from `record()`, so the annotation is completely ignored — the two saves run **without a transaction** and aren't atomic. A failure between them leaves a half-written ledger.

**Fix:** make the transactional method `public` and invoke it through the proxy (e.g. move `post` to another bean, or put `@Transactional` on `record` since it's the public entry point).
**What a strong senior would say:** *"`@Transactional` on a private, self-invoked method does nothing — the proxy can't advise private methods and self-calls bypass it anyway, so the debit and credit rows aren't atomic. I'd annotate the public entry point `record`, or extract `post` into a separate bean so the proxy applies."*
**Interviewer is testing:** proxy visibility rules for @Transactional.
**Mention first:** private/self-invoked → no transaction.

## Topic 20 · Exercise 2 — No transaction at all
**Findings**
- 🔴 **DEFINITE BUG — Non-atomic transfer.** With no `@Transactional`, each `save` may commit independently (or flush at different times), so a failure after the debit but before the credit **loses money** — the debit is committed, the credit isn't. Classic atomicity violation.

**Fix:** annotate `move` with `@Transactional` so both updates commit or roll back together. (Also consider optimistic/pessimistic locking for the balances — Topics 30/31.)
**What a strong senior would say:** *"There's no transaction, so a failure between the debit and the credit commits the debit alone and money vanishes. I'd wrap `move` in `@Transactional` so both balance changes are atomic. Separately I'd add concurrency control on the balances so simultaneous transfers don't lose updates."*
**Interviewer is testing:** recognizing a missing transaction boundary on a multi-write operation.
**Mention first:** non-atomic debit/credit.

## Topic 21 · Exercise 1 — External call inside the transaction
**Findings**
- 🔴 **DEFINITE BUG — Long external call inside the transaction.** The synchronous, up-to-30s `paymentClient.charge` runs while the DB transaction (and any row locks from the order/inventory writes) is held → the connection is pinned for the whole HTTP round-trip. Under load this exhausts the connection pool and holds locks, cascading into an outage.

**Fix:** keep the transaction to DB work only; move the payment call **outside** the transaction (charge first then persist, or persist then charge in a follow-up step with an outbox/state machine for consistency); add an HTTP timeout and idempotency. Never do remote I/O inside a DB transaction.
**What a strong senior would say:** *"Holding the transaction open across a 30-second payment call pins a DB connection and locks for the whole HTTP round-trip — under load that drains the pool and blocks other writers. I'd take the payment call out of the transaction, keep the tx to the DB writes, and coordinate the two with an outbox or a state field, plus a timeout and idempotency on the charge."*
**Interviewer is testing:** the I/O-inside-transaction anti-pattern and connection-pool impact.
**Mention first:** external call holding the tx/connection.

## Topic 21 · Exercise 2 — Split atomic operation
**Findings**
- 🔴 **DEFINITE BUG — No overarching transaction.** `debit` and `credit` each commit in their own transaction and the caller isn't transactional, so a failure (or crash) after `debit` commits but before `credit` **loses money** — the two halves aren't atomic. The transfer's atomic unit is split.

**Fix:** wrap the whole transfer in one transaction — a transactional `transfer(from, to, amt)` method that calls both operations so they commit/roll back together (default `REQUIRED` makes the inner methods join it). If the steps must be separate services, coordinate with a saga/outbox, not two independent commits.
**What a strong senior would say:** *"Debit and credit commit independently with no transaction spanning them, so a failure between them leaves money debited but not credited. I'd introduce a transactional `transfer` method that performs both under one transaction — with REQUIRED, the inner methods join it and it's atomic. If they genuinely can't share a transaction, this needs a saga with compensation."*
**Interviewer is testing:** transaction boundary must span the atomic unit; propagation basics.
**Mention first:** split commits → money loss.

## Topic 22 · Exercise 1 — Audit rolled back with the main operation
**Findings**
- 🟠 **DEFINITE BUG (vs requirement) — Audit joins and rolls back with the main tx.** `AuditService.record` uses default `REQUIRED`, so it **joins** `place`'s transaction. When `validate` throws, the whole transaction rolls back — including the audit row. The requirement ("audit must persist even on failure") is violated: failed attempts leave no audit trail.

**Fix:** make the audit write independent with `@Transactional(propagation = Propagation.REQUIRES_NEW)` so it commits in its own transaction regardless of the outer rollback. (Be mindful REQUIRES_NEW uses a second connection — fine for a single audit write.)
**What a strong senior would say:** *"Because the audit uses default REQUIRED, it joins the caller's transaction and rolls back with it — so exactly the failure cases we most want audited leave no record. I'd mark the audit `REQUIRES_NEW` so it commits independently of the order's outcome. I'd just note it consumes a second connection, which is fine for one small write."*
**Interviewer is testing:** REQUIRED vs REQUIRES_NEW for independent side-writes.
**Mention first:** audit rolled back on failure.

## Topic 22 · Exercise 2 — REQUIRES_NEW in a batch loop
**Findings**
- 🟠 **PERFORMANCE/RELIABILITY CONCERN — REQUIRES_NEW per item at scale.** 50,000 iterations each suspending the current transaction and starting a **new** one means 50,000 transaction begin/commits and repeated connection acquisition; if the outer context also holds a transaction/connection, each REQUIRES_NEW needs an **additional** connection concurrently → risk of pool exhaustion and heavy overhead. Per-item isolation is legitimate, but this granularity is costly.

**Fix:** batch — process in chunks (e.g. 500/tx) so a failure isolates a chunk rather than one row, cutting transaction overhead ~1000×; or handle item failures within one transaction using savepoints. Choose isolation granularity by the failure-isolation requirement vs throughput. (Also: `self.saveOne` correctly routes through the proxy — good, since REQUIRES_NEW needs the proxy.)
**What a strong senior would say:** *"Per-item REQUIRES_NEW gives failure isolation but at 50k rows it's 50k transactions and a lot of connection churn — and if the outer method is transactional, each new one needs an extra connection, risking pool exhaustion. I'd batch into chunks so a failure isolates a chunk instead of a row, which keeps most of the isolation benefit at a fraction of the cost. The self-injection to hit the proxy is correct, at least."*
**Interviewer is testing:** cost of REQUIRES_NEW at scale; batching trade-off; noticing the correct proxy self-call.
**Mention first:** per-item new transactions → overhead/pool pressure.

## Topic 23 · Exercise 1 — Check-then-update under default isolation
**Findings**
- 🟠 **DEFINITE BUG — Race under READ_COMMITTED.** Two concurrent `redeem`s both read `isRedeemed() == false` (each in its own snapshot), both set redeemed and save → the coupon is redeemed twice. READ_COMMITTED doesn't prevent this lost-update/write-skew; the atomic check-and-set isn't enforced.

**Fix (best → situational):** a conditional update at the DB — `UPDATE coupon SET redeemed=true, redeemed_by=? WHERE code=? AND redeemed=false` and treat 0 rows as "already redeemed" (atomic, no lock held). Alternatively pessimistic lock (`SELECT ... FOR UPDATE`) on the coupon row, or optimistic locking with `@Version` + retry, or REPEATABLE_READ/SERIALIZABLE with conflict handling. The conditional UPDATE is simplest and correct.
**What a strong senior would say:** *"Under READ_COMMITTED two redemptions can both see the coupon as unused and both succeed — the check-then-set isn't atomic. The cleanest fix is a conditional UPDATE guarded by `redeemed = false`, treating zero rows affected as already-redeemed. Alternatively a `SELECT ... FOR UPDATE` or a version column with retry. Raising isolation to SERIALIZABLE would also work but at a throughput cost and you'd need to handle serialization failures."*
**Interviewer is testing:** isolation levels vs application-level atomicity; conditional update.
**Mention first:** the double-redeem race.

## Topic 23 · Exercise 2 — SERIALIZABLE everywhere
**Findings**
- 🟠 **DESIGN/PERFORMANCE CONCERN — Over-isolation.** Blanket SERIALIZABLE on a high-throughput API (including simple reads like `getOrder`) forces the strictest concurrency control everywhere — heavy locking/serialization-failure retries, sharply reduced throughput, and more deadlocks/serialization errors that the code doesn't handle. Most of these operations don't need it.
- 🟡 Read-only lookups don't need SERIALIZABLE at all; and SERIALIZABLE in Postgres can throw serialization failures that must be caught and retried.

**Fix:** use the default isolation for the vast majority; apply higher isolation (or explicit locking) only to the specific operations with a real anomaly (like the coupon check in Ex1); mark reads `readOnly`; add retry handling wherever SERIALIZABLE is genuinely needed.
**What a strong senior would say:** *"SERIALIZABLE everywhere is a blunt instrument — on a hot API it kills throughput and creates serialization failures nobody's handling, and simple reads don't need it at all. I'd default to READ_COMMITTED, reserve stronger isolation or explicit locking for the specific operations with a real race, and add retry logic only where SERIALIZABLE is warranted."*
**Interviewer is testing:** isolation as a targeted tool, not a blanket safety setting; cost awareness.
**Mention first:** blanket over-isolation cost.

## Topic 24 · Exercise 1 — Checked exception, no rollback
**Findings**
- 🔴 **DEFINITE BUG — No rollback on checked exception.** Spring rolls back by default only for **unchecked** (RuntimeException/Error). `MailException extends Exception` is checked, so when `sendWelcome` throws, the transaction **commits** the saved user anyway — the opposite of the requirement (mail failure should undo the registration).

**Fix:** `@Transactional(rollbackFor = MailException.class)` (or `rollbackFor = Exception.class`), or make the mail failure an unchecked exception. Better still, decouple email from the transaction entirely (after-commit), since email delivery usually shouldn't gate registration — clarify the actual requirement.
**What a strong senior would say:** *"Spring only auto-rolls-back on unchecked exceptions, and MailException is checked, so the user gets saved even when the welcome email fails — contrary to the requirement. I'd add `rollbackFor = MailException.class`. Though I'd also question whether a mail failure should really roll back a registration; usually I'd send the email after commit and not couple them."*
**Interviewer is testing:** the checked-exception rollback default (`rollbackFor`).
**Mention first:** checked exception commits instead of rolling back.

## Topic 24 · Exercise 2 — Swallowed exception inside the transaction
**Findings**
- 🟠 **DEFINITE BUG — Caught exception prevents rollback.** Catching the `createLabel` failure inside the transactional method swallows it, so the transaction **continues and commits** — status set to FINAL and an invoice created even though the label step failed. The "all-or-nothing" requirement is violated; partial, inconsistent state is committed.

**Fix:** don't swallow if the step is essential — let it propagate so the transaction rolls back. If label creation is genuinely optional, make that explicit (and move it out of the atomic unit / after commit). Decide based on whether the label is required for a valid finalized order.
**What a strong senior would say:** *"Catching the exception inside the transaction means Spring never sees a failure, so it commits the FINAL status and the invoice despite the label failing — partial state, contrary to the all-or-nothing intent. If the label is required, I'd let the exception propagate so it rolls back; if it's truly optional, I'd pull it out of the transaction and handle it separately, not bury it in a catch."*
**Interviewer is testing:** that a swallowed exception defeats rollback; atomic-unit reasoning.
**Mention first:** swallowed exception → partial commit.

## Topic 25 · Exercise 1 — Writing under readOnly
**Findings**
- 🟠 **DEFINITE BUG — Mutation silently not persisted.** `readOnly = true` sets the Hibernate flush mode to MANUAL/NEVER, so dirty-checking doesn't flush — mutating `lastViewed` on the managed entity has **no effect** on the database (it's silently discarded at commit). The update quietly does nothing.

**Fix:** if the method writes, it must not be `readOnly` — use `@Transactional` (read-write). If you want the read to be optimized but still record the view, split: a read-only load and a separate small read-write update (or an atomic `UPDATE ... SET last_viewed = now()`), or drop `readOnly`.
**What a strong senior would say:** *"`readOnly = true` disables the automatic flush, so the `lastViewed` change is never written — the update silently vanishes. A method that mutates managed state can't be readOnly. I'd either drop readOnly here, or keep the read optimized and do the timestamp bump as a separate small write, ideally an atomic UPDATE."*
**Interviewer is testing:** understanding readOnly's flush-mode effect (a real silent-data-loss trap).
**Mention first:** writes silently discarded under readOnly.

## Topic 25 · Exercise 2 — readOnly as a guarantee
**Findings**
- 🟠 **POTENTIAL BUG + Misconception.** The comment is **wrong**: `readOnly = true` is a **hint** (flush mode + a possible driver/DB optimization), **not** an enforced guarantee that nothing writes. Here `markAccessed` runs a **native UPDATE query**, which bypasses Hibernate's flush entirely and **will** modify the DB despite `readOnly`. So the method does write, and the "safe to expose because it can't modify anything" reasoning is false.

**Fix:** don't rely on `readOnly` for safety/authorization. If the method should truly not write, remove `markAccessed` (or move it to a separate read-write path); if it should write, it's not read-only. For actual write-prevention, use DB permissions/a read replica, not the annotation.
**What a strong senior would say:** *"The claim is a misconception — `readOnly` is an optimization hint, not a write barrier. Proof: `markAccessed` is a native UPDATE that executes regardless of the flag, so this method does modify the database. I wouldn't treat `readOnly` as a security guarantee; if we need real write prevention, that's a database-permissions or read-replica concern."*
**Interviewer is testing:** knowing readOnly is a hint, not enforcement; native queries bypass it (false-security).
**Mention first:** readOnly isn't a write guarantee + the native UPDATE writes.

## Topic 26 · Exercise 1 — Long batch, no timeout
**Findings**
- 🟠 **DEFINITE BUG — Unbounded long transaction + locking.** Loading millions of rows into the persistence context and updating them in one transaction runs for a long time holding a connection, accumulating locks and undo/log, blocking other writers and risking lock timeouts/deadlocks; there's no `timeout` to bound it, and the whole thing is one giant atomic unit that can fail near the end and roll back everything.

**Fix:** chunk the work (paged batches, e.g. 1–5k rows/transaction), commit per chunk; use a bulk `UPDATE ... WHERE archived=false AND created < :cutoff` instead of entity-by-entity; add a transaction `timeout` and/or a statement timeout as a safety net; consider running off-peak.
**What a strong senior would say:** *"One transaction over millions of rows holds a connection and locks for the whole run, blocks writers, and risks a rollback of everything if it fails late. I'd do it as a bulk UPDATE, chunked with per-chunk commits so locks are short and progress is durable, and set a timeout as a backstop. Loading entities to flip a flag is also wasteful — a set-based UPDATE is far cheaper."*
**Interviewer is testing:** long-transaction risks; chunking + bulk update + timeouts.
**Mention first:** unbounded long transaction holding locks.

## Topic 26 · Exercise 2 — Timeout that misses the real hang
**Findings**
- 🟠 **POTENTIAL BUG — Transaction timeout doesn't bound the HTTP call.** `@Transactional(timeout = 5)` limits time spent in **transactional/DB operations** managed by Spring's transaction manager; it does **not** interrupt an in-progress external HTTP call. The `externalHttpClient.fetch` has no HTTP timeout of its own, so it can hang for minutes — the thread and connection stay blocked, and the 5s transaction timeout only triggers on the *next* DB operation, not during the hang.

**Fix:** set explicit **connect/read timeouts on the HTTP client** (the real fix), and ideally move the external call outside the transaction (Topic 21). The transaction timeout is a backstop for DB work, not a substitute for I/O timeouts.
**What a strong senior would say:** *"The transaction timeout bounds DB operations, not the HTTP call — so a hanging `fetch` isn't interrupted by it; it only fires when transactional work resumes. The actual fix is a connect/read timeout on the HTTP client, and better, get that call out of the transaction so it isn't holding a connection while it waits."*
**Interviewer is testing:** what @Transactional timeout actually governs; HTTP timeouts are separate.
**Mention first:** tx timeout doesn't cover the HTTP hang.

## Topic 27 · Exercise 1 — Internal call to a transactional method
**Findings**
- 🔴 **DEFINITE BUG — Self-invocation bypasses the proxy.** `runBilling` calls `billCustomer` on `this`, so the transactional proxy is bypassed and each billing runs **without a transaction** — the `charge` + invoice save aren't atomic, and the intended per-customer isolation doesn't exist. A failure can leave a customer charged but not invoiced.

**Fix:** invoke through the proxy — extract `billCustomer` into a separate bean and inject it, or self-inject the proxy (`@Lazy` self or `ObjectProvider`), so `@Transactional` (and ideally `REQUIRES_NEW` for per-customer isolation) actually applies.
**What a strong senior would say:** *"Calling `billCustomer` internally on `this` skips the proxy, so `@Transactional` never activates — each billing isn't atomic and isn't isolated. I'd move `billCustomer` to its own bean and inject it, and use REQUIRES_NEW so one customer's failure doesn't affect the others. Self-invocation is the classic reason a transaction 'silently doesn't work.'"*
**Interviewer is testing:** self-invocation and the proxy boundary.
**Mention first:** self-call → no transaction.

## Topic 27 · Exercise 2 — Self-invoked @Async
**Findings**
- 🟠 **DEFINITE BUG — @Async self-invocation runs synchronously.** `@Async` is proxy-based, exactly like `@Transactional`. `generateReport` calls `exportAsync` on `this`, bypassing the proxy, so `heavyExport` runs **synchronously on the caller's thread** — the request blocks for the whole export; no async. (Also `@EnableAsync` must be present, and `@Async` on a void method loses exceptions.)

**Fix:** call the async method through the proxy — put `exportAsync` in a separate bean and inject it (or self-inject), and ensure `@EnableAsync` + a configured task executor. Prefer returning `CompletableFuture` to observe failures.
**What a strong senior would say:** *"`@Async` has the same proxy limitation as `@Transactional`, so an internal self-call runs synchronously — the export blocks the request thread instead of going to a pool. I'd invoke it through a separate bean, confirm `@EnableAsync` and a real executor are configured, and return a CompletableFuture so exceptions aren't swallowed."*
**Interviewer is testing:** that proxy-based annotations (@Async too) fail on self-invocation.
**Mention first:** self-invoked @Async runs synchronously.

## Topic 28 · Exercise 1 — @Transactional on a final method
**Findings**
- 🟠 **DEFINITE BUG — Can't proxy a final method.** Spring uses CGLIB subclass proxies for classes (no interface here), which work by **overriding** methods — a `final` method can't be overridden, so the transactional advice isn't applied (Spring may log a warning; the method runs **without** a transaction). `bulkRecalculate` isn't transactional.

**Fix:** remove `final` from the method (or the class); or extract the method to an interface + interface-based proxy. Generally don't mark proxied methods/classes `final`.
**What a strong senior would say:** *"CGLIB proxies work by subclassing and overriding, and you can't override a final method — so `@Transactional` here is silently not applied and the bulk update runs without a transaction. I'd drop the `final`. It's an easy one to miss because it fails quietly rather than erroring."*
**Interviewer is testing:** CGLIB proxy limitation with final methods/classes.
**Mention first:** final method → advice not applied.

## Topic 28 · Exercise 2 — Transactional call from @PostConstruct
**Findings**
- 🟠 **DEFINITE BUG — Proxy not active during initialization + self-invocation.** `@PostConstruct init()` runs during bean initialization, and it calls `loadDefaults()` on `this` (self-invocation), so the transactional proxy isn't applied — `saveAll` runs **without a transaction**. (Even without the self-call, transactional behavior during `@PostConstruct` is unreliable because the proxy/transaction infrastructure may not be fully ready.)

**Fix:** move the data loading out of `@PostConstruct` to an `ApplicationRunner`/`@EventListener(ApplicationReadyEvent.class)` that calls a transactional method **on an injected bean** (through the proxy); or inject a self-proxy. Running DB initialization after the context is fully up is the robust pattern.
**What a strong senior would say:** *"Two things: the call is a self-invocation so the proxy is bypassed, and transactional behavior in `@PostConstruct` is unreliable anyway since the infrastructure isn't guaranteed ready. I'd load reference data from an ApplicationReadyEvent listener that invokes a transactional method on an injected bean, so the transaction actually applies and it runs after the context is up."*
**Interviewer is testing:** proxy/transaction availability during lifecycle callbacks + self-invocation.
**Mention first:** no transaction during @PostConstruct self-call.

## Topic 29 · Exercise 1 — Inner failure marks the whole tx rollback-only
**Findings**
- 🔴 **DEFINITE BUG — Catch doesn't save you; the tx is rollback-only.** `add` uses `REQUIRED`, so it joins `place`'s single physical transaction. When it throws, Spring marks that transaction **rollback-only**. Catching the exception lets `place` continue and even do more work, but at commit Spring throws `UnexpectedRollbackException` and the **entire** transaction (including the order save) rolls back. The "recommendation failure shouldn't fail the order" intent is violated — catching it isn't enough.

**Fix:** make the optional step run in its own transaction so its failure doesn't taint the outer one — `@Transactional(propagation = REQUIRES_NEW)` on `add` (its rollback is independent), or `NESTED` (savepoint) if the DB/driver supports it. Then catching the exception genuinely lets the order proceed.
**What a strong senior would say:** *"Because `add` is REQUIRED, throwing marks the shared transaction rollback-only, so even though we catch it, the commit fails with UnexpectedRollbackException and the order is rolled back too. Catching a REQUIRED inner failure doesn't isolate it. I'd run the recommendation in REQUIRES_NEW (or NESTED) so its failure stays contained and the order commits."*
**Interviewer is testing:** rollback-only marking and UnexpectedRollbackException — an advanced, high-value trap.
**Mention first:** caught inner exception still rolls back the whole tx.

## Topic 29 · Exercise 2 — NESTED expected to commit independently
**Findings**
- 🟠 **DEFINITE BUG (misconception) — NESTED savepoints aren't independent commits.** `NESTED` creates **savepoints** within the outer transaction; a NESTED sub-step can roll back to its savepoint without killing the outer tx, but completed NESTED steps are **not** committed independently — they only become durable when the **outer** transaction commits. So if step 5 fails and the outer transaction rolls back, steps 1–4 roll back too. The author's expectation (1–4 survive) is wrong. (Also NESTED requires JDBC savepoint support; not all setups enable it.)

**Fix:** if each step must be permanently committed regardless of later steps, use `REQUIRES_NEW` (each step commits in its own transaction), accepting that there's then no atomic all-or-nothing across steps — which means you also need compensation/idempotency for partial completion. Choose based on the real requirement (savepoint rollback vs independent durability).
**What a strong senior would say:** *"NESTED is savepoints, not independent transactions — completed sub-steps only persist when the outer transaction commits, so a later failure that rolls back the outer tx takes 1–4 with it. If the requirement is that each successful step stays committed, that's REQUIRES_NEW, and then I'd need compensation for the half-done run since there's no longer overall atomicity. NESTED also needs savepoint support, which not every setup has."*
**Interviewer is testing:** NESTED (savepoints) vs REQUIRES_NEW semantics — a genuine senior distinction.
**Mention first:** NESTED steps aren't independently committed.

## Topic 30 · Exercise 1 — No version field
**Findings**
- 🟠 **DEFINITE BUG — Lost update.** With no `@Version`, concurrent `adjustStock` calls both read the same stock, both compute from the stale value, and the second save overwrites the first — a lost update (e.g. two +/−10 adjustments net only one). No conflict detection.

**Fix:** add `@Version` for optimistic locking (a concurrent modification then throws `OptimisticLockException`, which you retry), or — for a pure increment — do an atomic `UPDATE product SET stock = stock + :delta WHERE id = :id`, which is race-free without locking. For a hot row, the atomic UPDATE is usually best.
**What a strong senior would say:** *"Read-modify-write with no version is a lost update — two concurrent adjustments clobber each other. For a simple delta I'd use an atomic `UPDATE ... SET stock = stock + :delta`, which is inherently race-free. If the logic is richer, I'd add `@Version` and retry on OptimisticLockException. Which one depends on whether it's a plain increment or conditional logic."*
**Interviewer is testing:** lost update; optimistic locking vs atomic update.
**Mention first:** lost update from RMW without version.

## Topic 30 · Exercise 2 — Version present, conflict unhandled
**Findings**
- 🟠 **DEFINITE BUG — Unhandled OptimisticLockException.** `@Version` is correctly present, so a concurrent edit correctly throws `OptimisticLockException` (or Spring's `ObjectOptimisticLockingFailureException`) on the losing save — but nothing handles it, so the user gets an opaque `500`. Detection works; UX/recovery doesn't. (Silently catching-and-ignoring would be worse — a lost update.)

**Fix:** handle the conflict meaningfully — map it to `409 Conflict` with a clear "the document changed, reload and reapply" message, and/or implement a retry (for mergeable operations) or a merge UI. Don't swallow it.
**What a strong senior would say:** *"The versioning is right and it correctly detects the conflict — the gap is the user experience: an unhandled OptimisticLockException surfaces as a 500. I'd catch it and return a 409 with a 'document was updated, please reload' message, or auto-retry if the change is safely mergeable. The one thing I wouldn't do is swallow it, which would reintroduce the lost update."*
**Interviewer is testing:** handling optimistic-lock conflicts gracefully (not just detecting them).
**Mention first:** conflict surfaces as a 500.

## Topic 31 · Exercise 1 — Lock held across an external call
**Findings**
- 🔴 **DEFINITE BUG — Pessimistic lock held across a 20s HTTP call.** `findByIdForUpdate` takes a `PESSIMISTIC_WRITE` row lock, then the code calls a up-to-20s external allocator **before** committing — so the row lock (and DB connection) is held for the whole external round-trip. Every other thread needing that product row blocks for up to 20s → severe contention/throughput collapse, and connection-pool pressure.

**Fix:** don't hold a DB lock across external I/O. Restructure: do the external allocation **outside** the lock, then take a short lock only to apply the result (re-validating), or use optimistic locking, or a reservation state machine. Keep the pessimistic critical section to fast DB-only work.
**What a strong senior would say:** *"We hold a write lock on the stock row across a 20-second HTTP call, so every concurrent reservation of that product blocks for up to 20 seconds and we pin a connection the whole time — that's a contention and pool disaster. I'd move the external allocation outside the lock and take a short lock only to apply the result, re-checking state. A DB lock should never span remote I/O."*
**Interviewer is testing:** pessimistic lock scope; never hold locks across I/O.
**Mention first:** lock held across the external call.

## Topic 31 · Exercise 2 — FOR UPDATE outside a transaction
**Findings**
- 🔴 **DEFINITE BUG — Pessimistic lock without a transaction does nothing useful.** `PESSIMISTIC_WRITE` requires an active transaction to hold the lock until commit. With no `@Transactional`, each repository call runs in its own auto-commit "transaction," so the `SELECT ... FOR UPDATE` acquires and **immediately releases** the lock when that call returns — by the time `save` runs, nothing is locked. Two concurrent `hold`s can both "lock," both set `heldBy`, and the second overwrites the first. The lock provides no protection.

**Fix:** annotate `hold` with `@Transactional` so the lock is held from the `SELECT ... FOR UPDATE` through the `save`/commit — making the read-check-write atomic against other transactions.
**What a strong senior would say:** *"A pessimistic lock only lives inside a transaction, and there's no `@Transactional` here, so the FOR UPDATE lock is released the instant the query method returns — the seat isn't protected during the update, and two holds can race. I'd wrap `hold` in `@Transactional` so the lock spans the read and the write. Pessimistic locking without a surrounding transaction is effectively a no-op."*
**Interviewer is testing:** that pessimistic locks need an enclosing transaction to be meaningful.
**Mention first:** no transaction → lock released immediately.

## Topic 32 · Exercise 1 — Read-modify-write across two requests
**Findings**
- 🟠 **DEFINITE BUG — Cross-request lost update.** Two editors both GET the article, both PUT their version; the second save overwrites the first with no detection — the first editor's changes are silently lost. There's no optimistic-concurrency token across the read/modify/write that spans two HTTP requests.

**Fix:** optimistic concurrency at the HTTP level — return a version/ETag on GET, require it on PUT (`@Version` on the entity, or an `If-Match` ETag); reject the stale write with `409 Conflict`/`412 Precondition Failed` so the user can reload and merge. Purely server-side transactions can't fix this because the two requests are separate — you need a version carried by the client.
**What a strong senior would say:** *"This is a lost update across two requests — because the read and the write are separate HTTP calls, a server transaction can't help; I need an optimistic token the client round-trips. I'd add `@Version`/an ETag returned on GET and required on PUT, and reject a stale update with 409 so the editor can reload and reapply, rather than silently clobbering the other person's edit."*
**Interviewer is testing:** HTTP-level optimistic concurrency (ETag/version), not just DB transactions.
**Mention first:** silent cross-request lost update.

## Topic 32 · Exercise 2 — Counter increment by load-modify-save
**Findings**
- 🟠 **DEFINITE BUG — Lost increments under concurrency.** `read views, +1, save` is a read-modify-write; thousands of concurrent views read the same value and write back the same `+1`, so most increments are lost — the count is badly undercounted. `@Transactional` alone doesn't prevent it under READ_COMMITTED (no lock on the row).

**Fix:** atomic DB update — `UPDATE article SET views = views + 1 WHERE id = :id` (race-free, no entity load). For a very hot counter, avoid per-view DB writes entirely: increment in Redis / an in-memory `LongAdder` and flush periodically, or use an append-and-aggregate approach — exact-per-view DB writes won't scale to thousands/sec regardless of correctness.
**What a strong senior would say:** *"Load-modify-save loses most increments under concurrency — everyone reads the same value and writes the same +1. The correct minimum is an atomic `UPDATE ... SET views = views + 1`. But at thousands of views per second I wouldn't hit the DB per view at all — I'd increment in Redis or a LongAdder and flush periodically, since exact synchronous DB writes on a hot counter are a scalability problem even when correct."*
**Interviewer is testing:** atomic update for counters + scalability of hot counters.
**Mention first:** lost increments from RMW.

---

# Senior Spring Boot Review Checklist — Part 4

**IoC / DI:** let the container wire — no `new` collaborators, no `context.getBean` service-locator · constructor injection with `final` fields (best practice — defend it) · field injection breaks tests and is null in the constructor · `@Autowired` on a `static` field doesn't work · ambiguous beans need `@Qualifier`/`@Primary` · one clear constructor (or `@Autowired` the intended one).
**Lifecycle / scope / config:** side-effecting init → `@PostConstruct`/ApplicationReadyEvent, not the constructor · every executor/resource needs `@PreDestroy` · `@Service`/controllers are **singletons** — no per-request mutable fields · prototype-in-singleton is created once (use `ObjectProvider`/scoped proxy) · constructor cycles fail at startup (redesign; `@Lazy` only masks) · `@Value` needs a default or validated `@ConfigurationProperties` · `@Bean` inter-method calls only share the singleton inside `@Configuration` (CGLIB), not `@Component`.
**REST:** thin controllers (no business logic, no shared state) · correct verbs (PUT=replace vs PATCH=partial; no GET body) · correct codes (201 create, 400 client error, 404 not-found, 409 conflict — never 200-for-errors or 500-for-validation) · `@Valid` is what triggers validation · never bind/return entities (over-posting in, secret leak + lazy-serialization out) · centralize errors in `@RestControllerAdvice`, never leak stack traces · cap client page size and always page with a stable sort (keyset for deep pages) · idempotency keys for create/pay, enforced by a DB unique constraint (not check-then-act).
**Security:** don't `permitAll` sensitive matchers · never trust client-supplied identity (`X-User-Id`) or role — use the authenticated principal · check object ownership (IDOR) · authorize with `@PreAuthorize`/security context.
**Transactions & proxies:** `@Transactional` needs a **public**, proxy-invoked, non-`final` method — private/self-invoked/final/`@PostConstruct` calls silently get **no** transaction (same for `@Async`) · the boundary must span the atomic unit and **exclude external I/O** · default rollback is unchecked-only (`rollbackFor` for checked) · a swallowed exception or a caught **REQUIRED** inner failure still commits/`UnexpectedRollbackException`s · `readOnly` disables flush (writes silently lost) and is a **hint, not a write guarantee** (native queries bypass it) · tx timeout bounds DB ops, not HTTP calls · REQUIRES_NEW = independent commit (good for audit; costly per-item at scale); NESTED = savepoints that still die with the outer rollback.
**Concurrency at the DB:** lost updates → `@Version` (+ handle `OptimisticLockException` as 409, don't swallow) or atomic `UPDATE ... SET x = x + :d` · pessimistic locks need an enclosing transaction and must **never** be held across I/O · cross-request edits need HTTP-level optimistic concurrency (ETag/version) · hot counters → atomic update or Redis/LongAdder, not per-request DB writes.

**Always:** name at least one *not-a-bug* (correct constructor injection, additive API field) to show judgment; separate a guaranteed bug from a "depends on the client contract / dataset size / isolation" concern and **ask**; lead with security 🔴 (open admin, header identity, IDOR, mass assignment, leaked secrets) and money/atomicity 🔴 (no transaction, self-invocation, lock across I/O).

---

*End of Part 4. Say **"next"** for Part 5 — JPA / Hibernate Code Review (17 topics × 2 = 34 exercises), the final part.*
