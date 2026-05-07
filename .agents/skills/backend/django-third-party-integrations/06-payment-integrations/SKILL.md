---
name: payment-integrations
description: Stripe payment integration with dj-stripe models, webhooks, and tasks in a modular structure.
---

# Payment Integrations — Stripe

## Folder Structure

```text
integrations/
├── models/
│   └── StripeCustomer.py
├── tasks/
│   └── process_stripe_webhook_task.py
├── admin/
│   └── StripeCustomerAdmin.py
├── api/v1/
│   └── views/
│       └── StripeWebhookView.py
└── tests/
    └── test_stripe_checkout.py
```

## Setup dj-stripe

Install and configure the library:

```bash
pip install dj-stripe
```

Add to `settings.py`:

```python
INSTALLED_APPS = [
    ...
    "djstripe",
]

STRIPE_LIVE_SECRET_KEY = env("STRIPE_LIVE_SECRET_KEY")
STRIPE_TEST_SECRET_KEY = env("STRIPE_TEST_SECRET_KEY")
STRIPE_LIVE_MODE = env.bool("STRIPE_LIVE_MODE", default=False)
DJSTRIPE_WEBHOOK_SECRET = env("DJSTRIPE_WEBHOOK_SECRET")

# Optional: foreign key to a custom user model
DJSTRIPE_USE_NATIVE_JSONFIELD = True
```

Run migrations:

```bash
python manage.py migrate djstripe
```

## Extend dj-stripe Models

Create `integrations/models/StripeCustomer.py`:

```python
from django.db import models
from djstripe.models import Customer


class StripeCustomer(Customer):
    """Extended Stripe customer with app-specific fields."""

    user = models.OneToOneField("auth.User", on_delete=models.CASCADE, related_name="stripe_customer")
    preferred_card = models.CharField(max_length=255, blank=True)
    billing_email = models.EmailField(default="")

    class Meta:
        proxy = False

    def get_active_subscriptions(self):
        """Get all active subscriptions for this customer."""
        return self.subscriptions.filter(status="active")

    def get_active_charges(self):
        """Get recent successful charges."""
        return self.charges.filter(status="succeeded").order_by("-created")[:10]

    def has_valid_payment_method(self) -> bool:
        """Check if customer has at least one valid payment method."""
        return self.sources.filter(deleted=False).exists()
```

Update `integrations/models/__init__.py`:

```python
from integrations.models.OAuthToken import OAuthToken
from integrations.models.StripeCustomer import StripeCustomer

__all__ = ["OAuthToken", "StripeCustomer"]
```

## Checkout Flow

Create a view to start checkout:

```python
# integrations/api/v1/views/CheckoutView.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
import stripe
from django.conf import settings


class CheckoutSessionView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request):
        price_id = request.data.get("price_id")
        success_url = request.data.get("success_url")
        cancel_url = request.data.get("cancel_url")

        session = stripe.checkout.Session.create(
            customer_email=request.user.email,
            payment_method_types=["card"],
            mode="payment",
            line_items=[{"price": price_id, "quantity": 1}],
            success_url=success_url,
            cancel_url=cancel_url,
        )
        return Response({"session_id": session.id, "client_secret": session.client_secret})
```

## Subscription Management

```python
# integrations/services.py
from djstripe.models import Customer, Subscription
from integrations.models import StripeCustomer
import stripe


def create_customer(user):
    """Create a Stripe customer for a user."""
    stripe_customer = StripeCustomer.objects.create(
        user=user,
        billing_email=user.email,
        id=user.id,  # Use Django user ID as Stripe customer ID
    )
    return stripe_customer


def create_subscription(user, price_id: str):
    """Create a subscription for a user."""
    stripe_customer = StripeCustomer.objects.get(user=user)
    subscription = stripe.Subscription.create(
        customer=stripe_customer.id,
        items=[{"price": price_id}],
    )
    return subscription


def cancel_subscription(subscription_id: str, immediately: bool = False):
    """Cancel a user's subscription."""
    if immediately:
        return stripe.Subscription.delete(subscription_id)
    else:
        return stripe.Subscription.modify(
            subscription_id,
            cancel_at_period_end=True
        )
```

## Webhook Processing

