# 🛡️ FRONTEND ENGINEERING STANDARDS

> These standards apply to **ALL frontend repositories** in the organization. PR-Agent will use these as binding rules
> when reviewing code.

---

## 1. TYPESCRIPT & TYPE SAFETY

- **No Any Types:** Avoid `any`. Use proper types or `unknown` with type guards.
- **Strict Null Checks:** Always handle `null` and `undefined`. Use optional chaining `?.` and nullish coalescing `??`.
- **Explicit Exported Return Types:** Exported functions/components must declare explicit return types.
- **Type Inference:** Let TypeScript infer types where obvious. Explicit types for function signatures and complex
  objects.
- **Interface vs Type:** Use `interface` for object shapes, `type` for unions/intersections.
- **Generics:** Use generic types for reusable components and hooks.
- **Type Guards:** Create type guard functions for runtime type checking.
- **Discriminated Unions:** Prefer discriminated unions over boolean-flag state shapes.
- **Utility Types & Literals:** Use utility types (`Pick`, `Omit`, `Partial`, `Readonly`, `Record`) and `as const` where
  they improve safety.

---

## 2. REACT BEST PRACTICES

### Component Design

- **Functional Components:** Always use functional components with hooks.
- **Single Responsibility:** Each component should do one thing well.
- **Composition Over Props:** Prefer component composition over complex prop configurations.
- **Prop Types:** Define explicit TypeScript interfaces for all component props.
- **DRY PRINCIPLE:** Do not repeat yourself. 
### Hooks

- **Hook Dependencies:** Always include all dependencies in useEffect/useCallback/useMemo deps arrays.
- **Custom Hooks:** Extract reusable logic into custom hooks prefixed with `use`.
- **Hook Rules:** Never call hooks conditionally or in loops. Always at top level.
- **Cleanup Functions:** Return cleanup functions in useEffect when needed (timers, subscriptions).

### Performance

- **React.memo:** Use for components that render often with same props.
- **useMemo:** Memoize expensive computations.
- **useCallback:** Memoize callback functions passed to child components.
- **Lazy Loading:** Use `React.lazy()` for route-based code splitting.
- **Keys in Lists:** Use stable, unique keys (not array index) for list items.

---

## 3. STATE MANAGEMENT

- **Local State First:** Use useState for component-local state.
- **Context Sparingly:** Use Context only for truly global state (auth, theme).
- **TanStack Query:** Use for server state (API data fetching).
- **Avoid Prop Drilling:** If passing props > 2 levels, use Context or composition.
- **Immutable Updates:** Never mutate state directly. Use spread operators or immutability helpers.

---

## 4. TANSTACK QUERY (REACT QUERY) PATTERNS

- **Query Keys:** Use structured, hierarchical keys: `['users', userId]` not `['user-123']`.
- **Query Functions:** Keep pure. No side effects in query functions.
- **Mutations:** Always handle `onSuccess`, `onError`, and cache invalidation.
- **Cache Invalidation:** Invalidate related queries after mutations.
- **Loading/Error States:** Always handle loading and error states in UI.
- **Stale Time:** Set appropriate `staleTime` to reduce unnecessary refetches.

---

## 5. FORMS & VALIDATION

- **React Hook Form:** Use for all forms. Avoid uncontrolled components.
- **Zod Schemas:** Define validation schemas with Zod in `src/schemas/`.
- **Controlled Inputs:** All form inputs must have `value` and `onChange`.
- **Validation Feedback:** Show clear, user-friendly error messages.
- **Disable Submit:** Disable submit button during form submission.

---

## 6. SECURITY

- **No Secrets in Frontend:** Never commit API keys, tokens, or secrets. Use env variables for public keys only.
- **XSS Prevention:** Avoid `dangerouslySetInnerHTML`. If necessary, sanitize with DOMPurify.
- **Input Sanitization:** Validate and sanitize all user input before sending to API.
- **HTTPS Only:** All API calls must use HTTPS.
- **Secure Storage:** Never store sensitive data (passwords, tokens) in localStorage. Use httpOnly cookies.
- **CSP Headers:** Implement Content Security Policy headers.

