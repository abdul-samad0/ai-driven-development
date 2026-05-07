---
name: caching-performance-optimization
description: Identify and apply caching and performance optimizations in Django/DRF backends, focusing on database query reduction, efficient ORM usage, and Django/Redis cache patterns. Use when the user mentions slowness, performance, scaling, query optimization, or caching.
---

# Caching and Performance Optimization

## Purpose

This skill guides the agent when improving performance of Django/Django REST Framework backends, with a focus on:

- Reducing database load (query count, N+1 issues, heavy aggregations)
- Applying appropriate caching layers (per-view, per-object, low-level cache API, Redis)
- Using efficient ORM patterns (`select_related`, `prefetch_related`, pagination, indexing)
- Avoiding premature optimization while still addressing clear bottlenecks

Use this skill whenever the user:

- Mentions slow endpoints, performance issues, or scaling problems
- Asks about caching strategies, Redis, or reducing DB load
- Wants a performance-oriented review of a Django/DRF API, view, or service

---

## Quick Start Checklist

When the user asks for performance or caching help in a Django/DRF backend:

1. **Clarify the hotspot**
   - Identify which API endpoint, view, service, or query is slow.
   - If not specified, infer from filenames, URLs, or context.

2. **Inspect data access**
   - Look for repeated queries, N+1 patterns, large `QuerySet` evaluations, or heavy in-Python loops over DB rows.
   - Check for missing `select_related` / `prefetch_related`.

3. **Check response size & pagination**
   - Confirm that list endpoints use pagination.
   - Ensure large payloads are not returned unnecessarily.

4. **Identify caching opportunities**
   - Determine if the result is:
     - Expensive to compute
     - Read-heavy vs write-heavy
     - Tolerant of slight staleness
   - Choose an appropriate caching strategy (see “Caching Strategies”).

5. **Apply optimizations with safety**
   - Keep behavior identical for correctness.
   - Avoid caching sensitive or user-specific data without scoping keys correctly.
   - Document cache keys and invalidation rules in code and/or docstrings.

6. **Re-check complexity**
   - Confirm reduced query count and smaller data processing.
   - Ensure no obvious new bottlenecks were introduced.

---

## Core Principles

- **Measure first where possible**: Focus on clear hotspots (slow endpoints, heavy DB usage), not speculative micro-optimizations.
- **Prefer simple, explicit caching**: Start with Django’s cache framework and query optimizations before complex custom mechanisms.
- **Optimize database access before code**: N+1 queries and large unpaginated lists are common root causes.
- **Scope caches carefully**: Keys must reflect all inputs that affect the cached result (user, filters, parameters, permissions).
- **Avoid stale or incorrect data**: Define clear invalidation rules; when in doubt, choose shorter TTLs or explicit invalidation hooks.

---

## Database & ORM Optimization

### Checklist

When inspecting a slow view, serializer, or service:

1. **Locate the query path**
   - Identify where the `QuerySet` is built and evaluated (services, managers, viewsets, serializers).
   - Look for `.all()`, `.filter()`, `.annotate()`, `.order_by()`, and subsequent iterations.

2. **Check for N+1 queries**
   - Repeated attribute access on related objects inside loops is a red flag.
   - In DRF serializers, repeated lookups on related fields without `select_related` or `prefetch_related` can cause N+1 issues.

3. **Apply `select_related` / `prefetch_related`**
   - Use `select_related` for `ForeignKey` and `OneToOneField`.
   - Use `prefetch_related` for `ManyToManyField` and reverse relations.
   - Ensure prefetching happens at the view/service layer before passing data into serializers.

4. **Enforce pagination**
   - For list endpoints, ensure pagination is enabled and appropriate page sizes are used.
   - Avoid returning full tables without strong justification.

5. **Avoid unnecessary `.values()` transformations**
   - Only use `.values()` / `.values_list()` when the performance benefit is clear and the structure is well understood.
   - Prefer returning model instances when serializers or business logic expect them.

6. **Use indexes and filters thoughtfully**
   - Fields frequently used in `filter`, `order_by`, and `lookup` should be indexed via model `Meta.indexes` or `db_index=True`, subject to DB constraints.
   - Avoid overly broad `LIKE` queries or unbounded “search everything” behaviour without limits.

---

## Caching Strategies

When considering caching, first decide:

- **Scope**: public vs user-specific vs tenant-specific
- **Freshness tolerance**: seconds, minutes, or must always be up to date
- **Invalidation mechanism**: time-based (TTL), event-based (signals or explicit service methods), or hybrid

### 1. Low-level cache API (`django.core.cache`)

Use when:

- A specific expensive computation result is reused across requests.
- You need fine-grained control over keys and TTLs.

Pattern:

- Include all relevant inputs in the cache key.
- Use clear, namespaced keys (e.g. `vol_dashboard:table:{symbol}:{date}`).
- Keep TTLs conservative unless invalidation is well-defined.

### 2. Per-view / decorator-based caching

Use when:

- A whole endpoint can be cached safely (e.g. public data, dashboard summaries with limited personalization).
- URL and query parameters fully determine the output.

Prefer:

- `cache_page` for simple full-response caching.
- `vary_on_headers` or custom key functions when necessary to include headers (e.g. auth headers, language).

Avoid:

- Caching user-specific or permission-sensitive endpoints without ensuring cache segregation (e.g. by user ID, tenant ID, permissions).

### 3. Template fragment caching (if templates are used)

Use when:

