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

| Extension | Context | Access |
|-----------|---------|--------|
| `.sx` | Server execution | Database, filesystem, env vars, secrets |
| `.cx` | Client execution | DOM, browser APIs, local storage |

**Rules:**
- `.sx` files never bundle to client (compiler enforced)
- `.cx` files cannot import `.sx` modules (compiler enforced)
- Communication between contexts happens through explicit APIs

```typescript
// user_service.sx - Server only
expose get_user(id: int): result: user, error {
  user = Database.query("SELECT * FROM users WHERE id = ?", [id]).first()
  
  if user == null {
    return err("User not found")
  }
  
  return user
}

// profile.cx - Client only
import { get_user } from "./user_service.sx"

// Compiler generates HTTP client call
data, err = await get_user(123)
```

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

// Mutation with .val
count = 0
count.val = 5      // Declare intent to mutate
count.val += 1     // Mutate

// Reading (no .val)
print(count)       // Outputs: 6
total = count + 5

// Compiler optimizes constants automatically
```

---

## Type System

### Primitives
```typescript
int, float, str, bool
datetime, date, time
```

### Collections
```typescript
list: T          // Ordered, indexed, duplicates allowed
map: K, V        // Key-value pairs, unique keys
set: T           // Unique values, unordered
```

### Type Inference from Literals
```typescript
// Inferred from literal
numbers = [1, 2, 3]                    // list: int
scores = { "a": 1, "b": 2 }            // map: str, int
tags = set(["rust", "go"])             // set: str

// Explicit types when needed
users: list: user = []
config: map: str, str = {}
ids: set: int = set([])
```

### Special Types
```typescript
result: T, E    // Success or error
option: T       // Some or None
unknown         // Dynamic/unknown type (for JSON, etc.)
```

### Nullable
```typescript
name: str       // Cannot be null
email?: str     // Can be null

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
value.is_list()             // true if list
value.is_map()              // true if map
value.is_set()              // true if set
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
flag = bool("true")         // true

