---
name: django-channels-testing
description: Test WebSocket consumers in Django Channels projects. Covers WebsocketCommunicator, test infrastructure setup, authentication strategies in tests, unit vs integration test boundaries, Redis mocking, channel layer testing, and common test patterns. Use when writing WebSocket tests, setting up test infrastructure, mocking auth or Redis, or debugging test failures in any Django Channels project.
---

# WebSocket Consumer Testing

## Test Infrastructure

### Required packages

| Package | Purpose |
|---|---|
| `pytest` | Test runner |
| `pytest-asyncio` | Async test support (`@pytest.mark.asyncio`) |
| `pytest-django` | Django test integration |
| `channels[daphne]` | Includes `channels.testing` module |
| `fakeredis` | In-memory Redis mock for unit tests |

### Key tool: WebsocketCommunicator

`channels.testing.WebsocketCommunicator` is the primary tool for testing consumers. It simulates a WebSocket client.

```python
from channels.testing import WebsocketCommunicator

communicator = WebsocketCommunicator(
    MyConsumer.as_asgi(),  # The consumer under test
    "/ws/my-endpoint/"      # The WebSocket URL path
)
```

### Channel layer for tests

Replace the Redis channel layer with `InMemoryChannelLayer` for unit tests:

```python
@pytest.fixture
def channel_layer(settings):
    settings.CHANNEL_LAYERS = {
        "default": {
            "BACKEND": "channels.layers.InMemoryChannelLayer",
        }
    }
    from channels.layers import get_channel_layer
    return get_channel_layer()
```

## Unit vs Integration Tests

| Aspect | Unit Test | Integration Test |
|---|---|---|
| **Channel layer** | `InMemoryChannelLayer` | Real Redis |
| **Redis** | `fakeredis` mock | Real Redis |
| **Auth** | `scope["user"]` set directly | Real JWT token |
| **Database** | Factory or fixtures | Real database |
| **Speed** | Fast (ms) | Slower (seconds) |
| **Scope** | One consumer in isolation | Full ASGI stack |

### When to use each

**Unit tests** (majority of tests):
- Consumer lifecycle (connect, disconnect)
- Input validation in `receive`
- Event handler behavior
- Error handling paths

**Integration tests** (selective):
- Full auth flow (JWT → middleware → consumer)
- Real Redis pub/sub delivery
- Cross-process group messaging
- End-to-end latency measurement

## Authentication in Tests

### Strategy 1: Bypass middleware (unit tests)

Set `scope["user"]` directly on the communicator:

```python
@pytest.mark.asyncio
async def test_consumer_connect(db, user_factory):
    user = user_factory()
    communicator = WebsocketCommunicator(
        MyConsumer.as_asgi(),
        "/ws/my-endpoint/"
    )
    communicator.scope["user"] = user

    connected, _ = await communicator.connect()
    assert connected
    await communicator.disconnect()
```

Best for unit tests — tests the consumer logic without middleware complexity.

### Strategy 2: Full ASGI stack (integration tests)

Test through the full `application` with a real token:

```python
from config.asgi import application

@pytest.mark.asyncio
async def test_full_auth_flow(db, user_with_token):
    user, token = user_with_token
    communicator = WebsocketCommunicator(
        application,
        f"/ws/my-endpoint/?token={token}"
    )

    connected, _ = await communicator.connect()
    assert connected
    await communicator.disconnect()
```

### Strategy 3: Mock the auth middleware

Mock the token validation function to avoid creating real JWTs:

```python
from unittest.mock import patch

@pytest.mark.asyncio
async def test_with_mocked_auth(db, user_factory):
    user = user_factory()
    with patch("config.asgi.get_user_from_token", return_value=user):
        communicator = WebsocketCommunicator(
            application,
            "/ws/my-endpoint/?token=any_value"
        )
        connected, _ = await communicator.connect()
        assert connected
        await communicator.disconnect()
```

## Core Test Patterns

### 1. Connection acceptance

```python
@pytest.mark.asyncio
async def test_accepts_connection(db, user_factory):
    communicator = WebsocketCommunicator(MyConsumer.as_asgi(), "/ws/path/")
    communicator.scope["user"] = user_factory()

    connected, close_code = await communicator.connect()
    assert connected
    assert close_code is None

    await communicator.disconnect()
```

### 2. Connection rejection

```python
@pytest.mark.asyncio
async def test_rejects_without_token(db):
    communicator = WebsocketCommunicator(application, "/ws/path/")

    connected, close_code = await communicator.connect()
    assert not connected
    assert close_code == 4000  # No token


@pytest.mark.asyncio
async def test_rejects_invalid_token(db):
    communicator = WebsocketCommunicator(
        application, "/ws/path/?token=invalid"
    )

    connected, close_code = await communicator.connect()
    assert not connected
    assert close_code == 4001  # Invalid token
```