- A specific expensive part of a template is repeatedly rendered across pages.
- The backend renders HTML templates with reusable fragments.

### 4. Custom caching in services

Use when:

- Business logic orchestrates multiple sources (DB, external APIs, computed metrics).
- The same logical “entity” or data slice is fetched/assembled repeatedly.

Pattern:

- Wrap the slow logic in a function in the service layer.
- Use a clear cache key builder function.
- Decide whether to use time-based TTLs or explicit invalidation from write paths.

---

## Redis and External Caches

When the project uses Redis (recommended for production):

- Prefer Redis as the cache backend for:
  - High read/write throughput
  - Shared cache across multiple app instances
- Ensure:
  - Proper configuration in Django `CACHES` setting.
  - Timeouts are set; avoid infinite TTL unless there is an explicit invalidation path.
  - Sensitive data is scoped correctly (user/tenant ids in keys) and not shared across tenants.

---

## Invalidation & Consistency

When adding caching, always define how and when cached values are invalidated:

1. **Time-based invalidation**
   - Use short TTLs for:
     - Fast-changing data
     - Data with no easy explicit invalidation hook
   - Longer TTLs are suitable for:
     - Derived analytics, reports, or dashboards updated periodically

2. **Event-based invalidation**
   - On model changes (creates, updates, deletes), explicitly clear or refresh relevant cache keys via service calls.
   - Prefer explicit invalidation calls from services or domain operations over Django signals, unless the project clearly embraces signals.

3. **Hybrid**
   - Combine time-based TTL with targeted invalidations for critical flows (e.g. after writes).

When unsure, err on the side of:

- Shorter TTLs
- Smaller cache scope
- Simpler invalidation logic

---

## Performance Review Workflow

When the user asks to “optimize performance” for a part of the codebase:

1. **Identify the target**
   - Pinpoint the view(s), serializer(s), repository/service(s), or specific slow operation described.
   - If not obvious, search for filenames, URL patterns, or function names mentioned by the user.

2. **Trace the request flow**
   - Start at the DRF view/viewset or Django view.
   - Follow through serializers, services, and repository layers to DB access.
   - Note:
     - Where queries are issued
     - Where `QuerySet`s are evaluated
     - Where large loops or transformations occur

3. **List current bottlenecks**
   - N+1 queries
   - Large unpaginated queries
   - Repeated expensive calculations with identical inputs
   - Unnecessary data loading or unused fields

4. **Propose improvements in order of impact**
   - Database query optimization (relation loading, filters, indexes).
   - Response size reduction (pagination, field selection).
   - Caching:
     - Low-level cache for specific computations or querysets
     - Per-view caching for read-heavy endpoints
     - Service-level caching for composite results

5. **Implement changes incrementally**
   - Make one category of change at a time (e.g. add `select_related` and pagination before adding caching).
   - Keep the behavior consistent; avoid changing API contracts without explicit user request.

6. **Document assumptions and trade-offs**
   - Mention:
     - Which data is considered safe to cache and why.
     - Expected staleness windows.
     - Invalidation strategy (TTL or explicit).

---

## Example Optimization Scenarios

### Scenario 1: N+1 queries in list endpoint

**Trigger signals:**

- DRF serializer loops over related models.
- Query count increases linearly with result size.

**Agent actions:**

- Add `select_related` / `prefetch_related` on the queryset in the view/service before passing to serializer.
- Ensure pagination is enabled.
- Avoid per-object cache if relation loading fixes the issue cleanly.

### Scenario 2: Expensive dashboard aggregation

**Trigger signals:**

- Endpoint aggregates a large dataset or hits multiple external APIs.
- Result is the same (or similar) for many users for a fixed time window.

**Agent actions:**

- Implement a service method that:
  - Builds a stable cache key based on dashboard parameters (e.g. symbol, date range, filters).
  - Uses low-level cache API (or Redis) with a reasonable TTL (e.g. 30–300 seconds).
- Invalidate or let the cache expire based on update frequency requirements.

### Scenario 3: Large unbounded result sets

**Trigger signals:**

- List endpoints return entire tables or very large datasets.
- User complaints involve slow responses or heavy memory usage.

**Agent actions:**

- Enforce pagination at the DRF viewset level.
- If necessary, add limits on maximum page size.
- Combine pagination with query optimization (`select_related` / `prefetch_related`).

---

## How to Apply This Skill in Practice

When invoked:

1. **Summarize context**
   - Briefly restate which endpoint, view, or service is being optimized and why (slow, heavy load, etc.).

2. **Show targeted code references**
   - Use code references for:
     - The query/ORM usage
     - The view/service containing the logic
   - Highlight where changes will be made.

3. **Explain chosen optimizations**
   - For each change (query optimization, pagination, caching), state:
     - What it does
     - Why it is safe
     - Any trade-offs (e.g. stale data window)

4. **Keep changes minimal but impactful**
   - Avoid introducing new patterns inconsistent with the project unless necessary.
   - Prefer adjusting existing services/repositories over adding many new layers.

5. **Respect project-wide standards**
   - Follow `AGENTS.md` / organization-wide standards (type safety, security, logging).
   - Ensure error handling and logging remain clear and non-verbose.
   - Avoid logging sensitive or excessive data when inspecting performance issues.

---

## Summary

Use this skill to:

- Systematically analyze Django/DRF performance issues.
- Optimize database access patterns before adding caches.
- Add well-scoped, explicit caching with clear invalidation rules.
- Keep changes aligned with project and organization standards around safety, correctness, and maintainability.

