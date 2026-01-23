# Rivo Language Specification v0.1

**Rivo: Full-stack in flow**

A modern, type-safe language that compiles to JavaScript/TypeScript and runs on Bun.

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

## Compilation Model

Rivo compiles to JavaScript/TypeScript and runs on **Bun**.

### File Extensions & Compile Targets

| Extension | Compiles To | Runs On | Purpose |
|-----------|-------------|---------|---------|
| `.sx` | TypeScript | Bun (server) | Server logic, APIs, database, filesystem |
| `.sc` | TypeScript + JSX | Bun (SSR) | Server-rendered components, hydration |
| `.cx` | JavaScript + JSX | Browser | Client-side interactivity, DOM access |

### Compilation Pipeline

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  .sx files  │────▶│              │────▶│  .ts files  │────▶ Bun (server)
└─────────────┘     │              │     └─────────────┘
                    │    Rivo      │
┌─────────────┐     │   Compiler   │     ┌─────────────┐
│  .sc files  │────▶│              │────▶│  .tsx files │────▶ Bun (SSR)
└─────────────┘     │              │     └─────────────┘
                    │              │
┌─────────────┐     │              │     ┌─────────────┐
│  .cx files  │────▶│              │────▶│  .js/.jsx   │────▶ Browser
└─────────────┘     └──────────────┘     └─────────────┘
```

### What the Compiler Does

1. **Type Checking** - Full static type analysis before compilation
2. **Syntax Transformation** - Rivo syntax → JavaScript/TypeScript
3. **Bundle Splitting** - Separates server (`.sx`, `.sc`) from client (`.cx`)
4. **API Generation** - Creates HTTP endpoints for `exposed` functions
5. **Client Generation** - Creates typed fetch calls for client → server communication
6. **Hydration Setup** - Wires `.sc` server renders to `.cx` client interactivity

### Compiler Rules

- `.sx` files **never** bundle to client (enforced)
- `.cx` files **cannot** import non-exposed `.sx` internals (enforced)
- `.cx` and `.sc` files **can** import `exposed` functions, types, and enums from `.sx`
- Types and enums are shared across all contexts (compile-time only)

### Running Rivo

```bash
# Install Rivo compiler
bun add -g rivo

# Compile project
rivo build

# Run with Bun
bun run dist/server.ts

# Development mode (watch + hot reload)
rivo dev
```

### Compiled Output Example

**Rivo source:**
```typescript
// math.sx
expose add(a: int, b: int): int = a + b
```

**Compiled TypeScript:**
```typescript
// dist/math.ts
export function add(a: number, b: number): number {
  return a + b;
}

// Auto-generated HTTP endpoint
router.post('/api/math/add', async (req) => {
  const { a, b } = await req.json();
  return Response.json(add(a, b));
});
```

**Auto-generated client:**
```typescript
// dist/client/math.ts
export async function add(a: number, b: number): Promise<number> {
  const res = await fetch('/api/math/add', {
    method: 'POST',
    body: JSON.stringify({ a, b })
  });
  return res.json();
}
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
name = "Jane"  // Compile error: cannot reassign immutable variable

// Mutation with .value
count = 0
count.value = 5      // Declare intent to mutate
count.value += 1     // Mutate

// Reading (no .value needed)
print(count)         // Outputs: 6
total = count + 5

// Compiler optimizes constants automatically
```

**Compiles to:**
```typescript
const name = "John";
const age = 25;

let count = 0;
count = 5;
count += 1;
```

**Note:** `.value` is a compile-time marker for mutation intent. It compiles to regular assignment but enables the compiler to track mutability.

---

## Type System

### Primitives

| Rivo Type | JavaScript Type | Notes |
|-----------|-----------------|-------|
| `int` | `number` | Compiler warns on float operations |
| `float` | `number` | Explicit float type |
| `str` | `string` | |
| `bool` | `boolean` | |
| `datetime` | `Date` | Full date + time |
| `date` | `Date` | Date only (time at 00:00) |
| `time` | `Date` | Time only (date at epoch) |
| `null` | `null` | Intentional absence |
| `undefined` | `undefined` | Uninitialized/missing |

```typescript
// Rivo
name: str = "John"
age: int = 25
price: float = 19.99
active: bool = true
created: datetime = now()

