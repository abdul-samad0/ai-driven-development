# Keep Kaam V1 Backend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the Django + DRF backend for Keep Kaam V1, limited to `Leave Management System` and `Project Allocation`, with production-ready validation, policy enforcement, test coverage, auditability, and performance-conscious data access.

**Architecture:** Use a modular Django backend under `backend/` with domain apps for `access`, `people`, `projects`, `leave`, `notifications`, and `audit`. Keep business rules in service/domain layers, expose workflows through DRF APIs, and treat policy-heavy leave/allocation logic as application services instead of serializer-only logic.

**Tech Stack:** Python, Django, Django REST Framework, PostgreSQL, django-extensions, django-fsm, pytest or Django test runner, Celery-compatible async hooks (if enabled later), Redis-ready caching design, Google OAuth integration hooks, structured logging.

---

## Documentation Stage

- Documentation stage: `Planning complete`
- Repository stage on `2026-05-08`: `Docs-only; backend implementation has not started`
- Execution readiness: `Ready for Phase 1 after review/signoff on the access-control and auth assumptions`

## Planning Notes

- This plan assumes the repository is still docs-first and the backend codebase does not yet exist.
- Planned backend root: `backend/`
- Planned API style: DRF endpoints with clear service-layer orchestration.
- Planned execution model: complete one phase, run its full test suite, request review/signoff, then move to the next phase.
- Backend skills intentionally reflected in this plan:
  - schema design
  - audit and metadata fields
  - migration safety
  - API foundation tests
  - permission tests
  - mocking side effects
  - serviceevent/logging discipline
  - caching/performance optimization

## Phase Review Contract

Every phase must end with all of the following before moving forward:

- All phase tests pass.
- Migrations apply cleanly from zero and against an existing database state.
- New endpoints have authentication, authorization, validation, and unhappy-path coverage.
- Query counts are checked on sensitive list/detail endpoints where regressions are likely.
- Audit/logging implications are accounted for.
- A short review note is prepared for you summarizing what was built, what remains, and what risks still exist.

## Proposed Backend Layout

```text
backend/
├── manage.py
├── config/
│   ├── settings/
│   │   ├── base.py
│   │   ├── local.py
│   │   ├── test.py
│   │   └── production.py
│   ├── urls.py
│   ├── asgi.py
│   ├── wsgi.py
│   └── api_router.py
├── apps/
│   ├── common/
│   ├── access/
│   ├── people/
│   ├── projects/
│   ├── leave/
│   ├── notifications/
│   └── audit/
├── tests/
│   ├── factories/
│   ├── integration/
│   └── smoke/
└── requirements/
```

## Domain Boundaries

- `common`: shared base models, enums helpers, pagination, API error helpers, date utilities, holiday helpers
- `access`: user account mapping, OAuth identity linkage, role assignment, permission wiring
- `people`: employee profile and employment-state rules
- `projects`: customer, project, milestones, documents, communication logs, project assignments, allocation logic
- `leave`: leave types, balances, requests, approval routing, attachments, approval steps, policy engine
- `notifications`: in-app notification records and dispatch orchestration
- `audit`: append-only audit events for leave and allocation actions

---

## Phase 1: Platform Foundation and Access Control

**Outcome:** A runnable backend project exists with environment-aware settings, custom user model strategy decided, Google OAuth-ready identity design, RBAC foundations, common abstractions, and test infrastructure.

**Review Gate:** Do not start domain modeling until login/token/role assumptions are approved.

**Files**

- Create: `backend/manage.py`
- Create: `backend/config/settings/base.py`
- Create: `backend/config/settings/local.py`
- Create: `backend/config/settings/test.py`
- Create: `backend/config/settings/production.py`
- Create: `backend/config/urls.py`
- Create: `backend/config/api_router.py`
- Create: `backend/apps/common/models.py`
- Create: `backend/apps/common/exceptions.py`
- Create: `backend/apps/common/pagination.py`
- Create: `backend/apps/access/models.py`
- Create: `backend/apps/access/choices.py`
- Create: `backend/apps/access/admin.py`
- Create: `backend/apps/access/api/v1/serializers.py`
- Create: `backend/apps/access/api/v1/views.py`
- Create: `backend/apps/access/api/v1/token_views.py`
- Create: `backend/tests/factories/access.py`
- Create: `backend/tests/integration/test_healthcheck.py`
- Create: `backend/tests/integration/access/test_current_user_api.py`
- Create: `backend/tests/integration/access/test_role_assignment_api.py`
- Create: `backend/tests/integration/access/test_token_auth_api.py`
- Initialize Django project under `backend/` and register base apps: `rest_framework`, `django_extensions`, `apps.common`, `apps.access`.
- Decide and implement the identity model split:
  - Django auth user as the canonical login identity
  - employee profile as domain data in a separate app
  - OAuth identity table for Google account linkage
  - role assignment bridge table for explicit RBAC visibility
- Create shared base classes according to backend skills:
  - `TimeStampedModel` inheritance for normal records
  - a local `SoftDeleteBaseModel` only where soft delete is truly needed
  - enums in `choices.py`
  - no inline status literals
- Add API-wide defaults:
  - authenticated by default
  - consistent pagination
  - standard error envelope only if the team wants one; otherwise preserve DRF defaults consistently
  - request correlation ID support if logging stack is added early
- Implement token bootstrap flows explicitly:
  - token obtain endpoint for API login
  - token refresh endpoint
  - decide whether logout is stateless only or uses refresh-token blacklisting
  - define the Google OAuth exchange path that results in first-party API token issuance
- Implement RBAC primitives:
  - roles for `SYSTEM_ADMIN`, `HR`, `DELIVERY_MANAGER`, `LEADERSHIP`, `ACCOUNTS`, `TEAM_MEMBER`, `CORE`, `COFOUNDER`
  - assignment history with `assigned_by`
  - effective-role helpers for permission classes
- Add database-backed integrity rules for identity and access records:
  - unique provider subject per OAuth identity
  - unique active role assignment semantics enforced with a DB constraint or equivalent conditional uniqueness rule
  - indexes for active role lookup and token-subject lookup
- Add baseline endpoints:
  - healthcheck
  - current authenticated user summary
  - role assignment list/detail if admin-only visibility is desired early

**Phase 1 Code Direction**

Use concise patterns like these while implementing Phase 1.

`backend/config/settings/base.py`

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "rest_framework",
    "django_extensions",
    "apps.common",
    "apps.access",
]

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": (
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ),
    "DEFAULT_PERMISSION_CLASSES": (
        "rest_framework.permissions.IsAuthenticated",
    ),
    "DEFAULT_PAGINATION_CLASS": "apps.common.pagination.DefaultPagination",
    "PAGE_SIZE": 20,
}
```

Keep `django.contrib.sessions` for Django admin and framework compatibility, but use token auth for API access.

Use `rest_framework_simplejwt` as the default token implementation unless project constraints require a different JWT provider.

`backend/apps/access/choices.py`

```python
from django.db.models import TextChoices


class AccessRoleChoices(TextChoices):
    SYSTEM_ADMIN = "SYSTEM_ADMIN", "System Admin"
    HR = "HR", "HR"
    DELIVERY_MANAGER = "DELIVERY_MANAGER", "Delivery Manager"
    LEADERSHIP = "LEADERSHIP", "Leadership"
    ACCOUNTS = "ACCOUNTS", "Accounts"
    TEAM_MEMBER = "TEAM_MEMBER", "Team Member"
    CORE = "CORE", "Core"
    COFOUNDER = "COFOUNDER", "Cofounder"
```

`backend/apps/common/models.py`

```python
from django.db import models
from django_extensions.db.models import TimeStampedModel


