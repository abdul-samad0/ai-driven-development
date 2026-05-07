# Reusable Query Logic

## Description

Patterns for model managers, custom querysets, view mixins, annotations, and batch operations. Covers how queries should be structured, optimized, and reused across views and serializers.

## When to Use

- Writing a new queryset for a view or serializer
- Adding computed fields or aggregations to a queryset
- Creating reusable query logic across multiple views
- Optimizing database queries with joins and prefetches
- Performing batch create/update operations
- Writing factory methods on models

---

## Rules

### 1. Custom Model Manager

Place custom managers in a **separate `managers.py`** file within the app. Assign on the model with `objects = MyManager()`.

```python
# managers.py
from django.contrib.auth.models import UserManager as DjangoUserManager

class UserManager(DjangoUserManager):
   use_in_migrations = True

   def _create_user(self, email, password, **extra_fields):
       if not email:
           raise ValueError("The given email must be set")
       email = self.normalize_email(email)
       user = self.model(email=email, **extra_fields)
       user.set_password(password)
       user.save(using=self._db)
       return user

   def create_user(self, email, password=None, **extra_fields):
       extra_fields.setdefault("is_staff", False)
       extra_fields.setdefault("is_superuser", False)
       return self._create_user(email, password, **extra_fields)

   def create_superuser(self, email, password=None, **extra_fields):
       extra_fields.setdefault("is_staff", True)
       extra_fields.setdefault("is_superuser", True)
       if extra_fields.get("is_staff") is not True:
           raise ValueError("Superuser must have is_staff=True.")
       if extra_fields.get("is_superuser") is not True:
           raise ValueError("Superuser must have is_superuser=True.")
       return self._create_user(email, password, **extra_fields)
```

```python
# models.py
class User(AbstractUser):
   objects = UserManager()
```

### 2. View Mixins for Complex Querysets

Extract complex querysets shared across multiple views into **mixin classes** in a `mixins.py` file. Each mixin defines `get_queryset()`:

```python
# mixins.py
class SampleQuerysetMixin:
   def get_queryset(self):
       urls_subquery = (
           SiteUrl.objects.filter(Sample=OuterRef("pk"))
           .values("Sample")
           .annotate(total_chars=Coalesce(Sum("character_count"), Value(0)))
           .values("total_chars")[:1]
       )
       return (
           Sample.objects.exclude(status=StatusChoices.DELETED)
           .order_by("-created")
           .select_related("setting", "setting__Sample_image", ...)
           .prefetch_related("setting__floating_messages", "spiders", ...)
           .annotate(
               url_char_count=Coalesce(Subquery(urls_subquery, output_field=IntegerField()), Value(0)),
           )
       )

class StickyButtonQuerysetMixin:
   def get_queryset(self):
       user_subquery = UserSubscription.objects.filter(
           user=OuterRef("owner"),
           status=UserSubscriptionStatus.ACTIVE,
           subscription__name__in=["PREMIUM", "UNLIMITED"],
       )
       return (
           SampleButton.objects.exclude(status=StatusChoices.DELETED)
           .select_related("privacy_policy_document", "owner", "support_email", "lead")
           .prefetch_related(...)
           .annotate(is_premium_user=Exists(user_subquery))
           .order_by("-created")
       )
```

### 3. select_related and prefetch_related

Use **`select_related`** for ForeignKey and OneToOneField joins (single-object relationships):

```python
.select_related(
   "setting",
   "setting__Sample_image",
   "setting__chat_icon",
   "setting__widget_icon",
   "setting__privacy_policy_document",
   "setting__support_email",
)
```

Use **`prefetch_related`** for reverse FK and many-to-many (multi-object relationships). Always use the `Prefetch` object when custom ordering or nested prefetching is needed:

```python
from django.db.models import Prefetch

.prefetch_related(
   "setting__floating_messages",
   "spiders",
   Prefetch(
       "chat_samples",
       queryset=SampleForm.objects.prefetch_related(
           Prefetch(
               "fields",
               queryset=SampleFormField.objects.prefetch_related(
                   "field_items__email_record"
               ).order_by("sorting_order"),
           )
       ),
   ),
)
```

**Key patterns**:
- Chain `__` for deep joins: `"setting__Sample_image"`
- Use `Prefetch` with a custom queryset for ordering: `.order_by("sorting_order")`
- Nest `Prefetch` objects for multi-level prefetching
- Use `select_related` in admin via `list_select_related = ("owner", "support_email")`

### 4. Subquery Annotations

Add computed fields to querysets using `Subquery` with `OuterRef`:

```python
from django.db.models import Count, Exists, F, IntegerField, OuterRef, Subquery, Sum, Value
from django.db.models.functions import Coalesce

# Count related objects
conversation_count_subquery = (
   SampleSession.objects.filter(Sample=OuterRef("pk"))
   .values("Sample")
   .annotate(count=Count("id"))
   .values("count")[:1]
)

# Sum related field values
urls_subquery = (
   SiteUrl.objects.filter(Sample=OuterRef("pk"))
   .values("Sample")
   .annotate(total_chars=Coalesce(Sum("character_count"), Value(0)))
   .values("total_chars")[:1]
)

# Latest related timestamp
last_seen_subquery = (
   SampleHistory.objects.filter(sample_session__Sample=OuterRef("pk"))
   .order_by("-created")
   .values("created")[:1]
)

# Existence check for boolean annotation
user_subquery = UserSubscription.objects.filter(
   user=OuterRef("owner"),
   status=UserSubscriptionStatus.ACTIVE,
)
.annotate(is_premium_user=Exists(user_subquery))
```

