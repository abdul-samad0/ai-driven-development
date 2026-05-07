---
name: django-channels-routing
description: Configure ASGI application and WebSocket URL routing for Django Channels. Covers ProtocolTypeRouter, URLRouter, middleware stacking, URL conventions, deployment considerations, and common pitfalls. Use when setting up ASGI, adding WebSocket endpoints, configuring middleware, or debugging routing issues in any Django project.
---

# ASGI Routing & WebSocket URLs

## ASGI Application Architecture

Django Channels replaces Django's WSGI with ASGI, enabling both HTTP and WebSocket on the same server.

```
Client Request
  ├─ HTTP  → django_asgi_app (standard Django views)
  └─ WebSocket → Middleware Stack → URLRouter → Consumer
```

### Critical: Initialization Order

`get_asgi_application()` **must** be called before importing any Django models or consumers. This initializes the Django ORM and app registry.

```python
import os
from django.core.asgi import get_asgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.production")

# Step 1: Initialize Django FIRST
django_asgi_app = get_asgi_application()

# Step 2: Now safe to import models and consumers
from django.urls import path
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack
```

Violating this order causes `django.core.exceptions.AppRegistryNotReady`.

## ProtocolTypeRouter

Separates protocols at the top level:

| Protocol | Handler | Purpose |
|---|---|---|
| `"http"` | `django_asgi_app` | Standard Django request/response cycle |
| `"websocket"` | Middleware → URLRouter | Real-time WebSocket connections |

```python
application = ProtocolTypeRouter({
    "http": django_asgi_app,
    "websocket": AuthMiddleware(URLRouter([...])),
})
```

## Middleware Stack

Middleware wraps from outermost to innermost. Each layer processes the connection before passing it down.

### Common stack patterns

**JWT auth (custom) + Django session auth:**
```
JWTAuthMiddleware → AuthMiddlewareStack → URLRouter → Consumer
```

**Django session auth only:**
```
AuthMiddlewareStack → URLRouter → Consumer
```

**No auth (public WebSocket):**
```
URLRouter → Consumer
```

### Middleware ordering rules

1. **Authentication middleware is outermost** — must run before anything that reads `scope["user"]`
2. **Custom middleware wraps built-in** — your JWT/API key middleware wraps `AuthMiddlewareStack`
3. **URLRouter is innermost** — routes the authenticated connection to the correct consumer

See [auth skill](../django-channels-auth/SKILL.md) for authentication middleware patterns.

## URL Pattern Conventions

### Best practices

1. **Prefix with `ws/`** — clearly separates WebSocket URLs from HTTP routes
2. **Use kebab-case** — `ws/live-prices/` not `ws/live_prices/` or `ws/livePrices/`
3. **Trailing slash** — consistent with Django's `APPEND_SLASH` behavior
4. **Register with `.as_asgi()`** — converts the consumer class into an ASGI application
5. **Keep routing centralized** — define all WebSocket routes in `asgi.py` (or a single `routing.py` if the list grows large)

### Import convention

Import **modules** rather than individual classes to avoid circular imports and keep the namespace clear:

```python
from myapp.notifications import consumers as notification_consumers
from myapp.analytics import consumers as analytics_consumers

URLRouter([
    path("ws/notifications/", notification_consumers.AlertConsumer.as_asgi()),
    path("ws/live-data/", analytics_consumers.StreamConsumer.as_asgi()),
])
```

### URL parameters

Use `path` converters for dynamic URL segments:

```python
path("ws/chat/<str:room_name>/", consumers.ChatConsumer.as_asgi()),
```

Access in the consumer via `self.scope["url_route"]["kwargs"]["room_name"]`.

## Settings

```python
# settings.py
ASGI_APPLICATION = "config.asgi.application"
```

The `CHANNEL_LAYERS` setting is also required if using `group_send`. See [redis skill](../django-channels-redis/SKILL.md).

## Deployment Considerations

### ASGI servers

| Server | Production-Ready | HTTP + WS | Notes |
|---|---|---|---|
| **Daphne** | Yes | Yes | Django Channels' reference server |
| **Uvicorn** | Yes | Yes | Fast, works with `channels` via ASGI |
| **Hypercorn** | Yes | Yes | HTTP/2 + WebSocket support |

### Scaling

- **Single process**: One Daphne/Uvicorn process handles both HTTP and WebSocket
- **Multi-process**: Run multiple workers behind a load balancer with sticky sessions (or use Redis channel layer for cross-process messaging)
- **Separate services**: Split HTTP (Gunicorn) and WebSocket (Daphne) behind a reverse proxy that routes by path prefix (`/ws/` → Daphne, everything else → Gunicorn)

## Common Pitfalls

1. **Importing before `get_asgi_application()`** — `AppRegistryNotReady`
2. **Missing `.as_asgi()`** — consumer class won't work as ASGI callable
3. **Wrong middleware order** — auth middleware must wrap the URLRouter, not the other way around
4. **Mixing WSGI and ASGI** — Django Channels requires an ASGI server; Gunicorn alone won't serve WebSockets
5. **No sticky sessions in load balancer** — WebSocket connections are stateful; the load balancer must route all frames of a connection to the same backend