class SoftDeleteBaseModel(TimeStampedModel):
    is_active = models.BooleanField(default=True)

    class Meta:
        abstract = True

    def delete(self, using=None, keep_parents=False):
        self.is_active = False
        self.save(update_fields=["is_active"])
```

`backend/apps/access/models.py`

```python
from django.conf import settings
from django.db import models
from django_extensions.db.models import TimeStampedModel

from apps.access.choices import AccessRoleChoices


class OAuthIdentity(TimeStampedModel):
    provider = models.CharField(max_length=32)
    provider_email = models.EmailField()
    provider_subject = models.CharField(max_length=255, unique=True)
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        related_name="oauth_identities",
        on_delete=models.CASCADE,
    )

    class Meta:
        indexes = [
            models.Index(fields=["provider", "provider_subject"]),
        ]


class UserRoleAssignment(TimeStampedModel):
    ended_at = models.DateTimeField(null=True, blank=True)
    role = models.CharField(max_length=32, choices=AccessRoleChoices.choices)
    assigned_by = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        related_name="assigned_roles",
        on_delete=models.PROTECT,
    )
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        related_name="role_assignments",
        on_delete=models.CASCADE,
    )

    class Meta:
        indexes = [
            models.Index(fields=["user", "role", "ended_at"]),
        ]
```

`backend/apps/access/api/v1/serializers.py`

```python
from rest_framework import serializers

from apps.access.models import UserRoleAssignment


class CurrentUserSerializer(serializers.Serializer):
    id = serializers.IntegerField()
    email = serializers.EmailField()


class UserRoleAssignmentSerializer(serializers.ModelSerializer):
    class Meta:
        model = UserRoleAssignment
        fields = [
            "id",
            "user",
            "role",
            "assigned_by",
            "ended_at",
            "created",
            "modified",
        ]
        read_only_fields = ["assigned_by", "created", "modified"]
```

`backend/apps/access/permissions.py`

```python
from rest_framework.permissions import BasePermission

from apps.access.choices import AccessRoleChoices


def user_has_role(user, role):
    return user.role_assignments.filter(role=role, ended_at__isnull=True).exists()


class IsSystemAdmin(BasePermission):
    def has_permission(self, request, view):
        return (
            request.user.is_authenticated
            and user_has_role(request.user, AccessRoleChoices.SYSTEM_ADMIN)
        )
```

`backend/apps/access/api/v1/views.py`

```python
from rest_framework.generics import ListCreateAPIView
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView

from apps.access.api.v1.serializers import (
    CurrentUserSerializer,
    UserRoleAssignmentSerializer,
)
from apps.access.models import UserRoleAssignment
from apps.access.permissions import IsSystemAdmin


class CurrentUserAPIView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request):
        serializer = CurrentUserSerializer(
            {"id": request.user.id, "email": request.user.email}
        )
        return Response(serializer.data)


class UserRoleAssignmentListCreateAPIView(ListCreateAPIView):
    permission_classes = [IsSystemAdmin]
    serializer_class = UserRoleAssignmentSerializer
    queryset = UserRoleAssignment.objects.select_related("user", "assigned_by")

    def perform_create(self, serializer):
        serializer.save(assigned_by=self.request.user)
```

`backend/apps/access/api/v1/urls.py`

```python
from django.urls import path
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

from apps.access.api.v1.views import (
    CurrentUserAPIView,
    UserRoleAssignmentListCreateAPIView,
)

app_name = "access"

urlpatterns = [
    path("token/", TokenObtainPairView.as_view(), name="token-obtain-pair"),
    path("token/refresh/", TokenRefreshView.as_view(), name="token-refresh"),
    path("me/", CurrentUserAPIView.as_view(), name="current-user"),
    path(
        "role-assignments/",
        UserRoleAssignmentListCreateAPIView.as_view(),
        name="role-assignment-list",
    ),
]
```

`backend/config/api_router.py`

```python
from django.urls import include, path


urlpatterns = [
    path("access/", include("apps.access.api.v1.urls")),
]
```

`backend/apps/access/admin.py`

```python
from django.contrib import admin

from apps.access.models import OAuthIdentity, UserRoleAssignment


@admin.register(OAuthIdentity)
class OAuthIdentityAdmin(admin.ModelAdmin):
    list_display = ("id", "provider", "provider_email", "provider_subject", "user")
    search_fields = ("provider_email", "provider_subject", "user__email")


@admin.register(UserRoleAssignment)
class UserRoleAssignmentAdmin(admin.ModelAdmin):
    list_display = ("id", "user", "role", "assigned_by", "ended_at")
    list_filter = ("role",)
    search_fields = ("user__email", "assigned_by__email")
```

`backend/tests/factories/access.py`

```python
import factory
from django.contrib.auth import get_user_model

from apps.access.choices import AccessRoleChoices
from apps.access.models import OAuthIdentity, UserRoleAssignment


class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = get_user_model()

    email = factory.Sequence(lambda n: f"user{n}@example.com")
    username = factory.Sequence(lambda n: f"user{n}")


class OAuthIdentityFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = OAuthIdentity

    provider = "google"
    provider_email = factory.Sequence(lambda n: f"user{n}@gmail.com")
    provider_subject = factory.Sequence(lambda n: f"google-subject-{n}")
    user = factory.SubFactory(UserFactory)


class UserRoleAssignmentFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = UserRoleAssignment

    role = AccessRoleChoices.TEAM_MEMBER
    assigned_by = factory.SubFactory(UserFactory)
    user = factory.SubFactory(UserFactory)
```

`backend/tests/integration/access/test_current_user_api.py`

```python
class CurrentUserAPIViewTestCase(APITestCase):
    @classmethod
    def setUpTestData(cls):
        cls.user = UserFactory()
        cls.base_url = reverse("access:current-user")

    def setUp(self):
        self.client.force_authenticate(user=self.user)

    def test_returns_current_user(self):
        response = self.client.get(self.base_url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data["id"], self.user.id)

    def test_unauthenticated(self):
        self.client.logout()
        with self.assertNumQueries(0):
            response = self.client.get(self.base_url)
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
```

`backend/tests/integration/access/test_token_auth_api.py`

```python
class TokenAuthAPIViewTestCase(APITestCase):
    @classmethod
    def setUpTestData(cls):
        cls.user = UserFactory(password="testpass123")
        cls.base_url = reverse("access:token-obtain-pair")

    def test_returns_access_and_refresh_tokens(self):
        response = self.client.post(
            self.base_url,
            {"email": self.user.email, "password": "testpass123"},
        )
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertIn("access", response.data)
        self.assertIn("refresh", response.data)
