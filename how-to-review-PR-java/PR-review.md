

As a Java/Spring Boot developer, I would review a PR in layers rather than just looking
for formatting or obvious bugs.
A good Senior-level PR review is essentially:
Here is the complete approach I would use in a real team and in a Senior interview.
- First understand the PR before reading the code
Don't immediately start commenting line-by-line.
First understand:
What is the PR trying to do?
Look at:
PR title
## Description
## Jira/ticket
Acceptance criteria
Architecture/design document if available
Related PRs
Database migrations
API contract changes
Ask yourself:
For example:
Now you know what you're reviewing.
- Understand the scope
## Check:
Understand the change → verify correctness → check design → check
production behavior → check tests → check security/performance → check
maintainability → give focused feedback.
What problem is this PR solving?
"Add an endpoint to create shipments and asynchronously request a shipping label
from the carrier."

What files changed?
How many lines changed?
Which modules changed?
Are database changes included?
Are API contracts changed?
Are configuration changes included?
Are dependencies added?
Are tests changed?
A very important Senior-level question:
If the ticket says "add shipment creation", but the PR also changes authentication, refactors
20 services, changes database configuration, and introduces a new framework, that's a
review concern.
Not necessarily because the code is wrong—but because scope has expanded.
- Start with the architecture / flow
Before reviewing individual methods, understand the request flow.
For a Spring Boot API, I might trace:
If asynchronous:
Does the implementation match the scope of the ticket?
## Client
## ↓
## Controller
## ↓
## Service
## ↓
## Repository
## ↓
## Database
## Client
## ↓
## Controller
## ↓
## Service
## ↓
## Database
## ↓
## Kafka

