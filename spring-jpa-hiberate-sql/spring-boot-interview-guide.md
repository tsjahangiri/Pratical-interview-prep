# 🏗️ Senior Spring Boot Engineer — Complete Interview Guide
## 15+ Years of Architecture Wisdom in One Document

> *Written from the perspective of a senior architect who has conducted hundreds of senior-level interviews. Every question here reflects what separates a 3-year developer from a 7+ year engineer. The difference is never syntax — it's always "why" and "what breaks in production."*

---

## 📋 Table of Contents

1. [Core Spring Boot Concepts](#1-core-spring-boot-concepts)
2. [Spring Framework Internals](#2-spring-framework-internals)
3. [Microservices Architecture](#3-microservices-architecture)
4. [Spring Data & Hibernate](#4-spring-data--hibernate)
5. [REST API Design & Best Practices](#5-rest-api-design--best-practices)
6. [Security](#6-security)
7. [Performance & Scalability](#7-performance--scalability)
8. [Testing](#8-testing)
9. [DevOps & Production](#9-devops--production)
10. [Coding & Design Problems](#10-coding--design-problems)
11. [Debugging & Tricky Questions](#11-debugging--tricky-questions)
12. [Rapid Revision Cheat Sheet](#12-rapid-revision-cheat-sheet)

---

## 1. Core Spring Boot Concepts

---

### Q1: How does Spring Boot auto-configuration actually work internally?

#### 🎯 What Interviewers Are Testing
Not whether you know `@EnableAutoConfiguration` exists — they want to know if you can trace the internal mechanism and debug it when it misbehaves in production.

#### 📖 Deep Explanation

**The Chain of Events at Startup:**

```
@SpringBootApplication
    → @EnableAutoConfiguration
        → AutoConfigurationImportSelector
            → Reads META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
                → Applies @Conditional filters
                    → Registers matching beans
```

```java
// @SpringBootApplication is a composed annotation:
@SpringBootConfiguration   // = @Configuration
@EnableAutoConfiguration   // triggers auto-config machinery
@ComponentScan             // scans current package and sub-packages
public @interface SpringBootApplication {}

// AutoConfigurationImportSelector reads this file (Spring Boot 3.x):
// META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
// Example entry:
// org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration
```

**Conditional Annotations — The Gatekeepers**

```java
// Each auto-config class is guarded by conditions
@AutoConfiguration
@ConditionalOnClass(DataSource.class)       // only if DataSource is on classpath
@ConditionalOnMissingBean(DataSource.class) // only if YOU haven't defined one
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }
}
```

**The Full List of Key Conditionals:**

| Annotation | Activates When |
|-----------|---------------|
| `@ConditionalOnClass` | Specified class is on the classpath |
| `@ConditionalOnMissingClass` | Specified class is NOT on classpath |
| `@ConditionalOnBean` | Specified bean exists in context |
| `@ConditionalOnMissingBean` | Specified bean does NOT exist |
| `@ConditionalOnProperty` | Property has specific value |
| `@ConditionalOnWebApplication` | App is a web application |
| `@ConditionalOnExpression` | SpEL expression evaluates to true |

**Debugging Auto-Configuration**

```bash
# Run with debug flag to see auto-config report
java -jar app.jar --debug

# Or in application.properties:
debug=true
logging.level.org.springframework.boot.autoconfigure=DEBUG

# Output shows three sections:
# Positive matches (applied) ✅
# Negative matches (skipped) ❌
# Exclusions 🚫
```

```java
// Excluding specific auto-configuration
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    SecurityAutoConfiguration.class
})
public class MyApp {}

// Or via properties (useful in tests)
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration
```

#### 🏭 Real-World Scenario
*Your team adds `spring-boot-starter-data-redis` as a dependency. Suddenly a `RedisConnectionFactory` is expected, and your app fails to start because Redis isn't running in CI. Why? Because `RedisAutoConfiguration` activated. Fix: either add Redis to CI, or exclude the auto-config in test profiles.*

#### 🔁 Follow-Up Tricky Questions
- *"How would you create your own Spring Boot starter with custom auto-configuration?"*
- *"What's the difference between `@Configuration` and `@AutoConfiguration`?"*
- *"If two auto-configurations both try to create a `DataSource` bean, which wins?"*

> **Answer to last Q:** `@ConditionalOnMissingBean` prevents both from activating simultaneously — whichever runs first creates the bean, the other sees it and skips. Order can be controlled with `@AutoConfigureBefore` / `@AutoConfigureAfter`.

#### ⚠️ Common Mistakes
- Thinking auto-config is "magic" — it's just `@Configuration` with `@Conditional` guards
- Not knowing how to disable auto-config when it misbehaves
- Confusing "classpath detection" with "actually connecting to the service"

---

### Q2: Explain the Spring Boot application startup lifecycle in detail.

#### 📖 Deep Explanation

```
SpringApplication.run()
    ↓
1. Create SpringApplication instance
    - Detect application type (SERVLET, REACTIVE, NONE)
    - Load SpringApplicationRunListeners (from spring.factories)
    - Load ApplicationContextInitializers
    - Load ApplicationListeners

2. Environment preparation
    - Load application.properties / application.yml
    - Activate profiles (@Profile)
    - Fire EnvironmentPreparedEvent

3. ApplicationContext creation
    - Create AnnotationConfigServletWebServerApplicationContext
    - Apply ApplicationContextInitializers

4. Context refresh (the heavy lifting)
    - Bean definition scanning (@ComponentScan)
    - Auto-configuration loading
    - Bean instantiation and dependency injection
    - BeanPostProcessor processing
    - @Bean method invocation
    - ApplicationContext published event

5. Post-startup hooks
    - CommandLineRunner.run()
    - ApplicationRunner.run()
    - ApplicationReadyEvent fired

6. Server starts (Tomcat/Netty)
```

```java
// Hooks you can use in each phase:

// Phase 1: Before context is even created
public class MyInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {
    @Override
    public void initialize(ConfigurableApplicationContext ctx) {
        // Useful for programmatic property sources
        ctx.getEnvironment().getPropertySources()
            .addFirst(new MapPropertySource("custom", Map.of("app.mode", "init")));
    }
}

// Phase 5a: After context refresh, receives command-line args as array
@Component
public class DataLoader implements CommandLineRunner {
    @Override
    public void run(String... args) throws Exception {
        // Load seed data, run migrations, warm up caches
        log.info("Loading initial data...");
    }
}

// Phase 5b: After context refresh, receives ApplicationArguments (structured)
@Component
public class AppRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) throws Exception {
        if (args.containsOption("migrate")) {
            runMigrations();
        }
    }
}

// Event listener approach
@Component
public class StartupListener {

    @EventListener(ApplicationReadyEvent.class)
    public void onApplicationReady() {
        // Fires AFTER everything is ready including web server
        // Best place for: cache warming, health check registration
    }

    @EventListener(ContextRefreshedEvent.class)
    public void onContextRefreshed() {
        // Fires on refresh — including context restarts in tests!
        // Can fire multiple times — be careful with one-time initialization
    }
}
```

**`@PostConstruct` vs `CommandLineRunner` vs `ApplicationRunner`**

| Hook | Scope | Use Case |
|------|-------|----------|
| `@PostConstruct` | Single bean | Initialize the bean itself (e.g., load config) |
| `InitializingBean.afterPropertiesSet()` | Single bean | Same as above, Spring-specific |
| `CommandLineRunner` | Application | Post-startup tasks needing all beans available |
| `ApplicationRunner` | Application | Same but with structured argument access |
| `@EventListener(ApplicationReadyEvent)` | Application | After web server is UP — safe for external calls |

#### 🔁 Follow-Up Tricky Questions
- *"What's the difference between `ContextRefreshedEvent` and `ApplicationReadyEvent`?"*
- *"Why should you avoid making HTTP calls in `@PostConstruct`?"*

> **Answer:** `@PostConstruct` runs during bean initialization — before the entire context is ready. Other beans may not exist yet. Calling an external service from `@PostConstruct` can also block startup or fail before connection pools are ready.

---

### Q3: What is `@SpringBootTest` vs `@WebMvcTest` vs `@DataJpaTest`? When do you use each?

#### 📖 Deep Explanation

```java
// @SpringBootTest: loads ENTIRE application context
// Use for: integration tests that need the full stack
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderServiceIntegrationTest {
    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldCreateOrder() {
        ResponseEntity<OrderDTO> response =
            restTemplate.postForEntity("/api/orders", new CreateOrderRequest(...), OrderDTO.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    }
}

// @WebMvcTest: loads ONLY web layer (controllers, filters, security)
// No service beans, no repository beans — they must be mocked
// Use for: testing controller logic, request/response mapping, validation
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @MockBean  // Required: service not loaded by @WebMvcTest
    private OrderService orderService;

    @Test
    void shouldReturn400WhenOrderAmountIsNegative() throws Exception {
        mockMvc.perform(post("/api/orders")
            .contentType(MediaType.APPLICATION_JSON)
            .content("{\"amount\": -100}"))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors[0].field").value("amount"));
    }
}

// @DataJpaTest: loads ONLY JPA layer (repositories, entities)
// Uses in-memory H2 by default, wraps each test in a transaction (auto-rollback)
// No web layer, no services
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE) // use real DB
class OrderRepositoryTest {
    @Autowired
    private OrderRepository orderRepository;

    @Test
    void shouldFindPendingOrdersOlderThan24Hours() {
        Order staleOrder = new Order(Status.PENDING, Instant.now().minus(25, HOURS));
        orderRepository.save(staleOrder);

        List<Order> result = orderRepository.findStaleOrders(Instant.now().minus(24, HOURS));
        assertThat(result).hasSize(1);
    }
}
```

**Slice Test Comparison**

| Annotation | Context Loaded | Speed | Use Case |
|-----------|---------------|-------|----------|
| `@SpringBootTest` | Full | Slow | End-to-end integration |
| `@WebMvcTest` | Web layer only | Fast | Controller logic, validation |
| `@DataJpaTest` | JPA layer only | Fast | Repository queries |
| `@JsonTest` | Jackson only | Very fast | Serialization/deserialization |
| `@RestClientTest` | RestTemplate | Fast | HTTP client testing |

---

## 2. Spring Framework Internals

---

### Q4: Explain the Bean lifecycle in detail. What are BeanPostProcessors and how does Spring use them?

#### 📖 Deep Explanation

```
Bean Lifecycle (detailed):

1.  BeanDefinition loaded (from @Component scan or @Bean methods)
2.  BeanFactoryPostProcessor runs (modifies bean DEFINITIONS before instantiation)
3.  Bean instantiated (constructor called)
4.  Properties injected (@Autowired, @Value)
5.  BeanNameAware.setBeanName() called
6.  BeanFactoryAware.setBeanFactory() called
7.  ApplicationContextAware.setApplicationContext() called
8.  BeanPostProcessor.postProcessBeforeInitialization() called   ← AOP, @Async, etc.
9.  @PostConstruct method called
10. InitializingBean.afterPropertiesSet() called
11. Custom init-method called (@Bean(initMethod="init"))
12. BeanPostProcessor.postProcessAfterInitialization() called    ← Proxy wrapping happens HERE
13. Bean is READY (put in singleton cache)

--- Application running ---

14. DisposableBean.destroy() called on shutdown
15. @PreDestroy method called
16. Custom destroy-method called
```

```java
// BeanPostProcessor is how Spring implements most of its magic
public interface BeanPostProcessor {
    // Called BEFORE @PostConstruct
    default Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }

    // Called AFTER @PostConstruct
    // This is where AOP PROXIES ARE CREATED
    default Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;
    }
}

// Real example: Spring creates @Transactional proxies via
// InfrastructureAdvisorAutoProxyCreator (a BeanPostProcessor)
// This is why self-invocation bypasses @Transactional —
// the proxy is external to the bean instance!

// Custom BeanPostProcessor example: auto-encrypt @Sensitive fields
@Component
public class SensitiveFieldEncryptionPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // Scan for @Sensitive annotations on fields and wrap the bean
        if (hasSensitiveFields(bean.getClass())) {
            return Proxy.newProxyInstance(
                bean.getClass().getClassLoader(),
                bean.getClass().getInterfaces(),
                new SensitiveFieldInterceptor(bean)
            );
        }
        return bean;
    }
}
```

**`BeanFactoryPostProcessor` vs `BeanPostProcessor`**

```java
// BeanFactoryPostProcessor: modifies bean DEFINITIONS (metadata)
// Runs BEFORE any bean is instantiated
@Component
public class PropertyOverrideProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) {
        BeanDefinition bd = factory.getBeanDefinition("dataSource");
        bd.getPropertyValues().add("maxPoolSize", 20);
        // Modifying the definition BEFORE the bean is created
    }
}

// BeanPostProcessor: modifies bean INSTANCES
// Runs AFTER each bean is created
```

---

### Q5: How does Spring AOP work? What's the difference between JDK dynamic proxy and CGLIB?

#### 📖 Deep Explanation

**AOP Proxy Creation**

Spring AOP is implemented via **proxy pattern** — Spring wraps your bean in a proxy that intercepts method calls. The proxy adds cross-cutting behavior (logging, transactions, security) without modifying the target bean.

```
Client → [Proxy] → Target Bean
              ↑
        Interceptor chain
        (advice runs here)
```

**JDK Dynamic Proxy vs CGLIB**

| Aspect | JDK Dynamic Proxy | CGLIB |
|--------|------------------|-------|
| Mechanism | Implements interface via `java.lang.reflect.Proxy` | Generates subclass via bytecode manipulation |
| Requirement | Target must implement an interface | Works on any class (including concrete) |
| Performance | Slightly faster for interface calls | Slightly slower (subclass overhead) |
| Final classes/methods | Works (proxies the interface) | Cannot proxy final classes or methods |
| Spring Boot default | CGLIB (since Spring Boot 2.0) | CGLIB |

```java
// JDK Proxy: only works when bean implements interface
public interface PaymentService {
    void processPayment(Payment payment);
}

@Service
public class PaymentServiceImpl implements PaymentService {
    @Override
    @Transactional // Spring wraps this with a JDK proxy (or CGLIB)
    public void processPayment(Payment payment) { ... }
}

// CGLIB: works even without interface
@Service
public class NotificationService { // no interface
    @Transactional // Spring creates a CGLIB subclass
    public void sendNotification(String message) { ... }
}

// This is why @Transactional on FINAL methods silently fails with CGLIB
@Service
public class UserService {
    @Transactional
    public final void updateUser(User user) { // ❌ CGLIB can't override final
        // Transaction annotation has NO effect here!
        // No exception thrown — silent bug!
    }
}
```

**AOP Advice Types**

```java
@Aspect
@Component
public class SecurityAuditAspect {

    // Before advice: runs before method execution
    @Before("@annotation(RequiresAdmin)")
    public void checkAdminAccess(JoinPoint jp) {
        if (!SecurityContext.isAdmin()) {
            throw new AccessDeniedException("Admin required for: " + jp.getSignature());
        }
    }

    // After returning: runs only if method completes normally
    @AfterReturning(pointcut = "execution(* com.app.service.*.*(..))",
                    returning = "result")
    public void logSuccess(JoinPoint jp, Object result) {
        auditLog.info("SUCCESS: {} returned {}", jp.getSignature(), result);
    }

    // After throwing: runs only if method throws
    @AfterThrowing(pointcut = "execution(* com.app.service.*.*(..))",
                   throwing = "ex")
    public void logFailure(JoinPoint jp, Exception ex) {
        auditLog.error("FAILED: {} threw {}", jp.getSignature(), ex.getMessage());
    }

    // Around: most powerful — wraps entire method execution
    @Around("@annotation(RateLimited)")
    public Object enforceRateLimit(ProceedingJoinPoint pjp) throws Throwable {
        String key = extractKey(pjp);
        if (rateLimiter.isExceeded(key)) {
            throw new RateLimitExceededException("Rate limit exceeded for: " + key);
        }
        long start = System.currentTimeMillis();
        try {
            Object result = pjp.proceed(); // call the actual method
            metricsService.record(key, System.currentTimeMillis() - start);
            return result;
        } catch (Exception e) {
            metricsService.recordError(key);
            throw e;
        }
    }
}
```

#### 🔁 Follow-Up Tricky Questions
- *"If I call a method within the same class that has `@Transactional`, will it work?"*

> **Answer:** No. This is the famous self-invocation problem. `this.method()` calls directly on the target object, bypassing the proxy. The transaction annotation is never processed.

- *"How do you fix self-invocation without extracting to a new class?"*

> **Answer:** Inject a reference to the proxy: `@Autowired ApplicationContext ctx; ctx.getBean(MyService.class).method()` — or use `@Scope("scopedProxy")`, or inject `self` reference via `@Lazy`.

---

### Q6: How does Spring's transaction management work internally?

#### 📖 Deep Explanation

```java
// What @Transactional ACTUALLY generates (simplified):
// Spring creates this proxy code around your method:

public void processOrder(Order order) { // THIS IS THE PROXY
    TransactionStatus status = null;
    try {
        // 1. Get or create transaction from TransactionManager
        status = transactionManager.getTransaction(transactionDefinition);

        // 2. Bind transaction to current thread via ThreadLocal
        TransactionSynchronizationManager.bindResource(dataSource, connectionHolder);

        // 3. Call your actual method
        target.processOrder(order); // YOUR CODE RUNS HERE

        // 4. Commit
        transactionManager.commit(status);

    } catch (RuntimeException ex) {
        // 5. Rollback on RuntimeException (default)
        transactionManager.rollback(status);
        throw ex;
    } finally {
        // 6. Unbind from ThreadLocal
        TransactionSynchronizationManager.unbindResource(dataSource);
    }
}
```

**Transaction Propagation — Complete Guide**

```java
@Service
public class OrderService {

    @Autowired
    private PaymentService paymentService;

    @Autowired
    private AuditService auditService;

    @Transactional // REQUIRED (default): creates TX_1
    public void placeOrder(Order order) {
        orderRepo.save(order); // uses TX_1

        // REQUIRED: PaymentService joins TX_1
        // If payment fails, ENTIRE transaction (including order save) rolls back
        paymentService.charge(order); // uses TX_1

        // REQUIRES_NEW: suspends TX_1, creates TX_2
        // Audit log commits INDEPENDENTLY — survives if outer transaction rolls back
        auditService.log("ORDER_PLACED"); // uses TX_2 — committed immediately

        // NESTED: creates savepoint within TX_1
        // If nested rolls back, only reverts to savepoint — TX_1 continues
        inventoryService.reserveStock(order); // nested within TX_1
    }
}

@Service
public class PaymentService {
    @Transactional(propagation = Propagation.REQUIRED)
    public void charge(Order order) {
        // Joins the existing transaction from OrderService
        // They share the SAME database connection
    }
}

@Service
public class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(String event) {
        // Suspended outer transaction
        // This gets its OWN database connection
        // Commits before outer transaction
    }
}
```

**Isolation Levels in Spring**

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public BigDecimal getAccountBalance(Long accountId) { ... }

@Transactional(isolation = Isolation.SERIALIZABLE)
public void transferFunds(Long from, Long to, BigDecimal amount) { ... }

// Timeout: rollback if transaction takes too long
@Transactional(timeout = 30) // seconds

// Read-only optimization: tells DB to optimize for reads
// Also prevents accidental writes
@Transactional(readOnly = true)
public List<Order> findOrders(OrderFilter filter) { ... }
```

---

## 3. Microservices Architecture

---

### Q7: How would you design a production-grade microservices system for an e-commerce platform?

#### 📖 Deep Explanation

**System Architecture**

```
                          ┌─────────────────────────────────────────────┐
                          │              CLIENT LAYER                    │
                          │    Mobile App | Web App | Third-party APIs   │
                          └────────────────────┬────────────────────────┘
                                               │
                          ┌────────────────────▼────────────────────────┐
                          │           API GATEWAY (Kong/Spring Cloud)    │
                          │  Rate Limiting | Auth | Routing | Load Bal. │
                          └──┬──────────┬────────────┬──────────────┬───┘
                             │          │            │              │
                    ┌────────▼───┐  ┌───▼───┐  ┌────▼─────┐  ┌────▼──────┐
                    │   Order   │  │Payment│  │ Inventory │  │  Customer │
                    │  Service  │  │Service│  │  Service  │  │  Service  │
                    └─────┬─────┘  └───┬───┘  └──────┬───┘  └─────┬─────┘
                          │            │              │             │
                    ┌─────▼─────┐  ┌───▼───┐  ┌──────▼───┐        │
                    │  Orders   │  │Payment│  │ Inventory │  ┌─────▼─────┐
                    │    DB     │  │  DB   │  │    DB     │  │ Customer  │
                    └───────────┘  └───────┘  └──────────┘  │    DB     │
                                                             └───────────┘
                          │            │              │             │
                    ┌─────────────────────────────────────────────────────┐
                    │              MESSAGE BUS (Kafka / RabbitMQ)          │
                    │    order.created | payment.processed | stock.updated │
                    └─────────────────────────────────────────────────────┘

                    ┌─────────────────────────────────────────────────────┐
                    │              INFRASTRUCTURE                          │
                    │   Eureka/Consul (Discovery) | Config Server          │
                    │   Zipkin/Jaeger (Tracing) | ELK (Logging)            │
                    │   Prometheus + Grafana (Metrics)                     │
                    └─────────────────────────────────────────────────────┘
```

**Service Discovery with Eureka**

```yaml
# Eureka Server
spring:
  application:
    name: eureka-server

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
```

```java
// Eureka Client (each microservice)
@SpringBootApplication
@EnableEurekaClient
public class OrderServiceApplication { ... }
```

```yaml
# order-service application.yml
spring:
  application:
    name: order-service

eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10   # heartbeat frequency
    lease-expiration-duration-in-seconds: 30 # eviction threshold
```

**Config Server — Centralized Configuration**

```yaml
# config-server application.yml
spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/company/app-config
          default-label: main
          search-paths: '{application}'  # folder per service
```

```yaml
# order-service bootstrap.yml (loads BEFORE application.yml)
spring:
  application:
    name: order-service
  cloud:
    config:
      uri: http://config-server:8888
      fail-fast: true  # Fail startup if config server unreachable
      retry:
        max-attempts: 5
        initial-interval: 2000
```

```java
// Refresh config without restart
@RestController
@RefreshScope  // Re-creates bean on /actuator/refresh call
public class OrderController {

    @Value("${order.max-retry-attempts:3}")
    private int maxRetryAttempts;
}

// Trigger refresh across all instances via Spring Cloud Bus:
// POST /actuator/busrefresh → broadcasts to all instances via Kafka/RabbitMQ
```

---

### Q8: Explain Circuit Breaker pattern. How does Resilience4j work?

#### 📖 Deep Explanation

**The Problem Without Circuit Breaker**

```
Order Service → Payment Service (DOWN / slow)
Without circuit breaker:
- Order Service threads block waiting for Payment Service
- Thread pool exhausts
- Order Service becomes unresponsive
- Cascading failure — the whole platform goes down
```

**Circuit Breaker States**

```
CLOSED → (failures exceed threshold) → OPEN → (wait-duration passes) → HALF_OPEN
HALF_OPEN → (test calls succeed) → CLOSED
HALF_OPEN → (test calls fail) → OPEN
```

```java
// pom.xml
// spring-cloud-starter-circuitbreaker-resilience4j

// Configuration
@Configuration
public class Resilience4jConfig {

    @Bean
    public Customizer<Resilience4JCircuitBreakerFactory> defaultCustomizer() {
        return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
            .circuitBreakerConfig(CircuitBreakerConfig.custom()
                .slidingWindowType(SlidingWindowType.COUNT_BASED)
                .slidingWindowSize(10)                    // evaluate last 10 calls
                .failureRateThreshold(50)                 // open if 50%+ fail
                .slowCallRateThreshold(50)                // open if 50%+ are slow
                .slowCallDurationThreshold(Duration.ofSeconds(2))
                .waitDurationInOpenState(Duration.ofSeconds(30)) // stay open 30s
                .permittedNumberOfCallsInHalfOpenState(3)        // test with 3 calls
                .recordExceptions(IOException.class, TimeoutException.class)
                .ignoreExceptions(BusinessValidationException.class) // don't count these
                .build())
            .timeLimiterConfig(TimeLimiterConfig.custom()
                .timeoutDuration(Duration.ofSeconds(3))    // per-call timeout
                .build())
            .build());
    }
}

// Service usage
@Service
public class OrderService {

    private final CircuitBreakerFactory cbFactory;
    private final PaymentClient paymentClient;

    public PaymentResult processPayment(Order order) {
        CircuitBreaker cb = cbFactory.create("payment-service");

        return cb.run(
            () -> paymentClient.charge(order),   // primary call
            throwable -> fallbackPayment(order, throwable) // fallback
        );
    }

    private PaymentResult fallbackPayment(Order order, Throwable ex) {
        log.warn("Payment service unavailable, queuing payment: {}", ex.getMessage());
        // Queue to Kafka for async retry
        paymentQueue.enqueue(order);
        return PaymentResult.QUEUED;
    }
}

// Annotation-based approach
@Service
public class InventoryService {

    @CircuitBreaker(name = "inventory", fallbackMethod = "fallbackStock")
    @Retry(name = "inventory", fallbackMethod = "fallbackStock")
    @Bulkhead(name = "inventory")
    public StockLevel checkStock(Long productId) {
        return inventoryClient.getStock(productId);
    }

    // Fallback must have same signature + Throwable parameter
    public StockLevel fallbackStock(Long productId, Throwable ex) {
        log.warn("Returning cached stock for product {}", productId);
        return stockCache.get(productId).orElse(StockLevel.UNKNOWN);
    }
}
```

**Retry Configuration**

```yaml
resilience4j:
  retry:
    instances:
      inventory:
        max-attempts: 3
        wait-duration: 500ms
        retry-exceptions:
          - java.net.ConnectException
          - java.util.concurrent.TimeoutException
        ignore-exceptions:
          - com.app.exception.NotFoundException
        exponential-backoff-multiplier: 2  # 500ms, 1s, 2s
  bulkhead:
    instances:
      inventory:
        max-concurrent-calls: 20
        max-wait-duration: 100ms
```

---

### Q9: How do you implement distributed tracing? Explain correlation IDs and trace context propagation.

#### 📖 Deep Explanation

```java
// Spring Boot 3.x uses Micrometer Tracing (replaces Sleuth)
// pom.xml:
// micrometer-tracing-bridge-otel
// opentelemetry-exporter-zipkin

// Auto-propagates trace context via HTTP headers:
// X-B3-TraceId: abc123def456
// X-B3-SpanId: def456abc789
// X-B3-Sampled: 1

// application.yml
management:
  tracing:
    sampling:
      probability: 1.0  # 100% in dev; 0.1 (10%) in production

spring:
  zipkin:
    base-url: http://zipkin:9411
    sender:
      type: web  # or KAFKA for high-throughput

// All log messages automatically include trace context
// [order-service, traceId=abc123, spanId=def456] Order placed for customer 42
```

```java
// Adding custom spans and events
@Service
public class OrderProcessingService {

    private final Tracer tracer;

    @NewSpan("validate-order")  // Creates a new child span
    public void validateOrder(Order order) {
        Span currentSpan = tracer.currentSpan();
        currentSpan.tag("order.id", order.getId().toString());
        currentSpan.tag("order.amount", order.getTotal().toString());

        // Heavy validation logic
        validateInventory(order);
        validatePaymentMethod(order);

        currentSpan.event("validation-complete");
    }

    // Propagating context across async boundaries (Kafka, ThreadPool)
    @Async
    public CompletableFuture<Void> processAsync(Order order) {
        // Micrometer automatically propagates context to @Async
        // For manual propagation:
        Observation observation = Observation.start("process-order", observationRegistry);
        return CompletableFuture.runAsync(() -> {
            try (Observation.Scope scope = observation.openScope()) {
                processOrder(order);
            }
        });
    }
}
```

**Correlation ID for Cross-Service Requests**

```java
// Filter: extract or generate correlation ID for every request
@Component
@Order(1)
public class CorrelationIdFilter extends OncePerRequestFilter {

    public static final String CORRELATION_ID_HEADER = "X-Correlation-ID";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {
        String correlationId = Optional
            .ofNullable(request.getHeader(CORRELATION_ID_HEADER))
            .orElse(UUID.randomUUID().toString());

        MDC.put("correlationId", correlationId);
        response.addHeader(CORRELATION_ID_HEADER, correlationId);

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove("correlationId");
        }
    }
}

// Feign client interceptor: propagate to downstream services
@Configuration
public class FeignConfig {

    @Bean
    public RequestInterceptor correlationIdInterceptor() {
        return requestTemplate -> {
            String correlationId = MDC.get("correlationId");
            if (correlationId != null) {
                requestTemplate.header(CorrelationIdFilter.CORRELATION_ID_HEADER, correlationId);
            }
        };
    }
}

// Logback config: include in every log line
// logback-spring.xml:
// <pattern>%d{HH:mm:ss} [%thread] [traceId=%X{traceId}] [corrId=%X{correlationId}] %-5level %logger - %msg%n</pattern>
```

---

### Q10: How would you handle a distributed transaction across multiple microservices? (Saga Pattern)

#### 📖 Deep Explanation

**The Problem: Two-Phase Commit Doesn't Work in Microservices**

```
Traditional 2PC:
Coordinator → asks all services to PREPARE
All services → PREPARE (hold locks!)
Coordinator → asks all services to COMMIT or ROLLBACK

Problems:
- Coordinator becomes single point of failure
- Services hold locks while waiting = reduced throughput
- Network partitions cause indefinite lock holding
```

**Saga Pattern — Choreography vs Orchestration**

```java
// CHOREOGRAPHY: Services react to events (event-driven)
// Each service publishes events, others subscribe and react

// Order Service
@Service
public class OrderService {

    @Transactional
    public void createOrder(CreateOrderCommand cmd) {
        Order order = new Order(cmd, OrderStatus.PENDING);
        orderRepo.save(order);

        // Publish event — Payment Service will react
        eventPublisher.publish(new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getTotalAmount()
        ));
    }

    // Compensating transaction: called if payment fails
    @KafkaListener(topics = "payment.failed")
    public void onPaymentFailed(PaymentFailedEvent event) {
        Order order = orderRepo.findById(event.getOrderId()).orElseThrow();
        order.setStatus(OrderStatus.CANCELLED);
        orderRepo.save(order);
        // Publish OrderCancelledEvent for inventory to release stock
        eventPublisher.publish(new OrderCancelledEvent(order.getId()));
    }
}

// Payment Service reacts to OrderCreatedEvent
@Service
public class PaymentService {

    @KafkaListener(topics = "order.created")
    @Transactional
    public void onOrderCreated(OrderCreatedEvent event) {
        try {
            Payment payment = processPayment(event);
            eventPublisher.publish(new PaymentSucceededEvent(event.getOrderId(), payment.getId()));
        } catch (InsufficientFundsException e) {
            eventPublisher.publish(new PaymentFailedEvent(event.getOrderId(), e.getMessage()));
        }
    }
}
```

```java
// ORCHESTRATION: Central saga orchestrator coordinates the flow
// Clearer flow, easier to monitor, single place for saga logic

@Service
public class OrderSagaOrchestrator {

    @Transactional
    public void execute(CreateOrderCommand cmd) {
        SagaState saga = sagaRepo.create(cmd);

        try {
            // Step 1: Reserve inventory
            saga.advance(SagaStep.RESERVE_INVENTORY);
            inventoryClient.reserve(cmd.getOrderId(), cmd.getItems());

            // Step 2: Process payment
            saga.advance(SagaStep.PROCESS_PAYMENT);
            paymentClient.charge(cmd.getOrderId(), cmd.getTotalAmount());

            // Step 3: Confirm order
            saga.advance(SagaStep.CONFIRM_ORDER);
            orderClient.confirm(cmd.getOrderId());

            saga.complete();

        } catch (Exception e) {
            // Compensate in reverse order
            compensate(saga, e);
        }
    }

    private void compensate(SagaState saga, Exception cause) {
        log.error("Saga {} failed at step {}, compensating", saga.getId(), saga.getCurrentStep());

        switch (saga.getCurrentStep()) {
            case CONFIRM_ORDER:
                orderClient.cancel(saga.getOrderId());
                // fall through
            case PROCESS_PAYMENT:
                paymentClient.refund(saga.getOrderId());
                // fall through
            case RESERVE_INVENTORY:
                inventoryClient.releaseReservation(saga.getOrderId());
                break;
        }

        saga.fail(cause.getMessage());
        sagaRepo.save(saga);
    }
}
```

**Outbox Pattern — Ensuring Event Delivery**

```java
// Problem: Save to DB + publish to Kafka is not atomic
// If Kafka publish fails after DB commit, event is lost

// Solution: Transactional Outbox
@Transactional
public void createOrder(Order order) {
    orderRepo.save(order); // save order

    // Save event to outbox table IN THE SAME TRANSACTION
    OutboxEvent outboxEvent = new OutboxEvent(
        "order.created",
        objectMapper.writeValueAsString(new OrderCreatedEvent(order))
    );
    outboxRepo.save(outboxEvent);
    // If either save fails, both roll back
}

// Separate process (or CDC like Debezium) reads outbox and publishes to Kafka
@Scheduled(fixedDelay = 1000)
@Transactional
public void publishOutboxEvents() {
    List<OutboxEvent> pending = outboxRepo.findByStatus(OutboxStatus.PENDING);
    for (OutboxEvent event : pending) {
        try {
            kafkaTemplate.send(event.getTopic(), event.getPayload()).get();
            event.setStatus(OutboxStatus.PUBLISHED);
            outboxRepo.save(event);
        } catch (Exception e) {
            event.incrementRetryCount();
            outboxRepo.save(event);
        }
    }
}
```

---

## 4. Spring Data & Hibernate

---

### Q11: What is the N+1 problem and what are all the ways to solve it?

> *(See the companion SQL/Hibernate guide for full detail)*

```java
// Problem: fetching 100 customers → 101 queries
List<Customer> customers = customerRepo.findAll(); // 1 query
customers.forEach(c -> c.getOrders().size());     // 100 queries!

// Solution 1: JOIN FETCH
@Query("SELECT DISTINCT c FROM Customer c LEFT JOIN FETCH c.orders")
List<Customer> findAllWithOrders();

// Solution 2: @EntityGraph
@EntityGraph(attributePaths = {"orders", "orders.items"})
List<Customer> findAll();

// Solution 3: Batch fetching
@BatchSize(size = 25)
@OneToMany(mappedBy = "customer")
private List<Order> orders;

// Solution 4: DTO Projection (best for read-only)
@Query("SELECT new com.app.dto.CustomerSummary(c.id, c.name, COUNT(o)) " +
       "FROM Customer c LEFT JOIN c.orders o GROUP BY c.id, c.name")
List<CustomerSummary> findCustomerSummaries();

// Detection: Enable Hibernate Statistics
spring.jpa.properties.hibernate.generate_statistics=true
// Or: datasource-proxy to count queries per request in tests
```

---

### Q12: Explain optimistic vs pessimistic locking in JPA. When do you use each?

#### 📖 Deep Explanation

```java
// OPTIMISTIC LOCKING: No DB locks. Detects conflicts at commit time.
// Best for: low-contention scenarios, read-heavy systems
@Entity
public class Product {
    @Id
    private Long id;

    private int stockQuantity;

    @Version  // Hibernate manages this column automatically
    private Long version;
    // version starts at 0, increments on every UPDATE
}

// How it works:
// T1: reads product (version=5, stock=10)
// T2: reads product (version=5, stock=10)
// T1: UPDATE products SET stock=9, version=6 WHERE id=1 AND version=5 → SUCCESS
// T2: UPDATE products SET stock=9, version=6 WHERE id=1 AND version=5 → 0 rows updated!
// T2: Hibernate sees 0 rows → throws OptimisticLockException

@Transactional
public void purchaseProduct(Long productId, int quantity) {
    try {
        Product product = productRepo.findById(productId).orElseThrow();
        product.setStockQuantity(product.getStockQuantity() - quantity);
        productRepo.save(product);
    } catch (OptimisticLockException | ObjectOptimisticLockingFailureException e) {
        // Retry logic
        throw new ConcurrentModificationException("Product was modified, please retry");
    }
}
```

```java
// PESSIMISTIC LOCKING: DB-level row lock. Blocks other transactions.
// Best for: high-contention, financial transactions, inventory deduction

// Pessimistic Write: SELECT ... FOR UPDATE
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id = :id")
Optional<Product> findByIdWithLock(@Param("id") Long id);

@Transactional
public void reserveTicket(Long ticketId, Long userId) {
    // Row is LOCKED — other transactions block here until we release
    Ticket ticket = ticketRepo.findByIdWithLock(ticketId)
        .orElseThrow(() -> new NotFoundException("Ticket not found"));

    if (ticket.getStatus() != TicketStatus.AVAILABLE) {
        throw new TicketAlreadyReservedException();
    }

    ticket.setStatus(TicketStatus.RESERVED);
    ticket.setReservedBy(userId);
    ticketRepo.save(ticket);
    // Lock released when transaction commits
}

// Pessimistic Read: SELECT ... FOR SHARE (allows concurrent reads)
@Lock(LockModeType.PESSIMISTIC_READ)
Optional<Product> findByIdForRead(@Param("id") Long id);
```

**Decision Guide**

| Scenario | Lock Type | Reason |
|---------|-----------|--------|
| User profile update | Optimistic | Low contention, conflicts rare |
| Inventory deduction | Pessimistic | Race conditions are costly |
| Bank account transfer | Pessimistic | Cannot afford conflicts |
| Distributed config update | Optimistic | Rare updates, easy to retry |
| Ticket booking (flash sale) | Pessimistic | High contention, must prevent oversell |

---

## 5. REST API Design & Best Practices

---

### Q13: How do you design idempotent APIs? Why does it matter?

#### 📖 Deep Explanation

**Idempotency**: Making the same request N times has the same effect as making it once.

```
HTTP Methods:
GET    → Naturally idempotent (read-only)
PUT    → Naturally idempotent (full replace)
DELETE → Naturally idempotent (delete once = delete N times)
POST   → NOT idempotent by default → must be made idempotent manually
PATCH  → Not inherently idempotent → depends on implementation
```

```java
// Problem: Client sends POST /orders, network times out, client retries
// Result: Two orders created instead of one!

// Solution: Idempotency Key
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @PostMapping
    public ResponseEntity<OrderDTO> createOrder(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @Valid @RequestBody CreateOrderRequest request) {

        // Check if we've seen this key before
        Optional<OrderDTO> cachedResult = idempotencyStore.get(idempotencyKey);
        if (cachedResult.isPresent()) {
            // Return the original response — same status, same body
            return ResponseEntity.status(HttpStatus.CREATED)
                                 .body(cachedResult.get());
        }

        // Process the order
        OrderDTO result = orderService.createOrder(request);

        // Cache the result with TTL (e.g., 24 hours)
        idempotencyStore.put(idempotencyKey, result, Duration.ofHours(24));

        return ResponseEntity.status(HttpStatus.CREATED).body(result);
    }
}

// Idempotency store backed by Redis
@Service
public class IdempotencyStore {
    private final RedisTemplate<String, String> redis;
    private final ObjectMapper objectMapper;

    public <T> Optional<T> get(String key, Class<T> type) {
        String value = redis.opsForValue().get("idempotency:" + key);
        if (value == null) return Optional.empty();
        return Optional.of(objectMapper.readValue(value, type));
    }

    public void put(String key, Object value, Duration ttl) {
        redis.opsForValue().set(
            "idempotency:" + key,
            objectMapper.writeValueAsString(value),
            ttl
        );
    }
}
```

**Idempotency for Stripe-style Payments**

```java
// Financial operations — absolutely must be idempotent
@Transactional
public PaymentResult processPayment(PaymentRequest request, String idempotencyKey) {

    // Check for existing payment with this key
    Optional<Payment> existing = paymentRepo.findByIdempotencyKey(idempotencyKey);
    if (existing.isPresent()) {
        log.info("Duplicate payment request detected, returning original result");
        return PaymentResult.from(existing.get());
    }

    Payment payment = new Payment();
    payment.setIdempotencyKey(idempotencyKey); // UNIQUE constraint in DB
    payment.setAmount(request.getAmount());

    try {
        // Process with payment gateway
        GatewayResult result = gateway.charge(request);
        payment.setStatus(PaymentStatus.SUCCEEDED);
        payment.setGatewayReference(result.getReference());
    } catch (GatewayException e) {
        payment.setStatus(PaymentStatus.FAILED);
        payment.setFailureReason(e.getMessage());
    }

    paymentRepo.save(payment);
    return PaymentResult.from(payment);
    // If two identical requests arrive simultaneously:
    // One will get a DB unique constraint violation → retry → returns cached result
}
```

---

### Q14: How do you design proper error handling for a Spring Boot REST API?

#### 📖 Deep Explanation

```java
// Global exception handler — ALL exceptions flow here
@RestControllerAdvice
public class GlobalExceptionHandler {

    // Business/domain exceptions
    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ApiError> handleOrderNotFound(
        OrderNotFoundException ex, HttpServletRequest request) {

        ApiError error = ApiError.builder()
            .timestamp(Instant.now())
            .status(HttpStatus.NOT_FOUND.value())
            .error("Order Not Found")
            .message(ex.getMessage())
            .path(request.getRequestURI())
            .traceId(MDC.get("traceId"))
            .build();

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    // Validation errors (@Valid failures)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiValidationError> handleValidationErrors(
        MethodArgumentNotValidException ex) {

        List<FieldError> fieldErrors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(fe -> new FieldError(fe.getField(), fe.getDefaultMessage(), fe.getRejectedValue()))
            .collect(Collectors.toList());

        ApiValidationError error = ApiValidationError.builder()
            .timestamp(Instant.now())
            .status(400)
            .error("Validation Failed")
            .errors(fieldErrors)
            .build();

        return ResponseEntity.badRequest().body(error);
    }

    // Optimistic lock conflicts
    @ExceptionHandler(ObjectOptimisticLockingFailureException.class)
    public ResponseEntity<ApiError> handleConcurrentModification(
        ObjectOptimisticLockingFailureException ex) {

        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(ApiError.of(409, "Resource was modified by another request. Please retry."));
    }

    // Catch-all: never expose internals
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleUnexpected(Exception ex, HttpServletRequest request) {
        // Log the full stack trace internally
        log.error("Unexpected error at {}: {}", request.getRequestURI(), ex.getMessage(), ex);

        // Return safe, generic message to client — NEVER leak stack traces
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ApiError.builder()
                .status(500)
                .error("Internal Server Error")
                .message("An unexpected error occurred. Please contact support.")
                .traceId(MDC.get("traceId")) // Client can reference trace for support
                .build());
    }
}

// Consistent error response structure
@Builder
public class ApiError {
    private Instant timestamp;
    private int status;
    private String error;
    private String message;
    private String path;
    private String traceId;  // For support reference
}
```

**API Versioning Strategies**

```java
// Strategy 1: URI versioning (most common)
@RequestMapping("/api/v1/orders")
// /api/v1/orders — Version 1
// /api/v2/orders — Version 2 (breaking changes)

// Strategy 2: Header versioning (cleaner URIs)
@GetMapping(value = "/api/orders",
            headers = "X-API-Version=2")
public ResponseEntity<OrderV2DTO> getOrdersV2() { ... }

// Strategy 3: Accept header versioning
@GetMapping(value = "/api/orders",
            produces = "application/vnd.company.orders-v2+json")
public ResponseEntity<OrderV2DTO> getOrdersV2() { ... }
```

**Pagination Best Practices**

```java
@GetMapping("/api/orders")
public ResponseEntity<Page<OrderDTO>> getOrders(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size,
    @RequestParam(defaultValue = "createdAt,desc") String sort,
    @RequestParam(required = false) OrderStatus status,
    @RequestParam(required = false) @DateTimeFormat(iso = ISO.DATE) LocalDate from,
    @RequestParam(required = false) @DateTimeFormat(iso = ISO.DATE) LocalDate to) {

    // Validate page size to prevent abuse
    size = Math.min(size, 100); // Never allow > 100 per page

    Sort sortObj = Sort.by(parseSortParam(sort));
    Pageable pageable = PageRequest.of(page, size, sortObj);
    OrderFilter filter = new OrderFilter(status, from, to);

    Page<OrderDTO> result = orderService.findOrders(filter, pageable);

    // Return with pagination metadata in headers
    HttpHeaders headers = new HttpHeaders();
    headers.add("X-Total-Count", String.valueOf(result.getTotalElements()));
    headers.add("X-Total-Pages", String.valueOf(result.getTotalPages()));

    return ResponseEntity.ok().headers(headers).body(result);
}

// Response includes cursor for next page
{
  "content": [...],
  "page": { "size": 20, "number": 0, "totalElements": 847, "totalPages": 43 },
  "_links": {
    "self": { "href": "/api/orders?page=0&size=20" },
    "next": { "href": "/api/orders?page=1&size=20" },
    "last": { "href": "/api/orders?page=42&size=20" }
  }
}
```

---

## 6. Security

---

### Q15: How does Spring Security's filter chain work? Explain the security context.

#### 📖 Deep Explanation

```
HTTP Request
    ↓
SecurityFilterChain (ordered list of filters)
    ↓
1. SecurityContextPersistenceFilter — restores SecurityContext from session
2. UsernamePasswordAuthenticationFilter — handles login form
3. BearerTokenAuthenticationFilter (JWT) — handles Bearer tokens
4. BasicAuthenticationFilter — handles Basic auth
5. ExceptionTranslationFilter — converts security exceptions to HTTP responses
6. FilterSecurityInterceptor — final authorization check
    ↓
Controller / Handler
```

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // enables @PreAuthorize, @PostAuthorize
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            // Disable CSRF for stateless REST APIs (JWT-based)
            .csrf(csrf -> csrf.disable())

            // Stateless session (no HttpSession)
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            // Authorization rules — ORDER MATTERS (most specific first)
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/orders/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )

            // Add JWT filter BEFORE UsernamePasswordAuthenticationFilter
            .addFilterBefore(jwtAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class)

            // Custom exception handling
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(customAuthenticationEntryPoint())
                .accessDeniedHandler(customAccessDeniedHandler())
            )
            .build();
    }
}
```

**JWT Authentication Filter**

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {
        // 1. Extract token from Authorization header
        String authHeader = request.getHeader("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            chain.doFilter(request, response); // no token = continue, let auth rules decide
            return;
        }

        String token = authHeader.substring(7);

        try {
            // 2. Validate and parse JWT
            Claims claims = jwtService.validateToken(token);

            // 3. Build authentication object
            String userId = claims.getSubject();
            List<String> roles = claims.get("roles", List.class);

            List<GrantedAuthority> authorities = roles.stream()
                .map(SimpleGrantedAuthority::new)
                .collect(Collectors.toList());

            UsernamePasswordAuthenticationToken auth =
                new UsernamePasswordAuthenticationToken(userId, null, authorities);
            auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));

            // 4. Store in SecurityContext (ThreadLocal — request-scoped)
            SecurityContextHolder.getContext().setAuthentication(auth);

        } catch (JwtExpiredException e) {
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            response.getWriter().write("{\"error\": \"Token expired\"}");
            return; // Don't continue chain — request is rejected
        } catch (JwtException e) {
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            response.getWriter().write("{\"error\": \"Invalid token\"}");
            return;
        }

        chain.doFilter(request, response);
    }
}
```

**OAuth2 + Keycloak Integration**

```yaml
# application.yml — Resource Server configuration
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://keycloak:8080/realms/myrealm
          # Spring auto-fetches JWKS from: {issuer-uri}/.well-known/openid-configuration
```

```java
@Configuration
@EnableWebSecurity
public class OAuth2SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .oauth2ResourceServer(oauth2 ->
                oauth2.jwt(jwt ->
                    jwt.jwtAuthenticationConverter(keycloakJwtConverter())
                )
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .build();
    }

    // Keycloak stores roles in a nested claim — extract them
    @Bean
    public JwtAuthenticationConverter keycloakJwtConverter() {
        JwtGrantedAuthoritiesConverter converter = new JwtGrantedAuthoritiesConverter();
        converter.setAuthoritiesClaimName("realm_access.roles");
        converter.setAuthorityPrefix("ROLE_");

        JwtAuthenticationConverter authConverter = new JwtAuthenticationConverter();
        authConverter.setJwtGrantedAuthoritiesConverter(converter);
        return authConverter;
    }
}

// Method-level security
@RestController
public class AdminController {

    @GetMapping("/api/admin/users")
    @PreAuthorize("hasRole('ADMIN') and #oauth2.hasScope('user:read')")
    public List<UserDTO> getUsers() { ... }

    @PostMapping("/api/orders/{orderId}/cancel")
    @PreAuthorize("hasRole('ADMIN') or @orderSecurityService.isOwner(#orderId, authentication)")
    public void cancelOrder(@PathVariable Long orderId) { ... }
}

// Custom security service for complex rules
@Service("orderSecurityService")
public class OrderSecurityService {
    public boolean isOwner(Long orderId, Authentication auth) {
        Order order = orderRepo.findById(orderId).orElseThrow();
        return order.getCustomerId().equals(auth.getName());
    }
}
```

---

## 7. Performance & Scalability

---

### Q16: How do you implement caching in Spring Boot? Explain Redis integration and cache strategies.

#### 📖 Deep Explanation

```java
// application.yml
spring:
  cache:
    type: redis
  data:
    redis:
      host: redis-cluster.internal
      port: 6379
      password: ${REDIS_PASSWORD}
      lettuce:
        pool:
          max-active: 20
          max-idle: 10
          min-idle: 5

// Cache configuration
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))          // Default TTL
            .disableCachingNullValues()                  // Don't cache null results
            .serializeKeysWith(SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(SerializationPair.fromSerializer(
                new GenericJackson2JsonRedisSerializer()));

        // Per-cache TTL configuration
        Map<String, RedisCacheConfiguration> cacheConfigs = Map.of(
            "products",        defaultConfig.entryTtl(Duration.ofHours(2)),
            "user-sessions",   defaultConfig.entryTtl(Duration.ofMinutes(30)),
            "exchange-rates",  defaultConfig.entryTtl(Duration.ofMinutes(5)),
            "reference-data",  defaultConfig.entryTtl(Duration.ofDays(1))
        );

        return RedisCacheManager.builder(factory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigs)
            .build();
    }
}

