# SOLID Principles — Traps, Violations & Senior Interview Questions

---

## SRP Traps

---

### Trap 1 — "God Class" That Grows Silently

The most dangerous SRP violation because it happens gradually. Every sprint someone adds "just one more method" and no single addition seems wrong.

```java
// Starts innocent — becomes a monster over time
public class UserService {
    // Authentication concerns
    public User login(String email, String password) { ... }
    public void logout(String sessionId) { ... }
    public String generateJwtToken(User user) { ... }
    public boolean validateToken(String token) { ... }

    // Profile concerns
    public void updateProfile(long userId, ProfileDto dto) { ... }
    public void changePassword(long userId, String newPassword) { ... }
    public void uploadAvatar(long userId, byte[] image) { ... }

    // Notification concerns
    public void sendWelcomeEmail(User user) { ... }
    public void sendPasswordResetEmail(String email) { ... }
    public void sendSmsVerification(String phone) { ... }

    // Analytics concerns
    public void trackLoginEvent(User user) { ... }
    public void trackProfileUpdate(long userId) { ... }

    // Admin concerns
    public List<User> getAllUsers() { ... }
    public void banUser(long userId) { ... }
    public void exportUsersToCSV() { ... }
}
// This class has 6 different reasons to change
// Every team touches it — merge conflicts guaranteed
```

**Fix:** One service per concern — `AuthService`, `UserProfileService`, `NotificationService`, `UserAnalyticsService`, `UserAdminService`.

---

### Trap 2 — Utility Classes as SRP Dumping Grounds

```java
// "Util" and "Helper" classes are SRP violations dressed in humble names
public class StringUtils {
    public static String trim(String s) { ... }
    public static boolean isEmail(String s) { ... }
    public static String toSlug(String title) { ... }
    public static String maskCreditCard(String number) { ... }
    public static String formatCurrency(double amount, Currency c) { ... }
    public static boolean isPalindrome(String s) { ... }
}
// These methods have nothing to do with each other — multiple responsibilities
```

**Fix:** Group by actual cohesion — `EmailValidator`, `SlugGenerator`, `PaymentFormatter`.

---

### Trap 3 — SRP Confused with "One Method Per Class"

```java
// WRONG interpretation — over-engineering, not SRP
public class CalculateTax { public double calculate(Order o) { ... } }
public class ApplyDiscount { public double apply(Order o) { ... } }
public class FormatTotal   { public String format(double total) { ... } }

// CORRECT — all of these belong together, they share one responsibility: pricing
public class PricingService {
    public double calculateTax(Order order) { ... }
    public double applyDiscount(Order order, Discount d) { ... }
    public String formatTotal(double total) { ... }
}
```

SRP is about **cohesion of purpose**, not method count.

---

### Trap 4 — SRP at Wrong Granularity

```java
// Too coarse — entire payment system in one class
public class PaymentProcessor {
    public void processCreditCard(Payment p) { ... }
    public void processPayPal(Payment p) { ... }
    public void processCrypto(Payment p) { ... }
    public void processRefund(Payment p) { ... }
    public void generateReceipt(Payment p) { ... }
    public void notifyMerchant(Payment p) { ... }
    // Adding a new payment method = editing this class (also OCP violation)
}
```

Each payment method is a separate reason to change. Receipt generation and merchant notification are separate concerns entirely.

---

## OCP Traps

---

### Trap 1 — Adding a New "Type" Requires Editing Multiple Classes

```java
// Before adding "Triangle":
class AreaCalculator    { if circle... if rectangle... }
class PerimeterCalc     { if circle... if rectangle... }
class ShapeRenderer     { if circle... if rectangle... }
class ShapeSerializer   { if circle... if rectangle... }

// After adding "Triangle":
class AreaCalculator    { if circle... if rectangle... if triangle... } // edit
class PerimeterCalc     { if circle... if rectangle... if triangle... } // edit
class ShapeRenderer     { if circle... if rectangle... if triangle... } // edit
class ShapeSerializer   { if circle... if rectangle... if triangle... } // edit
```

Every `if-instanceof` chain is a symptom. When you add a new type and have to grep the codebase for every place to add another branch — OCP is violated.

---