Then ask:
For example, I don't want:
I'd expect something closer to:
- Correctness comes first
This is the most important part.
## Ask:
Look for:
## ↓
## Consumer
## ↓
## External Carrier
Does the PR put responsibilities in the correct layer?
@RestController
public class ShipmentController {
@PostMapping
public void create() {
// database logic
// Kafka logic
// carrier API call
// business logic
// DTO mapping
## }
## }
## Controller
## ↓
Application/Service layer
## ↓
Domain/business logic
## ↓
Repository / external clients
Does this code actually do what the requirement says?
## Java

Wrong conditions
instead of:
Or better, an enum.
Missing cases
For example:
What happens with:
Null handling
## Ask:
Can this value actually be null?
## Is
## Optional
being misused?
Can a nested property cause NPE?
## Example:
What happens if
customer
or
address
is null?
if (status != "DELIVERED")
if (!"DELIVERED".equals(status))
switch (status) {
case CREATED:
case SHIPPED:
## ...
## }
## CANCELLED
## FAILED
## DELIVERED
## RETURNED
order.getCustomer().getAddress().getCity()
## Java
## Java
## Java
## Java

- Check business logic
This is where a Senior engineer adds a lot of value.
Don't only ask:
## Ask:
## Example:
What if two requests arrive simultaneously?
Both could see:
and both create labels.
Now we have a business correctness problem.
You might need:
database constraint
idempotency
locking
state transition validation
unique key
depending on requirements.
- API review
For Spring Boot APIs, review:
"Does this compile?"
"Does this correctly implement the business rule?"
if (shipment.getStatus() == CREATED) {
createLabel();
## }
## CREATED
## Java

HTTP method
Is it actually:
appropriately?
## URL
Prefer meaningful resources:
rather than:
Request DTO
Don't expose JPA entities directly.
## Prefer:
Response DTO
Don't blindly return entities.
## GET
## POST
## PUT
## PATCH
## DELETE
POST /shipments
GET /shipments/{id}
POST /createShipment
public record CreateShipmentRequest(
String recipient,
String carrier
## ) {}
public record ShipmentResponse(
UUID id,
String trackingCode,
ShipmentStatus status
## ) {}
## Java
## Java

HTTP status
Check whether:
is appropriate.
For asynchronous processing, for example,
## 202 Accepted
might make more sense than
pretending the operation is completed.
- Input validation
Look for:
## Example:
## 200
## 201
## 202
## 204
## 400
## 401
## 403
## 404
## 409
## 422
## 429
## 500
## 503
@NotNull
@NotBlank
@Size
@Email
@Positive
@Min
@Max
public record CreateShipmentRequest(
@NotBlank
String recipient,
@NotBlank
## Java
## Java

And verify:
is actually used.
- Exception handling
Look for code like:
 Major concern.
## Or:
Also problematic.
## Ask:
Spring applications often use centralized handling:
String carrier
## ) {}
@Valid
@RequestBody
try {
## ...
} catch (Exception e) {
return null;
## }
catch (Exception e) {
e.printStackTrace();
## }
What should happen when this operation fails?
@RestControllerAdvice
public class GlobalExceptionHandler {
## }
## Java
## Java
## Java
## Java

## Check:
correct exception types
meaningful HTTP responses
no sensitive information leaked
logging
correlation/request IDs if applicable
- Transaction review
This is extremely important in Spring Boot.
If you see:
I'd immediately ask:
## Because:
can hold DB resources unnecessarily.
Also check:
transaction boundary
read-only transactions
rollback behavior
@Transactional
public void createOrder() {
saveOrder();
callExternalPaymentService();
savePayment();
## }
Should an external network call really happen inside this DB transaction?
## BEGIN TRANSACTION
## ↓
DB operation
## ↓
HTTP call
## ↓
slow external service
## ↓
## COMMIT
## Java

checked vs unchecked exceptions
transaction propagation
isolation level
lazy loading
self-invocation of
@Transactional
- Database / JPA review
This is a huge area for Java PRs.
N+1 queries
Watch for:
with lazy relationships.
Could generate:
Fetch strategy
## Check:
Don't blindly change everything to:
orders.forEach(order ->
order.getCustomer().getName()
## );
1 query for orders
## +
N queries for customers
@OneToMany
@ManyToOne
@OneToOne
FetchType.EAGER
## Java
## Java
## Java

That can create huge queries and unnecessary data loading.
## Queries
## Check:
indexes
joins
filtering
pagination
sorting
query complexity
unnecessary queries
missing constraints
If the PR adds:
I'd ask:
## Potentially:
But I wouldn't automatically demand that index without understanding the workload.
## 11. Pagination
This is a common PR problem.
## 
## SELECT *
FROM shipments
WHERE customer_id = ?
AND status = ?
ORDER BY created_at DESC
Do we have an appropriate index for this access pattern?
(customer_id, status, created_at)
## SQL
## SQL
## Java

for potentially thousands/millions of records.
Prefer appropriate pagination:
For very large/high-throughput datasets, also consider whether offset pagination is
appropriate versus keyset/cursor pagination.
- Database migrations
If schema changes:
## Check:
Flyway/Liquibase migration
backward compatibility
indexes
constraints
nullable/non-nullable changes
default values
migration ordering
production data impact
whether the migration locks a large table
For example:
Could fail on an existing populated table.
A Senior review asks:
repository.findAll();
Page<Shipment> findByCustomerId(
UUID customerId,
Pageable pageable
## );
ALTER TABLE shipment
ADD COLUMN tracking_code VARCHAR(255) NOT NULL;
How does this migration behave against production data?
## Java
## SQL

## 13. Concurrency
For backend systems, this is one of the most important areas.
Look for shared mutable state:
used by multiple requests.
## Remember:
So this is dangerous:
because requests can interfere with each other.
Prefer request data in:
- Thread pools / async code
If you see:
private int counter;
Spring singleton beans are shared between threads.
@Service
public class OrderService {
private Order currentOrder;
## }
method parameters
local variables
database
appropriate external state
## Java
## Java
## Java
## Java

inside a request:
 investigate.
## Ask:
Who owns this executor?
Is it reused?
Is it shut down?
What happens under high traffic?
What happens when tasks queue up?
Are exceptions captured?
Does the request wait?
Is there backpressure?
Also check:
Don't assume asynchronous code automatically improves performance.
- Kafka / messaging review
If Kafka is involved, check:
## Producer
What happens if publishing fails?
Is the DB update and Kafka event atomic?
Could DB succeed while Kafka fails?
This leads to the Transactional Outbox discussion.
Executors.newFixedThreadPool(100);
CompletableFuture
@Async
ExecutorService
parallelStream()
DB transaction
├── business data
└── outbox event
Outbox publisher
## Java

## Consumer
## Check:
duplicate messages
idempotency
retries
dead-letter topic
ordering
partitioning
consumer groups
offset handling
poison messages
Very important:
- External API calls
For something like:
check:
## Timeout
What happens if carrier never responds?
## Retry
Should we retry?
How many times?
With what backoff?
## Idempotency
## If:
## ↓
Kafka delivery does not magically mean your business operation happens exactly
once.
carrierClient.createLabel(...)
## Java

and we don't know whether the carrier created the label, blindly retrying could create two
labels.
That's a real production problem.
Circuit breaker
If the carrier is down, should we continue sending thousands of requests?
Rate limiting
What if carrier allows:
but we send:
## 17. Performance
Don't optimize everything.
Look for realistic bottlenecks:
## Database
## N+1
missing index
huge result sets
unnecessary queries
## Network
sequential external calls
large payloads
repeated calls
## CPU
expensive loops
unnecessary serialization
request → carrier
timeout
100 requests/minute
## 5,000/minute

inefficient algorithms
## Memory
loading millions of records
huge collections
unbounded caches
large files in memory
## 18. Caching
If Redis/cache is introduced, ask:
## Then:
What is cached?
## TTL?
eviction?
invalidation?
stale data acceptable?
cache stampede?
cache failure?
serialization?
key design?
memory limits?
The biggest cache question is often:
A cache should normally not become an unexpected single point of failure.
## 19. Security
For every PR ask:
## Authentication
Who can call this endpoint?
## Authorization
Can user A access user B's shipment?
Why do we need a cache?
What happens when the cache is unavailable?

For example:
Don't only check:
## Also:
Sensitive data
## Check:
passwords
tokens
API keys
customer credentials
## PII
payment information
## Never:
Also look for:
SQL injection
path traversal
insecure deserialization
sensitive data in responses
secrets committed to Git
## 20. Logging
## Ask:
## Bad:
GET /shipments/123
Does shipment 123 exist?
Does this authenticated user have permission to access shipment 123?
log.info("Carrier credentials: {}", credentials);
If this fails at 3 AM, can we understand what happened?
## Java

## Better:
But be careful not to log:
passwords
tokens
credentials
sensitive PII
Also consider:
structured logs
correlation IDs
request IDs
appropriate log levels
## 21. Observability
For production services, check:
## Metrics
## Examples:
## Logs
Can we understand failures?
## Tracing
log.error("Something went wrong");
log.error(
"Failed to create label for shipment {} with carrier {}",
shipmentId,
carrier,
exception
## );
shipment_creation_total
carrier_request_latency
carrier_request_failures
kafka_consumer_lag
## Java
## Java

Can we follow:
across distributed services?
## 22. Testing
Don't just check whether tests exist.
Check what they test.
Unit tests
Business logic.
Integration tests
Spring context + database + messaging where appropriate.
API tests
Request/response behavior.
Edge cases
## Examples:
Failure scenarios
A good backend test suite shouldn't only test:
It should test:
API → service → Kafka → consumer → carrier
null
empty
duplicate
invalid carrier
already processed
not found
timeout
external failure
happy path
happy path
+ invalid input

- Test quality
Watch for tests that simply mock everything:
and therefore don't test meaningful behavior.
Also check:
meaningful assertions
test naming
test isolation
flaky tests
unnecessary mocking
missing integration tests
- Code quality
Now look at maintainability.
## Check:
## Naming
## Bad:
## Better:
+ business failure
+ infrastructure failure
+ concurrency/idempotency where relevant
when(repository.findById(...))
when(client.call(...))
when(repository.save(...))
process()
handle()
doStuff()
data
result
## Java
## Java

Method size
Huge methods are difficult to reason about.
## Classes
Does one class have 15 responsibilities?
## Duplication
Repeated business logic may need extraction.
But don't automatically create:
for every 5-line method.
- SOLID principles
Review relevant principles, not mechanically.
## Single Responsibility
Does the class have too many responsibilities?
Open/Closed
Will adding another carrier require modifying 100
if/else
branches?
## Instead:
createShipmentLabel()
publishTrackingUpdate()
findActiveShipment()
## Factory
## Strategy
AbstractFactory
## Builder
HandlerFactory
## Manager
## Coordinator
interface CarrierClient {
Label createLabel(...);
## Java
## Java

with implementations:
## Dependency Inversion
Prefer abstractions where they provide meaningful flexibility.
Spring DI naturally helps here.
- Spring-specific review
Look for:
Dependency injection
Prefer constructor injection:
Bean scope
Are mutable singleton beans safe?
## Configuration
Don't hardcode:
Use configuration.
## }
DhlCarrierClient
DpdCarrierClient
InPostCarrierClient
@Service
public class ShipmentService {
private final ShipmentRepository repository;
public ShipmentService(ShipmentRepository repository) {
this.repository = repository;
## }
## }
private String carrierUrl = "https://...";
## Java
## Java

## Profiles
## Check:
Configuration properties
For structured configuration:
can be preferable to scattering
@Value
## .
- Dependency review
If someone adds:
ask:
## Check:
unnecessary libraries
vulnerable/outdated versions
duplicate functionality
licensing concerns
transitive dependencies
- API compatibility
If the PR changes:
application.yml
application-dev.yml
application-prod.yml
@ConfigurationProperties
## <dependency>
## ...
## </dependency>
Why is this dependency needed?
## Java
## XML

to:
that's potentially a breaking change.
## Check:
frontend consumers
other services
mobile clients
external clients
versioning
backward compatibility
- Configuration and deployment
If the PR changes infrastructure:
## Check:
environment variables
secrets
Kubernetes manifests
resource limits
readiness/liveness probes
autoscaling
Docker image
## CI/CD
database migration deployment order
## Example:
## {
"trackingCode": "123"
## }
## {
"trackingNumber": "123"
## }
## JSON
## JSON

Does the deployment behave safely?
## 30. Reliability
Think about production failure.
For every important external dependency ask:
Then consider:
But only where appropriate.
- Check race conditions
## Example:
Application starts
## ↓
Database migration?
## ↓
## Readiness?
## ↓
## Traffic
What happens when it is slow?
What happens when it returns 500?
What happens when it times out?
What happens when it is completely unavailable?
What happens when the response is malformed?
## Timeout
## Retry
## Backoff
Circuit breaker
## Fallback
## Queue
Dead-letter queue
## Idempotency

This looks correct but isn't necessarily safe.
Two requests:
Potential duplicate.
Possible solution depends on requirements:
This is a very common Senior-level review issue.
- Review the actual diff, not just the final code
This is an underrated technique.
## Ask:
Sometimes the existing code was correct and the new change introduced the bug.
Look for:
if (!repository.existsByOrderId(orderId)) {
repository.save(order);
## }
Request A → exists? NO
Request B → exists? NO
A → save
B → save
UNIQUE constraint
## +
appropriate transaction/error handling
Why was this line changed?
removed validation
changed transaction boundary
changed query
changed exception handling
changed null behavior
changed API response
changed concurrency behavior
## Java

- Check whether comments are actually correct
Never trust comments blindly.
## Example:
That comment itself could be wrong.
As a reviewer:
- Prioritize your comments
Don't leave 35 comments saying:
while missing:
Use priorities.
##  Blocking
Must fix.
## Examples:
security vulnerability
data corruption
broken transaction
incorrect business logic
serious race condition
##  Important
// DateTimeFormatter is not thread safe
private static final DateTimeFormatter FORMATTER = ...
Verify behavior, don't review comments instead of code.
Could rename this variable.
This can create duplicate payments.
## Java

Should fix before merge.
## Examples:
N+1 query
missing error handling
scalability problem
missing idempotency
##  Suggestion
## Improvement.
## Examples:
readability
maintainability
API naming
## ⚪ Nit
## Formatting/style.
Don't overwhelm the author with these.
- How I would actually review a PR
My practical workflow would be:
- Read ticket
## ↓
- Understand expected behavior
## ↓
- Look at PR summary/diff
## ↓
- Understand architecture/data flow
## ↓
- Review API
## ↓
- Review business logic
## ↓
- Review DB/JPA
## ↓
- Review transactions
## ↓
- Review concurrency
## ↓

- The Senior mindset
When reviewing a PR, don't think:
## Think:
Then mentally simulate:
Normal request
Invalid request
Database failure
External API failure
- Review external APIs/Kafka
## ↓
- Review error handling
## ↓
- Review security
## ↓
- Review performance
## ↓
- Review tests
## ↓
- Review observability
## ↓
- Review maintainability
## ↓
- Run/test mentally important scenarios
## ↓
- Leave prioritized comments
## ↓
- Give final PR summary
"Can I find something to comment on?"
"If this code goes to production tomorrow, what could go wrong?"
Request → Controller → Service → DB → Response
Request → validation → error
Service → DB failure → transaction/exception → response

Duplicate request
Concurrent requests
High traffic
What becomes the bottleneck?
Dependency outage
Does the whole system collapse?
- What I'd say in a Senior interview
If an interviewer asks:
"How do you review a PR?"
A strong answer would be:
Service → Carrier timeout → retry? → failure handling
## Request A
Request A again
## ↓
Does the system create duplicate data?
## Thread A
## Thread B
## ↓
Can they corrupt shared state?
100 req/s
→ 1,000 req/s
→ 10,000 req/s
Redis DOWN
Kafka DOWN
## DB DOWN
Carrier DOWN
"I first understand the business requirement and the scope of the PR before looking at
individual lines. Then I trace the main request and data flow through the controller,

That sounds much more Senior than:
The mental checklist I would memorize
service, database, messaging and external dependencies. I first prioritize correctness
and business logic, then look at transaction boundaries, JPA queries and database
performance, concurrency, error handling, API contracts, security, and external
dependency failures. I also check whether the implementation is scalable and
observable and whether the tests cover both happy paths and important failure
scenarios. Finally, I prioritize my review comments by severity rather than commenting
on every stylistic issue. I try to distinguish real bugs from potential concerns and
suggestions, and I avoid overengineering."
"I check code quality, naming and whether tests are there."
## REQUIREMENT
## ↓
## CORRECTNESS
## ↓
## BUSINESS LOGIC
## ↓
## API
## ↓
## TRANSACTION
## ↓
## DATABASE / JPA
## ↓
## CONCURRENCY
## ↓
EXTERNAL APIs / KAFKA
## ↓
## ERROR HANDLING
## ↓
## SECURITY
## ↓
## PERFORMANCE
## ↓
## SCALABILITY
## ↓
## OBSERVABILITY
## ↓
## TESTS
## ↓
## MAINTAINABILITY
## ↓
## PRODUCTION FAILURE SCENARIOS

The key is not to blindly run through this checklist on every PR. Start with correctness and
the highest-risk parts of the change, then go deeper where the PR actually warrants it.
WorkMotion Software GmbH
## Global Hiring Made Easy
Hire in 160+ countries without local setup.
## Ad
