---
name: base-integration-client
description: HTTP client wrapper for external APIs with retries, timeouts, and auth injection in a modular clients/ folder.
---

# Base Integration Client

## Folder Structure

Place all client code in `clients/`:

```text
integrations/clients/
├── __init__.py
├── base.py              # BaseServiceClient and AsyncBaseServiceClient
├── github.py            # GitHub API client
├── stripe.py            # Stripe API client
├── sendgrid.py          # SendGrid API client
└── slack.py             # Slack API client
```

## Base Client — Synchronous

Create `integrations/clients/base.py`:

```python
import logging
import time
from typing import Any
from abc import ABC, abstractmethod

import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

logger = logging.getLogger(__name__)


class ServiceClientError(Exception):
    """Raised when an external service call fails."""
    def __init__(self, service: str, message: str, status_code: int | None = None):
        self.service = service
        self.status_code = status_code
        super().__init__(f"[{service}] {message}")


class BaseServiceClient(ABC):
    """Base client for third-party API integrations."""

    SERVICE_NAME: str = ""
    TIMEOUT: int = 30
    MAX_RETRIES: int = 3
    RETRY_BACKOFF: float = 0.5

    def __init__(self):
        self._session: requests.Session | None = None

    @abstractmethod
    def get_base_url(self) -> str:
        """Return the base URL for this service."""
        raise NotImplementedError("Subclasses must implement get_base_url()")

    @abstractmethod
    def get_auth_headers(self) -> dict[str, str]:
        """Return authentication headers."""
        raise NotImplementedError("Subclasses must implement get_auth_headers()")

    @property
    def session(self) -> requests.Session:
        if self._session is None:
            self._session = requests.Session()
            self._session.headers.update(self.get_auth_headers())
            self._session.headers.update({"Accept": "application/json"})
        return self._session

    def _make_request(self, method: str, path: str, **kwargs) -> Any:
        """Execute HTTP request with retry logic."""
        url = f"{self.get_base_url()}{path}"
        last_exc = None

        for attempt in range(1, self.MAX_RETRIES + 1):
            try:
                resp = self.session.request(method, url, timeout=self.TIMEOUT, **kwargs)
                resp.raise_for_status()
                return resp.json() if resp.content else None
            except requests.HTTPError as exc:
                if exc.response.status_code == 429:
                    retry_after = int(exc.response.headers.get("Retry-After", self.RETRY_BACKOFF * (2 ** attempt)))
                    logger.warning(f"{self.SERVICE_NAME} rate-limited, retry in {retry_after}s")
                    time.sleep(retry_after)
                    last_exc = exc
                    continue
                if exc.response.status_code >= 500:
                    logger.warning(f"{self.SERVICE_NAME} server error {exc.response.status_code}, retry {attempt}/{self.MAX_RETRIES}")
                    time.sleep(self.RETRY_BACKOFF * (2 ** attempt))
                    last_exc = exc
                    continue
                raise ServiceClientError(self.SERVICE_NAME, str(exc), exc.response.status_code) from exc
            except requests.RequestException as exc:
                logger.warning(f"{self.SERVICE_NAME} request error, retry {attempt}/{self.MAX_RETRIES}")
                time.sleep(self.RETRY_BACKOFF * (2 ** attempt))
                last_exc = exc

        raise ServiceClientError(
            self.SERVICE_NAME,
            f"Failed after {self.MAX_RETRIES} retries: {last_exc}",
            getattr(getattr(last_exc, 'response', None), 'status_code', None)
        )

    def get(self, path: str, **kwargs) -> Any:
        return self._make_request("GET", path, **kwargs)

    def post(self, path: str, **kwargs) -> Any:
        return self._make_request("POST", path, **kwargs)

    def put(self, path: str, **kwargs) -> Any:
        return self._make_request("PUT", path, **kwargs)

    def patch(self, path: str, **kwargs) -> Any:
        return self._make_request("PATCH", path, **kwargs)

    def delete(self, path: str, **kwargs) -> Any:
        return self._make_request("DELETE", path, **kwargs)

    def close(self):
        if self._session:
            self._session.close()
```

## Usage Example

Create `integrations/clients/github.py`:

