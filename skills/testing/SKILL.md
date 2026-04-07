---
name: Testing
description: Vitest integration testing for Express backend routes in Meu Bairro — real database, no ORM mocks, Vitest + Supertest
---

# Testing — Meu Bairro

## Philosophy

Test backend routes hitting a **real database**. No unit mocks, no frontend component tests, no isolated service tests. Integration tests catch the bugs that matter: missing `isNull(deleted_at)`, broken RBAC, cascade failures, constraint violations.

**What to test:** Auth flow, RBAC permissions, CRUD operations, soft deletes.
**What NOT to test:** UI components, pure functions, push notifications (fire-and-forget).

---

## Stack

| Technology | Role |
|------------|------|
| Vitest     | Test runner |
| Supertest  | HTTP request testing |
| PostgreSQL | Real database (separate test DB) |
| Drizzle Kit | Migration management |

---

## Setup

### Vitest Config

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    dir: './tests',
    globalSetup: './tests/globalSetup.ts',
    setupFiles: ['./tests/setup.ts'],
  },
});
```

### Environment (tests/.env.test)

```env
DATABASE_URL=postgres://.../meubairro_test
JWT_SECRET=test-jwt-secret
```

### Global Setup

```ts
// tests/globalSetup.ts
export default async function setup() {
  // runs once before/after all tests
  // start/stop a test database connection pool if needed
}
```

### Per-File Setup

```ts
// tests/setup.ts
import { beforeAll, afterAll, afterEach } from 'vitest';
import { db } from '../src/db/index.js';
import * as schema from '../src/db/schema.js';

beforeAll(async () => {
  // Apply migrations to test DB
  await migrate(db, {});
});

afterEach(async () => {
  // Truncate all tables to ensure clean state between tests
  const tables = Object.values(schema).filter((t) => typeof t === 'object');
  for (const table of tables) {
    await db.execute(sql`TRUNCATE TABLE ${table} CASCADE`);
  }
});

afterAll(async () => {
  // Close DB connection
  await db.$client.end();
});
```

---

## Request Helpers

Write helpers to reduce boilerplate. Every test file gets:

```ts
// tests/utils/testClient.ts
import request from 'supertest';
import app from '../src/app.js';

const BASE = '/api';

function authenticated(token: string) {
  return {
    get(path: string) {
      return request(app).get(`${BASE}${path}`).set('Authorization', `Bearer ${token}`);
    },
    post(path: string) {
      return request(app).post(`${BASE}${path}`).set('Authorization', `Bearer ${token}`);
    },
    put(path: string) {
      return request(app).put(`${BASE}${path}`).set('Authorization', `Bearer ${token}`);
    },
    delete(path: string) {
      return request(app).delete(`${BASE}${path}`).set('Authorization', `Bearer ${token}`);
    },
  };
}

export { request, app, authenticated, BASE };
```

---

## Test Structure

```txt
tests/
├── setup.ts              # DB connection, migrations, cleanup
├── globalSetup.ts        # One-time setup/teardown
├── utils/
│   ├── factory.ts        # Seed functions for users, posts, statuses
│   └── testClient.ts     # Supertest wrapper with auth
├── auth.test.ts          # Register, login, refresh, logout
├── posts.test.ts         # Post CRUD, soft delete
├── rbac.test.ts          # Role permissions, hierarchy
└── statuses.test.ts      # Status flow, confirmations
```

---

## Seed Factories

```ts
// tests/utils/factory.ts
import { db } from '../../src/db/index.js';
import { users, neighborhoodMembers, posts } from '../../src/db/schema.js';
import { hash } from '../../src/utils/hash.js';

export async function createTestUser(email = `test-${Date.now()}@mail.com`, password = '123456') {
  const [user] = await db.insert(users).values({
    email,
    password: await hash(password),
  }).returning();
  return user;
}

export async function createTestToken(userId: string, opts = {}) {
  const { expires_in = '7d', secret = process.env.JWT_SECRET! } = opts;
  return jwt.sign({ user_id: userId, email: `${userId}@test.com` }, secret, { expiresIn: expires_in });
}

export async function createTestMember(userId: string, neighborhoodId: string, role = 'membro') {
  const [member] = await db.insert(neighborhoodMembers).values({
    user_id: userId,
    neighborhood_id: neighborhoodId,
    role,
  }).returning();
  return member;
}
```

---

## Test Examples

### Auth Flow

```ts
// tests/auth.test.ts
import { describe, expect, test } from 'vitest';
import { request } from './setup.js';
import { db } from '../src/db/index.js';
import { users, refreshTokens } from '../src/db/schema.js';
import { eq } from 'drizzle-orm';

