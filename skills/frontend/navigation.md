---
name: Frontend Navigation
description: Use when adding new screens, route groups, deep links, or changing the auth guard and tab navigator in the meubairro React Native app
tags: [skill, frontend, navigation, expo-router, routing, how-to]
aliases: [Navigation Skill, Expo Router Skill, Routing Skill]
---

# Frontend Navigation — Meu Bairro

## Stack

| Technology | Role |
|------------|------|
| Expo Router | File-based routing (React Navigation under the hood) |
| `useRouter` | Programmatic navigation |
| `useSegments` | Read current route group for auth guard |

---

## File-Based Route Structure

```
app/
├── _layout.tsx                  ← Root layout: provider tree + auth guard
├── index.tsx                    ← Splash/initial redirect
├── (auth)/
│   ├── _layout.tsx              ← Stack navigator for auth screens
│   ├── login.tsx
│   └── register.tsx
├── (onboarding)/
│   ├── _layout.tsx              ← Stack navigator for onboarding
│   ├── index.tsx                ← Choose: create or join neighborhood
│   ├── create-neighborhood.tsx
│   ├── join-by-code.tsx
│   └── pending-approval.tsx
└── (app)/
    ├── _layout.tsx              ← Tab navigator (4 tabs)
    ├── status/index.tsx
    ├── notices/index.tsx
    ├── notices/create.tsx
    ├── business/index.tsx
    ├── business/create.tsx
    ├── profile/index.tsx
    └── profile/members.tsx
```

Route groups in parentheses (e.g. `(auth)`) are logical separators — they don't appear in the URL. Each `_layout.tsx` defines the navigator for that group.

---

## Auth Guard (`app/_layout.tsx`)

The root `RootLayoutNav` component reads from `useAuth()` and calls `router.replace()` based on state. This is the **only** place routing decisions are made.

```ts
const { user, token, isLoading } = useAuth();
const segments = useSegments(); // e.g. ['(app)', 'notices']

useEffect(() => {
  if (isLoading) return;

  const inAuthGroup = segments[0] === '(auth)';
  const inOnboardingGroup = segments[0] === '(onboarding)';

  if (!token) {
    if (!inAuthGroup) router.replace('/(auth)/login');
  } else if (user) {
    if (!user.neighborhood_id && user.pending_request) {
      if (!inOnboardingGroup) router.replace('/(onboarding)/pending-approval');
    } else if (!user.neighborhood_id) {
      if (!inOnboardingGroup) router.replace('/(onboarding)');
    } else {
      if (inAuthGroup || inOnboardingGroup) router.replace('/(app)/status');
    }
  }
}, [token, user, isLoading, segments]);
```

**Key points:**
- Always check `isLoading` first — auth state isn't ready until the stored token is resolved
- Use `router.replace()` not `router.push()` so the user can't back-navigate to a wrong state
- The `segments[0]` check avoids redundant redirects when already in the correct group

---

## Tab Navigator (`app/(app)/_layout.tsx`)

Four tabs, defined as `<Tabs.Screen>` entries. Each tab maps to a folder under `app/(app)/`.

| Tab | Folder | Icon |
|-----|--------|------|
| Status | `status/` | Activity |
| Avisos | `notices/` | Megaphone |
| Negócios | `business/` | Store |
| Perfil | `profile/` | User |

Tab bar config: `tabBarActiveTintColor: '#059669'` (emerald-600), white background, `height: 60 + insets.bottom`.

---

## Programmatic Navigation

```ts
import { useRouter } from 'expo-router';

const router = useRouter();

router.push('/(app)/notices/create');  // push onto stack
router.back();                          // go back
router.replace('/(app)/status');        // replace (no back)
```

Use `router.back()` after a successful form submission to return to the feed.

---

## Adding a New Screen

1. Create the file at the right path, e.g. `app/(app)/notices/[id].tsx` for a detail screen
2. If it needs its own stack header, add a `<Stack.Screen>` entry in the parent `_layout.tsx`
3. For admin-only screens (e.g. member management), check `user.role` and redirect if needed — the backend will still enforce the permission
4. The tab navigator auto-discovers screens in `app/(app)/` — no registration needed for new routes under existing tabs

---

## Deep Linking (Invite Links)

Invite links follow the pattern: `meubairro://join/<invite_token>`

- Handled via Expo Router's automatic deep link support
- The token is parsed with `useLocalSearchParams()` in the join screen
- App must be configured in `app.config.js` with the correct `scheme`

---

## Key Hooks

```ts
import { useRouter, useSegments, useLocalSearchParams } from 'expo-router';

const router = useRouter();           // programmatic navigation
const segments = useSegments();       // current route group array
const { id } = useLocalSearchParams(); // URL params (e.g. /notices/[id])
```