// Cache annotations
@Service
public class ProductService {

    // @Cacheable: check cache first, call method only on cache miss
    @Cacheable(value = "products", key = "#productId",
               condition = "#productId != null",     // only cache if key is valid
               unless = "#result == null")            // don't cache null results
    public ProductDTO getProduct(Long productId) {
        return productRepo.findById(productId)
            .map(productMapper::toDTO)
            .orElseThrow(() -> new ProductNotFoundException(productId));
    }

    // @CachePut: always update cache (used on write operations)
    @CachePut(value = "products", key = "#result.id")
    public ProductDTO updateProduct(Long id, UpdateProductRequest request) {
        Product product = productRepo.findById(id).orElseThrow();
        productMapper.update(product, request);
        Product saved = productRepo.save(product);
        return productMapper.toDTO(saved);
    }

    // @CacheEvict: remove from cache (used on delete)
    @CacheEvict(value = "products", key = "#productId")
    public void deleteProduct(Long productId) {
        productRepo.deleteById(productId);
    }

    // Evict multiple caches at once
    @Caching(evict = {
        @CacheEvict(value = "products", key = "#categoryId"),
        @CacheEvict(value = "product-count", allEntries = true)
    })
    public void addProductToCategory(Long categoryId, Long productId) { ... }
}
```

**Cache Aside vs Write-Through vs Write-Behind**

```java
// Cache-Aside (most common in Spring): application manages cache manually
public ProductDTO getProduct(Long id) {
    // 1. Check cache
    // 2. If miss, load from DB
    // 3. Populate cache
    // 4. Return result
}

