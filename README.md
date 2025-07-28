# `json-joy` JIT Router

A high-performance HTTP router that constructs a hybrid Trie and Radix tree of the route structure and then JIT compiles the routes into a single optimized function for maximum performance.

## Key Features

- **Ultra-Fast Performance**: JIT compilation generates optimized JavaScript functions for route matching
- **Hybrid Tree Structure**: Combines Trie and Radix tree algorithms for efficient path matching
- **Flexible Route Patterns**: Supports exact matches, parameters, wildcards, and regex patterns
- **Parameter Extraction**: Automatically extracts and validates route parameters
- **TypeScript Support**: Full TypeScript definitions with generic data types
- **Zero Dependencies**: Core runtime has no external dependencies for maximum efficiency

## Installation

```bash
npm install @jsonjoy.com/jit-router
```

## Quick Start

```typescript
import { Router } from '@jsonjoy.com/jit-router';

// Create a new router instance
const router = new Router();

// Add routes with associated data
router.add('GET /users', { handler: 'listUsers' });
router.add('GET /users/{id}', { handler: 'getUser' });
router.add('POST /users/{id}/posts', { handler: 'createPost' });

// Compile routes into an optimized matcher function
const matcher = router.compile();

// Match incoming requests
const match = matcher('GET /users/123');
if (match) {
  console.log(match.data);    // { handler: 'getUser' }
  console.log(match.params);  // ['123']
}
```

## Destinations

Destinations are the core concept in jit-router. Each destination represents an endpoint with associated data and can be reached via one or more route patterns.

### Creating Destinations

```typescript
// Single route destination
router.add('GET /ping', 'PING_HANDLER');

// Multiple routes leading to same destination
router.add(['GET /ping', 'GET /pong'], 'PING_HANDLER');

// With custom data
router.add('GET /users/{id}', { 
  handler: getUserHandler,
  middleware: [authMiddleware],
  cache: true 
});
```

### Destination Properties

Each destination contains:
- **routes**: Array of route patterns that lead to this destination
- **data**: Associated data (handler, metadata, etc.)
- **match**: Match object returned when route is matched

## Route Patterns and Step Specifications

Routes consist of steps that define the matching pattern. There are three types of steps:

### 1. Exact Steps

Match literal text exactly:

```typescript
router.add('GET /api/users', 'USERS_API');
router.add('POST /login', 'LOGIN');
```

### 2. Parameter Steps (Until Steps)

Extract parameters from the URL path:

```typescript
// Basic parameter - matches until next '/'
router.add('GET /users/{id}', 'GET_USER');

// Custom delimiter - matches until specified character
router.add('GET /files/{name}.{ext}', 'GET_FILE');

// Wildcard - matches until end of string
router.add('GET /static/{path::\n}', 'STATIC_FILES');
```

#### Parameter Syntax:
- `{name}` - matches until next `/` (default delimiter)
- `{name::delimiter}` - matches until specified delimiter
- `{name::\n}` - matches until end of string (wildcard)

### 3. Regex Steps

Match parameters with regex patterns:

```typescript
// HTTP method matching
router.add('{method:(GET|POST)} /api/{endpoint}', 'API_HANDLER');

// Numeric IDs only
router.add('GET /users/{id:[0-9]+}', 'GET_USER_BY_ID');

// Optional path segments
router.add('GET /posts{slug:(/[^/]+)?}', 'GET_POST');

// Complex patterns
router.add('{method:[A-Z]+} /rpc/{procedure}', 'RPC_HANDLER');
```

#### Regex Syntax:
- `{name:pattern}` - matches with regex pattern
- `{name:pattern:delimiter}` - regex pattern with custom delimiter

## Parameter Extraction

Parameters are automatically extracted and provided in the match result:

```typescript
const router = new Router();
router.add('GET /users/{userId}/posts/{postId}', 'GET_USER_POST');
const matcher = router.compile();

const match = matcher('GET /users/123/posts/456');
// match.params = ['123', '456']

// Parameters correspond to the order they appear in the route
const [userId, postId] = match.params;
```

### Named Parameter Access

For better parameter handling, consider structuring your data to include parameter names:

```typescript
router.add('GET /users/{userId}/posts/{postId}', {
  handler: 'getUserPost',
  params: ['userId', 'postId']
});

const match = matcher('GET /users/123/posts/456');
if (match) {
  const params = {};
  match.data.params.forEach((name, index) => {
    params[name] = match.params[index];
  });
  // params = { userId: '123', postId: '456' }
}
```

