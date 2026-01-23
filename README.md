# Rivo Language Specification v0.1

**Rivo: Full-stack in flow**

A modern, type-safe language for full-stack web development with explicit server/client boundaries.

---

## Philosophy: CLEAR Principles

Rivo is built on **CLEAR** - a pragmatic approach to modern software development:

- **Composition** - Build with small, modular pieces that combine naturally
- **Locality** - Keep related logic together, avoid scattering across files
- **Extendability** - Start simple, grow organically as needs arise
- **Adaptability** - Embrace change without architectural rewrites
- **Readability** - Code that speaks clearly to humans and machines

**Core Values:**
- Pragmatic over dogmatic
- Explicit over clever
- Simple over complex
- Flow over ceremony

---

## Execution Contexts

| Extension | Context | Access | Similar To |
|-----------|---------|--------|------------|
| `.sx` | Server execution | Database, filesystem, env vars, secrets | Node.js / server-side TS |
| `.sc` | Server component | HTML + props from `.sx`, compiled on server before passing to client | SSR / RSC |
| `.cx` | Client execution | DOM, browser APIs, local storage | Browser JS / client TS |

**Rules:**
- `.sx` files never bundle to client (compiler enforced)
- `.sc` files render on server, can hydrate with `.sx` functions
- `.cx` files run only in browser, can hydrate with `.sx` functions
- `.cx` files cannot import non-exposed `.sx` internals (compiler enforced)
- `.cx` and `.sc` files CAN import `exposed` functions, types, and enums from `.sx` modules
- Communication between contexts happens through explicit APIs

**Hydration:**
- `.sc` components render HTML on server, hydrate interactivity on client
- `.cx` components render entirely on client
- Both can call `exposed` functions from `.sx` files

```typescript
// user_service.sx - Server only (data layer)
expose get_user(id: int): result<user, error> {
  user = Database.query("SELECT * FROM users WHERE id = ?", [id]).first()
  
  if user == null {
    return err("User not found")
  }
  
  return user
}

// user_card.sc - Server component (SSR)
import { get_user } from "./user_service.sx"

component UserCard(user_id: int) {
  // Runs on server during render
  user, err = await get_user(user_id)
  
  if err {
    return <p class="error">Failed to load user</p>
  }
  
  // HTML rendered on server, sent to client
  <div class="user-card">
    <h2>{user.name}</h2>
    <p>{user.email}</p>
  </div>
}

// interactive_button.cx - Client component (interactive)
import { update_user_status, user_status } from "./user_service.sx"

component InteractiveButton(user_id: int, initial_status: user_status) {
  status = initial_status
  loading = false
  
  async toggle() {
    loading.value = true
    new_status = status == user_status.active 
      ? user_status.suspended 
      : user_status.active
    
    updated, err = await update_user_status(user_id, new_status)
    if !err {
      status.value = updated.status
    }
    loading.value = false
  }
  
  <button @click={toggle} disabled={loading}>
    {loading ? "..." : (status == user_status.active ? "Suspend" : "Activate")}
  </button>
}
```

### Context Comparison

| Feature | `.sx` | `.sc` | `.cx` |
|---------|-------|-------|-------|
| Runs on server | ✅ | ✅ (render) | ❌ |
| Runs on client | ❌ | ✅ (hydrate) | ✅ |
| Database access | ✅ | Via `.sx` | Via `.sx` |
| DOM access | ❌ | ❌ | ✅ |
| Renders HTML | ❌ | ✅ (SSR) | ✅ (CSR) |
| Interactive | ❌ | With `.cx` children | ✅ |
| SEO friendly | N/A | ✅ | ❌ |

---

## Variables

**Immutable by default.** No keywords needed.

```typescript
// Declaration (immutable)
name = "John"
age = 25
max_users = 1000
api_url = "https://api.com"

// Reassignment fails
name = "Jane"  // Error

// Mutation with .value
count = 0
count.value = 5      // Declare intent to mutate
count.value += 1     // Mutate

// Reading (no .value needed)
print(count)         // Outputs: 6
total = count + 5

// Compiler optimizes constants automatically
```

**Note:** `.value` is a reserved accessor for mutation. Primitives are internally wrapped objects, so user-defined types cannot have a `.value` property that conflicts.

---

## Type System

### Primitives
```typescript
int, float, str, bool
datetime, date, time
null                 // Intentional absence of value
undefined            // Uninitialized or missing value (useful for JSON)
```

### Collections
```typescript
list<T>              // Ordered, indexed, duplicates allowed
map<K, V>            // Key-value pairs, unique keys
set<T>               // Unique values, unordered
```

### Tuples
```typescript
tuple<T, U>          // Fixed-size, heterogeneous (2 elements)
tuple<T, U, V>       // Can have 2+ elements

// Examples
point = (10, 20)                            // tuple<int, int>
entry = ("alice", 100)                      // tuple<str, int>
record: tuple<str, int, bool> = ("a", 1, true)

// Destructuring
(name, score) = entry
(x, y) = point

// Used by map.entries()
scores.entries()     // list<tuple<str, int>>
```

### Type Inference from Literals
```typescript
// Inferred from literal
numbers = [1, 2, 3]                    // list<int>
scores = { "a": 1, "b": 2 }            // map<str, int>
tags = set(["rust", "go"])             // set<str>
point = (10, 20)                       // tuple<int, int>

// Explicit types when needed
users: list<user> = []
config: map<str, str> = {}
ids: set<int> = set([])
coords: tuple<float, float> = (0.0, 0.0)
```

### Special Types
```typescript
result<T, E>    // Success or error
option<T>       // Some or None
error           // Error type (contains message: str, code?: int)
unknown         // Dynamic/unknown type (for JSON, etc.)
```

### Nullable
```typescript
name: str       // Cannot be null
email?: str     // Can be null or undefined

// Optional chaining
user?.address?.city

// Null coalescing (only null/undefined)
display = user?.name ?? "Anonymous"

// Elvis operator (any falsy value)
display = user.name ?: "Anonymous"
```