// Write-Through: write to cache AND DB synchronously
@CachePut(value = "products", key = "#product.id")
@Transactional
public ProductDTO saveProduct(Product product) {
    Product saved = productRepo.save(product);
    return productMapper.toDTO(saved); // also stored in cache
}

// Write-Behind: write to cache, async write to DB (eventual consistency)
// Used in: leaderboards, counters, views count
@Async
public void incrementViewCount(Long productId) {
    String key = "views:" + productId;
    redisTemplate.opsForValue().increment(key);
    // Flush to DB via scheduled job every minute
}
```

**Cache Stampede Prevention**

```java
// Problem: Cache expires → 1000 concurrent requests hit DB simultaneously
// Solution: Probabilistic early expiration or distributed lock

@Service
public class CacheService {

    private final RedisTemplate<String, String> redis;
    private final RLock lock; // Redisson distributed lock

    public ProductDTO getProductWithStampedeProtection(Long productId) {
        String cacheKey = "product:" + productId;
        String cached = redis.opsForValue().get(cacheKey);

        if (cached != null) return deserialize(cached);

        // Use distributed lock to prevent stampede
        String lockKey = "lock:product:" + productId;
        RLock lock = redisson.getLock(lockKey);

        try {
            if (lock.tryLock(100, 5000, TimeUnit.MILLISECONDS)) {
                // Double-check after acquiring lock
                cached = redis.opsForValue().get(cacheKey);
                if (cached != null) return deserialize(cached);

                ProductDTO result = loadFromDatabase(productId);
                redis.opsForValue().set(cacheKey, serialize(result), Duration.ofHours(2));
                return result;
            }
        } finally {
            if (lock.isHeldByCurrentThread()) lock.unlock();
        }

        return loadFromDatabase(productId); // fallback
    }
}
```

---

### Q17: How do you implement async processing in Spring Boot?

#### 📖 Deep Explanation

```java
// Configuration: customize thread pools
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);           // Always-active threads
        executor.setMaxPoolSize(20);           // Max threads under load
        executor.setQueueCapacity(200);        // Queue before creating new threads
        executor.setThreadNamePrefix("async-"); // For debugging/logging
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        // CallerRunsPolicy: if pool is full, calling thread runs the task (backpressure)
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(30);
        executor.initialize();
        return executor;
    }

    // Separate pool for email/notification tasks
    @Bean("emailExecutor")
    public Executor emailExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setThreadNamePrefix("email-");
        executor.initialize();
        return executor;
    }
}

