---
name: Authentication Flow
description: Use when implementing anything related to login, register, logout, JWT, refresh tokens, auth middleware, or post-login navigation routing in Meu Bairro
---

# Authentication Flow — Meu Bairro

## Overview

Email + password authentication with JWT session management. Authentication is required for all app features.

---

## Backend

### Registration Flow

```
1. User sends email + password
2. Validate: email format, uniqueness, password strength
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
    "user": { "id": "uuid", "email": "user@email.com" },
    "token": "jwt-token",
    "refresh_token": "refresh-token-string"
  }
}
```

---

### Login Flow

```
1. Find user by email (lowercase)
2. Compare password with stored hash (bcrypt)
3. Generate Access Token + Refresh Token
4. Store Refresh Token in DB
5. Return { user, token, refresh_token }
```

**Endpoint:** `POST /api/auth/login` (public)

---

### Refresh Flow

**Endpoint:** `POST /api/auth/refresh` (public)

```
1. Receive { refresh_token }
2. Find token in refresh_tokens table
3. Check: not revoked, not expired
4. Generate new Access Token
5. Return { token }
```

---

### Logout

**Endpoint:** `POST /api/auth/logout` (protected)

```
1. Receive { refresh_token }
2. Set revoked_at = now() in refresh_tokens table
3. Client clears both tokens from SecureStore
```

---

### JWT Token

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

### Auth Middleware (`src/middleware/auth.ts`)

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

### GET `/api/auth/me`

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

## Frontend

### Token Storage

- Store both tokens in `expo-secure-store` (encrypted native storage)
- Keys: `TOKEN_KEY` (access token), `REFRESH_TOKEN_KEY` (refresh token)
- **Never use AsyncStorage for tokens**

---

### Centralized HTTP Client (`lib/api.ts`)

All requests go through Axios instance with:
- `Authorization: Bearer <token>` header injected automatically
- **Axios Response Interceptor** for silent token refresh:

```
1. Response returns 401 (Access Token expired)
2. Interceptor pauses the request
3. Calls POST /api/auth/refresh with refresh_token
4. On success: saves new token, retries original request silently
5. On failure: clears tokens, redirects to login
```

---

### Auth Context (`contexts/auth.ts`)

Manages global auth state:

| Property / Method         | Description                                      |
|---------------------------|--------------------------------------------------|
| `user`                    | Current user data                                |
| `token`                   | Access Token JWT                                 |
| `isLoading`               | Auth state loading flag                          |
| `login(email, password)`  | Login and store both tokens                      |
| `register(email, password)`| Register and store both tokens                  |
| `logout()`                | Revoke refresh token, clear storage, redirect    |
| `refreshUser()`           | Re-fetch `/api/auth/me` to update state          |

---

### Post-Login Navigation (Root Layout `app/_layout.tsx`)

After login, call `GET /api/auth/me` and route based on state:

```
Token exists?
├── No  → (auth)/login
└── Yes → GET /api/auth/me
          ├── no neighborhood_id  → (onboarding)/index
          ├── pending_request     → (onboarding)/pending-approval
          └── active member       → (app)/status
```

---

## Security Rules

| Rule                                           | Enforced By |
|------------------------------------------------|-------------|
| Passwords hashed with bcrypt                   | Backend     |
| Never return password in API responses         | Backend     |
| Validate token on every protected route        | Backend     |
| Never trust data from the client               | Backend     |
| Token stored in SecureStore (not AsyncStorage) | Frontend    |
| Frontend only stores and sends the token       | Frontend    |

---

## Environment Variables

**Backend:**
```env
JWT_SECRET=secure-random-key
DATABASE_URL=postgres://...
```

**Frontend:**
```env
EXPO_PUBLIC_API_URL=http://localhost:3000/api
```