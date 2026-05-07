---
name: django-third-party-integrations
description: >-
  Build third-party API integrations in Django/DRF with modular, organized folder structure — service clients, OAuth/API-key auth, token refresh, retries, rate-limiting, webhooks, and secret management. Use when integrating external APIs (Stripe, Twilio, SendGrid, Slack, AWS, etc.), adding webhooks, implementing OAuth flows, handling API authentication, or building resilience patterns.
---

# Django Third-Party Integrations

Guide for integrating external services into Django/DRF projects with a clean, modular folder structure.

## Modular Folder Structure

Create an `integrations/` Django app with the following structure:

```text
integrations/
├── __init__.py
├── apps.py
├── admin/
│   ├── __init__.py
│   ├── OAuthTokenAdmin.py
│   └── WebhookEventAdmin.py
├── filters/
│   ├── __init__.py
│   └── WebhookEventFilter.py
├── tasks/
│   ├── __init__.py
│   ├── sync_webhook_task.py
│   ├── token_refresh_task.py
│   └── retry_failed_requests_task.py
├── models/
│   ├── __init__.py
│   ├── OAuthToken.py
│   ├── WebhookEvent.py
│   └── IntegrationLog.py
├── api/
│   ├── __init__.py
│   └── v1/
│       ├── __init__.py
│       ├── urls.py
│       ├── views/
│       │   ├── __init__.py
│       │   ├── WebhookReceiveView.py
│       │   └── IntegrationStatusView.py
│       └── serializers/
│           ├── __init__.py
│           ├── OAuthTokenSerializer.py
│           ├── WebhookEventSerializer.py
│           └── IntegrationLogSerializer.py
├── clients/
│   ├── __init__.py
│   ├── base.py
│   ├── github.py
│   ├── stripe.py
│   ├── sendgrid.py
│   └── slack.py
├── tests/
│   ├── __init__.py
│   ├── test_clients.py
│   ├── test_webhooks.py
│   └── test_oauth_flow.py
└── migrations/
    └── __init__.py
```

## Key Responsibilities

- **`clients/`**: HTTP client wrappers (`base.py`, provider-specific clients)
- **`models/`**: ORM models for OAuth tokens, webhook logs, integration state
- **`api/v1/`**: DRF views and serializers for webhook receivers and status endpoints
- **`tasks/`**: Celery background tasks for webhook processing, token refresh, retries
- **`admin/`**: Django admin customizations for managing integrations
- **`filters/`**: Queryset filters for searching webhook events and logs
- **`tests/`**: Test files for clients, webhooks, OAuth flows

## Subskills

Read the relevant subskill when the task narrows:

- **[01-base-integration-client](01-base-integration-client/SKILL.md)** — HTTP client wrapper with retries and auth injection
- **[02-authentication-system](02-authentication-system/SKILL.md)** — OAuth 2.0 and API key patterns with models
- **[03-sendgrid](03-sendgrid/SKILL.md)** — Email via SendGrid
- **[04-twilio](04-twilio/SKILL.md)** — SMS via Twilio
- **[05-slack](05-slack/SKILL.md)** — Messaging via Slack webhooks
- **[06-payment-integrations](06-payment-integrations/SKILL.md)** — Stripe with dj-stripe models and webhooks
- **[07-resilience-patterns](07-resilience-patterns/SKILL.md)** — Circuit breaker, idempotency, DRF exception handling
- **[08-webhook-system](08-webhook-system/SKILL.md)** — Receiving, validating, logging, and processing webhooks

## Getting Started

1. Create the `integrations/` app: `python manage.py startapp integrations`
2. Follow the folder structure above
3. Start with **01-base-integration-client** to create your first client
4. Add authentication patterns from **02-authentication-system**
5. Implement webhooks with **06-webhook-system**
