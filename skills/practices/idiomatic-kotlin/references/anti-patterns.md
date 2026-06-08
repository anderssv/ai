# Idiomatic Kotlin: Anti-Pattern Reference

> These illustrate the principle. Consider what fits your context.

## Contents

- Nullability
- Scope functions
- Collections
- Data classes
- Sealed classes and interfaces
- Value classes
- Extension functions
- Properties
- Coroutines
- String handling
- Type aliases
- Object declarations and companion objects
- Kotlin-specific constructs
- Default parameter values
- DSL builders

---

## 1. Nullability

### Optional vs nullable types

```kotlin
// NON-IDIOMATIC
fun findUser(id: String): Optional<User> {
    return Optional.ofNullable(repository.find(id))
}
val name = findUser("1").map { it.name }.orElse("Unknown")

// IDIOMATIC
fun findUser(id: String): User? = repository.find(id)
val name = findUser("1")?.name ?: "Unknown"
```

`Optional` adds heap allocation and doesn't integrate with `?.`, `?:`, or smart casts. Reserve for Java interop boundaries.

### `!!` overuse

```kotlin
// NON-IDIOMATIC
fun process(user: User?) {
    val name = user!!.name
    val email = user!!.email
    sendEmail(name, email)
}

// IDIOMATIC: early return or requireNotNull
fun process(user: User?) {
    val u = user ?: return
    sendEmail(u.name, u.email)
}

fun process(user: User?) {
    val u = requireNotNull(user) { "User must not be null" }
    sendEmail(u.name, u.email)
}
```

`!!` converts compile-time safety into a runtime crash. Use `?.`, elvis, `requireNotNull`, or `checkNotNull` instead.

### Platform type leakage

```kotlin
// NON-IDIOMATIC: platform type propagates
val name = javaApi.getName()  // Inferred as String! (platform type)

// IDIOMATIC: declare type explicitly at boundary
val name: String = javaApi.getName()   // Throws early if null
val name: String? = javaApi.getName()  // Handles null explicitly
```

Platform types (`String!`) bypass null safety. Declare the type at Java interop boundaries.

---

## 2. Scope Functions

### `apply` for side effects (use `also`)

```kotlin
// NON-IDIOMATIC
val list = mutableListOf(1, 2, 3).apply {
    println("Created list: $this")
}

// IDIOMATIC
val list = mutableListOf(1, 2, 3).also {
    println("Created list: $it")
}
```

`apply` implies configuration. `also` signals side effects.

### `let` when `run` is cleaner

```kotlin
// NON-IDIOMATIC
service.let {
    it.port = 8080
    it.host = "localhost"
    it.start()
}

// IDIOMATIC
service.run {
    port = 8080
    host = "localhost"
    start()
}
```

When calling members on the object, `this`-based functions remove `it.` noise.

### Nested scope functions

```kotlin
// NON-IDIOMATIC
user?.let { u ->
    u.address?.let { addr ->
        addr.city?.let { city ->
            println(city)
        }
    }
}

// IDIOMATIC
val city = user?.address?.city
if (city != null) println(city)
```

Nested scopes create indentation pyramids and ambiguous `it`/`this` references.

### `with` on nullable

```kotlin
// NON-IDIOMATIC
with(getNullableConfig()!!) {
    timeout = 5000
}

// IDIOMATIC
getNullableConfig()?.run {
    timeout = 5000
}
```

`with` isn't an extension function, so `?.` doesn't chain. Use `run` or `let` for nullable receivers.

---

## 3. Collections

### Imperative accumulation

```kotlin
// NON-IDIOMATIC
val result = mutableListOf<String>()
for (user in users) {
    if (user.isActive) {
        result.add(user.name.uppercase())
    }
}

// IDIOMATIC
val result = users
    .filter { it.isActive }
    .map { it.name.uppercase() }
```

### Manual map building

```kotlin
// NON-IDIOMATIC
val map = mutableMapOf<String, User>()
for (user in users) {
    map[user.id] = user
}

// IDIOMATIC
val map = users.associateBy { it.id }

// Related:
val nameToAge = users.associate { it.name to it.age }
val byDepartment = users.groupBy { it.department }
val idToName = users.associateWith { it.name }
```