## Code Generation and Compilation

The router uses JIT (Just-In-Time) compilation to generate highly optimized JavaScript functions:

### Compilation Process

1. **Tree Construction**: Routes are organized into a hybrid Trie/Radix tree structure
2. **Code Generation**: The tree is traversed to generate optimized JavaScript code
3. **Function Creation**: Code is compiled into a fast matcher function

```typescript
const router = new Router();
router.add('GET /users/{id}', 'USER_HANDLER');

// View the internal tree structure
console.log(router.toString());

// Compile to optimized function
const matcher = router.compile();

// View generated code (for debugging)
console.log(matcher.toString());
```

### Performance Characteristics

The JIT compilation produces functions with these optimizations:
- **No loops**: Uses conditional branches instead of iteration
- **String operations**: Optimized string slicing and comparison
- **Early returns**: Matches return immediately when found
- **Minimal allocations**: Reuses objects and avoids unnecessary memory allocation

## Advanced Usage Examples

### Complex Routing Patterns

```typescript
const router = new Router();

// API versioning
router.add('GET /api/v{version:[12]}/users', 'API_USERS');

// File serving with extensions
router.add('GET /assets/{file}.{ext:(js|css|png|jpg)}', 'STATIC_ASSET');

// Optional trailing slashes
router.add('GET /blog{trailing:/?}', 'BLOG_INDEX');

// Nested parameters with validation
router.add('POST /orgs/{org}/repos/{repo}/issues/{num:[0-9]+}', 'GITHUB_ISSUE');

// Wildcard with method matching
router.add('{method:(GET|HEAD)} /files/{path::\n}', 'FILE_HANDLER');

const matcher = router.compile();
```

### Custom Router Options

```typescript
// Custom default delimiter
const router = new Router({ 
  defaultUntil: '|'  // Use '|' instead of '/' as default delimiter
});

router.add('GET |users|{id}', 'USER_HANDLER');
```

### Type-Safe Usage with TypeScript

```typescript
interface RouteData {
  handler: string;
  middleware?: Function[];
  cache?: boolean;
}

const router = new Router<RouteData>();

router.add('GET /users/{id}', {
  handler: 'getUser',
  middleware: [authMiddleware],
  cache: true
});

const matcher = router.compile();
const match = matcher('GET /users/123');

if (match) {
  // TypeScript knows match.data is RouteData
  console.log(match.data.handler);     // 'getUser'
  console.log(match.data.cache);       // true
  console.log(match.params);           // ['123']
}
```

### Route Introspection

```typescript
const router = new Router();
router.add('GET /users/{id}', 'USER_HANDLER');
router.add('POST /users', 'CREATE_USER');

// Inspect destinations
console.log('Destinations:', router.destinations.length);
router.destinations.forEach((dest, i) => {
  console.log(`[${i}]:`, dest.routes.map(r => r.toText()));
});

// View routing tree structure
const tree = router.tree();
console.log(tree.toString('  '));
```

## API Reference

### Router Class

#### Constructor
```typescript
new Router<Data>(options?: RouterOptions)
```

**Options:**
- `defaultUntil?: string` - Default delimiter for parameter steps (default: '/')

#### Methods

##### `add(route: string | string[], data: Data): void`
Add a route or multiple routes to the same destination.

##### `addDestination(destination: Destination): void`
Add a pre-constructed destination.

##### `compile(): RouteMatcher<Data>`
Compile routes into an optimized matcher function.

##### `tree(): RoutingTreeNode`
Get the internal routing tree structure.

### RouteMatcher Function

```typescript
(route: string) => Match<Data> | undefined
```

Returns a `Match` object if the route matches, `undefined` otherwise.

### Match Object

```typescript
interface Match<Data> {
  data: Data;           // Associated route data
  params: string[];     // Extracted parameters
}
```

### Route Step Types

#### ExactStep
Matches literal text exactly.

#### UntilStep  
Matches parameters until a delimiter:
- `name: string` - Parameter name
- `until: string` - Delimiter character

#### RegexStep
Matches parameters with regex patterns:
- `name: string` - Parameter name  
- `regex: string` - Regex pattern
- `until: string` - Delimiter character

## Error Handling

The router handles various edge cases gracefully:

```typescript
const router = new Router();
router.add('GET /users/{id}', 'USER_HANDLER');
const matcher = router.compile();

// Non-matching routes return undefined
console.log(matcher('GET /posts/123'));     // undefined
console.log(matcher('POST /users/123'));    // undefined

// Empty or invalid routes return undefined  
console.log(matcher(''));                   // undefined
console.log(matcher('INVALID'));           // undefined
```


