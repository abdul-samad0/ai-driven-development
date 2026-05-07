---
name: sendgrid-email
description: Email integration using Django's email backend with SendGrid, async Celery tasks, and an EmailLog model for audit records.
---

# SendGrid Email Integration

Use Django's built-in email system with `django-sendgrid-v5` as the backend. Every email is sent asynchronously via a Celery task and recorded in an `EmailLog` model.

## Package

Add to `config/requirements/stage.in` and `config/requirements/prod.in`:

```
django-sendgrid-v5
```

Then compile and install:

```bash
make cr
```

## Settings

Create `config/sendgrid.py`:

```python
from decouple import config

DEFAULT_FROM_EMAIL = config("DEFAULT_FROM_EMAIL")
SENDGRID_API_KEY = config("SENDGRID_API_KEY")
SENDGRID_SANDBOX_MODE_IN_DEBUG = False
EMAIL_BACKEND = "sendgrid_backend.SendgridBackend"
```

Import it in both `config/settings/stage.py` and `config/settings/prod.py`:

```python
from config.sendgrid import *  # noqa
```

## EmailLog Model

Create `communications/models.py`:

```python
from django_extensions.db.models import TimeStampedModel


class EmailLog(TimeStampedModel):
    """Audit record for every outbound email."""

    subject = models.CharField(max_length=255)
    to_email = models.JSONField()           # list of recipient addresses
    from_email = models.EmailField()
    body = models.TextField()
    status = models.CharField(
        max_length=20,
        choices=[("sent", "Sent"), ("failed", "Failed")],
        default="sent",
    )
    error = models.TextField(blank=True, default="")

    class Meta:
        verbose_name = "Email Log"
        verbose_name_plural = "Email Logs"

    def __str__(self):
        return f"{self.subject} → {self.to_email} ({self.status})"
```

Run migrations:

```bash
python manage.py makemigrations communications
python manage.py migrate
```

## send_email Helper

Create `communications/email.py`:

```python
from logging import getLogger

from django.conf import settings
from django.core.mail import EmailMessage

from communications.models import EmailLog


logger = getLogger(__name__)


def send_email(
    email_subject,
    email_message,
    to_email,
    from_email=settings.DEFAULT_FROM_EMAIL,
    reply_to=None,
):
    email = EmailMessage(
        subject=email_subject,
        body=email_message,
        from_email=from_email,
        to=to_email,
        reply_to=reply_to,
    )
    email.content_subtype = "html"

    status, error = "sent", ""
    try:
        email.send()
    except Exception as e:
        logger.exception(f"Failed to send email to {to_email}.", extra={"error": e})
        status, error = "failed", str(e)

    EmailLog.objects.create(
        subject=email_subject,
        to_email=to_email,
        from_email=from_email,
        body=email_message,
        status=status,
        error=error,
    )
```

## Celery Task

Create `communications/tasks.py`:

```python
from celery import shared_task
from django.template.loader import render_to_string
from decouple import config

from communications.email import send_email


@shared_task
def send_invitation_email(user_id, invited_by, organization_id, uuid):
    """Send a single invitation email. Called once per invite."""
    from django.contrib.auth import get_user_model
    from organizations.models import Organization

    User = get_user_model()
    organization = Organization.objects.only("name").get(pk=organization_id)
    user = User.objects.only("email", "first_name").get(pk=user_id)

    context = {
        "invitation_url": f"{config('INVITATION_URL')}{uuid}",
        "invited_by": invited_by,
        "user_name": user.first_name,
        "organization": organization.name,
    }
    email_subject = "Invitation to join organization"
    email_message = render_to_string("organizations/organization_invite.html", context)
    send_email(email_subject, email_message, [user.email])
```

## Dispatching Multiple Emails

Send each email as a separate Celery task — never batch into one task:

```python
for invite in invites:
    send_invitation_email.delay(
        user_id=invite.user_id,
        invited_by=invite.invited_by,
        organization_id=invite.organization_id,
        uuid=invite.uuid,
    )
```

## .env Variables

```env
DEFAULT_FROM_EMAIL=noreply@yourdomain.com
SENDGRID_API_KEY=SG.xxxxxxxxxxxxxxxx
INVITATION_URL=https://yourdomain.com/invitations/
```

## Folder Structure

```text
communications/
├── __init__.py
├── apps.py
├── email.py          ← send_email helper + EmailLog write
├── models.py         ← EmailLog model
├── tasks.py          ← Celery tasks
├── admin.py          ← register EmailLog in Django admin
└── migrations/
config/
└── sendgrid.py       ← EMAIL_BACKEND + API key settings
```

## Admin Registration

```python
# communications/admin.py
from django.contrib import admin
from communications.models import EmailLog


@admin.register(EmailLog)
class EmailLogAdmin(admin.ModelAdmin):
    list_display = ("subject", "to_email", "status", "sent_at")
    list_filter = ("status", "sent_at")
    search_fields = ("subject", "to_email", "from_email")
    readonly_fields = ("subject", "to_email", "from_email", "body", "status", "error", "sent_at")
```

## Testing

```python
# communications/tests/test_email.py
from unittest.mock import patch
from django.test import TestCase
from communications.email import send_email
from communications.models import EmailLog


class TestSendEmail(TestCase):
    @patch("django.core.mail.EmailMessage.send")
    def test_successful_send_logs_record(self, mock_send):
        mock_send.return_value = 1
        send_email("Hello", "<p>Hi</p>", ["user@example.com"])
        log = EmailLog.objects.get()
        self.assertEqual(log.status, "sent")
        self.assertEqual(log.subject, "Hello")

    @patch("django.core.mail.EmailMessage.send", side_effect=Exception("SMTP error"))
    def test_failed_send_logs_error(self, mock_send):
        send_email("Hello", "<p>Hi</p>", ["user@example.com"])
        log = EmailLog.objects.get()
        self.assertEqual(log.status, "failed")
        self.assertIn("SMTP error", log.error)
```