// Compiles to TypeScript
const name: string = "John";
const age: number = 25;
const price: number = 19.99;
const active: boolean = true;
const created: Date = new Date();
```

### Collections

| Rivo Type | JavaScript Type | Notes |
|-----------|-----------------|-------|
| `list<T>` | `Array<T>` | With extended methods |
| `map<K, V>` | `Map<K, V>` | With extended methods |
| `set<T>` | `Set<T>` | With extended methods |

```typescript
// Rivo
numbers = [1, 2, 3]                    // list<int>
scores = { "a": 1, "b": 2 }            // map<str, int>
tags = set(["rust", "go"])             // set<str>

// Compiles to
const numbers: number[] = [1, 2, 3];
const scores = new Map([["a", 1], ["b", 2]]);
const tags = new Set(["rust", "go"]);
```

### Tuples

```typescript
// Rivo
point = (10, 20)                       // tuple<int, int>
entry = ("alice", 100)                 // tuple<str, int>

// Destructuring
(x, y) = point
(name, score) = entry

// Compiles to
const point: [number, number] = [10, 20];
const entry: [string, number] = ["alice", 100];

const [x, y] = point;
const [name, score] = entry;
```

### Type Inference

```typescript
// Types inferred from literals
numbers = [1, 2, 3]                    // list<int>
scores = { "a": 1, "b": 2 }            // map<str, int>
point = (10, 20)                       // tuple<int, int>

// Explicit types when needed
users: list<user> = []
config: map<str, str> = {}
coords: tuple<float, float> = (0.0, 0.0)
```

### Special Types

```typescript
result<T, E>    // Success or error (Either monad)
option<T>       // Some or None (Maybe monad)
error           // Error type { message: str, code?: int }
unknown         // Dynamic type (for JSON, etc.)
```

**Compiles to:**
```typescript
// result<T, E> compiles to discriminated union
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

// option<T> compiles to
type Option<T> = T | null;

// error compiles to
type RivoError = { message: string; code?: number };
```

### Nullable

```typescript
// Rivo
name: str       // Cannot be null
email?: str     // Can be null or undefined

// Optional chaining
user?.address?.city

// Null coalescing (only null/undefined)
display = user?.name ?? "Anonymous"

// Elvis operator (any falsy value)
display = user.name ?: "Anonymous"

// Compiles to
const name: string;
const email: string | null | undefined;

user?.address?.city;
const display = user?.name ?? "Anonymous";
const display = user.name || "Anonymous";  // Elvis → ||
```

### Custom Types

```typescript
// Rivo
type user {
  id: int,
  name: str,
  email?: str,
  status: user_status,
  created_at: datetime
}

// Compiles to TypeScript interface
interface User {
  id: number;
  name: string;
  email?: string;
  status: UserStatus;
  created_at: Date;
}
```

### Enums (Backed)

```typescript
// Rivo - String-backed
enum user_status {
  pending: "pending",
  active: "active",
  suspended: "suspended"
}

// Rivo - Int-backed
enum priority {
  low: 1,
  medium: 2,
  high: 3
}

// Usage
status = user_status.active
priority = priority.high

// Access backing value
status.value           // "active"
priority.value         // 3

// From backing value
user_status.from("active")     // user_status.active
priority.from(3)               // priority.high

// Compiles to TypeScript
enum UserStatus {
  Pending = "pending",
  Active = "active",
  Suspended = "suspended"
}

enum Priority {
  Low = 1,
  Medium = 2,
  High = 3
}
```

### Enum Methods

```typescript
// Rivo
role.values()              // [role.admin, role.moderator, role.user]
role.backing_values()      // ["admin", "moderator", "user"]
role.from("admin")         // option<role> → Some(role.admin) or None
role.from_result("admin")  // result<role, error>

// Compiles to runtime helpers
Object.values(Role);
Object.values(Role);
parseRole("admin");  // Generated helper function
```

---

## Type Checking and Casting

### Type Checking Methods

```typescript
// Rivo
value.is_int()
value.is_float()
value.is_str()
value.is_bool()
value.is_null()
value.is_undefined()
value.is_list()
value.is_map()
value.is_set()

// Compiles to
typeof value === 'number' && Number.isInteger(value)
typeof value === 'number' && !Number.isInteger(value)
typeof value === 'string'
typeof value === 'boolean'
value === null
value === undefined
Array.isArray(value)
value instanceof Map
value instanceof Set
```

### Type Casting

```typescript
// Rivo
age = int("42")             // 42
price = float("19.99")      // 19.99
text = str(123)             // "123"
flag = bool("true")         // true