```

`backend/tests/integration/access/test_role_assignment_api.py`

```python
class UserRoleAssignmentListCreateAPIViewTestCase(APITestCase):
    @classmethod
    def setUpTestData(cls):
        cls.admin_user = UserFactory()
        cls.member_user = UserFactory()
        UserRoleAssignmentFactory(
            user=cls.admin_user,
            role=AccessRoleChoices.SYSTEM_ADMIN,
        )
        cls.base_url = reverse("access:role-assignment-list")

    def test_system_admin_can_create_role_assignment(self):
        self.client.force_authenticate(user=self.admin_user)
        payload = {
            "user": self.member_user.id,
            "role": AccessRoleChoices.HR,
        }
        response = self.client.post(self.base_url, payload)
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)

    def test_non_admin_gets_403(self):
        self.client.force_authenticate(user=self.member_user)
        response = self.client.get(self.base_url)
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)
```

**Validation and Error Cases**

- duplicate OAuth identity for the same provider subject
- duplicate active identical role assignment
- assigning roles to inactive users
- non-admin attempting role assignment changes
- invalid, expired, or missing token identity
- stale Google-linked identity with no corresponding user

**Test Coverage**

- model tests for role assignment uniqueness and lifecycle
- token obtain/refresh tests and invalid-credential tests
- 401 tests for protected endpoints using logout + zero-query assertions where applicable
- 403 tests for non-admin role-management access
- serializer tests for invalid role payloads
- endpoint tests for current-user response shape
- migration smoke test from empty database

**Performance / Operational Notes**

- index user-email, provider-subject, active-role lookups
- avoid permission checks that trigger repeated uncached group/role queries per request
- centralize role resolution in a reusable helper to prevent N+1 permission lookups across list views

**Phase Exit Criteria**

- app boots locally with test settings
- auth/token foundation is stable
- RBAC primitives are approved
- base test harness is ready for all later phases

---

## Phase 2: People, Customer, Project, and Assignment Domain Modeling

**Outcome:** Core people and project-allocation entities exist with safe schema design, lifecycle fields, reusable query patterns, and clean migrations.

**Review Gate:** Do not implement leave workflows until employee/project/assignment relationships are approved.

**Files**

- Create: `backend/apps/people/models.py`
- Create: `backend/apps/people/choices.py`
- Create: `backend/apps/people/admin.py`
- Create: `backend/apps/projects/models.py`
- Create: `backend/apps/projects/choices.py`
- Create: `backend/apps/projects/admin.py`
- Create: `backend/apps/projects/querysets.py`
- Create: `backend/apps/projects/services/assignment_validation.py`
- Create: `backend/apps/projects/services/assignment_mutation_service.py`
- Create: `backend/apps/projects/services/project_creation.py`
- Create: `backend/apps/people/services/employee_lifecycle_service.py`
- Create: `backend/apps/projects/migrations/`
- Create: `backend/tests/factories/people.py`
- Create: `backend/tests/factories/projects.py`
- Create: `backend/tests/integration/projects/test_project_models.py`
- Create: `backend/tests/integration/projects/test_assignment_validation.py`
- Create: `backend/tests/integration/people/test_employee_deactivation_service.py`
- Implement `EmployeeProfile` with employment type, employment status, join date, notice-period fields, deactivation state, and link to auth user.
- Implement `Customer`, `Project`, `ProjectMilestone`, `ProjectDocument`, `CustomerCommunicationLog`, `ProjectIntakeEvent`, and `ProjectAssignment`.
- Preserve scope discipline:
  - no onboarding module
  - no asset module
  - no broad CRM pipeline
  - only the intake trigger needed for `deal won -> create project`
- Encode assignment rules in a service layer, not only serializer validation:
  - one employee can hold multiple active assignments on the same project
  - identical parallel assignments are disallowed
  - different roles/purposes on the same project are allowed
  - temporary overallocation above 100% requires justification and end date
  - temporary overallocation cannot exceed 125% total
  - temporary overallocation cannot exceed 21 consecutive days
  - future-dated assignments contribute to total allocation
- Add reusable query methods for:
  - active assignments
  - current allocation by employee
  - future allocation impact
  - most recent active assignment for leave approver routing
- Implement an `EmployeeLifecycleService` that owns employee deactivation side effects across modules:
  - mark employee as deactivated
  - deactivate future-dated project assignments
  - expose a clear hook that the leave app can use to cancel pending and future approved leave
  - keep cross-app mutation orchestration in one service boundary so deactivation logic does not drift between apps
- Implement assignment create/update/deactivate mutations through an atomic service boundary:
  - wrap write flows in `transaction.atomic()`
  - lock the employee’s active and future assignment rows during validation-sensitive mutations
  - re-check total allocation inside the transaction before commit
  - return deterministic conflict errors when a concurrent write invalidates the attempted change
- Implement safe migrations:
  - schema migrations via `makemigrations`
  - separate data migrations if seed/reference rows are needed
  - batched backfills if any reference data grows large later
- Add explicit DB constraints in the schema layer for concurrency-sensitive records:
  - one intake event per `source_reference`
  - one active role assignment per `user + role`
  - one leave-balance account per `employee + leave_type`

**Phase 2 Code Direction**

`backend/apps/people/choices.py`

```python
from django.db.models import TextChoices


class EmploymentTypeChoices(TextChoices):
    FULL_TIME = "FULL_TIME", "Full Time"
    PART_TIME = "PART_TIME", "Part Time"
    CONTRACTUAL = "CONTRACTUAL", "Contractual"
    INTERN = "INTERN", "Intern"
    PROBATION = "PROBATION", "Probation"
```

`backend/apps/people/models.py`

```python
from django.conf import settings
from django.db import models
from django_extensions.db.models import TimeStampedModel

from apps.people.choices import EmploymentTypeChoices


class EmployeeProfile(TimeStampedModel):
    employee_code = models.CharField(max_length=32, unique=True)
    employment_type = models.CharField(
        max_length=24,
        choices=EmploymentTypeChoices.choices,
    )
    full_name = models.CharField(max_length=255)
    is_in_notice_period = models.BooleanField(default=False)
    join_date = models.DateField()
    deactivated_at = models.DateTimeField(null=True, blank=True)
    notice_period_start_date = models.DateField(null=True, blank=True)
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,
        related_name="employee_profile",
        on_delete=models.CASCADE,
    )
```

`backend/apps/projects/models.py`

```python
from django.db import models
from django_extensions.db.models import TimeStampedModel


class Customer(TimeStampedModel):
    customer_name = models.CharField(max_length=255)
    primary_contact_email = models.EmailField()


class Project(TimeStampedModel):
    project_name = models.CharField(max_length=255)
    budget_amount = models.DecimalField(max_digits=22, decimal_places=2)
    completion_percentage = models.DecimalField(max_digits=5, decimal_places=2, default=0)
    customer = models.ForeignKey(
        "projects.Customer",
        related_name="projects",
        on_delete=models.CASCADE,
    )


class ProjectAssignment(TimeStampedModel):
    allocation_percentage = models.DecimalField(max_digits=5, decimal_places=2)
    justification = models.TextField(null=True, blank=True)
    role_name = models.CharField(max_length=100)
    start_date = models.DateField()
    end_date = models.DateField(null=True, blank=True)
    employee = models.ForeignKey(
        "people.EmployeeProfile",
        related_name="project_assignments",
        on_delete=models.CASCADE,
    )
    project = models.ForeignKey(
        "projects.Project",
        related_name="assignments",
        on_delete=models.CASCADE,
    )


class ProjectIntakeEvent(TimeStampedModel):
    source_type = models.CharField(max_length=64)
    source_reference = models.CharField(max_length=255, unique=True)
    created_project = models.ForeignKey(
        "projects.Project",
        related_name="intake_events",
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
    )
```

`backend/apps/projects/querysets.py`

```python
from django.db.models import Q, Sum
from django.utils import timezone


def active_assignments(queryset):
    today = timezone.localdate()
    return queryset.filter(start_date__lte=today).filter(
        Q(end_date__isnull=True) | Q(end_date__gte=today)
    )


def employee_total_allocation(queryset, employee):
    return (
        active_assignments(queryset.filter(employee=employee))
        .aggregate(total=Sum("allocation_percentage"))
        .get("total")
        or 0
    )
```

`backend/apps/projects/services/assignment_validation.py`

```python
from decimal import Decimal

from django.core.exceptions import ValidationError


def validate_assignment_payload(*, total_allocation, allocation, end_date, justification):
    proposed_total = Decimal(total_allocation) + Decimal(allocation)

    if allocation <= 0 or allocation > 100:
        raise ValidationError("Allocation percentage must be between 0 and 100.")
    if proposed_total > Decimal("125"):
        raise ValidationError("Total allocation cannot exceed 125%.")
    if proposed_total > Decimal("100") and not end_date:
        raise ValidationError("Temporary overallocation requires an end date.")
    if proposed_total > Decimal("100") and not justification:
        raise ValidationError("Temporary overallocation requires justification.")
