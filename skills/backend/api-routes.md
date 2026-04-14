---
name: Backend API Routes
description: Use when creating new REST API endpoints, adding middleware, implementing RBAC permission checks, or adding business logic to services in Meu Bairro
tags: [skill, api, routes, backend, rbac, middleware, how-to]
aliases: [API Routes Skill, Backend Skill, Routes Skill]
---

# Backend API Routes — Meu Bairro

## Stack

| Technology   | Role                          |
|--------------|-------------------------------|
| Node.js      | Server runtime                |
| TypeScript   | Static typing                 |
| Express      | HTTP framework                |
| Helmet       | Security headers middleware   |
| Drizzle ORM  | Type-safe ORM for PostgreSQL  |
| Zod          | Schema validation             |
| jsonwebtoken | JWT generation and validation |
| bcrypt       | Password hashing              |

---

## Folder Structure

```
src/
├── routes/       # Route definitions per resource
├── middleware/   # auth, validate, upload
├── services/     # Business logic (one file per domain)
├── db/           # Schema and connection
├── types/        # Shared TypeScript types
└── utils/        # AppError, asyncHandler, hash, token, constants
```

---

## Request Lifecycle

```
Client Request
  → authMiddleware (validates JWT, sets req.user)
  → validate(schema) (Zod body validation)
  → asyncHandler(controller)
      → service (business logic + DB queries)
  → Standardized JSON response
  → centralErrorHandler (catches all AppError throws)
```

---

## Response Format

**Success:**
```json
{ "data": { ... }, "message": "Recurso criado com sucesso" }
```

**Error:**
```json
{ "error": { "code": "VALIDATION_ERROR", "message": "E-mail já cadastrado" } }
```

---

## HTTP Status Codes

| Code | Usage                             |
|------|-----------------------------------|
| 200  | Success                           |
| 201  | Resource created                  |
| 400  | Validation error                  |
| 401  | Not authenticated (invalid token) |
| 403  | No permission (RBAC)              |
| 404  | Resource not found                |
| 500  | Internal server error             |

---

## Creating a New Route — Step by Step

### 1. Route File (`src/routes/your-resource.ts`)

```ts
import { Router, Response } from 'express';
import { authMiddleware, AuthRequest } from '../middleware/auth.js';
import { memberMiddleware } from '../middleware/member.js'; // sets req.member
import { validate } from '../middleware/validate.js';
import { asyncHandler } from '../utils/asyncHandler.js';
import { yourSchema } from '../validators/your-resource.js';
import { yourService } from '../services/your-resource.service.js';

const router = Router();

// GET — list resources
router.get(
  '/',
  authMiddleware,
  memberMiddleware,
  asyncHandler(async (req: AuthRequest, res: Response) => {
    const data = await yourService.list(req.member!.neighborhood_id);
    res.json({ data });
  })
);

// POST — create resource
router.post(
  '/',
  authMiddleware,
  memberMiddleware,
  validate(yourSchema),
  asyncHandler(async (req: AuthRequest, res: Response) => {
    const data = await yourService.create({
      ...req.body,
      author_id: req.user!.id,
      neighborhood_id: req.member!.neighborhood_id,
    });
    res.status(201).json({ data, message: 'Criado com sucesso' });
  })
);

export default router;
```

### 2. Register in `src/index.ts`

```ts
import yourResourceRoutes from './routes/your-resource.js';
app.use('/api/your-resource', yourResourceRoutes);
```

### 3. Service File (`src/services/your-resource.service.ts`)

```ts
import { db } from '../db/index.js';
import { yourTable } from '../db/schema.js';
import { eq, and, isNull } from 'drizzle-orm';
import { AppError } from '../utils/AppError.js';

export const yourService = {
  async list(neighborhoodId: string) {
    return db
      .select()
      .from(yourTable)
      .where(
        and(
          eq(yourTable.neighborhood_id, neighborhoodId),
          isNull(yourTable.deleted_at)
        )
      );
  },

  async create(data: { field: string; author_id: string; neighborhood_id: string }) {
    const [created] = await db.insert(yourTable).values(data).returning();
    return created;
  },
};
```

### 4. Zod Validator (`src/validators/your-resource.ts`)

```ts
import { z } from 'zod';

export const yourSchema = z.object({
  field: z.string().min(1).max(255),
});
```

---

## Middleware Reference

### `authMiddleware`
- Validates JWT from `Authorization: Bearer <token>`
- Sets `req.user = { id, email }`
- Returns 401 if missing or invalid

### `memberMiddleware`
- Fetches the user's active neighborhood membership
- Sets `req.member = { id, user_id, neighborhood_id, role }`
- Returns 403 if user has no active membership

### `validate(schema)`
- Validates `req.body` against a Zod schema
- Returns 400 with field-level errors on failure

### `upload` (Multer)
- Handles `multipart/form-data`
- Allowed types: JPEG, PNG, WebP
- Max size: 5 MB
- Saves to `uploads/posts/<uuid>.<ext>`

---

