---
name: Backend Push Notifications
description: Use when adding server-side push logic — device token management, Expo Server SDK, cron-triggered sends, or fire-and-forget patterns in meubairro-backend
tags: [skill, notifications, push, expo, backend, how-to]
aliases: [Backend Push Skill, Push Service Skill]
---

# Backend Push Notifications — Meu Bairro

## Overview

Silent push notifications via Expo Server SDK. Fire-and-forget — pushes enhance UX but are not required for core function. Clients use React Query's `refetchOnWindowFocus` as fallback.

**Current state:** Full notification support implemented.  
**Missing:** FCM (Android) + APNs (iOS) production credentials, `eas.json` setup.

---

## Architecture

```
Expo Push Service (proxy)
        ↑
   push.service.ts              ← builds + sends messages
        ↑
   cron.service.ts              ← detects expired statuses
        ↑
  device.service.ts             ← resolves tokens by neighborhood
        ↑
  user_devices table            ← stores expo push tokens
```

---

## Stack

| Component | Role |
|-----------|------|
| `expo-server-sdk` | Server-side push sending |
| `user_devices` table | Stores expo push tokens per user |

---

## PushAction Type

```ts
type PushAction =
  | { action: 'STATUS_EXPIRED'; neighborhoodId: string }
  | { action: 'POST_CREATED'; neighborhoodId: string; postType: 'aviso' | 'negocio'; title?: string; body?: string }
  | { action: string; neighborhoodId: string; title?: string; body?: string; sound?: string };
```

- `STATUS_EXPIRED`: Silent (no title/body) — `_contentAvailable: true` only
- `POST_CREATED`: Visible — includes `title` ("Novo Aviso"/"Novo Negócio") and `body`

---

## Message Builder

```ts
const messages: ExpoPushMessage[] = tokens.map(token => ({
  to: token,
  data,
  title: 'title' in data ? data.title : undefined,
  body: 'body' in data ? data.body : undefined,
  sound: 'sound' in data ? data.sound : undefined,
  _contentAvailable: true,   // Always triggers background fetch
  priority: 'normal',
}));
```

---

## Device Token Management

**Endpoint:** `POST /api/devices/token` (authenticated)

- Accepts `{ expoPushToken: string, platform: 'ios' | 'android' }`
- Upserts by `expo_push_token` (unique constraint)
- Updates `last_seen_at` on every call

**Endpoint:** `DELETE /api/devices/token` (authenticated)

- Accepts `{ expoPushToken: string }` (optional)
- Removes the token from `user_devices`
- Also triggered automatically on `DeviceNotRegistered` error

---

## Token Resolution

Tokens are resolved per neighborhood — only active members receive pushes:

```ts
// deviceService.getTokensByNeighborhood
SELECT user_devices.expo_push_token
FROM user_devices
INNER JOIN neighborhood_members ON user_devices.user_id = neighborhood_members.user_id
WHERE neighborhood_members.neighborhood_id = ?
  AND neighborhood_members.deleted_at IS NULL;
```

---

## Cron Job

```ts
// Runs periodically — marks expired statuses as inactive
// and sends silent push to affected neighborhoods
processExpiredStatuses();
```

---

## Adding a New Push Action

Extend the union in `push.service.ts`:

```ts
export type PushAction =
  | { action: 'STATUS_EXPIRED'; neighborhoodId: string }
  | { action: 'POST_CREATED'; neighborhoodId: string; postType: 'aviso' | 'negocio'; title?: string; body?: string }
  | { action: 'MEMBER_JOINED'; neighborhoodId: string; role: string; title?: string; body?: string };
```

Each new action must also be handled in the frontend listeners — see `skills/frontend/push-notifications.md`.

---

## Production Setup

### FCM (Android)

1. Create a Firebase project
2. Go to Project Settings → Cloud Messaging → copy FCM Server Key
3. Upload via Expo: `expo push:android:upload --api-key <FCM_KEY>`

### APNs (iOS)

1. Create key at developer.apple.com → Certificates → Keys (enable APNs)
2. Download `.p8` file
3. Upload via: `eas credentials` → configure push notifications

No changes needed on the Expo Server SDK side — it routes to FCM/APNs automatically once keys are configured in Expo.

---

## Rules

| Rule | Why |
|------|-----|
| Always fire-and-forget | Push is enhancement, not core functionality |
| Always validate tokens with `Expo.isExpoPushToken()` | Invalid tokens cause send failures |
| Process `DeviceNotRegistered` tickets | Keeps DB clean, stops sending to dead devices |
| Scope pushes by neighborhood | Users should only get relevant notifications |
| Never block route handler on push success | User experience must not depend on push delivery |
| Chunk messages at 100 | Expo push API batch limit |
| Never send title/body in silent pushes | Silent pushes wake the app, not show UI notification |

---

## Current Gaps

| Gap | Priority |
|-----|----------|
| FCM + APNs credentials | Critical — required for production builds |
| `eas.json` + projectId setup | High — `eas init` needed in `app.config.js` |
| Push delivery analytics | Low |
