# Meu Bairro - Documentation & Architecture

> Neighborhood communication app — React Native mobile app with Node.js REST API.

## 🗂️ Related Repositories

| Repository | Role | URL |
|------------|------|-----|
| **meubairro** | Frontend (React Native + Expo) | https://github.com/GustxDS/meubairro |
| **meubairro-backend** | Backend (Node.js + Express + Drizzle) | https://github.com/GustxDS/meubairro-backend |
| **meubairro-docs** | This repo — architecture, specs, and AI context | — |

**Why separate repos:** Independent deploys, clean Git history per layer, Expo works best standalone. Types kept in sync manually — sufficient for MVP.

## 🗺️ The Meu Bairro Ecosystem

```
┌──────────────────────────────────────────┐
│              Mobile App                  │
│  React Native + Expo + NativeWind + RNR  │
│              (TypeScript)                │
└──────────────────┬───────────────────────┘
                   │ HTTP (REST / JSON)
                   │ Authorization: Bearer <token>
┌──────────────────▼───────────────────────┐
│              Backend API                 │
│  Node.js + TypeScript + Express          │
│  Middleware (Auth) → Routes → Services   │
└──────────────────┬───────────────────────┘
                   │ SQL (Drizzle ORM)
┌──────────────────▼───────────────────────┐
│              PostgreSQL (Neon)           │
│  users, neighborhoods, posts, status,    │
│  confirmations, join_requests            │
└──────────────────────────────────────────┘
```

## 📁 Repository Structure

This repo has three main directories:

- **`spec/`** — Human-readable specifications: architecture, product vision, API endpoints, frontend UI rules, roles, auth flow, realtime status, posts, and neighborhoods.
- **`skills/`** — AI context files: coding rules for components, database schema, authentication, backend routes, push notifications, testing strategy, and code explanation guidelines.
- **`plans/`** — Active task lists, technical problems, and implementation phases.

## 🤖 Using with AI

If you are an AI assistant opening this project, read in this order:

1. **`spec/architecture.md`** — Full tech stack, folder structure, data flow, communication patterns.
2. **`skills/database-schema/SKILL.md`** — All tables, types, constraints, and relationships.
3. **`skills/backend-api-routes/SKILL.md`** — Backend routing conventions and patterns.
4. **`skills/frontend-components/SKILL.md`** — Navigation tree, component interfaces, design system, React Query patterns.
5. **`skills/authentication/SKILL.md`** — Auth flow, RBAC, JWT, session management.
6. **`skills/push-notifications/SKILL.md`** — Push architecture, device tokens, fire-and-forget pattern.
7. **`skills/testing/SKILL.md`** — Integration testing strategy with Vitest + Supertest.

## 📋 Specifications Quick Reference

| Spec | Covers |
|------|--------|
| [`spec/product.md`](spec/product.md) | Product vision, problem, solution, target audience |
| [`spec/architecture.md`](spec/architecture.md) | Tech stack, repo structure, API patterns, environments |
| [`spec/auth.md`](spec/auth.md) | Authentication, JWT, refresh tokens, logout flow |
| [`spec/roles.md`](spec/roles.md) | RBAC — membro, admin, superadmin permissions and hierarchy |
| [`spec/neighborhood.md`](spec/neighborhood.md) | Community creation, join by code/link, membership states |
| [`spec/posts.md`](spec/posts.md) | Avisos (notices) and Negócios (businesses), categories, expiration |
| [`spec/realtime.md`](spec/realtime.md) | Status do Bairro — real-time event confirmations |
| [`spec/frontend/ui.md`](spec/frontend/ui.md) | Navigation structure, component interfaces, design system, React Query patterns |
| [`spec/backend/endpoints.md`](spec/backend/endpoints.md) | Complete API endpoint reference |
| [`spec/backend/database.md`](spec/backend/database.md) | Full database schema specification |

## ⚡ Quick Start

### Frontend (meubairro)
```bash
yarn expo start
yarn android
```

### Backend (meubairro-backend)
```bash
# .env required: DATABASE_URL, JWT_SECRET, PORT
yarn dev        # tsx watch with hot-reload
yarn db:push    # sync schema to database
yarn run db:seed    # seed test data locally
```