### Custom Types
```typescript
type user {
  id: int,
  name: str,
  email?: str,
  status: user_status,
  created_at: datetime
}

type address {
  street: str,
  city: str,
  country: str
}
```

### Enums (Backed)

**Enums with explicit backing values for database storage and serialization:**

```typescript
// String-backed enums
enum user_status {
  pending: "pending",
  active: "active",
  suspended: "suspended",
  deleted: "deleted"
}

// Int-backed enums
enum priority {
  low: 1,
  medium: 2,
  high: 3,
  critical: 4
}

// Usage
user = {
  name: "John",
  status: user_status.active    // Stores "active"
}

task = {
  title: "Fix bug",
  priority: priority.high       // Stores 3
}

// Access backing value
status_value = user_status.active.value    // "active"
priority_num = priority.high.value         // 3

// Compare
if user.status == user_status.active {
  print("User is active")
}

// Database storage - uses backing value
Database.insert("users", {
  name: "John",
  status: user_status.pending   // Stores "pending" in DB
})

// From backing value
status = user_status.from("active")        // user_status.active
priority = priority.from(3)                // priority.high

// JSON serialization - uses backing value
json.encode(user)   // { "name": "John", "status": "active" }
```

### Enum Methods

```typescript
enum role {
  admin: "admin",
  moderator: "moderator",
  user: "user",
  guest: "guest"
}

// Get all values
role.values()              // [role.admin, role.moderator, role.user, role.guest]

// Get all backing values
role.backing_values()      // ["admin", "moderator", "user", "guest"]

// From backing value (returns option)
parsed = role.from("admin")         // Some(role.admin)
parsed = role.from("invalid")       // None

// Or with result
parsed, err = role.from_result("admin")
if err {
  return err("Invalid role")
}
```

---

## Type Checking and Casting

### Type Checking Methods
```typescript
// Check types at runtime
value.is_int()              // true if int
value.is_float()            // true if float
value.is_str()              // true if string
value.is_bool()             // true if boolean
value.is_null()             // true if null
value.is_undefined()        // true if undefined
value.is_list()             // true if list
value.is_map()              // true if map
value.is_set()              // true if set
value.is_tuple()            // true if tuple
```

### Type Casting Functions
```typescript
// Built-in casting functions
int(value)                  // Cast to int
float(value)                // Cast to float
str(value)                  // Cast to string
bool(value)                 // Cast to boolean

// Examples
age = int("42")             // 42
price = float("19.99")      // 19.99
text = str(123)             // "123"

// Boolean casting - strict accepted values only
flag = bool("true")         // true
flag = bool("false")        // false
flag = bool(1)              // true
flag = bool(0)              // false

// These throw errors:
bool("")                    // Error: invalid boolean value
bool("yes")                 // Error: invalid boolean value
bool("1")                   // Error: use int() first, then bool()

// Casting returns result types - errors propagate automatically
expose process_age(input: str): result<int, error> {
  age = int(input)          // Errors propagate
  
  if age < 0 {
    return err("Age must be positive")
  }
  
  return age
}

// Explicit error handling when needed
age, err = int("abc")
if err {
  return err("Invalid age format")
}
```

### Collection Conversions
```typescript
// List/Set conversions
numbers = [1, 2, 2, 3]
unique = set(numbers)       // set([1, 2, 3])
back = list(unique)         // [1, 2, 3] (order undefined)

// Tuple to list
point = (10, 20)
coords = list(point)        // [10, 20]
```

### Comparison with Casting
```typescript
// Strict equality requires same type
3 == "3"                    // false (different types)
3 == int("3")               // true
str(3) == "3"               // true
```

### Schema Validation

**Validate unknown data against a type. Errors propagate automatically.**

```typescript
// Define types for API responses
type user_response {
  id: int,
  name: str,
  email?: str,
  age?: int
}

type order_response {
  id: int,
  total: float,
  items: list<order_item>,
  customer: customer_info
}

type order_item {
  product_id: int,
  quantity: int,
  price: float
}

type customer_info {
  id: int,
  name: str,
  email?: str
}

// Validate against type
data = json.parse(input).validate(user_response)
// data is now typed as user_response

// Use in functions
expose fetch_user(id: int): result<user, error> {
  response = await http.get("/users/{id}")
  data = json.parse(response.body).validate(user_response)
  
  return {
    id: data.id,
    name: data.name,
    age: data.age ?: 0,
    status: user_status.from(data.status) ?: user_status.pending
  }
}

expose fetch_order(id: int): result<order, error> {
  response = await http.get("/orders/{id}")
  data = json.parse(response.body).validate(order_response)
  
  return data
}

// Enums in types validate against backing values
type task_response {
  id: int,
  title: str,
  priority: priority,      // Validates 1, 2, 3, or 4
  status: task_status      // Validates "pending", "active", etc.
}

// Union types in validation
type flexible_response {
  id: int | str,           // Accept either
  count: int | null        // Accept int or null
}

// Custom error message
data = json.parse(input).validate(user_response, "Invalid API response")

// Filter by type (for truly dynamic data)
mixed = [1, "hello", 3.14, true, null]
numbers = mixed.filter((v) = v.is_int())     // [1]
strings = mixed.filter((v) = v.is_str())     // ["hello"]
```

---

## Functions

**No keywords.** Context-aware declarations.

### Function Syntax Rules

```typescript
// One-liner, type inferred - implicit return
add(a: int, b: int) = a + b

// One-liner, explicit type - implicit return
add(a: int, b: int): int = a + b

// One-liner with conditional - use ternary
max(a: int, b: int): int = a > b ? a : b

// Multi-line - explicit return required
add(a: int, b: int): int {
  result = a + b
  log("Adding {a} + {b}")
  return result
}
```

### Function Declaration Syntax

The parser treats `identifier(params) =` or `identifier(params) {` as a function declaration:

```typescript
// This is a function declaration that calls another function
result(data: str) = process(data)

// Variable assignment is different - no parameters
result = calculate(5)    // Calls calculate(), assigns return value to result
```

### Examples

