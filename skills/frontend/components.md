---
name: Frontend Components
description: How to build reusable React Native components following the Meu Bairro design system (NativeWind + RNR)
tags: [skill, frontend, components, nativewind, react-native, design-system, how-to]
aliases: [Frontend Components Skill, Component Skill, UI Skill]
---

# Frontend Components — Meu Bairro

## Stack

| Technology     | Role                                      |
|----------------|-------------------------------------------|
| React Native   | Mobile runtime                            |
| Expo           | Framework, build, native API access       |
| Expo Router    | File-based navigation                     |
| NativeWind     | Tailwind CSS styling for React Native     |
| RNR            | Reusable base components (copy-paste)     |
| TypeScript     | Static typing throughout                  |

---

## Component Organization

```
components/
├── ui/           # RNR base components (copied from library)
│   └── index.ts  # barrel — re-exports all ui primitives
├── custom/       # Project-specific components
│   └── index.ts  # barrel — re-exports all custom components
└── index.ts      # root barrel — re-exports everything
```

- **`ui/`**: Never modify RNR components in-place. If you need custom behavior, wrap them in a custom component.
- **`custom/`**: All project-specific components go here. Import UI primitives from `'../ui'` (relative path).

---

## Import Conventions

| Context | Import pattern | Example |
|---|---|---|
| Screen files (`app/`) | Sub-barrel with `@/` alias | `import { Button, Text } from '@/components/ui'` |
| Screen files (`app/`) | Sub-barrel with `@/` alias | `import { Header, PostCard } from '@/components/custom'` |
| Custom components (`components/custom/`) | Relative sub-barrel | `import { Text } from '../ui'` |
| Sibling custom imports | Direct relative path | `import { ExpirationBadge } from './expiration-badge'` |

**Important:** Custom components must **never** import from `@/components/custom` — that would be circular (the barrel re-exports them). Always use a direct relative path for sibling imports.

---

## Base Components (from RNR — `components/ui/`)

| Component      | Variants / Notes                                        |
|----------------|---------------------------------------------------------|
| Button         | primary, secondary, outline, destructive                |
| Input          | text, password, textarea                                |
| Card           | Standard container with padding and rounded corners     |
| Badge          | Used for categories, roles, types                       |
| Avatar         | User avatar display                                     |
| Dialog / Modal | For confirmations and destructive actions               |
| Select / Picker| Dropdown selection                                      |
| Tabs           | Tab-based content switching                             |

---

## Custom Components (in `components/custom/`)

### StatusCard

Card for the real-time neighborhood status feature.

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
- Toggle between "Confirmar" / "Confirmado" based on `userConfirmed`
- If expired (`active = false`), render in a disabled/gray state (opacity 50%, not interactive)
- Display social proof: "X pessoas confirmaram"

---

### PostCard

Reusable card for both Avisos and Negócios feeds.

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
- Show category/type as a colored Badge
- Show author email and creation date
- Show remaining time via `ExpirationBadge`
- Show image if present
- If `type = negocio`, show `WhatsAppButton` at the bottom

---

### CategoryFilter

Horizontal filter bar for the Avisos feed.

```tsx
interface CategoryFilterProps {
  selected: string;
  onSelect: (category: string) => void;
  categories: Array<{ value: string; label: string }>;
}
```

**Fixed categories for Avisos:**
- `""` → "Todos"
- `"seguranca"` → "Segurança"
- `"utilidade_publica"` → "Utilidade Pública"
- `"eventos"` → "Eventos"

---

### ExpirationBadge

Badge showing remaining time until a post/status expires.

```tsx
interface ExpirationBadgeProps {
  expiresAt: string; // ISO timestamp
}
```

**Rules:**
- Format: "Xh restantes", "Xmin restantes", "X dias restantes"
- Amber color when < 1 hour remaining
- Red color when expired

---

### MemberRow

Row component for the member management list.

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

**RBAC visibility rules (frontend only reflects, backend enforces):**

| Action                 | Admin | Superadmin |
|------------------------|:-----:|:----------:|
| See members            | ✓     | ✓          |
| Approve join requests  | ✓     | ✓          |
| Remove member          | ✓     | ✓          |
| Promote to admin       | ✓     | ✓          |
| Demote admin           | ✗     | ✓          |
| Remove admin           | ✗     | ✓          |
| Transfer superadmin    | ✗     | ✓          |

---

### EmptyState

```tsx
interface EmptyStateProps {
  icon?: string;
  title: string;
  description?: string;
}
```

---

### WhatsAppButton

"Tenho Interesse" button that opens WhatsApp with a pre-filled message.

```tsx
interface WhatsAppButtonProps {
  contact: string; // phone number
  message?: string; // defaults to: "Olá, vi seu anúncio no app do bairro e tenho interesse."
}
```

- Use `Linking.openURL` with: `https://wa.me/55${contact}?text=${encodeURIComponent(message)}`

---

### PendingBanner

Banner for the onboarding flow showing pending approval status.

```tsx
interface PendingBannerProps {
  onCancel?: () => void;
}
```

---

### Header

Universal top bar.

```tsx
interface HeaderProps {
  title: string;
  subtitle?: string;
  rightElement?: React.ReactNode;
}
```

---

## Design System (NativeWind Theme)

### Color Palette

```
Primary:       emerald-600 / emerald-500
Secondary:     slate-700 / slate-600
Background:    white / slate-50
Surface:       white (cards)
Text:          slate-900 / slate-600
Error:         red-500
Success:       green-500
Warning:       amber-500
```

### Typography

- **Headings**: `font-bold`, sizes `text-lg` / `text-xl` / `text-2xl`
- **Body**: `font-normal`, `text-base`
- **Caption**: `font-normal`, `text-sm`, `text-slate-500`

### Spacing

- Screen padding: `px-4 py-6`
- Card gap: `gap-3` or `gap-4`
- Card border radius: `rounded-xl`

### Visual States

| State    | Style                             |
|----------|-----------------------------------|
| Disabled | `opacity-50`, not interactive     |
| Loading  | Skeleton screen or inline spinner |
| Error    | Red border + error message below  |
| Success  | Check icon + green color          |

---

## Feedback Patterns

### Loading
- **Feeds**: Use skeleton screens
- **Buttons**: Show spinner inside button during actions
- **Lists**: Support `pull-to-refresh`

### Errors
- **Forms**: Inline messages below fields
- **Network**: Toast/Snackbar for network or action errors
- **Generic**: Error screen with "Tentar novamente" button

### Success
- **Actions**: Toast/Snackbar (e.g., "Post criado", "Membro aprovado")
- **Navigation**: Auto-redirect after successful actions

---

## Creating a New Component — Checklist

1. Create file in `components/custom/YourComponent.tsx`
2. Define a TypeScript interface for all props
3. Import base primitives from `'../ui'` (relative sub-barrel)
4. Use NativeWind classes (Tailwind) for all styling
5. Follow the color palette and spacing conventions above
6. Handle all visual states (loading, error, disabled, empty)
7. Export as default or named export consistently
8. Keep the component focused — one responsibility per component