### 3. Receiving messages

```python
# JSON response
response = await communicator.receive_json_from()
assert response["status"] == "connected"

# Raw text response
text = await communicator.receive_from()
assert "connected" in text

# With timeout (for messages that may not arrive)
response = await communicator.receive_json_from(timeout=2)
```

### 4. Sending messages

```python
# JSON
await communicator.send_json_to({"action": "SUBSCRIBE", "topic": "updates"})

# Raw text
await communicator.send_to(text_data='{"action": "SUBSCRIBE"}')
```

### 5. Broadcast verification

Test that `group_send` delivers to connected clients:

```python
@pytest.mark.asyncio
async def test_broadcast_delivery(db, user_factory, channel_layer):
    communicator = WebsocketCommunicator(MyConsumer.as_asgi(), "/ws/path/")
    communicator.scope["user"] = user_factory()

    await communicator.connect()
    await communicator.receive_json_from()  # Consume connection message

    # Broadcast via channel layer
    from channels.layers import get_channel_layer
    layer = get_channel_layer()
    await layer.group_send(
        "My_Group",
        {"type": "notify_update", "data": {"id": 1, "status": "ready"}}
    )

    response = await communicator.receive_json_from()
    assert response["id"] == 1
    assert response["status"] == "ready"

    await communicator.disconnect()
```

### 6. Input validation

```python
@pytest.mark.asyncio
async def test_rejects_invalid_input(db, user_factory):
    communicator = WebsocketCommunicator(MyConsumer.as_asgi(), "/ws/path/")
    communicator.scope["user"] = user_factory()

    await communicator.connect()
    await communicator.receive_json_from()

    # Missing required field
    await communicator.send_json_to({"action": "SUBSCRIBE"})
    response = await communicator.receive_json_from()
    assert "error" in response

    # Invalid action
    await communicator.send_json_to({"action": "INVALID", "topic": "x"})
    response = await communicator.receive_json_from()
    assert "error" in response

    await communicator.disconnect()
```

### 7. Disconnect cleanup

```python
@pytest.mark.asyncio
async def test_disconnect_cleans_groups(db, user_factory, channel_layer):
    communicator = WebsocketCommunicator(MyConsumer.as_asgi(), "/ws/path/")
    communicator.scope["user"] = user_factory()

    await communicator.connect()
    await communicator.receive_json_from()
    await communicator.disconnect()

    # Sending to the group after disconnect should not error
    layer = get_channel_layer()
    await layer.group_send(
        "My_Group",
        {"type": "notify_update", "data": {"check": "no error"}}
    )
```

## Redis Mocking

### fakeredis for unit tests

```python
import fakeredis
from unittest.mock import patch

@pytest.fixture
def mock_redis():
    fake = fakeredis.FakeRedis()
    with patch("django_redis.get_redis_connection", return_value=fake):
        yield fake
```

Use when testing consumers that use direct Redis (pub/sub, sets, counters).

### Real Redis for integration tests

Mark integration tests and run them selectively:

```python
@pytest.mark.integration
@pytest.mark.asyncio
async def test_pubsub_delivery(db, user_factory):
    """Requires running Redis."""
    # Test with real Redis pub/sub
```

## Common Testing Mistakes

1. **Missing `@pytest.mark.asyncio`** — test silently doesn't run as async
2. **Not consuming intermediate messages** — `receive_json_from()` returns the next queued message, not the one you expect. Consume connection confirmations before testing broadcast delivery.
3. **Forgetting `await communicator.disconnect()`** — leaks resources, may affect other tests
4. **Using production channel layer in tests** — always override with `InMemoryChannelLayer`
5. **Not setting `scope["user"]`** — consumer crashes on `self.scope["user"]`
6. **Testing pub/sub without fakeredis** — test tries to connect to real Redis and fails
7. **Timeout too short on `receive_json_from`** — async tests may need > 1s timeout in CI
8. **Not awaiting communicator methods** — easy to forget `await` on `send_json_to` or `receive_json_from`

## Logging Verification

```python
from unittest.mock import patch

@pytest.mark.asyncio
async def test_logs_connection(db, user_factory):
    communicator = WebsocketCommunicator(MyConsumer.as_asgi(), "/ws/path/")
    communicator.scope["user"] = user_factory()

    with patch("myapp.consumers.logger") as mock_logger:
        await communicator.connect()
        mock_logger.info.assert_called()
        await communicator.disconnect()
```

## Related Skills

- [Consumers](../django-channels-consumers/SKILL.md) — the consumers being tested
- [Auth](../django-channels-auth/SKILL.md) — auth strategies that affect test setup
- [Redis](../django-channels-redis/SKILL.md) — Redis techniques that need mocking