```

`backend/apps/people/services/employee_lifecycle_service.py`

```python
from django.db import transaction
from django.utils import timezone


class EmployeeLifecycleService:
    @staticmethod
    @transaction.atomic
    def deactivate_employee(employee):
        employee.deactivated_at = timezone.now()
        employee.save(update_fields=["deactivated_at"])
        employee.project_assignments.filter(start_date__gt=timezone.localdate()).update(
            end_date=timezone.localdate()
        )
        return employee
```

`backend/tests/integration/projects/test_assignment_validation.py`

```python
class AssignmentValidationTestCase(TestCase):
    def test_requires_justification_for_overallocation(self):
        with self.assertRaises(ValidationError):
            validate_assignment_payload(
                total_allocation=90,
                allocation=20,
                end_date=date.today() + timedelta(days=7),
                justification="",
            )
```

**Validation and Error Cases**

- assignment start date after end date
- temporary overallocation missing end date
- temporary overallocation missing justification
- total allocation over 125%
- duplicate active assignment with same employee + project + role + overlapping period
- assignment to inactive employee
- assignment to archived/inactive project
- negative allocation or allocation over 100% on a single assignment
- malformed project budget or completion percentage
- future assignment conflicting with temporary overallocation cap

**Test Coverage**

- model tests for field defaults, relationship integrity, and lifecycle behavior
- service tests for every assignment validation rule
- overlap tests for same-project same-role duplicate prevention
- edge tests for exactly 100%, exactly 125%, and 125% + 0.01 overflow
- notice/deactivation-related tests affecting assignment eligibility
- query tests for active/future allocation helpers

**Performance / Operational Notes**

- add indexes for `employee`, `project`, `status`, `start_date`, `end_date`, and active assignment queries
- precompute only where necessary; prefer query annotations before introducing cache
- ensure project list/detail APIs can use `select_related` and `prefetch_related` from the start
- row locking should be scoped narrowly to assignment-mutation transactions so normal reads stay cheap

**Phase Exit Criteria**

- ERD-backed models are implemented cleanly
- assignment rules are passing tests
- project-allocation data model is stable enough for API work

---

## Phase 3: Leave Domain Model and Policy Engine

**Outcome:** All leave entities and the full leave-policy engine exist, including eligibility, conversions, balance behavior, routing rules, overlap logic, revision semantics, and notice-period overrides.

**Review Gate:** This is the highest-risk business-rules phase. Do not expose public leave APIs until the policy matrix is approved.

**Files**

- Create: `backend/apps/leave/models.py`
- Create: `backend/apps/leave/choices.py`
- Create: `backend/apps/leave/querysets.py`
- Create: `backend/apps/leave/services/policy_engine.py`
- Create: `backend/apps/leave/services/balance_service.py`
- Create: `backend/apps/leave/services/approval_routing.py`
- Create: `backend/apps/leave/services/leave_request_service.py`
- Create: `backend/apps/leave/services/leave_lifecycle_service.py`
- Create: `backend/apps/leave/services/holiday_calculator.py`
- Create: `backend/apps/leave/services/business_calendar.py`
- Create: `backend/apps/leave/admin.py`
- Create: `backend/apps/leave/fixtures/seed_leave_types.json`
- Create: `backend/tests/factories/leave.py`
- Create: `backend/tests/integration/leave/test_leave_policy_engine.py`
- Create: `backend/tests/integration/leave/test_leave_balance_service.py`
- Create: `backend/tests/integration/leave/test_approval_routing.py`
- Create: `backend/tests/integration/leave/test_leave_lifecycle_service.py`
- Implement `LeaveType`, `SpecialLeaveCategory`, `LeaveBalanceAccount`, `LeaveRequest`, `LeaveAttachment`, `LeaveApprovalStep`, and `PublicHoliday`.
- Decide request status lifecycle using `FSMField` or an equally disciplined explicit transition mechanism:
  - `PENDING`
  - `APPROVED`
  - `REJECTED`
  - `NEEDS_REVISION`
  - `CANCELLED`
- Build a policy engine with pure-function style checks where possible:
  - leave-type eligibility
  - join-date-based annual leave eligibility
  - proration logic
  - rounding to nearest 0.5 day upward
  - casual monthly cap
  - casual > 2 consecutive days auto-converts to annual
  - sick > 2 consecutive days requires certificate unless invalid request is blocked
  - unpaid/special exempt from auto-conversion
  - wedding once per tenure
  - notice-period restrictions with HR/admin override
  - half-day disallowed
  - overlap blocking against `PENDING`, `APPROVED`, `NEEDS_REVISION`
- Define one business-calendar service as the source of truth for leave-day counting:
  - explicit weekend definition for the business calendar
  - how public holidays affect deduction vs visibility
  - which leave types use working-day counting vs calendar-day counting
  - how sandwich expansion is calculated and where it applies
- Implement approver routing:
  - primary approver is delivery manager of employee’s most recently assigned active project
  - fallback to HR if no active project
  - fallback to HR if project lacks delivery manager
  - dual-gate logic for backdated, wedding, special, unpaid
- Implement balance behavior:
  - annual reset on January 1 for eligible employees
  - first-year proration by remaining full calendar months
  - balance deduction with sandwich/calendar expansion when applicable
  - cancellation restoring balance only when business rules allow
- Implement a `LeaveLifecycleService` that owns leave mutations triggered by lifecycle events:
  - cancel pending leave for deactivated employees
  - cancel already-approved future leave for deactivated employees
  - restore or preserve balances according to the leave status being cancelled
  - emit structured domain results so audit and notification layers can respond consistently later
- Seed reference leave types and special-leave categories safely.

**Validation and Error Cases**

- leave end date before start date
- zero-day request
- half-day payload or unsupported fractional unit
- casual leave requested by probation employee
- annual leave requested before 12-month eligibility
- sick leave longer than 2 consecutive days without certificate
- special leave without category
- wedding leave requested twice in tenure
- leave during notice period without override authority
- backdated leave with invalid actor role
- overlap with `pending`, `approved`, or `needs revision`
- revision changing dates after manager approval and preserving expected approval semantics
- deactivated employee creating new leave

**Test Coverage**

- dense policy-engine test matrix for every leave type
- calendar/holiday tests around weekends and public holidays
- calendar-rule tests proving weekend, holiday, and sandwich behavior from one shared service
- exact-edge tests for:
  - 2-day casual
  - 3-day casual conversion
  - 2-day sick vs 3-day sick with and without certificate
  - annual eligibility on month boundary
  - proration rounding to 0.5
  - notice-period override by HR/admin vs rejection for team member
- approval-routing tests for active-project, no-project, and no-manager fallback cases
- overlap tests with each blocking status
- deactivation tests for future approved and pending leave cancellation side effects

**Performance / Operational Notes**

- keep policy engine mostly deterministic and stateless so it is easy to test and cheap to run
- avoid repeated project/assignment lookups in routing by using optimized query helpers
- add indexes on leave request employee, status, start/end date, leave type
- avoid recalculating yearly balances on every request; compute on lifecycle events or cache stable summaries carefully
- design lifecycle services so deactivation-triggered cancellations can be processed in bounded batches if employee history grows large

**Phase 3 Code Direction**

`backend/apps/leave/choices.py`

```python
from django.db.models import TextChoices


class LeaveTypeChoices(TextChoices):
    ANNUAL = "ANNUAL", "Annual"
    SICK = "SICK", "Sick"
    CASUAL = "CASUAL", "Casual"
    UNPAID = "UNPAID", "Unpaid"
    WEDDING = "WEDDING", "Wedding"
    SPECIAL = "SPECIAL", "Special"


