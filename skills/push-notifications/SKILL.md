---
name: Push Notifications
description: End-to-end push notification flow in Meu Bairro — Expo push (dev/production), device token management, fire-and-forget pattern, cron-triggered sends
---

# Push Notifications — Meu Bairro

## Overview

Silent push notifications via Expo Server SDK. Fire-and-forget pattern — pushes enhance UX but are not required for app function. Clients use React Query's `refetchOnWindowFocus` as fallback.

**Current state:** Full notification support implemented — foreground banners, background alerts, tap-to-navigate, and visible pushes for posts. Silent pushes for expired statuses.
**Missing:** FCM (Android) + APNs (iOS) production credentials, `eas.json` setup.

---

## Architecture

```
Expo Push Service (proxy)
        ↑
   push.service.ts              ← Silent push, no title/body
        ↑
   cron.service.ts              ← Detects expired statuses
        ↑
  device.service.ts             ← Resolves user tokens by neighborhood
        ↑
  frontend usePushNotifications ← Registers token on app launch
```

---

## Stack

| Component | Role |
|-----------|------|
| Expo Server SDK (`expo-server-sdk`) | Server-side push sending |
| expo-notifications | Client-side notification handling |
| expo-server-dev-client | Dev mode push (development only) |
| user_devices table | Stores expo push tokens per user |

---

## Backend: Push Service

### PushAction Type

```ts
type PushAction =
  | { action: 'STATUS_EXPIRED'; neighborhoodId: string }  // Silent
  | { action: 'POST_CREATED'; neighborhoodId: string; postType: 'aviso' | 'negocio'; title?: string; body?: string }
  | { action: string; neighborhoodId: string; title?: string; body?: string; sound?: string };
```

- `STATUS_EXPIRED`: Silent (no title/body) — stays as `_contentAvailable: true` only
- `POST_CREATED`: Visible — includes `title` ("Novo Aviso"/"Novo Negócio") and `body` (description snippet)
- Custom actions may include `sound` and `title`/`body` for visible pushes

### Message Builder

```ts
const messages: ExpoPushMessage[] = tokens.map(token => ({
  to: token,
  data,
  title: 'title' in data ? data.title : undefined,
  body: 'body' in data ? data.body : undefined,
  sound: 'sound' in data ? data.sound : undefined,
  _contentAvailable: true,     // Always triggers background fetch
  priority: 'normal',
}));
```

---

## Backend

### Device Token Management

**Endpoint:** `POST /api/devices/token` (authenticated)

- Accepts `{ expoPushToken: string, platform: 'ios' | 'android' }`
- Upserts by `expo_push_token` (unique constraint)
- Updates `last_seen_at` on every call

**Endpoint:** `DELETE /api/devices/token` (authenticated)

- Accepts `{ expoPushToken: string }` (optional)
- Removes the token from `user_devices`
- Also triggered automatically on `DeviceNotRegistered` error

### Token Resolution

Tokens are resolved per neighborhood:

```ts
// deviceService.getTokensByNeighborhood
SELECT user_devices.expo_push_token
FROM user_devices
INNER JOIN neighborhood_members ON user_devices.user_id = neighborhood_members.user_id
WHERE neighborhood_members.neighborhood_id = ?
  AND neighborhood_members.deleted_at IS NULL;
```

**This means:** Only active neighborhood members receive pushes for that neighborhood.

### Cron Jobs

```ts
// Runs periodically, marks expired statuses as inactive
// and sends silent push to affected neighborhoods
processExpiredStatuses();
```

---

## Frontend

### Registration Flow

```ts
// usePushNotifications hook (called on app start)
1. Check if running on physical device
2. Request notification permissions
3. Get Expo push token
4. Send to backend: POST /api/devices/token
5. On Android: create 'default' notification channel (HIGH importance, vibrate enabled)
```

### Handling Incoming Notifications

**Implemented in `app/(app)/_layout.tsx`:**

1. **`setNotificationHandler`** — Shows in-app banners, plays sound, sets badge for foreground notifications
2. **`addNotificationReceivedListener`** — Invalidates React Query when notification arrives (data freshness)
3. **`addNotificationResponseReceivedListener`** — Taps navigate to correct tab:
   - `STATUS_EXPIRED` → `/status`
   - `POST_CREATED` (aviso) → `/notices`
   - `POST_CREATED` (negocio) → `/business`

**What happens when app is closed/background:**
- System handles the notification via OS notification tray
- User taps → app opens, expo-router boots, and the notification data is available via the response listener

---

## Production Setup (FCM + APNs)

Currently, Expo dev push works without credentials. For production, you need:

### FCM (Android)

1. Create a Firebase project at [console.firebase.google.com](https://console.firebase.google.com)
2. Add an Android app with bundle ID matching your Expo app
3. Download `google-services.json` — **not needed for push**, just for Firebase project linkage
4. Go to Project Settings → Cloud Messaging → copy **FCM Server Key**
5. Add to Expo EAS config:

```json
// eas.json
{
  "cli": {
    "version": ">= 3.0.0"
  },
  "build": {
    "production": {
      "android": {
        "credentials": {
          "fcm": {
            "projectName": "your-firebase-project",
            "fcmPushCredentials": "path-to-credentials.json"
          }
        }
      }
    }
  }
}
```

Or configure via Expo dashboard: `expo push:android:upload --api-key <FCM_KEY>`

### APNs (iOS)

1. Create an Apple Developer account (paid)
2. In [developer.apple.com](https://developer.apple.com), go to Certificates → Keys → Create New Key
3. Enable **Apple Push Notifications service (APNs)**
4. Download the `.p8` file
5. Note the Key ID and your Team ID
6. Upload to Expo: `eas credentials` → configure push notifications

### Env Vars (backend)

No changes needed on the Expo Server SDK side — it will route to FCM/APNs automatically once the iOS/Android keys are configured in Expo.

---

## Push Action Types (extending)

When adding a new push type, extend the union in `push.service.ts`:

```ts
export type PushAction =
  | { action: 'STATUS_EXPIRED'; neighborhoodId: string }
  | { action: 'POST_CREATED'; neighborhoodId: string; postType: 'aviso' | 'negocio'; title?: string; body?: string }
  | { action: 'MEMBER_JOINED'; neighborhoodId: string; role: string; title?: string; body?: string };
```

Each new action must be handled in both frontend listeners (invalidation and response listener in `app/(app)/_layout.tsx`).

---

## Rules

| Rule | Why |
|------|-----|
| Always fire-and-forget | Push is an enhancement, not core functionality |
| Always validate tokens with `Expo.isExpoPushToken()` | Invalid tokens cause send failures |
| Process tickets for `DeviceNotRegistered` | Keeps DB clean, stops sending to dead devices |
| Scope pushes by neighborhood | Users should only get relevant notifications |
| Never block route on push success | User experience must not depend on push delivery |
| Chunk messages at 100 | Expo push API limit per batch |
| Never send title/body in silent pushes | Silent pushes should wake the app, not show UI notification |

---

## Current Gaps

| Gap | Priority | Notes |
|-----|----------|-------|
| FCM + APNs credentials | Critical | Required for production builds; dev push already works |
| `eas.json` + projectId setup | High | `eas init` needed in `app.config.js` |
| Push analytics/metrics | Low | No tracking of delivery success/failure rates |
| Topic-based subscriptions | Low | Current per-token approach works; no need for Expo topics |
| Notification preferences | Low | Future: per-user opt-in/out for certain push types |