---
name: Frontend Data Fetching
description: Use when writing queries, mutations, or cache invalidation with React Query in the meubairro React Native app
tags: [skill, frontend, react-query, data-fetching, cache, how-to]
aliases: [React Query Skill, Data Fetching Skill, Query Skill]
---

# Frontend Data Fetching — Meu Bairro

## Stack

| Technology | Role |
|------------|------|
| `@tanstack/react-query` v5 | Server state management |
| `axios` (via `lib/api.ts`) | HTTP client |
| `focusManager` | Ties React Query refetch to AppState |

---

## QueryClient Config (`app/_layout.tsx`)

```ts
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: 2,
      staleTime: 1000 * 60 * 5, // Data is fresh for 5 minutes
      refetchOnWindowFocus: true, // Auto-refetch on app resume (AppState 'active')
    },
  },
});
```

`refetchOnWindowFocus` is wired to React Native's `AppState` via `focusManager`:

```ts
function onAppStateChange(status: AppStateStatus) {
  if (Platform.OS !== 'web') {
    focusManager.setFocused(status === 'active');
  }
}
AppState.addEventListener('change', onAppStateChange);
```

This means every time the user brings the app to the foreground, stale queries are automatically re-fetched — no manual pull-to-refresh required for basic freshness.

---

## Query Key Conventions

Query keys are arrays. Always include scoping identifiers to avoid cross-user cache collisions.

| Data | Query Key |
|------|-----------|
| Neighborhood info | `['neighborhood', user?.neighborhood_id]` |
| Avisos feed | `['notices']` |
| Negócios feed | `['business']` |
| Status list | `['status', neighborhoodId]` |
| Members list | `['members']` |
| Notifications | `['notifications']` |

---

## Writing a Query

```ts
import { useQuery } from '@tanstack/react-query';
import api from '@/lib/api';
import { useAuth } from '@/contexts/auth-context';

export function useNeighborhood() {
  const { user, token } = useAuth();

  const { data, isLoading } = useQuery({
    queryKey: ['neighborhood', user?.neighborhood_id],
    queryFn: async () => {
      const { data } = await api.get('/neighborhoods/current');
      return data.data;
    },
    enabled: !!token && !!user?.neighborhood_id, // don't run if not authenticated
  });

  return { data, isLoading };
}
```

**Rules:**
- Always set `enabled` to prevent queries from running before auth is ready
- Return `data.data` — the API wraps everything in `{ data: ... }`
- Extract the hook to `hooks/use-<resource>.ts` when reused across screens

---

## Writing a Mutation

```ts
import { useMutation, useQueryClient } from '@tanstack/react-query';

const queryClient = useQueryClient();

const createMutation = useMutation({
  mutationFn: async (payload: NoticeForm) => {
    await api.post('/posts/notices', payload);
  },
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['notices'] });
    router.back();
  },
  onError: (err: ApiError) => {
    setError(err.response?.data?.error?.message ?? 'Erro inesperado');
  },
});
```

Always invalidate the relevant query key on `onSuccess` so the feed re-fetches the new data.

---

## Cache Invalidation from Push Notifications

When a push notification arrives, the app invalidates the relevant query instead of navigating or re-fetching manually. Implemented in `app/(app)/_layout.tsx`:

```ts
Notifications.addNotificationReceivedListener((notification) => {
  const { action, neighborhoodId, postType } = notification.request.content.data;

  switch (action) {
    case 'STATUS_EXPIRED':
      queryClient.invalidateQueries({ queryKey: ['status', neighborhoodId] });
      break;
    case 'POST_CREATED':
      if (postType === 'aviso') queryClient.invalidateQueries({ queryKey: ['notices'] });
      if (postType === 'negocio') queryClient.invalidateQueries({ queryKey: ['business'] });
      break;
  }
  // Always refresh notification list so badge updates in real time
  queryClient.invalidateQueries({ queryKey: ['notifications'] });
});
```

This is the primary real-time update mechanism — no WebSockets needed for the MVP.

---

## Dedicated Query Hooks

Form logic and data queries are extracted into reusable hooks:

| Hook | File | Key exports |
|------|------|-------------|
| `useNeighborhood` | `hooks/use-neighborhood.ts` | Neighborhood query |
| `useNotifications` | `hooks/use-notifications.ts` | Notifications list query |
| `useUnreadCount` | `hooks/use-notifications.ts` | Unread count (derived from notifications) |
| `useMarkAllRead` | `hooks/use-notifications.ts` | Mutation: PATCH /notifications/read-all |
| `useCreateNotice` | `hooks/use-create-notice.ts` | Full form state + mutation for notices |
| `useCreateBusiness` | `hooks/use-create-business.ts` | Full form state + mutation for business |

---

## Pull-to-Refresh

Use React Query's `refetch` returned from `useQuery`:

```tsx
const { data, isLoading, refetch } = useQuery({ ... });

<FlatList
  data={data}
  refreshControl={
    <RefreshControl refreshing={isLoading} onRefresh={refetch} />
  }
/>
```

---

## Error Handling in Queries

React Query surfaces errors via `isError` and `error`. Display inline or let the `ErrorFallback` boundary catch uncaught throws:

```tsx
const { data, isError, error } = useQuery({ ... });

if (isError) {
  return <EmptyState title="Erro ao carregar" description={(error as ApiError).message} />;
}
```

---

## Files

| File | Role |
|------|------|
| `app/_layout.tsx` | QueryClient setup, AppState focus management |
| `app/(app)/_layout.tsx` | Notification-triggered cache invalidation |
| `hooks/use-neighborhood.ts` | Neighborhood query hook |
| `hooks/use-notifications.ts` | Notifications query + unread count + mark-all-read |
| `hooks/use-create-notice.ts` | Notice form + mutation hook |
| `hooks/use-create-business.ts` | Business form + mutation hook |
| `lib/api.ts` | Axios instance used in all queryFns |