### Nested loops

```kotlin
// NON-IDIOMATIC
val allTags = mutableListOf<String>()
for (post in posts) {
    for (tag in post.tags) {
        allTags.add(tag)
    }
}

// IDIOMATIC
val allTags = posts.flatMap { it.tags }
```

### Large pipeline without sequences

```kotlin
// NON-IDIOMATIC: multiple intermediate lists
val result = hugeList
    .filter { it.isValid }
    .map { it.transform() }
    .take(10)

// IDIOMATIC: lazy evaluation
val result = hugeList.asSequence()
    .filter { it.isValid }
    .map { it.transform() }
    .take(10)
    .toList()
```

Sequences process lazily, avoiding intermediate allocations. Especially beneficial with short-circuiting (`take`, `first`).

### Mutable collections in public API

```kotlin
// NON-IDIOMATIC
fun getUsers(): MutableList<User> = mutableListOf(user1, user2)

// IDIOMATIC
fun getUsers(): List<User> = listOf(user1, user2)
```

---

## 4. Data Classes

### Manual boilerplate

```kotlin
// NON-IDIOMATIC
class User(val name: String, val email: String) {
    override fun equals(other: Any?): Boolean { ... }
    override fun hashCode() = Objects.hash(name, email)
    override fun toString() = "User(name=$name, email=$email)"
}

// IDIOMATIC
data class User(val name: String, val email: String)
```

### Builder pattern for immutable updates

```kotlin
// NON-IDIOMATIC
val updated = User.Builder(original).name("New Name").build()

// IDIOMATIC
val updated = original.copy(name = "New Name")
```

### Destructuring

```kotlin
// NON-IDIOMATIC
val pair = "key" to "value"
println(pair.first)
println(pair.second)

// IDIOMATIC
val (key, value) = "key" to "value"
for ((id, user) in usersById) { ... }
```

### When NOT to use data classes

```kotlin
// NON-IDIOMATIC: data class for entity with identity
data class DatabaseEntity(val id: Long, var name: String, var status: String)
// equals/hashCode includes mutable fields, breaks HashMap behavior

// IDIOMATIC: regular class with explicit equals on ID
class DatabaseEntity(val id: Long, var name: String, var status: String) {
    override fun equals(other: Any?) = other is DatabaseEntity && id == other.id
    override fun hashCode() = id.hashCode()
}
```

Data classes derive `equals`/`hashCode` from ALL constructor properties. Wrong for entities where equality is by ID.

---

## 5. Sealed Classes and Interfaces

### Enums for states with varying data

```kotlin
// NON-IDIOMATIC
enum class ResultType { SUCCESS, ERROR, LOADING }
class Result(val type: ResultType, val data: Any?, val error: Throwable?)

// IDIOMATIC
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Throwable) : Result<Nothing>()
    data object Loading : Result<Nothing>()
}
```

Each subclass carries exactly the data it needs. Illegal states are unrepresentable.

### `else` on sealed class `when`

```kotlin
// NON-IDIOMATIC: else hides future subclasses
fun handle(result: Result<String>) = when (result) {
    is Result.Success -> showData(result.data)
    else -> showError()
}

// IDIOMATIC: exhaustive -- compiler catches missing cases
fun handle(result: Result<String>) = when (result) {
    is Result.Success -> showData(result.data)
    is Result.Error -> showError(result.exception)
    Result.Loading -> showLoading()
}
```

### Sealed interface for cross-hierarchy

```kotlin
sealed interface DomainError {
    data class NotFound(val id: String) : DomainError
    data class ValidationFailed(val errors: List<String>) : DomainError
    data class Unauthorized(val reason: String) : DomainError
}
```

Sealed interfaces allow subclasses to have independent class hierarchies.

---

## 6. Value Classes

### Primitive obsession