### Trap 2 — OCP Applied to the Wrong Level

```java
// Over-applying OCP — premature abstraction
public interface TaxCalculator { double calculate(double amount); }
public class StandardTaxCalculator implements TaxCalculator { ... }
// ...just to calculate tax once, that never changes

// OCP doesn't mean "wrap everything in an interface"
// It means: identify the axes of variation that WILL change, protect those
```

OCP should be applied to the parts of your system that have a history of changing or are likely to change. Over-abstracting stable code adds complexity with no benefit.

---

### Trap 3 — Modifying a Published API (Classic OCP Violation)

```java
// v1 — clients depend on this
public class ReportService {
    public Report generate(String type) { ... }
}

// v2 — breaking change: adding a required parameter
public class ReportService {
    public Report generate(String type, ReportConfig config) { ... } // BREAKS all callers
}

// OCP-compliant v2 — extend, don't modify
public class ReportService {
    public Report generate(String type) { return generate(type, ReportConfig.defaults()); }
    public Report generate(String type, ReportConfig config) { ... } // new overload
}
```

---

### Trap 4 — Open for the Wrong Kind of Extension

```java
// This is "open" but for the wrong thing — inheritance isn't always the answer
public class DataExporter {
    public void export(List<Data> data) {
        // 500 lines of logic
    }

    // Subclasses override the whole method — fragile, duplicates logic
}

// Better: open via composition (Strategy Pattern)
public class DataExporter {
    private final ExportFormat format;
    private final ExportDestination destination;

    public DataExporter(ExportFormat format, ExportDestination destination) {
        this.format = format;
        this.destination = destination;
    }

    public void export(List<Data> data) {
        byte[] formatted = format.format(data); // extension point 1
        destination.write(formatted);           // extension point 2
    }
}
// Add CSV, JSON, XML formats without touching DataExporter
// Add file, S3, email destinations without touching DataExporter
```

---

## LSP Traps

---

### Trap 1 — UnsupportedOperationException in Subclasses

The most common LSP violation in Java. Often found in collection subclasses.

```java
// Java's own Collections.unmodifiableList violates LSP!
List<String> mutable   = new ArrayList<>(Arrays.asList("a", "b"));
List<String> immutable = Collections.unmodifiableList(mutable);

// List contract implies add() works — immutable throws UnsupportedOperationException
immutable.add("c"); // throws at runtime — code written against List's contract breaks

// The lesson: don't use inheritance when the subtype can't honor the supertype's full contract
```

---

### Trap 2 — Strengthening Preconditions Silently

```java
public class EmailSender {
    // PRECONDITION: recipient != null
    public void send(String recipient, String body) {
        Objects.requireNonNull(recipient, "Recipient required");
        // send...
    }
}

public class CorporateEmailSender extends EmailSender {
    @Override
    public void send(String recipient, String body) {
        // STRENGTHENS precondition — now requires corporate domain too
        if (!recipient.endsWith("@company.com")) {
            throw new IllegalArgumentException("Only corporate emails allowed");
        }
        super.send(recipient, body);
    }
}

EmailSender sender = new CorporateEmailSender();
sender.send("user@gmail.com", "Hello"); // parent said this was fine — now it throws
```

---

### Trap 3 — Weakening Postconditions

```java
public class InvoiceRepository {
    // POSTCONDITION: always returns a non-null, non-empty list (at minimum empty list)
    public List<Invoice> findByCustomer(long customerId) {
        return jdbcTemplate.query(...); // returns empty list when none found
    }
}

public class CachedInvoiceRepository extends InvoiceRepository {
    @Override
    public List<Invoice> findByCustomer(long customerId) {
        List<Invoice> cached = cache.get(customerId);
        return cached; // returns NULL on cache miss — breaks callers who trust the contract!
    }
}

// Fix:
return cached != null ? cached : Collections.emptyList();
```

---

### Trap 4 — The History Constraint Violation

