---
name: django-channels-broadcasting
description: Send WebSocket messages from outside consumer context in Django Channels. Covers all messaging patterns (broadcast, unicast, per-user, request-response), sending from views/services/models/Celery/management commands, serialization best practices, group management, and choosing between group_send vs send vs Redis publish. Use when sending WebSocket messages from any part of a Django application, choosing a messaging pattern, or designing group naming schemes.
---

# WebSocket Messaging & Broadcasting

## Messaging Patterns Overview

| Pattern | Direction | Mechanism | Use Case |
|---|---|---|---|
| **Broadcast** | Server → all in group | `group_send()` | Alerts, announcements, dashboards |
| **Multicast** | Server → subset of clients | `group_send()` to filtered group | Tier-based content, role-based updates |
| **Unicast** | Server → one connection | `channel_layer.send()` | Admin to specific user, direct messages |
| **Per-user** | Server → all of one user's connections | `group_send()` to `user_{id}` group | Personal notifications, order updates |
| **Request-response** | Client → Server → same client | `receive()` → `self.send()` | Data queries, command responses |
| **Server push** | Server → client (unsolicited) | Background task → `self.send()` | Live data streaming |
| **Redis pub/sub** | Publisher → all subscribers | `redis.publish()` | High-throughput data feeds |

### Choosing a pattern

```
Who should receive the message?

All clients in a feature/room
  └─ Broadcast via group_send() to a static group

All clients matching a criteria (role, tier, region)
  └─ Multicast via group_send() to a criteria-based group

All connections of one specific user
  └─ Per-user group_send() to user_{id}_* group

One specific WebSocket connection
  └─ Unicast via channel_layer.send() to a channel_name

Responding to a client request
  └─ Request-response via self.send() in receive()

High-throughput data stream
  └─ Redis publish() directly (bypass Channels layer)
```

## Sending from Outside Consumers

The core challenge: consumers are async, but views, services, and Celery tasks are sync. Use `async_to_sync` to bridge.

### The universal pattern

```python
from channels.layers import get_channel_layer
from asgiref.sync import async_to_sync

channel_layer = get_channel_layer()
if channel_layer:
    async_to_sync(channel_layer.group_send)(
        "group_name",
        {"type": "event_handler_name", "data": serialized_payload}
    )
```

### Guard clause (mandatory)

`get_channel_layer()` returns `None` if `CHANNEL_LAYERS` is not configured (common in test environments). Always guard:

```python
if channel_layer:
    async_to_sync(channel_layer.group_send)(...)
else:
    logger.warning("Channel layer unavailable, skipping WebSocket broadcast")
```

## Source Contexts

### From Views / API Endpoints

Broadcast after a successful write operation:

```python
class OrderCreateView(CreateAPIView):
    def perform_create(self, serializer):
        order = serializer.save()
        broadcast_order_update(order)
```

### From Services

The recommended approach. Keep broadcast logic in a dedicated service function:

```python
def broadcast_order_update(order):
    """Serialize and broadcast an order update via WebSocket."""
    data = OrderSerializer(order).data
    data["server_timestamp"] = timezone.now().isoformat()
    OrderConsumer.send_message(data)
```

### From Model `save()`

Override `save()` to broadcast on every create/update:

```python
def save(self, **kwargs):
    super().save(**kwargs)
    channel_layer = get_channel_layer()
    if channel_layer:
        async_to_sync(channel_layer.group_send)(
            "Announcements",
            {"type": "notify_announcement", "data": {...}}
        )
```

**Caution**: This triggers on every save, including bulk operations and migrations. Consider using Django signals with a guard flag if you need more control.

### From Celery Tasks

Same pattern — Celery workers can access Redis (and thus the channel layer):

```python
@shared_task
def process_report(user_id, report_id):
    report = generate_report(report_id)
    channel_layer = get_channel_layer()
    if channel_layer:
        async_to_sync(channel_layer.group_send)(
            f"user_{user_id}_notifications",
            {"type": "notify_report_ready", "data": ReportSerializer(report).data}
        )
```