class LeaveStatusChoices(TextChoices):
    PENDING = "PENDING", "Pending"
    APPROVED = "APPROVED", "Approved"
    REJECTED = "REJECTED", "Rejected"
    NEEDS_REVISION = "NEEDS_REVISION", "Needs Revision"
    CANCELLED = "CANCELLED", "Cancelled"
```

`backend/apps/leave/models.py`

```python
from django.db import models
from django_extensions.db.models import TimeStampedModel
from django_fsm import FSMField

from apps.leave.choices import LeaveStatusChoices, LeaveTypeChoices


class LeaveType(TimeStampedModel):
    code = models.CharField(max_length=24, choices=LeaveTypeChoices.choices, unique=True)
    name = models.CharField(max_length=100)


class LeaveBalanceAccount(TimeStampedModel):
    balance_days = models.DecimalField(max_digits=6, decimal_places=2, default=0)
    employee = models.ForeignKey(
        "people.EmployeeProfile",
        related_name="leave_balances",
        on_delete=models.CASCADE,
    )
    leave_type = models.ForeignKey(
        "leave.LeaveType",
        related_name="balance_accounts",
        on_delete=models.CASCADE,
    )


class LeaveRequest(TimeStampedModel):
    start_date = models.DateField()
    end_date = models.DateField()
    requested_days = models.DecimalField(max_digits=6, decimal_places=2)
    status = FSMField(
        max_length=20,
        choices=LeaveStatusChoices.choices,
        default=LeaveStatusChoices.PENDING,
    )
    employee = models.ForeignKey(
        "people.EmployeeProfile",
        related_name="leave_requests",
        on_delete=models.CASCADE,
    )
    leave_type = models.ForeignKey(
        "leave.LeaveType",
        related_name="leave_requests",
        on_delete=models.CASCADE,
    )
```

`backend/apps/leave/services/policy_engine.py`

```python
from decimal import Decimal, ROUND_UP

from django.core.exceptions import ValidationError

from apps.leave.choices import LeaveTypeChoices
from apps.people.choices import EmploymentTypeChoices


def round_up_to_half_day(value):
    return (Decimal(value) * 2).quantize(Decimal("1"), rounding=ROUND_UP) / 2


def validate_leave_request(*, employee, leave_code, requested_days, has_certificate):
    if leave_code == LeaveTypeChoices.CASUAL and employee.employment_type == EmploymentTypeChoices.PROBATION:
        raise ValidationError("Probation employees cannot request casual leave.")
    if leave_code == LeaveTypeChoices.SICK and requested_days > 2 and not has_certificate:
        raise ValidationError("Medical certificate is required for sick leave longer than 2 days.")
```

`backend/apps/leave/services/business_calendar.py`

```python
from datetime import timedelta


WEEKEND_DAYS = {5, 6}


def is_weekend(day):
    return day.weekday() in WEEKEND_DAYS


def iter_calendar_days(start_date, end_date):
    current = start_date
    while current <= end_date:
        yield current
        current += timedelta(days=1)
```

`backend/apps/leave/services/approval_routing.py`

```python
def resolve_leave_approver(employee):
    assignment = employee.project_assignments.order_by("-start_date").first()
    if not assignment:
        return "HR"
    return assignment.project.assignments.filter(role_name="Delivery Manager").first() or "HR"
```

`backend/apps/leave/services/balance_service.py`

```python
from decimal import Decimal


def deduct_balance(balance_account, days):
    balance_account.balance_days = Decimal(balance_account.balance_days) - Decimal(days)
    balance_account.save(update_fields=["balance_days"])
    return balance_account
```

`backend/apps/leave/services/leave_lifecycle_service.py`

```python
from django.db import transaction

from apps.leave.choices import LeaveStatusChoices


class LeaveLifecycleService:
    @staticmethod
    @transaction.atomic
    def cancel_for_deactivated_employee(employee):
        return employee.leave_requests.filter(
            status__in=[
                LeaveStatusChoices.PENDING,
                LeaveStatusChoices.APPROVED,
            ]
        ).update(status=LeaveStatusChoices.CANCELLED)
```

`backend/tests/integration/leave/test_leave_policy_engine.py`

```python
class LeavePolicyEngineTestCase(TestCase):
    def test_requires_certificate_for_long_sick_leave(self):
        employee = EmployeeProfileFactory(employment_type=EmploymentTypeChoices.FULL_TIME)
        with self.assertRaises(ValidationError):
            validate_leave_request(
                employee=employee,
                leave_code=LeaveTypeChoices.SICK,
                requested_days=3,
                has_certificate=False,
            )
```

`backend/tests/integration/leave/test_approval_routing.py`

```python
class ApprovalRoutingTestCase(TestCase):
    def test_falls_back_to_hr_when_employee_has_no_active_project(self):
        employee = EmployeeProfileFactory()
        approver = resolve_leave_approver(employee)
        self.assertEqual(approver, "HR")
```

**Phase Exit Criteria**

- leave policy engine is green
- reference data is stable
- routing and balance logic are approved as matching the scoped brief

---

## Phase 4: Leave APIs, Approval Workflow, and Employee/Manager UX Endpoints

**Outcome:** The leave module is exposed through secure APIs for creation, revision, approval, cancellation, listing, and reporting-ready detail views.

**Review Gate:** API contracts should be reviewed before frontend or automation consumers depend on them.

**Files**

- Create: `backend/apps/leave/api/v1/serializers.py`
- Create: `backend/apps/leave/api/v1/views.py`
- Create: `backend/apps/leave/api/v1/urls.py`
- Create: `backend/apps/leave/permissions.py`
- Create: `backend/apps/leave/filters.py`
- Create: `backend/apps/leave/services/approval_actions.py`
- Create: `backend/apps/leave/services/attachment_service.py`
- Create: `backend/apps/leave/services/reporting_service.py`
- Create: `backend/tests/integration/leave/test_leave_request_api.py`
- Create: `backend/tests/integration/leave/test_leave_approval_api.py`
- Create: `backend/tests/integration/leave/test_leave_revision_api.py`
- Create: `backend/tests/integration/leave/test_leave_cancellation_api.py`
- Create: `backend/tests/integration/leave/test_leave_attachment_api.py`
- Create: `backend/tests/integration/leave/test_leave_reporting_api.py`
- Implement employee endpoints:
  - create leave request
  - revise rejected / needs-revision request
  - cancel future approved request within allowed rules
  - list own requests with filters
  - view own leave request detail
  - view own balances
- Implement manager/HR endpoints:
  - pending approval queue
  - approve / reject / request revision
  - HR second-stage review for dual-gate types
  - override operations limited to authorized roles
- Implement scoped leave reporting endpoints for frontend operational use:
  - leave reporting list endpoint with filters
  - reporting-ready detail payloads for status, leave type, employee, dates, and approval-history context
  - role-limited reporting scope so actors only see data allowed by product rules
- Preserve domain semantics:
  - same reference ID on resubmission
  - previous comments visible
  - manager re-review not required after HR rejection + revision when rules say so
  - fallback HR-only path where manager route is absent
- Implement concrete medical-certificate handling for V1:
  - require a certificate attachment or stored document reference for sick leave longer than 2 consecutive days before submission succeeds
  - store attachment metadata and retrievable file location through a defined attachment service
  - expose download/view metadata only to authorized actors
  - ensure revised leave requests can replace or retain the certificate as needed by policy
- Add query filters:
  - status
  - leave type
  - date range
  - employee
  - approver queue view
- Add reporting filters:
  - status
  - leave type
  - date range
  - employee
  - actor-visible scope
- Implement approval, revision, cancellation, and submission actions through atomic service boundaries:
  - wrap mutation flows in `transaction.atomic()`
  - lock the target leave request and balance rows where needed
  - re-check mutable status and balance assumptions inside the transaction before commit
  - return deterministic conflict/already-processed responses for concurrent actions

**Validation and Error Cases**

- unauthorized employee attempting to approve leave
- manager approving request outside assigned queue
- HR attempting second-stage action before first-stage manager outcome where manager route exists
- cancellation after leave start date
- revision on non-revisable status
- malformed file metadata or unsupported attachment type
- invalid filter combinations or inverted date ranges
- stale concurrent update to same leave request
- missing required medical certificate for long sick leave
- unauthorized actor attempting to access certificate metadata or file reference
- unauthorized actor attempting to access leave reporting beyond allowed scope

**Test Coverage**

- endpoint tests for create/list/detail/update action flows
- 401 and 403 tests on every protected endpoint class
- serializer validation tests for invalid payloads
- action tests for approve/reject/revise/cancel outcomes
- concurrency-aware tests where the same request is acted on twice
- attachment tests for certificate-required sick leave flows
- query-count assertions for queue and list endpoints
- reporting endpoint tests for permissions, filters, empty states, and approval-history visibility

**Performance / Operational Notes**

- approval queue endpoint must use `select_related` for employee, leave type, and current approval step
- employee list endpoint must be paginated
- leave reporting endpoint must be paginated and filter-bounded by default
- avoid per-row balance recomputation in list serializers; annotate or prefetch summaries
- keep row locking limited to mutation paths so list and queue endpoints stay non-blocking

**Phase 4 Code Direction**

`backend/apps/leave/api/v1/serializers.py`

```python
from rest_framework import serializers

