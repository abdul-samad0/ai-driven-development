# Consumer Reference — Transport Pattern Deep Dive

## Channels Layer Consumer Structure

This is the standard pattern for consumers using `group_send`. Suitable for notifications, alerts, chat, activity feeds — any moderate-throughput broadcast.

### Lifecycle flow

```
Client connects
  → connect(): accept → log → group_add → send confirmation
External event (view/service/Celery)
  → group_send(group, {type, data})
  → event handler: forward data to client
Client disconnects
  → disconnect(): group_discard → log → cleanup
```

### Key implementation points

1. **`__init__`** initializes `group_name` and any instance state. No I/O.
2. **`connect`** calls `accept()` first, then joins groups. Wrap in try/except.
3. **Event handlers** are named to match the `type` field: `"notify_update"` → `notify_update(self, event)`.
4. **`disconnect`** calls `group_discard` for every group joined. Log the close code.
5. **Static `send_message()`** enables broadcasting from sync code without a consumer instance.

### When to add `receive`

Only add a `receive` method if clients send messages (subscribe/unsubscribe, chat input, commands). Broadcast-only consumers don't need it.

### Database queries

All ORM calls require `@database_sync_to_async`:

```python
@database_sync_to_async
def check_permission(self):
    return self.scope["user"].has_perm("myapp.view_resource")
```

---

## Direct Redis Pub/Sub Consumer Structure

This pattern bypasses the Channels layer for high-throughput streaming. The consumer manages its own Redis subscription and background listener.

### Lifecycle flow

```
Client connects
  → connect(): accept → log → INCR global counter
  → initialize pubsub → create_task(_listen_to_redis)
Client sends subscribe command
  → receive(): validate → redis_pubsub.subscribe(channel)
  → add to subscribed list → sadd for tracking
Background listener (_listen_to_redis)
  → polls get_message via run_in_executor
  → deserializes message envelope → forwards payload to WebSocket
Client disconnects
  → disconnect(): cancel listener task → close pubsub
  → srem from all tracked sets → DECR global counter
```

### Key implementation points

1. **`asyncio.create_task`** starts the background listener on connect
2. **`run_in_executor`** wraps every blocking Redis call (subscribe, unsubscribe, get_message, close)
3. **`get_message(ignore_subscribe_messages=True, timeout=0.01)`** — 10ms poll timeout for responsive streaming
4. **`asyncio.sleep(0.001)`** yields control when no messages, preventing busy-wait
5. **`CancelledError`** must be caught and re-raised in the listener for clean shutdown
6. **Message envelope** wraps payload with `enqueued_at` timestamp for latency tracking
7. **Sequence counter** (`_seq`) enables client-side gap detection
8. **Disconnect cleanup order**: cancel task → await task → close pubsub → clean sets → decrement counter

### Message envelope format

Publishers should wrap data for latency measurement:

```python
{
    "enqueued_at": 1711900800.123,
    "payload": { ... actual data ... }
}
```

The consumer extracts `enqueued_at` and passes it through so the client can calculate end-to-end latency.

---

## Direct Redis Streams Consumer Structure

Use Streams when messages must not be lost, when consumers need to replay from a position, or when you need consumer groups for load balancing.

### Lifecycle flow

```
Client connects
  → connect(): accept → create_task(_listen_to_stream)
Background listener (_listen_to_stream)
  → XREAD or XREADGROUP with block timeout
  → forwards each entry to WebSocket → updates last_id
Client disconnects
  → disconnect(): cancel listener task
```

### Key implementation points

1. **`XREAD`** for simple consumers — reads from a stream position
2. **`XREADGROUP`** for consumer groups — distributes messages across workers, supports `XACK`
3. **`last_message_id`** tracks position — enables replay from last known ID on reconnect
4. **`MAXLEN`** on publish side — bounds memory usage by trimming old entries
5. **Block timeout** in `XREAD` — use 100-500ms for responsive reading without busy-wait

### When to choose Streams over Pub/Sub

| Need | Pub/Sub | Streams |
|---|---|---|
| Message durability | No — lost if no subscriber | Yes — persisted until trimmed |
| Replay / catch-up | No | Yes — read from any position |
| Consumer groups | No | Yes — XREADGROUP + XACK |
| Delivery guarantee | At-most-once | At-least-once (with XACK) |
| Throughput | Higher (no persistence overhead) | Slightly lower (disk writes) |

---

## Hybrid Consumer (Multiple Groups + Conditional Joins)

Some consumers join multiple groups — a static broadcast group plus dynamic per-user groups.

### Pattern

1. **Static group** — joined unconditionally on connect (e.g., system-wide alerts)
2. **Dynamic per-user group** — joined conditionally based on user state (e.g., `user_{id}_notifications`)
3. **Separate event handlers** — one per group type
4. **Disconnect cleans up all groups** — both static and dynamic

### Key implementation points

1. Store each group name as an instance variable so `disconnect` can discard all
2. Use `@database_sync_to_async` for the eligibility check
3. If the user isn't eligible for the per-user group, log it and skip — don't error

---

## Static Send Helper Pattern

For broadcasting from sync code (views, services, Celery tasks, model methods) without a consumer instance:

### Best practices

1. **Define as `@staticmethod`** on the consumer class — keeps broadcast logic co-located
2. **Guard with `if channel_layer:`** — returns `None` in environments without Redis
3. **Wrap in try/except** — broadcasting should never crash the caller
4. **Accept pre-serialized data** — don't serialize inside the helper; let the caller handle it
5. **Log success and failure** — include the message type or a summary for debugging