```java
public class ImmutablePoint {
    protected int x, y;
    public ImmutablePoint(int x, int y) { this.x = x; this.y = y; }
    // INVARIANT: x and y never change after construction
    public int getX() { return x; }
    public int getY() { return y; }
}

public class MutablePoint extends ImmutablePoint {
    public MutablePoint(int x, int y) { super(x, y); }

    // VIOLATES history constraint — parent guarantees immutability
    public void setX(int x) { this.x = x; }
    public void setY(int y) { this.y = y; }
}

// Code that caches an ImmutablePoint assuming it won't change:
ImmutablePoint p = new MutablePoint(1, 2);
cache.put("origin", p);
((MutablePoint) p).setX(99); // silently corrupts the cache
```

---

## ISP Traps

---

### Trap 1 — Interface That Grows With Every Feature Request

```java
// v1
public interface PaymentGateway {
    PaymentResult charge(double amount, String cardToken);
}

// v2 — someone adds refund
public interface PaymentGateway {
    PaymentResult charge(double amount, String cardToken);
    RefundResult refund(String transactionId); // not all gateways support this
}

// v3 — someone adds recurring billing
public interface PaymentGateway {
    PaymentResult charge(double amount, String cardToken);
    RefundResult refund(String transactionId);
    SubscriptionResult createSubscription(SubscriptionPlan plan); // PayPal doesn't support this
    void cancelSubscription(String subscriptionId);
}

// Now every gateway must implement everything or stub it with UnsupportedOperationException
public class SimpleGateway implements PaymentGateway {
    public PaymentResult charge(...)       { /* real impl */ }
    public RefundResult refund(...)        { throw new UnsupportedOperationException(); }
    public SubscriptionResult createSubscription(...) { throw new UnsupportedOperationException(); }
    public void cancelSubscription(...)   { throw new UnsupportedOperationException(); }
}

// Fix: segregate by capability
public interface Chargeable     { PaymentResult charge(double amount, String cardToken); }
public interface Refundable     { RefundResult refund(String transactionId); }
public interface Subscribable   {
    SubscriptionResult createSubscription(SubscriptionPlan plan);
    void cancelSubscription(String subscriptionId);
}

public class FullFeaturedGateway implements Chargeable, Refundable, Subscribable { ... }
public class SimpleGateway      implements Chargeable { ... } // only what it can do
```

---

### Trap 2 — ISP Confused with Interface Explosion

```java
// Over-applying ISP — now you have 30 single-method interfaces
public interface Saveable     { void save(); }
public interface Deletable    { void delete(); }
public interface Findable     { Object find(long id); }
public interface Updatable    { void update(Object o); }
public interface Countable    { int count(); }
// Every class implements 5 interfaces — noise, no clarity

// Better: group by coherent use case
public interface Repository<T> {
    void save(T entity);
    Optional<T> findById(long id);
    void delete(long id);
    List<T> findAll();
}
// Consistent, predictable, doesn't fragment without reason
```

---

### Trap 3 — Client Depends on Fat Interface It Doesn't Use

```java
public interface ReportService {
    Report generatePDF(ReportRequest req);
    Report generateExcel(ReportRequest req);
    Report generateCSV(ReportRequest req);
    void scheduleReport(ReportRequest req, Schedule s);
    void emailReport(Report report, String recipient);
    List<Report> getHistory(long userId);
}

// This class only needs PDF generation — depends on the whole interface
public class DashboardController {
    private ReportService reportService; // forced to mock 5 methods it doesn't use in tests

    public void downloadReport(ReportRequest req) {
        Report pdf = reportService.generatePDF(req);
        // ...
    }
}

// Fix: depend only on what you use
public interface PdfReportGenerator {
    Report generatePDF(ReportRequest req);
}
public class DashboardController {
    private PdfReportGenerator pdfGenerator; // minimal, testable
}
```

---

## DIP Traps

---

### Trap 1 — new Inside Business Logic (Hidden Dependency)

```java
// High-level class creates its own low-level dependency — impossible to test
public class OrderService {
    private final PaymentProcessor processor;
    private final EmailService emailService;
    private final Logger logger;

    public OrderService() {
        this.processor    = new StripePaymentProcessor();   // hard-coded to Stripe
        this.emailService = new SendGridEmailService();     // hard-coded to SendGrid
        this.logger       = new FileLogger("/var/log/app"); // hard-coded to file log
    }
}

// To test OrderService you need a real Stripe account, SendGrid account, and file system
// Switching payment processors means editing OrderService
```

