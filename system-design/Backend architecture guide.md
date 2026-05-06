# 📚 Backend Architecture & System Design Guide
### Beginner → Interview-Ready | Practical, Real-World Focused

---

> **How to use this guide:**
> Read top-to-bottom once, then use the Cheat Sheet (Section 9) for last-minute revision.
> Every section builds on the previous. Examples are always from real systems.

---

# 1. What is Software Architecture?

## Simple Explanation

Software architecture is **the high-level blueprint of a system** — the decisions about how components are structured, how they communicate, and how the system will scale and evolve over time.

Think of it like city planning. A city planner doesn't lay bricks or pour concrete. They decide:
- Where roads go (communication paths)
- Where power plants and water towers sit (infrastructure)
- Which neighborhoods are residential vs industrial (separation of concerns)
- How traffic flows between zones (data flow)

The individual buildings (code) come later. But if the city plan is wrong, adding more buildings makes things worse, not better.

**In software:** Architecture decides things like —
- Do we split into many small services or one big application?
- How does the mobile app talk to the backend?
- Where do we put a cache to reduce database load?
- How do we make sure the system stays up when one component fails?

## Why It Matters

| Without Architecture | With Architecture |
|---|---|
| Works for 100 users, breaks at 10,000 | Designed to scale to millions |
| One developer's change breaks another's work | Clear boundaries and ownership |
| Adding features takes weeks (tangled code) | Features plug into defined slots |
| System goes down when DB server crashes | Redundancy built into design |
| Can't tell what went wrong when production breaks | Observability designed in from the start |

## Real-World Analogy

**Architecture is like an airport design.**

A small regional airport has one terminal, one runway, one check-in counter. Simple, works for 50 flights a day. Now try running Heathrow with that design — impossible.

Heathrow has:
- Multiple terminals (separated concerns)
- Dedicated baggage systems (specialized subsystems)
- Multiple runways with air traffic control (load distribution + coordination)
- Emergency systems that don't interfere with normal operations (fault isolation)
- A way to add a new terminal without shutting down the others (extensibility)

That planning IS architecture. The steel and concrete (code) just follows the blueprint.

## The Architect's Core Questions

Before writing a single line of code, a good architect asks:

```
1. What are the functional requirements? (What must the system DO?)
2. What are the non-functional requirements? (How fast? How available? How secure?)
3. What are the constraints? (Team size, budget, existing technology?)
4. What will change in the future? (What needs to be easy to modify?)
5. What are the failure modes? (What can break, and what happens when it does?)
```

---

# 2. Architecture Patterns (CORE)

---

## 2.1 Monolith Architecture

### 🧠 Simple Explanation

Everything lives in one codebase, one deployable unit, one process. The entire application — user interface logic, business rules, database access — is built and deployed together.

Think of it as a Swiss Army knife: everything in one tool.

### 🏗️ Structure

```
┌──────────────────────────────────────────┐
│           MONOLITH APPLICATION           │
│                                          │
│  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │  User    │  │  Order   │  │Payment │ │
│  │  Module  │  │  Module  │  │ Module │ │
│  └──────────┘  └──────────┘  └────────┘ │
│                                          │
│  ┌──────────────────────────────────┐    │
│  │       Shared Database Layer      │    │
│  └──────────────────────────────────┘    │
└──────────────────────────────────────────┘
               │
        Single Database
```

All modules share memory, the same codebase, and often the same database. One `git push` deploys everything.

### 📦 When to Use

- **Early stage startups** — when you need to move fast and validate ideas
- **Small teams** (< 5 engineers) — overhead of distributed systems isn't worth it
- **Simple domain** — not enough complexity to justify splitting
- **Tight deadlines** — fastest time to market
- **Unknown domain** — if you don't understand the problem well enough to split it correctly, don't split yet

### ❌ When NOT to Use

- Multiple large teams need to deploy independently
- Different parts of the system have drastically different scaling needs (checkout vs product search)
- Different modules need different technology stacks
- A bug in one module repeatedly crashes unrelated functionality
- The codebase has become so large that understanding it takes weeks for new engineers

### 💡 Real-World Example

**GitHub started as a monolith** (Ruby on Rails), and so did **Twitter**, **Shopify**, **Stack Overflow**, and **Basecamp**. Most successful companies started monolith-first.

Shopify is famous for running one of the world's largest Rails monoliths. Their approach: invest heavily in a **modular monolith** (strict module boundaries in one codebase) rather than migrating to microservices.

### ⚠️ Interview Talking Points

> "The monolith gets a bad reputation, but it's often the correct choice. The right question isn't 'should we use microservices?' but 'what problem are we solving?' A monolith with clear module boundaries, good test coverage, and a clean deployment pipeline can outperform a poorly designed microservices system. I'd choose a monolith for a new product and migrate only once I have a clear understanding of the domain boundaries and a concrete scaling problem to solve."

**Key trade-offs to mention:**
- Simple to develop and debug vs. hard to scale and deploy at large scale
- One deployment unit vs. no partial deployments
- Shared memory = fast in-process calls vs. single point of failure

---

## 2.2 Microservices Architecture

### 🧠 Simple Explanation

Split the application into small, independent services. Each service owns one responsibility, has its own database, and deploys independently. Services communicate over a network.

Think of it as a food court: each stall (service) specializes in one cuisine and operates independently. The food court (platform) provides shared utilities (power, space, customers).

### 🏗️ Structure

```
                    ┌─────────────┐
                    │ API Gateway │
                    └──────┬──────┘
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │    User     │ │    Order    │ │   Payment   │
    │   Service   │ │   Service   │ │   Service   │
    └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
           │               │               │
    ┌──────┴──────┐ ┌──────┴──────┐ ┌──────┴──────┐
    │  Users DB   │ │  Orders DB  │ │ Payments DB │
    └─────────────┘ └─────────────┘ └─────────────┘
```

### 📦 When to Use

- Multiple large teams needing independent deployments
- Different services need to scale independently (search vs checkout)
- Different services benefit from different tech stacks (ML service in Python, core in Go)
- Strict organizational boundaries (one team = one service = one P&L)
- System requires 99.99%+ availability (isolate failures)

### ❌ When NOT to Use