## RBAC — Role-Based Access Control

### Roles and Levels

```ts
const ROLE_LEVEL = { membro: 1, admin: 2, superadmin: 3 };
type Role = "membro" | "admin" | "superadmin";
```

**Hierarchy:** `superadmin > admin > membro`

**Core rule:** A user can only act on users with a strictly lower level.

### Permissions Matrix

| Action                        | Membro | Admin | Superadmin |
|-------------------------------|:------:|:-----:|:----------:|
| Create posts (any type)       | ✓      | ✓     | ✓          |
| Confirm status                | ✓      | ✓     | ✓          |
| Delete own posts              | ✓      | ✓     | ✓          |
| Delete any post               | ✗      | ✓     | ✓          |
| Approve/reject join requests  | ✗      | ✓     | ✓          |
| Remove member                 | ✗      | ✓     | ✓          |
| Promote member → admin        | ✗      | ✓     | ✓          |
| Remove admin                  | ✗      | ✗     | ✓          |
| Demote admin → member         | ✗      | ✗     | ✓          |
| Promote admin → superadmin    | ✗      | ✗     | ✓          |
| Transfer superadmin           | ✗      | ✗     | ✓          |
| Delete neighborhood           | ✗      | ✗     | ✓          |
| Leave neighborhood            | ✓      | ✓     | ✗*         |

\* Superadmin must transfer ownership before leaving.

### Checking Role in a Route

```ts
router.put(
  '/members/:id/role',
  authMiddleware,
  memberMiddleware,
  asyncHandler(async (req: AuthRequest, res: Response) => {
    const { role } = req.member!;

    if (!['admin', 'superadmin'].includes(role)) {
      throw new AppError('Sem permissão', 403, 'FORBIDDEN');
    }

    await memberService.changeRole(req.member!, req.params.id, req.body.role);
    res.json({ message: 'Role atualizado' });
  })
);
```

### Hierarchy Check in Services

```ts
const ROLE_LEVEL = { membro: 1, admin: 2, superadmin: 3 };

if (ROLE_LEVEL[actor.role] <= ROLE_LEVEL[target.role]) {
  throw new AppError('Não pode agir sobre este usuário', 403, 'FORBIDDEN');
}
```

### Superadmin Transfer (atomic)

```ts
await db.update(neighborhoodMembers).set({ role: 'admin' }).where(eq(...actor));
await db.update(neighborhoodMembers).set({ role: 'superadmin' }).where(eq(...target));
```

---

## Critical Rules

1. **Never use try/catch in controllers** — wrap with `asyncHandler()` and throw `AppError` in services
2. **Always scope data by neighborhood** — every query must filter by `neighborhood_id`
3. **Never return passwords** in any API response
4. **All permission checks on the backend** — frontend only hides buttons, never enforces
5. **Soft deletes everywhere** — use `set({ deleted_at: new Date() })`, filter with `isNull(deleted_at)`
6. **One neighborhood per user** — always check membership before allowing content creation
7. **Perform Graceful Shutdowns** — close the express server and Drizzle DB connections on SIGINT/SIGTERM

---

## Existing Endpoints Reference

| Method | Endpoint                        | Access      |
|--------|---------------------------------|-------------|
| POST   | /api/auth/register              | Public      |
| POST   | /api/auth/login                 | Public      |
| POST   | /api/auth/refresh               | Public      |
| POST   | /api/auth/logout                | Auth        |
| GET    | /api/auth/me                    | Auth        |
| POST   | /api/neighborhoods              | Member      |
| GET    | /api/neighborhoods/current      | Member      |
| DELETE | /api/neighborhoods/current      | Superadmin  |
| POST   | /api/neighborhoods/join/code    | Member      |
| POST   | /api/neighborhoods/join/link/:t | Member      |
| DELETE | /api/neighborhoods/join         | Member      |
| GET    | /api/join-requests              | Admin+      |
| PUT    | /api/join-requests/:id/approve  | Admin+      |
| PUT    | /api/join-requests/:id/reject   | Admin+      |
| GET    | /api/members                    | Admin+      |
| PUT    | /api/members/:id/role           | Admin+      |
| DELETE | /api/members/:id                | Admin+      |
| POST   | /api/members/leave              | Member      |
| GET    | /api/posts/notices              | Member      |
| POST   | /api/posts/notices              | Member      |
| DELETE | /api/posts/notices/:id          | Member (author) or Admin+ |
| GET    | /api/posts/business             | Member      |
| POST   | /api/posts/business             | Member      |
| DELETE | /api/posts/business/:id         | Member (author) or Admin+ |
| POST   | /api/uploads                    | Member      |
| GET    | /api/status                     | Member      |
| POST   | /api/status/:type/confirm       | Member      |
| DELETE | /api/status/:type/confirm       | Member      |
| POST   | /api/devices/token              | Auth        |
| DELETE | /api/devices/token              | Auth        |
| GET    | /api/notifications              | Auth        |
| PATCH  | /api/notifications/read-all     | Auth        |