// Compiles to (with runtime validation)
const age = parseInt("42", 10);
const price = parseFloat("19.99");
const text = String(123);
const flag = "true" === "true";  // Strict boolean parsing
```

### Schema Validation

```typescript
// Rivo
type api_response {
  id: int,
  name: str,
  email?: str
}

data = json.parse(input).validate(api_response)

// Compiles to (using generated validator)
const data = validateApiResponse(JSON.parse(input));

// Generated validator
function validateApiResponse(data: unknown): ApiResponse {
  if (typeof data !== 'object' || data === null) {
    throw new ValidationError('Expected object');
  }
  if (typeof data.id !== 'number') {
    throw new ValidationError('Expected id to be number');
  }
  // ... etc
  return data as ApiResponse;
}
```

---

## Functions

**No keywords.** Context-aware declarations.

### Function Syntax

```typescript
// One-liner, implicit return
add(a: int, b: int) = a + b

// One-liner, explicit type
add(a: int, b: int): int = a + b

// Multi-line, explicit return required
add(a: int, b: int): int {
  result = a + b
  return result
}

// Compiles to
const add = (a: number, b: number) => a + b;

const add = (a: number, b: number): number => a + b;

function add(a: number, b: number): number {
  const result = a + b;
  return result;
}
```

### Function Declaration vs Variable Assignment

```typescript
// Function declaration (has parameters)
double(n: int) = n * 2

// Variable assignment (no parameters)
result = calculate(5)

// Compiler distinguishes by presence of parameter list
```

### Optional Parameters

```typescript
// Rivo
greet(name: str, greeting: str = "Hello") = "{greeting}, {name}!"

fetch_users(limit: int, offset?: int): list<user> {
  // offset can be null
}

// Compiles to
const greet = (name: string, greeting: string = "Hello") => `${greeting}, ${name}!`;

function fetchUsers(limit: number, offset?: number): User[] {
  // ...
}
```

### Lambdas (Anonymous Functions)

```typescript
// Rivo
users.map((u) = u.name)
users.filter((u) = u.active)
items.reduce(0, (acc, i) = acc + i)

// Multi-line lambda
items.map((item) = {
  if item.invalid {
    return null
  }
  return item.value
})

// Compiles to
users.map((u) => u.name);
users.filter((u) => u.active);
items.reduce((acc, i) => acc + i, 0);

items.map((item) => {
  if (item.invalid) {
    return null;
  }
  return item.value;
});
```

### Async Functions

```typescript
// Rivo
async fetch_user(id: int): result<user, error> {
  response = await http.get("/users/{id}")
  return json.parse(response.body).validate(user)
}

// Compiles to
async function fetchUser(id: number): Promise<Result<User, RivoError>> {
  const response = await http.get(`/users/${id}`);
  return validateUser(JSON.parse(response.body));
}
```

---

## Collections

### List

```typescript
// Rivo
numbers = [1, 2, 3]

numbers.count()             // 3
numbers.is_empty()          // false
numbers.has(2)              // true
numbers.get(0)              // 1
numbers.first()             // 1
numbers.last()              // 3

numbers.add(4)              // Mutates: [1, 2, 3, 4]
numbers.set(0, 10)          // Mutates: [10, 2, 3, 4]
numbers.delete(0)           // Mutates: [2, 3, 4]

numbers.map((n) = n * 2)
numbers.filter((n) = n > 2)
numbers.reduce(0, (acc, n) = acc + n)
numbers.sort((a, b) = a <=> b)

// Compiles to (with runtime extensions)
const numbers = [1, 2, 3];

numbers.length;
numbers.length === 0;
numbers.includes(2);
numbers[0];
numbers[0];
numbers[numbers.length - 1];

numbers.push(4);
numbers[0] = 10;
numbers.splice(0, 1);

numbers.map((n) => n * 2);
numbers.filter((n) => n > 2);
numbers.reduce((acc, n) => acc + n, 0);
numbers.sort((a, b) => a - b);
```

### Map

```typescript
// Rivo
scores = { "alice": 100, "bob": 85 }

scores.count()              // 2
scores.has("alice")         // true
scores.get("alice")         // option<int> → 100
scores.keys()               // ["alice", "bob"]
scores.values()             // [100, 85]
scores.entries()            // [("alice", 100), ("bob", 85)]