```typescript
// Simple functions
double(n: int) = n * 2
greet(name: str) = "Hello, {name}!"

// Explicit return types
double(n: int): int = n * 2
format_name(first: str, last: str): str = "{first} {last}"

// Generic return types
fetch_users(limit: int = 10): list<user> = 
  Database.query("SELECT * FROM users LIMIT {limit}")

// Result types - one-liner
get_first_user(): result<user, error> = 
  Database.query("SELECT * FROM users").first()

// Result types - multi-line with explicit return
get_user(id: int): result<user, error> {
  user = Database.query("SELECT * FROM users WHERE id = ?", [id])
    .first()
  
  if user == null {
    return err("Not found")
  }
  
  return user
}

// Optional parameters
fetch_users(limit: int, offset?: int): list<user> {
  query = "SELECT * FROM users LIMIT {limit}"
  if offset != null {
    query += " OFFSET {offset}"
  }
  return Database.query(query)
}

// Anonymous functions (lambdas)
users.map((u) = u.name)
users.filter((u) = u.active)
items.reduce(0, (acc, i) = acc + i)
```

### Lambda Returns

```typescript
// Return exits the lambda, not the outer function
processed = items.map((item) = {
  if item.invalid {
    return null  // Returns null from this lambda only
  }
  return item.value
})

// Outer function continues after map completes
print(processed)
```

---

## Collections

### List (Ordered, Indexed, Duplicates)

```typescript
// Creation
numbers = [1, 2, 3, 1, 2]
users: list<user> = []
empty = []

// Common API
numbers.count()             // 5 (number of elements)
numbers.is_empty()          // false
numbers.clear()             // Remove all
numbers.has(2)              // true (contains check)

// Numeric operations
numbers.sum()               // 9
prices = [19.99, 29.99]
prices.sum()                // 49.98

// Access and modification
numbers.get(0)              // 1 (value at index)
numbers.set(0, 10)          // Set index 0 to 10
numbers.add(4)              // Append to end
numbers.delete(0)           // Remove at index 0

// Boundaries
numbers.first()             // First element
numbers.last()              // Last element
numbers.slice(1, 3)         // Sublist [start, end)

// Iteration
numbers.map((n) = n * 2)
numbers.filter((n) = n > 2)
numbers.reduce(0, (acc, n) = acc + n)
numbers.sort((a, b) = a <=> b)
numbers.reverse()
numbers.each((n) = print(n))
```

---

### Map (Key-Value, Unique Keys)

```typescript
// Creation
scores = { "alice": 100, "bob": 85 }
config: map<str, int> = {}
empty = {}

// Common API
scores.count()              // 2 (number of keys)
scores.is_empty()           // false
scores.clear()              // Remove all
scores.has("alice")         // true (key exists)

// Access and modification
scores.get("alice")         // option<int> (Some(100) or None)
scores.set("alice", 95)     // Set key "alice" to 95
scores.delete("alice")      // Remove key "alice"

// Views
scores.keys()               // list<str>
scores.values()             // list<int>
scores.entries()            // list<tuple<str, int>>

// Iteration
scores.map((k, v) = v * 2)
scores.filter((k, v) = v > 90)
scores.each((k, v) = print("{k}: {v}"))

// Getting with defaults
score = scores.get("alice") ?: 0
score = scores.get("alice") ?? 0
```

---

### Set (Unique Values, Unordered)

```typescript
// Creation
tags = set(["rust", "go", "python"])
ids: set<int> = set([])
empty = set([])

// Common API
tags.count()                // 3 (number of elements)
tags.is_empty()             // false
tags.clear()                // Remove all
tags.has("rust")            // true (value exists)

// Numeric operations (if numeric set)
ids = set([1, 2, 3])
ids.sum()                   // 6

// Modification
tags.add("ruby")            // Add value (no duplicates)
tags.add("rust")            // No-op, already exists
tags.delete("rust")         // Remove value

// Conversion
list(tags)                  // Convert to list

// Set operations
a = set([1, 2, 3])
b = set([2, 3, 4])

a.union(b)                  // set([1, 2, 3, 4])
a.intersect(b)              // set([2, 3])
a.difference(b)             // set([1])

// Iteration
tags.map((t) = t.upper())
tags.filter((t) = t.starts_with("r"))
tags.each((t) = print(t))
```

---

### Unified Collection API

| Method | List | Map | Set | Description |
|--------|------|-----|-----|-------------|
| `.count()` | ✓ | ✓ | ✓ | Number of elements/keys |
| `.is_empty()` | ✓ | ✓ | ✓ | Is collection empty |
| `.clear()` | ✓ | ✓ | ✓ | Remove all elements |
| `.has(value/key)` | ✓ | ✓ | ✓ | Check existence |
| `.sum()` | ✓ | - | ✓ | Sum numeric values |
| `.get(index/key)` | ✓ | ✓ | - | Get by index/key |
| `.set(index/key, val)` | ✓ | ✓ | - | Set by index/key |
| `.add(value)` | ✓ | - | ✓ | Add element |
| `.delete(idx/key/val)` | ✓ | ✓ | ✓ | Remove element |
| `.map(fn)` | ✓ | ✓ | ✓ | Transform elements |
| `.filter(fn)` | ✓ | ✓ | ✓ | Filter elements |
| `.each(fn)` | ✓ | ✓ | ✓ | Iterate with side effects |

---

### Collection Mutability

```typescript
// Collections are mutable by default (practical choice)
numbers = [1, 2, 3]
numbers.add(4)              // Mutates in place
numbers.set(0, 10)          // Mutates in place

// No .value needed for collection methods
// (Unlike primitive reassignment)

// Reassignment still requires .value
numbers = [5, 6, 7]         // Error
numbers.value = [5, 6, 7]   // OK
```

---

### Option Type for Map.get()

```typescript
// Map.get() returns option
scores = { "alice": 100 }

result = scores.get("alice")
// result is option<int>

// Use default operators
score = scores.get("alice") ?: 0
score = scores.get("alice") ?? 0
```

---

## Operators

