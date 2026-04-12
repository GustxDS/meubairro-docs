---
name: Frontend Storage
description: Use when persisting data on the device — which storage layer to use, when, and how to handle the web compatibility shim
tags: [skill, frontend, storage, securestore, mmkv, how-to]
aliases: [Storage Skill, SecureStore Skill, MMKV Skill]
---

# Frontend Storage — Meu Bairro

## Two Storage Layers

| Layer | Library | Use For | Encrypted |
|-------|---------|---------|-----------|
| SecureStore | `expo-secure-store` | Auth tokens (access + refresh) | Yes (OS keychain) |
| MMKV | `react-native-mmkv` via `lib/storage.ts` | App state, preferences, non-sensitive cache | Yes (AES-256) |

**Rule: tokens always go through SecureStore — never AsyncStorage, never MMKV.**

---

## SecureStore — Token Management

All SecureStore calls are encapsulated in `lib/api.ts`. Don't call SecureStore directly from components.

```ts
// lib/api.ts — the only place tokens are read/written
const TOKEN_KEY = 'auth_token';
const REFRESH_TOKEN_KEY = 'refresh_token';

export async function saveToken(token: string) { ... }
export async function getToken(): Promise<string | null> { ... }
export async function removeToken() { ... }

export async function saveRefreshToken(token: string) { ... }
export async function getRefreshToken(): Promise<string | null> { ... }
export async function removeRefreshToken() { ... }
```

SecureStore is async — all functions return Promises.

---

## Web Compatibility Shim

SecureStore doesn't work in Expo Go web or testing environments. `lib/api.ts` automatically falls back to `localStorage`:

```ts
export async function saveToken(token: string) {
  if (Platform.OS === 'web') {
    localStorage.setItem(TOKEN_KEY, token);
  } else {
    await SecureStore.setItemAsync(TOKEN_KEY, token);
  }
}
```

This shim is only relevant for web browser testing. Production builds (iOS/Android) always use SecureStore.

---

## MMKV — App State

For anything that isn't a sensitive secret (user preferences, lightweight cached data, feature flags), use MMKV via `lib/storage.ts`:

```ts
import { storage } from '@/lib/storage';

// Write
storage.set('onboarding_seen', true);

// Read
const seen = storage.getBoolean('onboarding_seen');

// Delete
storage.delete('onboarding_seen');
```

MMKV is synchronous — no `await` needed. It's significantly faster than AsyncStorage.

### `clientStorage` adapter

`lib/storage.ts` also exports a `clientStorage` object with the standard `setItem`/`getItem`/`removeItem` interface. This is used by:

- **Zustand persist middleware** — `use-permissions-store.ts` persists notification/location permission flags to MMKV via `createJSONStorage(() => clientStorage)`
- Can also be used for React Query persistence if needed

```ts
import { clientStorage } from '@/lib/storage';

// Standard interface: setItem(key, value), getItem(key), removeItem(key)
```

---

## What Goes Where

| Data | Storage Layer | Key |
|------|--------------|-----|
| JWT access token | SecureStore | `auth_token` |
| Refresh token | SecureStore | `refresh_token` |
| Expo push token | MMKV | `expo_push_token` |
| Permission flags | MMKV (Zustand persist) | `permissions-storage` |
| User preferences | MMKV | varies |
| Onboarding flags | MMKV | varies |
| React Query cache | In-memory (React Query manages this) | — |
| Images / files | Expo FileSystem (if needed) | — |

---

## Files

| File | Role |
|------|------|
| `lib/api.ts` | SecureStore helpers (saveToken, getToken, etc.) + web shim |
| `lib/storage.ts` | MMKV instance (`storage`) + standard adapter (`clientStorage`) |
| `contexts/auth-context.tsx` | Consumes token helpers from `lib/api.ts` + manages push token in MMKV |
| `hooks/use-permissions-store.ts` | Zustand store persisted to MMKV via `clientStorage` |
| `hooks/usePushNotifications.ts` | Stores `expo_push_token` in MMKV after registration |
