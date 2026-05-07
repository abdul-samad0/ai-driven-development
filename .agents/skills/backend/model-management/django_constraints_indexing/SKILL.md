# Constraints & Indexing

## Description

Rules for database constraints, indexing strategies, data integrity enforcement, and deletion behavior. Covers unique constraints, `on_delete` strategies, validators, and soft-delete patterns.

## When to Use

- Adding unique constraints to models
- Choosing `on_delete` behavior for ForeignKey/OneToOneField
- Deciding whether to add database indexes
- Adding field-level validators
- Ensuring data integrity at the database level

---

## Rules

### 1. UniqueConstraint (Preferred Over unique_together)

Use `models.UniqueConstraint` inside `Meta.constraints` — not the older `unique_together` syntax.

```python
class WidgetProLead(TimeStampedModel):
   name = models.CharField(max_length=255)
   widget = models.ForeignKey("widget_pro.WidgetPro", related_name="widget_samples", on_delete=models.CASCADE)

   class Meta:
       constraints = [
           models.UniqueConstraint(
               fields=["widget", "name"],
               name="unique_sample_name_per_widget",
           )
       ]
```

**Naming convention**: `unique_<field1>_<field2>_per_<parent>` or a descriptive snake_case name.

### 2. Field-Level unique=True

Use for fields that must be globally unique:

```python
email = models.EmailField(_("email address"), unique=True)
doc_id = models.UUIDField(default=uuid.uuid4, editable=False, unique=True)
```

### 3. on_delete Strategy by Relationship Type

Use three deletion strategies based on the relationship semantics:

#### CASCADE — Ownership relationships
When the parent is deleted, children are deleted too. Use when the child has no meaning without its parent.

```python
# Parent owns child records
Sample = models.ForeignKey("Sample.Sample", related_name="urls", on_delete=models.CASCADE)
Sample = models.OneToOneField("Sample.Sample", related_name="setting", on_delete=models.CASCADE)
user = models.ForeignKey("users.User", on_delete=models.CASCADE, related_name="notifications")
```

#### SET_NULL — Optional/decorative references
When the referenced object can be removed without affecting the referencing model. Always paired with `null=True, blank=True`.

```python
Sample_image = models.OneToOneField(
   "Sample.Image",
   related_name="Sample_theme_setting",
   on_delete=models.SET_NULL,
   null=True,
   blank=True,
)

reaction = models.ForeignKey(
   "core.Reaction", on_delete=models.SET_NULL, related_name="conversations", null=True, blank=True
)

support_email = models.ForeignKey("users.EmailRecord", null=True, blank=True, on_delete=models.SET_NULL)
```

#### PROTECT — Financial/critical data
Prevents deletion of referenced objects that are tied to billing or subscription data.

```python
subscription = models.ForeignKey(
   "subscription.Subscription",
   related_name="user_subscriptions",
   on_delete=models.PROTECT,
)

user = models.ForeignKey("users.User", related_name="subscriptions", on_delete=models.PROTECT)
feature = models.ForeignKey("subscription.SubscriptionFeature", on_delete=models.PROTECT)
```

**Decision guide**:

| Relationship type | on_delete | Example |
|---|---|---|
| Parent owns child | `CASCADE` | Sample → SiteUrl |
| Optional reference | `SET_NULL` | ThemeSetting → Image |
| Financial/billing | `PROTECT` | UserSubscription → Subscription |

### 4. Indexing Strategy

Do not add explicit `db_index=True` or `Meta.indexes` unless there is a measured performance problem. Rely on implicit indexes from:

- `unique=True` fields (automatically indexed)
- ForeignKey fields (automatically indexed by Django)
- `UniqueConstraint` (creates a unique index)

### 5. Field-Level Validators

Use validators sparingly — only for file extension restrictions and custom format validation:

```python
# File extension validation
file = models.FileField(
   upload_to="documents/%Y/%m/",
   validators=[FileExtensionValidator(allowed_extensions=["pdf"])],
)

image = models.FileField(
   upload_to="images/%Y/%m/",
   validators=[FileExtensionValidator(allowed_extensions=["jpg", "jpeg", "png", "gif", "svg"])],
)

# Custom validator for phone numbers
whatsapp_number = models.TextField(
   max_length=18, null=True, blank=True, validators=[validate_whatsapp_number]
)
```

Define custom validators in a `validators.py` file within the app.

FSM-based models use status transitions to `DELETED` instead:

```python
@transition(
   field=status,
   source=[SampleStatusChoices.ACTIVE, SampleStatusChoices.INACTIVE, SampleStatusChoices.DRAFT],
   target=SampleStatusChoices.DELETED,
)
def delete_Sample(self):
   pass

def delete(self, using=None, keep_parents=False):
   self.delete_Sample()
   self.save()
```

### 7. Queryset-Level Exclusion of Deleted Records

Always exclude deleted records rather than filtering for active ones:

```python
Sample.objects.exclude(status=StatusChoices.DELETED)
SampleButton.objects.exclude(status=StatusChoices.DELETED)

# Complex exclusion with Q objects
.exclude(
   Q(Sample_sample__Sample__status=StatusChoices.DELETED)
   | Q(Sample_sample__sample_button__status=StatusChoices.DELETED)
)
```

### 8. editable=False for System-Managed Fields

Fields that should never be modified after creation:

```python
doc_id = models.UUIDField(default=uuid.uuid4, editable=False, unique=True)
```

---

## Implementation Guidelines

1. When adding a new composite uniqueness constraint, use `UniqueConstraint` in `Meta.constraints` with a descriptive `name`.
2. Choose `on_delete` based on the relationship semantics — ownership (CASCADE), optional (SET_NULL), or financial (PROTECT).
3. Do not add database indexes proactively. Let implicit indexes from FK and unique fields handle it.
4. For file upload fields, always add `FileExtensionValidator` to restrict allowed types.
5. For soft-deletable models, use `SoftDeleteBaseModel` or FSM-based status transitions — never hard delete user-facing data.
6. Always filter out deleted records in querysets using `.exclude(status=StatusChoices.DELETED)`.

---

## Common Mistakes to Avoid

1. **Do not use `unique_together`** in Meta. Use `UniqueConstraint` in `Meta.constraints` instead.
2. **Do not use `on_delete=models.DO_NOTHING`**. It is never appropriate.
3. **Do not use `SET_NULL` without `null=True, blank=True`**. They must always appear together.
4. **Do not add `db_index=True`** without a proven performance need.
5. **Do not use `PROTECT` for non-financial relationships**. Reserve it for subscription/billing models.
6. **Do not hard-delete records** on models that use soft delete or FSM. Always use the model's `delete()` method or FSM transitions.
7. **Do not forget to exclude deleted records** in querysets. Always add `.exclude(status=StatusChoices.DELETED)` or check `is_active`.