```python
from django.conf import settings
from integrations.clients.base import BaseServiceClient


class GitHubClient(BaseServiceClient):
    SERVICE_NAME = "GitHub"
    TIMEOUT = 15

    def get_base_url(self) -> str:
        return "https://api.github.com"

    def get_auth_headers(self) -> dict[str, str]:
        return {"Authorization": f"Bearer {settings.GITHUB_TOKEN}"}

    def get_user(self, username: str) -> dict:
        return self.get(f"/users/{username}")

    def list_repos(self, username: str) -> list[dict]:
        return self.get(f"/users/{username}/repos")
```

Instantiate in views:

```python
from integrations.clients.github import GitHubClient

client = GitHubClient()
user = client.get_user("octocat")
client.close()
```

## Async Variant

For ASGI deployments, create `integrations/clients/base.py` with async support using `httpx`:

```python
import httpx
from abc import ABC, abstractmethod


class AsyncBaseServiceClient(ABC):
    SERVICE_NAME: str = ""
    TIMEOUT: int = 30
    MAX_RETRIES: int = 3
    RETRY_BACKOFF: float = 0.5

    def __init__(self):
        self._client: httpx.AsyncClient | None = None

    @abstractmethod
    def get_base_url(self) -> str:
        raise NotImplementedError("Subclasses must implement get_base_url()")

    @abstractmethod
    def get_auth_headers(self) -> dict[str, str]:
        raise NotImplementedError("Subclasses must implement get_auth_headers()")

    @property
    def client(self) -> httpx.AsyncClient:
        if self._client is None or self._client.is_closed:
            self._client = httpx.AsyncClient(
                base_url=self.get_base_url(),
                timeout=self.TIMEOUT,
                headers=self.get_auth_headers(),
            )
        return self._client

    async def _make_request(self, method: str, path: str, **kwargs):
        import asyncio
        last_exc = None
        for attempt in range(1, self.MAX_RETRIES + 1):
            try:
                resp = await self.client.request(method, path, **kwargs)
                resp.raise_for_status()
                return resp.json() if resp.content else None
            except httpx.HTTPStatusError as exc:
                if exc.response.status_code in (429, 500, 502, 503, 504):
                    retry_after = int(exc.response.headers.get("Retry-After", self.RETRY_BACKOFF * (2 ** attempt)))
                    await asyncio.sleep(retry_after)
                    last_exc = exc
                    continue
                raise ServiceClientError(self.SERVICE_NAME, str(exc), exc.response.status_code) from exc
            except httpx.RequestError as exc:
                await asyncio.sleep(self.RETRY_BACKOFF * (2 ** attempt))
                last_exc = exc
        raise ServiceClientError(self.SERVICE_NAME, f"Failed after {self.MAX_RETRIES} retries: {last_exc}")

    async def get(self, path, **kw):    return await self._make_request("GET", path, **kw)
    async def post(self, path, **kw):   return await self._make_request("POST", path, **kw)
    async def put(self, path, **kw):    return await self._make_request("PUT", path, **kw)
    async def patch(self, path, **kw):  return await self._make_request("PATCH", path, **kw)
    async def delete(self, path, **kw): return await self._make_request("DELETE", path, **kw)

    async def aclose(self):
        if self._client and not self._client.is_closed:
            await self._client.aclose()
```

## Testing

Create `integrations/tests/test_clients.py`:

```python
import responses
from django.test import TestCase
from integrations.clients.github import GitHubClient


class TestGitHubClient(TestCase):
    @responses.activate
    def test_get_user(self):
        responses.add(
            responses.GET,
            "https://api.github.com/users/octocat",
            json={"login": "octocat", "id": 123},
            status=200,
        )
        client = GitHubClient()
        user = client.get_user("octocat")
        self.assertEqual(user["login"], "octocat")
```

## Integration with Models

Store client instances in service classes for dependency injection:

```python
# integrations/services.py
from integrations.clients.github import GitHubClient


class GitHubService:
    def __init__(self):
        self.client = GitHubClient()

    def get_user_info(self, username: str) -> dict:
        return self.client.get_user(username)

    def close(self):
        self.client.close()
```