@Service
public class OrderNotificationService {

    // @Async must be on a public method, called from OUTSIDE the class (proxy!)
    @Async("emailExecutor")  // uses specific pool
    public CompletableFuture<Void> sendOrderConfirmation(Order order) {
        try {
            emailService.send(
                order.getCustomerEmail(),
                "Order Confirmed",
                buildEmailTemplate(order)
            );
            return CompletableFuture.completedFuture(null);
        } catch (EmailException e) {
            return CompletableFuture.failedFuture(e);
        }
    }

    // Combining multiple async operations
    public OrderSummary buildFullOrderSummary(Long orderId) {
        CompletableFuture<Order> orderFuture =
            CompletableFuture.supplyAsync(() -> orderRepo.findById(orderId).orElseThrow());

        CompletableFuture<Customer> customerFuture =
            orderFuture.thenCompose(order ->
                CompletableFuture.supplyAsync(() -> customerRepo.findById(order.getCustomerId()).orElseThrow()));

        CompletableFuture<List<OrderItem>> itemsFuture =
            CompletableFuture.supplyAsync(() -> itemRepo.findByOrderId(orderId));

        // Wait for all three to complete
        return CompletableFuture.allOf(orderFuture, customerFuture, itemsFuture)
            .thenApply(v -> new OrderSummary(
                orderFuture.join(),
                customerFuture.join(),
                itemsFuture.join()
            ))
            .join();
    }
}
```

---

## 8. Testing

---

### Q18: What is your testing strategy for a Spring Boot microservice? What do you test vs not test?

#### 📖 Deep Explanation

```
Testing Pyramid for Microservices:

         /E2E\          <- Contract tests (Pact), minimal UI smoke tests
        /------\
       / Integr \       <- @SpringBootTest, Testcontainers, real DBs
      /----------\
     / Unit Tests \     <- JUnit 5, Mockito — fast, isolated, bulk of tests
    /--------------\