```kotlin
// NON-IDIOMATIC
fun createOrder(customerId: String, productId: String, quantity: Int) { ... }
createOrder(productId, customerId, 5)  // Silently swapped

// IDIOMATIC
@JvmInline value class CustomerId(val value: String)
@JvmInline value class ProductId(val value: String)
@JvmInline value class Quantity(val value: Int) {
    init { require(value > 0) { "Quantity must be positive" } }
}

fun createOrder(customerId: CustomerId, productId: ProductId, quantity: Quantity) { ... }
```

Value classes provide compile-time type safety with zero runtime overhead.

### Type alias vs value class

```kotlin
// NON-IDIOMATIC: typealias gives no type safety
typealias UserId = String
typealias OrderId = String
fun find(id: UserId) { ... }
find(OrderId("order-1"))  // Compiles -- no type boundary

// IDIOMATIC: value class when you need type safety
@JvmInline value class UserId(val value: String)
@JvmInline value class OrderId(val value: String)
```

Type aliases are just names. Use them only for simplifying complex generic types (e.g., `typealias EventHandler<T> = (T) -> Unit`).

---

## 7. Extension Functions

### Utility classes

```kotlin
// NON-IDIOMATIC
object StringUtils {
    fun isPalindrome(s: String): Boolean = s == s.reversed()
}
StringUtils.isPalindrome("racecar")

// IDIOMATIC
fun String.isPalindrome(): Boolean = this == reversed()
"racecar".isPalindrome()
```

### Extension abuse

```kotlin
// NON-IDIOMATIC: extension on Any
fun Any.log() = println(this)

// NON-IDIOMATIC: unrelated to receiver
fun String.saveToDatabase() { database.save(this) }

// IDIOMATIC: relates to receiver's domain
fun String.toSlug(): String =
    lowercase().replace(Regex("[^a-z0-9]+"), "-").trim('-')
```

Extensions should be conceptually "methods the type should have." Restrict visibility (`private`, `internal`) to where they're needed.

### Extension vs member

Core behavior belongs as a member function (participates in overriding). Cross-cutting or client-specific behavior belongs as an extension (statically resolved, can be scoped).

---

## 8. Properties

### Java-style getters/setters

```kotlin
// NON-IDIOMATIC
class User {
    private var _name: String = ""
    fun getName(): String = _name
    fun setName(name: String) { _name = name }
}

// IDIOMATIC
class User {
    var name: String = ""
        set(value) {
            require(value.isNotBlank()) { "Name must not be blank" }
            field = value
        }
}
```

### Property vs function

Prefer a property when: cheap, no side effects, same result for same state.
Use a function when: expensive, may throw, or has side effects.

```kotlin
val fullName: String get() = "$firstName $lastName"     // property
fun loadProfile(): Profile = database.query(id)          // function
```

### `lazy` vs `lateinit`

`lazy` -- for vals, deferred computation, thread-safe:
```kotlin
val config: Config by lazy { loadConfig() }
```

`lateinit` -- for vars, DI/framework-initialized, must be set before use:
```kotlin
lateinit var repository: Repository
```

### Backing property pattern

```kotlin
class ViewModel {
    private val _items = mutableListOf<Item>()
    val items: List<Item> get() = _items

    fun addItem(item: Item) { _items.add(item) }
}
```

Exposes mutable internal state as immutable public view.

---

## 9. Coroutines

### Callbacks vs suspend

```kotlin
// NON-IDIOMATIC
fun loadUser(id: String, callback: (User?, Throwable?) -> Unit) {
    thread {
        try { callback(api.fetchUser(id), null) }
        catch (e: Exception) { callback(null, e) }
    }
}

// IDIOMATIC
suspend fun loadUser(id: String): User = api.fetchUser(id)
```

### GlobalScope

```kotlin
// NON-IDIOMATIC
fun startWork() {
    GlobalScope.launch { doExpensiveWork() }
}

// IDIOMATIC
suspend fun startWork() = coroutineScope {
    launch { doExpensiveWork() }
}
```

`GlobalScope` creates orphaned coroutines that ignore parent cancellation.

### `runBlocking` in production