**Fix:** Inject all dependencies via constructor. The class should never `new` up its collaborators.

---

### Trap 2 — Depending on Concrete Class Even With Injection

```java
// The type is the problem, not just the instantiation
public class OrderService {
    private final MySQLOrderRepository repository; // concrete class — still DIP violation

    public OrderService(MySQLOrderRepository repository) { // injected but still wrong
        this.repository = repository;
    }
}

// Fix: depend on the abstraction
public class OrderService {
    private final OrderRepository repository; // interface — DIP satisfied

    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }
}
```

---

### Trap 3 — Abstraction at the Wrong Level (Leaky Details)

```java
// The interface leaks infrastructure details — abstraction is polluted
public interface OrderRepository {
    void save(Order order);
    Order findById(long id);
    ResultSet executeQuery(String sql);       // SQL leaking into business interface!
    Connection getConnection();              // database detail in domain interface!
}

// High-level code now indirectly depends on JDBC — DIP violated despite the interface

// Fix: keep the interface in the domain language
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(long id);
    List<Order> findByCustomer(long customerId);
    // No SQL, no connections — pure business operations
}
```

---

### Trap 4 — Service Locator as Hidden DIP Violation

```java
// Looks like DIP but isn't — dependencies are still hidden inside the class
public class OrderService {
    public void placeOrder(Order order) {
        OrderRepository repo = ServiceLocator.get(OrderRepository.class); // hidden dependency
        repo.save(order);
    }
}

// You can't tell from the constructor what OrderService needs
// Tests must configure ServiceLocator globally — test pollution
// True DIP: dependencies declared explicitly in constructor
public class OrderService {
    private final OrderRepository repository;
    public OrderService(OrderRepository repository) { this.repository = repository; }
}
```

---

### Trap 5 — Circular Dependency (DIP Done Wrong)

```java
// Module A depends on Module B
public class OrderService {
    private final InvoiceService invoiceService;
}

// Module B depends on Module A — circular!
public class InvoiceService {
    private final OrderService orderService;
}

// Both classes know about each other — they're really one entangled module
// Symptom: cannot instantiate either without the other

// Fix: introduce a third abstraction both depend on
public interface OrderEventPublisher {
    void publish(OrderPlacedEvent event);
}
// OrderService publishes events, InvoiceService listens
// Neither knows about the other — dependency cycle broken
```

---

## Senior Interview Questions & Model Answers

---

### Q1: "Can you give a real-world example of an OCP violation you've seen or fixed?"

**Model answer structure:**
1. Describe the before state — "We had a `ReportGenerator` with a giant `if-else` block for each format."
2. Explain the pain — "Every new format required editing the class, re-testing everything, risky."
3. Describe the fix — "Introduced a `ReportFormat` interface, moved each format to its own class."
4. Result — "Adding a new format became a 30-minute job that touched zero existing code."

---

### Q2: "How do SRP and OCP relate to each other?"

> "SRP defines the boundaries of a class — one reason to change. OCP then says when you add a new reason (a new behavior), express it as a new class rather than modifying existing ones. If you follow SRP, OCP becomes much easier because each class is already small and focused — there's a natural place to extend rather than edit."

---

### Q3: "Is it ever okay to violate SOLID?"

**Yes — expected senior answer:**

> "SOLID principles are guidelines, not laws. Applying them to code that will never change, or to very small scripts, adds complexity without benefit. The principles exist to manage complexity in growing systems. I apply them most strictly at boundaries that are likely to change — payment methods, notification channels, data stores, business rules. I apply them loosely or not at all to stable infrastructure plumbing or throwaway scripts."

Key violation scenarios:
- Simple CRUD with no business logic — DIP overhead isn't worth it
- Performance-critical code where indirection (interfaces, polymorphism) costs measurably
- Prototypes/spikes — clarity over structure

---

### Q4: "How does the Dependency Inversion Principle enable unit testing?"