```

**Unit Tests — Isolate Business Logic**

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepo;

    @Mock
    private PaymentService paymentService;

    @Mock
    private InventoryService inventoryService;

    @InjectMocks
    private OrderService orderService;

    @Test
    void shouldCreateOrderWhenInventoryAvailable() {
        // Given
        CreateOrderRequest request = new CreateOrderRequest(
            CUSTOMER_ID, List.of(new OrderItem(PRODUCT_ID, 2))
        );
        when(inventoryService.checkAvailability(PRODUCT_ID, 2)).thenReturn(true);
        when(orderRepo.save(any(Order.class))).thenAnswer(inv -> {
            Order order = inv.getArgument(0);
            order.setId(123L);
            return order;
        });

        // When
        OrderDTO result = orderService.createOrder(request);

        // Then
        assertThat(result.getId()).isEqualTo(123L);
        assertThat(result.getStatus()).isEqualTo(OrderStatus.PENDING);

        verify(inventoryService).checkAvailability(PRODUCT_ID, 2);
        verify(orderRepo).save(any(Order.class));
        verify(paymentService, never()).charge(any()); // Payment not triggered yet
    }

    @Test
    void shouldRejectOrderWhenInventoryInsufficient() {
        when(inventoryService.checkAvailability(any(), anyInt())).thenReturn(false);

        assertThatThrownBy(() -> orderService.createOrder(request))
            .isInstanceOf(InsufficientInventoryException.class)
            .hasMessageContaining("insufficient inventory");

        verify(orderRepo, never()).save(any()); // Order should NOT be saved
    }
}
```

