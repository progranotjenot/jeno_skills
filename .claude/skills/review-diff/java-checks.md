# Java / Spring Review Checklist (shared catalog, optional language module)

This is the single source of truth for Java and Spring checks. The `review-diff` skill loads it only when the changed files are Java. Other languages need their own module in this folder.

Each section lists what to check, search hints (grep under `src/`), the signs of the problem, and a typical fix. Search hints only show where to look. Decide only after reading the whole relevant code path, never from one grep line.

## Severity levels

- **Critical**: exploitable vulnerability or data loss. Example: SQL injection with user input, password committed to the repository.
- **High**: serious production bug. Example: race condition causing double charges, external call inside a transaction without a timeout.
- **Medium**: problem under load or in a rare scenario. Example: N+1 on a list endpoint, no timeout on a rarely used client.
- **Low**: style, hygiene, minor risk. Example: `@Transactional` on a class where it is not needed.
- **Info**: observation that needs no action.

## Contents

1. Injection
2. Secrets and leaks in logs
3. Concurrency in stateless services
4. Database race conditions
5. `@Transactional`
6. N+1 and lazy loading
7. External calls inside a transaction
8. Validation
9. Null safety
10. Timeouts and resilience
11. Error handling and information leaks
12. Other vulnerabilities (crypto, configuration, deserialization)
13. Performance and data (indexes, pagination, time, money)

---

## 1. Injection

**SQL / JPQL injection.** User input goes into a query string.
- Search: `createQuery\(.*\+`, `createNativeQuery\(.*\+`, `@Query\(.*\+`, `String.format\(.*(SELECT|WHERE|ORDER)`, `jdbcTemplate\.(query|update)\(.*\+`.
- Sign: a string built with `+` or `String.format` contains a value from a request, DTO or header.
- Fix: named parameters (`:name` with `setParameter`, or `@Query` with `:param`).

**Sort and field names from input.** `Sort.by(request.getParameter(...))` or a `Pageable` with arbitrary `sort`. Hibernate does not allow SQL injection through a property name, but unknown names lead to 500. Sorting native SQL from input is dangerous.
- Search: `Sort.by\(`, `Sort.Order`, `ORDER BY`, `nativeQuery`.
- Fix: whitelist of allowed fields, or map to an enum.

**LIKE without escaping.** `%` and `_` from input change the meaning of the query and can cause a full scan.
- Fix: escape special characters or use `ESCAPE`.

**OS commands and file paths.**
- Search: `Runtime.getRuntime\(\).exec`, `new ProcessBuilder`, `new File\(.*(request|param|input)`, `Paths.get\(.*(request|param)`.
- Fix: normalize the path, check that it stays inside the base directory (`normalize()` and `startsWith`), never pass input to a shell.

**SSRF.** The server makes an HTTP request to a URL supplied by the user.
- Search: `restTemplate.*\(.*(url|uri|request)`, `WebClient.*uri\(`, `RestClient`, `URL\(`.
- Fix: host allowlist; block `localhost`, `169.254.x.x` and private ranges.

## 2. Secrets and leaks in logs

**Secrets in code and config.**
- Search: `password\s*=`, `secret`, `token`, `apiKey`, `api-key`, `BEGIN (RSA |EC )?PRIVATE KEY`, `jdbc:.*password=` in `src/`, `application*.properties`, `application*.yml`, `*.env`, `docker-compose*.yml`.
- Sign: the value is hard-coded and not from an environment variable or a `${...}` placeholder.
- Fix: `${DB_PASSWORD}` from environment variables or a secret store. If a secret has already reached git, rotate it: deleting it in a later commit does not help.
- Never write the secret value into a report. Give only the file, line and type.

**Secrets in logs.**
- Search: `log\.(info|debug|warn|error)\(.*(password|token|secret|authorization|request\b|dto)`, `logger\.` near `toString`.
- Sign: a whole object is logged and it contains a secret field. Also check `@ToString` and `toString()` on DTOs with passwords.
- Fix: exclude the field from `toString` (`@ToString.Exclude`), log only the identifier.