from apps.leave.models import LeaveRequest


class LeaveRequestSerializer(serializers.ModelSerializer):
    class Meta:
        model = LeaveRequest
        fields = [
            "id",
            "leave_type",
            "start_date",
            "end_date",
            "requested_days",
            "status",
        ]
        read_only_fields = ["status"]
```

`backend/apps/leave/permissions.py`

```python
from rest_framework.permissions import BasePermission

from apps.access.permissions import user_has_role
from apps.access.choices import AccessRoleChoices


class CanManageLeaveApprovals(BasePermission):
    def has_permission(self, request, view):
        return request.user.is_authenticated and (
            user_has_role(request.user, AccessRoleChoices.HR)
            or user_has_role(request.user, AccessRoleChoices.DELIVERY_MANAGER)
            or user_has_role(request.user, AccessRoleChoices.SYSTEM_ADMIN)
        )
```

`backend/apps/leave/services/attachment_service.py`

```python
from django.core.exceptions import ValidationError


class AttachmentService:
    @staticmethod
    def validate_medical_certificate(*, leave_code, requested_days, attachment):
        if leave_code == "SICK" and requested_days > 2 and not attachment:
            raise ValidationError("Medical certificate is required.")
```

`backend/apps/leave/services/approval_actions.py`

```python
from django.db import transaction

from apps.leave.choices import LeaveStatusChoices


class ApprovalActionService:
    @staticmethod
    @transaction.atomic
    def approve(leave_request):
        leave_request.status = LeaveStatusChoices.APPROVED
        leave_request.save(update_fields=["status"])
        return leave_request
```

`backend/apps/leave/services/reporting_service.py`

```python
from apps.leave.models import LeaveRequest


class LeaveReportingService:
    @staticmethod
    def visible_leave_requests(*, user):
        queryset = LeaveRequest.objects.select_related("employee", "leave_type")

        if user.is_superuser:
            return queryset

        return queryset
```

`backend/apps/leave/api/v1/views.py`

```python
from rest_framework.generics import ListCreateAPIView

from apps.leave.api.v1.serializers import LeaveRequestSerializer
from apps.leave.models import LeaveRequest


class MyLeaveRequestListCreateAPIView(ListCreateAPIView):
    serializer_class = LeaveRequestSerializer

    def get_queryset(self):
        return LeaveRequest.objects.filter(employee=self.request.user.employee_profile)

    def perform_create(self, serializer):
        serializer.save(employee=self.request.user.employee_profile)
```

`backend/tests/integration/leave/test_leave_attachment_api.py`

```python
class LeaveAttachmentValidationTestCase(APITestCase):
    def test_sick_leave_over_two_days_requires_certificate(self):
        response = self.client.post(self.base_url, self.payload_without_certificate)
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)
```

`backend/tests/integration/leave/test_leave_reporting_api.py`

```python
class LeaveReportingAPIViewTestCase(APITestCase):
    def test_unauthorized_actor_cannot_access_reporting_scope(self):
        response = self.client.get(self.base_url)
        self.assertIn(response.status_code, [status.HTTP_401_UNAUTHORIZED, status.HTTP_403_FORBIDDEN])
```

**Phase Exit Criteria**

- leave workflows are usable end-to-end through API
- all permission paths are covered
- API contracts are stable enough for consumers

---

## Phase 5: Project Allocation APIs, Capacity Views, and Intake Trigger

**Outcome:** The project-allocation domain becomes operable through APIs for project setup, assignment management, capacity visibility, communication logs, and the deal-won project intake path.

**Review Gate:** Allocation endpoints and summary contracts should be reviewed before dashboard/frontend work starts.

**Files**

- Create: `backend/apps/projects/api/v1/serializers.py`
- Create: `backend/apps/projects/api/v1/views.py`
- Create: `backend/apps/projects/api/v1/urls.py`
- Create: `backend/apps/projects/permissions.py`
- Create: `backend/apps/projects/filters.py`
- Create: `backend/apps/projects/services/capacity_service.py`
- Create: `backend/apps/projects/services/intake_service.py`
- Create: `backend/tests/integration/projects/test_project_api.py`
- Create: `backend/tests/integration/projects/test_assignment_api.py`
- Create: `backend/tests/integration/projects/test_capacity_api.py`
- Create: `backend/tests/integration/projects/test_intake_trigger_api.py`
- Implement endpoints for:
  - customers
  - projects
  - milestones
  - project documents metadata
  - customer communication logs
  - project assignments
  - allocation summary / heatmap-ready data
- Enforce assignment authority only for:
  - system administrator
  - people and culture manager
  - leadership / head of delivery
- Build capacity summary views for:
  - employee allocation total
  - overallocated roster
  - future allocation conflicts
  - project staffing view
- Implement `deal won -> create project` entrypoint as a constrained intake endpoint or service hook, without introducing a full sales-pipeline module.

**Validation and Error Cases**

- unauthorized actor creating assignments
- intake event attempting duplicate project creation for same source reference
- milestone date outside plausible project boundaries if business wants enforcement
- document metadata missing required fields
- communication log without actor context
- invalid completion percentage or health score enum
- assignment edits causing retroactive overallocation violations

**Test Coverage**

- CRUD and action tests for project entities that are in active scope
- permission tests for assignment authority roles
- service tests for intake deduplication
- capacity summary tests for exact threshold boundaries
- list filter tests and ordering tests
- query-count assertions for high-value summary endpoints

**Performance / Operational Notes**

- use annotated querysets for capacity summaries instead of Python loops over assignments
- cache only if real dashboards show repeated heavy reads; start with efficient SQL
- make summary endpoints explicitly paginated or bounded

**Phase 5 Code Direction**

`backend/apps/projects/api/v1/serializers.py`

```python
from rest_framework import serializers

from apps.projects.models import Project, ProjectAssignment


class ProjectSerializer(serializers.ModelSerializer):
    class Meta:
        model = Project
        fields = ["id", "customer", "project_name", "budget_amount", "completion_percentage"]


