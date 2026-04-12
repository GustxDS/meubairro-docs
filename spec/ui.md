---
name: Frontend Components and Navigation
description: Use when building screens, components, navigation flows, or any UI element in Meu Bairro (React Native + Expo Router + NativeWind)
tags: [spec, frontend, ui, components, navigation, react-native, nativewind, react-query]
aliases: [Frontend UI, UI Spec, Frontend Components Spec]
---

# Frontend Components and Navigation — Meu Bairro

## Stack

| Technology           | Role                                      |
|----------------------|-------------------------------------------|
| React Native         | Mobile runtime                            |
| Expo                 | Framework, build, native API access       |
| Expo Router          | File-based navigation                     |
| NativeWind           | Tailwind CSS styling for React Native     |
| RNR                  | Reusable base components (copy-paste)     |
| TanStack React Query | Server state, cache, optimistic updates   |
| expo-image           | High-performance image rendering          |
| TypeScript           | Static typing throughout                  |

---

## Navigation Structure

```
app/
├── _layout.tsx                       # Root Layout — auth guard, Sentry, ErrorBoundary, QueryClient
├── index.tsx                         # Splash / Redirect
├── (auth)/
│   ├── _layout.tsx                   # Stack navigator
│   ├── login.tsx
│   └── register.tsx
├── (onboarding)/
│   ├── _layout.tsx                   # Stack navigator
│   ├── index.tsx                     # Create or Join neighborhood
│   ├── create-neighborhood.tsx
│   ├── join-by-code.tsx
│   └── pending-approval.tsx
└── (app)/
    ├── _layout.tsx                   # Stack — notification handlers, offline banner
    ├── notifications/index.tsx       # Notification history screen
    └── (tabs)/
        ├── _layout.tsx               # Tab Navigator (4 tabs)
        ├── status/
        │   ├── _layout.tsx
        │   └── index.tsx             # Status do Bairro
        ├── notices/
        │   ├── _layout.tsx
        │   ├── index.tsx
        │   └── create.tsx
        ├── business/
        │   ├── _layout.tsx
        │   ├── index.tsx
        │   └── create.tsx
        └── profile/
            ├── _layout.tsx
            ├── index.tsx
            └── members.tsx           # Admin/Superadmin only
```

### Root Layout Decision Logic

```
No token          → (auth)/login
Token + no bairro → (onboarding)/index
Token + pending   → (onboarding)/pending-approval
Token + active    → (app)/(tabs)/status
```

Determined by calling `GET /api/auth/me` on app startup.

### App Layout (`app/(app)/_layout.tsx`)

Stack navigator wrapping the tabs + notifications screen. Handles:
- Push notification foreground listener (cache invalidation by action type)
- Notification tap listener (navigates to relevant tab)
- Offline banner (`useNetInfo` → displays "Sem conexão com a internet")

### Tab Navigator (`app/(app)/(tabs)/_layout.tsx`)

| Tab      | Icon       | Screen              |
|----------|------------|---------------------|
| Status   | Activity   | Status do Bairro    |
| Avisos   | Megaphone  | Feed de Avisos      |
| Negócios | Store      | Feed de Negócios    |
| Perfil   | User       | Perfil/Config       |

---

## Component Organization

```
components/
├── ui/           # RNR base components — never modify directly
│   └── index.ts  # barrel: Badge, Button, Card, Input, Separator, Text
├── custom/       # Project-specific components
│   └── index.ts  # barrel: Header, PostCard, StatusCard, NotificationBell…
└── index.ts      # root barrel (re-exports ui + custom)
```

Import from barrels: `import { Button, Text } from '@/components/ui'` or `import { Header, PostCard } from '@/components/custom'`. If custom behavior is needed, wrap — don't modify RNR files.

---

## Base Components (RNR — `components/ui/`)

| Component      | Variants / Notes                            |
|----------------|---------------------------------------------|
| Button         | primary, secondary, outline, destructive    |
| Input          | text, password, textarea                    |
| Card           | Standard container with padding and radius  |
| Badge          | Categories, roles, types                    |
| Avatar         | User avatar display                         |
| Dialog / Modal | Confirmations and destructive actions       |
| Select/Picker  | Dropdown selection                          |
| Tabs           | Tab-based content switching                 |

---

## Custom Components (`components/custom/`)

### StatusCard

```tsx
interface StatusCardProps {
  type: 'falta_agua' | 'falta_energia' | 'barulho_excessivo' | 'coleta_lixo';
  totalConfirmations: number;
  userConfirmed: boolean;
  active: boolean;
  onConfirm: () => void;
  onRemoveConfirmation: () => void;
}
```

**Rules:**
- Show icon + type name + confirmation count
- Toggle "Confirmar" / "Confirmado" based on `userConfirmed`
- If `active = false`: render with `opacity-50`, not interactive
- Social proof: "X pessoas confirmaram"

---

### PostCard

```tsx
interface PostCardProps {
  id: string;
  title: string;
  description: string;
  type: 'aviso' | 'negocio';
  category?: 'seguranca' | 'utilidade_publica' | 'eventos';
  businessType?: 'produto' | 'servico';
  imageUrl?: string | null;
  author: { id: string; email: string };
  createdAt: string;
  expiresAt: string;
  contact?: string;
  canDelete?: boolean;
  onDelete?: () => void;
}
```

**Rules:**
- Truncate description with "ver mais" link
- Category/type as colored Badge
- Remaining time via `ExpirationBadge`
- If `type = negocio`: show `WhatsAppButton`
- Delete button only when `canDelete = true`

---

### CategoryFilter

```tsx
interface CategoryFilterProps {
  selected: string;
  onSelect: (category: string) => void;
}
```

