---
name: audit-metadata-fields
description: "Rules for Django model audit and metadata fields. TRIGGER when: adding timestamps, soft delete, lifecycle/FSM status, ordering, feature flags, or database indexes to a model. DO NOT TRIGGER for: API design, serializer fields, or non-Django projects."
---

# Audit & Metadata Fields

Rules for timestamps, lifecycle management, ordering, feature flags, and indexing patterns on Django models.

---

## Defaults

- Always inherit from `TimeStampedModel` — never add `created_at`/`updated_at` manually.
- Use `SoftDeleteBaseModel` for simple soft delete, `FSMField` for multi-state lifecycles.
- Always pass `update_fields` when saving a single changed field.
- For nullable date fields, always pair `null=True, blank=True` together.

---

## Rules

### 1. Timestamps — TimeStampedModel

Inherit from `TimeStampedModel` (from `django-extensions`). Provides `created` and `modified` automatically.

```python
from django_extensions.db.models import TimeStampedModel

class MyModel(TimeStampedModel):
    name = models.CharField(max_length=256)
    # `created` and `modified` are available automatically
```

> Do NOT use `created_at`, `updated_at`, `auto_now_add`, or `auto_now` directly.

---

### 2. Soft Delete — SoftDeleteBaseModel

Define once in your core models module and inherit where needed:

```python
class SoftDeleteBaseModel(TimeStampedModel):
    is_active = models.BooleanField(default=True)

    class Meta:
        abstract = True

    def delete(self, using=None, keep_parents=False):
        self.is_active = False
        self.save(update_fields=["is_active"])
```

Use for settings-type models that must not be physically removed. Always call `.delete()` on the **instance**, never on a queryset.

---

### 3. FSM Lifecycle — FSMField

Use `django-fsm` for models with multi-state transitions instead of a plain `CharField`:

```python
from django_fsm import FSMField, transition

class MyModel(TimeStampedModel):
    status = FSMField(
        max_length=10,
        choices=StatusChoices.choices,
        default=StatusChoices.DRAFT,
    )

    @transition(field=status, source=StatusChoices.DRAFT, target=StatusChoices.ACTIVE)
    def publish(self): pass

    @transition(field=status, source=StatusChoices.ACTIVE, target=StatusChoices.INACTIVE)
    def deactivate(self): pass

    @transition(field=status, source=StatusChoices.INACTIVE, target=StatusChoices.ACTIVE)
    def activate(self): pass

    @transition(field=status, source="*", target=StatusChoices.DELETED)
    def delete_object(self): pass

    def delete(self, using=None, keep_parents=False):
        self.delete_object()
        self.save()
```

Transition methods (`publish`, `activate`, `deactivate`, delete) are illustrative — define what fits your model's actual states.

---

### 4. Ordering

Add an `order` field when instances need explicit ordering:

```python
order = models.PositiveIntegerField(default=0)

class Meta:
    ordering = ["order"]
```

- Field name: `order` — not `position`, `rank`, or `index`
- Always pair with `Meta.ordering`
- Use `update_fields=["order"]` when reordering in bulk

---

### 5. Feature Flags

```python
feature_flags = models.JSONField(
    default=default_feature_flags,
    help_text="Feature flags to enable or disable features per record.",
)
```

Default must be a **callable** — never a dict literal.

---

### 6. Database Indexes

Index fields used in frequent filter queries:

```python
# Single field
is_active = models.BooleanField(default=True, db_index=True)

# Composite via Meta
class Meta:
    indexes = [
        models.Index(fields=["status", "created"]),
        models.Index(fields=["is_active", "modified"]),
    ]
```

- Index `status` and `is_active` on any model queried at scale
- Prefer `Meta.indexes` for composites over stacking `db_index=True`
- Don't index low-cardinality booleans in isolation on small tables

---

## Pitfalls

- **Don't add `created`/`modified` manually** — `TimeStampedModel` provides them.
- **Don't name timestamp fields `created_at`/`updated_at`** — standard names are `created` and `modified`.
- **Don't use `auto_now_add`/`auto_now` directly** — inherit from `TimeStampedModel`.
- **Don't use `CharField` for status with transitions** — use `FSMField`.
- **Don't call `.delete()` on a queryset for soft-deletable models** — call it on the instance.
- **Don't omit `update_fields`** when saving a single field change — risks overwriting concurrent changes.
- **Don't hard-delete FSM models** — transition to DELETED and save.
- **Don't use `position`, `rank`, or `index`** for ordering fields — use `order`.
- **Don't skip indexes on high-traffic filter fields** — causes full table scans at scale.