scores.set("charlie", 90)
scores.delete("bob")

// Compiles to
const scores = new Map([["alice", 100], ["bob", 85]]);

scores.size;
scores.has("alice");
scores.get("alice");
[...scores.keys()];
[...scores.values()];
[...scores.entries()];

scores.set("charlie", 90);
scores.delete("bob");
```

### Set

```typescript
// Rivo
tags = set(["rust", "go", "python"])

tags.count()                // 3
tags.has("rust")            // true
tags.add("ruby")
tags.delete("go")

tags.union(other_set)
tags.intersect(other_set)
tags.difference(other_set)

// Compiles to
const tags = new Set(["rust", "go", "python"]);

tags.size;
tags.has("rust");
tags.add("ruby");
tags.delete("go");

new Set([...tags, ...otherSet]);
new Set([...tags].filter(x => otherSet.has(x)));
new Set([...tags].filter(x => !otherSet.has(x)));
```

### Unified Collection API

| Method | List | Map | Set | Compiles To |
|--------|------|-----|-----|-------------|
| `.count()` | ✓ | ✓ | ✓ | `.length` / `.size` |
| `.is_empty()` | ✓ | ✓ | ✓ | `.length === 0` / `.size === 0` |
| `.has(x)` | ✓ | ✓ | ✓ | `.includes()` / `.has()` |
| `.get(x)` | ✓ | ✓ | - | `[x]` / `.get()` |
| `.set(x, v)` | ✓ | ✓ | - | `[x] = v` / `.set()` |
| `.add(x)` | ✓ | - | ✓ | `.push()` / `.add()` |
| `.delete(x)` | ✓ | ✓ | ✓ | `.splice()` / `.delete()` |
| `.map(fn)` | ✓ | ✓ | ✓ | `.map()` / iteration |
| `.filter(fn)` | ✓ | ✓ | ✓ | `.filter()` / iteration |
| `.each(fn)` | ✓ | ✓ | ✓ | `.forEach()` |

---

## Operators

### Arithmetic

| Rivo | JavaScript | Notes |
|------|------------|-------|
| `a + b` | `a + b` | |
| `a - b` | `a - b` | |
| `a * b` | `a * b` | |
| `a / b` | `a / b` | |
| `a % b` | `a % b` | |
| `a ** b` | `a ** b` | Exponentiation |

### Comparison

| Rivo | JavaScript | Notes |
|------|------------|-------|
| `a == b` | `a === b` | Always strict equality |
| `a != b` | `a !== b` | Always strict inequality |
| `a < b` | `a < b` | |
| `a > b` | `a > b` | |
| `a <= b` | `a <= b` | |
| `a >= b` | `a >= b` | |
| `a <=> b` | `compare(a, b)` | Spaceship operator |

```typescript
// Rivo spaceship
users.sort((a, b) = a.age <=> b.age)

// Compiles to
users.sort((a, b) => {
  if (a.age < b.age) return -1;
  if (a.age > b.age) return 1;
  return 0;
});
```

### Logical

| Rivo | JavaScript |
|------|------------|
| `a && b` | `a && b` |
| `a \|\| b` | `a \|\| b` |
| `!a` | `!a` |

### Assignment

| Rivo | JavaScript | Notes |
|------|------------|-------|
| `name = value` | `const name = value` | Immutable |
| `name.value = x` | `name = x` | Mutable |
| `x.value += 1` | `x += 1` | Compound |

### Null Handling

| Rivo | JavaScript | Notes |
|------|------------|-------|
| `a?.b` | `a?.b` | Optional chaining |
| `a ?? b` | `a ?? b` | Null coalescing |
| `a ?: b` | `a \|\| b` | Elvis (falsy coalescing) |

### Range

```typescript
// Rivo
for i in 0..10 { }    // Inclusive [0, 10]
for i in 0..<10 { }   // Exclusive [0, 10)

// Compiles to
for (let i = 0; i <= 10; i++) { }
for (let i = 0; i < 10; i++) { }
```

### String Interpolation

```typescript
// Rivo
message = "Hello, {name}!"
text = "You have {count * 2} items"

