# Consumer Architecture Patterns

## Pattern 1: Broadcast-Only Consumer

**Use when**: All clients receive the same data. No client input needed.

### Characteristics
- No `receive` method
- Single static group
- One event handler
- Optional static `send_message()` for external callers

### Architecture
```
[View/Service/Celery] → send_message() → group_send() → event handler → self.send()
```

### When to use
- System-wide alerts and announcements
- Real-time dashboards where all users see the same data
- Status page updates
- Maintenance mode notifications

---

## Pattern 2: Subscribe/Unsubscribe Consumer (Channels Layer)

**Use when**: Clients dynamically choose which topics/channels to follow.

### Characteristics
- `receive` handles `SUBSCRIBE` / `UNSUBSCRIBE` actions
- Maintains a `subscribed_topics` list per connection
- Joins/leaves Channels groups dynamically
- Validates input before acting

### Architecture
```
Client → receive({action: "SUBSCRIBE", topic: "X"}) → group_add("X")
Server → group_send("X", data) → event handler → self.send()
Client → receive({action: "UNSUBSCRIBE", topic: "X"}) → group_discard("X")
```

### Input validation checklist
- [ ] JSON is parseable
- [ ] `action` field is present and valid (`SUBSCRIBE` or `UNSUBSCRIBE`)
- [ ] `topic`/`channel` field is present and non-empty
- [ ] Topic name passes allowlist or format validation (prevent injection)
- [ ] User has permission to subscribe to the requested topic

---

## Pattern 3: Per-User Group Consumer

**Use when**: Each user receives personalized messages (order updates, direct messages).

### Characteristics
- Group name includes user ID: `user_{id}_notifications`
- Joined on connect (or conditionally based on user state)
- Sender targets a specific user by constructing the group name
- If user is offline, message is silently dropped

### Architecture
```
[Service] → group_send("user_42_notifications", data)
  → Only user 42's consumer instances receive it
```

### Considerations
- One user may have multiple connections (multiple tabs/devices) — all receive the message
- Use `@database_sync_to_async` if eligibility check requires ORM
- Always `group_discard` in `disconnect` for both static and per-user groups

---

## Pattern 4: High-Throughput Streaming Consumer (Redis Pub/Sub)

**Use when**: Data volume is too high for the Channels layer (> 100 msg/sec per channel).

### Characteristics
- Bypasses Channels layer entirely
- Uses `django_redis.get_redis_connection()` directly
- Background `asyncio.create_task` polls Redis pub/sub
- `run_in_executor` for all blocking Redis calls
- Connection tracking with Redis sets (`SADD`/`SREM`)
- Global connection counter with `INCR`/`DECR`

### Architecture
```
[Publisher] → redis.publish(channel, envelope)
  → _listen_to_redis() → get_message() → self.send()
```

### Cleanup order (critical)
1. Cancel the listener task
2. Await the task (catch `CancelledError`)
3. Close the pubsub connection (via `run_in_executor`)
4. Remove from all Redis tracking sets
5. Decrement the global connection counter (guard against negative)

---

## Pattern 5: Durable Streaming Consumer (Redis Streams)

**Use when**: Messages must not be lost, or consumers need replay/catch-up.

### Characteristics
- Uses Redis Streams (`XREAD` or `XREADGROUP`)
- Tracks `last_message_id` per connection for replay
- Background `asyncio.create_task` reads from stream
- Supports consumer groups for load-balanced consumption

### Architecture
```
[Publisher] → redis.xadd(stream, data, maxlen=N)
  → _listen_to_stream() → xread() → self.send()
```

### When to prefer over Pub/Sub
- Financial transactions, audit events, order status updates
- Consumers that reconnect and need to catch up on missed messages
- Multiple workers processing from the same stream (consumer groups)

---

## Pattern 6: Request-Response Consumer

**Use when**: Client sends a query, server responds with data (no ongoing subscription).

### Characteristics
- `receive` processes the request and responds directly via `self.send()`
- No groups involved — communication is purely between one client and the server
- Can combine with subscription patterns (e.g., request initial data, then subscribe to updates)

### Architecture
```
Client → receive({action: "GET_STATUS", order_id: 42})
  → query DB → self.send({status: "shipped"})
```

### Best practices
- Validate all input fields
- Use `@database_sync_to_async` for ORM queries
- Return structured error responses for invalid requests
- Consider rate-limiting if queries are expensive

---

## Anti-Patterns to Avoid

### 1. Fat Consumers
**Problem**: Business logic, serialization, and DB queries inside consumer methods.
**Fix**: Move to a service layer. Consumer should only handle WebSocket lifecycle and message routing.

### 2. Sync Consumers for Async Workloads
**Problem**: Using `WebsocketConsumer` (sync) when the consumer does I/O.
**Fix**: Use `AsyncWebsocketConsumer`. Sync consumers tie up a thread per connection.

### 3. Unguarded Global Counters
**Problem**: `DECR` without checking current value → counter goes negative.
**Fix**: Always check `if current > 0` before decrementing.

### 4. Missing Cleanup in Disconnect
**Problem**: Forgetting to cancel tasks, close pubsub, or clean Redis sets.
**Fix**: Follow the disconnect cleanup order. Test that resources are freed.

### 5. Blocking the Event Loop
**Problem**: Calling sync Redis/DB/HTTP directly in async code.
**Fix**: `@database_sync_to_async` for ORM, `run_in_executor` for everything else.

### 6. No Input Validation in Receive
**Problem**: Trusting client-sent JSON without validation.
**Fix**: Validate all fields, check allowlists, catch `JSONDecodeError`.