// Casting returns result types - errors propagate automatically
expose process_age(input: str): result: int, error {
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
```

### Comparison with Casting
```typescript
// Strict equality requires same type
3 == "3"                    // false (different types)
3 == int("3")               // true
str(3) == "3"               // true
```

### Working with JSON and Unknown Types
```typescript
// Parse API response (returns unknown type)
expose fetch_user(id: int): result: user, error {
  response = await http.get("/users/{id}")
  data = json.parse(response.body)
  
  if !data.is_map() {
    return err("Invalid response format")
  }
  
  if !data.has("id") || !data.get("id").is_int() {
    return err("Missing or invalid id")
  }
  
  if !data.has("name") || !data.get("name").is_str() {
    return err("Missing or invalid name")
  }
  
  // Handle flexible types - cast as needed
  age_value = data.get("age")
  age = age_value.is_int() ? age_value : int(age_value)
  
  // Parse enum from backing value
  status_str = str(data.get("status"))
  status = user_status.from(status_str) ?: user_status.pending
  
  return {
    id: data.get("id"),
    name: data.get("name"),
    age: age,
    status: status
  }
}

// Filter by type
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

// Multi-line - explicit return required
add(a: int, b: int): int {
  result = a + b
  log("Adding {a} + {b}")
  return result
}
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
fetch_users(limit: int = 10): list: user = 
  Database.query("SELECT * FROM users LIMIT {limit}")

// Result types - one-liner
get_first_user(): result: user, error = 
  Database.query("SELECT * FROM users").first()

// Result types - multi-line with explicit return
get_user(id: int): result: user, error {
  user = Database.query("SELECT * FROM users WHERE id = ?", [id])
    .first()
  
  if user == null {
    return err("Not found")
  }
  
  return user
}

// Optional parameters
fetch_users(limit: int, offset?: int): list: user {
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

---

## Collections

### List (Ordered, Indexed, Duplicates)

```typescript
// Creation
numbers = [1, 2, 3, 1, 2]
users: list: user = []
empty = []

// Common API
numbers.count()             // 5 (number of elements)
numbers.empty()             // false
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
numbers.remove(2)           // Remove first occurrence of value 2

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
config: map: str, int = {}
empty = {}

// Common API
scores.count()              // 2 (number of keys)
scores.empty()              // false
scores.clear()              // Remove all
scores.has("alice")         // true (key exists)

// Access and modification
scores.get("alice")         // option: int (Some(100) or None)
scores.set("alice", 95)     // Set key "alice" to 95
scores.delete("alice")      // Remove key "alice"

// Views
scores.keys()               // list: str
scores.values()             // list: int
scores.entries()            // list: (str, int)

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
ids: set: int = set([])
empty = set([])

// Common API
tags.count()                // 3 (number of elements)
tags.empty()                // false
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
| `.empty()` | ✓ | ✓ | ✓ | Is collection empty |
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

// No .val needed for collection methods
// (Unlike primitive reassignment)

// Reassignment still requires .val
numbers = [5, 6, 7]         // Error
numbers.val = [5, 6, 7]     // OK
```

---

### Option Type for Map.get()

```typescript
// Map.get() returns option
scores = { "alice": 100 }

result = scores.get("alice")
// result is option: int

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
name.val = value       // Mutable assignment

// Compound assignment (requires .val)
count.val += 1
count.val -= 5
count.val *= 2
count.val /= 3
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
.val        // Mutation accessor

user.name
user?.email
count.val = 5
```

### Function
```typescript
=           // Lambda definition
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
|           // Union type
?           // Optional/nullable

name: str
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

### If (Single Check)
```typescript
// Guard clauses
if !user.verified {
  return err("Not verified")
}

if user.banned {
  return err("User banned")
}

// Single condition
if loading {
  show_spinner()
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

// With guards
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

**When a function returns `result: T, error`, calling other result-returning functions automatically propagates errors:**

```typescript
// Errors propagate automatically in result functions
expose get_user(id: int): result: user, error {
  user = Database.query("SELECT * FROM users WHERE id = ?", [id]).first()
  
  if user == null {
    return err("User not found")
  }
  
  return user
}

expose update_user(id: int, name: str): result: user, error {
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
  loading.val = true
  error.val = null
  
  data, err = await get_user(user_id)
  if err {
    error.val = err
    loading.val = false
    return
  }
  
  user.val = data
  loading.val = false
}

// Guard clauses
async save_user(name: str, email: str) {
  updated, err = await update_user(user_id, name, email)
  if err {
    error.val = "Save failed: {err}"
    return
  }
  
  user.val = updated
  editing.val = false
}
```

### Guard Clauses
```typescript
validate_user(user: user): result: bool, error {
  if user.email.empty() {
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
validate_email(email: str): result: bool, error {
  if email.empty() {
    return err("Email required")
  }
  
  if !email.includes("@") {
    return err("Invalid email")
  }
  
  return true
}

// Exposed - available to client
expose get_user(id: int): result: user, error {
  user = Database.query("SELECT * FROM users WHERE id = ?", [id]).first()
  
  if user == null {
    return err("User not found")
  }
  
  return user
}

expose create_user(name: str, email: str, age: int): result: user, error {
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

expose update_user_status(id: int, status: user_status): result: user, error {
  existing = get_user(id)
  
  return Database.update("users", id, {
    status: status
  })
}

expose list_users(limit: int = 10): result: list: user, error {
  return Database.query("SELECT * FROM users LIMIT {limit}").all()
}
```

### Import and Call from Client

**Import exposed functions directly - compiler generates HTTP client:**

```typescript
// profile.cx (Client)

import { get_user, update_user_status, user_status } from "./user_service.sx"

component UserProfile(user_id: int) {
  user?: user = null
  loading = true
  error?: str = null
  
  async load_user() {
    loading.val = true
    error.val = null
    
    // Tuple destructuring for error handling
    data, err = await get_user(user_id)
    if err {
      error.val = err
      loading.val = false
      return
    }
    
    user.val = data
    loading.val = false
  }
  
  async suspend_user() {
    updated, err = await update_user_status(user_id, user_status.suspended)
    if err {
      error.val = err
      return
    }
    
    user.val = updated
  }
  
  on_mount(() = {
    load_user()
  })
  
  <div>
    {loading && <p>Loading...</p>}
    
    {error && <p class="error">{error}</p>}
    
    {!loading && !error && user && (
      <div>
        <h1>{user.name}</h1>
        <p>{user.email}</p>
        <p>Age: {user.age}</p>
        <p>Status: {match user.status {
          user_status.pending = "⏳ Pending",
          user_status.active = "✅ Active",
          user_status.suspended = "🚫 Suspended",
          user_status.deleted = "❌ Deleted"
        }}</p>
        
        {user.status == user_status.active && (
          <button @click={suspend_user}>Suspend User</button>
        )}
      </div>
    )}
  </div>
}
```

### How It Works

**Server (`.sx`):**
- `expose` keyword marks functions as API endpoints
- Regular functions stay server-only
- Compiler generates HTTP endpoints: `/api/user_service/get_user`
- Enums serialize to backing values in JSON

**Client (`.cx`):**
- Import exposed functions like normal imports
- Import enums and types from server files
- Compiler generates typed HTTP fetch calls
- Same function signature on both sides
- Full type safety across the boundary
- Enums deserialize from backing values

**Benefits:**
✅ **Explicit** - `expose` shows API surface
✅ **Type-safe** - Full checking across boundary
✅ **No magic** - Direct function imports
✅ **Secure** - Non-exposed functions can't be called
✅ **Simple** - No separate API layer needed
✅ **Enums travel** - Share enums between server and client

---

## Components

```typescript
component Counter(initial: int = 0) {
  count = initial
  
  increment() {
    count.val += 1
  }
  
  <div>
    <p>Count: {count}</p>
    <button @click={increment}>+</button>
  </div>
}
```

### Event Handling
```typescript
component Form() {
  name = ""
  errors = []
  
  validate(): bool {
    errors.val = []
    
    if name.count() < 2 {
      errors.add("Name too short")
    }
    
    return errors.empty()
  }
  
  submit(e) {
    e.prevent_default()
    
    if !validate() {
      return
    }
    
    save_user({ name })
  }
  
  <form @submit={submit}>
    <input 
      value={name}
      @input={(e) = name.val = e.target.value}
    />
    <button>Submit</button>
    {errors.map((e) = <p class="error">{e}</p>)}
  </form>
}
```

### Conditional Rendering
```typescript
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

component UserList() {
  users: list: user = []
  loading = true
  error?: str = null
  
  async load_users() {
    loading.val = true
    error.val = null
    
    data, err = await list_users(20)
    if err {
      error.val = err
      loading.val = false
      return
    }
    
    users.val = data
    loading.val = false
  }
  
  on_mount(() = {
    load_users()
  })
  
  <div>
    {loading && <p>Loading users...</p>}
    
    {error && <p class="error">{error}</p>}
    
    {!loading && !error && (
      <ul>
        {users.map((u) = (
          <li key={u.id}>
            {u.name} - {u.email} - {match u.status {
              user_status.pending = "Pending",
              user_status.active = "Active",
              user_status.suspended = "Suspended",
              user_status.deleted = "Deleted"
            }}
          </li>
        ))}
      </ul>
    )}
  </div>
}
```

---

## Modules

```typescript
// Export types and functions (available within project)
export get_user(id: int): result: user, error { }
export max_users = 1000
export default UserService

// Export enums
export enum user_status {
  pending: "pending",
  active: "active"
}

// Expose (API endpoint - callable from client)
expose get_user(id: int): result: user, error { }

// Import
import { get_user, create_user } from "./user_service.sx"
import * as UserService from "./user_service.sx"
import type { user } from "./types.sx"
import { user_status } from "./user_service.sx"
```

---

## Async/Await

```typescript
async fetch_user(id: int): result: user, error {
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
enum, err, export, expose, false, float, for, if, 
import, in, int, list, map, match, null, return, 
set, str, true, try, type, unknown, while
```

---

## Complete Example

```typescript
// user_service.sx (Server)

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

validate_email(email: str): result: bool, error {
  if email.empty() {
    return err("Email required")
  }
  
  if !email.includes("@") {
    return err("Invalid email format")
  }
  
  return true
}

expose get_user(id: int): result: user, error {
  user = Database.query("SELECT * FROM users WHERE id = ?", [id]).first()
  
  if user == null {
    return err("User not found")
  }
  
  return user
}

expose create_user(request: create_user_request): result: user, error {
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

expose list_users(limit: int = 10, offset: int = 0): result: list: user, error {
  return Database.query(
    "SELECT * FROM users LIMIT {limit} OFFSET {offset}"
  ).all()
}

expose update_user_status(id: int, status: user_status): result: user, error {
  existing = get_user(id)
  
  return Database.update("users", id, {
    status: status
  })
}

expose update_user_role(id: int, role: user_role): result: user, error {
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
// external_api.sx (Server)

expose fetch_external_user(api_id: str): result: user, error {
  response = await http.get("https://api.example.com/users/{api_id}")
  data = json.parse(response.body)
  
  if !data.is_map() {
    return err("Invalid response format")
  }
  
  if !data.has("name") || !data.get("name").is_str() {
    return err("Missing or invalid name")
  }
  
  age_value = data.get("age")
  age = age_value.is_int() ? age_value : int(age_value)
  
  // Parse status from API
  status_str = str(data.get("status"))
  status = user_status.from(status_str) ?: user_status.pending
  
  return create_user({
    name: str(data.get("name")),
    email: str(data.get("email")) ?: "",
    age: age
  })
}
```

```typescript
// user_profile.cx (Client)

import { 
  get_user, 
  update_user_status, 
  update_user_role,
  user_status,
  user_role
} from "./user_service.sx"

component UserProfile(user_id: int) {
  user?: user = null
  loading = true
  error?: str = null
  editing = false
  
  async load_user() {
    loading.val = true
    error.val = null
    
    data, err = await get_user(user_id)
    if err {
      error.val = err
      loading.val = false
      return
    }
    
    user.val = data
    loading.val = false
  }
  
  async change_status(new_status: user_status) {
    updated, err = await update_user_status(user_id, new_status)
    if err {
      error.val = err
      return
    }
    
    user.val = updated
  }
  
  async change_role(new_role: user_role) {
    updated, err = await update_user_role(user_id, new_role)
    if err {
      error.val = err
      return
    }
    
    user.val = updated
  }
  
  on_mount(() = {
    load_user()
  })
  
  <div>
    {loading && <p>Loading...</p>}
    
    {error && <p class="error">{error}</p>}
    
    {!loading && !error && user && (
      <div>
        <h1>{user.name}</h1>
        <p>{user.email}</p>
        <p>Age: {user.age}</p>
        
        <div>
          <strong>Status:</strong>
          {match user.status {
            user_status.pending = <span class="badge pending">⏳ Pending</span>,
            user_status.active = <span class="badge active">✅ Active</span>,
            user_status.suspended = <span class="badge suspended">🚫 Suspended</span>,
            user_status.deleted = <span class="badge deleted">❌ Deleted</span>
          }}
        </div>
        
        <div>
          <strong>Role:</strong>
          {match user.role {
            user_role.admin = <span class="badge admin">👑 Admin</span>,
            user_role.moderator = <span class="badge mod">🛡️ Moderator</span>,
            user_role.user = <span class="badge user">👤 User</span>,
            user_role.guest = <span class="badge guest">🔓 Guest</span>
          }}
        </div>
        
        <div class="actions">
          {user.status == user_status.active && (
            <button @click={() = change_status(user_status.suspended)}>
              Suspend User
            </button>
          )}
          
          {user.status == user_status.suspended && (
            <button @click={() = change_status(user_status.active)}>
              Activate User
            </button>
          )}
          
          <select @change={(e) = change_role(user_role.from(e.target.value))}>
            {user_role.values().map((role) = (
              <option 
                value={role.value} 
                selected={user.role == role}
              >
                {role.value}
              </option>
            ))}
          </select>
        </div>
      </div>
    )}
  </div>
}
```

---

## Project Structure

```
project/
├── src/
│   ├── server/
│   │   ├── user_service.sx
│   │   ├── auth_service.sx
│   │   ├── types.sx
│   │   └── database.sx
│   ├── client/
│   │   ├── components/
│   │   │   ├── UserCard.cx
│   │   │   └── LoginForm.cx
│   │   └── utils/
│   │       └── api.cx
│   └── shared/
│       └── enums.sx
├── tests/
│   ├── user_service_test.sx
│   └── components_test.cx
└── rivo.config.sx
```

---

## Best Practices

### Use Guard Clauses
Check error conditions early and return. Happy path stays at the main level.

### Tuple Destructuring for Error Handling
Use `data, err = function()` pattern for explicit error handling in client code.

### Automatic Error Propagation
In `result`-returning functions, errors propagate automatically - no manual checking needed.

### Use Strict Equality
Rivo only has `==` and it's always strict (checks type and value). No need to remember `===`.

### Type Check Unknown Data
When working with JSON or dynamic data, always use `.is_*()` methods before accessing.

### Cast Types Explicitly
Use `int()`, `str()`, `float()`, `bool()` for explicit type conversion. No implicit coercion.

### Use Backed Enums
Define enums with explicit backing values for database storage and API serialization.

### Expose Only What's Needed
Use `expose` sparingly - only mark functions that should be callable from the client.

### Share Enums Between Server and Client
Import enums in client code to ensure type safety across the boundary.

### Prefer Match for Multi-Branch
When you have more than two conditions, use `match` instead of nested ternaries.

### Choose ?? vs ?: Carefully
Use `??` when you only want to replace null/undefined. Use `?:` when you want to replace any falsy value (including 0, false, "").

### Use Spaceship for Sorting
Three-way comparison operator makes sorting cleaner and more efficient.

### Use Unified Collection API
Use `.count()`, `.has()`, `.add()`, etc. consistently across all collection types.

### Leverage Type Inference
Let the compiler infer collection types from literals when possible.

### Embrace Locality
Keep related logic together in cohesive units rather than scattering across files.

### Compose Naturally
Build systems from small, modular pieces that combine cleanly.

### Be Explicit
Make mutation and error handling visible and intentional.

### Stay Pragmatic
Choose simplicity over architectural purity when appropriate.

---

## Roadmap

- Component lifecycle hooks (on_mount, on_unmount, effects)
- Decorators/attributes
- Generics and constraints
- Testing framework
- Package manager
- Standard library
- Build tooling
- WebAssembly target

---

**Language:** Rivo  
**Tagline:** Full-stack in flow  
**Version:** 0.1  
**Status:** Draft  
**Last Updated:** 2026-01-22

"Like a river, code should flow naturally between its banks."