**Secrets in error messages and responses.** An exception that contains request parameters is sent to the client through `ProblemDetail.detail`.

**Risky configuration.**
- Search: `management.endpoints.web.exposure.include=\*`, `spring.h2.console.enabled=true`, `show-sql=true` in production, `include-stacktrace`, `server.error.include-stacktrace=always`.
- Sign: actuator exposes `env` or `heapdump`; H2 console reachable outside dev.

## 3. Concurrency in stateless services

A stateless service must not keep mutable data in singleton fields between requests. Spring beans are singletons by default and serve all threads.

**Mutable fields in beans.**
- Search: in classes annotated with `@Service`, `@Component`, `@RestController`, `@Configuration`: non-`final` fields, `private List<`, `private Map<`, `private static` with a mutable type.
- Sign: a field is changed in one method and read in another. Two requests see each other's data.
- Fix: move the state into method arguments, the database, or a `ConcurrentHashMap` if it must live in one instance (and remember it is not shared across instances).

**Unsafe objects.**
- Search: `SimpleDateFormat`, `DecimalFormat`, `Calendar`, `static .*Matcher`, `Random` in a bean field.
- Fix: `DateTimeFormatter` (immutable), `ThreadLocalRandom`, create `Matcher` locally.

**ThreadLocal.** If a `ThreadLocal` is not cleared with `remove()`, data from one request leaks into the next request on the same pool thread.
- Search: `ThreadLocal`, `MDC.put` without `MDC.clear`.

**@Async and thread pools.**
- Search: `@Async`, `Executors.new`, `new Thread(`.
- Sign: `Executors.newCachedThreadPool()` without a limit; `@Async` without its own `Executor`; security context and MDC are not propagated to the thread.
- Fix: `ThreadPoolTaskExecutor` with bounded queue and size; `DelegatingSecurityContextExecutor` or a `TaskDecorator` for MDC.

**Background jobs duplicated across instances.** `@Scheduled` runs on every instance.
- Search: `@Scheduled`, `@EnableScheduling`.
- Sign: no distributed lock. With two instances the job runs twice.
- Fix: ShedLock or a separate worker. First check whether the job needs to run on several instances at all.

## 4. Database race conditions

**Check-then-act.** A check happens first, then a write; another thread can slip in between.
- Search: `existsBy` or `findBy` near `save(` with no unique constraint.
- Sign: the database has no constraint, so two requests can both pass the check.
- Fix: a unique constraint in the database, and map `DataIntegrityViolationException` to 409. A check in code plus a constraint in the database is a deliberate pair, not duplication.

**Lost update.** Read, change, write; two requests overwrite each other.
- Search: `findById` → `setX` → `save` without `@Version`.
- Fix: `@Version` on the entity (optimistic locking) and map `ObjectOptimisticLockingFailureException` to 409. For counters and balances: an atomic `UPDATE ... SET x = x + :delta` or `@Lock(PESSIMISTIC_WRITE)`.
- Check that the global error handler (`@RestControllerAdvice`) maps `ObjectOptimisticLockingFailureException` to 409, not 500.

**Missing `@Version`.** For entities edited from several places, this is High.

**Uniqueness only in code.** A uniqueness check in the service without a database index does not protect against races.
- Search: `existsBy` with no `unique = true` or `@UniqueConstraint`.

**Isolation level.**
- Search: `@Transactional(isolation = ...)`, `READ_UNCOMMITTED`. Do not change without a reason. Check that calculations based on several reads do not run under `READ_COMMITTED` without locking.

## 5. `@Transactional`

**Self-invocation.** A method calls another `@Transactional` method of the same class directly. The Spring proxy is bypassed, so no transaction starts.
- Search: inside a class with `@Transactional`, a call `this.method(` or a plain `method(` where the target has the annotation.
- Fix: move the method to another bean, or use `TransactionTemplate`.

