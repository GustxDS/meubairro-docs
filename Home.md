---
tags: [home, index, overview]
aliases: [Home, Index, Start Here]
---

# Meu Bairro — Knowledge Base

> Neighborhood communication app — React Native + Expo frontend / Node.js + Express + Drizzle backend / PostgreSQL (Neon).

---

## Ecosystem

| Repo | Role |
|------|------|
| `meubairro` | Frontend — React Native + Expo + NativeWind + RNR |
| `meubairro-backend` | Backend — Node.js + Express + Drizzle ORM |
| `meubairro-docs` | This vault — architecture, specs, skills, plans |

**Data flow:** Mobile → REST (Bearer JWT) → Express API → Drizzle ORM → PostgreSQL

---

## Product & Architecture

- [[spec/product|Product Vision]] — problem, solution, target audience, MVP scope
- [[spec/architecture|Architecture]] — full tech stack, folder structure, data flow, environments

---

## Domain Specs

```
Auth & Identity
  [[spec/auth|Authentication]] → [[spec/roles|Roles & RBAC]]
         ↓
Community
  [[spec/neighborhood|Neighborhoods]] — join by link or code, membership states
         ↓
Content
  [[spec/posts|Posts (Avisos & Negócios)]] | [[spec/realtime|Real-time Status]]
         ↓
Data & API
  [[spec/database|Database Schema]] | [[spec/endpoints|API Endpoints (23)]]
         ↓
Frontend
  [[spec/ui|Frontend UI & React Query Patterns]]
```

---

## Implementation Skills

Use these when writing code — they contain patterns, examples, and conventions:

### Frontend

| Skill | Use When |
|-------|----------|
| [[skills/frontend/authentication\|Frontend Auth]] | Auth context, SecureStore tokens, Axios interceptor, post-login routing |
| [[skills/frontend/components\|Frontend Components]] | New screens, component library, NativeWind design system |
| [[skills/frontend/navigation\|Navigation]] | New routes, route groups, auth guard, deep links |
| [[skills/frontend/forms\|Forms]] | Form screens, react-hook-form + Zod, image upload, submit state |
| [[skills/frontend/data-fetching\|Data Fetching]] | React Query queries/mutations, cache invalidation, push-triggered refetch |
| [[skills/frontend/error-handling\|Error Handling]] | Error boundaries, Sentry, logger, toast, offline banner |
| [[skills/frontend/storage\|Storage]] | SecureStore vs MMKV — which to use and when |

### Backend

| Skill | Use When |
|-------|----------|
| [[skills/backend/authentication\|Backend Auth]] | Login, register, JWT signing, refresh tokens, auth middleware |
| [[skills/backend/api-routes\|API Routes]] | New endpoints, RBAC checks, middleware, services |
| [[skills/backend/database-schema\|Database Schema]] | Schema changes, Drizzle queries, soft deletes, migrations |
| [[skills/backend/push-notifications\|Push Notifications]] | Device token management, Expo Server SDK, cron sends |
| [[skills/backend/testing\|Testing]] | Vitest + Supertest integration tests |

### General

| Skill | Use When |
|-------|----------|
| [[skills/code_explanation\|Code Explanation Guide]] | How to explain patterns (asyncHandler, AppError, etc.) |

---

## Active Plans

- [[plans/problems|Refactoring & Bug Fixes]] — 7 pending improvements (validations, perf, schema)

---

## Quick Reference

### Critical Rules
- **Always** validate permissions on backend — frontend only hides/shows UI
- **Always** filter with `isNull(deleted_at)` — soft deletes everywhere
- **Always** scope queries by `neighborhood_id`
- **Never** return passwords in API responses
- **Never** filter expired posts in JS — use `gt(posts.expires_at, now)` in SQL

### Roles
```
membro (0) < admin (1) < superadmin (2)
```

### API Response Format
```json
{ "data": { ... }, "message": "..." }      // success
{ "error": { "code": "...", "message": "..." } }  // error
```

### AI Reading Order
For a new AI session, read in this order:
1. [[spec/architecture|Architecture]]
2. [[skills/backend/database-schema|Database Schema]]
3. [[skills/backend/api-routes|API Routes]]
4. [[skills/frontend/components|Frontend Components]]
5. [[skills/frontend/authentication|Frontend Auth]] + [[skills/backend/authentication|Backend Auth]]
6. [[skills/frontend/navigation|Navigation]]
7. [[skills/frontend/data-fetching|Data Fetching]]
8. [[spec/endpoints|API Endpoints]]
9. [[spec/database|Database Spec]]
10. [[spec/ui|Frontend UI Spec]]