Always wrap Subquery results with `Coalesce(..., Value(0))` to handle NULL:

```python
.annotate(
   url_char_count=Coalesce(Subquery(urls_subquery, output_field=IntegerField()), Value(0)),
)
```

### 5. @classmethod Factory Methods

Use `@classmethod` for creating records with related objects or complex initialization logic:

```python
@classmethod
def create_user_subscription(cls, data):
   status = UserSubscriptionStatus.ACTIVE
   if data.get("trial_end") and data.get("trial_start"):
       status = UserSubscriptionStatus.ON_TRAIL
   user_subscription = cls.objects.create(
       subscription=data["subscription"],
       user=data["user"],
       ...
   )
   UserSubscriptionFeatureUsage.create_user_subscription_features_usage(
       user_subscription=user_subscription
   )
   return user_subscription

@classmethod
def record_conversation(cls, Sample, user_query, assistant_response, session_id, chat_type, ...):
   sample_session, _ = SampleSession.objects.get_or_create(
       Sample=Sample, session_id=session_id,
   )
   data = [
       {"role": SampleChoices.USER, "text": user_query, "sample_session": sample_session, ...},
       {"role": SampleChoices.ASSISTANT, "text": assistant_response, "sample_session": sample_session, ...},
   ]
   SampleHistory.objects.bulk_create([SampleHistory(**d) for d in data])
```

### 6. bulk_create for Batch Inserts

Always use `bulk_create` when inserting multiple records of the same type:

```python
Notification.objects.bulk_create(notifications)
FileSource.objects.bulk_create([FileSource(Sample=self, file=file) for file in files])
cls.objects.bulk_create(user_subscription_features_usage)
SampleHistory.objects.bulk_create(chat_histories)
FAQ.objects.bulk_create(faq_objects)
```

### 7. get_or_create for Idempotent Records

Use when a record may or may not already exist. Assign the boolean to `_` or `__`:

```python
sample_session, _ = SampleSession.objects.get_or_create(
   Sample=Sample, session_id=session_id,
)

email_record, __ = EmailRecord.objects.get_or_create(
   email=support_email, user=owner
)

spider, created = Spider.objects.get_or_create(Sample=Sample)
```

### 8. update_fields on save()

When updating specific fields, always pass `update_fields` to `save()` to avoid race conditions and unnecessary writes:

```python
self.save(update_fields=["is_active"])
self.save(update_fields=["status"])
self.save(update_fields=["name", "content"])
self.save(update_fields=["usage"])
self.save(update_fields=["stripe_subscription_id", "subscription", "start_date", "end_date", "status"])
```

### 9. transaction.atomic for Multi-Model Writes

Use `transaction.atomic()` when a single operation touches multiple models or multiple records:

```python
# As a decorator
@transaction.atomic
def update_Sample(self, site_urls, pdf_files):
   self.create_update_sources(site_urls, pdf_files)
   create_or_update_embeddings(self)

# As a context manager (preferred)
with transaction.atomic():
   FAQ.objects.bulk_create(faq_objects)

with transaction.atomic():
   Notification.objects.bulk_create(notifications)
```

### 10. Ordering Convention

Default ordering is by `-created` (newest first):

```python
.order_by("-created")
```

For ordered child records, use a `sorting_order` field:

```python
SampleFormField.objects.order_by("sorting_order")
```

### 11. Filtering Deleted Records

Always exclude deleted records rather than filtering for active ones:

```python
Sample.objects.exclude(status=StatusChoices.DELETED)
```

### 12. Complex Q Object Filters

Use `Q` objects for OR conditions and negation in complex filters:

```python
from django.db.models import Q

.exclude(
   Q(Sample_sample__Sample__status=StatusChoices.DELETED)
   | Q(Sample_sample__sample_button__status=StatusChoices.DELETED)
)
```

---

## Implementation Guidelines

1. Place custom managers in `managers.py`, view mixins in `mixins.py`.
2. For querysets used by more than one view, create a mixin class with `get_queryset()`.
3. Always use `select_related` for FK/OneToOne and `prefetch_related` for reverse FK/M2M.
4. Use `Prefetch` with custom querysets when ordering or nested prefetch is needed.
5. Wrap `Subquery` results with `Coalesce` to handle NULL values.
6. Use `bulk_create` instead of looped `create()` calls.
7. Always pass `update_fields` to `save()` when updating specific fields.
8. Wrap multi-model writes in `transaction.atomic()`.
9. Use `@classmethod` factory methods for complex record creation logic.
10. Order results by `-created` unless a specific ordering field exists.

---

## Common Mistakes to Avoid

1. **Do not use `.filter(is_active=True)` for FSM models.** Use `.exclude(status=StatusChoices.DELETED)`.
2. **Do not call save() without update_fields** when only changing specific fields.
3. **Do not use looped `.create()` or `.save()`** when inserting multiple records. Use `bulk_create`.
4. **Do not forget `Coalesce` on Subquery annotations.** NULL subquery results will propagate NULL.
5. **Do not put complex queryset logic directly in views.** Extract it into a mixin in `mixins.py`.
6. **Do not use `prefetch_related` for single-object FK/OneToOne.** Use `select_related` instead.
7. **Do not forget `output_field` on Subquery** when the type cannot be inferred.