class ProjectAssignmentSerializer(serializers.ModelSerializer):
    class Meta:
        model = ProjectAssignment
        fields = [
            "id",
            "employee",
            "project",
            "role_name",
            "allocation_percentage",
            "start_date",
            "end_date",
            "justification",
        ]
```

`backend/apps/projects/permissions.py`

```python
from rest_framework.permissions import BasePermission

from apps.access.choices import AccessRoleChoices
from apps.access.permissions import user_has_role


class CanManageAssignments(BasePermission):
    def has_permission(self, request, view):
        return any(
            user_has_role(request.user, role)
            for role in [
                AccessRoleChoices.SYSTEM_ADMIN,
                AccessRoleChoices.HR,
                AccessRoleChoices.LEADERSHIP,
            ]
        )
```

`backend/apps/projects/services/capacity_service.py`

```python
from django.db.models import Sum

from apps.projects.models import ProjectAssignment


class CapacityService:
    @staticmethod
    def employee_capacity_summary():
        return (
            ProjectAssignment.objects.values("employee_id")
            .annotate(total_allocation=Sum("allocation_percentage"))
            .order_by("-total_allocation")
        )
```

`backend/apps/projects/services/intake_service.py`

```python
from apps.projects.models import Project, ProjectIntakeEvent


class IntakeService:
    @staticmethod
    def create_project_from_won_deal(*, source_reference, project_name, customer):
        intake_event, created = ProjectIntakeEvent.objects.get_or_create(
            source_reference=source_reference,
            defaults={"source_type": "DEAL_WON"},
        )
        if not created:
            return intake_event.created_project
        project = Project.objects.create(project_name=project_name, customer=customer, budget_amount=0)
        intake_event.created_project = project
        intake_event.save(update_fields=["created_project"])
        return project
```

`backend/tests/integration/projects/test_capacity_api.py`

```python
class CapacityServiceTestCase(TestCase):
    def test_returns_employee_allocation_summary(self):
        summary = CapacityService.employee_capacity_summary()
        self.assertIsNotNone(summary)
```

**Phase Exit Criteria**

- project allocation is operational through API
- intake trigger is stable
- capacity outputs are accurate and performant enough for V1

---

## Phase 6: Audit Logs, Notifications, and Accounts-Facing Reporting

**Outcome:** Important leave and allocation actions generate auditable history, notification records, and reportable outputs without broadening scope into unrelated HR or payroll modules.

**Review Gate:** Side-effect behavior should be approved before enabling outbound integrations or UI-triggered notifications.

**Files**

- Create: `backend/apps/audit/models.py`
- Create: `backend/apps/audit/api/v1/serializers.py`
- Create: `backend/apps/audit/api/v1/views.py`
- Create: `backend/apps/audit/services/audit_service.py`
- Create: `backend/apps/notifications/models.py`
- Create: `backend/apps/notifications/choices.py`
- Create: `backend/apps/notifications/api/v1/serializers.py`
- Create: `backend/apps/notifications/api/v1/views.py`
- Create: `backend/apps/notifications/api/v1/urls.py`
- Create: `backend/apps/notifications/services/notification_service.py`
- Create: `backend/apps/common/services/idempotency_service.py`
- Create: `backend/apps/reports/` 
- Create: `backend/apps/reports/api/v1/views.py`
- Create: `backend/tests/integration/audit/test_audit_log_service.py`
- Create: `backend/tests/integration/notifications/test_notification_service.py`
- Create: `backend/tests/integration/notifications/test_notification_api.py`
- Create: `backend/tests/integration/reports/test_accounts_leave_report_api.py`
- Implement append-only audit logging for:
  - leave request creation
  - leave approval actions
  - leave revision
  - leave cancellation
  - project assignment create/update/deactivate
  - project creation via intake trigger
- Implement notification records for:
  - leave submitted
  - leave approved
  - leave rejected
  - leave sent for revision
  - assignment changes when business rules require stakeholder awareness
- Expose notification APIs needed by the frontend plan:
  - paginated notification list
  - unread-only filter support
  - mark-as-read mutation
  - unread-count summary if lightweight enough to expose in the same phase
- Keep side effects non-blocking in design:
  - create DB records synchronously
  - keep external delivery adapters optional and swappable later
  - never let notification failure break the core transaction unless explicitly required
- Implement side-effect idempotency explicitly:
  - generate stable event keys from entity + transition/action
  - store or constrain those keys at the audit/notification layer
  - skip duplicate emissions on retries or concurrent duplicate calls
- Implement accounts-facing leave deduction reporting only for scoped needs, not full payroll.

**Validation and Error Cases**

- duplicate audit event generation on retried actions
- notification recipient missing or inactive
- actor attempting to mark another user's notification as read
- report filters invalid or unauthorized
- side-effect failure after main transaction success

**Test Coverage**

- mocked side-effect tests for notification dispatch boundaries
- audit service tests asserting exact event names and payload shape
- notification API tests for pagination, unread filtering, authorization, and mark-as-read behavior
- report tests for permissions, filters, and empty-state behavior
- idempotency tests where repeated action calls should not emit duplicate side effects

**Performance / Operational Notes**

- index audit tables by entity type, entity id, actor, created timestamp
- keep notification lists paginated
- keep unread-count queries narrow and cheap if exposed
- for reports, use narrow selects / values where full model hydration is unnecessary

**Phase 6 Code Direction**

`backend/apps/audit/models.py`

```python
from django.conf import settings
from django.db import models
from django_extensions.db.models import TimeStampedModel


class AuditLog(TimeStampedModel):
    entity_type = models.CharField(max_length=64)
    entity_id = models.CharField(max_length=64)
    event_name = models.CharField(max_length=100)
    event_key = models.CharField(max_length=255, unique=True)
    payload = models.JSONField(default=dict)
    actor = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        related_name="audit_logs",
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
    )
```

`backend/apps/notifications/models.py`

```python
from django.conf import settings
from django.db import models
from django_extensions.db.models import TimeStampedModel


class Notification(TimeStampedModel):
    event_name = models.CharField(max_length=100)
    event_key = models.CharField(max_length=255, unique=True)
    payload = models.JSONField(default=dict)
    is_read = models.BooleanField(default=False)
    recipient = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        related_name="notifications",
        on_delete=models.CASCADE,
    )
```

`backend/apps/audit/services/audit_service.py`

```python
from apps.audit.models import AuditLog
from apps.common.services.idempotency_service import build_event_key


class AuditService:
    @staticmethod
    def record(*, actor, entity_type, entity_id, event_name, payload=None):
        return AuditLog.objects.get_or_create(
            event_key=build_event_key(
                entity_type=entity_type,
                entity_id=entity_id,
                action=event_name,
            ),
            defaults={
                "actor": actor,
                "entity_type": entity_type,
                "entity_id": entity_id,
                "event_name": event_name,
                "payload": payload or {},
            },
        )[0]
```

`backend/apps/common/services/idempotency_service.py`

```python
def build_event_key(*, entity_type, entity_id, action):
    return f"{entity_type}:{entity_id}:{action}"
```

`backend/apps/notifications/services/notification_service.py`

```python
from apps.notifications.models import Notification


class NotificationService:
    @staticmethod
    def create(*, recipient, event_name, payload=None):
        return Notification.objects.create(
            recipient=recipient,
            event_name=event_name,
            payload=payload or {},
        )
```

`backend/apps/notifications/api/v1/serializers.py`

```python
from rest_framework import serializers

from apps.notifications.models import Notification


class NotificationSerializer(serializers.ModelSerializer):
    class Meta:
        model = Notification
        fields = ["id", "event_name", "payload", "is_read", "created"]
        read_only_fields = fields
