---
name: authentication-system
description: OAuth 2.0 and API key authentication patterns with ORM models in a modular structure.
---

# Authentication System

## Folder Structure

```text
integrations/
├── models/
│   ├── __init__.py
│   └── OAuthToken.py
├── clients/
│   ├── __init__.py
│   ├── base.py
│   └── stripe.py
└── tasks/
    ├── __init__.py
    └── token_refresh_task.py
```

## API Key Authentication

Implement `get_auth_headers()` in client classes:

```python
# integrations/clients/stripe.py
from django.conf import settings
from integrations.clients.base import BaseServiceClient


class StripeClient(BaseServiceClient):
    SERVICE_NAME = "Stripe"

    def get_base_url(self) -> str:
        return "https://api.stripe.com/v1"

    def get_auth_headers(self) -> dict[str, str]:
        return {"Authorization": f"Bearer {settings.STRIPE_SECRET_KEY}"}

    def create_charge(self, amount: int, currency: str, source: str) -> dict:
        return self.post("/charges", json={
            "amount": amount,
            "currency": currency,
            "source": source
        })
```

Store keys in environment variables:

```python
# settings.py
from decouple import config

STRIPE_SECRET_KEY = config("STRIPE_SECRET_KEY")
GITHUB_TOKEN = config("GITHUB_TOKEN")
SENDGRID_API_KEY = config("SENDGRID_API_KEY")
```

## OAuth 2.0 — OAuthToken Model

Create `integrations/models/OAuthToken.py`:

```python
from django.db import models
from django.utils import timezone


class OAuthToken(models.Model):
    """Store OAuth tokens for third-party services."""

    user = models.ForeignKey("auth.User", on_delete=models.CASCADE, related_name="oauth_tokens")
    provider = models.CharField(max_length=64, db_index=True)  # "github", "google", etc.
    access_token = models.TextField()
    refresh_token = models.TextField(blank=True)
    expires_at = models.DateTimeField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        unique_together = ("user", "provider")
        indexes = [
            models.Index(fields=["user", "provider"]),
            models.Index(fields=["expires_at"]),
        ]

    @property
    def is_expired(self) -> bool:
        return timezone.now() >= self.expires_at

    def __str__(self):
        return f"{self.user} - {self.provider}"
```

Update `integrations/models/__init__.py`:

```python
from integrations.models.OAuthToken import OAuthToken

__all__ = ["OAuthToken"]
```

## OAuth 2.0 — Client Credentials Flow

For server-to-server OAuth:

```python
# integrations/clients/oauth_client.py
from django.core.cache import cache
from django.conf import settings
import requests

from integrations.clients.base import BaseServiceClient, ServiceClientError


class OAuthServiceClient(BaseServiceClient):
    TOKEN_URL: str = ""
    TOKEN_CACHE_KEY: str = ""
    CLIENT_ID: str = ""
    CLIENT_SECRET: str = ""

    def get_auth_headers(self) -> dict[str, str]:
        return {"Authorization": f"Bearer {self._get_token()}"}

    def _get_token(self) -> str:
        token = cache.get(self.TOKEN_CACHE_KEY)
        if token:
            return token
        return self._refresh_token()

    def _refresh_token(self) -> str:
        try:
            resp = requests.post(self.TOKEN_URL, data={
                "grant_type": "client_credentials",
                "client_id": self.CLIENT_ID,
                "client_secret": self.CLIENT_SECRET,
            }, timeout=10)
            resp.raise_for_status()
        except requests.RequestException as exc:
            raise ServiceClientError(self.SERVICE_NAME, f"Token refresh failed: {exc}") from exc

        data = resp.json()
        token = data["access_token"]
        expires_in = data.get("expires_in", 3600)
        cache.set(self.TOKEN_CACHE_KEY, token, timeout=expires_in - 60)
        return token
```

## OAuth 2.0 — User Delegated Flow

Service function to manage user tokens:

```python
# integrations/services.py
from datetime import timedelta
from django.utils import timezone
from integrations.models import OAuthToken


def get_user_token(user, provider: str) -> str:
    """Get a valid access token for a user, refreshing if expired."""
    try:
        token_obj = OAuthToken.objects.get(user=user, provider=provider)
    except OAuthToken.DoesNotExist:
        raise ValueError(f"No OAuth token found for {provider}")

    if token_obj.is_expired and token_obj.refresh_token:
        token_obj.access_token = refresh_oauth_token(token_obj.refresh_token, provider)
        token_obj.expires_at = timezone.now() + timedelta(seconds=3600)
        token_obj.save()

    return token_obj.access_token


def refresh_oauth_token(refresh_token: str, provider: str) -> str:
    """Refresh an OAuth token with the provider."""
    # Implementation depends on provider
    # Return the new access_token
    pass


def store_oauth_token(user, provider: str, access_token: str, refresh_token: str = "", expires_in: int = 3600):
    """Store or update an OAuth token."""
    expires_at = timezone.now() + timedelta(seconds=expires_in)
    OAuthToken.objects.update_or_create(
        user=user,
        provider=provider,
        defaults={
            "access_token": access_token,
            "refresh_token": refresh_token,
            "expires_at": expires_at,
        }
    )
```

## Token Refresh Task

Create `integrations/tasks/token_refresh_task.py`:

```python
from celery import shared_task
from django.utils import timezone
from integrations.models import OAuthToken
from integrations.services import refresh_oauth_token


@shared_task
def refresh_expired_tokens():
    """Refresh tokens that are about to expire."""
    from datetime import timedelta

    # Find tokens expiring in the next hour
    expiring_soon = OAuthToken.objects.filter(
        expires_at__lte=timezone.now() + timedelta(hours=1),
        expires_at__gt=timezone.now(),
        refresh_token__isnull=False,
        refresh_token__gt="",
    )

    for token_obj in expiring_soon:
        try:
            new_token = refresh_oauth_token(token_obj.refresh_token, token_obj.provider)
            token_obj.access_token = new_token
            token_obj.expires_at = timezone.now() + timedelta(seconds=3600)
            token_obj.save()
        except Exception as exc:
            # Log and continue
            print(f"Failed to refresh token for {token_obj.user}: {exc}")
```

Configure periodic task in `settings.py`:

```python
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    "refresh-oauth-tokens": {
        "task": "integrations.tasks.token_refresh_task.refresh_expired_tokens",
        "schedule": crontab(minute=0, hour="*/6"),  # Every 6 hours
    },
}
```

## Security Best Practices

- **Never commit secrets** — Use `django-environ` or `python-decouple`
- **Keep `.env.example`** — With placeholder keys
- **Use Django signing** — For sensitive tokens stored in DB (`django.core.signing`)
- **Scope API keys** — To minimum required permissions
- **Rotate tokens** — Regularly rotate long-lived keys
- **Use HTTPS only** — For all external API calls
- **Log securely** — Never log full tokens, only last 4 characters

## Admin Customization

Create `integrations/admin/OAuthTokenAdmin.py`:

```python
from django.contrib import admin
from integrations.models import OAuthToken


@admin.register(OAuthToken)
class OAuthTokenAdmin(admin.ModelAdmin):
    list_display = ("user", "provider", "is_expired", "created_at")
    list_filter = ("provider", "created_at")
    search_fields = ("user__username", "provider")
    readonly_fields = ("created_at", "updated_at", "access_token")

    def is_expired(self, obj):
        return obj.is_expired
    is_expired.boolean = True
```