**Method is not public or is final.** The proxy does not intercept `private`, `protected` (in typical setups) or `final` methods.
- Search: `@Transactional` before `private`, `final`, `static`.

**`@Transactional` on a controller.** The transaction ends before the response is serialized, or it holds during HTTP client calls.
- Search: `@Transactional` in `controller/` files.

**`readOnly` on a write.** `@Transactional(readOnly = true)` on a method that writes may silently drop changes (depends on driver and Hibernate flush mode).
- Search: `readOnly = true` together with `save|delete|set` in the same method.
- If a service class is annotated `@Transactional(readOnly = true)`, every write method must override it with `@Transactional`. Check each write method.

**Checked exceptions do not roll back.** By default, rollback happens only for `RuntimeException` and `Error`.
- Search: a `@Transactional` method that throws a checked exception without `rollbackFor`.
- Fix: `@Transactional(rollbackFor = Exception.class)`.

**Swallowed exception.** A `catch` inside a transaction without rethrowing. If the exception came from a nested `@Transactional` call, the session is marked rollback-only and the outer commit fails with `UnexpectedRollbackException`.

**REQUIRES_NEW.** A new transaction does not see uncommitted data of the outer one and can deadlock on locks. Use only with a clear reason.

**Transactional events.** `@EventListener` that writes to the database, or `@TransactionalEventListener` without `AFTER_COMMIT` when it sends something to the outside world.

**Transaction too wide.** A long method with `@Transactional` holds a pooled connection. Sign: files, network, `Thread.sleep` or a large loop inside the transaction.

## 6. N+1 and lazy loading

**N+1.** One query for a list, then one query per element.
- Search: `for (` or `stream().map(` containing a `repository.` call or a relation getter (`getOwner()`, `getItems()`).
- Sign: a mapper touches a related entity or collection inside a loop.
- Fix: `@EntityGraph`, `JOIN FETCH` in JPQL, `@BatchSize`. Careful: fetching a collection with `JOIN FETCH` together with `Pageable` triggers warning HHH90003004 and pagination in memory. In that case use two queries: first the page of ids, then fetch by those ids.
- For new associations, a query-count test is useful (for example, counting SQL statements with Hibernate statistics or datasource-proxy).

**EAGER by default.** `@ManyToOne` and `@OneToOne` are EAGER by default, which pulls the relation into every query.
- Search: `@ManyToOne`, `@OneToOne` without `fetch = FetchType.LAZY`; `fetch = FetchType.EAGER`.

**LazyInitializationException.** A lazy relation is read after the session is closed, for example in a controller or mapper with `open-in-view=false`.
- Sign: a mapper or controller accesses a collection or relation the service did not load.
- Fix: load what is needed in the service (`@EntityGraph`), or use a DTO projection in the query.

**Unbounded collection.** `findAll()` or a `@OneToMany` collection without pagination on a list endpoint.

## 7. External calls inside a transaction

An external call (HTTP, queue, mail, file, S3) inside `@Transactional` holds a database connection while waiting for the network. With a slow external service the connection pool runs out. If the external call succeeds and the transaction rolls back, the systems diverge: the event exists outside, not in the database.

- Search: inside a `@Transactional` method, calls to `restTemplate`, `webClient`, `RestClient`, `*Client.`, `kafkaTemplate.send`, `mailSender.send`, `amazonS3`, `Files.`, `publisher.publish`.
- Fix:
  - Move the call outside the transaction (commit first, then call).
  - For events: `@TransactionalEventListener(phase = AFTER_COMMIT)`, or an outbox table with a separate sender.
  - For a call that must be idempotent, add an idempotency key.

## 8. Validation

**Missing `@Valid`.** A `@RequestBody` without `@Valid` or `@Validated` is not validated; constraints on the DTO do nothing.
- Search: `@RequestBody` without `@Valid`/`@Validated`.

