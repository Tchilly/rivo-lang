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
export get_user(id: int): result: user, error = 
  Database.query("SELECT * FROM users WHERE id = ?", [id])

// profile.cx - Client only
import { get_user } from "./user_service.sx"  // Compile error
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
  created_at: datetime
}

enum status {
  pending,
  active,
  suspended
}
```

---

## Type Checking and Conversion

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

### Type Conversion Methods
```typescript
// Convert between types
value.to_str()              // Convert to string
value.to_int()              // Convert to int (can fail)
value.to_float()            // Convert to float (can fail)
value.to_bool()             // Convert to boolean
value.to_list()             // Convert to list
value.to_set()              // Convert to set

// Examples
42.to_str()                 // "42"
"42".to_int()               // 42
3.14.to_int()               // 3 (truncates)
true.to_int()               // 1
[1, 2, 2].to_set()          // set([1, 2])
```

### Comparison with Conversion
```typescript
// Strict equality requires same type
3 == "3"                    // false (different types)
3 == "3".to_int()           // true
3.to_str() == "3"           // true
```

### Working with JSON and Unknown Types
```typescript
// Parse API response (returns unknown type)
async fetch_user(id: int): result: user, error {
  response = await http.get("/users/{id}")
  data = json.parse(response.body)
  
  // Type check before using
  if !data.is_map() {
    return err("Invalid response format")
  }
  
  // Check and extract fields
  if !data.has("id") || !data.get("id").is_int() {
    return err("Missing or invalid id")
  }
  
  if !data.has("name") || !data.get("name").is_str() {
    return err("Missing or invalid name")
  }
  
  // Email is optional - check if present
  email = null
  if data.has("email") {
    email_value = data.get("email")
    if !email_value.is_null() && !email_value.is_str() {
      return err("Invalid email type")
    }
    email = email_value.is_str() ? email_value : null
  }
  
  // Age might come as string or int - handle both
  age_value = data.get("age")
  age = match {
    age_value.is_int() = age_value,
    age_value.is_str() = age_value.to_int()?,
    _ = return err("Invalid age type")
  }
  
  return {
    id: data.get("id"),
    name: data.get("name"),
    email: email,
    age: age
  }
}

// Filter mixed types
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
numbers.sum()               // 7 (1+2+3+1+2)
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
tags.to_list()              // Convert to list

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

match scores.get("alice") {
  Some(score) = print("Score: {score}"),
  None = print("Not found")
}

// Or use default operators
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
?           // Error propagation (results)

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

// Error propagation
user = get_user(id)?
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

### Operator Examples
```typescript
// Arithmetic with precedence
result = 2 + 3 * 4        // 14 (not 20)
result = (2 + 3) * 4      // 20

// Strict equality
5 == 5          // true
5 == "5"        // false (different types)
0 == false      // false (different types)
true == 1       // false (different types)

// Logical short-circuit
user.active && process(user)

// Null coalescing vs Elvis - IMPORTANT DIFFERENCE
name = user?.name ?? "Guest"      // Only if null/undefined
port = config.port ?: 8080        // If any falsy (null, 0, false, "")

// Elvis replaces ALL falsy values
display = user.online ?: "Offline"  // false becomes "Offline"
retries = attempts ?: 3             // 0 becomes 3

// Null coalescing keeps non-null values
count = getValue() ?? 0   // Only replaces null/undefined
online = user.active ?? true  // false stays false, only null becomes true

// Optional chaining
city = user?.address?.city ?: "Unknown"

// Spaceship for sorting
users.sort((a, b) = a.name <=> b.name)
items.sort((a, b) = b.priority <=> a.priority)

// Error propagation
validated = get_user(id)?
  .validate()?
  .update()

// Range in match
grade = match score {
  90..100 = "A",
  80..89 = "B",
  _ = "F"
}
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

// Result matching
match get_user(id) {
  ok(user) = process(user),
  err(e) = log_error(e)
}

// In components
<div>
  {match status {
    "loading" = <Spinner />,
    "error" = <Error />,
    "success" = <Content />,
    _ = null
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

### Result Type
```typescript
// One-liner
get_first_user(): result: user, error = 
  Database.query("SELECT * FROM users").first()