## Benchmarks

Comparing `json-joy` against the second fastest router `find-my-way` (used in Fastify):

```
npx ts-node src/__bench__/realistic.bench.ts
```

Results:

```
json-joy router x 1,799,920 ops/sec ±0.74% (99 runs sampled), 556 ns/op
find-my-way x 389,132 ops/sec ±4.60% (87 runs sampled), 2570 ns/op
json-joy router: GET /ping x 151,628,101 ops/sec ±1.83% (87 runs sampled), 7 ns/op
find-my-way: GET /ping x 14,820,512 ops/sec ±0.23% (100 runs sampled), 67 ns/op
json-joy router: GET /pong x 103,442,010 ops/sec ±0.49% (101 runs sampled), 10 ns/op
find-my-way: GET /pong x 12,396,065 ops/sec ±0.14% (95 runs sampled), 81 ns/op
json-joy router: POST /ping x 155,689,270 ops/sec ±0.26% (96 runs sampled), 6 ns/op
find-my-way: POST /ping x 14,964,742 ops/sec ±0.62% (100 runs sampled), 67 ns/op
json-joy router: POST /echo x 103,757,580 ops/sec ±0.24% (94 runs sampled), 10 ns/op
find-my-way: POST /echo x 19,484,996 ops/sec ±0.13% (100 runs sampled), 51 ns/op
json-joy router: GET /info x 86,407,568 ops/sec ±0.45% (101 runs sampled), 12 ns/op
find-my-way: GET /info x 19,090,564 ops/sec ±0.23% (99 runs sampled), 52 ns/op
json-joy router: GET /types x 82,040,568 ops/sec ±0.19% (93 runs sampled), 12 ns/op
find-my-way: GET /types x 18,258,167 ops/sec ±0.13% (94 runs sampled), 55 ns/op
json-joy router: PUT /events x 155,566,843 ops/sec ±0.30% (95 runs sampled), 6 ns/op
find-my-way: PUT /events x 17,947,965 ops/sec ±0.29% (95 runs sampled), 56 ns/op
json-joy router: GET /users x 67,873,498 ops/sec ±0.19% (97 runs sampled), 15 ns/op
find-my-way: GET /users x 18,427,182 ops/sec ±0.70% (97 runs sampled), 54 ns/op
json-joy router: GET /users/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx x 24,268,077 ops/sec ±0.38% (98 runs sampled), 41 ns/op
find-my-way: GET /users/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx x 5,830,219 ops/sec ±0.20% (101 runs sampled), 172 ns/op
json-joy router: GET /users/123 x 23,750,636 ops/sec ±0.92% (98 runs sampled), 42 ns/op
find-my-way: GET /users/123 x 8,883,381 ops/sec ±0.15% (98 runs sampled), 113 ns/op
json-joy router: DELETE /users/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx x 21,578,200 ops/sec ±0.98% (96 runs sampled), 46 ns/op
find-my-way: DELETE /users/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx x 6,081,393 ops/sec ±0.82% (96 runs sampled), 164 ns/op
json-joy router: POST /users/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx x 19,981,015 ops/sec ±0.16% (96 runs sampled), 50 ns/op
find-my-way: POST /users/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx x 6,094,531 ops/sec ±0.27% (96 runs sampled), 164 ns/op
json-joy router: GET /users/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/followers x 17,566,830 ops/sec ±0.92% (99 runs sampled), 57 ns/op
find-my-way: GET /users/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/followers x 4,584,840 ops/sec ±0.27% (97 runs sampled), 218 ns/op
json-joy router: GET /users/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/followers/yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy x 13,432,092 ops/sec ±0.26% (100 runs sampled), 74 ns/op
find-my-way: GET /users/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/followers/yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy x 2,959,002 ops/sec ±0.17% (100 runs sampled), 338 ns/op
json-joy router: POST /users/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/followers/yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy x 12,074,790 ops/sec ±1.05% (98 runs sampled), 83 ns/op
find-my-way: POST /users/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/followers/yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy x 3,211,678 ops/sec ±0.31% (96 runs sampled), 311 ns/op
json-joy router: DELETE /users/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/followers/yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy x 12,878,647 ops/sec ±0.30% (99 runs sampled), 78 ns/op
find-my-way: DELETE /users/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/followers/yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy x 3,214,022 ops/sec ±0.69% (95 runs sampled), 311 ns/op
json-joy router: GET /posts x 47,369,117 ops/sec ±1.64% (83 runs sampled), 21 ns/op
find-my-way: GET /posts x 11,424,540 ops/sec ±0.17% (101 runs sampled), 88 ns/op
json-joy router: GET /posts/search x 91,571,586 ops/sec ±1.22% (94 runs sampled), 11 ns/op
find-my-way: GET /posts/search x 6,931,399 ops/sec ±0.16% (100 runs sampled), 144 ns/op
json-joy router: POST /posts x 71,501,062 ops/sec ±0.91% (93 runs sampled), 14 ns/op
find-my-way: POST /posts x 13,755,641 ops/sec ±0.21% (98 runs sampled), 73 ns/op
json-joy router: GET /posts/jhasdf982lsd x 20,794,603 ops/sec ±0.79% (97 runs sampled), 48 ns/op
find-my-way: GET /posts/jhasdf982lsd x 6,220,203 ops/sec ±0.17% (100 runs sampled), 161 ns/op
json-joy router: POST /posts/jhasdf982lsd x 17,002,131 ops/sec ±0.24% (94 runs sampled), 59 ns/op
find-my-way: POST /posts/jhasdf982lsd x 6,783,108 ops/sec ±0.64% (98 runs sampled), 147 ns/op
json-joy router: DELETE /posts/jhasdf982lsd x 18,912,944 ops/sec ±0.38% (95 runs sampled), 53 ns/op
find-my-way: DELETE /posts/jhasdf982lsd x 8,353,962 ops/sec ±0.84% (99 runs sampled), 120 ns/op
json-joy router: GET /posts/jhasdf982lsd/tags x 16,857,334 ops/sec ±4.01% (92 runs sampled), 59 ns/op
find-my-way: GET /posts/jhasdf982lsd/tags x 5,169,375 ops/sec ±0.39% (98 runs sampled), 193 ns/op
json-joy router: GET /posts/jhasdf982lsd/tags/top x 16,012,780 ops/sec ±0.38% (99 runs sampled), 62 ns/op
find-my-way: GET /posts/jhasdf982lsd/tags/top x 4,062,952 ops/sec ±0.65% (97 runs sampled), 246 ns/op
json-joy router: GET /posts/jhasdf982lsd/tags/123 x 12,144,904 ops/sec ±0.21% (100 runs sampled), 82 ns/op
find-my-way: GET /posts/jhasdf982lsd/tags/123 x 4,036,060 ops/sec ±0.15% (98 runs sampled), 248 ns/op
json-joy router: DELETE /posts/jhasdf982lsd/tags/123 x 11,797,726 ops/sec ±0.50% (94 runs sampled), 85 ns/op
find-my-way: DELETE /posts/jhasdf982lsd/tags/123 x 5,269,916 ops/sec ±0.10% (100 runs sampled), 190 ns/op
json-joy router: POST /posts/jhasdf982lsd/tags x 15,483,784 ops/sec ±0.20% (100 runs sampled), 65 ns/op
find-my-way: POST /posts/jhasdf982lsd/tags x 5,586,607 ops/sec ±0.10% (100 runs sampled), 179 ns/op
json-joy router: GET /api/collections x 89,443,474 ops/sec ±1.14% (93 runs sampled), 11 ns/op
find-my-way: GET /api/collections x 11,365,669 ops/sec ±0.15% (102 runs sampled), 88 ns/op
json-joy router: POST /api/collections x 88,558,408 ops/sec ±1.15% (93 runs sampled), 11 ns/op
find-my-way: POST /api/collections x 11,427,491 ops/sec ±0.14% (98 runs sampled), 88 ns/op
json-joy router: GET /api/collections/123 x 15,327,034 ops/sec ±0.21% (101 runs sampled), 65 ns/op
find-my-way: GET /api/collections/123 x 6,654,562 ops/sec ±0.10% (102 runs sampled), 150 ns/op
json-joy router: PUT /api/collections/123 x 14,766,966 ops/sec ±0.23% (100 runs sampled), 68 ns/op
find-my-way: PUT /api/collections/123 x 7,446,186 ops/sec ±0.14% (99 runs sampled), 134 ns/op
json-joy router: POST /api/collections/123 x 13,397,211 ops/sec ±0.23% (100 runs sampled), 75 ns/op
find-my-way: POST /api/collections/123 x 5,801,550 ops/sec ±6.34% (92 runs sampled), 172 ns/op
json-joy router: DELETE /api/collections/123 x 16,470,990 ops/sec ±0.26% (99 runs sampled), 61 ns/op
find-my-way: DELETE /api/collections/123 x 6,618,101 ops/sec ±4.54% (90 runs sampled), 151 ns/op
json-joy router: GET /api/collections/123/documents x 10,819,176 ops/sec ±20.48% (90 runs sampled), 92 ns/op
find-my-way: GET /api/collections/123/documents x 5,083,007 ops/sec ±0.10% (100 runs sampled), 197 ns/op
json-joy router: POST /api/collections/123/documents x 11,471,312 ops/sec ±0.18% (101 runs sampled), 87 ns/op
find-my-way: POST /api/collections/123/documents x 5,024,672 ops/sec ±0.14% (97 runs sampled), 199 ns/op
json-joy router: GET /api/collections/123/documents/456 x 8,776,775 ops/sec ±0.25% (98 runs sampled), 114 ns/op
find-my-way: GET /api/collections/123/documents/456 x 3,892,995 ops/sec ±0.30% (100 runs sampled), 257 ns/op
json-joy router: PUT /api/collections/123/documents/456 x 9,946,515 ops/sec ±1.36% (97 runs sampled), 101 ns/op
find-my-way: PUT /api/collections/123/documents/456 x 4,347,762 ops/sec ±0.62% (98 runs sampled), 230 ns/op
json-joy router: POST /api/collections/123/documents/456 x 8,120,919 ops/sec ±0.44% (99 runs sampled), 123 ns/op
find-my-way: POST /api/collections/123/documents/456 x 3,882,741 ops/sec ±1.26% (92 runs sampled), 258 ns/op
json-joy router: DELETE /api/collections/123/documents/456 x 10,739,860 ops/sec ±0.89% (97 runs sampled), 93 ns/op
find-my-way: DELETE /api/collections/123/documents/456 x 4,483,659 ops/sec ±1.17% (98 runs sampled), 223 ns/op
json-joy router: GET /api/collections/123/documents/456/revisions x 8,553,359 ops/sec ±0.91% (98 runs sampled), 117 ns/op
find-my-way: GET /api/collections/123/documents/456/revisions x 3,379,187 ops/sec ±0.17% (101 runs sampled), 296 ns/op
json-joy router: GET /api/collections/123/documents/456/revisions/find x 8,092,442 ops/sec ±1.10% (97 runs sampled), 124 ns/op
find-my-way: GET /api/collections/123/documents/456/revisions/find x 2,829,474 ops/sec ±0.17% (100 runs sampled), 353 ns/op
json-joy router: POST /api/collections/123/documents/456/revisions x 7,750,406 ops/sec ±0.93% (99 runs sampled), 129 ns/op
find-my-way: POST /api/collections/123/documents/456/revisions x 3,412,482 ops/sec ±0.14% (98 runs sampled), 293 ns/op
json-joy router: GET /api/collections/123/documents/456/revisions/789 x 7,133,453 ops/sec ±0.51% (98 runs sampled), 140 ns/op
find-my-way: GET /api/collections/123/documents/456/revisions/789 x 2,851,795 ops/sec ±0.80% (95 runs sampled), 351 ns/op
json-joy router: DELETE /api/collections/123/documents/456/revisions/789 x 8,168,205 ops/sec ±0.24% (96 runs sampled), 122 ns/op
find-my-way: DELETE /api/collections/123/documents/456/revisions/789 x 3,282,963 ops/sec ±0.10% (98 runs sampled), 305 ns/op
json-joy router: GET /files/user123-movies/2019/01/01/1.mp4 x 13,990,918 ops/sec ±0.19% (100 runs sampled), 71 ns/op
find-my-way: GET /files/user123-movies/2019/01/01/1.mp4 x 5,132,706 ops/sec ±0.90% (101 runs sampled), 195 ns/op
json-joy router: PUT /files/user123-movies/2019/01/01/1.mp4 x 13,951,426 ops/sec ±0.23% (99 runs sampled), 72 ns/op
find-my-way: PUT /files/user123-movies/2019/01/01/1.mp4 x 5,184,318 ops/sec ±0.23% (99 runs sampled), 193 ns/op
json-joy router: GET /static/some/path/to/file.txt x 15,788,214 ops/sec ±1.06% (99 runs sampled), 63 ns/op
find-my-way: GET /static/some/path/to/file.txt x 8,104,177 ops/sec ±0.14% (100 runs sampled), 123 ns/op
Fastest is json-joy router: POST /ping,json-joy router: PUT /events,json-joy router: GET /ping
```