```kotlin
// NON-IDIOMATIC
fun getUser(): User = runBlocking { userRepository.findById(id) }

// IDIOMATIC: keep the suspending chain unbroken
suspend fun getUser(): User = userRepository.findById(id)
```

`runBlocking` blocks the calling thread. Appropriate only at top-level entry points (`main()`, test functions, bridging from non-suspending frameworks).

### `async` without `await`

```kotlin
// NON-IDIOMATIC: exception silently dropped
scope.async { riskyOperation() }

// IDIOMATIC: use launch for fire-and-forget
scope.launch { riskyOperation() }
```

### Callback streams vs Flow

```kotlin
// IDIOMATIC
fun locationUpdates(): Flow<Location> = callbackFlow {
    val listener = LocationListener { trySend(it) }
    registerListener(listener)
    awaitClose { unregisterListener(listener) }
}
```

`Flow` integrates with cancellation, supports backpressure, and composes with operators.

---

## 10. String Handling

### Concatenation vs templates

```kotlin
// NON-IDIOMATIC
val msg = "Hello, " + user.name + "! You have " + count + " messages."

// IDIOMATIC
val msg = "Hello, ${user.name}! You have $count messages."
```

### StringBuilder vs buildString

```kotlin
// NON-IDIOMATIC
val sb = StringBuilder()
sb.append("Name: ").append(user.name).append("\n")
val result = sb.toString()

// IDIOMATIC
val result = buildString {
    appendLine("Name: ${user.name}")
    append("Email: ${user.email}")
}
```

### SLF4J parameterized logging vs string templates

```kotlin
// NON-IDIOMATIC: Java-style placeholder logging
log.info("Starting process for orderId: {}, callId: {}, originatorId: {}", orderId, callId, originatorId)

// IDIOMATIC: Kotlin string interpolation
log.info("Starting process for orderId: $orderId, callId: $callId, originatorId: $originatorId")
```

SLF4J's `{}` placeholders exist to avoid string concatenation cost when the log level is disabled — a Java concern. In Kotlin, string templates are cheap (compiled to `StringBuilder`), and the readability gain from seeing values inline far outweighs the negligible performance difference. The placeholder style forces you to mentally match positional arguments to `{}` slots, which is error-prone. Use string interpolation for all logging.

### Escaped newlines vs multiline strings

```kotlin
// NON-IDIOMATIC
val json = "{\n  \"name\": \"Kotlin\"\n}"

// IDIOMATIC
val json = """
    {
      "name": "Kotlin"
    }
""".trimIndent()
```

---

## 11. Object Declarations and Companion Objects

### Java singleton pattern

```kotlin
// NON-IDIOMATIC: double-checked locking
class Connection private constructor() {
    companion object {
        @Volatile private var instance: Connection? = null
        fun getInstance(): Connection =
            instance ?: synchronized(this) {
                instance ?: Connection().also { instance = it }
            }
    }
}

// IDIOMATIC
object Connection {
    fun query(sql: String): Result { ... }
}
```

### Constants

```kotlin
// NON-IDIOMATIC
companion object {
    val MAX_RETRIES = 3  // Runtime property access
}

// IDIOMATIC
companion object {
    const val MAX_RETRIES = 3  // Inlined at compile time
}
```

Use `const val` for primitives and strings. Top-level `const val` for truly global constants.

---

## 12. Kotlin-Specific Constructs

### `require` / `check` / `error`

```kotlin
// NON-IDIOMATIC
if (age < 0) throw IllegalArgumentException("Age must be non-negative")

// IDIOMATIC
require(age >= 0) { "Age must be non-negative, was $age" }
check(isInitialized) { "Must initialize before processing" }
error("Unknown type: $type")  // for unreachable states
```

### `use()` for Closeable

```kotlin
// NON-IDIOMATIC
val stream = FileInputStream("file.txt")
try { process(stream.readBytes()) }
finally { stream.close() }

// IDIOMATIC
FileInputStream("file.txt").use { process(it.readBytes()) }
```

### `when` expression