**Integration Tests with Testcontainers**

```java
@SpringBootTest
@Testcontainers
@ActiveProfiles("integration-test")
class OrderRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("orders_test")
        .withUsername("test")
        .withPassword("test");

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7")
        .withExposedPorts(6379);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", () -> redis.getMappedPort(6379));
    }

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void shouldFindOrdersWithComplexFilter() {
        // Given
        Order order1 = createOrder(Status.PENDING, LocalDate.now().minusDays(2));
        Order order2 = createOrder(Status.SHIPPED, LocalDate.now().minusDays(1));
        orderRepository.saveAll(List.of(order1, order2));

        // When
        List<Order> pendingOrders = orderRepository.findByStatusAndCreatedAfter(
            Status.PENDING, LocalDate.now().minusDays(3)
        );

        // Then
        assertThat(pendingOrders).hasSize(1);
        assertThat(pendingOrders.get(0).getStatus()).isEqualTo(Status.PENDING);
    }
}
```

**Contract Testing with Spring Cloud Contract**

```groovy
// Contract definition (producer side)
// src/test/resources/contracts/order/shouldCreateOrder.groovy
Contract.make {
    description "should create order and return 201"
    request {
        method POST()
        url '/api/orders'
        headers { contentType(applicationJson()) }
        body([
            customerId: 42,
            items: [[productId: 1, quantity: 2]]
        ])
    }
    response {
        status 201
        headers { contentType(applicationJson()) }
        body([
            id: $(anyPositiveInt()),
            status: "PENDING",
            customerId: 42
        ])
    }
}
```

**What to Test vs NOT Test**

| Test This | Skip This |
|-----------|-----------|
| Business logic / domain rules | Getter/setter methods |
| Repository queries (with Testcontainers) | Framework behavior (@Autowired works) |
| Controller request/response mapping | Lombok-generated code |
| Security rules (@PreAuthorize) | Spring Boot's auto-configuration |
| Event handling / Kafka consumers | Third-party library internals |
| Error handling / edge cases | Trivial one-liners |
| Service integration flows | Configuration classes with no logic |

---

## 9. DevOps & Production

---

