---
name: slack-messaging
description: Slack messaging integration using incoming webhooks in a modular clients/ folder.
---

# Slack Messaging Integration

## Folder Structure

```text
integrations/clients/
├── __init__.py
├── base.py
└── slack.py
```

## Client

Create `integrations/clients/slack.py`:

```python
from django.conf import settings
from integrations.clients.base import BaseServiceClient


class SlackNotifier(BaseServiceClient):
    """Send messages to Slack via incoming webhooks."""

    SERVICE_NAME = "Slack"
    TIMEOUT = 10
    MAX_RETRIES = 2

    def get_base_url(self) -> str:
        return settings.SLACK_WEBHOOK_URL

    def get_auth_headers(self) -> dict[str, str]:
        return {}  # Webhook URL includes authentication

    def send_message(self, text: str, channel: str | None = None, blocks: list | None = None):
        """Send a message to Slack."""
        payload = {"text": text}
        if channel:
            payload["channel"] = channel
        if blocks:
            payload["blocks"] = blocks
        return self.post("", json=payload)

    def send_alert(self, title: str, message: str, severity: str = "warning"):
        """Send an alert with color-coded attachment."""
        return self.send_message(
            text=title,
            blocks=[{
                "type": "section",
                "text": {
                    "type": "mrkdwn",
                    "text": f"*{title}*\n{message}"
                }
            }]
        )
```

## Django Settings

```python
# settings.py
SLACK_WEBHOOK_URL = config("SLACK_WEBHOOK_URL")
```

## Service Wrapper

```python
# integrations/services.py
from integrations.clients.slack import SlackNotifier


class NotificationService:
    """High-level notification service."""

    @staticmethod
    def notify_admin(title: str, message: str):
        notifier = SlackNotifier()
        notifier.send_alert(title, message)
```

## Celery Task

```python
# integrations/tasks/__init__.py
from celery import shared_task
from integrations.clients.slack import SlackNotifier

@shared_task
def notify_slack_on_error(error_message: str):
    notifier = SlackNotifier()
    notifier.send_alert("Error Alert", error_message, severity="error")
```

## Testing

```python
import responses
from django.test import TestCase
from integrations.clients.slack import SlackNotifier


class TestSlackNotifier(TestCase):
    @responses.activate
    def test_send_message(self):
        responses.add(
            responses.POST,
            "https://hooks.slack.com/services/TEST/HOOK",
            json={"ok": True},
            status=200,
        )
        notifier = SlackNotifier()
        notifier.send_message("Hello from tests")
```