```java
// Without DIP — impossible to unit test OrderService
public class OrderService {
    public void placeOrder(Order order) {
        new MySQLRepository().save(order); // needs real DB
        new StripeProcessor().charge(...); // needs real Stripe
        new SendGrid().send(...);          // needs real email
    }
}

// With DIP — test with fast, deterministic fakes
public class OrderServiceTest {
    @Test
    public void placeOrder_savesOrderAndChargesCard() {
        InMemoryOrderRepository repo     = new InMemoryOrderRepository();
        FakePaymentProcessor    payments = new FakePaymentProcessor();
        FakeEmailService        emails   = new FakeEmailService();

        OrderService service = new OrderService(repo, payments, emails);
        service.placeOrder(testOrder);

        assertThat(repo.findById(testOrder.getId())).isPresent();
        assertThat(payments.wasCharged()).isTrue();
        assertThat(emails.getSentCount()).isEqualTo(1);
    }
}
// Fast, no network, no database, no external services, runs in milliseconds
```

---

### Q5: "What's the difference between ISP and SRP?"

| | SRP | ISP |
|---|---|---|
| Applies to | Classes | Interfaces |
| Concern | A class has one reason to change | A client depends only on methods it uses |
| Violation sign | Class touches too many concerns | Interface has methods some implementors stub as unsupported |
| Fix | Split the class | Split the interface |

They're complementary — ISP prevents fat interfaces, SRP prevents fat classes.

---

### Q6: "Walk me through how you'd refactor this class to follow all SOLID principles."

```java
// Common interview whiteboard class:
public class UserManager {
    private Connection dbConn = DriverManager.getConnection("jdbc:mysql://...");

    public void registerUser(String email, String password) {
        String hash = DigestUtils.md5(password);
        dbConn.execute("INSERT INTO users VALUES ('" + email + "', '" + hash + "')");
        sendWelcomeEmail(email);
        logRegistration(email);
    }

    private void sendWelcomeEmail(String email) { /* SMTP logic */ }
    private void logRegistration(String email) { /* file logging */ }
}
```

**Walk through:**
- **SRP:** Split into `UserService` (registration logic), `UserRepository` (DB), `EmailService` (email), `AuditLogger` (logging)
- **OCP:** `EmailService` as interface — can swap implementations without changing `UserService`
- **LSP:** Ensure any `EmailService` implementation fulfills the contract (no exceptions on valid input)
- **ISP:** `UserRepository` only exposes user-related methods, not generic DB operations
- **DIP:** `UserService` receives `UserRepository`, `EmailService`, `AuditLogger` via constructor — no `new`, no `DriverManager` in business logic

---

### Q7: "What's wrong with this interface?"

```java
public interface Animal {
    void eat();
    void sleep();
    void fly();
    void swim();
    void breatheUnderwater();
}
```

**Expected answer:**
- ISP violation — `Dog` can't fly, `Eagle` can't swim, `Fish` can't sleep above water
- Forces all implementors to stub meaningless methods or throw `UnsupportedOperationException`
- LSP violation — any code using `Animal` and calling `fly()` breaks for `Dog`
- Fix: segregate into `Eatable`, `Flyable`, `Swimmable`, `AquaticBreather` — implement only what applies

---

## The SOLID Smell Detector

Use these smells to quickly spot violations in code review:

| Smell | Likely Violation |
|---|---|
| Class has `and` in its name (`UserAndInvoiceService`) | SRP |
| Method has a `switch`/`if` on a type field | OCP |
| Subclass method throws `UnsupportedOperationException` | LSP or ISP |
| Interface has 10+ methods | ISP |
| Class has `new SomeConcreteClass()` in business methods | DIP |
| Class is impossible to unit test without infrastructure | DIP |
| Adding a feature requires changing 5 existing files | OCP |
| Class named `*Manager`, `*Handler`, `*Utils` | SRP |
| Interface method parameter is a `Connection` or `ResultSet` | DIP (leaky abstraction) |

---

## The Senior Mental Model

> **SOLID is not about following rules — it's about managing change.**
>
> SRP makes change *safe* (small, focused classes).
> OCP makes change *cheap* (extend instead of modify).
> LSP makes change *predictable* (substitution works).
> ISP makes change *isolated* (small contracts, small blast radius).
> DIP makes change *flexible* (swap implementations freely).
>
> A codebase that follows SOLID is one where you can add a new feature on a Friday afternoon without fear.
