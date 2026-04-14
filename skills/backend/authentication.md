---
name: Backend Authentication
description: Use when implementing login, register, logout, JWT signing, refresh tokens, or the auth middleware in meubairro-backend
tags: [skill, auth, jwt, refresh-token, backend, how-to]
aliases: [Backend Auth Skill, JWT Skill]
---

# Backend Authentication — Meu Bairro

## Overview

Email + password authentication with JWT session management. All auth logic lives in `src/routes/auth.ts` and `src/middleware/auth.ts`.

---

## Registration Flow

```
1. User sends name, email + password
2. Validate: email format, uniqueness, password strength (Zod)
3. Hash password with bcrypt
4. Insert into `users` table
5. Generate Access Token (7 days) and Refresh Token (30 days)
6. Store Refresh Token in `refresh_tokens` table
7. Return { user, token, refresh_token }
```

**Endpoint:** `POST /api/auth/register` (public)

**Response (201):**
```json
{
  "data": {
    "user": { "id": "uuid", "name": "João", "email": "user@email.com" },
    "token": "jwt-token",
    "refresh_token": "refresh-token-string"
  }
}
```

---

## Login Flow

```
1. Find user by email (lowercase)
2. Compare password with stored hash (bcrypt)
3. Generate Access Token + Refresh Token
4. Store Refresh Token in DB
5. Return { user, token, refresh_token }
```

**Endpoint:** `POST /api/auth/login` (public)

---

## Refresh Flow

**Endpoint:** `POST /api/auth/refresh` (public)

```
1. Receive { refresh_token }
2. Find token in refresh_tokens table
3. Check: not revoked, not expired
4. Generate new Access Token
5. Return { token }
```

---

## Logout

**Endpoint:** `POST /api/auth/logout` (protected)

```
1. Receive { refresh_token }
2. Set revoked_at = now() in refresh_tokens table
3. Client clears both tokens from SecureStore
```

---

## JWT Token

**Payload (minimal):**
```json
{ "user_id": "uuid", "email": "user@email.com" }
```

**Rules:**
- Expiration: 7 days (Access), 30 days (Refresh)
- Signed with `JWT_SECRET` env var
- Never store sensitive data in token
- Send via: `Authorization: Bearer <token>`

---

## Auth Middleware (`src/middleware/auth.ts`)

Applied to all protected routes. Sets `req.user`.

```ts
function authMiddleware(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];

  if (!token) {
    return res.status(401).json({ error: { code: 'UNAUTHORIZED', message: 'Token não fornecido' } });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!);
    req.user = { id: decoded.user_id, email: decoded.email };
    next();
  } catch {
    return res.status(401).json({ error: { code: 'UNAUTHORIZED', message: 'Token inválido' } });
  }
}
```

---

## GET `/api/auth/me`

Key endpoint that drives post-login frontend routing.

**Response (200):**
```json
{
  "data": {
    "id": "uuid",
    "email": "user@email.com",
    "neighborhood_id": "uuid | null",
    "role": "membro | admin | superadmin | null",
    "pending_request": true | false
  }
}
```

---

## Security Rules

| Rule | Enforced By |
|------|-------------|
| Passwords hashed with bcrypt | Backend |
| Never return password in API responses | Backend |
| Validate token on every protected route | Backend |
| Never trust data from the client | Backend |

---

## Environment Variables

```env
JWT_SECRET=secure-random-key
DATABASE_URL=postgres://...
```