### Arithmetic
```typescript
a + b       // Addition
a - b       // Subtraction
a * b       // Multiplication
a / b       // Division
a % b       // Modulo (remainder)
a ** b      // Exponentiation (power)

// Examples
sum = 5 + 3           // 8
power = 2 ** 8        // 256
```

### Comparison
```typescript
a == b      // Equal to (strict: checks type AND value)
a != b      // Not equal to (strict)
a < b       // Less than
a > b       // Greater than
a <= b      // Less than or equal
a >= b      // Greater than or equal
a <=> b     // Spaceship (three-way comparison)

// Note: No === or !== operators
// == always checks both type and value (strict equality)
5 == 5        // true
5 == "5"      // false (different types)
0 == false    // false (different types)
null == null  // true
"" == false   // false (different types)

// Spaceship returns:
// -1 if a < b
// 0  if a == b
// 1  if a > b

// Examples
5 <=> 10    // -1
10 <=> 10   // 0
15 <=> 10   // 1

// Sorting with spaceship
users.sort((a, b) = a.age <=> b.age)
posts.sort((a, b) = b.date <=> a.date)  // Descending

// Comparison in match
winner = match score1 <=> score2 {
  -1 = "Player 2 wins",
  0 = "Tie",
  1 = "Player 1 wins"
}
```

### Logical
```typescript
a && b      // Logical AND
a || b      // Logical OR
!a          // Logical NOT

// Short-circuit evaluation
user.active && user.verified
user.admin || user.moderator
```

### Assignment
```typescript
name = value           // Immutable assignment
name.value = value     // Mutable assignment

// Compound assignment (requires .value)
count.value += 1
count.value -= 5
count.value *= 2
count.value /= 3
```

### Range
```typescript
start..end            // Inclusive range [start, end]
start..<end           // Exclusive range [start, end)

// Examples
for i in 0..10 { }    // 0 through 10 (inclusive)
for i in 0..<10 { }   // 0 through 9 (exclusive)
slice = items[0..<5]  // First 5 items
```

### Optional/Nullable
```typescript
?           // Nullable type marker
?.          // Optional chaining
??          // Null coalescing (ONLY null/undefined)
?:          // Elvis operator (ANY falsy value)

// Nullable marker
email?: str

// Optional chaining
city = user?.address?.city

// Null coalescing - ONLY checks null/undefined
name = user?.name ?? "Anonymous"
port = config?.port ?? 8080

// Values pass through if not null/undefined:
count = 0
result = count ?? 100        // result = 0 (0 is not null)

flag = false
result = flag ?? true        // result = false (false is not null)

text = ""
result = text ?? "default"   // result = "" (empty string is not null)

// Elvis operator - checks ANY falsy value (null, undefined, false, 0, "", etc.)
port = config.port ?: 8080
title = post.title ?: "Untitled"

// Replaces falsy values:
count = 0
result = count ?: 100        // result = 100 (0 is falsy)

flag = false
result = flag ?: true        // result = true (false is falsy)

text = ""
result = text ?: "default"   // result = "default" (empty string is falsy)

// Comparison:
value = 0
a = value ?? 100   // a = 0 (because 0 is not null)
b = value ?: 100   // b = 100 (because 0 is falsy)

value = ""
a = value ?? "default"  // a = "" (not null)
b = value ?: "default"  // b = "default" (falsy)
```

### String
```typescript
+           // String concatenation
{expr}      // String interpolation

greeting = "Hello" + " " + "World"
message = "Hello, {name}!"
```

### Member Access
```typescript
.           // Property access
?.          // Optional property access
.value      // Mutation accessor

user.name
user?.email
count.value = 5
```

### Function
```typescript
=           // Lambda/function definition
()          // Function call
.           // Method call

// Examples
users.map((u) = u.name)
calculate(5, 10)
items.filter((i) = i.active)
```

### Type
```typescript
:           // Type annotation
<>          // Generic type parameters
|           // Union type
?           // Optional/nullable

name: str
users: list<user>
config: map<str, list<int>>
id: int | str
email?: str
```

### Match/Control
```typescript
=           // Match arm result
_           // Wildcard pattern

match status {
  "active" = handle_active(),
  "pending" = handle_pending(),
  _ = handle_default()
}
```

### Ternary
```typescript
? :         // Ternary operator

status = active ? "online" : "offline"
```

### Decorator/Event
```typescript
@           // Event handler

<button @click={increment}>Click</button>
```

### Precedence (Highest to Lowest)
```typescript
1.  ()  []  .  ?.                   // Grouping, indexing, member access
2.  !  -  (unary)                   // Unary operators
3.  **                              // Exponentiation
4.  *  /  %                         // Multiplication, division, modulo
5.  +  -                            // Addition, subtraction
6.  ..  ..<                         // Range
7.  <  <=  >  >=  <=>               // Comparison (including spaceship)
8.  ==  !=                          // Equality (strict)
9.  &&                              // Logical AND
10. ||                              // Logical OR
11. ??                              // Null coalescing
12. ?:                              // Elvis operator
13. ? :                             // Ternary
14. =                               // Assignment, lambda definition
```

---

## Control Flow

### If (Guard Clauses)

Rivo uses `if` for guard clauses and early returns only. There is no `else` keyword.

```typescript
// Guard clauses - check and return early
if !user.verified {
  return err("Not verified")
}

if user.banned {
  return err("User banned")
}

// Happy path continues here...
process_user(user)

// Single condition side effects
if loading {
  show_spinner()
}
```

### Why No `else`?

Rivo intentionally omits `else`. This encourages cleaner patterns:

- **Guard clauses** - Handle error cases early, keep happy path at main indentation
- **Ternary** - For simple either/or value expressions
- **Match** - For multi-branch logic

```typescript
// ❌ Instead of nested if-else:
// if condition {
//   do_a()
// } else {
//   do_b()
// }

// ✅ Use ternary for values:
result = condition ? value_a : value_b

// ✅ Use guard clause + continue:
if !condition {
  return
}
do_a()

// ✅ Use match for multiple branches:
result = match {
  condition_a = value_a,
  condition_b = value_b,
  _ = default_value
}
```