// Compiles to
const message = `Hello, ${name}!`;
const text = `You have ${count * 2} items`;
```

### Precedence (Highest to Lowest)

```
1.  ()  []  .  ?.         Grouping, indexing, member access
2.  !  -  (unary)         Unary operators
3.  **                    Exponentiation
4.  *  /  %               Multiplication, division, modulo
5.  +  -                  Addition, subtraction
6.  ..  ..<               Range
7.  <  <=  >  >=  <=>     Comparison
8.  ==  !=                Equality
9.  &&                    Logical AND
10. ||                    Logical OR
11. ??                    Null coalescing
12. ?:                    Elvis
13. ? :                   Ternary
14. =                     Assignment
```

---

## Control Flow

### If (Guard Clauses Only)

Rivo uses `if` for guard clauses. There is **no `else`** keyword.

```typescript
// Rivo
if !user.verified {
  return err("Not verified")
}

if user.banned {
  return err("User banned")
}

// Happy path continues...
process_user(user)

// Compiles to
if (!user.verified) {
  return { ok: false, error: "Not verified" };
}

if (user.banned) {
  return { ok: false, error: "User banned" };
}

processUser(user);
```

### Why No `else`?

Use **ternary** for either/or values, **match** for multi-branch logic:

```typescript
// Ternary for values
status = active ? "online" : "offline"

// Match for multi-branch
grade = match {
  score >= 90 = "A",
  score >= 80 = "B",
  score >= 70 = "C",
  _ = "F"
}
```

### Ternary

```typescript
// Rivo
status = active ? "online" : "offline"

// Compiles to
const status = active ? "online" : "offline";
```

### Match

```typescript
// Rivo - Value matching
action = match user.role {
  "admin" = admin_panel(),
  "moderator" = mod_panel(),
  _ = user_panel()
}

// Rivo - Condition matching
grade = match {
  score >= 90 = "A",
  score >= 80 = "B",
  score >= 70 = "C",
  _ = "F"
}

// Compiles to
const action = (() => {
  switch (user.role) {
    case "admin": return adminPanel();
    case "moderator": return modPanel();
    default: return userPanel();
  }
})();

const grade = (() => {
  if (score >= 90) return "A";
  if (score >= 80) return "B";
  if (score >= 70) return "C";
  return "F";
})();
```

### Loops

```typescript
// Rivo - For-in
for user in users {
  print(user.name)
}

// Rivo - Range (inclusive)
for i in 0..10 {
  print(i)
}

// Rivo - Range (exclusive)
for i in 0..<10 {
  print(i)
}

// Rivo - While
while condition {
  process()
}

// Break and continue
for item in items {
  if item.skip {
    continue
  }
  if item.done {
    break
  }
  process(item)
}

// Compiles to
for (const user of users) {
  console.log(user.name);
}

for (let i = 0; i <= 10; i++) {
  console.log(i);
}

for (let i = 0; i < 10; i++) {
  console.log(i);
}

while (condition) {
  process();
}

for (const item of items) {
  if (item.skip) continue;
  if (item.done) break;
  process(item);
}
```

---

## Error Handling

### Result Type

```typescript
// Rivo
get_user(id: int): result<user, error> {
  user = Database.query("SELECT * FROM users WHERE id = ?", [id]).first()
  
  if user == null {
    return err("User not found")
  }
  
  return user
}

// Compiles to
function getUser(id: number): Result<User, RivoError> {
  const user = Database.query("SELECT * FROM users WHERE id = ?", [id]).first();
  
  if (user === null) {
    return { ok: false, error: { message: "User not found" } };
  }
  
  return { ok: true, value: user };
}
```

### Automatic Error Propagation

```typescript
// Rivo - errors propagate automatically in result functions
update_user(id: int, name: str): result<user, error> {
  existing = get_user(id)      // Error propagates if get_user fails
  return Database.update("users", id, { name: name })
}

// Compiles to
function updateUser(id: number, name: string): Result<User, RivoError> {
  const existingResult = getUser(id);
  if (!existingResult.ok) return existingResult;  // Propagate
  const existing = existingResult.value;
  
  return Database.update("users", id, { name });
}
```

### Tuple Destructuring for Explicit Handling

```typescript
// Rivo
data, err = await get_user(user_id)
if err {
  error.value = err
  return
}
user.value = data

// Compiles to
const result = await getUser(userId);
if (!result.ok) {
  error = result.error;
  return;
}
const data = result.value;
user = data;
```

---

## Modules & Exports

### Export (Internal)

```typescript
// Rivo - available within project
export get_user(id: int): result<user, error> { }
export max_users = 1000
export type user { }
export enum status { }

