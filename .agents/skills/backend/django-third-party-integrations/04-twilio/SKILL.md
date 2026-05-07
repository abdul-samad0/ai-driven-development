---
name: twilio-sms
description: SMS integration using Twilio SDK in a modular clients/ folder.
---

# Twilio SMS Integration

## Folder Structure

```text
integrations/clients/
├── __init__.py
└── twilio.py
```

## Client

Create `integrations/clients/twilio.py`:

```python
from twilio.rest import Client
from django.conf import settings


class TwilioService:
    """SMS service using Twilio SDK."""

    def __init__(self):
        self._client = Client(
            settings.TWILIO_ACCOUNT_SID,
            settings.TWILIO_AUTH_TOKEN
        )
        self._from = settings.TWILIO_FROM_NUMBER

    def send_sms(self, to: str, body: str):
        """Send an SMS message."""
        message = self._client.messages.create(
            to=to,
            from_=self._from,
            body=body
        )
        return {"sid": message.sid, "status": message.status}

    def get_message_status(self, message_sid: str):
        """Check SMS delivery status."""
        message = self._client.messages(message_sid).fetch()
        return {"sid": message.sid, "status": message.status}
```

## Django Settings

```python
# settings.py
TWILIO_ACCOUNT_SID = config("TWILIO_ACCOUNT_SID")
TWILIO_AUTH_TOKEN = config("TWILIO_AUTH_TOKEN")
TWILIO_FROM_NUMBER = config("TWILIO_FROM_NUMBER")
```

## Celery Task

```python
# integrations/tasks/__init__.py
from celery import shared_task
from integrations.clients.twilio import TwilioService

@shared_task
def send_sms_notification(phone: str, message: str):
    service = TwilioService()
    result = service.send_sms(phone, message)
    return result
```

## Testing

```python
from unittest.mock import MagicMock, patch
from django.test import TestCase
from integrations.clients.twilio import TwilioService


class TestTwilioService(TestCase):
    @patch("integrations.clients.twilio.Client")
    def test_send_sms(self, mock_client_cls):
        mock_message = MagicMock(sid="SM123", status="queued")
        mock_client_cls.return_value.messages.create.return_value = mock_message

        service = TwilioService()
        result = service.send_sms("+15550001111", "Hello")

        self.assertEqual(result["sid"], "SM123")
        self.assertEqual(result["status"], "queued")
```
