---
name: schema-design
description: "Rules for designing Django model schemas. TRIGGER when: creating a new model, adding fields, defining relationships, choosing field types or defaults, or setting up a new Django app. DO NOT TRIGGER for: migration writing, serializer design, or API schema design."
---

# Schema Design

Rules and conventions for designing Django model schemas. Covers field ordering, base classes, relationships, field types, and naming.

---

## Defaults

- Always inherit from `TimeStampedModel` or `SoftDeleteBaseModel` — never bare `models.Model`.
- Always provide `related_name` on `ForeignKey` and `OneToOneField`.
- Always use string-based references for cross-app relations: `"app_label.ModelName"`.
- Define choices in `choices.py` using `TextChoices` — never inline, never `IntegerChoices`.
- JSONField defaults must be callable — never a dict or list literal.
- Optional fields always use both `null=True, blank=True` together. Boolean fields are never nullable.

---

## Rules

### 1. Field Ordering

Group fields by type (alphabetically by Django class name), then alphabetically within each group. After all regular fields: one blank line, then relational fields in order: `OneToOneField` → `ForeignKey` → `ManyToManyField` (each group alphabetical, no blank lines between groups).

**Type order:** `BooleanField` → `CharField` → `DateTimeField` → `DecimalField` → `EmailField` → `FSMField` → `IntegerField` → `JSONField` → `PositiveIntegerField`/`PositiveSmallIntegerField` → `TextField` → `URLField` → `UUIDField`

```python
class MySettings(SoftDeleteBaseModel):
    # BooleanFields (alphabetical)
    feature_one_enabled = models.BooleanField(default=False)
    feature_two_enabled = models.BooleanField(default=True)
    # CharFields (alphabetical)
    domain = models.CharField(max_length=255)
    status = models.CharField(max_length=12, choices=StatusChoices.choices, default=StatusChoices.ACTIVE)
    # JSONFields (alphabetical)
    metadata = models.JSONField(default=dict)
    settings = models.JSONField(default=default_settings)
    # TextFields (alphabetical)
    description = models.TextField(null=True, blank=True)
    # URLFields (alphabetical)
    allowed_host = models.URLField(null=True, blank=True)

    # OneToOneFields (alphabetical)
    icon = models.OneToOneField("myapp.Image", related_name="icon", on_delete=models.SET_NULL, null=True, blank=True)
    parent = models.OneToOneField("myapp.ParentModel", related_name="settings", on_delete=models.CASCADE)
    # ForeignKeys (alphabetical)
    owner = models.ForeignKey("users.User", related_name="settings", null=True, blank=True, on_delete=models.SET_NULL)
```

---

### 2. Base Classes

| Need | Base Class | Source |
|------|-----------|--------|
| Timestamps only | `TimeStampedModel` | `django_extensions.db.models` |
| Timestamps + soft delete | `SoftDeleteBaseModel` | your core models module |

```python
from django_extensions.db.models import TimeStampedModel

class MyModel(TimeStampedModel):
    name = models.CharField(max_length=255)
```

```python
from myapp.core.models import SoftDeleteBaseModel

class MySettings(SoftDeleteBaseModel):
    domain = models.CharField(max_length=255)
```

---

### 3. Choices and Enums

Define in `choices.py`, use `TextChoices`, values in `UPPER_CASE`:

```python
# choices.py
from django.db.models import TextChoices

class StatusChoices(TextChoices):
    PENDING = "PENDING", "Pending"
    COMPLETED = "COMPLETED", "Completed"
    CANCELLED = "CANCELLED", "Cancelled"
```

```python
# models.py
status = models.CharField(
    max_length=12,
    choices=StatusChoices.choices,
    default=StatusChoices.PENDING,
)
```

`max_length` must fit the longest choice value.

---

### 4. State Machine Fields (FSM)

Use `FSMField` from `django-fsm` for any status field with guarded transitions:

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

    @transition(field=status, source="*", target=StatusChoices.DELETED)
    def delete_object(self): pass
```

---

### 5. Foreign Key Relationships

```python
# Cross-app — always use string reference
owner = models.ForeignKey("users.User", related_name="owned_items", on_delete=models.CASCADE)
category = models.ForeignKey("myapp.Category", related_name="items", on_delete=models.CASCADE)
```

Always provide `related_name`. Use a name that reads naturally from the **target model's perspective**.

---

### 6. OneToOneField for Settings/Configuration

```python
# Required config
parent = models.OneToOneField("myapp.ParentModel", related_name="settings", on_delete=models.CASCADE)

# Optional asset (e.g. image)
cover_image = models.OneToOneField(
    "myapp.Image", related_name="settings_cover",
    on_delete=models.SET_NULL, null=True, blank=True,
)
```

---

### 7. JSONField

| Use case | Pattern |
|---------|---------|
| Translatable content | `models.JSONField(default=default_label)` |
| Feature flags | `models.JSONField(default=default_feature_flags)` |
| Structured response | `models.JSONField(default=dict)` |
| Optional metadata | `models.JSONField(null=True, blank=True)` |

Default must always be a **callable**, never `default={}` or `default=[]`.

---

### 8. Nullable Field Convention

```python
# Optional — always both null and blank
notes = models.TextField(null=True, blank=True)

# Required — omit both
name = models.CharField(max_length=255)

# Boolean — never null, always has a default
is_active = models.BooleanField(default=True)
```

---

### 9. File and Image Fields

```python
# Date-based upload paths
document = models.FileField(upload_to="documents/%Y/%m/")
avatar = models.ImageField(upload_to="avatars/%Y/%m/", null=True, blank=True)

# Restricted types
file = models.FileField(
    upload_to="uploads/%Y/%m/",
    validators=[FileExtensionValidator(allowed_extensions=["pdf"])],
)
```

---

### 10. DecimalField for Currency

```python
price = models.DecimalField(max_digits=22, decimal_places=2)
```

---

### 11. Model Meta

Set `verbose_name` and `verbose_name_plural` when the auto-generated name would be wrong or unclear — particularly for irregular plurals or multi-word model names:

```python
class Meta:
    verbose_name = "item category"
    verbose_name_plural = "item categories"
```

Always lowercase. Set `verbose_name_plural` explicitly when Django's auto-pluralisation would produce an incorrect result (e.g. `"category"` → `"categorys"`).

---

### 12. `__str__` Method

Every model must implement `__str__`:

```python
# Name-based
def __str__(self): return self.name

# Relational
def __str__(self): return f"{self.name} of {self.owner}"

# Config/settings
def __str__(self): return f"Settings for {self.parent}"

# Fallback
def __str__(self): return f"{self.id}"
```

---

### 13. `help_text`

Add `help_text` on any field whose purpose isn't immediately obvious:

```python
interval = models.PositiveSmallIntegerField(
    default=0,
    help_text="Interval in days. Set to 0 to disable.",
)
```

---

## Pitfalls

- **Don't scatter fields randomly** — follow the type-group + alphabetical ordering convention.
- **Don't define choices inline** — use a `TextChoices` class in `choices.py`.
- **Don't use `IntegerChoices`** — always use `TextChoices`.
- **Don't use `null=True` on BooleanField** — always provide a default.
- **Don't inherit from bare `models.Model`** — use `TimeStampedModel` or `SoftDeleteBaseModel`.
- **Don't omit `related_name`** on `ForeignKey` or `OneToOneField`.
- **Don't pass mutable defaults to JSONField** — use a callable.
- **Don't use `CharField` for status with lifecycle transitions** — use `FSMField`.
- **Don't forget `verbose_name_plural`** when the model name has an irregular plural.