**Missing constraints.** DTO fields without `@NotBlank`, `@Size`, `@Pattern`, `@Positive`, `@NotNull`.
- Search: fields in `dto/` packages without annotations.
- Check that `@Size(max = N)` does not exceed the column `length` in the entity. Otherwise a long string produces a 500 from the database.

**Constraints on the entity instead of the DTO.** An entity should not be an input parameter.

**Mass assignment.** The controller accepts an entity and saves it. The client can set `id`, `version`, `createdAt`.
- Search: `@RequestBody` with a type that is an `@Entity`.

**PUT replaces the whole resource.** Fields not sent in PUT become `null`. Check that required fields are `@NotNull` in the request DTO.

**Pagination.** Page size must be capped (`spring.data.web.pageable.max-page-size`). Check that negative or zero `page` and `size` do not produce a 500.

**Enum from string.** An unknown value must yield 400, not 500. Check handling of `MethodArgumentTypeMismatchException` and `HttpMessageNotReadableException` in the global error handler. If it extends `ResponseEntityExceptionHandler`, the base class already covers these cases.

**Normalization.** Strings should be trimmed (`strip()`) before saving, otherwise spaces create duplicates. Check that the uniqueness check and the save use the same normalized value.

## 9. Null safety

- `Optional.get()` without `isPresent()` or `orElseThrow`. Search: `\.get\(\)` near `Optional`, `findById(...).get()`.
- Unboxing `null` into a primitive: `int x = map.get(key);`, `Integer` in arithmetic. Search: `Map<.*Integer` and `get(` without a check.
- Returning `null` instead of an empty collection. Clients and `stream()` break. Fix: `List.of()` or `Collections.emptyList()`.
- `equals` on a possibly `null` object. Fix: `Objects.equals`, or a constant on the left: `"ACTIVE".equals(status)`.
- String comparison with `==`. Search: `== "` and `== null` on strings from input.
- Annotations: if the project uses `@Nullable` / `@NonNull`, check they are used consistently.
- Check that `findById(...).orElseThrow(...)` throws a domain exception with an error code, not a bare `NoSuchElementException`.

## 10. Timeouts and resilience

**REST clients without timeouts.** The most common problem: a thread waits forever when the external service hangs.
- Search: `new RestTemplate\(\)`, `RestClient.create\(\)`, `WebClient.create\(\)`, `HttpClient.newHttpClient\(\)`, `new HttpClient`, Feign without `connectTimeout` and `readTimeout`.
- Fix: set `connectTimeout` (1–3 s) and `readTimeout` (a few seconds), via `ClientHttpRequestFactory`, or `HttpClient.newBuilder().connectTimeout(...)` and `HttpRequest.timeout(...)`.
- Spring Boot: properties `spring.http.client.connect-timeout` and `read-timeout` exist only in supported versions. Check the Spring Boot version in the build file before recommending them.

**Database query timeout.** A slow query without a limit holds a thread.
- Fix: `@QueryHints(@QueryHint(name = "jakarta.persistence.query.timeout", value = "2000"))`, or `spring.jpa.properties.jakarta.persistence.query.timeout`.

**Connection pool.** `spring.datasource.hikari.connection-timeout`, `maximum-pool-size`. Defaults are reasonable, but check them explicitly under load.

**Retries.** A retry without backoff or idempotency creates duplicates and overloads a failing service.
- Search: `@Retryable`, `while` loops with `try/catch` and retry, `retry`.
- Sign: a `POST` is retried without an idempotency key, no pause between attempts, no attempt limit.

**Circuit breaker.** An external call with no protection: when the service is down, all threads wait for the timeout. Suggest Resilience4j when there are several external calls. Do not add it without discussion, since it is a new dependency.

## 11. Error handling and information leaks

**Swallowed exception.** `catch (Exception e) {}`, or a `catch` that only logs at debug level.
- Search: `catch \(.*\) \{\s*\}`, `catch` containing only `printStackTrace()`.