### From Management Commands

Useful for administrative broadcasts (maintenance mode, data reloads):

```python
class Command(BaseCommand):
    def handle(self, *args, **options):
        channel_layer = get_channel_layer()
        if channel_layer:
            async_to_sync(channel_layer.group_send)(
                "System_Alerts",
                {"type": "notify_system_event", "data": {"event": "maintenance_start"}}
            )
```

## Static Send Helper Pattern

Encapsulate broadcast logic in a `@staticmethod` on the consumer class:

### Best practices

1. **Define on the consumer** — keeps broadcast logic co-located with the handler
2. **Accept pre-serialized data** — caller is responsible for serialization
3. **Guard against missing channel layer** — log a warning, don't crash
4. **Wrap in try/except** — broadcasting should never crash the caller
5. **Log success and failure** — include message type for debugging

## Type-Method Mapping

The `"type"` key in `group_send` maps to a consumer method by replacing dots with underscores:

```
"type": "notify.order.update" → method: notify_order_update()
"type": "notify_order_update" → method: notify_order_update()
```

**Convention**: Use underscores directly in type names (simpler, no conversion needed).

## Unicast: Direct Send to One Consumer

Target a single connection by its `channel_name`:

```python
async_to_sync(channel_layer.send)(
    target_channel_name,
    {"type": "personal_message", "data": payload}
)
```

**Requirement**: You need the target's `channel_name`. Options:
- Store it in Redis on connect (e.g., `HSET user_channels:{user_id} {channel_name} connected`)
- Store it in the database if connections are long-lived
- Use per-user groups instead (simpler, handles multiple connections)

**Prefer per-user groups over unicast** when targeting a user (not a specific connection).

## Serialization Best Practices

1. **Always serialize before broadcasting** — never send raw Django model instances
2. **Use DRF serializers** — consistent with your API responses
3. **Add `server_timestamp`** — enables client-side latency measurement
4. **Re-fetch with `select_related`/`prefetch_related`** — ensure complete data for serialization
5. **Keep payloads lean** — don't send the entire object if clients only need a few fields

## Group Naming Conventions

| Pattern | Format | Example |
|---|---|---|
| Static feature groups | PascalCase_Underscore | `Order_Updates`, `System_Alerts` |
| Dynamic topic groups | snake_case | `chat_room_42`, `activity_feed` |
| Per-user groups | `user_{id}_purpose` | `user_42_notifications` |
| Data stream channels | UPPERCASE_components | `PRICES_SPX_1T`, `METRICS_CPU` |

### Rules
- Group names must be strings, no special characters except underscores
- Keep names deterministic — both sender and consumer must construct the same name
- Document the naming scheme for your project

## High-Throughput Broadcasting (Direct Redis)

For data feeds exceeding Channels layer capacity, publish directly to Redis:

```python
redis_conn.publish(channel, json.dumps({
    "enqueued_at": time.time(),
    "payload": data,
}))
```

The consumer's background listener picks up the message. See [redis skill](../django-channels-redis/SKILL.md) for technique selection.

## Checklist for New Broadcast Points

- [ ] Choose the right pattern (broadcast, unicast, per-user, pub/sub)
- [ ] Guard with `if channel_layer:` check
- [ ] `"type"` matches the consumer's event handler method name
- [ ] Data is serialized (DRF serializer, not raw model)
- [ ] `server_timestamp` included in payload
- [ ] Exception handling — broadcasting never crashes the caller
- [ ] Group name follows project naming conventions
- [ ] Tested: message arrives at connected client

## Related Skills

- [Consumers](../django-channels-consumers/SKILL.md) — event handlers that receive these messages
- [Redis techniques](../django-channels-redis/SKILL.md) — choosing between group_send and direct Redis
- [Testing](../django-channels-testing/SKILL.md) — verifying broadcasts in tests