### Q19: How do you containerize a Spring Boot application for production Kubernetes deployment?

#### 📖 Deep Explanation

```dockerfile
# Production-grade Dockerfile with multi-stage build
# Stage 1: Build
FROM eclipse-temurin:21-jdk-jammy AS builder
WORKDIR /app

# Copy dependency files first (better Docker layer caching)
COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .
RUN ./mvnw dependency:go-offline -B  # cache dependencies

COPY src src
RUN ./mvnw package -DskipTests -B

# Extract layers for optimized Docker image
RUN java -Djarmode=layertools -jar target/*.jar extract

# Stage 2: Runtime (minimal image)
FROM eclipse-temurin:21-jre-jammy
WORKDIR /app

# Security: run as non-root user
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser

# Copy layers from builder (most-changed layers last)
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./

USER appuser

# JVM flags for containers
ENV JAVA_OPTS="-XX:+UseContainerSupport \
               -XX:MaxRAMPercentage=75.0 \
               -XX:+HeapDumpOnOutOfMemoryError \
               -XX:HeapDumpPath=/tmp/heapdump.hprof \
               -Djava.security.egd=file:/dev/./urandom"

EXPOSE 8080

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.JarLauncher"]
```

```yaml
# Kubernetes deployment with production best practices
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # 1 extra pod during update
      maxUnavailable: 0   # Never go below 3 pods
  selector:
    matchLabels:
      app: order-service
  template:
    spec:
      containers:
        - name: order-service
          image: company/order-service:1.2.3  # Always use specific tags, not :latest
          ports:
            - containerPort: 8080

          # Resource limits prevent one pod from starving others
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"

          # Liveness: restart pod if app is stuck/deadlocked
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 15
            failureThreshold: 3

          # Readiness: only send traffic when app is ready
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3

          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "production"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:      # Never put secrets in env vars directly
                  name: db-secrets
                  key: password
```

```java
// Spring Boot Actuator health indicators for Kubernetes probes
@Component
public class OrderServiceHealthIndicator implements HealthIndicator {

    @Autowired
    private OrderRepository orderRepository;

    @Override
    public Health health() {
        try {
            // Quick DB connectivity check
            orderRepository.count();
            return Health.up()
                .withDetail("database", "connected")
                .withDetail("service", "order-service")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withException(e)
                .withDetail("service", "order-service")
                .build();
        }
    }
}

// application.yml — expose probes
management:
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
```

---

### Q20: How do you implement proper observability in a Spring Boot application?

#### 📖 Deep Explanation

**Structured Logging (JSON format)**

```xml
<!-- logback-spring.xml -->
<configuration>
  <springProfile name="production">
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
      <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <includeMdcKeyName>traceId</includeMdcKeyName>
        <includeMdcKeyName>spanId</includeMdcKeyName>
        <includeMdcKeyName>correlationId</includeMdcKeyName>
        <includeMdcKeyName>userId</includeMdcKeyName>
      </encoder>
    </appender>
    <root level="INFO">
      <appender-ref ref="JSON"/>
    </root>
  </springProfile>
</configuration>
```

```json
// Structured log output — easily parsed by Elasticsearch/Loki
{
  "timestamp": "2024-01-15T10:30:45.123Z",
  "level": "INFO",
  "logger": "com.app.service.OrderService",
  "message": "Order created successfully",
  "traceId": "abc123def456",
  "spanId": "def456",
  "correlationId": "req-789",
  "userId": "user-42",
  "orderId": "order-99",
  "service": "order-service",
  "environment": "production"
}
```

**Prometheus Metrics**

```java
// pom.xml: micrometer-registry-prometheus

@Service
public class OrderService {

    private final Counter ordersCreated;
    private final Counter ordersFailed;
    private final Timer orderProcessingTime;
    private final Gauge pendingOrdersGauge;

    public OrderService(MeterRegistry registry, OrderRepository orderRepo) {
        this.ordersCreated = Counter.builder("orders.created")
            .description("Total orders created")
            .tag("service", "order-service")
            .register(registry);

        this.ordersFailed = Counter.builder("orders.failed")
            .description("Total orders that failed")
            .register(registry);

        this.orderProcessingTime = Timer.builder("orders.processing.duration")
            .description("Time to process an order")
            .register(registry);

        // Gauge: reports current value each time it's scraped
        this.pendingOrdersGauge = Gauge.builder("orders.pending.count",
            () -> (double) orderRepo.countByStatus(OrderStatus.PENDING))
            .register(registry);
    }

    public OrderDTO createOrder(CreateOrderRequest request) {
        return orderProcessingTime.record(() -> {
            try {
                OrderDTO result = doCreateOrder(request);
                ordersCreated.increment(1, Tags.of("status", "success"));
                return result;
            } catch (Exception e) {
                ordersFailed.increment(1, Tags.of("reason", e.getClass().getSimpleName()));
                throw e;
            }
        });
    }
}
```

```yaml
# Grafana Dashboard Alert Example (in code-as-config)
# Alert: order processing time > 2 seconds P99
# Alert: error rate > 5% in 5 minutes
# Alert: pending orders > 1000 (processing backlog)
```

---

## 10. Coding & Design Problems

---

### Problem 1: Design a Rate Limiter (asked at senior interviews)

```java
// Requirement: limit each user to 100 API calls per minute
// Redis-backed, distributed (works across multiple instances)

@Aspect
@Component
public class RateLimiterAspect {

    private final RedisTemplate<String, String> redis;

    @Around("@annotation(RateLimited)")
    public Object enforceRateLimit(ProceedingJoinPoint pjp) throws Throwable {
        String userId = getCurrentUserId();
        RateLimited annotation = getAnnotation(pjp, RateLimited.class);

        if (isRateLimitExceeded(userId, annotation.limit(), annotation.window())) {
            throw new RateLimitExceededException(
                "Rate limit exceeded. Try again in " + annotation.window() + " seconds."
            );
        }

        return pjp.proceed();
    }

    // Sliding window rate limiter using Redis sorted sets
    private boolean isRateLimitExceeded(String userId, int limit, int windowSeconds) {
        String key = "rate_limit:" + userId;
        long now = System.currentTimeMillis();
        long windowStart = now - (windowSeconds * 1000L);

        // Run as a Lua script for atomicity
        String luaScript = """
            local key = KEYS[1]
            local now = tonumber(ARGV[1])
            local window_start = tonumber(ARGV[2])
            local limit = tonumber(ARGV[3])
            local window_seconds = tonumber(ARGV[4])

            -- Remove old entries outside the window
            redis.call('ZREMRANGEBYSCORE', key, '-inf', window_start)

            -- Count current requests in window
            local count = redis.call('ZCARD', key)

            if count < limit then
                -- Add current request
                redis.call('ZADD', key, now, now .. '-' .. math.random(10000))
                redis.call('EXPIRE', key, window_seconds)
                return 0  -- not exceeded
            else
                return 1  -- exceeded
            end
            """;

        Long result = redis.execute(
            new DefaultRedisScript<>(luaScript, Long.class),
            List.of(key),
            String.valueOf(now),
            String.valueOf(windowStart),
            String.valueOf(limit),
            String.valueOf(windowSeconds)
        );

        return result != null && result == 1;
    }
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimited {
    int limit() default 100;
    int window() default 60; // seconds
}

// Usage
@RestController
public class SearchController {

    @GetMapping("/api/search")
    @RateLimited(limit = 10, window = 60)
    public SearchResult search(@RequestParam String query) { ... }
}
```

---

### Problem 2: Design a Distributed Notification Service

```java
// Requirements:
// - Multi-channel: Email, SMS, Push notifications
// - Reliable delivery (no dropped notifications)
// - Async processing
// - Retry with exponential backoff
// - Idempotent (no duplicate sends)

// Notification event (Kafka message)
public record NotificationEvent(
    String notificationId,      // UUID for idempotency
    String recipientId,
    NotificationType type,      // EMAIL, SMS, PUSH
    String templateId,
    Map<String, String> templateVariables,
    int priority,               // 1=critical, 5=low
    Instant scheduledFor        // null = send immediately
) {}

// Kafka consumer with retry
@Service
@KafkaListener(topics = "notifications", groupId = "notification-service",
               containerFactory = "notificationKafkaListenerFactory")
public class NotificationProcessor {

    @KafkaHandler
    @Transactional
    public void processNotification(NotificationEvent event,
                                    @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
        // Idempotency check
        if (notificationRepo.existsByNotificationId(event.notificationId())) {
            log.info("Duplicate notification {}, skipping", event.notificationId());
            return; // Already processed
        }

        // Record attempt
        NotificationRecord record = notificationRepo.save(
            new NotificationRecord(event.notificationId(), NotificationStatus.PROCESSING)
        );

        try {
            NotificationChannel channel = channelFactory.getChannel(event.type());
            String rendered = templateEngine.render(event.templateId(), event.templateVariables());
            channel.send(event.recipientId(), rendered);

            record.setStatus(NotificationStatus.SENT);
            record.setSentAt(Instant.now());

        } catch (ChannelException e) {
            record.setStatus(NotificationStatus.FAILED);
            record.setFailureReason(e.getMessage());
            record.incrementRetryCount();

            if (record.getRetryCount() < 3) {
                throw e; // Retry via Kafka consumer retry mechanism
            } else {
                // Max retries exceeded → dead letter queue
                record.setStatus(NotificationStatus.DEAD_LETTERED);
            }
        } finally {
            notificationRepo.save(record);
        }
    }
}

// Channel abstraction
public interface NotificationChannel {
    void send(String recipientId, String content);
    NotificationType getType();
}

@Component
public class EmailNotificationChannel implements NotificationChannel {
    @Override
    @CircuitBreaker(name = "email-service", fallbackMethod = "fallbackEmail")
    @Retry(name = "email-service")
    public void send(String recipientId, String content) {
        emailProvider.send(recipientId, content); // SES, SendGrid, etc.
    }
}
```

---

## 11. Debugging & Tricky Questions

---

### Q21: "My `@Transactional` is not working. Walk me through how you'd debug it."

#### 📖 Complete Debugging Checklist