describe('POST /api/auth/register', () => {
  test('should create user and return tokens', async () => {
    const res = await request(app)
      .post('/api/auth/register')
      .send({ email: 'new@test.com', password: '123456' })
      .expect(201);

    expect(res.body.data).toHaveProperty('token');
    expect(res.body.data).toHaveProperty('refresh_token');
    expect(res.body.data.user).toHaveProperty('id');
    expect(res.body.data.user).not.toHaveProperty('password');
  });

  test('should reject duplicate email', async () => {
    await request(app)
      .post('/api/auth/register')
      .send({ email: 'dup@test.com', password: '123456' });

    await request(app)
      .post('/api/auth/register')
      .send({ email: 'dup@test.com', password: '123456' })
      .expect(400, res => {
        expect(res.body.error.code).toBe('VALIDATION_ERROR');
      });
  });
});

describe('POST /api/auth/refresh', () => {
  test('should reject revoked token', async () => {
    const { data } = await request(app)
      .post('/api/auth/register')
      .send({ email: 'revoke@test.com', password: '123456' })
      .expect(201);

    await request(app)
      .post('/api/auth/logout')
      .send({ refresh_token: data.refresh_token })
      .expect(200);

    await request(app)
      .post('/api/auth/refresh')
      .send({ refresh_token: data.refresh_token })
      .expect(401);
  });
});
```

### RBAC Permissions

```ts
// tests/rbac.test.ts
import { describe, expect, test, beforeAll } from 'vitest';
import { authenticated, request, app } from './setup.js';
import { createTestUser, createTestMember, createTestToken } from './utils/factory.js';
import { db } from '../src/db/index.js';
import { neighborhoods, neighborhoodMembers } from '../src/db/schema.js';

let admin: any, member: any, neighborhood: any;
let adminToken: string, memberToken: string;

beforeAll(async () => {
  // Seed: create a neighborhood and members
  neighborhood = await db.insert(neighborhoods).values({
    name: 'Test Hood', city: 'City', state: 'ST', access_code: 'abcd1234', invite_token: 'tok'
  }).returning().then(r => r[0]);

  member = await createTestUser('member@test.com');
  await createTestMember(member.id, neighborhood.id, 'membro');
  memberToken = await createTestToken(member.id);

  admin = await createTestUser('admin@test.com');
  await createTestMember(admin.id, neighborhood.id, 'admin');
  adminToken = await createTestToken(admin.id);
});

describe('RBAC — post deletion', () => {
  test('member can delete own post', async () => {
    const res = await authenticated(memberToken)
      .post('/api/posts/notices')
      .send({ title: 'Test', description: 'Desc', category: 'seguranca' });
    const postId = res.body.data.id;

    await authenticated(memberToken)
      .delete(`/api/posts/notices/${postId}`)
      .expect(200);
  });

  test('admin can delete any post', async () => {
    const res = await authenticated(memberToken)
      .post('/api/posts/notices')
      .send({ title: 'Test', description: 'Desc', category: 'seguranca' });
    const postId = res.body.data.id;

    await authenticated(adminToken)
      .delete(`/api/posts/notices/${postId}`)
      .expect(200);
  });

  test('member cannot delete other member post', async () => {
    const adminPost = await authenticated(adminToken)
      .post('/api/posts/notices')
      .send({ title: 'Admin Post', description: 'Desc', category: 'seguranca' });
    const postId = adminPost.body.data.id;

    await authenticated(memberToken)
      .delete(`/api/posts/notices/${postId}`)
      .expect(403, res => {
      expect(res.body.error.code).toBe('FORBIDDEN');
    });
  });
});
```

---

## Naming Conventions

| Pattern | Example |
|---------|---------|
| File | `*.test.ts` in `tests/` |
| Describe | group by route + action: `'POST /api/auth/register'` |
| Test | `'should <expected behavior>'` or `'should reject/reject/forbid <condition>'` |
| Variables | `member`, `admin`, `superadmin` — role-level names in lowercase |

---

## Assertions

Assert on the shape of the response, not exact values (UUIDs, timestamps change):

```ts
// GOOD
expect(res.body.data).toHaveProperty('id');
expect(res.body.data).toHaveProperty('title', 'Test');

// BAD
expect(res.body.data.id).toBe('a1b2c3...'); // UUID changes per run
```

Use `.expect(status)` chaining for clean 404/401/403 assertions:

```ts
await authenticated(token).get('/api/resource/missing-id')
  .expect(404);
```

---

## What to Test (Priority)

| Priority | Area | What to verify |
|----------|------|----------------|
| **Critical** | Auth | Register, login, refresh, logout, token expiry |
| **Critical** | RBAC | Every admin-only route: admin can, member cannot |
| **High** | Posts | CRUD, soft delete, neighborhood scoping, expiration filter |
| **High** | Soft Deletes | After delete, `GET` must not return the resource |
| **Medium** | Membership | Join, leave, approve/reject requests, role changes |
| **Medium** | Statuses | Create, confirm, unconfirm, auto-expire |
| **Low** | Push | Skip — fire-and-forget, external dependency |