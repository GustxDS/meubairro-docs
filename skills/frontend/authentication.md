---
name: Frontend Authentication
description: Use when working with auth state, token storage, the Axios interceptor, or post-login navigation routing in the meubairro React Native app
tags: [skill, auth, frontend, securestore, axios, context, how-to]
aliases: [Frontend Auth Skill, Auth Context Skill]
---

# Frontend Authentication — Meu Bairro

## Overview

Auth state lives in `contexts/auth-context.tsx`. All HTTP goes through the Axios instance in `lib/api.ts`, which injects the JWT and silently refreshes on 401.

---

## Token Storage

Tokens are stored in `expo-secure-store` (encrypted native storage). **Never use AsyncStorage for tokens.**

```ts
// lib/api.ts
const TOKEN_KEY = 'auth_token';
const REFRESH_TOKEN_KEY = 'refresh_token';

// On native: SecureStore.setItemAsync / getItemAsync / deleteItemAsync
// On web:    localStorage (Expo Go web / testing only)
```

Helper functions exported from `lib/api.ts`:
- `saveToken(token)` / `getToken()` / `removeToken()`
- `saveRefreshToken(token)` / `getRefreshToken()` / `removeRefreshToken()`

---

## Axios Instance (`lib/api.ts`)

Single shared Axios instance with `baseURL = EXPO_PUBLIC_API_URL`.

### Request interceptor — injects Bearer token

```ts
api.interceptors.request.use(async (config) => {
  const token = Platform.OS === 'web'
    ? localStorage.getItem(TOKEN_KEY)
    : await SecureStore.getItemAsync(TOKEN_KEY);

  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});
```

### Response interceptor — silent refresh on 401

```ts
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;
      const refreshToken = await getRefreshToken();
      if (refreshToken) {
        const { data } = await axios.post(`${api.defaults.baseURL}/auth/refresh`, { refresh_token: refreshToken });
        await saveToken(data.data.token);
        originalRequest.headers.Authorization = `Bearer ${data.data.token}`;
        return api(originalRequest); // retry original request
      }
      // Refresh failed — clear tokens, auth context detects null and redirects to login
      await removeToken();
      await removeRefreshToken();
    }
    return Promise.reject(error);
  }
);
```

---

## Auth Context (`contexts/auth-context.tsx`)

Wraps the entire app via `<AuthProvider>` in `app/_layout.tsx`.

### State

| Property | Type | Description |
|----------|------|-------------|
| `user` | `User \| null` | Current user (id, email, neighborhood_id, role, pending_request) |
| `token` | `string \| null` | Access JWT |
| `isLoading` | `boolean` | True until stored token is checked on mount |

### Methods

| Method | Description |
|--------|-------------|
| `login(email, password)` | POST /auth/login → saves both tokens → calls fetchUser |
| `register(email, password)` | POST /auth/register → saves both tokens → sets user state |
| `logout()` | Deletes push token from backend → revokes refresh token → clears storage → nulls state |
| `refreshUser()` | Re-fetches GET /auth/me to sync user state (e.g. after joining a neighborhood) |

### On mount — `loadStoredAuth`

```ts
// Checks SecureStore for existing tokens on app start
// If found → calls fetchUser (GET /auth/me)
// If fetchUser fails → clears tokens (expired/invalid)
```

### Sentry integration

```ts
// On successful fetchUser:
Sentry.setUser({ id: data.id, email: data.email });

// On logout:
Sentry.setUser(null);
```

---

## Post-Login Routing (`app/_layout.tsx`)

`RootLayoutNav` watches `{ user, token, isLoading }` and routes accordingly:

```
token === null
  → router.replace('/(auth)/login')

token exists + user loaded:
  ├── no neighborhood_id AND pending_request === true
  │     → router.replace('/(onboarding)/pending-approval')
  ├── no neighborhood_id
  │     → router.replace('/(onboarding)')
  └── neighborhood_id exists (active member)
        → router.replace('/(app)/status')
```

The splash screen stays visible until `isLoading` is false (both font load and auth check complete).

---

## Usage Pattern

```tsx
import { useAuth } from '@/contexts/auth-context';

function MyScreen() {
  const { user, logout } = useAuth();

  return (
    <Text>Logged in as {user?.email}</Text>
  );
}
```

---

## Environment Variable

```env
# .env / app.config.js
EXPO_PUBLIC_API_URL=http://localhost:3000/api
```
