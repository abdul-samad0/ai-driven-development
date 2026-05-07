---
name: django-channels-consumers
description: Design and implement WebSocket consumers for Django/DRF projects using Django Channels. Covers library selection, AsyncWebsocketConsumer lifecycle, transport pattern selection, consumer architecture, error handling, and performance. Use when building real-time features, choosing a WebSocket library, creating consumers, or making architectural decisions about WebSocket communication in Django.
---

# WebSocket Consumers for Django/DRF

## Library Selection

### Why Django Channels

| Library | Best For | Limitations |
|---|---|---|
| **Django Channels** | Full Django integration, group messaging, auth middleware, Redis channel layer | Heavier than raw ASGI |
| **Starlette/FastAPI WebSockets** | Standalone microservices, already using FastAPI | No Django ORM integration, separate deployment |
| **django-websocket** | Legacy projects on WSGI | No async support, deprecated patterns |
| **Socket.IO (python-socketio)** | Auto-reconnection, rooms, namespaces, browser fallbacks | Extra protocol layer, not Django-native |

**Choose Django Channels when:**
- Your project is Django/DRF-based
- You need Django auth, ORM, and middleware in WebSocket handlers
- You need group-based messaging (broadcast to rooms/topics)
- You want a single deployment serving both HTTP and WebSocket

**Required packages:**
- `channels[daphne]` — Core framework + ASGI server
- `channels-redis` — Redis channel layer backend (if using group_send)
- `django-redis` — Direct Redis access (if using pub/sub, streams, or connection tracking)

### Async vs Sync Consumers

**Always use `AsyncWebsocketConsumer`** unless you have a compelling reason not to.

| Consumer | Use When |
|---|---|
| `AsyncWebsocketConsumer` | Default choice. Non-blocking I/O, supports `asyncio`, scales better under concurrent connections |
| `WebsocketConsumer` (sync) | Only if all downstream code is synchronous and cannot be adapted. Ties up a thread per connection |
| `JsonWebsocketConsumer` | Convenience wrapper that auto-encodes/decodes JSON. Use if every message is JSON |

## Consumer Lifecycle

Every consumer follows this lifecycle. Each method has specific responsibilities:

### `__init__`
- Call `super().__init__(*args, **kwargs)`
- Initialize instance variables: group names, Redis connections, subscription lists
- Never perform I/O here (no DB queries, no Redis calls, no network)

### `connect`
1. Call `await self.accept()` first
2. Log the connection with `channel_name` and authenticated user
3. Join groups (`group_add`) or initialize Redis pub/sub
4. Send a connection confirmation to the client
5. Wrap in `try/except` — log errors with full context

### `receive`
1. Validate that `text_data` is present
2. Parse JSON safely (catch `json.JSONDecodeError`)
3. Validate required fields before processing
4. Route to action methods based on an `action` field
5. Send structured error responses for invalid input

### `disconnect`
1. Leave all groups (`group_discard`)
2. Cancel background tasks (`asyncio.create_task` cleanup)
3. Close external connections (Redis pubsub, etc.)
4. Clean up tracking data (Redis sets, counters)
5. Log disconnection with close code
6. Handle `CancelledError` from async tasks

### Event Handlers
- Named to match the `type` field in `group_send` (dots become underscores)
- Each handler receives an `event` dict and forwards `event["data"]` to the client
- Wrap in `try/except` — a failed handler should not crash the consumer

## Choosing a Transport Pattern

There are two fundamental approaches to delivering messages to consumers. Choose based on throughput, durability, and complexity needs.

### Channels Layer (`group_send`)
- **Throughput**: Low to moderate (< 100 msg/sec per group)
- **Durability**: None — messages are ephemeral
- **Complexity**: Low — no direct Redis code needed
- **Use when**: Notifications, alerts, activity feeds, chat, status updates
- **How it works**: `group_add` registers a consumer in a group → `group_send` fans out to all members → each consumer's event handler receives the message

### Direct Redis (pub/sub, streams, or lists)
- **Throughput**: High (1000+ msg/sec per channel)
- **Durability**: Depends on technique (see [redis skill](../django-channels-redis/SKILL.md))
- **Complexity**: Higher — you manage subscriptions, background tasks, and cleanup
- **Use when**: Live data feeds, market tickers, real-time metrics, high-frequency IoT data
- **How it works**: Publisher pushes to Redis → consumer runs a background `asyncio.create_task` that polls Redis and forwards messages to the WebSocket

See [redis skill](../django-channels-redis/SKILL.md) for the complete decision framework on which Redis technique to use.

## Consumer Architecture Principles

### Single Responsibility
One consumer per domain concern. Don't combine order notifications and live price feeds in the same consumer.

### Separation of Concerns
- **Consumer**: WebSocket lifecycle, group management, message routing
- **Service layer**: Business logic, serialization, data fetching
- **Broadcasting helpers**: Static methods or utility functions for sending from outside consumers

### ORM in Async Consumers
All Django ORM calls must use `@database_sync_to_async`:

```python
from channels.db import database_sync_to_async

@database_sync_to_async
def get_user_preferences(self):
    return self.scope["user"].preferences.filter(active=True).values()
```

Without this decorator, Django raises `SynchronousOnlyOperation`.

### Blocking Calls in Async Consumers
Any synchronous/blocking call (Redis, HTTP, file I/O) must use `run_in_executor`:

```python
result = await asyncio.get_event_loop().run_in_executor(
    None, self.redis_connection.get, "some_key"
)
```

## Error Handling Best Practices

1. **Wrap every lifecycle method** (`connect`, `disconnect`, event handlers) in `try/except`
2. **Log with context**: Include user, channel_name, group, and the operation that failed
3. **Never expose internals**: Send generic error messages to clients, log details server-side
4. **Catch specific exceptions first**, then catch `Exception` as a fallback
5. **Re-raise `CancelledError`**: This is not an error — it's how `asyncio` cancels tasks

```python
except asyncio.CancelledError:
    raise  # Always re-raise — this is task cancellation, not an error
except SpecificError as error:
    logger.error(f"Specific issue in {self.group_name}: {error}")
except Exception as error:
    logger.error(f"Unexpected error for user {self.scope['user']}: {error}")
```

## Performance Guidelines

1. **Avoid blocking the event loop** — use `run_in_executor` for sync I/O
2. **Minimize work in event handlers** — serialize data before broadcasting, not in the handler
3. **Use connection pooling** — `django_redis` pools by default; tune `max_connections` in settings
4. **Batch Redis operations** — use pipelines when doing multiple independent Redis commands
5. **Set appropriate timeouts** — `expiry`, `socket_timeout` in channel layer config
6. **Profile message latency** — add timestamps to payloads for end-to-end measurement

## Related Skills

- [Routing & ASGI setup](../django-channels-routing/SKILL.md)
- [Authentication](../django-channels-auth/SKILL.md)
- [Redis technique selection](../django-channels-redis/SKILL.md)
- [Broadcasting & messaging patterns](../django-channels-broadcasting/SKILL.md)
- [Testing](../django-channels-testing/SKILL.md)

## Additional Resources

- Detailed transport pattern examples: [reference.md](reference.md)
- Common consumer architecture patterns: [examples.md](examples.md)