---

## 7. ACCESSIBILITY (A11Y)

- **Semantic HTML:** Use proper HTML elements (button, nav, main, article, etc.).
- **Alt Text:** All images must have descriptive `alt` attributes.
- **Form Labels:** Every input must have an associated `<label>` or `aria-label`.
- **Keyboard Navigation:** Ensure all interactive elements are keyboard accessible (tab order).
- **Focus Management:** Manage focus after modal opens/closes and route changes.
- **ARIA Attributes:** Use `aria-label`, `aria-describedby`, `aria-live` where appropriate.
- **No Div-as-Button:** Do not use `<div>`/`<span>` as buttons without proper semantics and keyboard behavior.
- **Color Contrast:** Maintain WCAG AA contrast ratio (4.5:1 for normal text).
- **Screen Reader Testing:** Test with screen readers (NVDA, JAWS, VoiceOver).

---

## 8. PERFORMANCE & OPTIMIZATION

- **Bundle Size:** Keep main bundle < 200KB. Use code splitting and lazy loading.
- **Image Optimization:** Use modern formats (WebP, AVIF). Implement lazy loading.
- **Memoization:** Memoize expensive computations and stable callbacks.
- **Debounce/Throttle:** Debounce search inputs and throttle scroll handlers.
- **Virtualization:** Use virtual scrolling for large lists (react-window, react-virtualized).
- **Web Vitals:** Monitor Core Web Vitals (LCP < 2.5s, FID < 100ms, CLS < 0.1).
- **Avoid Premature Memoization:** Flag `useMemo`/`useCallback` usage without measurable performance need.

---

## 9. ERROR HANDLING

- **Error Boundaries:** Wrap components in error boundaries to catch render errors.
- **Try-Catch:** Wrap async operations in try-catch blocks.
- **User-Friendly Messages:** Show clear, actionable error messages to users.
- **Error Logging:** Log errors to monitoring service (Sentry, LogRocket).
- **Fallback UI:** Provide fallback UI for error states.

---

## 10. CODE QUALITY & ORGANIZATION

### File Structure

- **Naming Convention:** PascalCase for components, camelCase for utilities/hooks.
- **Index Files:** Use index.ts for public API exports.

### Imports

- **Absolute Imports:** Use path aliases (`#/`, `~/`) for cleaner imports.
- **Import Order:** Sort: React → External libs → Internal → Styles.
- **Named Exports:** Prefer named exports over default exports.

### Comments

- **Why, Not What:** Comment intent and reasoning, not obvious code.
- **TODO/FIXME:** Use TODO and FIXME comments for technical debt.
- **JSDoc:** Document complex utility functions with JSDoc.

---

## 11. TESTING

- **Test Critical Paths:** Test user flows, not implementation details.
- **React Testing Library:** Use RTL for component tests (not Enzyme).
- **Accessibility Tests:** Use `axe-core` for automated a11y testing.
- **Mock API Calls:** Mock API calls with MSW (Mock Service Worker).
- **Coverage:** Aim for 80%+ coverage on critical business logic.

---

## 12. API INTEGRATION

- **Axios Instance:** Use configured Axios instance from `src/services/axiosInstance.ts`.
- **Request/Response Interceptors:** Handle auth tokens and error responses globally.
- **Loading States:** Always show loading indicators during API calls.
- **Error Handling:** Gracefully handle network errors and API errors.
- **Retry Logic:** Implement retry for transient failures.

---

## 13. ROUTING

- **React Router v7:** Use Data Routers with loaders and actions.
- **Protected Routes:** Wrap authenticated routes with `PrivateRoute` component.
- **Lazy Route Loading:** Lazy load route components for code splitting.
- **404 Handling:** Provide a catch-all route for 404 pages.
- **Navigate Programmatically:** Use `useNavigate()` hook, not `window.location`.

---

## 14. STYLING

- **Tailwind CSS:** Use Tailwind utility classes. Avoid inline styles.
- **Component Variants:** Use `class-variance-authority` for component variants.
- **Responsive Design:** Mobile-first responsive design with Tailwind breakpoints.
- **Dark Mode:** Support dark mode if required by design.
- **Consistent Spacing:** Use Tailwind spacing scale (4px increments).