Create `integrations/tasks/process_stripe_webhook_task.py`:

```python
from celery import shared_task
from django.utils import timezone
from djstripe.models import Event
from integrations.models import StripeCustomer


@shared_task
def process_stripe_event(event_id: str):
    """Process a Stripe webhook event."""
    try:
        event = Event.objects.get(id=event_id)
    except Event.DoesNotExist:
        return {"error": "Event not found"}

    event_type = event.type
    data = event.data["object"]

    if event_type == "checkout.session.completed":
        handle_checkout_completed(data)
    elif event_type == "customer.subscription.created":
        handle_subscription_created(data)
    elif event_type == "customer.subscription.deleted":
        handle_subscription_deleted(data)
    elif event_type == "invoice.payment_succeeded":
        handle_payment_succeeded(data)
    elif event_type == "invoice.payment_failed":
        handle_payment_failed(data)

    return {"status": "processed", "type": event_type}


def handle_checkout_completed(data):
    """Handle successful checkout."""
    session_id = data.get("id")
    customer_id = data.get("customer")
    # Store order/payment record
    print(f"Checkout completed: {session_id}")


def handle_subscription_created(data):
    """Handle new subscription."""
    subscription_id = data.get("id")
    customer_id = data.get("customer")
    print(f"Subscription created: {subscription_id}")


def handle_subscription_deleted(data):
    """Handle subscription cancellation."""
    subscription_id = data.get("id")
    print(f"Subscription deleted: {subscription_id}")


def handle_payment_succeeded(data):
    """Handle successful payment."""
    invoice_id = data.get("id")
    amount = data.get("amount_paid")
    print(f"Payment succeeded: {invoice_id} - ${amount}")


def handle_payment_failed(data):
    """Handle failed payment."""
    invoice_id = data.get("id")
    print(f"Payment failed: {invoice_id}")
```

dj-stripe automatically handles webhooks and creates `Event` objects. Just register signal receivers:

```python
# integrations/signals.py
from django.dispatch import receiver
from djstripe import webhooks
from integrations.tasks.process_stripe_webhook_task import process_stripe_event


@receiver(webhooks.webhook_received)
def on_stripe_webhook(sender, event, **kwargs):
    """Process Stripe webhooks asynchronously."""
    process_stripe_event.delay(event.id)
```

Wire up signals in `integrations/apps.py`:

```python
from django.apps import AppConfig


class IntegrationsConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "integrations"

    def ready(self):
        import integrations.signals
```

## Admin Customization

Create `integrations/admin/StripeCustomerAdmin.py`:

```python
from django.contrib import admin
from integrations.models import StripeCustomer


@admin.register(StripeCustomer)
class StripeCustomerAdmin(admin.ModelAdmin):
    list_display = ("user", "billing_email", "has_valid_payment_method", "created")
    list_filter = ("created",)
    search_fields = ("user__username", "billing_email")
    readonly_fields = ("id", "created")

    def has_valid_payment_method(self, obj):
        return obj.has_valid_payment_method()
    has_valid_payment_method.boolean = True
```

## Testing

Create `integrations/tests/test_stripe_checkout.py`:

```python
from django.test import TestCase
from django.contrib.auth.models import User
from integrations.models import StripeCustomer


class TestStripeCheckout(TestCase):
    def setUp(self):
        self.user = User.objects.create_user("testuser", "test@example.com", "password")

    def test_create_stripe_customer(self):
        customer = StripeCustomer.objects.create(
            user=self.user,
            billing_email=self.user.email,
            id=str(self.user.id),
        )
        self.assertEqual(customer.billing_email, self.user.email)
```

## Error Handling

```python
# integrations/exceptions.py
from rest_framework.views import exception_handler as drf_exception_handler
from rest_framework.response import Response
import stripe


def stripe_exception_handler(exc, context):
    """Handle Stripe-specific errors."""
    if isinstance(exc, stripe.error.StripeError):
        return Response(
            {"detail": "Payment processing failed. Please try again."},
            status=402,
        )
    return drf_exception_handler(exc, context)
```

Configure in `settings.py`:

```python
REST_FRAMEWORK = {
    "EXCEPTION_HANDLER": "integrations.exceptions.stripe_exception_handler",
}
```
