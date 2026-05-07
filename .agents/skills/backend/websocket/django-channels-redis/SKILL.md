---
name: django-channels-redis
description: Choose and implement the right Redis technique for WebSocket communication in Django Channels. Covers decision framework for Redis Pub/Sub vs Streams vs Pipelines vs Channel Layer propagation vs Lists, configuration tuning, connection pooling, and operational patterns. Use when choosing a Redis data structure for real-time messaging, configuring CHANNEL_LAYERS, optimizing Redis performance, or deciding between Channels layer propagation and direct Redis access.
---

# Redis Technique Selection for Django WebSockets

## The Core Decision: Channels Layer vs Direct Redis

Before choosing a specific Redis technique, decide whether to use the **Channels Redis layer** (abstracted) or **direct Redis access**.

| Approach | How | When |
|---|---|---|
| **Channels layer** (`group_send`) | `channels_redis.core.RedisChannelLayer` — Redis is hidden behind an abstraction | Moderate throughput, simple broadcast, you want Django Channels to manage groups |
| **Direct Redis** (`django_redis`) | `get_redis_connection()` — you write Redis commands directly | High throughput, need specific Redis features (pub/sub, streams, pipelines), fine-grained control |

**You can use both in the same project.** Some consumers use the Channels layer (notifications, alerts), while others use direct Redis (live data feeds).

## Redis Technique Comparison

| Technique | Delivery | Durability | Throughput | Consumer Groups | Use Case |
|---|---|---|---|---|---|
| **Channels layer** (propagation) | Fan-out to group | Ephemeral (TTL-based) | Moderate | Via Channels groups | Notifications, alerts, chat |
| **Pub/Sub** | Fan-out to subscribers | None — fire and forget | High | No | Live data feeds, tickers, metrics |
| **Streams** | Pull-based or push via XREAD | Persistent until trimmed | Moderate-high | Yes (XREADGROUP) | Order events, audit logs, durable feeds |
| **Lists** (LPUSH/BRPOP) | Point-to-point queue | Persistent until consumed | Moderate | Manual (multiple lists) | Task distribution, work queues |
| **Pipeline** | N/A — batching optimization | N/A | N/A | N/A | Any multi-command operation |
| **Sets** (SADD/SREM) | N/A — data structure | Persistent | N/A | N/A | Connection tracking, membership |
| **Strings** (INCR/DECR) | N/A — data structure | Persistent | N/A | N/A | Counters, flags |

## Decision Framework

```
What is the primary requirement?

1. BROADCAST messages to a group of WebSocket clients
   ├─ Message rate < 100/sec per group?
   │   └─ YES → Channels Layer (group_send)
   │            Simplest. No direct Redis code. Built-in group management.
   │
   └─ Message rate > 100/sec per group?
       ├─ Can messages be lost if a client is slow?
       │   └─ YES → Redis Pub/Sub (direct)
       │            Highest throughput. Fire-and-forget. No persistence.
       │
       └─ Messages must not be lost?
           └─ Redis Streams (XADD/XREAD)
              Persistent. Replay from position. Consumer groups.

2. QUEUE messages for a single worker to process
   └─ Redis Lists (LPUSH/BRPOP) or Celery
      Point-to-point. Each message consumed once. Consider Celery if
      you already have it — it uses Redis/RabbitMQ as broker.

3. TRACK state (who is connected, how many connections)
   ├─ Track unique members per group → Redis Sets (SADD/SREM/SMEMBERS)
   └─ Track a global count → Redis Strings (INCR/DECR)

4. OPTIMIZE multiple Redis calls
   └─ Redis Pipeline
      Batch N commands into 1 network round-trip.
      Use for disconnect cleanup, bulk operations.
```

## Technique 1: Channels Layer (Propagation)

The default approach. Redis acts as a message broker behind the `channels_redis` backend.

### How it works
1. `group_add(group, channel)` — registers a consumer in a group (stored in Redis)
2. `group_send(group, message)` — Redis propagates the message to all members
3. Each consumer's event handler receives the message and sends to the WebSocket
4. `group_discard(group, channel)` — removes the consumer from the group

### Configuration

```python
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {
            "hosts": [{"address": REDIS_URL}],
            "capacity": 50000,     # Max queued messages per channel
            "expiry": 5,           # Message TTL in seconds
            "group_expiry": 86400, # Group membership TTL (24h)
        },
    },
}
```

### Tuning guide

| Setting | Default | Tune When |
|---|---|---|
| `capacity` | 100 | Increase for bursty traffic. Too low → messages dropped |
| `expiry` | 60 | Decrease for real-time data (stale messages are useless). 5s is aggressive but appropriate for live data |
| `group_expiry` | 86400 | Time before idle group memberships are pruned. 24h is usually fine |
| `max_connections` | 10 | Increase proportionally to concurrent WebSocket connections. 300+ for production |
| `socket_timeout` | 5 | Increase if Redis is remote or under heavy load |
| `retry_on_timeout` | False | Set `True` for production resilience |

### When to use
- Notifications, alerts, announcements
- Chat rooms with moderate message rates
- Activity feeds, status updates
- Any broadcast where < 100 msg/sec per group

### Limitations
- Higher latency than direct Redis (abstraction overhead)
- No message persistence (ephemeral by design)
- No consumer group semantics (every member receives every message)

## Technique 2: Redis Pub/Sub (Direct)