---

## 15. REQUIRED PROJECT FILES

Every frontend project must include these root-level files:

- **README.md:** Setup instructions, project structure, deployment guide.
- **CONTRIBUTING.md (or CONTRIBUTION.md):** Branch naming, commit conventions, PR process.
- **.github/pull_request_template.md:** Standardized PR checklist.

Projects missing these files are non-compliant.

---

## 16. ARCHITECTURE & FOLDER PLACEMENT

- **`/src/components/ui`:** Atomic UI only. No business logic.
- **`/src/components/forms`:** Form UI only, typed by Zod schemas from `/src/schemas`.
- **`/src/pages`:** Page-level composition only. No business logic.
- **`/src/hooks`:** Logic-only hooks, no JSX. Hook filenames must start with `use`.
- **`/src/services`:** API clients only, using `axiosInstance.ts`. Never call axios directly in components.
- **`/src/schemas`:** Zod schemas for form/API validation.
- **`/src/types`:** Shared and API type definitions.
- **`/src/utils`:** Pure helpers only. No DOM/React dependencies.
- **`/src/data-layer`:** Data-access layer for React Query (query keys, query/mutation functions, and schema-based
  validation).

---

## 17. JAVASCRIPT RULES

- **Const by Default:** Use `const` unless reassignment is required (`let`). `var` is forbidden.
- **Destructuring:** Use object/array destructuring where applicable.
- **No Arrow Functions in JSX:** Define named handlers before JSX.
- **No Inline Styles:** Use Tailwind/CSS classes.
- **No Magic Numbers/Strings:** Extract to named constants/enums.
- **Default Parameters:** Prefer function default params over short-circuit fallbacks.
- **Named Constants for Settings:** Use constants for timeout and configuration values.

---

## 18. REACT COMPONENT RULES

- **Function Components Only:** Class components are forbidden.
- **One Component Per File:** Split files to keep components focused.
- **File Size Limit:** Component files must not exceed 300 lines.
- **Destructured Props:** Destructure at function signature/top-level.
- **JSDoc Required:** Every component must include concise JSDoc for purpose and props.
- **Business Logic Comments:** Add brief inline comments for non-trivial business logic.
- **Break Nested JSX:** Extract subcomponents to reduce nesting and complexity.
- **Stable Keys:** Use unique stable IDs from data for list keys.

---

## 19. HOOKS RULES

- **Naming:** Custom hooks must be prefixed with `use`.
- **Hook Rules:** Never call hooks conditionally or in loops.
- **Effect Scope:** Avoid large `useEffect` blocks. Extract named helpers or split effects.
- **Memoization Discipline:** Do not add `useCallback`/`useMemo` only to silence linting; use only with proven
  performance need.

---

## 20. CLEAN CODE & IMPORTS

- **No Dead Code:** Remove commented-out and unreachable code in PRs.
- **Early Returns:** Prefer guard clauses over deep nesting.
- **Meaningful Naming:** Avoid vague names (`x`, `tmp`, `val`).
- **Import Order:** External libraries → absolute imports → relative imports.


---

## 22. ENVIRONMENT & TOOLING

- **`.env.example`:** Must include all required env vars.
- **ESLint:** Must use org-aligned linting rules.
- **Prettier:** Must use standardized formatting.
- **Pre-commit Hooks:** Husky/lefthook must run lint and tests before commit.

---

## 23. TESTING & CI/CD

- **Unit Tests Required:** For reusable/public functions, reusable components, and custom hooks.
- **UI Testing Stack:** Use Jest + `@testing-library/react`.
- **Merge Gate:** All tests must pass before PR merge.
- **CI Pipeline:** Must run lint, tests, and build checks.
- **Deployment Gate:** Production deployment requires passing CI and manual approval.

---

## 24. FINAL ENFORCEMENT

These standards are mandatory across all frontend teams and projects. Code violating these rules must be rejected during
review. **Priority:** These standards are non-negotiable. Code reviews should catch violations before merge.