**Error logged and rethrown.** One error ends up in the log several times. Low severity, but it makes incident analysis harder.

**Exception message sent to the client.** `e.getMessage()` in `ProblemDetail.detail` can reveal SQL, paths, class names.
- Search: `getMessage()` in `common/` and `controller/` packages.
- Expected behaviour: the handler for unknown exceptions returns 500 without internal details (no message, stack trace or SQL). Verify this.

**Stable error codes.** If external clients rely on error codes, they must not be renamed or removed. Check the diff for changed existing lines in the enum or code table.

**Stack trace exposed.** `server.error.include-stacktrace=always`, `server.error.include-message=always`.

**PII in logs.** Email, phone, card number, IP without need.
- Search: `log.*(email|phone|card|ssn|iban)`.

**Log injection.** User input containing line breaks goes into the log unescaped. Fix: parameterized logging (`log.info("x={}", x)`) and a length limit.

## 12. Other vulnerabilities

**Cryptography.**
- Search: `MD5`, `SHA-1`, `SHA1`, `DES`, `ECB`, `new Random\(`, `Math.random`.
- Passwords: only `BCrypt`, `Argon2` or `PBKDF2` through `PasswordEncoder`. Tokens and keys: `SecureRandom`.

**TLS verification.**
- Search: `TrustAllCerts`, `X509TrustManager` with an empty `checkServerTrusted`, `setHostnameVerifier.*true`, `ALLOW_ALL_HOSTNAME_VERIFIER`, `trustStore` set to `""`.
- This is Critical.

**Deserialization.**
- Search: `ObjectInputStream`, `enableDefaultTyping`, `activateDefaultTyping`, `@JsonTypeInfo` with `Id.CLASS` on input data, `XMLDecoder`.
- Fix: do not use `ObjectInputStream` on untrusted data; do not enable default typing in Jackson.

**CORS and CSRF.**
- Search: `allowedOrigins("*")` together with `allowCredentials(true)`; `csrf().disable()` or `csrf(...disable)` without a reason.
- For a REST API without cookie sessions, disabling CSRF is acceptable, but confirm that no cookie-based authentication exists.

**Authentication and authorization.** Check that all endpoints are protected with the right roles when Spring Security is in use. If Spring Security is absent, record it as Info for production.

**Files.** Size limits (`spring.servlet.multipart.max-file-size`), type checked by content rather than extension, storage outside the web root.

**equals and hashCode on entities.** Including `id` in `hashCode` while `id == null` (before save) breaks `Set`. Search: `@EqualsAndHashCode`, `@Data` on an `@Entity` (Lombok `@Data` generates `equals` over all fields, which is unsafe for JPA).

**Dependencies.** CVE checks are not part of a manual review. Suggest running `org.owasp.dependencycheck` if the plugin is configured. Do not run it without a request.

## 13. Performance and data

**Indexes.** Columns used in `WHERE`, `ORDER BY` and `JOIN` without an index. Check `@Table(indexes = ...)` and repository query conditions.

**Unbounded result.** A query with no `LIMIT` or `Pageable` on a table that grows.

**Time.** `LocalDateTime.now()` without an injected `Clock` is hard to test and causes time-zone issues. Search: `now()` in services. Timestamps handled by auditing are fine.

**Money.** `double` or `float` for money. Search: `double.*(price|amount|sum|balance)`. Fix: `BigDecimal`, and `NUMERIC` in the database.

**Overflow.** `int` for totals and counters, multiplication without `Math.multiplyExact`. Search: `int .*(total|sum|count)` where values can be large.

**Date and string types in queries.** Passing a string where a date type is expected in JPQL.

---

## How to write a finding

For each finding give the location as `path/File.java:line`, the problem, the harm scenario, and a concrete fix. If the project already has a correct pattern (for example, a case-insensitive check paired with a unique constraint in the database), cite it as the model for the fix.