// Compiles to
export function getUser(id: number): Result<User, RivoError> { }
export const maxUsers = 1000;
export interface User { }
export enum Status { }
```

### Expose (API Endpoint)

```typescript
// Rivo - creates HTTP endpoint + client
expose get_user(id: int): result<user, error> {
  return Database.query("SELECT * FROM users WHERE id = ?", [id]).first()
}

// Compiles to:

// 1. Server function (dist/server/user_service.ts)
export function getUser(id: number): Result<User, RivoError> {
  return Database.query("SELECT * FROM users WHERE id = ?", [id]).first();
}

// 2. HTTP endpoint (auto-registered)
router.post('/api/user_service/get_user', async (req) => {
  const { id } = await req.json();
  const result = getUser(id);
  return Response.json(result);
});

// 3. Client function (dist/client/user_service.ts)
export async function getUser(id: number): Promise<Result<User, RivoError>> {
  const res = await fetch('/api/user_service/get_user', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ id })
  });
  return res.json();
}
```

### Import

```typescript
// Rivo
import { get_user, create_user } from "./user_service.sx"
import { user_status } from "./types.sx"
import type { user } from "./types.sx"

// Compiles to
import { getUser, createUser } from "./user_service";
import { UserStatus } from "./types";
import type { User } from "./types";
```

---

## Async/Await

```typescript
// Rivo
async fetch_user(id: int): result<user, error> {
  response = await http.get("/users/{id}")
  return json.parse(response.body).validate(user)
}

// Parallel execution
[user, posts, friends] = await [
  fetch_user(id),
  fetch_posts(id),
  fetch_friends(id)
]

// Compiles to
async function fetchUser(id: number): Promise<Result<User, RivoError>> {
  const response = await http.get(`/users/${id}`);
  return validateUser(JSON.parse(response.body));
}

const [user, posts, friends] = await Promise.all([
  fetchUser(id),
  fetchPosts(id),
  fetchFriends(id)
]);
```

---

## Components (JSX)

Components in `.sc` and `.cx` files compile to JSX.

### Basic Component

```typescript
// Rivo (.cx)
component Counter(initial: int = 0) {
  count = initial
  
  increment() {
    count.value += 1
  }
  
  <div>
    <p>Count: {count}</p>
    <button @click={increment}>+</button>
  </div>
}

// Compiles to JSX (framework-agnostic)
function Counter({ initial = 0 }: { initial?: number }) {
  let count = initial;  // Reactivity depends on framework
  
  const increment = () => {
    count += 1;
  };
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={increment}>+</button>
    </div>
  );
}
```

### Event Handling

| Rivo | Compiles To |
|------|-------------|
| `@click={fn}` | `onClick={fn}` |
| `@input={fn}` | `onInput={fn}` |
| `@submit={fn}` | `onSubmit={fn}` |
| `@change={fn}` | `onChange={fn}` |
| `@keydown={fn}` | `onKeyDown={fn}` |

### Server vs Client Components

```typescript
// user_card.sc - Renders on server, compiles to server-side JSX
component UserCard(user: user) {
  <div class="user-card">
    <h2>{user.name}</h2>
    <p>{user.email}</p>
  </div>
}

// counter.cx - Renders on client, compiles to client-side JSX
component Counter(initial: int = 0) {
  count = initial
  
  <div>
    <span>{count}</span>
    <button @click={() = count.value += 1}>+</button>
  </div>
}
```

---

## String Interpolation

```typescript
// Rivo
name = "John"
message = "Hello, {name}!"
complex = "Result: {calculate(x) + 10}"

// Multi-line
query = "
  SELECT * FROM users
  WHERE active = true
"

// Raw strings (no escaping)
regex = r"\d+\.\d+"
path = r"C:\Users\name"

// Compiles to
const name = "John";
const message = `Hello, ${name}!`;
const complex = `Result: ${calculate(x) + 10}`;

const query = `
  SELECT * FROM users
  WHERE active = true
`;

const regex = "\\d+\\.\\d+";
const path = "C:\\Users\\name";
```

---

## Comments

```typescript
// Single line comment

/* Multi-line
   comment */