```kotlin
// NON-IDIOMATIC: if-else chain
if (obj is Int) return "Integer"
else if (obj is String) return "String"
else return "Unknown"

// IDIOMATIC
fun describe(obj: Any): String = when (obj) {
    is Int -> "Integer: $obj"
    is String -> "String of length ${obj.length}"
    else -> "Unknown"
}
```

Prefer `when` over if-else chains for 3+ branches.

### Expression-form `if`/`try`

```kotlin
// NON-IDIOMATIC
var result: String
if (condition) { result = "yes" } else { result = "no" }

// IDIOMATIC
val result = if (condition) "yes" else "no"
val number = try { input.toInt() } catch (e: NumberFormatException) { 0 }
```

### Single-expression functions

```kotlin
// NON-IDIOMATIC
fun double(x: Int): Int {
    return x * 2
}

// IDIOMATIC
fun double(x: Int) = x * 2
```

### Destructuring in lambdas

```kotlin
// NON-IDIOMATIC
map.forEach { entry -> println("${entry.key} = ${entry.value}") }

// IDIOMATIC
map.forEach { (key, value) -> println("$key = $value") }
```

---

## 13. Default Parameter Values

Default parameter values are an anti-pattern. They should only be used in very rare cases. A default value may serve as an intermediate step during a refactoring, but it must always be removed before committing.

### Avoid in production code

```kotlin
// NON-IDIOMATIC: default values hide call-site intent
fun createUser(name: String, role: String = "user", active: Boolean = true): User { ... }
createUser("Alice")  // What role? Active? Reader must check the function signature.

// IDIOMATIC: explicit at call site
fun createUser(name: String, role: String, active: Boolean): User { ... }
createUser("Alice", role = "user", active = true)
```

Default values in production code obscure what's actually happening at the call site. The reader has to navigate to the function definition to understand the full behavior.

### As a refactoring stepping stone (temporary only)

When adding a new parameter to a widely-called function, a default value avoids updating every call site in one step:

```kotlin
// Step 1: Add parameter with default (intermediate, do not commit)
fun createUser(name: String, role: String, active: Boolean, region: Region = Region.DEFAULT): User { ... }

// Step 2: Update all call sites to pass explicit values
createUser("Alice", role = "user", active = true, region = Region.EU)

// Step 3: Remove the default before committing
fun createUser(name: String, role: String, active: Boolean, region: Region): User { ... }
```

This is a useful technique during refactoring, but the default must be removed before the change is committed. Committed code should always have explicit values at all call sites.

### Acceptable in test code

```kotlin
// In test helpers, defaults reduce noise and focus on what varies
fun aUser(
    name: String = "Test User",
    email: String = "test@example.com",
    role: String = "user",
) = User(name, email, role)

// Tests focus on what matters
val admin = aUser(role = "admin")
val customUser = aUser(name = "Alice", email = "alice@example.com")
```

In test code, default values are useful for builders and test data setup -- they let each test specify only the fields relevant to that test case, keeping tests focused and readable.

---

## 14. DSL Builders

### `@DslMarker` for scope control

```kotlin
// Without @DslMarker: inner lambdas accidentally call outer receivers
html {
    body {
        body { }  // Compiles -- calls html's body() again
    }
}

// With @DslMarker: compile error on outer receiver access
@DslMarker
annotation class HtmlDsl

@HtmlDsl class HtmlBuilder { fun body(init: BodyBuilder.() -> Unit) { ... } }
@HtmlDsl class BodyBuilder { fun p(text: String) { ... } }
```

### Builder pattern

Lambda with receiver + builder class + `apply`:

```kotlin
class QueryBuilder {
    private var table = ""
    private val conditions = mutableListOf<String>()
    fun from(table: String) { this.table = table }
    fun where(condition: String) { conditions.add(condition) }
    fun build() = buildString {
        append("SELECT * FROM $table")
        if (conditions.isNotEmpty()) {
            append(" WHERE ${conditions.joinToString(" AND ")}")
        }
    }
}

fun query(init: QueryBuilder.() -> Unit) = QueryBuilder().apply(init).build()

val sql = query {
    from("users")
    where("age > 18")
    where("active = true")
}
```