### Ternary (Either/Or)
```typescript
status = active ? "online" : "offline"
message = error ? "Failed" : "Success"

// In components
<div>
  {loading ? <Spinner /> : <Content />}
  {user ? <Profile user={user} /> : <Login />}
</div>
```

### Elvis (Truthy Fallback)
```typescript
// Use when you want to replace ANY falsy value
display = user.name ?: "Anonymous"
port = config.port ?: 8080
retries = attempts ?: 3

// Different from null coalescing
count = items.count() ?? 0   // 0 stays 0
count = items.count() ?: 1   // 0 becomes 1
```

### Match (Multi-Branch)
```typescript
// Pattern matching
action = match user.role {
  "admin" = admin_panel(),
  "moderator" = mod_panel(),
  "user" = user_panel(),
  _ = guest_panel()
}

// Enum matching
message = match user.status {
  user_status.pending = "Awaiting approval",
  user_status.active = "Account active",
  user_status.suspended = "Account suspended",
  user_status.deleted = "Account deleted"
}

// With guards (condition-based matching)
grade = match {
  score >= 90 = "A",
  score >= 80 = "B",
  score >= 70 = "C",
  score >= 60 = "D",
  _ = "F"
}

// Spaceship in match
result = match a <=> b {
  -1 = "Less than",
  0 = "Equal",
  1 = "Greater than"
}

// In components (multi-branch rendering)
<div>
  {match {
    loading = <p>Loading...</p>,
    error != null = <p class="error">{error}</p>,
    _ = <Content data={data} />
  }}
</div>
```

### Loops
```typescript
// For-in loop (type inferred from collection)
for user in users {
  print(user.name)
}

// Range loops
for i in 0..10 {
  print(i)
}

for i in 0..<10 {
  print(i)
}

// While loop
while condition {
  process()
}

// Break and continue work as expected
for item in items {
  if item.skip {
    continue
  }
  if item.done {
    break
  }
  process(item)
}
```

### Method Chaining
```typescript
users
  .filter((u) = u.active)
  .map((u) = u.email)
  .sort((a, b) = a <=> b)
```

---

## Error Handling

### Result Type and Automatic Propagation

**When a function returns `result<T, error>`, calling other result-returning functions automatically propagates errors:**

```typescript
// Errors propagate automatically in result functions
expose get_user(id: int): result<user, error> {
  user = Database.query("SELECT * FROM users WHERE id = ?", [id]).first()
  
  if user == null {
    return err("User not found")
  }
  
  return user
}

expose update_user(id: int, name: str): result<user, error> {
  // get_user error propagates automatically
  existing = get_user(id)
  
  // Database error propagates automatically
  return Database.update("users", id, { name: name })
}
```

### Tuple Destructuring for Explicit Error Handling

**Use Go-style tuple destructuring when you need explicit error handling:**

```typescript
// Explicit error handling with tuple destructuring
async load_user(user_id: int) {
  loading.value = true
  error.value = null
  
  data, err = await get_user(user_id)
  if err {
    error.value = err
    loading.value = false
    return
  }
  
  user.value = data
  loading.value = false
}

// Guard clauses
async save_user(name: str, email: str) {
  updated, err = await update_user(user_id, name, email)
  if err {
    error.value = "Save failed: {err}"
    return
  }
  
  user.value = updated
  editing.value = false
}
```

### Guard Clauses
```typescript
validate_user(user: user): result<bool, error> {
  if user.email.is_empty() {
    return err("Email required")
  }
  
  if !user.email.includes("@") {
    return err("Invalid email")
  }
  
  if user.age < 18 {
    return err("Must be 18+")
  }
  
  return true
}
```

---

## Server-Client Communication

### Expose API Endpoints

**Use `expose` keyword to mark server functions as API endpoints:**

```typescript
// user_service.sx (Server)

enum user_status {
  pending: "pending",
  active: "active",
  suspended: "suspended",
  deleted: "deleted"
}

type user {
  id: int,
  name: str,
  email: str,
  age: int,
  status: user_status,
  created_at: datetime
}

// Private function (server-only)
validate_email(email: str): result<bool, error> {
  if email.is_empty() {
    return err("Email required")
  }
  
  if !email.includes("@") {
    return err("Invalid email")
  }
  
  return true
}

// Exposed - available to client
expose get_user(id: int): result<user, error> {
  user = Database.query("SELECT * FROM users WHERE id = ?", [id]).first()
  
  if user == null {
    return err("User not found")
  }
  
  return user
}

expose create_user(name: str, email: str, age: int): result<user, error> {
  validate_email(email)
  
  if age < 18 {
    return err("Must be 18 or older")
  }
  
  return Database.insert("users", {
    name: name,
    email: email,
    age: age,
    status: user_status.pending,
    created_at: now()
  })
}

expose update_user_status(id: int, status: user_status): result<user, error> {
  existing = get_user(id)
  
  return Database.update("users", id, {
    status: status
  })
}

expose list_users(limit: int = 10): result<list<user>, error> {
  return Database.query("SELECT * FROM users LIMIT {limit}").all()
}
```

### Import and Call from Components

**Import exposed functions in both `.sc` and `.cx` files:**

```typescript
// user_card.sc (Server Component - SSR)
import { get_user, user_status } from "./user_service.sx"

component UserCard(user_id: int) {
  // Runs on server during render - direct call, no HTTP
  user, err = await get_user(user_id)
  
  if err {
    return <p class="error">Failed to load user</p>
  }
  
  <div class="user-card">
    <h2>{user.name}</h2>
    <p>{user.email}</p>
    <span class="status">{match user.status {
      user_status.pending = "⏳ Pending",
      user_status.active = "✅ Active",
      user_status.suspended = "🚫 Suspended",
      user_status.deleted = "❌ Deleted"
    }}</span>
  </div>
}
```

