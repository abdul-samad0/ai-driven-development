---
name: django-channels-auth
description: Implement WebSocket authentication for Django Channels. Covers authentication strategy selection (JWT query string, session cookie, ticket-based), custom middleware patterns, token validation, connection rejection, and security best practices. Use when adding authentication to WebSocket endpoints, choosing an auth strategy, building auth middleware, or securing WebSocket connections in any Django project.
---

# WebSocket Authentication

## Choosing an Authentication Strategy

WebSocket connections cannot set custom HTTP headers from browsers, so standard `Authorization: Bearer <token>` doesn't work. Choose from these proven strategies:

| Strategy | How Token is Sent | Best For | Trade-offs |
|---|---|---|---|
| **JWT via query string** | `ws://host/ws/path/?token=<jwt>` | SPAs with JWT auth, mobile apps | Token visible in server logs and browser history |
| **Session cookie** | Automatic via browser cookies | Traditional Django apps with session auth | Requires same-origin or CORS config |
| **Ticket-based** | Short-lived ticket from HTTP endpoint | High-security applications | Extra HTTP round-trip; most secure |
| **First-message auth** | Token sent as first WebSocket message | When query string is not viable | Connection accepted before auth; must handle unauthenticated window |

### Decision guide

```
Is the project an SPA using JWT (SimpleJWT / Auth0 / Firebase)?
├─ YES → JWT via query string
│
└─ NO → Is it a traditional Django app with session login?
         ├─ YES → Session cookie (AuthMiddlewareStack handles it)
         │
         └─ NO → Is security critical (banking, healthcare)?
                  ├─ YES → Ticket-based auth
                  └─ NO → JWT via query string (simplest to implement)
```

## Strategy 1: JWT via Query String

The most common pattern for Django/DRF projects using `djangorestframework-simplejwt`.

### How it works

1. Client includes JWT access token in the WebSocket URL: `ws://host/ws/endpoint/?token=<jwt>`
2. Custom middleware extracts the token from `scope["query_string"]`
3. Middleware validates the token using SimpleJWT's `AccessToken`
4. Middleware loads the user from the database with `@database_sync_to_async`
5. Middleware sets `scope["user"]` for downstream consumers
6. On failure, middleware sends a `websocket.close` frame with an error code

### Custom middleware structure

```python
from channels.middleware import BaseMiddleware
from channels.db import database_sync_to_async
from rest_framework_simplejwt.tokens import AccessToken
```

**Middleware `__call__` flow:**
1. Decode `scope["query_string"]` and extract the token parameter
2. If no token → close with code `4000`
3. Validate token → if invalid → close with code `4001`
4. Load user from DB → if not found → close with code `4001`
5. Set `scope["user"] = user`
6. Call `super().__call__(scope, receive, send)`

### Error codes

| Code | Meaning |
|------|---------|
| `4000` | No token provided |
| `4001` | Invalid token (expired, malformed, or user not found) |
| `4002+` | Available for custom error types |

WebSocket close codes 4000-4999 are reserved for application use per RFC 6455.

### Security mitigations for query string tokens

- **Use short-lived access tokens** — 5-15 minute expiry
- **Use HTTPS/WSS only** — prevents token interception in transit
- **Don't log query strings** — configure your web server and reverse proxy to strip or mask `?token=` from access logs
- **Validate on every connect** — tokens are checked once at connection time, not per message

## Strategy 2: Session Cookie

Works automatically with Django's session framework. No custom middleware needed.

### How it works

1. User logs in via standard Django auth (form login, OAuth, etc.)
2. Browser stores the session cookie
3. On WebSocket connect, the browser sends the cookie automatically
4. `AuthMiddlewareStack` reads the session and populates `scope["user"]`

### Setup

```python
from channels.auth import AuthMiddlewareStack

"websocket": AuthMiddlewareStack(
    URLRouter([...])
)
```

No custom middleware required. `AuthMiddlewareStack` handles session lookup.

### When to use
- Traditional Django apps with `django.contrib.sessions`
- Server-rendered templates (not SPAs)
- Same-origin WebSocket connections

### Limitations
- Cross-origin WebSocket connections need CORS configuration
- Won't work with token-based SPAs that don't use cookies

## Strategy 3: Ticket-Based Auth

Most secure approach. Uses a short-lived, single-use ticket obtained via an authenticated HTTP endpoint.

### How it works

1. Client calls an HTTP endpoint with their normal auth (JWT, session): `POST /api/ws-ticket/`
2. Server generates a random ticket, stores it in cache (Redis) with short TTL (30-60 seconds), and returns it
3. Client connects to WebSocket with the ticket: `ws://host/ws/endpoint/?ticket=<ticket>`
4. Middleware validates the ticket against the cache, loads the user, and deletes the ticket (single-use)
5. On success, sets `scope["user"]`

### Advantages
- Token is single-use — cannot be replayed
- Short TTL — reduces window of exposure
- Not stored in browser history (if generated per-connect)

### When to use
- Financial applications
- Healthcare/compliance environments
- Any system where query string token exposure is unacceptable

## Strategy 4: First-Message Auth

Token is sent as the first message after the WebSocket connection is established.

### How it works

1. Client connects without auth — connection is accepted
2. Client sends first message: `{"type": "authenticate", "token": "<jwt>"}`
3. Consumer validates the token in `receive`
4. If valid → set `self.scope["user"]`, begin normal operation
5. If invalid → close the connection

### Trade-offs
- **Pro**: Token not visible in URL, logs, or browser history
- **Con**: Connection is accepted before authentication — must handle the unauthenticated window
- **Con**: More complex consumer logic (state machine: unauthenticated → authenticated)
- **Con**: Must set a timeout — disconnect if no auth message received within N seconds

## Connection Rejection Pattern

Reject connections in middleware using the raw ASGI `send` callable:

```python
await send({"type": "websocket.close", "code": 4001})
return  # Stop processing — don't call super().__call__()
```

**Do not** use `self.close()` in middleware — the middleware does not have that method.

## Database Queries in Auth Middleware

All ORM queries must use `@database_sync_to_async`:

```python
@database_sync_to_async
def get_user_from_token(token_string):
    access_token = AccessToken(token_string)
    return User.objects.get(id=access_token.payload.get("user_id"))
```

Without this, Django raises `SynchronousOnlyOperation` in the async ASGI context.

## Security Best Practices

1. **Never log raw tokens** — log error types and user IDs, not credentials
2. **Always use WSS (TLS)** in production — prevents token interception
3. **Validate on connect, not per message** — WebSocket auth happens once at handshake
4. **Set appropriate token lifetimes** — shorter is better for WebSocket tokens
5. **Catch all validation exceptions** — never let token parsing errors crash the middleware
6. **Use application-level close codes (4000-4999)** — per RFC 6455
7. **Rate-limit connection attempts** — prevent brute-force token guessing

## Related Skills

- [Routing & middleware stack](../django-channels-routing/SKILL.md) — where middleware is configured
- [Consumers](../django-channels-consumers/SKILL.md) — how `scope["user"]` is used
- [Testing](../django-channels-testing/SKILL.md) — how to test auth in WebSocket tests