**Fixed categories:**
- `""` → "Todos"
- `"seguranca"` → "Segurança"
- `"utilidade_publica"` → "Utilidade Pública"
- `"eventos"` → "Eventos"

---

### ExpirationBadge

```tsx
interface ExpirationBadgeProps {
  expiresAt: string; // ISO timestamp
}
```

**Rules:**
- Format: "Xh restantes" / "Xmin restantes" / "X dias restantes"
- Amber color when < 1 hour remaining
- Red color when expired

---

### MemberRow

```tsx
interface MemberRowProps {
  id: string;
  email: string;
  role: 'membro' | 'admin' | 'superadmin';
  currentUserRole: 'admin' | 'superadmin';
  onChangeRole?: (newRole: string) => void;
  onRemove?: () => void;
}
```

**RBAC visibility (frontend reflects, backend enforces):**

| Action                | Admin | Superadmin |
|-----------------------|:-----:|:----------:|
| Approve join requests | ✓     | ✓          |
| Remove member         | ✓     | ✓          |
| Promote to admin      | ✓     | ✓          |
| Demote admin          | ✗     | ✓          |
| Remove admin          | ✗     | ✓          |
| Transfer superadmin   | ✗     | ✓          |
| Delete neighborhood   | ✗     | ✓          |

---

### WhatsAppButton

```tsx
interface WhatsAppButtonProps {
  contact: string;
  message?: string; // default: "Olá, vi seu anúncio no app do bairro e tenho interesse."
}
```

Deep link format:
```
https://wa.me/55${contact}?text=${encodeURIComponent(message)}
```

---

### EmptyState

```tsx
interface EmptyStateProps {
  icon?: string; // Lucide icon name
  title: string;
  description?: string;
}
```

---

### Header

```tsx
interface HeaderProps {
  title: string;
  subtitle?: string;
  rightElement?: React.ReactNode;
}
```

Used at the top of all main tab screens. Full width, standard padding.

---

### PendingBanner

```tsx
interface PendingBannerProps {
  onCancel?: () => void; // cancel join request
}
```

---

### NotificationBell

Bell icon with unread badge, used in screen headers.

```tsx
// No props — reads unread count from useUnreadCount() hook
// Navigates to /(app)/notifications on tap
// Badge capped at 99+
```

---

### ImagePickerField

Gallery picker with image preview and remove button.

```tsx
type ImagePickerFieldProps = {
  uri: string | null;
  onPick: () => void;
  onRemove: () => void;
};
```

- Empty state: dashed border + "Adicionar foto" placeholder
- With image: full preview + X button overlay to remove

---

### CreateScreenLayout

Reusable layout wrapper for create/edit form screens.

```tsx
type CreateScreenLayoutProps = {
  title: string;
  children: React.ReactNode;
};
```

- SafeAreaView + KeyboardAvoidingView + ScrollView
- Back button + heading title
- Standard padding and gap

---

## Design System (NativeWind)

### Color Palette

```
Primary:     emerald-600 / emerald-500
Secondary:   slate-700 / slate-600
Background:  white / slate-50
Surface:     white (cards)
Text:        slate-900 / slate-600
Error:       red-500
Success:     green-500
Warning:     amber-500
```

### Typography

| Usage    | Classes                          |
|----------|----------------------------------|
| Heading  | `font-bold text-lg/xl/2xl`       |
| Body     | `font-normal text-base`          |
| Caption  | `font-normal text-sm text-slate-500` |

### Spacing

- Screen padding: `px-4 py-6`
- Card gap: `gap-3` or `gap-4`
- Card border radius: `rounded-xl`

### Visual States

| State    | Style                                      |
|----------|--------------------------------------------|
| Disabled | `opacity-50`, not interactive              |
| Loading  | Skeleton screen or inline spinner          |
| Error    | Red border + error message below field     |
| Success  | Check icon + green color                   |

---

## Data Fetching (React Query)

### Pattern for a feed screen

```tsx
const { data, isLoading, refetch } = useQuery({
  queryKey: ['notices', neighborhoodId],
  queryFn: () => api.get('/posts/notices').then(r => r.data.data),
});
```

### Invalidate after mutation

```tsx
const queryClient = useQueryClient();

const mutation = useMutation({
  mutationFn: (data) => api.post('/posts/notices', data),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['notices', neighborhoodId] });
  },
});
```

### Optimistic update (StatusCard)

```tsx
useMutation({
  mutationFn: () => api.post(`/status/${type}/confirm`),
  onMutate: async () => {
    await queryClient.cancelQueries({ queryKey: ['status', neighborhoodId] });
    const previous = queryClient.getQueryData(['status', neighborhoodId]);
    queryClient.setQueryData(['status', neighborhoodId], (old) => /* update */);
    return { previous };
  },
  onError: (_, __, context) => {
    queryClient.setQueryData(['status', neighborhoodId], context?.previous);
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['status', neighborhoodId] });
  },
});
```

---

## Key Patterns

- **Forms**: Always use `KeyboardAvoidingView`
- **Lists**: Always support `pull-to-refresh` via `refetch`
- **Images**: Always use `expo-image` (not native `<Image>`) for aggressive disk caching
- **Tokens**: Always `expo-secure-store`, never `AsyncStorage`
- **HTTP**: All requests through `lib/api.ts` (centralized Axios instance)
- **Screen width target**: 360–430px (mobile-first, single column)

---

## New Component Checklist

1. Create `components/custom/YourComponent.tsx`
2. Define TypeScript interface for all props
3. Import base primitives from `'../ui'` (relative sub-barrel)
4. Use NativeWind classes for all styling
5. Follow color palette and spacing conventions
6. Handle all states: loading, error, empty, disabled
7. One responsibility per component
8. Export from `components/custom/index.ts` barrel