```typescript
// status_toggle.cx (Client Component - Interactive)
import { update_user_status, user_status } from "./user_service.sx"

component StatusToggle(user_id: int, initial_status: user_status) {
  status = initial_status
  loading = false
  error?: str = null
  
  async toggle() {
    loading.value = true
    error.value = null
    
    new_status = status == user_status.active 
      ? user_status.suspended 
      : user_status.active
    
    // Compiler generates HTTP client call
    updated, err = await update_user_status(user_id, new_status)
    if err {
      error.value = err
      loading.value = false
      return
    }
    
    status.value = updated.status
    loading.value = false
  }
  
  <div>
    {error && <p class="error">{error}</p>}
    <button @click={toggle} disabled={loading}>
      {loading ? "..." : (status == user_status.active ? "Suspend" : "Activate")}
    </button>
  </div>
}
```

```typescript
// profile_page.sc (Server Component with Client Interactivity)
import { get_user, user_status } from "./user_service.sx"
import { StatusToggle } from "./status_toggle.cx"

component ProfilePage(user_id: int) {
  user, err = await get_user(user_id)
  
  if err {
    return <p>User not found</p>
  }
  
  // Static content rendered on server
  // StatusToggle hydrates on client for interactivity
  <div class="profile">
    <h1>{user.name}</h1>
    <p>{user.email}</p>
    <p>Age: {user.age}</p>
    <p>Member since: {user.created_at.format("MMMM D, YYYY")}</p>
    
    // Client component embedded in server component
    <StatusToggle user_id={user_id} initial_status={user.status} />
  </div>
}
```

### How It Works

**Server (`.sx`):**
- `expose` keyword marks functions as API endpoints
- Regular functions stay server-only
- Compiler generates HTTP endpoints: `/api/user_service/get_user`
- Enums serialize to backing values in JSON

**Server Components (`.sc`):**
- Render HTML on server
- Can call `.sx` functions directly (no HTTP overhead)
- Can embed `.cx` components for interactivity
- Output is sent as HTML + hydration data

**Client Components (`.cx`):**
- Run entirely in browser
- Import exposed functions - compiler generates HTTP fetch calls
- Full interactivity with DOM access
- Same function signature as server

**Benefits:**
- ✅ **Explicit** - `expose` shows API surface
- ✅ **Type-safe** - Full checking across boundary
- ✅ **No magic** - Direct function imports
- ✅ **Secure** - Non-exposed functions can't be called
- ✅ **Simple** - No separate API layer needed
- ✅ **Enums travel** - Share enums between server and client
- ✅ **SSR + Hydration** - Best of both worlds with `.sc` + `.cx`

---

## Components

### Server Components (.sc)

Server components render on the server and send HTML to the client. They can fetch data directly and embed client components for interactivity.

```typescript
// page_layout.sc
import { get_current_user } from "./auth_service.sx"
import { NavBar } from "./nav_bar.cx"

component PageLayout(children: node) {
  user, err = await get_current_user()
  
  <html>
    <head>
      <title>My App</title>
    </head>
    <body>
      // NavBar is a client component - hydrates for interactivity
      <NavBar user={user} />
      <main>
        {children}
      </main>
    </body>
  </html>
}
```

```typescript
// user_list.sc
import { list_users, user_status } from "./user_service.sx"

component UserList(limit: int = 10) {
  users, err = await list_users(limit)
  
  if err {
    return <p class="error">Failed to load users</p>
  }
  
  if users.is_empty() {
    return <p>No users found</p>
  }
  
  <ul class="user-list">
    {users.map((user) = (
      <li key={user.id}>
        <a href="/users/{user.id}">{user.name}</a>
        <span>{user.email}</span>
        <span class="status {user.status.value}">{user.status.value}</span>
      </li>
    ))}
  </ul>
}
```

### Client Components (.cx)

Client components run in the browser and have full interactivity.

```typescript
// counter.cx
component Counter(initial: int = 0) {
  count = initial
  
  increment() {
    count.value += 1
  }
  
  decrement() {
    count.value -= 1
  }
  
  <div class="counter">
    <button @click={decrement}>-</button>
    <span>{count}</span>
    <button @click={increment}>+</button>
  </div>
}
```

### Event Handling
```typescript
// form.cx
component ContactForm() {
  name = ""
  email = ""
  message = ""
  errors: list<str> = []
  submitting = false
  
  validate(): bool {
    errors.value = []
    
    if name.count() < 2 {
      errors.add("Name too short")
    }
    
    if !email.includes("@") {
      errors.add("Invalid email")
    }
    
    if message.count() < 10 {
      errors.add("Message too short")
    }
    
    return errors.is_empty()
  }
  
  async submit(e) {
    e.prevent_default()
    
    if !validate() {
      return
    }
    
    submitting.value = true
    
    result, err = await send_contact_message({ name, email, message })
    if err {
      errors.add("Failed to send: {err}")
      submitting.value = false
      return
    }
    
    // Reset form
    name.value = ""
    email.value = ""
    message.value = ""
    submitting.value = false
  }
  
  <form @submit={submit}>
    {!errors.is_empty() && (
      <div class="errors">
        {errors.map((e) = <p class="error">{e}</p>)}
      </div>
    )}
    
    <input 
      type="text"
      placeholder="Name"
      value={name}
      @input={(e) = name.value = e.target.value}
    />
    
    <input 
      type="email"
      placeholder="Email"
      value={email}
      @input={(e) = email.value = e.target.value}
    />
    
    <textarea
      placeholder="Message"
      value={message}
      @input={(e) = message.value = e.target.value}
    />
    
    <button type="submit" disabled={submitting}>
      {submitting ? "Sending..." : "Send"}
    </button>
  </form>
}
```

### Conditional Rendering
```typescript
// user_profile.cx
component UserProfile(user?: user) {
  <div>
    {user ? (
      <div>
        <h1>{user.name}</h1>
        <p>{user.email}</p>
      </div>
    ) : (
      <p>No user found</p>
    )}
  </div>
}
```

