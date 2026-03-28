# Spinner Language Specification

> A simple language for creating web applications inspired by the simplicity of Sinatra — and then made even simpler. No fluff, no cruft, just your app.

---

## Table of Contents

1. [Overview](#overview)
2. [Running Spinner](#running-spinner)
3. [File-Level Directives](#file-level-directives)
4. [Routes](#routes)
5. [Return Values](#return-values)
6. [Strings](#strings)
7. [The fn{} Block](#the-fn-block)
8. [Built-in Variables](#built-in-variables)
9. [Control Flow](#control-flow)
10. [Functions](#functions)
11. [Indexing](#indexing)
12. [Restructuring](#restructuring)
13. [With Blocks](#with-blocks)
14. [HTTP Calls](#http-calls)
15. [Logging](#logging)
16. [Path Classes](#path-classes)
17. [Static File Serving](#static-file-serving)
18. [Error Handling](#error-handling)
19. [Example Application](#example-application)

---

## Overview

A Spinner application is made up of one or more files containing valid Spinner code. Each file can define routes, functions, and file-level directives. Spinner has a compile phase and a runtime phase — all symbols are resolved at compile time, so there are no forward reference or ordering issues at runtime.

Spinner listens on port **8000** by default.

The shortest complete Spinner application:

```
{ message: "Hello, Avatar" }
```

This responds to `GET` and `POST` requests at `/` with a `200 OK` and `Content-Type: application/json`.

---

## Running Spinner

Start a Spinner app by pointing the CLI at a file:

```sh
spinner app.spn
```

Override the default port at startup:

```sh
spinner app.spn --port 3000
```

Logs go to stdout by default and can be redirected using standard shell mechanisms:

```sh
spinner app.spn > app.log
spinner app.spn 2>&1 | tee app.log
```

---

## File-Level Directives

Directives configure behaviour for the routes that follow them in the file. A route is only affected by a directive if the directive appears **before** the route definition. This means routes defined before a directive are unaffected by it — no special syntax required.

`root` and `require` can appear multiple times in a file, each applying to the routes that follow until the next occurrence overrides it. This allows different sections of a file to have different roots and required headers.

```
post /users/register fn{ ... }
post /users/login fn{ ... }

require (Authorization)
root /api/v1

get /notes fn{ ... }
get /notes/[guid]:id fn{ ... }

root /api/v1/admin
require (Authorization, X-Admin-Key)

get /users fn{ ... }
delete /users/[guid]:id fn{ ... }
```

In the above example, `register` and `login` are unaffected by any `require` or `root`. The notes routes inherit `root /api/v1` and `require (Authorization)`. The admin routes inherit `root /api/v1/admin` and `require (Authorization, X-Admin-Key)`.

### root

Prefixes all subsequent routes in the file with the given path.

```
root /api/v1
```

- Optional. Defaults to `/`.
- Can appear multiple times — each occurrence applies to routes that follow it.

### require

Specifies headers that must be present on all incoming requests to subsequent routes in the file. Headers can optionally include a validation block for complex validation logic.

```
require (Token, X-App-Key)
```

Headers without a block are presence-checked only. Headers with a block are validated using custom logic:

```
require (
    Token {
        decoded = auth.decode(Token)
        reject unless decoded
    },
    X-App-Key {
        reject if X-App-Key != "expected-key"
    },
    Some-Header
)
```

Inside a validation block, the header's value is injected as a variable with the header's name. Validation passes by default — if execution reaches the end of the block without a `reject`, the header is accepted. A failing block automatically returns `400 Bad Request`.

The `reject` keyword supports four equivalent forms — use whichever reads most naturally:

```
reject if condition
reject if not condition
reject unless condition
reject if !condition
```

Routes that follow a `require` block with validation are guaranteed to have valid headers by the time they execute — no need to repeat validation logic inside `fn{}` blocks.

- Optional. No required headers by default.
- Multiple headers are comma-separated inside parentheses.
- Can appear multiple times — each occurrence applies to routes that follow it.

### include

Composes multiple Spinner files into one application.

```
include {
    helpers
    users
    notes
}
```

- Files do not need a `.spn` extension — any file containing valid Spinner code is accepted.
- Multiple files are listed inside the block.
- Include order does not matter. All symbols are resolved at compile time.
- An `include` block is affected by any `root` and `require` directives that appear before it.

### %detach

Opts a specific route out of the `root` and `require` directives that would otherwise apply to it, regardless of position in the file.

```
%detach
get /health "ok"

%detach
post /users/login fn{ ... }
```

- Applies only to the single route immediately following it.
- Useful for cases where repositioning the route is impractical or would hurt readability.
- The interaction between `%detach` and the ordering behaviour is still being defined.

---

## Routes

A route is made up of the following parts:

```
[method] [path] [status] [response headers] [return value | fn{ ... }]
```

Every part is optional except the return value.

### Defaults

| Part | Default |
|------|---------|
| method | `get, post` |
| path | `/` |
| status | `200 OK` |

### Methods

Supported HTTP methods: `get`, `post`, `put`, `delete`, `head`.

Multiple methods can be specified comma-separated:

```
get, post /some/route "ok"
```

### Paths

Static path:
```
get /users "ok"
```

Named parameter:
```
get /users/:id fn{ ... }
```

Named parameter with path class constraint:
```
get /users/[guid]:id fn{ ... }
```

Wildcard:
```
get /admin/* fn{ ... }
```

Catch-all (matches any unmatched route):
```
* 404 "Resource $path not found"
```

If no catch-all is defined, Spinner returns its own default:
```json
{ "error": "Resource not found" }
```
with a `404` status.

### Long paths

When a path is too long to fit on one line, break it at any `/` and continue on the next line. The `/` must be the last character on the line, followed by optional whitespace and a newline. Spinner joins the segments back into a single path.

```
get /this/route/
        is/too/long/
        to/fit/on/
        a/
        single/line 200 fn{ ... }
```

Is equivalent to:

```
get /this/route/is/too/long/to/fit/on/a/single/line 200 fn{ ... }
```

### Status

The HTTP status code to return:

```
get /secret 403
<html>
  <body><p>Forbidden</p></body>
</html>
```

### Response headers

Specify response headers for a route in a parenthesised block after the status code, with each header on its own line in `Header-Name: Value` format:

```
post /some/path 200
(
  First-Header: Value of first header
  Second-Header: Value of second header
)
fn {
    ...
}
```

Header values can be interpolated using `$` for simple values and `{}` for expressions:

```
post /some/path 200
(
  Content-Disposition: attachment; filename="$filename"
  X-Request-Id: {generateId()}
)
fn {
    ...
}
```

For brevity, the block can be written on a single line with comma-separated headers:

```
post /some/path 200 (X-Custom-Header: value, X-Another: value) fn { ... }
```

Headers set in the block can be added to or overridden dynamically inside `fn{}` using `res[headers]`:

```
post /some/path 200
(
  X-Custom-Header: static-value
)
fn {
    res[headers][X-Custom-Header] = "overridden value"
    res[headers][X-Extra-Header] = "dynamic value"
}
```

---

## Return Values

The return value is the only required part of a route. It drives the automatic `Content-Type` header.

| Return value | Content-Type |
|---|---|
| JSON object or array | `application/json` |
| HTML | `text/html` |
| Plain text | `text/plain` |
| `nil` | No `Content-Type`, `Content-Length: 0` |

### JSON

```
{ message: "Hello, Avatar" }
[1, 2, 3]
```

### HTML

```
get /welcome
<html>
  <head><title>Welcome</title></head>
  <body><p>Hello</p></body>
</html>
```

If the root element is `<html>`, Spinner automatically prepends `<!DOCTYPE html>`.

### Plain text

```
get /ping "pong"
```

### nil

```
get, post /azula-love-interest 204 nil
```

Produces an empty response body, no `Content-Type` header, and `Content-Length: 0`.

---

## Strings

Spinner has four string forms:

| Form | Multiline | Interpolating |
|---|---|---|
| `"..."` | No | Yes |
| `'...'` | No | No |
| `"""..."""` | Yes | Yes |
| `'''...'''` | No | No |

### Interpolation

Use `$` for simple cases — variables, array access, hash/dictionary access, object properties:

```
"Hello, $name"
"Item: $cart[0]"
"Token: $req[headers][Token]"
"Host: $config[host]"
```

Use `{}` for expressions and function calls:

```
"Result: {someFunction(name)}"
"Total: {items.length * price}"
```

---

## The fn{} Block

Routes can return a value inline or delegate to a `fn{}` block for more complex logic:

```
get /lake_laogai 403 fn{
    logger.info "Attempt to infiltrate the Dai Li"
    return
        <html>
            <body><p>Long Feng forbids you from entering here</p></body>
        </html>
}
```

`fn{}` blocks have access to all built-in variables (`req`, `res`, `logger`, and for wildcard routes, `path` and `segments`).

---

## Built-in Variables

Spinner automatically injects the following variables into every `fn{}` block.

### req — incoming request

| Variable | Description |
|---|---|
| `req[headers]` | Incoming request headers |
| `req[body]` | Request body |
| `req[query]` | Query string parameters |
| `req[params]` | Named route parameters (e.g. `:id`) |

Access named route parameters:
```
req[params][id]
```

Access a specific header:
```
req[headers][Authorization]
```

### res — outgoing response

| Variable | Description |
|---|---|
| `res[status]` | HTTP status code |
| `res[message]` | HTTP status message (e.g. `"OK"`, `"Not Found"`) |
| `res[body]` | Response body |
| `res[headers]` | Response headers |

Using `res` gives full manual control over the response and can be used instead of or alongside the inline return syntax.

### Wildcard route variables

Available inside `fn{}` blocks for wildcard routes (`/admin/*`):

| Variable | Description |
|---|---|
| `path` | The full wildcard match as a string (e.g. `"users/123"`) |
| `segments` | The path broken into an array (e.g. `["users", "123"]`) |

### logger

See [Logging](#logging).

---

## Control Flow

### Conditionals

```
if condition {
    ...
} else if condition {
    ...
} else {
    ...
}
```

`unless` is an alias for `if not`:

```
unless condition {
    ...
}

if not condition {
    ...
}
```

Parentheses around conditions are optional.

### Loops

```
while condition { ... }
until condition { ... }    
while not condition { ... }
```

`until` and `while not` are aliases for each other.

---

## Functions

Define named functions with the `fn` keyword:

```
fn myFunction {
    ...
}

fn myFunction(param1, param2) {
    ...
}
```

- Functions are **file-scoped** by default.
- Use the `public` keyword to make a function visible to other files via `include`:

```
public fn checkAuth {
    userId = auth.verify(req[headers][Authorization])
    unless userId {
        res[status] = 401
        res[message] = "Unauthorized"
        return
    }
    return userId
}
```

- Call functions directly by name. Parentheses are optional:

```
userId = checkAuth()
userId = checkAuth

note = findNote(req[params][id], userId)
note = findNote req[params][id], userId
```

- Parentheses are recommended when chaining index access on a return value, as dropping them can be ambiguous:

```
userId = auth.decode(Authorization)[id]
```

The above is clear — `auth.decode` is called with `Authorization`, and `[id]` indexes into the result. Without parentheses, `auth.decode Authorization [id]` could be read as passing both `Authorization` and `[id]` as arguments rather than chaining.

- Use `{}` only when interpolating a function's return value inside a string:

```
"Welcome, {getUsername userId}"
"Your token is {auth.generateToken user[id]}"
```

---

## Indexing

Square bracket indexing is used to access values in hashes, dictionaries, objects, and arrays.

### Unquoted keys

Keys that match `[a-zA-Z_][a-zA-Z0-9_-]*` do not need to be quoted:

```
req[headers][Authorization]
res[body][userId]
config[api-key]
```

### Quoted keys

Keys containing special characters must be quoted:

```
dict["something; special:should,quote"]
```

### Variable keys

Use `$` to specify a variable as the key:

```
hash[$some_variable]
req[headers][$headerName]
```

### Expression keys

Use `{}` to specify an expression that returns a string:

```
dictionary[{complex_expression()}]
dictionary[{getKey()}]
```

### Array indexing

Use a number to index into an array:

```
flowers[3]
segments[0]
cart[0]
```

---

## Restructuring

Spinner supports bidirectional restructuring — the same syntax works for both destructuring and structuring.

### Destructuring (right to left)

Unpack values from a source into local variables:

```
{ name, age } = req[body]
{ title, content } = req[body]
```

### Structuring (left to right)

Compose a structure from variables already in scope:

```
res[body][person] = { name, age, country, element }
```

---

## With Blocks

`with` blocks scope repeated assignments into a target object, eliminating repetitive prefix chains:

```
with res[body] {
    status = "ok"
    message = "Created"
}
```

`with` blocks can be nested. Inner blocks resolve relative to the outer target:

```
with res[body] {
    status = "ok"
    with person {
        name = "Zuko"
        age = 16
        with powers {
            element = "fire"
            temperament = "fierce"
        }
    }
}
```

This is equivalent to:
```
res[body][status] = "ok"
res[body][person][name] = "Zuko"
res[body][person][age] = 16
res[body][person][powers][element] = "fire"
res[body][person][powers][temperament] = "fierce"
```

---

## HTTP Calls

### Client style

Create a reusable client for repeated calls to the same base URL:

```
client = http(base: "https://api.example.com/v1", headers: { X-App-Key: "some key" })
```

Then make requests on the client. Parentheses are optional:

```
response = client.get /some/path
response = client.post /some/path
response = client.put /some/path
response = client.delete /some/path
response = client.head /some/path
```

### Direct style

For one-off requests:

```
response = http.get "https://api.example.com/users"
response = http.post "https://api.example.com/users"
```

### Response

The response is accessed via the same bracket pattern as `req`:

```
response[status]
response[headers]
response[body]
```

### Supported methods

`get`, `post`, `put`, `delete`, `head`.

---

## Logging

Spinner injects a `logger` object into every `fn{}` block. Logs go to stdout by default and can be redirected with standard shell mechanisms.

### Log levels

```
logger.debug "..."
logger.info "..."
logger.warn "..."
logger.critical "..."
logger.fatal "..."
```

Spinner also logs errors automatically when something goes wrong inside a `fn{}` block.

---

## Path Classes

Path classes constrain route parameter segments at the routing level. Non-matching requests are rejected before reaching the handler.

Syntax:
```
get /users/[class]:param fn{ ... }
get /users/[class:length]:param fn{ ... }
get /users/[class:min-max]:param fn{ ... }
```

Only one class per segment. `[guid]` classes do not require a length constraint as the format is already well-defined.

### Available classes

| Class | Description |
|---|---|
| `[alpha]` | Alphabets, upper and lower case |
| `[lalpha]` | Lower case alphabets |
| `[ualpha]` | Upper case alphabets |
| `[alnum]` | Alphanumeric |
| `[ualnum]` | Upper case alphabets and numbers |
| `[lalnum]` | Lower case alphabets and numbers |
| `[base64]` | URL-safe Base64 |
| `[hex]` | Hexadecimal |
| `[uhex]` | Upper case hexadecimal |
| `[lhex]` | Lower case hexadecimal |
| `[urlenc]` | URL-encoded string |
| `[uguid]` | GUID, upper case |
| `[lguid]` | GUID, lower case |
| `[guid]` | GUID, any case |

### Examples

```
get /notes/[guid]:id fn{ ... }
get /users/[lalpha:3-20]:username fn{ ... }
get /files/[hex:32]:hash fn{ ... }
get /tokens/[base64:64]:token fn{ ... }
```

---

## Static File Serving

Serve static files with the `static` directive:

```
static /route path/to/web/root
static /route path/to/web/root path/to/404.html
```

- If the path leads to a directory, Spinner serves `index.html` from that directory.
- The custom 404 page is optional. Spinner returns its own default 404 page if absent.
- Multiple `static` directives can appear in one file.

### Examples

```
static /assets path/to/public
static / path/to/dist
static /app path/to/app path/to/404.html
```

---

## Error Handling

### Runtime errors in fn{}

Spinner catches errors automatically, logs them, and returns an appropriate response:

- Client errors → `400 Bad Request`
- Server errors → `500 Internal Server Error`

### Manual error responses

Set `res[status]` and `res[message]` and return:

```
unless note {
    res[status] = 404
    res[message] = "Note not found"
    return
}
```

### Unmatched routes

Define a catch-all route to handle unmatched requests:

```
* 404 "Resource $path not found"
```

If absent, Spinner returns:
```json
{ "error": "Resource not found" }
```
with a `404` status.

---

## Example Application

A simple notes application with user authentication.

**main.spn**
```
include {
    helpers
    users
    notes
}

get /health "ok"

post /users/register fn{ ... }
post /users/login fn{ ... }

require (
    Authorization {
        decoded = auth.decode(Authorization)
        reject unless decoded
    }
)

root /api/v1

* 404 "Resource $path not found"
```

**helpers.spn**
```
public fn findNote(id, userId) {
    note = db.find("notes", { id, userId })
    unless note {
        res[status] = 404
        res[message] = "Note $id not found"
        return
    }
    return note
}
```

**users.spn**
```
%detach
post /users/register fn{
    { name, email, password } = req[body]
    unless name && email && password {
        res[status] = 400
        res[message] = "Name, email and password are required"
        return
    }
    user = db.save("users", { name, email, password })
    unless user {
        res[status] = 500
        res[message] = "Could not create user"
        return
    }
    res[status] = 201
    with res[body] {
        id = user[id]
        name = name
        email = email
    }
}

%detach
post /users/login fn{
    { email, password } = req[body]
    unless email && password {
        res[status] = 400
        res[message] = "Email and password are required"
        return
    }
    user = db.find("users", { email, password })
    unless user {
        res[status] = 401
        res[message] = "Invalid credentials"
        return
    }
    token = auth.generateToken(user[id])
    res[body] = { token }
}
```

**notes.spn**
```
get /notes fn{
    userId = auth.decode(req[headers][Authorization])[id]
    notes = db.find("notes", { userId })
    res[body] = notes
}

post /notes fn{
    userId = auth.decode(req[headers][Authorization])[id]
    { title, content } = req[body]
    unless title {
        res[status] = 400
        res[message] = "Title is required"
        return
    }
    note = db.save("notes", { userId, title, content })
    res[status] = 201
    res[body] = note
}

get /notes/[guid]:id fn{
    userId = auth.decode(req[headers][Authorization])[id]
    note = findNote(req[params][id], userId)
    res[body] = note
}

put /notes/[guid]:id fn{
    userId = auth.decode(req[headers][Authorization])[id]
    note = findNote(req[params][id], userId)
    { title, content } = req[body]
    unless title {
        res[status] = 400
        res[message] = "Title is required"
        return
    }
    updated = db.update("notes", req[params][id], { title, content })
    res[body] = updated
}

delete /notes/[guid]:id 204 nil fn{
    userId = auth.decode(req[headers][Authorization])[id]
    findNote(req[params][id], userId)
    db.delete("notes", req[params][id])
}
```