Bypass the Channels layer for maximum throughput. Publisher pushes, subscribers receive immediately.

### How it works
1. **Publisher** calls `redis_conn.publish(channel, serialized_data)`
2. **Consumer** has a background `asyncio.create_task` polling `pubsub.get_message()`
3. Messages are delivered to all subscribers instantly
4. If no subscribers are listening, the message is discarded

### When to use
- Live market data, stock tickers, price feeds
- Real-time metrics and monitoring dashboards
- IoT sensor data streams
- Any high-frequency broadcast (> 100 msg/sec per channel)

### When NOT to use
- Messages must not be lost → use Streams
- Consumers need to replay/catch up → use Streams
- Low message rate where simplicity matters → use Channels layer

### Key implementation considerations
- All blocking Redis calls must use `asyncio.get_event_loop().run_in_executor()`
- Background listener must handle `CancelledError` for clean shutdown
- Add a message envelope with `enqueued_at` timestamp for latency tracking
- Track connections with Redis Sets for observability

## Technique 3: Redis Streams

Durable, persistent message log with replay and consumer group support.

### How it works
1. **Publisher** calls `redis_conn.xadd(stream, data, maxlen=N)` — appends to the stream
2. **Consumer** calls `xread(streams, last_id, block=timeout)` — reads new entries from a position
3. With consumer groups: `xreadgroup(group, consumer, streams, ">", block=timeout)` — distributes messages
4. Consumer acknowledges processing: `xack(stream, group, message_id)`

### When to use
- **Durability required**: Messages persisted until explicitly trimmed
- **Replay/catch-up**: New or reconnecting consumers can read from any position
- **Load balancing**: Consumer groups distribute messages across workers
- **At-least-once delivery**: `XACK` ensures messages are processed
- **Audit/compliance**: Stream is an append-only log

### When NOT to use
- Fire-and-forget broadcasts (pub/sub is simpler and faster)
- Very high frequency where persistence overhead is unacceptable

### Key implementation considerations
- Set `MAXLEN` to bound memory usage: `xadd(stream, data, maxlen=10000)`
- Track `last_message_id` per consumer for resume-from-position
- Use `block` timeout (100-500ms) to avoid busy-waiting
- `XREADGROUP` + `XACK` for at-least-once delivery guarantees

### Pub/Sub vs Streams comparison

| Factor | Pub/Sub | Streams |
|---|---|---|
| Message persistence | None | Until trimmed |
| Missed messages | Lost forever | Can replay |
| Consumer groups | No | Yes |
| Delivery guarantee | At-most-once | At-least-once (with XACK) |
| Throughput | Higher | Slightly lower |
| Memory usage | Minimal | Grows with retention |
| Complexity | Lower | Higher |

## Technique 4: Redis Pipeline

Not a messaging technique — it's a **performance optimization** for batching multiple Redis commands.

### How it works
Instead of N network round-trips for N commands, pipeline sends all commands in one batch and reads all responses at once.

### When to use
- **Disconnect cleanup**: Removing from many tracking sets
- **Bulk counting**: Checking connection counts across multiple groups
- **Any loop** with 3+ independent Redis commands

### Best practice
```python
pipe = redis_connection.pipeline()
for channel in subscribed_channels:
    pipe.srem(channel, channel_name)
pipe.execute()  # Single round-trip instead of N
```

### When NOT to use
- When commands depend on each other (command B needs result of command A)
- Single commands (overhead of pipeline setup isn't worth it)

## Technique 5: Redis Lists (Task Queues)

Point-to-point messaging where each message is consumed by exactly one worker.

### How it works
1. **Producer** calls `LPUSH(queue, message)` — adds to the left
2. **Consumer** calls `BRPOP(queue, timeout)` — blocks until a message is available, pops from the right

### When to use
- Work distribution to background workers
- Task queues (though Celery is usually better for this in Django)
- Rate-limited processing pipelines

### When NOT to use
- Fan-out/broadcast (use pub/sub or Channels layer)
- Durable event log (use Streams)
- You already have Celery (use Celery tasks instead)

## Connection Tracking Patterns

### Per-group connection tracking (Sets)
```
SADD group_name channel_name    # On subscribe
SREM group_name channel_name    # On unsubscribe
SMEMBERS group_name             # Get all connections
SCARD group_name                # Count connections
```

### Global connection counter (Strings)
```
INCR GLOBAL_CONNECTION_COUNT    # On connect
DECR GLOBAL_CONNECTION_COUNT    # On disconnect (with guard)
GET GLOBAL_CONNECTION_COUNT     # Read current count
```

**Critical**: Always guard DECR against negative values. Process crashes can leave the counter inconsistent.

## Connection Pooling

`django_redis` manages connection pools automatically. Tune `max_connections` based on your workload:

| Deployment | Suggested `max_connections` |
|---|---|
| Development | 10 (default) |
| Staging | 50-100 |
| Production (moderate) | 100-300 |
| Production (high-throughput) | 300-1000 |

Monitor pool exhaustion with Redis `INFO clients` and your application's error logs.

## Related Skills

- [Consumers](../django-channels-consumers/SKILL.md) — how consumers use these Redis techniques
- [Broadcasting](../django-channels-broadcasting/SKILL.md) — sending messages from outside consumers
- [Routing](../django-channels-routing/SKILL.md) — ASGI configuration where CHANNEL_LAYERS is used