```

`backend/apps/notifications/api/v1/views.py`

```python
from rest_framework.generics import ListAPIView, UpdateAPIView

from apps.notifications.api.v1.serializers import NotificationSerializer
from apps.notifications.models import Notification


class MyNotificationListAPIView(ListAPIView):
    serializer_class = NotificationSerializer

    def get_queryset(self):
        queryset = Notification.objects.filter(recipient=self.request.user).order_by("-created")
        unread_only = self.request.query_params.get("unread_only")
        if unread_only == "true":
            queryset = queryset.filter(is_read=False)
        return queryset


class NotificationMarkReadAPIView(UpdateAPIView):
    serializer_class = NotificationSerializer

    def get_queryset(self):
        return Notification.objects.filter(recipient=self.request.user)
```

`backend/tests/integration/notifications/test_notification_api.py`

```python
class NotificationAPIViewTestCase(APITestCase):
    def test_user_can_list_own_notifications(self):
        response = self.client.get(self.base_url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
```

`backend/apps/reports/api/v1/views.py`

```python
from rest_framework.response import Response
from rest_framework.views import APIView

from apps.leave.models import LeaveRequest


class AccountsLeaveDeductionReportAPIView(APIView):
    def get(self, request):
        data = LeaveRequest.objects.values("employee_id", "leave_type_id", "requested_days")
        return Response(list(data))
```

`backend/tests/integration/audit/test_audit_log_service.py`

```python
class AuditServiceTestCase(TestCase):
    def test_records_audit_log(self):
        entry = AuditService.record(
            actor=None,
            entity_type="leave_request",
            entity_id="123",
            event_name="leave_request_created",
        )
        self.assertEqual(entry.event_name, "leave_request_created")
```

**Phase Exit Criteria**

- scoped observability and notifications are present
- accounts reporting is available
- side effects are reliable and non-invasive

---

## Phase 7: Hardening, Observability, Performance, and Release Readiness

**Outcome:** The backend is resilient enough for staging rollout with logging, performance verification, security review, migration sanity, and regression coverage across the entire scoped V1 backend.

**Review Gate:** This phase closes the backend implementation cycle and should be approved before frontend integration or deployment promotion.

**Files**

- Create: `backend/apps/common/logging.py`
- Modify: `backend/config/settings/base.py`
- Create: `backend/tests/smoke/test_full_leave_flow.py`
- Create: `backend/tests/smoke/test_full_allocation_flow.py`
- Create: `backend/tests/integration/test_permissions_matrix.py`
- Create: `backend/tests/integration/test_query_regressions.py`
- Create: `backend/docs/backend-runbook.md`
- Add structured logging strategy aligned with the Django logging skill:
  - app logger
  - request/error logger
  - safe contextual fields
  - no sensitive payload leaks
- Add end-to-end regression scenarios:
  - employee submits leave
  - manager approves
  - HR final review where needed
  - balance updates
  - audit + notification records created
  - project assignment causes overallocation warning path
- Add a permissions matrix test suite covering all major actor types against major endpoint groups.
- Add query-regression tests on:
  - leave queue
  - leave list
  - project staffing list
  - capacity summary
- Run migration reliability checks:
  - fresh database bootstrap
  - upgrade path from prior migration state
  - seed/reference data integrity
- Prepare operational docs:
  - environment variables
  - local bootstrap
  - test commands
  - migration commands
  - common debugging flows

**Validation and Error Cases**

- logging formatter breaking on missing context
- staging config missing required secrets
- migration ordering mistakes
- accidental N+1 in serializers introduced late
- permission drift after new endpoint additions

**Test Coverage**

- smoke tests for the two main business modules
- permission matrix coverage
- logging tests where helper wrappers exist
- regression tests for common unhappy paths
- optional load-test planning note if high-risk summary endpoints prove expensive

**Performance / Operational Notes**

- measure before caching heavier summaries
- prefer `select_related`, `prefetch_related`, pagination, and indexed filters before adding Redis complexity
- only cache stable, read-heavy outputs with clearly scoped keys and invalidation rules

**Phase 7 Code Direction**

`backend/apps/common/logging.py`

```python
import logging


def get_app_logger(name):
    return logging.getLogger(f"keep_kaam.{name}")
```

`backend/config/settings/base.py`

```python
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "simple": {
            "format": "%(asctime)s %(levelname)s %(name)s %(message)s",
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "simple",
        },
    },
    "loggers": {
        "keep_kaam": {
            "handlers": ["console"],
            "level": "INFO",
        },
    },
}
```

`backend/tests/smoke/test_full_leave_flow.py`

```python
class FullLeaveFlowSmokeTestCase(APITestCase):
    def test_employee_to_manager_leave_flow(self):
        response = self.client.post(self.base_url, self.valid_payload)
        self.assertIn(response.status_code, [status.HTTP_201_CREATED, status.HTTP_200_OK])
```

`backend/tests/integration/test_query_regressions.py`

```python
class QueryRegressionTestCase(APITestCase):
    def test_leave_queue_query_count(self):
        with self.assertNumQueries(5):
            response = self.client.get(self.leave_queue_url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
```

`backend/docs/backend-runbook.md`

```md
# Backend Runbook

- Install dependencies
- Run migrations
- Seed reference data
- Run test suite
- Verify token auth config
```

**Phase Exit Criteria**

- full backend regression suite is green
- logging and operational docs exist
- main endpoints are secure, performant, and review-ready

---

## Cross-Phase Validation Matrix

These checks should be referenced repeatedly throughout implementation:

### Authentication and Authorization

- 401 on unauthenticated requests
- 403 on authenticated but unauthorized requests
- actor-role coverage for team member, delivery manager, HR, system admin, leadership, accounts

### Data Validation

- required-field validation
- enum validation
- date-range validation
- numeric boundary validation
- state-transition validation
- duplicate / overlap / uniqueness validation

### Workflow Integrity

- idempotent retry behavior where action endpoints may be retried
- correct status transitions only
- revision flow preserving allowed historical context
- deactivation cascades applied correctly

### Error Handling

- invalid payload shape
- invalid foreign keys
- unauthorized action attempts
- stale-object / already-processed action attempts
- attachment/document reference errors

### Performance

- list endpoints paginated by default
- query-count assertions on queue and summary endpoints
- high-cardinality filters indexed where justified
- serializer access patterns checked for N+1 risk

---

## Suggested Phase Sequence for Review

1. Phase 1: foundation and RBAC
2. Phase 2: people/projects schema
3. Phase 3: leave policy engine
4. Phase 4: leave APIs
5. Phase 5: allocation APIs
6. Phase 6: audit/notifications/reporting
7. Phase 7: hardening and release readiness

This sequence minimizes rework because leave routing depends on project assignment truth, and both modules depend on stable access control first.

---

## Assumptions to Keep Visible During Execution

- Backend code will live in `backend/` because the repo also needs room for frontend and docs.
- PostgreSQL is the target database, even if local development starts with a simpler setup.
- Google OAuth and token-based authentication are part of the foundation, but broad SSO edge integrations beyond the scoped actors are not.
- Notifications start as internal records first; external delivery channels can be added after the core domain is stable.
- Reporting remains scoped to leave deduction and operational allocation visibility, not payroll or full BI.

---

## Handoff

Plan complete and saved to [2026-05-08-keep-kaam-v1-backend-implementation-plan.md](/Users/mac/Documents/keep-kaam-training/docs/superpowers/plans/2026-05-08-keep-kaam-v1-backend-implementation-plan.md).

Recommended execution mode for this plan:

1. Subagent-driven per phase, because the phases are naturally separable and each one benefits from review before advancing.
2. Inside each approved phase, execute in TDD order: failing tests first, minimal implementation, verification, then review summary.