/**
 * Documentation comment
 * @param id - User ID
 * @returns The user object
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

## Standard Library (Runtime)

The Rivo runtime provides extended methods that compile to JavaScript equivalents:

### Collections

```typescript
// These methods are available on all collections
.count()      // Length/size
.is_empty()   // Check if empty
.has(x)       // Contains check
.clear()      // Remove all
.map(fn)      // Transform
.filter(fn)   // Filter
.each(fn)     // Iterate
```

### Strings

```typescript
str.count()           // .length
str.is_empty()        // .length === 0
str.includes(s)       // .includes()
str.starts_with(s)    // .startsWith()
str.ends_with(s)      // .endsWith()
str.upper()           // .toUpperCase()
str.lower()           // .toLowerCase()
str.trim()            // .trim()
str.split(sep)        // .split()
str.replace(a, b)     // .replace()
```

### JSON

```typescript
json.parse(str)                    // JSON.parse()
json.encode(obj)                   // JSON.stringify()
json.parse(str).validate(type)     // Parse + validate against type
```

### HTTP (Server)

```typescript
http.get(url)                      // fetch(url)
http.post(url, body)               // fetch(url, { method: 'POST', body })
http.put(url, body)                // fetch(url, { method: 'PUT', body })
http.delete(url)                   // fetch(url, { method: 'DELETE' })
```

### Database (Server)

```typescript
Database.query(sql, params)        // Prepared statement
Database.insert(table, data)       // Insert record
Database.update(table, id, data)   // Update record
Database.delete(table, id)         // Delete record
```

### DateTime

```typescript
now()                              // new Date()
date.format(pattern)               // Format date string
date.add_days(n)                   // Date arithmetic
date.add_hours(n)
date.diff(other)                   // Difference between dates
```

---

## Project Structure

```
project/
├── src/
│   ├── server/              # .sx files (server logic)
│   │   ├── user_service.sx
│   │   ├── auth_service.sx
│   │   └── database.sx
│   ├── components/
│   │   ├── server/          # .sc files (SSR components)
│   │   │   ├── UserCard.sc
│   │   │   └── Layout.sc
│   │   └── client/          # .cx files (client components)
│   │       ├── Counter.cx
│   │       └── Form.cx
│   └── shared/
│       ├── types.sx         # Shared types
│       └── enums.sx         # Shared enums
├── dist/                    # Compiled output
│   ├── server/              # Compiled .ts files
│   ├── client/              # Compiled .js files
│   └── api/                 # Generated API routes
├── tests/
│   └── *.test.sx
├── public/                  # Static assets
├── rivo.config.sx           # Compiler config
└── package.json
```

---

## Compiler Configuration

```typescript
// rivo.config.sx
export default {
  // Source directory
  src: "./src",
  
  // Output directory
  dist: "./dist",
  
  // Target runtime
  runtime: "bun",
  
  // API prefix for exposed functions
  api_prefix: "/api",
  
  // Enable strict mode
  strict: true,
  
  // Minify client output
  minify: true,
  
  // Source maps
  sourcemaps: true
}
```

---

## CLI Commands

```bash
# Initialize new project
rivo init my-project

# Compile project
rivo build

# Development mode (watch + hot reload)
rivo dev

# Type check without compiling
rivo check

# Format code
rivo fmt

# Run tests
rivo test

# Run compiled server
bun run dist/server/index.ts
```

---

## Best Practices

### Use Guard Clauses
Check error conditions early and return. No `else` needed.

### Automatic Error Propagation
In `result`-returning functions, errors propagate automatically.

### Use Schema Validation
Validate JSON with `.validate(type)` for type-safe parsing.

### Expose Only What's Needed
Use `expose` sparingly - only for client-callable functions.

### Use Backed Enums
Define enums with backing values for serialization.

### Prefer Match for Multi-Branch
Use `match` instead of nested ternaries for complex logic.

### Choose ?? vs ?: Carefully
`??` for null/undefined only, `?:` for any falsy value.

### Use Type Inference
Let the compiler infer types from literals when possible.

---

## Roadmap

- [ ] Component lifecycle hooks (on_mount, on_unmount)
- [ ] Try/catch exception handling
- [ ] Decorators/attributes
- [ ] Generics and constraints
- [ ] Testing framework
- [ ] Package manager integration
- [ ] Standard library expansion
- [ ] LSP (Language Server Protocol)
- [ ] VS Code extension
- [ ] REPL
- [ ] WebAssembly target

---

**Language:** Rivo  
**Tagline:** Full-stack in flow  
**Version:** 0.1  
**Target:** Bun (JavaScript/TypeScript)  
**Status:** Draft  
**Last Updated:** 2026-01-23

"Like a river, code should flow naturally between its banks."