- Small teams (distributed systems complexity isn't worth it)
- Early stage — don't know domain boundaries yet
- Tight coupling between services anyway (you'll have a "distributed monolith")
- Team doesn't have DevOps maturity (monitoring, tracing, CI/CD pipelines per service)

### 💡 Real-World Example

**Netflix** decomposes their system into hundreds of microservices: separate services for recommendations, playback, user profiles, billing, search, device management. Each team owns their service fully. This lets the recommendation team deploy improvements without touching the billing code.

**Amazon** famously mandated microservices in the early 2000s via Jeff Bezos's "API mandate": every team must expose their data/functionality through service APIs, teams may only communicate through these APIs.

### ⚠️ Interview Talking Points

> "Microservices solve organizational and scaling problems, but they introduce distributed systems complexity: network failures, service discovery, distributed tracing, and eventual consistency. I wouldn't choose microservices to solve a technical problem — I'd choose them when teams need to move independently. The biggest anti-pattern I see is creating microservices before understanding the domain — you end up with tightly coupled services that need to be deployed together, a 'distributed monolith' that has all the downsides of both approaches."

---

## 2.3 Layered Architecture (N-Tier)

### 🧠 Simple Explanation

Organize code into horizontal layers where each layer has a specific responsibility and only communicates with the layer directly below it.

Think of a restaurant: customers interact with waiters (Presentation), waiters communicate with the kitchen (Business Logic), the kitchen gets ingredients from the pantry (Data Access), and the pantry talks to suppliers (Database).

### 🏗️ Structure

```
┌────────────────────────────────────┐
│         Presentation Layer         │  ← HTTP controllers, REST endpoints
│    (Handles HTTP, Input/Output)     │
└────────────────────────────────────┘
                  │
                  ▼
┌────────────────────────────────────┐
│         Business Logic Layer       │  ← Use cases, domain rules, validation
│    (Core application logic)        │
└────────────────────────────────────┘
                  │
                  ▼
┌────────────────────────────────────┐
│         Data Access Layer          │  ← Repositories, ORM, queries
│    (Database interaction)          │
└────────────────────────────────────┘
                  │
                  ▼
┌────────────────────────────────────┐
│            Database                │  ← PostgreSQL, MySQL, MongoDB
└────────────────────────────────────┘
```

**Code example (Python/FastAPI):**
```python
# Presentation Layer — HTTP Controller
@app.post("/users")
def create_user(request: CreateUserRequest):
    user = user_service.create_user(request.email, request.password)
    return UserResponse.from_domain(user)

# Business Logic Layer — Service
class UserService:
    def create_user(self, email: str, password: str) -> User:
        if self.user_repo.exists_by_email(email):
            raise DuplicateEmailError(email)
        hashed = hash_password(password)
        return self.user_repo.save(User(email=email, password=hashed))

# Data Access Layer — Repository
class UserRepository:
    def save(self, user: User) -> User:
        row = self.db.execute(
            "INSERT INTO users (email, password) VALUES (?, ?) RETURNING id",
            [user.email, user.password]
        )
        return user.with_id(row["id"])
```

### 📦 When to Use

- Standard CRUD applications with clear separation of concerns
- Teams with clear frontend/backend/database specialists
- Codebases that need to be easy to onboard new developers
- Applications where the business logic is the most valuable part

### ❌ When NOT to Use

- Performance-critical paths where passing through every layer adds latency
- Complex domains — layered architecture tends to produce "anemic domain models" (logic in services, not objects)
- When you need to test business logic without database infrastructure

### 💡 Real-World Example

Most Spring Boot (Java) and Django (Python) applications use layered architecture by default. An e-commerce backend: Controllers receive HTTP requests → Services validate and apply business rules → Repositories query PostgreSQL.

### ⚠️ Interview Talking Points

> "Layered architecture is the most common pattern I see in enterprise backends — it's simple to understand and maintain. The main risk is that business logic leaks into controllers or repositories when developers take shortcuts. I enforce it by code review and sometimes by making the layers compile-time separate modules. For simple CRUD services, layered is perfect. For complex domains, I'd look at Clean or Hexagonal."

---

## 2.4 Clean Architecture

### 🧠 Simple Explanation

Like layered architecture, but the dependency rule runs in ONE direction: **inward**. The core business logic (domain) has **zero dependencies on frameworks, databases, or external systems**. Everything depends on the domain; the domain depends on nothing.

Robert C. Martin ("Uncle Bob") coined this. The idea: your business rules should work regardless of whether you use PostgreSQL or MongoDB, REST or gRPC, React or Angular. The infrastructure details are **plugins** to the core.

### 🏗️ Structure

```
         ┌─────────────────────────────────────┐
         │           Frameworks & Drivers       │  Outermost ring
         │  (Web, Database, UI, External APIs)  │
         │   ┌──────────────────────────────┐   │
         │   │      Interface Adapters      │   │
         │   │  (Controllers, Repositories, │   │
         │   │   Presenters, Gateways)      │   │
         │   │  ┌─────────────────────┐    │   │
         │   │  │   Application Core  │    │   │
         │   │  │  (Use Cases /       │    │   │
         │   │  │   Application Logic)│    │   │
         │   │  │  ┌──────────────┐  │    │   │
         │   │  │  │   Entities   │  │    │   │
         │   │  │  │ (Core Domain │  │    │   │
         │   │  │  │  Business    │  │    │   │
         │   │  │  │   Rules)     │  │    │   │
         │   │  │  └──────────────┘  │    │   │
         │   │  └─────────────────────┘    │   │
         │   └──────────────────────────────┘   │
         └─────────────────────────────────────┘

DEPENDENCY RULE: Dependencies only point INWARD.
Inner rings know NOTHING about outer rings.
```

**Code example:**
```python
# INNER RING — Entity (pure domain, no imports from framework)
class Order:
    def __init__(self, items: list[OrderItem]):
        self.items = items
        self.status = OrderStatus.PENDING

    def calculate_total(self) -> Decimal:
        return sum(item.price * item.qty for item in self.items)

    def confirm(self):
        if not self.items:
            raise EmptyOrderError()
        self.status = OrderStatus.CONFIRMED

# USE CASE — Application logic (depends only on entities + interfaces)
class ConfirmOrderUseCase:
    def __init__(self, order_repo: OrderRepository, payment_gateway: PaymentGateway):
        self.order_repo = order_repo          # Interface (abstract)
        self.payment_gateway = payment_gateway  # Interface (abstract)

    def execute(self, order_id: str) -> Order:
        order = self.order_repo.get_by_id(order_id)
        order.confirm()
        self.payment_gateway.charge(order.calculate_total())
        return self.order_repo.save(order)

# OUTER RING — Infrastructure (concrete implementation, depends inward)
class PostgresOrderRepository(OrderRepository):  # Implements the interface
    def get_by_id(self, order_id: str) -> Order:
        row = self.db.execute("SELECT * FROM orders WHERE id = ?", [order_id])
        return Order.from_row(row)
```

### 📦 When to Use

- Complex business domains (fintech, healthcare, insurance)
- Long-lived applications that will evolve significantly
- When you need to switch infrastructure (e.g., swap databases) without rewriting business logic
- When unit testing business logic in isolation is important

### ❌ When NOT to Use

- Simple CRUD apps (enormous overhead for minimal benefit)
- Small teams unfamiliar with the pattern (steep learning curve)
- When development speed is the primary concern

### 💡 Real-World Example

Financial systems and enterprise applications heavily use Clean Architecture. A banking application where the domain (account, transaction, transfer) must remain stable even as the infrastructure evolves — new payment providers, new databases, regulatory requirements.

### ⚠️ Interview Talking Points

> "Clean Architecture shines in complex, long-lived domains where the business rules are the valuable intellectual property. The key insight is the dependency inversion principle: high-level policy (business rules) shouldn't depend on low-level details (database, framework). By making infrastructure depend on the domain through interfaces, I can test all business logic without spinning up a database. The trade-off is significant upfront complexity — I wouldn't apply it to a simple REST CRUD service."

---

## 2.5 Hexagonal Architecture (Ports & Adapters)

### 🧠 Simple Explanation

Similar intent to Clean Architecture, but framed differently. The application core sits in the middle. The outside world communicates through **Ports** (interfaces defined by the core) and **Adapters** (implementations of those interfaces).

**Ports** = openings in the hexagon (defined by your app)  
**Adapters** = plugs that fit into those openings (defined by the infrastructure)

Think of a laptop: it has USB-C ports (defined by the laptop's design). Any manufacturer can make a USB-C adapter (charger, monitor, USB hub) that fits the standard port. The laptop doesn't care about the specific adapter brand.

### 🏗️ Structure

```
                  REST Adapter   CLI Adapter   Test Adapter
                       │              │              │
                       └──────────────┼──────────────┘
                                      │
                              ┌───────┴────────┐
                   Driving    │    INBOUND     │  (Left side — things that
                    Ports ────▶│     PORTS      │   drive the application)
                              │                │
                              │  APPLICATION   │
                              │     CORE       │
                              │                │
                    Driven    │    OUTBOUND    │  (Right side — things the
                    Ports ────▶│     PORTS      │   application drives)
                              └───────┬────────┘
                                      │
                       ┌──────────────┼──────────────┐
                       │              │              │
               PostgreSQL Adapter  Redis Adapter  Email Adapter
```

**Key difference from layered:** There's no "top" or "bottom." The core is surrounded symmetrically by adapters on all sides. You can drive the application from any direction: REST, CLI, message queue, or a test.

### 📦 When to Use

- Same as Clean Architecture — complex domains, long-lived applications
- When you need to test the application core without any infrastructure
- Multiple ways to interact with the same core (REST + gRPC + CLI + message queue all trigger the same use cases)

### 💡 Real-World Example

A payment processing service: the core handles payment rules. Inbound adapters: REST API, Kafka consumer (for async payment requests). Outbound adapters: Stripe API, PostgreSQL, Audit Log service. You can run the entire payment logic in unit tests with mock adapters, zero infrastructure.

### ⚠️ Interview Talking Points

> "Hexagonal architecture and Clean Architecture share the same core insight: isolate your domain from infrastructure. I prefer the Ports & Adapters framing because it makes testing trivial — I swap the Postgres adapter with an in-memory adapter in tests, so the entire test suite runs in milliseconds without a database. In interviews I often draw the hexagon with 'driving ports' on the left (HTTP, queue) and 'driven ports' on the right (database, external APIs) to explain it visually."

---

## 2.6 Event-Driven Architecture (EDA)

### 🧠 Simple Explanation

Services communicate by **emitting and reacting to events** rather than calling each other directly. When something happens (an order is placed), a service emits an event. Any service that cares about that event reacts to it.

Think of a newspaper: the publisher prints the news. Thousands of subscribers read it in their own time. The publisher doesn't know who's subscribed, and doesn't wait for readers to finish before printing the next edition.

### 🏗️ Structure

```
                     ┌──────────────────┐
                     │   Event Broker   │
                     │  (Kafka / SNS)   │
                     └────────┬─────────┘
                              │
         Publishes:           │          Subscribes:
    "order.placed"  ──────────┤
    "payment.done"            │──────────▶  Notification Service
    "item.shipped"            │──────────▶  Analytics Service
                              │──────────▶  Inventory Service
    ┌──────────────┐          │──────────▶  Fraud Detection
    │ Order Service│──────────┘
    │  (Producer)  │
    └──────────────┘
```

**Event schema example:**
```json
{
  "eventId": "evt-8f3a2b",
  "eventType": "order.placed",
  "version": "1.0",
  "timestamp": "2024-06-15T14:23:00Z",
  "source": "order-service",
  "data": {
    "orderId": "ord-992",
    "userId": "usr-123",
    "items": [{"sku": "LAPTOP-PRO", "quantity": 1, "price": 1299.00}],
    "total": 1299.00
  }
}
```

### 📦 When to Use

- Multiple services need to react to the same event
- Decoupling services that would otherwise need to know about each other
- Audit trail required (event log is a natural audit log)
- Workflows where parallel processing improves speed
- When you want to add new consumers without changing producers

### ❌ When NOT to Use

- Simple request/response flows (adds unnecessary complexity)
- When you need immediate strong consistency (event-driven is inherently eventual)
- Very small systems (a message broker is heavy infrastructure for a simple app)
- When debugging and tracing are not yet mature in the team

### 💡 Real-World Example

**Uber's** surge pricing is event-driven: ride requests, driver locations, and traffic data are all events flowing through Kafka. The pricing engine subscribes to these events and updates prices in near-real time without any service needing to call the pricing engine directly.

### ⚠️ Interview Talking Points

> "Event-driven architecture is powerful for decoupling, but it trades simplicity for scalability. The producer doesn't care who consumes the event — I can add a new Fraud Detection service that reacts to 'order.placed' without touching the Order Service. The hardest parts are: debugging (a request spans multiple services and events — you need distributed tracing), schema evolution (old consumers break if you rename fields — use versioning), and eventual consistency (the inventory update happens asynchronously — there's a window where the order exists but inventory hasn't been reserved)."

---

## 2.7 Serverless Architecture

### 🧠 Simple Explanation

Run individual functions in the cloud without managing any servers. You write a function, upload it, and the cloud provider runs it on demand — zero, one, or a million times — and you pay per invocation.

Think of a taxi vs. owning a car. Owning a server = owning a car (always available, always costs money). Serverless = taking a taxi (available on demand, pay only when you use it).

### 🏗️ Structure

```
HTTP Request                        Event Trigger
      │                                   │
      ▼                                   ▼
┌───────────┐                     ┌───────────────┐
│ API       │                     │  S3 Upload    │
│ Gateway   │                     │  Event        │
└─────┬─────┘                     └───────┬───────┘
      │                                   │
      ▼                                   ▼
┌─────────────────┐           ┌─────────────────────┐
│ Lambda Function │           │   Lambda Function   │
│ /create-order   │           │  process-image      │
└────────┬────────┘           └──────────┬──────────┘
         │                              │
         ▼                              ▼
   ┌──────────┐                  ┌──────────────┐
   │ DynamoDB │                  │  S3 Bucket   │
   └──────────┘                  └──────────────┘
```

**Lambda function example:**
```python
import json
import boto3

def handler(event, context):
    """
    Triggered by API Gateway POST /orders
    event contains the HTTP request details
    """
    body = json.loads(event['body'])
    order_id = create_order(body['user_id'], body['items'])

    return {
        'statusCode': 201,
        'body': json.dumps({'orderId': order_id}),
        'headers': {'Content-Type': 'application/json'}
    }
```

### 📦 When to Use

- Event-driven workloads (process image on S3 upload)
- Variable/unpredictable traffic (0 to 10,000 requests/hour)
- Scheduled tasks (cron jobs)
- Short-lived, stateless operations
- Startups wanting zero infrastructure management

### ❌ When NOT to Use

- Long-running processes (Lambda max: 15 minutes)
- Latency-sensitive APIs (cold starts add 100ms–2s)
- Stateful applications (Lambda is ephemeral)
- High, sustained traffic (EC2 becomes cheaper at consistent load)
- When you need fine-grained control over the runtime environment

### 💡 Real-World Example

**Netflix** uses AWS Lambda for image encoding notifications — when a new movie is uploaded to S3, a Lambda triggers the encoding pipeline. **iRobot** uses Lambda to process data from millions of Roomba vacuums — traffic spikes unpredictably, and Lambda auto-scales instantly.

### ⚠️ Interview Talking Points

> "Serverless is operationally simple — no servers to patch, scales to zero, zero cost when idle. The trade-offs are cold starts (functions take time to 'warm up' after being idle), vendor lock-in (Lambda is AWS-specific), and difficulty debugging complex workflows. I'd use it for event-triggered processing, async tasks, and highly variable traffic. For latency-sensitive APIs with consistent traffic, a containerized service is more cost-effective and predictable."

---

# 3. Deep Dive: Microservices

## 3.1 What Problem Does It Solve?

### The Monolith Scaling Problem

Imagine an e-commerce company with 10 million users. Their monolith runs on 20 servers. Black Friday approaches:

```
Problem 1: Product Search is at 10x normal load. Checkout is at 2x.
  → Must scale ALL 20 servers, even the checkout code that doesn't need it.
  → Wastes cost.

Problem 2: A new team joins to build a Recommendations engine.
  → They must understand and work within the existing codebase.
  → Their code changes risk breaking checkout.

Problem 3: The Recommendations team wants to use Python + ML libraries.
  → The monolith is Java. They can't.

Problem 4: A bug in the Order module crashes the whole application.
  → Product search goes down too. Users can't browse.
```

**Microservices solves these by:**
1. **Independent scaling** — scale Search service 10x, Checkout service 2x
2. **Team autonomy** — Recommendations team owns their service completely
3. **Technology diversity** — Recommendations in Python, Checkout in Go, Orders in Java
4. **Fault isolation** — Order service crash doesn't take down Search

## 3.2 Service Communication

### Option A: Synchronous — REST over HTTP

```
Order Service ──── HTTP POST /reserve ────▶ Inventory Service
              ◀─── { "reserved": true } ───

Characteristics:
✓ Simple to implement and debug
✓ Immediate response (request/reply model)
✗ Tight temporal coupling (both must be up)
✗ Caller blocked while waiting
```

**Example:**
```python
# Order Service calling Inventory Service
import httpx

async def place_order(order: Order) -> OrderConfirmation:
    # Synchronous: Order Service waits for Inventory Service
    response = await httpx.post(
        "http://inventory-service/reserve",
        json={"items": order.items},
        timeout=5.0
    )
    if response.status_code != 200:
        raise InventoryError("Could not reserve items")

    reservation = response.json()
    # Continue with order confirmation...
```

### Option B: Synchronous — gRPC

```
Better than REST for:
- Internal service-to-service calls
- High throughput, low latency
- Strongly typed contracts (protobuf)
- Streaming support

Used by: Netflix, Google, Square, Dropbox
```

```protobuf
// inventory.proto
service InventoryService {
  rpc ReserveItems(ReserveRequest) returns (ReserveResponse);
  rpc StreamStockLevels(StockRequest) returns (stream StockUpdate);
}

message ReserveRequest {
  repeated Item items = 1;
}

message ReserveResponse {
  bool success = 1;
  string reservation_id = 2;
}
```

### Option C: Asynchronous — Message Queue

```
Order Service ──── "order.placed" event ────▶ Message Broker (Kafka)
                                                       │
                                    ┌──────────────────┼────────────────┐
                                    ▼                  ▼                ▼
                            Inventory Svc      Payment Svc     Email Svc
                         (reserve items)    (charge card)   (send confirm)

Characteristics:
✓ Services don't need to be online simultaneously
✓ Retry on failure built in
✓ Fan-out (one event → many consumers)
✗ Eventual consistency (not immediate)
✗ Harder to debug (request spans multiple services)
```

## 3.3 Pros & Cons

| | Pros | Cons |
|---|---|---|
| **Deployment** | Deploy services independently | More CI/CD pipelines to maintain |
| **Scaling** | Scale only what needs scaling | Service discovery required |
| **Reliability** | Failure isolation | Network calls can fail |
| **Teams** | Autonomous team ownership | API contract management needed |
| **Technology** | Choose right tool per service | Polyglot = operational complexity |
| **Testing** | Test in isolation | Integration testing harder |
| **Debugging** | Clear service boundaries | Distributed tracing required |

## 3.4 Challenges in Detail

### Challenge 1: Distributed Transactions

In a monolith, a database transaction covers everything. In microservices, a "transaction" spans services with separate databases.

```
E-commerce order placement:
1. Reserve inventory (Inventory Service)
2. Charge credit card (Payment Service)
3. Create order record (Order Service)
4. Send confirmation (Email Service)

Problem: What if step 2 succeeds but step 3 fails?
→ Card charged, no order created, no email sent. Customer confused. 😱
```

**Solution: Saga Pattern**

```
Choreography Saga:
1. Order Service: creates order in PENDING state → emits "order.created"
2. Inventory Service: receives event, reserves items → emits "items.reserved"
3. Payment Service: receives event, charges card → emits "payment.completed"
4. Order Service: receives event, marks CONFIRMED → emits "order.confirmed"
5. Email Service: receives event, sends email

If Payment fails at step 3:
→ Emits "payment.failed"
→ Inventory Service receives it, releases reservation (compensating transaction)
→ Order Service marks CANCELLED
→ Email Service sends "sorry" email
```

### Challenge 2: Service Discovery

With 200 microservices and auto-scaling, how does Service A know the IP address of Service B?

```
Problem:
Service B runs on 5 instances: 10.0.1.2:8080, 10.0.1.3:8080, 10.0.1.5:8080...
They're added and removed constantly. Can't hardcode IPs.

Solution: Service Registry (Consul, Kubernetes DNS)

Service B registers itself on startup:
  "payment-service" → [10.0.1.2:8080, 10.0.1.3:8080, 10.0.1.5:8080]

Service A looks up "payment-service" → gets list → load balances
```

In Kubernetes, this is automatic:
```yaml
# Kubernetes Service: gives payment-service a stable DNS name
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment
  ports:
    - port: 80
# Other services call http://payment-service/charge
# Kubernetes routes to any healthy pod matching selector
```

### Challenge 3: Latency & Network Failures

In a monolith, calling another module is a function call: nanoseconds. In microservices, it's a network call: milliseconds — and it can fail.

```
Monolith: OrderService.confirmOrder() → nanoseconds, never fails
Microservices: POST http://inventory-service/reserve → 1-50ms, can timeout, can return 503

Solution: Resilience patterns
1. Timeouts: Never wait more than N seconds
2. Retries with backoff: Retry failed calls with increasing delays
3. Circuit Breaker: Stop calling a failing service (fail fast)
4. Fallback: Return cached/default response when service fails
```

## 3.5 Real-World Example: E-Commerce Microservices

```
Customer Journey → Architecture:

1. User browses products
   → Request hits CDN (cached product images)
   → API Gateway routes to Product Service
   → Product Service reads from Product DB (read replica for performance)

2. User searches
   → API Gateway routes to Search Service
   → Search Service queries Elasticsearch

3. User adds to cart
   → API Gateway routes to Cart Service
   → Cart Service uses Redis (fast, ephemeral — cart doesn't need strong durability)

4. User checks out
   → API Gateway routes to Order Service
   → Order Service starts Saga:
      a. Inventory Service: reserve items
      b. Payment Service: charge card (via Stripe)
      c. Order Service: confirm order
      d. Notification Service: send email/SMS
      e. Fulfillment Service: notify warehouse

5. User tracks order
   → Order Service reads order state from DB
   → Fulfillment Service publishes shipping events to Kafka
   → Notification Service subscribes, sends tracking updates
```

---

# 4. System Design Building Blocks

---

## 4.1 API Gateway

### What It Is

A single entry point that sits in front of all your backend services. Every client request goes through it.

### Why We Use It

Without an API Gateway, clients must know about every service's location. Adding auth to 50 services means adding auth logic 50 times.

```
WITHOUT API Gateway:                WITH API Gateway:
Mobile → User Service               Mobile ─────────────────┐
Mobile → Order Service              Web Browser ────────────┤  API Gateway
Mobile → Product Service            Third-party API ─────────┤  (one entry point)
Mobile → Cart Service               Internal CLI ────────────┘
# Client knows all service URLs          │
# Auth implemented in every service      ├──▶ User Service
# Rate limiting in every service         ├──▶ Order Service
# Logging in every service               ├──▶ Product Service
                                         └──▶ Cart Service
                                    # Auth, rate limiting, logging: ONCE
```

### Where It Sits

```
Internet → [API Gateway] → Internal network → [Microservices]
```

### What It Does

- **Request routing:** `/api/users/*` → User Service, `/api/orders/*` → Order Service
- **Authentication:** Validate JWT before forwarding to services
- **Rate limiting:** 100 requests/minute per user
- **SSL termination:** HTTPS outside, HTTP inside private network
- **Request/response transformation:** Modify headers, translate protocols
- **Load balancing:** Distribute across service instances
- **Logging:** Central request log for all services
- **Caching:** Cache common responses at the gateway level

### Real-World Example

**AWS API Gateway + Lambda:** Every request goes through API Gateway. It validates the JWT using a Lambda Authorizer, then routes to the appropriate Lambda function. Rate limiting is configured in API Gateway, not in any Lambda.

```
Client → API Gateway
          │ Validates JWT ──▶ Auth Lambda (returns allow/deny)
          │ Rate limit check (100 req/min per IP)
          │
          ├── GET /products/* ──▶ Product Lambda
          ├── POST /orders/*  ──▶ Order Lambda
          └── GET /users/*    ──▶ User Lambda
```

---

## 4.2 Load Balancer

### What It Is

Distributes incoming traffic across multiple server instances. Ensures no single server is overwhelmed.

### Why We Use It

```
WITHOUT Load Balancer:          WITH Load Balancer:
                                              ┌─▶ Server 1 (33% load)
All traffic ─▶ Server 1         Traffic ─▶ LB ─▶ Server 2 (33% load)
              (100% load)                    └─▶ Server 3 (33% load)

Server 1 dies → 100% outage     Server 1 dies → LB detects, routes to 2 & 3
                                 → 50% load each, 0% outage (for users)
```

### Load Balancing Algorithms

```python
# Round Robin: requests go to servers in turn
servers = [Server1, Server2, Server3]
def round_robin(request, counter):
    return servers[counter % len(servers)]

# Least Connections: send to server with fewest active connections
def least_connections(request):
    return min(servers, key=lambda s: s.active_connections)

# IP Hash: same client always hits same server (for sticky sessions)
def ip_hash(request):
    return servers[hash(request.client_ip) % len(servers)]
```

### Layer 4 vs Layer 7

```
L4 (Transport Layer — TCP/UDP):
  Routes based on IP + Port
  Doesn't read HTTP content
  Faster, less CPU
  Use: database proxies, raw TCP load balancing

L7 (Application Layer — HTTP):
  Routes based on URL, headers, cookies
  Can do SSL termination, content-based routing
  Slower but smarter
  Use: web applications, microservices routing
  Example: Route /api/v2/* to new servers, /api/v1/* to old servers
```

### Real-World Example

**AWS ALB (Application Load Balancer):** An e-commerce site has 20 EC2 instances. ALB routes `/static/*` to CloudFront (CDN), `/api/checkout` to checkout-specific instances, all other `/api/*` to general API instances. If an instance fails its health check (`GET /health` returns non-200), ALB stops sending traffic to it within 10 seconds.

---

## 4.3 Reverse Proxy

### What It Is

A server that sits in front of backend servers, receiving requests on their behalf.

### Reverse Proxy vs Load Balancer

```
Load Balancer: distributes load across MULTIPLE instances of the SAME service
Reverse Proxy: sits in front of servers, adds features (caching, compression, SSL, auth)

In practice, most L7 load balancers ARE reverse proxies.
Nginx can act as both.
```

### What a Reverse Proxy Adds

```
Client → [Nginx Reverse Proxy] → Backend Server

Reverse proxy can:
1. SSL termination (handle HTTPS, forward plain HTTP to backend)
2. Compression (gzip responses before sending to client)
3. Caching (cache backend responses, serve from cache on repeat requests)
4. Security (hide backend IPs, DDoS protection, WAF rules)
5. Rate limiting
6. Request logging
```

**Nginx config example:**
```nginx
server {
    listen 443 ssl;
    server_name api.example.com;

    ssl_certificate /certs/cert.pem;

    # Rate limiting
    limit_req zone=api burst=20 nodelay;

    # Gzip compression
    gzip on;
    gzip_types application/json;

    # Route to backend
    location /api/ {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Cache static API responses
    location /api/products {
        proxy_cache api_cache;
        proxy_cache_valid 200 60s;
        proxy_pass http://backend_servers;
    }
}
```

---

## 4.4 Service Discovery

### What It Is

A mechanism for services to find each other's network addresses dynamically, without hardcoding IPs.

### Why We Need It

```
Problem in dynamic environments (auto-scaling, containers):
- Services start and stop constantly
- IP addresses change
- Can't hardcode "payment service is at 10.0.1.42:8080"

Solution:
- Service Registry: a database of service name → current instances
- Services register on startup, deregister on shutdown
- Health checks remove dead instances automatically
```

### Two Discovery Patterns

```
Client-Side Discovery:
  Service A queries registry directly → gets list of Service B instances
  Service A load balances itself → calls one instance
  Used by: Netflix Eureka (with Ribbon load balancer)

Server-Side Discovery:
  Service A sends request to Load Balancer
  Load Balancer queries registry → picks instance → forwards request
  Service A doesn't know registry exists
  Used by: AWS ALB, Kubernetes Service
```

### Kubernetes Service Discovery

```yaml
# Every service gets a DNS name automatically
# payment-service → resolves to cluster IP → routes to healthy pods

apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment
  ports:
    - port: 80

# Order service can now call:
# http://payment-service/charge
# Kubernetes DNS resolves, routes to any healthy payment pod
```

---

## 4.5 Database: SQL vs NoSQL

### SQL (Relational Databases)

```
Structure: Tables with rows and columns, strict schema
ACID: Atomicity, Consistency, Isolation, Durability
Examples: PostgreSQL, MySQL, SQLite, Oracle

Best for:
- Financial transactions (ACID required)
- Complex relationships (users, orders, products all linked)
- Reporting and analytics (flexible queries with JOINs)
- When data structure is well-defined and won't change much

Example: E-commerce orders, user accounts, inventory
```

**SQL Example:**
```sql
-- Complex query that NoSQL can't do easily
SELECT
    u.name,
    COUNT(o.id) AS order_count,
    SUM(o.total) AS total_spent
FROM users u
JOIN orders o ON o.user_id = u.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
WHERE p.category = 'electronics'
  AND o.created_at > '2024-01-01'
GROUP BY u.id
HAVING total_spent > 500
ORDER BY total_spent DESC;
```

### NoSQL Databases

```
Key-Value:   Redis, DynamoDB     → Sessions, cache, leaderboards
Document:    MongoDB, Firestore  → User profiles, product catalogs
Wide-Column: Cassandra, HBase    → Time-series, activity feeds
Graph:       Neo4j, Neptune      → Social networks, recommendation graphs
Search:      Elasticsearch       → Full-text search, log analytics

Best for:
- Horizontal scale (shard across many nodes)
- Flexible/changing schema
- Specific access patterns (all data for user X in one query)
- High write throughput (Cassandra writes millions/sec)
- Huge datasets (petabyte scale)
```

**MongoDB Example:**
```javascript
// All user data in one document — no JOIN needed
db.users.findOne({ _id: "usr-123" })
// Returns:
{
  "_id": "usr-123",
  "name": "Alice",
  "email": "alice@example.com",
  "preferences": { "currency": "EUR", "notifications": true },
  "recentOrders": [
    { "id": "ord-456", "total": 99.99, "status": "delivered" }
  ]
  // Everything in one document = one DB read
}
```

### Decision Framework

```
Use SQL when:                      Use NoSQL when:
─────────────────────────────      ──────────────────────────────
Relationships between entities     Massive horizontal scale needed
ACID transactions required         Flexible/evolving schema
Complex queries needed             Single-entity access patterns
Moderate scale (<100k writes/sec)  Millions of writes per second
Team knows SQL well                Specific data model fits (graph, time-series)
```

---

## 4.6 Caching with Redis

### What It Is

An in-memory data store used as a fast lookup layer in front of slower data sources (databases, external APIs).

### Why We Use It

```
Database read: 5-50ms (disk I/O, query planning, network)
Redis read: 0.1-1ms (in-memory, no query planning)
→ 50x-500x faster

With caching:
  80% of reads hit cache (< 1ms) → database sees only 20% of reads
  → 5x less database load → no need to scale database as fast
```

### Cache Patterns

```python
# Pattern 1: Cache-Aside (most common)
def get_user(user_id: str) -> User:
    # 1. Check cache first
    cached = redis.get(f"user:{user_id}")
    if cached:
        return User.parse_raw(cached)  # Cache HIT

    # 2. Cache miss: fetch from DB
    user = db.query("SELECT * FROM users WHERE id = ?", [user_id])

    # 3. Store in cache with TTL (expire after 1 hour)
    redis.setex(f"user:{user_id}", 3600, user.json())

    return user  # Cache MISS, but now populated
```

```python
# Pattern 2: Write-Through (update cache on every write)
def update_user(user_id: str, data: dict) -> User:
    user = db.update("users", user_id, data)
    redis.setex(f"user:{user_id}", 3600, user.json())  # Keep cache fresh
    return user

# Pattern 3: Write-Behind (async DB write)
def update_user_fast(user_id: str, data: dict):
    redis.setex(f"user:{user_id}", 3600, json.dumps(data))  # Write to cache
    queue.enqueue("persist_user", user_id, data)  # Async DB write
    # Fast response, DB updated eventually
```

### What to Cache (and What Not To)

```
GOOD candidates for caching:
✓ Frequently read, rarely changed: product details, user profiles
✓ Expensive to compute: aggregated stats, recommendation scores
✓ Session data: user authentication state
✓ Rate limit counters: per-user request counts

BAD candidates for caching:
✗ Frequently changing data: stock prices, real-time inventory
✗ User-specific sensitive data (carefully consider security)
✗ Large data that doesn't fit in memory budget
✗ Data where stale reads have serious consequences (account balance)
```

### Redis Beyond Caching

```python
# Session storage
redis.setex(f"session:{token}", 3600, user_id)

# Rate limiting (atomic counter)
count = redis.incr(f"rate:{user_id}:{minute}")
if count == 1:
    redis.expire(f"rate:{user_id}:{minute}", 60)
if count > 100:
    raise RateLimitError()

# Distributed lock (only one process can run at a time)
acquired = redis.set("lock:cron-job", "1", nx=True, ex=300)

# Sorted set leaderboard
redis.zadd("leaderboard", {"player_alice": 1500, "player_bob": 2100})
top_10 = redis.zrevrange("leaderboard", 0, 9, withscores=True)

# Pub/Sub messaging
redis.publish("notifications", json.dumps({"user_id": "123", "msg": "Your order shipped!"}))
```

---

## 4.7 Message Queue (Kafka / RabbitMQ)

### What It Is

A durable buffer between services. Producer writes a message; consumer reads and processes it independently.

### Why We Use It

```
Without queue (synchronous):
  POST /order → Order Service → calls Inventory (200ms) → calls Payment (500ms) → calls Email (300ms)
  Total: 1000ms+ blocking. If any service is down → order fails.

With queue (async):
  POST /order → Order Service saves order → puts message in queue → returns in 50ms
  Workers consume queue:
    Inventory Worker: processes 200ms
    Payment Worker: processes 500ms   ← all happening in parallel
    Email Worker: processes 300ms
  Total: 50ms response + async processing. Any worker can fail/retry independently.
```

### Kafka vs RabbitMQ

```
RabbitMQ:
  Message broker: routes messages, deletes on successful consumption
  Push-based: broker pushes messages to consumers
  Perfect for: task queues, RPC, routing complex messages
  Message retention: deleted after consumption
  Scale: up to ~100,000 msg/sec

Kafka:
  Distributed event log: messages retained for configurable time (days/weeks)
  Pull-based: consumers pull at their own rate
  Perfect for: event streaming, audit logs, replay, high throughput
  Message retention: retained even after consumption (replay possible)
  Scale: millions of msg/sec
  Used by: LinkedIn, Netflix, Uber, Airbnb
```

### Kafka Core Concepts

```
Topic: named stream of events ("order-events", "payment-events")
Partition: a topic split into N ordered logs (parallelism unit)
Offset: position of a consumer in a partition
Consumer Group: N consumers sharing work on a topic (each partition → one consumer)

                     Topic: "order-events"
           ┌──────────────────────────────────────────┐
Partition 0│ msg1 │ msg2 │ msg5 │ msg8 │ msg11 │ ... │
Partition 1│ msg3 │ msg4 │ msg6 │ msg9 │ msg12 │ ... │
Partition 2│ ─── │ ─── │ msg7 │ msg10│ msg13 │ ... │
           └──────────────────────────────────────────┘
                                        ↑              ↑
                               Consumer A          Consumer B
                               (reads P0, P1)     (reads P2)
```

**Kafka producer example:**
```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['kafka:9092'],
    value_serializer=lambda v: json.dumps(v).encode()
)

def place_order(order):
    db.save(order)  # Save to DB first
    producer.send('order-events', {
        'event_type': 'order.placed',
        'order_id': order.id,
        'user_id': order.user_id,
        'total': float(order.total)
    })
    return {"status": "accepted", "order_id": order.id}
```

---

## 4.8 CDN (Content Delivery Network)

### What It Is

A globally distributed network of servers (edge nodes) that cache and serve content from locations geographically close to users.

### Why We Use It

```
Without CDN:
  User in Tokyo → requests image → hits server in Frankfurt → 200ms latency

With CDN:
  User in Tokyo → requests image → hits CDN edge node in Tokyo → 5ms latency
                                (first request fetches from Frankfurt, caches locally)
                                (all subsequent Tokyo requests: 5ms)
```

### What CDN Caches

```
Static assets (long TTL — days/weeks):
  - JavaScript bundles (app.abc123.js — filename includes hash)
  - CSS files
  - Images, fonts, videos
  - PDF downloads

Semi-dynamic (medium TTL — minutes/hours):
  - Product catalog pages
  - News articles
  - API responses for public data (top products, trending posts)

Dynamic (no cache / short TTL — seconds):
  - User-specific data
  - Real-time inventory
  - Authentication responses
```

### Cache Headers

```http
# Tell CDN to cache for 1 year (versioned asset — hash in filename)
Cache-Control: public, max-age=31536000, immutable

# Tell CDN to revalidate every hour
Cache-Control: public, max-age=3600, s-maxage=3600

# Don't cache (user-specific or real-time)
Cache-Control: private, no-store
```

### Real-World Example

**Netflix:** Video content is pre-positioned at CDN edge nodes worldwide before shows release. When millions of people watch a show premiere simultaneously, they're all served from their nearest edge node — not from Netflix's origin servers.

---

## 4.9 Rate Limiting

### What It Is

Controlling how many requests a client can make in a time period. Returns HTTP 429 (Too Many Requests) when limit is exceeded.

### Why We Use It

```
Protection against:
1. DDoS attacks (100,000 requests/second from one IP)
2. Abusive clients (scraping your entire product catalog)
3. Runaway bugs (a client with infinite retry loop)
4. Fair resource distribution (one user shouldn't starve others)
5. Cost control (ML inference APIs are expensive per call)
```

### Algorithms

```
Fixed Window Counter: count in 60s window. Problem: 100 req at 00:59 + 100 at 01:00 = 200 in 2 seconds.

Sliding Window: track per-second within rolling window. More accurate, more memory.

Token Bucket: bucket holds N tokens, refills at R/second.
  Each request consumes 1 token.
  Allows bursting up to N, enforces average rate R.
  → Best for APIs that need to allow short bursts.

Leaky Bucket: requests enter a queue, process at constant rate.
  Smooths traffic completely. No bursts.
  → Best for rate-shaping outbound requests.
```

**Token Bucket in Redis:**
```python
def is_allowed(user_id: str, limit: int = 100, window: int = 60) -> bool:
    key = f"rate:{user_id}:{int(time.time() // window)}"

    pipe = redis.pipeline()
    pipe.incr(key)
    pipe.expire(key, window)
    count, _ = pipe.execute()

    return count <= limit

# Usage
@app.before_request
def check_rate_limit():
    user_id = get_current_user_id()
    if not is_allowed(user_id):
        return jsonify({"error": "Rate limit exceeded"}), 429
```

**Rate limit response headers:**
```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1718467260
Retry-After: 47
```

---

# 5. Communication Patterns

## 5.1 Synchronous vs Asynchronous

```
SYNCHRONOUS:
  Caller sends request → waits → receives response → continues

  Client ────── request ──────▶ Server
         ◀───── response ───── (5-500ms later)

  Client is BLOCKED during this time.

ASYNCHRONOUS:
  Caller sends request → continues immediately → result comes later

  Client ────── message ──────▶ Message Queue ──▶ Worker
         ◀──── "accepted" ───── (immediately)
  [Later:]
  Worker processes message → result stored or event fired
```

## 5.2 When to Use Synchronous vs Asynchronous

| Scenario | Synchronous | Asynchronous |
|---|---|---|
| User needs result NOW | ✓ Must use sync | ✗ Can't use async |
| Long-running task | ✗ Blocks user | ✓ Return job ID immediately |
| Multiple services must react to same event | ✗ Must call each one | ✓ Publish event once |
| Some services might be temporarily offline | ✗ Request fails | ✓ Queue buffers it |
| Strong consistency required | ✓ Sync is easier | ✗ Eventual only |
| Very high throughput needed | ✗ Back-pressure harder | ✓ Queue absorbs spikes |

## 5.3 REST vs Messaging

```
REST (Request/Response):
  ✓ Simple, universally understood
  ✓ Immediate response, easy to debug
  ✓ Standard tooling (curl, browsers, Postman)
  ✗ Tight coupling (both services must be up)
  ✗ No built-in retry/durability
  Use: Client-to-service, service-to-service for synchronous operations

Messaging (Kafka, RabbitMQ, SQS):
  ✓ Loose coupling (producer/consumer don't know each other)
  ✓ Built-in retry and durability
  ✓ Fan-out (one message → many consumers)
  ✓ Buffer spikes (queue absorbs burst)
  ✗ Eventual consistency only
  ✗ Harder to debug (no direct request trace)
  Use: Service-to-service async workflows, events, notifications
```

## 5.4 Event-Driven Communication

```
3 Levels of Event-Driven:

Level 1 — Notification Events:
  "Something happened." No data included.
  Consumer must call back to get details.
  Pros: Tiny events. Cons: Extra call to get data.
  Example: { "event": "order.placed", "order_id": "ord-123" }

Level 2 — Event-Carried State Transfer:
  "Something happened, here's the full relevant data."
  Consumer doesn't need to call back.
  Example: Full order object in the event
  Pros: No extra call. Cons: Larger events, schema evolution harder.

Level 3 — Event Sourcing:
  The event log IS the system's state.
  Current state = replay of all past events.
  Pros: Full audit trail, time travel. Cons: Complex, large log.
  Used by: Financial systems, audit-heavy applications
```

---

# 6. Data & Scaling Concepts

## 6.1 Horizontal vs Vertical Scaling

```
VERTICAL SCALING (Scale Up):
  Add more resources to the SAME machine.
  2 CPU cores → 16 CPU cores
  8GB RAM → 256GB RAM

  ┌────────────────┐       ┌──────────────────────────┐
  │ Server         │  →→→  │ Bigger Server            │
  │ 4 CPU, 16GB    │       │ 32 CPU, 256GB            │
  └────────────────┘       └──────────────────────────┘

  ✓ No code changes needed
  ✓ Simple — one server
  ✗ Hard ceiling (can't buy infinitely large server)
  ✗ Still a single point of failure
  ✗ Downtime during upgrade

HORIZONTAL SCALING (Scale Out):
  Add MORE machines running the SAME software.

  ┌──────────┐       ┌──────────┐ ┌──────────┐ ┌──────────┐
  │ Server 1 │  →→→  │ Server 1 │ │ Server 2 │ │ Server 3 │
  └──────────┘       └──────────┘ └──────────┘ └──────────┘
                      (behind a load balancer)

  ✓ Unlimited scale (add more servers)
  ✓ No single point of failure
  ✓ Cost-efficient (commodity hardware)
  ✗ Application must be stateless (or state in shared store)
  ✗ More operational complexity
  ✗ Distributed system problems (coordination, consistency)
```

**Rule of thumb:**
```
Databases → Start vertical, add read replicas, then shard
App servers → Horizontal from the start (make them stateless)
Caches → Horizontal (Redis Cluster)
```

## 6.2 Sharding

Sharding = splitting a database horizontally across multiple servers. Each server holds a **subset** of the data.

```
Without sharding (all data on one DB):
  Users table: 1 billion rows → single DB can't handle writes

With sharding (data split across servers):
  Shard 1: user_id 0-249,999,999   (250M users)
  Shard 2: user_id 250M-499,999,999
  Shard 3: user_id 500M-749,999,999
  Shard 4: user_id 750M-999,999,999

Query for user_id=300,000,000:
  App: 300M / 250M = shard 2 → query shard 2 directly
```

**Shard key selection is critical:**

```python
# Bad shard key: timestamp
# All new data goes to the latest shard → HOT SHARD

# Bad shard key: status (active/inactive)
# 95% of users "active" → one shard gets all the work

# Good shard key: user_id (high cardinality, evenly distributed)
def get_shard(user_id: int, num_shards: int = 4) -> int:
    return hash(user_id) % num_shards

# Consistent hashing: better for resharding
# Adding a shard only moves 1/N of existing data
```

**Sharding challenges:**
```
Cross-shard queries: SELECT * FROM orders WHERE status='pending'
  → Must query ALL shards and merge results (expensive)
  → Solution: Denormalize or use a separate analytics DB

Distributed transactions: User creates order that spans user + order shards
  → Use saga pattern, avoid cross-shard transactions

Resharding: Growing from 4 to 8 shards requires moving half your data
  → Solution: Consistent hashing, virtual nodes
```

## 6.3 Replication

Replication = maintaining copies of data on multiple servers.

```
PRIMARY-REPLICA REPLICATION:
                    Writes go here
                         │
                    ┌────▼────┐
           ┌────────│ Primary │────────┐
           │        └─────────┘        │
     replicates                   replicates
           │                           │
    ┌──────▼──────┐             ┌──────▼──────┐
    │  Replica 1  │             │  Replica 2  │
    │ (reads)     │             │ (reads)     │
    └─────────────┘             └─────────────┘

Reads distributed across replicas → scales read throughput
Writes go to primary only → single write bottleneck (but enough for most cases)
If primary fails → promote replica to primary (failover)
```

**Sync vs Async Replication:**
```
SYNCHRONOUS:
  Primary writes → waits for replica to confirm → confirms to client
  Guarantees: zero data loss on failover
  Cost: slower writes (waits for replica)
  Use: financial systems, anywhere data loss is catastrophic

ASYNCHRONOUS:
  Primary writes → confirms to client immediately → replicates in background
  Guarantees: fast writes
  Risk: data loss if primary crashes before replica syncs (seconds of data)
  Use: most web applications (acceptable trade-off)
```

## 6.4 CAP Theorem

**The theorem:** In a distributed system, during a network partition, you can guarantee only ONE of:
- **Consistency** — every read sees the most recent write
- **Availability** — every request receives a response (not an error)

**Partition Tolerance** (P) is not optional — network failures always happen. So the real choice is **C vs A during a partition.**

```
Simple analogy:
Two ATMs in a network. Network between them goes down.

C choice (CP — banks choose this):
  "Better to reject a withdrawal than risk giving out money twice"
  → ATM B refuses all transactions until network restored
  → Available: NO. Consistent: YES.

A choice (AP — DNS chooses this):
  "Better to give slightly outdated info than refuse to answer"
  → ATM B serves from its last known state
  → Available: YES. Consistent: MAYBE (stale data possible).
```

**Common systems:**
```
CP Systems (Consistency + Partition Tolerance):
  PostgreSQL, MySQL, ZooKeeper, etcd, MongoDB (default config)
  "Reject requests rather than serve stale data"

AP Systems (Availability + Partition Tolerance):
  Cassandra, CouchDB, DynamoDB (default), DNS
  "Keep serving, even if data might be slightly stale"
```

## 6.5 Consistency vs Availability (Extended)

```
STRONG CONSISTENCY:
  After a write, all reads immediately reflect the write.
  Implementation: Wait for all replicas before confirming (sync replication)
  Trade-off: Higher latency, lower availability during partition
  Use: Account balances, inventory counts, anything where stale = wrong

EVENTUAL CONSISTENCY:
  After a write, replicas will converge to the same value — eventually.
  Implementation: Async replication, replicas catch up in milliseconds-seconds
  Trade-off: Reads may return stale data briefly
  Use: Social media feeds, product recommendations, search indexes
  "Is your friend seeing your Facebook post 2 seconds late? Probably fine."

READ-YOUR-WRITES:
  Weaker than strong, stronger than eventual.
  You always see your own writes.
  Others may see stale data.
  Implementation: Route your reads to the node you wrote to
  Use: Social profiles (you always see your own updates)
```

---

# 7. Real System Design Example

## Designing a URL Shortener (bit.ly)

### Step 1: Clarify Requirements

**Functional Requirements:**
```
1. User submits a long URL → get a short URL (e.g., sho.rt/abc123)
2. User visits short URL → redirected to original long URL
3. Custom aliases: user can choose short code (e.g., sho.rt/my-link)
4. Analytics: count how many times a short URL was visited
5. Expiry: short URLs can have an expiration date
```

**Non-Functional Requirements:**
```
Scale:
  - 100 million URLs created per day
  - 10 billion URL redirects per day (~115,000 redirects/sec)
  - Read:Write ratio ≈ 100:1 (heavy read)

Performance:
  - Redirect latency: < 50ms (user already waiting to be redirected)
  - Creation latency: < 200ms

Availability:
  - 99.99% uptime (core business function)

Storage:
  - Keep URLs for 5 years
  - ~500 bytes per record × 100M/day × 365 × 5 ≈ ~90TB
```

### Step 2: High-Level Architecture

```
                         ┌───────────────────────────────────────┐
                         │              CDN                       │
                         │  (caches popular redirects globally)   │
                         └───────────────────┬───────────────────┘
                                             │ (cache miss)
                         ┌───────────────────▼───────────────────┐
                         │           API Gateway                  │
                         │  Rate limiting, auth, routing          │
                         └──────────────┬────────────────────────┘
                                        │
              ┌─────────────────────────┼──────────────────────────┐
              │                         │                          │
    ┌─────────▼──────────┐  ┌───────────▼──────────┐  ┌──────────▼──────────┐
    │   Create URL API   │  │  Redirect Service    │  │  Analytics Service  │
    │  (POST /shorten)   │  │  (GET /:code)        │  │  (async, best-effort)│
    └─────────┬──────────┘  └───────────┬──────────┘  └──────────┬──────────┘
              │                         │                          │
    ┌─────────▼──────────┐  ┌───────────▼──────────┐  ┌──────────▼──────────┐
    │   ID Generator     │  │      Redis Cache     │  │     Kafka Topic     │
    │  (unique short     │  │  (hot URLs in RAM,   │  │  ("url.clicked")    │
    │   codes)           │  │   ~1ms lookup)       │  └──────────┬──────────┘
    └─────────┬──────────┘  └───────────┬──────────┘             │
              │                  miss   │                ┌────────▼────────┐
    ┌─────────▼──────────────────────────▼─────────────▶│  ClickHouse DB  │
    │              Primary Database (PostgreSQL)         │  (analytics)    │
    │    url_id | short_code | long_url | expiry | user  └─────────────────┘
    └────────────────────────────────────────────────────┘
               │
    ┌──────────▼──────────┐
    │   Read Replicas      │  ← Redirect Service reads from here
    │   (3x replicas)      │
    └─────────────────────┘
```

### Step 3: Key Components & Decisions

#### Short Code Generation

```python
# Option A: Random (simple, collision risk)
import secrets
def generate_code(length=7):
    chars = string.ascii_letters + string.digits  # 62 chars
    return ''.join(secrets.choice(chars) for _ in range(7))
# 62^7 = 3.5 trillion possible codes → collision probability tiny

# Option B: Hash + truncate
def generate_code(long_url: str) -> str:
    hash_value = hashlib.md5(long_url.encode()).hexdigest()
    return base62_encode(int(hash_value[:8], 16))[:7]
# Pro: same URL always gets same code
# Con: collisions when different URLs produce same first 7 chars

# Option C: Auto-increment ID + Base62 encode (BEST for scale)
def id_to_code(id: int) -> str:
    chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    result = []
    while id > 0:
        result.append(chars[id % 62])
        id //= 62
    return ''.join(reversed(result))
# ID 1000000 → "4c92" (short, unique, no collision)
# Use distributed ID generator (Snowflake) for multi-server scale
```

#### Redirect Flow (Critical Path — must be fast)

```python
@app.get("/{short_code}")
async def redirect(short_code: str):
    # 1. Check Redis cache first (~0.1ms)
    cached_url = await redis.get(f"url:{short_code}")

    if cached_url:
        # 2a. Cache HIT: fire analytics event async, return redirect
        asyncio.create_task(track_click(short_code))  # Non-blocking
        return RedirectResponse(url=cached_url, status_code=301)

    # 2b. Cache MISS: check database (read replica)
    record = await db_replica.fetch_one(
        "SELECT long_url, expires_at FROM urls WHERE short_code = ?",
        [short_code]
    )

    if not record:
        raise HTTPException(404, "URL not found")

    if record["expires_at"] and record["expires_at"] < datetime.utcnow():
        raise HTTPException(410, "URL expired")

    # 3. Populate cache (cache for 24 hours)
    await redis.setex(f"url:{short_code}", 86400, record["long_url"])

    # 4. Track analytics asynchronously
    asyncio.create_task(track_click(short_code))

    return RedirectResponse(url=record["long_url"], status_code=301)
```

#### Why 301 vs 302 Redirect?

```
301 Permanent: Browser caches the redirect. Subsequent visits bypass server.
  ✓ Saves bandwidth, reduces server load
  ✗ Analytics undercounted (browser skips our server on repeat visits)

302 Temporary: Browser always goes to server.
  ✓ Every visit tracked for analytics
  ✗ More server load
  ✗ CDN can't cache

Decision: Use 302 if analytics matter (most URL shorteners do). 301 only for CDN caching strategy.
```

#### Database Schema

```sql
CREATE TABLE urls (
    id          BIGINT PRIMARY KEY,      -- auto-increment, basis for short code
    short_code  VARCHAR(10) NOT NULL,
    long_url    TEXT NOT NULL,
    created_by  BIGINT REFERENCES users(id),
    created_at  TIMESTAMP DEFAULT NOW(),
    expires_at  TIMESTAMP,
    is_custom   BOOLEAN DEFAULT FALSE,
    is_active   BOOLEAN DEFAULT TRUE,

    UNIQUE(short_code)
);

CREATE INDEX idx_short_code ON urls(short_code);  -- Primary lookup
CREATE INDEX idx_user_urls ON urls(created_by, created_at DESC);  -- "my links" page

CREATE TABLE clicks (
    id          BIGINT PRIMARY KEY,
    short_code  VARCHAR(10) NOT NULL,
    clicked_at  TIMESTAMP DEFAULT NOW(),
    ip_address  INET,
    user_agent  TEXT,
    referrer    TEXT,
    country     VARCHAR(2)  -- from IP geolocation
);
-- Note: This table will be huge. Consider ClickHouse or BigQuery for analytics.
```

#### Scaling Strategy

```
Current bottlenecks (in order of hitting them):

1. Redirect reads at 115,000 RPS:
   → Redis cache with 99% hit rate → 1,150 DB reads/sec → handles fine

2. Database read load:
   → 3 PostgreSQL read replicas handle remaining reads
   → Add replicas as needed

3. Cache capacity:
   → Most URLs accessed infrequently (long tail)
   → Cache only hot URLs (top 20% of URLs = 80% of traffic)
   → Redis eviction: LRU policy

4. URL write load (100M/day = ~1,200 writes/sec):
   → Single PostgreSQL primary handles up to ~10,000 writes/sec easily

5. Analytics (10 billion events/day):
   → Write to Kafka (high throughput, non-blocking)
   → Stream processor aggregates into ClickHouse
   → Never write analytics to main DB (would kill it)

6. At 10x scale (1 trillion events/day):
   → Shard the DB by short_code hash
   → Cassandra for URL store (optimized for high read throughput)
   → Global deployment (CDN + regional databases)
```

---

# 8. Interview Tips & Talking Points

## 8.1 How to Structure a System Design Answer

**The RADIO Framework:**

```
R — Requirements clarification (2-3 min)
A — API design (2 min)
D — Data model (2 min)
I — Infrastructure/High-level design (5-10 min)
O — Optimization / deep-dives (5-10 min)
```

**Step-by-step walkthrough:**

```
1. REQUIREMENTS (never skip this)
   "Before I start designing, let me clarify requirements.
    Functional: What must the system do?
    Non-functional: What scale? What latency? What availability?
    Let me make assumptions if you don't know: I'll assume 1M DAU, 100:1 read:write..."

2. ESTIMATION (shows you think about scale)
   "100M users × 10 actions/day = 1B events/day = ~12,000 events/sec
    Each event 200 bytes → 200GB/day → 70TB/year
    Peak traffic: 3x average = ~36,000 events/sec"

3. HIGH-LEVEL DESIGN (draw the boxes)
   Start with the client, go right: Client → CDN → LB → API → DB
   Then add supporting infrastructure: Cache, Queue, etc.

4. DEEP DIVE (interviewer will guide)
   Be ready to drill into: DB schema, API design, one specific component
   Typical: "How would you handle X scaling? What if Y fails?"

5. TRADE-OFFS (always mention)
   "I chose PostgreSQL over Cassandra here because we need ACID transactions.
    The trade-off is we're limited to ~100k writes/sec on one node,
    but we won't hit that for 2 years given our projected growth."
```

## 8.2 Common Mistakes in System Design Interviews

**Mistake 1: Starting to design before clarifying requirements**
```
❌ Immediately drawing boxes: "OK so we have microservices..."
✓ "Before I start, can I ask a few questions about scale and requirements?"
```

**Mistake 2: Jumping to complex solutions**
```
❌ Starting with Kafka, Redis, Cassandra, Kubernetes for a simple system
✓ Start simple: "First let me design for 10,000 users, then we can discuss scaling"
```

**Mistake 3: Ignoring trade-offs**
```
❌ "We'll use Redis for caching" (no justification)
✓ "Redis gives us sub-millisecond reads. The trade-off is cost and cache invalidation complexity. For this use case it's worth it because..."
```

**Mistake 4: Not discussing failure scenarios**
```
❌ Only the happy path
✓ "What happens if Redis goes down? The cache-aside pattern means we fall back to the database — slower but still works."
```

**Mistake 5: Ignoring the database**
```
❌ "We'll just put the data in a database"
✓ Schema design, indexing strategy, which DB type and why, replication strategy
```

**Mistake 6: Silence**
```
❌ Drawing silently for 5 minutes
✓ Think out loud. "I'm considering two options here: Option A is simpler but doesn't scale well. Option B is more complex but handles 10x the load. Let me go with Option B..."
```

## 8.3 Phrases That Impress Interviewers

```python
# Shows you think in trade-offs
"The trade-off between X and Y here is..."
"I'd choose X over Y because for our requirements, [reason]"

# Shows you think about failure
"What happens if this component goes down?"
"We'd need a circuit breaker here to prevent cascade failures"

# Shows you think about scale
"This works for 1M users, but at 100M we'd need to..."
"The database will be our bottleneck first. We'd add read replicas then cache"

# Shows real-world experience
"Stripe uses this exact pattern for idempotency keys"
"Kafka is designed for exactly this use case — Netflix uses it for..."

# Shows you're not over-engineering
"I'd start with a monolith here. The team is small and we don't know the domain well enough to split it correctly yet"
"We don't need Kubernetes for this yet — a simple auto-scaling group achieves the same goal"
```

## 8.4 Architecture Interview — Talking About Patterns

**How to describe an architecture decision:**
```
1. State the choice: "I'd use Event-Driven Architecture for notifications"
2. State the problem it solves: "Because 8 different services need to react to the same events"
3. Explain how: "Order Service emits events to Kafka. Email, SMS, Push, Analytics each subscribe independently"
4. State the trade-off: "The downside is eventual consistency and harder debugging. We'd add distributed tracing with Jaeger"
5. Alternative considered: "I considered REST calls from Order Service to each consumer, but that's tight coupling — adding a 9th consumer requires touching Order Service"
```

---

# 9. Cheat Sheet (Quick Revision)

---

## 9.1 Architecture Patterns — When to Use

| Pattern | Use When | Avoid When |
|---|---|---|
| **Monolith** | Small team, early stage, simple domain | Large teams, drastically different scaling needs |
| **Microservices** | Multiple large teams, independent scaling needed | Small teams, early stage, unknown domain |
| **Layered** | Standard CRUD backend, clear layer responsibilities | Complex domain with rich business logic |
| **Clean/Hexagonal** | Complex domain, long-lived app, lots of infrastructure switching | Simple CRUD, small team, time pressure |
| **Event-Driven** | Multiple consumers of same event, decoupling needed | Needs strong consistency, small system |
| **Serverless** | Variable traffic, event-triggered, no ops team | Long-running, low latency, high sustained traffic |

---

## 9.2 Building Blocks — Quick Reference

| Component | What it does | Key trade-off |
|---|---|---|
| **API Gateway** | Single entry, auth, rate limit, routing | SPOF if not clustered |
| **Load Balancer** | Distribute traffic, remove dead servers | Adds a hop |
| **Reverse Proxy** | SSL, caching, compression, security | Configuration complexity |
| **Service Discovery** | Find service instances dynamically | Registry must be HA |
| **SQL DB** | ACID, joins, complex queries | Hard to scale writes horizontally |
| **NoSQL DB** | Scale horizontally, flexible schema | No ACID, no joins |
| **Redis Cache** | Sub-millisecond reads, session, rate limit | Memory expensive, cache invalidation hard |
| **Message Queue** | Async, decoupling, retry, fan-out | Eventual consistency only |
| **CDN** | Low latency globally, reduce origin load | Only for cacheable content |
| **Rate Limiting** | Protection, fair use, cost control | Adds latency to every request |

---

## 9.3 Key Comparisons

### SQL vs NoSQL (Decision Guide)

```
If you answer YES to any of these → use SQL:
  □ Do you need ACID transactions across multiple entities?
  □ Do you need complex JOINs and ad-hoc queries?
  □ Is your schema stable and well-defined?
  □ Is the scale below 100k writes/second?

If you answer YES to any of these → consider NoSQL:
  □ Do you need to scale writes to millions per second?
  □ Is the schema evolving rapidly or document-like?
  □ Is the access pattern always by a single key (user_id)?
  □ Is the dataset petabyte-scale?
  □ Do you need a specific data model (graph, time-series)?
```

### Sync vs Async

```
Use SYNCHRONOUS when:
  □ Caller needs the result to continue
  □ Response time matters to the user experience
  □ Strong consistency is required
  □ Simple request/response pattern

Use ASYNCHRONOUS when:
  □ Operation takes > 500ms (email, video encoding, reports)
  □ Multiple services need to react to the same event
  □ Service might be temporarily unavailable
  □ You need to handle traffic spikes gracefully
  □ You can accept eventual consistency
```

### Monolith vs Microservices

```
MONOLITH is better when:
  □ Team < 10 engineers
  □ Product < 2 years old (still finding product-market fit)
  □ Domain not well understood (boundaries unclear)
  □ Fast iteration speed is the priority

MICROSERVICES better when:
  □ Team > 50 engineers (multiple autonomous teams)
  □ Clear domain boundaries exist
  □ Services have drastically different scaling needs
  □ Independent deployment frequency is high
  □ Team has DevOps maturity (tracing, CI/CD per service)
```

---

## 9.4 Scaling Decision Tree

```
Traffic grows. What breaks first?

1. App server CPU/memory high?
   → Add more app servers (horizontal scale)
   → Ensure they're STATELESS (session in Redis)

2. Database reads slow?
   → Add Redis cache (hit cache, not DB)
   → Add read replicas

3. Database writes slow?
   → Optimize queries (add indexes, use EXPLAIN ANALYZE)
   → If still slow: partition table, then shard

4. Cache full / expensive?
   → Evict LRU keys
   → Redis Cluster for horizontal cache scale

5. External API calls slow?
   → Add circuit breaker
   → Make calls async (queue)
   → Cache their responses

6. Globally distributed users with latency?
   → CDN for static assets
   → Edge caching for API responses
   → Multi-region deployment

7. Single service becomes a bottleneck?
   → Extract it to a dedicated service
   → Scale that service independently
```

---

## 9.5 CAP Theorem Quick Reference

| System | Category | When partition happens |
|---|---|---|
| PostgreSQL | CP | Rejects requests to stay consistent |
| MySQL | CP | Rejects requests to stay consistent |
| Cassandra | AP | Continues serving (may return stale) |
| DynamoDB | AP (default) | Continues serving (may return stale) |
| ZooKeeper | CP | Rejects requests |
| Redis | AP (default) | Continues serving |
| MongoDB | CP (default) | Rejects to stay consistent |

---

## 9.6 Key Definitions (1-2 Lines)

**API Gateway:** Single entry point for all clients; handles auth, rate limiting, routing, SSL termination.

**CAP Theorem:** Distributed systems can guarantee only two of: Consistency, Availability, Partition Tolerance. Since partitions always happen, real choice is C vs A.

**Circuit Breaker:** Pattern that stops calling a failing service after N failures, allowing it to recover instead of cascade-failing.

**CDN:** Globally distributed cache serving static/semi-static content from servers close to users.

**CQRS:** Command Query Responsibility Segregation — separate read and write models.

**Dead Letter Queue:** Where failed messages go after exhausting retries; prevents loss of messages.

**Event Sourcing:** Storing state as a sequence of events; current state = replay of all events.

**Horizontal Scaling:** Adding more machines (nodes) to handle load. Requires stateless services.

**Idempotency:** Same operation applied multiple times produces the same result as applying once.

**Load Balancer:** Distributes incoming requests across multiple servers; detects and removes unhealthy servers.

**Message Queue:** Durable buffer between producers and consumers for async communication.

**Microservices:** Architectural pattern where a system is split into small, independently deployable services.

**Rate Limiting:** Controlling how many requests a client can make in a time window.

**Read Replica:** Copy of a database that handles read traffic; primary handles writes only.

**Redis:** In-memory data store; used for caching, sessions, rate limiting, queues, pub/sub.

**Reverse Proxy:** Server in front of backend servers; adds SSL, caching, compression, security.

**Saga Pattern:** Distributed transaction pattern using compensating transactions for rollback.

**Service Discovery:** Mechanism for services to find each other's network addresses dynamically.

**Sharding:** Splitting a database horizontally across multiple servers; each holds a subset of data.

**Vertical Scaling:** Adding more CPU/RAM to an existing machine. Has a hard ceiling.

---

## 9.7 Interview Conversation Starters

```
Opening any system design:
"Before I jump in, let me clarify a few requirements..."

Estimating scale:
"Let me do a quick back-of-envelope: if we have X users doing Y..."

Making a decision:
"I'm going to choose X here because [reason]. The trade-off is [Y], which I think is acceptable because [Z]."

When you're unsure:
"I can think of two approaches here — let me walk through both and pick one..."

Adding depth:
"One thing I want to flag is what happens when [component] fails..."

Showing real-world awareness:
"This is similar to how [Company] handles [problem]..."
```

---

*This guide covers the foundations you need for backend architecture and system design interviews. The key is not to memorize — it's to understand the **WHY** behind each decision. Every technology choice is a trade-off. Every pattern solves a specific problem. Know the problem first, then the solution will make sense.*

*Practice by designing systems out loud: URL shortener, Instagram, Uber, Netflix. Time yourself. Get comfortable with ambiguity.*

*Good luck. 🚀*