### Mixing Server and Client Components
```typescript
// dashboard.sc
import { get_stats, get_recent_activity } from "./analytics_service.sx"
import { LiveChart } from "./live_chart.cx"
import { NotificationBell } from "./notification_bell.cx"

component Dashboard() {
  stats, _ = await get_stats()
  activity, _ = await get_recent_activity(10)
  
  <div class="dashboard">
    <header>
      <h1>Dashboard</h1>
      // Client component for real-time notifications
      <NotificationBell />
    </header>
    
    // Static stats rendered on server
    <div class="stats">
      <div class="stat">
        <span class="label">Users</span>
        <span class="value">{stats.total_users}</span>
      </div>
      <div class="stat">
        <span class="label">Revenue</span>
        <span class="value">${stats.revenue}</span>
      </div>
    </div>
    
    // Client component for interactive chart
    <LiveChart initial_data={activity} />
    
    // Static list rendered on server
    <ul class="activity">
      {activity.map((item) = (
        <li key={item.id}>{item.description}</li>
      ))}
    </ul>
  </div>
}
```

---

## Modules

```typescript
// Export types and functions (available within project)
export get_user(id: int): result<user, error> { }
export max_users = 1000
export default UserService

// Export enums
export enum user_status {
  pending: "pending",
  active: "active"
}

// Expose (API endpoint - callable from .sc and .cx)
expose get_user(id: int): result<user, error> { }

// Import
import { get_user, create_user } from "./user_service.sx"
import * as UserService from "./user_service.sx"
import type { user } from "./types.sx"
import { user_status } from "./user_service.sx"

// Import components
import { UserCard } from "./user_card.sc"
import { Counter } from "./counter.cx"
```

---

## Async/Await

```typescript
async fetch_user(id: int): result<user, error> {
  response = await http.get("/users/{id}")
  return json.decode(response.body)
}

// Parallel execution
[user, posts, friends] = await [
  fetch_user(id),
  fetch_posts(id),
  fetch_friends(id)
]

// With tuple destructuring
async load_data() {
  user_data, user_err = await fetch_user(id)
  if user_err {
    return
  }
  
  posts_data, posts_err = await fetch_posts(id)
  if posts_err {
    return
  }
}
```

---

## String Interpolation

```typescript
name = "John"
message = "Hello, {name}!"

count = 5
text = "You have {count * 2} items"

// Multi-line
query = "
  SELECT * FROM users
  WHERE active = true
"

// Raw strings
regex = r"\d+\.\d+"
```

---

## Comments

```typescript
// Single line

/* Multi-line
   comment */

/**
 * Documentation
 * @param id - User ID
 */
```

---

## Reserved Keywords

```
async, await, bool, break, catch, component, continue,
date, datetime, enum, err, error, export, expose, false, 
float, for, if, import, in, int, list, map, match, null, 
on_mount, on_unmount, option, result, return, set, str, 
time, true, try, tuple, type, undefined, unknown, while
```

---

## Complete Example

```typescript
// types.sx (Shared types)

enum user_status {
  pending: "pending",
  active: "active",
  suspended: "suspended",
  deleted: "deleted"
}

enum user_role {
  admin: "admin",
  moderator: "moderator",
  user: "user",
  guest: "guest"
}

type user {
  id: int,
  name: str,
  email: str,
  age: int,
  status: user_status,
  role: user_role,
  created_at: datetime
}

type create_user_request {
  name: str,
  email: str,
  age: int
}
```

```typescript
// user_service.sx (Server - Data Layer)

import { user, user_status, user_role, create_user_request } from "./types.sx"

validate_email(email: str): result<bool, error> {
  if email.is_empty() {
    return err("Email required")
  }
  
  if !email.includes("@") {
    return err("Invalid email format")
  }
  
  return true
}

expose get_user(id: int): result<user, error> {
  user = Database.query("SELECT * FROM users WHERE id = ?", [id]).first()
  
  if user == null {
    return err("User not found")
  }
  
  return user
}

expose create_user(request: create_user_request): result<user, error> {
  validate_email(request.email)
  
  if request.age < 18 {
    return err("Must be 18 or older")
  }
  
  user = Database.insert("users", {
    name: request.name,
    email: request.email,
    age: request.age,
    status: user_status.pending,
    role: user_role.user,
    created_at: now()
  })
  
  return user
}

expose list_users(limit: int = 10, offset: int = 0): result<list<user>, error> {
  return Database.query(
    "SELECT * FROM users LIMIT {limit} OFFSET {offset}"
  ).all()
}

expose update_user_status(id: int, status: user_status): result<user, error> {
  existing = get_user(id)
  
  return Database.update("users", id, {
    status: status
  })
}

expose update_user_role(id: int, role: user_role): result<user, error> {
  existing = get_user(id)
  
  return Database.update("users", id, {
    role: role
  })
}

log_user_action(user_id: int, action: str) {
  Database.insert("audit_log", {
    user_id: user_id,
    action: action,
    timestamp: now()
  })
}
```

```typescript
// external_api.sx (Server - External API Integration)

import { user, user_status, create_user_request } from "./types.sx"
import { create_user } from "./user_service.sx"

// Type for external API response (may differ from internal user type)
type external_user_response {
  name: str,
  email?: str,
  age: int | str    // External API is inconsistent
}

expose fetch_external_user(api_id: str): result<user, error> {
  response = await http.get("https://api.example.com/users/{api_id}")
  
  // Validate against type - clean and declarative
  data = json.parse(response.body).validate(external_user_response)
  
  // Handle flexible age type
  age = data.age.is_int() ? data.age : int(data.age)
  
  return create_user({
    name: data.name,
    email: data.email ?: "",
    age: age
  })
}
```

```typescript
// user_card.sc (Server Component - SSR)

import { user, user_status } from "./types.sx"

component UserCard(user: user) {
  <div class="user-card">
    <h2>{user.name}</h2>
    <p>{user.email}</p>
    <p>Age: {user.age}</p>
    <span class="badge {user.status.value}">
      {match user.status {
        user_status.pending = "⏳ Pending",
        user_status.active = "✅ Active",
        user_status.suspended = "🚫 Suspended",
        user_status.deleted = "❌ Deleted"
      }}
    </span>
  </div>
}
```

