---
name: Frontend Error Handling
description: Use when adding error boundaries, logging errors, showing toasts, or handling offline state in the meubairro React Native app
tags: [skill, frontend, errors, sentry, logging, toast, offline, how-to]
aliases: [Error Handling Skill, Logging Skill, Observability Skill]
---

# Frontend Error Handling — Meu Bairro

## Layers

| Layer | Tool | What it catches |
|-------|------|-----------------|
| Render errors | `react-error-boundary` + `ErrorFallback` | Thrown exceptions during React render |
| Runtime errors | Sentry | All unhandled exceptions in production |
| Logging | `lib/logger.ts` | Structured dev logs / prod error tracking |
| User feedback | `lib/toast.ts` (`showToast`) | Success confirmations, API error messages |
| Connectivity | `use-net-info.ts` | Offline state banner + mutation guard |

---

## Error Boundary (`app/_layout.tsx`)

The entire app is wrapped in a single `ErrorBoundary` at the root:

```tsx
import { ErrorBoundary } from 'react-error-boundary';
import { ErrorFallback } from '@/components/custom/error-fallback';

<ErrorBoundary
  FallbackComponent={ErrorFallback}
  onError={(error, info) => {
    Sentry.captureException(error, { extra: { componentStack: info.componentStack } });
  }}
  onReset={() => {
    queryClient.clear(); // clear all stale query cache on retry
  }}
>
  {/* entire app tree */}
</ErrorBoundary>
```

`ErrorFallback` renders a "Tentar novamente" button that calls `resetErrorBoundary()`, which triggers `onReset`.

---

## Sentry (`app/_layout.tsx`)

```ts
Sentry.init({
  dsn: Constants.expoConfig?.extra?.sentryDsn,
  enabled: !__DEV__,               // silent in development
  environment: __DEV__ ? 'development' : 'production',
  tracesSampleRate: 0.2,
});

export default Sentry.wrap(RootLayout); // wraps the root component
```

User identity is set/cleared by the auth context:

```ts
// On login:
Sentry.setUser({ id: user.id, email: user.email });

// On logout:
Sentry.setUser(null);
```

---

## Logger (`lib/logger.ts`)

Thin wrapper — logs to `console` in development, sends to Sentry in production.

```ts
export const logger = {
  log: (...args: unknown[]) => {
    if (__DEV__) console.log(...args);
  },
  warn: (...args: unknown[]) => {
    if (__DEV__) console.warn(...args);
  },
  error: (message: string, error?: unknown) => {
    if (__DEV__) console.error(message, error);
    else Sentry.captureException(error ?? new Error(message));
  },
};
```

**Usage:**
```ts
import { logger } from '@/lib/logger';

logger.error('Error during backend logout', err);
```

Never use `console.log/warn/error` directly — use `logger` so production errors are captured automatically.

---

## Toast Notifications (`lib/toast.ts`)

Wraps the `burnt` library for native toast on both iOS and Android.

```ts
import { showToast } from '@/lib/toast';

showToast.success('Aviso publicado!');
showToast.error('Erro ao criar aviso');
showToast.info('Link copiado');
```

**Available presets:** `success` (checkmark), `error` (red), `info` (neutral). All show for 3 seconds.

**When to use toast:**
- After a successful mutation (create, delete, approve)
- On API error from a mutation's `onError` callback
- **Not** for query loading states (use skeleton/spinner instead)

---

## Offline Detection (`hooks/use-net-info.ts`)

```ts
const { isConnected } = useNetInfo();
```

The app layout (`app/(app)/_layout.tsx`) renders an offline banner at the top when `!isConnected`:

```tsx
{!isConnected && (
  <View style={{ backgroundColor: '#1e293b', flexDirection: 'row', ... }}>
    <WifiOff color="#fff" size={13} />
    <Text style={{ color: '#fff', ... }}>Sem conexão com a internet</Text>
  </View>
)}
```

Mutations should check `isConnected` before submitting to give immediate feedback instead of waiting for a network timeout.

---

## API Error Shape

Errors from Axios mutations carry this shape (see `types/index.ts`):

```ts
type ApiError = {
  response?: {
    data?: {
      error?: {
        code: string;
        message: string;
      };
    };
  };
};
```

Extract the user-facing message:

```ts
onError: (err: ApiError) => {
  setError(err.response?.data?.error?.message ?? 'Erro inesperado');
},
```

---

## Files

| File | Role |
|------|------|
| `app/_layout.tsx` | ErrorBoundary, Sentry init, Sentry.wrap |
| `lib/logger.ts` | Structured logger (dev console / prod Sentry) |
| `lib/toast.ts` | Toast helper (`showToast.success/error/info`) via `burnt` |
| `hooks/use-net-info.ts` | Network connectivity hook |
| `components/custom/error-fallback.tsx` | Fallback UI with retry button |