// Multi-line with explicit return
get_user(id: int): result: user, error {
  user = Database.query("SELECT * FROM users WHERE id = ?", [id])
    .first()
  
  if user == null {
    return err("Not found")
  }
  
  return user
}
```

### Error Propagation
```typescript
process_payment(user_id: int, amount: float): result: payment, error {
  user = get_user(user_id)?      // Return early on error
  balance = check_balance(user)?
  
  if balance < amount {
    return err("Insufficient funds")
  }
  
  return charge(user, amount)
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

component Dashboard(status: str) {
  <div>
    {match status {
      "loading" = <Spinner />,
      "error" = <ErrorMessage />,
      "success" = <Content />,
      _ = <div>Unknown status</div>
    }}
  </div>
}
```

---

## Modules

```typescript
// Export
export get_user(id: int): result: user, error { }
export max_users = 1000
export default UserService

// Import
import { get_user, create_user } from "./user_service.sx"
import * as UserService from "./user_service.sx"
import type { user } from "./types.sx"
```

---

## Async/Await

```typescript
async fetch_user(id: int): result: user, error {
  response = await http.get("/users/{id}")
  
  if response.status != 200 {
    return err("Failed")
  }
  
  return json.decode(response.body)
}

// Parallel execution
[user, posts, friends] = await [
  fetch_user(id),
  fetch_posts(id),
  fetch_friends(id)
]
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
async, await, break, catch, component, continue,
enum, err, export, false, for, if, import, in,
match, null, ok, return, true, try, type, unknown, while
```

---

## Complete Example

```typescript
// user_service.sx (Server)

type user {
  id: int,
  name: str,
  email: str,
  created_at: datetime
}

export get_user(id: int): result: user, error {
  user = Database.query("SELECT * FROM users WHERE id = ?", [id])
    .first()
  
  if user == null {
    return err("User not found")
  }
  
  return user
}

export create_user(name: str, email: str): result: user, error {
  if email.empty() {
    return err("Email required")
  }
  
  if !email.includes("@") {
    return err("Invalid email")
  }
  
  user = Database.insert("users", {
    name,
    email,
    created_at: now()
  })?
  
  return user
}

export get_users_sorted_by_name(): list: user {
  users = Database.query("SELECT * FROM users").all()
  return users.sort((a, b) = a.name <=> b.name)
}

export get_total_score(user_id: int): int {
  scores = Database.query("SELECT score FROM games WHERE user_id = ?", [user_id])
    .all()
  return scores.sum()
}

// Parse API response with type checking
export async fetch_external_user(api_id: str): result: user, error {
  response = await http.get("https://api.example.com/users/{api_id}")
  
  if response.status != 200 {
    return err("API request failed")
  }
  
  data = json.parse(response.body)
  
  // Type check and validate
  if !data.is_map() {
    return err("Invalid response format")
  }
  
  if !data.has("name") || !data.get("name").is_str() {
    return err("Missing or invalid name")
  }
  
  // Age might be string or int - handle both
  age_value = data.get("age")
  age = match {
    age_value.is_int() = age_value,
    age_value.is_str() = age_value.to_int()?,
    _ = return err("Invalid age")
  }
  
  return create_user(data.get("name"), age)
}
```

```typescript
// user_profile.cx (Client)

import { api } from "./api.cx"

component UserProfile(user_id: int) {
  user?: object = null
  loading = true
  error?: str = null
  
  async load_user() {
    loading.val = true
    
    match await api.get_user(user_id) {
      ok(data) = {
        user.val = data
        loading.val = false
      },
      err(e) = {
        error.val = e
        loading.val = false
      }
    }
  }
  
  on_mount(() = {
    load_user()
  })
  
  <div>
    {match {
      loading = <p>Loading...</p>,
      error != null = <p class="error">{error}</p>,
      _ = (
        <div>
          <h1>{user.name ?: "No name"}</h1>
          <p>{user.email ?: "No email"}</p>
          <p>Joined: {user.created_at.format("MMM DD, YYYY")}</p>
        </div>
      )
    }}
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
│   │   └── database.sx
│   ├── client/
│   │   ├── components/
│   │   │   ├── UserCard.cx
│   │   │   └── LoginForm.cx
│   │   └── utils/
│   │       └── api.cx
│   └── shared/
│       └── types.sx
├── tests/
│   ├── user_service_test.sx
│   └── components_test.cx
└── rivo.config.sx
```

---

## Best Practices

### Use Guard Clauses
Check error conditions early and return. Happy path stays at the main level.

### Explicit Returns in Multi-Line Functions
Use explicit `return` statements in multi-line functions for clarity.

### Use Strict Equality
Rivo only has `==` and it's always strict (checks type and value). No need to remember `===`.

### Type Check Unknown Data
When working with JSON or dynamic data, always use `.is_*()` methods before accessing.

### Convert Types Explicitly
Use `.to_*()` methods for explicit type conversion. No implicit coercion.

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