```typescript
// user_list.sc (Server Component - SSR)

import { list_users } from "./user_service.sx"
import { UserCard } from "./user_card.sc"

component UserList(limit: int = 10) {
  users, err = await list_users(limit)
  
  if err {
    return <p class="error">Failed to load users: {err}</p>
  }
  
  if users.is_empty() {
    return <p>No users found</p>
  }
  
  <div class="user-list">
    {users.map((user) = <UserCard key={user.id} user={user} />)}
  </div>
}
```

```typescript
// status_editor.cx (Client Component - Interactive)

import { update_user_status, update_user_role, user_status, user_role } from "./user_service.sx"

component StatusEditor(user_id: int, initial_status: user_status, initial_role: user_role) {
  status = initial_status
  role = initial_role
  loading = false
  error?: str = null
  
  async change_status(new_status: user_status) {
    loading.value = true
    error.value = null
    
    updated, err = await update_user_status(user_id, new_status)
    if err {
      error.value = err
      loading.value = false
      return
    }
    
    status.value = updated.status
    loading.value = false
  }
  
  async change_role(new_role: user_role) {
    loading.value = true
    error.value = null
    
    updated, err = await update_user_role(user_id, new_role)
    if err {
      error.value = err
      loading.value = false
      return
    }
    
    role.value = updated.role
    loading.value = false
  }
  
  <div class="status-editor">
    {error && <p class="error">{error}</p>}
    
    <div class="field">
      <label>Status:</label>
      <div class="buttons">
        {status == user_status.active && (
          <button @click={() = change_status(user_status.suspended)} disabled={loading}>
            Suspend
          </button>
        )}
        {status == user_status.suspended && (
          <button @click={() = change_status(user_status.active)} disabled={loading}>
            Activate
          </button>
        )}
        {status == user_status.pending && (
          <button @click={() = change_status(user_status.active)} disabled={loading}>
            Approve
          </button>
        )}
      </div>
    </div>
    
    <div class="field">
      <label>Role:</label>
      <select 
        @change={(e) = change_role(user_role.from(e.target.value))}
        disabled={loading}
      >
        {user_role.values().map((r) = (
          <option value={r.value} selected={role == r}>
            {r.value}
          </option>
        ))}
      </select>
    </div>
  </div>
}
```

```typescript
// profile_page.sc (Server Component with Client Interactivity)

import { get_user } from "./user_service.sx"
import { user_status, user_role } from "./types.sx"
import { StatusEditor } from "./status_editor.cx"

component ProfilePage(user_id: int) {
  user, err = await get_user(user_id)
  
  if err {
    return (
      <div class="error-page">
        <h1>User Not Found</h1>
        <p>{err}</p>
        <a href="/users">Back to Users</a>
      </div>
    )
  }
  
  <div class="profile-page">
    <header>
      <h1>{user.name}</h1>
      <p class="email">{user.email}</p>
    </header>
    
    <section class="details">
      <h2>Details</h2>
      <dl>
        <dt>Age</dt>
        <dd>{user.age}</dd>
        
        <dt>Member Since</dt>
        <dd>{user.created_at.format("MMMM D, YYYY")}</dd>
        
        <dt>Current Status</dt>
        <dd>{user.status.value}</dd>
        
        <dt>Current Role</dt>
        <dd>{user.role.value}</dd>
      </dl>
    </section>
    
    <section class="admin">
      <h2>Admin Controls</h2>
      // Client component for interactivity
      <StatusEditor 
        user_id={user_id} 
        initial_status={user.status}
        initial_role={user.role}
      />
    </section>
  </div>
}
```

```typescript
// app.sc (Root Layout)

import { UserList } from "./user_list.sc"
import { ProfilePage } from "./profile_page.sc"

component App(route: str, params: map<str, str>) {
  <html>
    <head>
      <title>User Management</title>
      <link rel="stylesheet" href="/styles.css" />
    </head>
    <body>
      <nav>
        <a href="/">Home</a>
        <a href="/users">Users</a>
      </nav>
      
      <main>
        {match route {
          "/" = <h1>Welcome</h1>,
          "/users" = <UserList limit={20} />,
          "/users/:id" = <ProfilePage user_id={int(params.get("id"))} />,
          _ = <h1>404 - Not Found</h1>
        }}
      </main>
    </body>
  </html>
}
```

---

## Project Structure

```
project/
├── src/
│   ├── server/                  # Server-only code (.sx)
│   │   ├── user_service.sx
│   │   ├── auth_service.sx
│   │   ├── external_api.sx
│   │   └── database.sx
│   ├── components/
│   │   ├── server/              # Server components (.sc)
│   │   │   ├── UserCard.sc
│   │   │   ├── UserList.sc
│   │   │   ├── ProfilePage.sc
│   │   │   └── Layout.sc
│   │   └── client/              # Client components (.cx)
│   │       ├── StatusEditor.cx
│   │       ├── Counter.cx
│   │       ├── LoginForm.cx
│   │       └── LiveChart.cx
│   ├── pages/                   # Page components (typically .sc)
│   │   ├── home.sc
│   │   ├── users.sc
│   │   ├── profile.sc
│   │   └── settings.cx          # Client-only page if needed
│   └── shared/
│       ├── types.sx             # Shared types
│       └── enums.sx             # Shared enums
├── tests/
│   ├── user_service_test.sx
│   ├── user_card_test.sc
│   └── status_editor_test.cx
├── public/
│   ├── styles.css
│   └── images/
└── rivo.config.sx
```

---

## Best Practices

### Use Guard Clauses
Check error conditions early and return. Happy path stays at the main level. No `else` needed.

### Tuple Destructuring for Error Handling
Use `data, err = function()` pattern for explicit error handling in client code.

### Automatic Error Propagation
In `result`-returning functions, errors propagate automatically - no manual checking needed.

### Use Schema Validation
Validate JSON and unknown data against types using `.validate(type)` for clean, declarative parsing.

### Use Strict Equality
Rivo only has `==` and it's always strict (checks type and value). No need to remember `===`.

### Cast Types Explicitly
Use `int()`, `str()`, `float()`, `bool()` for explicit type conversion. No implicit coercion.

### Use Backed Enums
Define enums with explicit backing values for database storage and API serialization.

### Expose Only What's Needed
Use `