```java
// ❌ REASON 1: Self-invocation (most common)
@Service
public class UserService {
    @Transactional
    public void outerMethod() {
        this.innerMethod(); // Calls real object, NOT the proxy → no transaction!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void innerMethod() { ... }
}
// FIX: Extract to another bean, or inject self via @Autowired/@Lazy

// ❌ REASON 2: Method is private
@Service
public class UserService {
    @Transactional
    private void processUser() { ... } // Spring can't proxy private methods!
    // No exception thrown — just silently ignored
}
// FIX: Make method public

// ❌ REASON 3: @Transactional on final method with CGLIB
@Service
public class UserService {
    @Transactional
    public final void saveUser() { ... } // CGLIB can't override final
}
// FIX: Remove final

// ❌ REASON 4: Checked exception doesn't rollback by default
@Transactional
public void transferFunds() throws InsufficientFundsException {
    debit(from, amount);
    credit(to, amount); // Throws checked exception → NO ROLLBACK by default!
}
// FIX:
@Transactional(rollbackFor = InsufficientFundsException.class)

// ❌ REASON 5: @Transactional on non-Spring-managed class
public class UserHelper { // Not @Component/@Service!
    @Transactional // Spring never processes this — class not in context
    public void helper() { ... }
}

// ❌ REASON 6: @EnableTransactionManagement missing (rare in Spring Boot but possible)
// Spring Boot's @SpringBootApplication includes it, but custom configs may miss it

// ❌ REASON 7: Using the wrong PlatformTransactionManager in multi-datasource setup
@Transactional("secondaryTransactionManager") // Must specify which TM
public void saveToSecondaryDb() { ... }
```

---

### Q22: Common Production Issues and How to Diagnose Them

```java
// ISSUE 1: Memory leak — heap keeps growing, GC can't free it

// Diagnostic:
// 1. Enable GC logging: -Xlog:gc*:gc.log
// 2. Take heap dump: jmap -dump:format=b,file=heap.hprof <pid>
//    Or via Actuator: POST /actuator/heapdump
// 3. Analyze with Eclipse MAT or VisualVM

// Common causes:
// - Static collections growing unbounded
// - ThreadLocal not cleaned up
// - Event listeners never removed
// - Long-running transactions keeping objects in Hibernate 1st level cache

// ISSUE 2: Thread pool exhaustion — all requests start timing out

// Diagnostic:
// GET /actuator/metrics/executor.active (HikariCP, thread pools)
// Thread dump: kill -3 <pid> or jstack <pid>
// Look for: all threads in WAITING state on the same lock

// Common cause: upstream service is slow → threads pile up waiting for response
// Fix: Add timeouts to all outbound calls, configure circuit breakers

// ISSUE 3: Database connection pool exhaustion

// Diagnostic in logs:
// HikariPool-1 - Connection is not available, request timed out after 30000ms

// Find leaked connections:
spring.datasource.hikari.leak-detection-threshold=10000 # log stack trace of long-held connections

// Common causes:
// - @Transactional too broad scope (holds connection for entire long operation)
// - Not closing connections in finally blocks (rare with Spring, common in raw JDBC)
// - Too many connections per request (N+1 with pessimistic locking)

// ISSUE 4: Spring context failing to start in production but not locally
// Usually: missing environment variables or external service not reachable
// spring.cloud.config.fail-fast=true → Config Server unreachable
// spring.datasource.url pointing to non-existent DB

// Solution: actuator/env endpoint to see resolved properties
// Or: @Value("${db.url:NOT_SET}") — add defaults to catch missing props early
```

---

### Q23: Tricky Interview Questions with Surprising Answers

```java
// Q: What happens when you define the same @Bean in multiple @Configuration classes?
// A: The LAST one loaded wins (alphabetical by class name by default).
//    With @AutoConfiguration, ordering can be controlled.
//    This is why @ConditionalOnMissingBean exists — to not override user beans.

// Q: Can you have two @Primary beans of the same type?
// A: Yes, but injection will throw NoUniqueBeanDefinitionException.
//    @Primary just means "prefer this one when no @Qualifier is specified."
//    Two @Primary of same type = ambiguity → exception.

// Q: What's the difference between @Component and @Bean?
// A: @Component: class-level, Spring scans and manages instantiation
//    @Bean: method-level in @Configuration, YOU control instantiation logic
//    @Bean is needed when: creating beans from third-party classes (no source)
//    Or when bean creation requires complex logic

// Q: Can a @Configuration class be a Spring bean itself?
// A: YES! @Configuration implies @Component. The class is a managed bean.
//    In fact, @Configuration classes are themselves proxied by CGLIB!
//    This is why calling @Bean methods within a @Configuration class
//    returns the same instance (goes through the proxy = singleton guaranteed).

// Proof of the above:
@Configuration
public class AppConfig {
    @Bean
    public Service service() {
        return new Service(datasource()); // Calls datasource() — looks like new object each time?
    }

    @Bean
    public DataSource datasource() {
        return new DataSource(); // But this is PROXIED — returns same singleton!
    }
    // service() and any other @Bean method that calls datasource()
    // gets the SAME DataSource instance — the proxy intercepts and returns the cached bean
}

// Q: What is @Lazy and when would you use it?
// A: Delays bean initialization until first use.
//    Use for: expensive beans not needed on startup, circular dependency resolution

// Q: How do you resolve a circular dependency?
// Constructor injection → Spring fails fast with BeanCurrentlyInCreationException ✅ (correct behavior!)
// Field injection → Spring resolves via proxy (but masks the design problem)
// Fix options:
//   1. Redesign to remove circular dependency (best)
//   2. Use @Lazy on one injection point
//   3. Use setter injection (lazy evaluation)
//   4. Use ApplicationContext.getBean() at method level (avoid)

// Q: What's the difference between @RestController and @Controller?
// A: @RestController = @Controller + @ResponseBody on every method
//    Means return values are serialized directly to HTTP response body
//    @Controller without @ResponseBody: return value is a view name (for Thymeleaf/JSP)
```

---

## 12. Rapid Revision Cheat Sheet

---

### 🔥 Top 20 Things That Trip Up Senior Candidates

```
CORE SPRING BOOT:
✅ @SpringBootApplication = @Configuration + @EnableAutoConfiguration + @ComponentScan
✅ Auto-config reads META-INF/spring/AutoConfiguration.imports, applies @Conditional guards
✅ debug=true shows auto-config report (positive/negative matches)
✅ @ConditionalOnMissingBean: user beans always win over auto-config beans
✅ CommandLineRunner vs ApplicationRunner: both post-startup, latter has structured args
✅ @EventListener(ApplicationReadyEvent): safest hook — web server already UP

SPRING INTERNALS:
✅ BeanPostProcessor.postProcessAfterInitialization: WHERE PROXIES ARE CREATED
✅ BeanFactoryPostProcessor: modifies bean DEFINITIONS (before instantiation)
✅ AOP proxy: JDK (requires interface) vs CGLIB (subclass, Spring Boot 2+ default)
✅ Final methods + CGLIB = silently broken @Transactional (no exception!)
✅ Self-invocation bypasses proxy = @Transactional / @Cacheable / @Async all fail
✅ @Configuration classes are CGLIB-proxied → @Bean methods return singletons

TRANSACTIONS:
✅ Default rollback: RuntimeException only. Checked exceptions = COMMIT (bug trap!)
✅ REQUIRES_NEW: own transaction + own DB connection (useful for audit logs)
✅ Self-invocation = no transaction (most common interview gotcha)
✅ @Transactional on private methods = silently ignored
✅ readOnly=true: optimization hint to DB + prevents accidental writes

MICROSERVICES:
✅ Circuit Breaker states: CLOSED → OPEN → HALF_OPEN → CLOSED
✅ Saga: Choreography (event-driven) vs Orchestration (central coordinator)
✅ Outbox Pattern: save event to DB in same TX as domain change → relay to Kafka
✅ REQUIRES_NEW for audit logging: survives main transaction rollback

SECURITY:
✅ JWT filter: add BEFORE UsernamePasswordAuthenticationFilter
✅ STATELESS sessions: no HttpSession, validate JWT on every request
✅ @PreAuthorize: evaluated at proxy level — only works on public methods
✅ OAuth2 Resource Server: validates JWT signature using JWKS from issuer

PERFORMANCE:
✅ N+1: detect with Hibernate statistics, fix with JOIN FETCH / @EntityGraph / @BatchSize
✅ IDENTITY strategy disables JDBC batch inserts — use SEQUENCE
✅ @Cacheable on private methods = silently ignored (proxy!)
✅ Cache stampede: use distributed lock (Redisson) or probabilistic early expiration
✅ connection.leak-detection-threshold: logs stack trace of long-held connections

TESTING:
✅ @WebMvcTest: web layer only — must @MockBean all services
✅ @DataJpaTest: JPA only — uses H2 by default, wraps in auto-rollback transaction
✅ @SpringBootTest: full context — slow but real
✅ Testcontainers: real DB in Docker — avoid H2 dialect differences
✅ @DynamicPropertySource: override properties per test container
```

### 🎯 Senior-Level Power Phrases

> *"Before adding an index, I always check cardinality and the read/write ratio first..."*

> *"Self-invocation bypasses the AOP proxy — that's the #1 cause of @Transactional not working in production..."*

> *"For distributed transactions in microservices, 2PC doesn't scale — I'd use the Saga pattern with either choreography or orchestration depending on complexity..."*

> *"The Outbox Pattern solves the dual-write problem — you can't atomically write to the DB and publish to Kafka in one step without it..."*

> *"For circuit breakers, I configure separate bulkheads per downstream service so one slow service doesn't exhaust the thread pool for others..."*

> *"In production, EAGER loading on @OneToMany is a red flag — it means every entity load triggers joins the developer may not have intended..."*

> *"Kubernetes liveness vs readiness probes serve different purposes — liveness restarts a dead pod, readiness removes it from load balancer traffic. Getting this wrong causes rolling deployment downtime..."*

---

*Last updated: 2025 | Senior Spring Boot Interview Guide*
*Technologies: Java 21, Spring Boot 3.x, Spring Cloud 2023.x, Kubernetes 1.28+*
*Author perspective: 15+ years enterprise Java, microservices at scale*
