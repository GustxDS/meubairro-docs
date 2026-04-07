---
name: Frontend Components
description: How to build reusable React Native components following the Meu Bairro design system (NativeWind + RNR)
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
├── ui/          # RNR base components (copied from library)
└── custom/      # Project-specific components
```

- **`ui/`**: Never modify RNR components in-place. If you need custom behavior, wrap them in a custom component.
- **`custom/`**: All project-specific components go here. Always import base primitives from `../ui/`.

---

## Base Components (from RNR — `components/ui/`)

These are pre-built and copied into the project. Use them as building blocks:

| Component      | Variants / Notes                                        |
|----------------|---------------------------------------------------------|
| Button         | primary, secondary, outline, destructive                |
| Input          | text, password, textarea                                |
| Card           | Standard container with padding and rounded corners     |
| Badge          | Used for categories, roles, types                       |
| Avatar         | User avatar display                                    |
| Dialog / Modal | For confirmations and destructive actions               |
| Select / Picker| Dropdown selection                                     |
| Tabs           | Tab-based content switching                             |

---

## Custom Components (in `components/custom/`)

When creating a new component, follow these patterns:

### StatusCard

Card for the real-time neighborhood status feature.

```tsx
// components/custom/StatusCard.tsx
import { Card } from '~/components/ui/card';
import { Button } from '~/components/ui/button';
import { Badge } from '~/components/ui/badge';

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
  // Moderation
  canDelete?: boolean;
  onDelete?: () => void;
}
```

**Rules:**
- Truncate description with "ver mais" link
- Show category/type as a colored Badge
- Show author email
- Show creation date
- Show remaining time until expiration via `ExpirationBadge`
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
- Calculate remaining time from `now` to `expiresAt`
- Format as: "Xh restantes", "Xmin restantes", "X dias restantes"
- Use amber color when < 1 hour remaining
- Use red color when expired

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

| Action                 | Admin can do | Superadmin can do |
|------------------------|:------------:|:-----------------:|
| See members            | ✓            | ✓                 |
| Approve join requests  | ✓            | ✓                 |
| Remove member          | ✓            | ✓                 |
| Promote to admin       | ✓            | ✓                 |
| Demote admin           | ✗            | ✓                 |
| Remove admin           | ✗            | ✓                 |
| Transfer superadmin    | ✗            | ✓                 |

---

### EmptyState

Placeholder for empty lists.

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

**Rules:**
- Use `Linking.openURL` with WhatsApp deep link
- Format: `https://wa.me/55${contact}?text=${encodeURIComponent(message)}`

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

Universal Top Bar Header component.

```tsx
interface HeaderProps {
  title: string;
  subtitle?: string;
  rightElement?: React.ReactNode;
}
```

**Rules:**
- Used at the top of main tabs instead of inline Text.
- Takes up full width and uses standard padding.

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

| State      | Style                                         |
|------------|-----------------------------------------------|
| Disabled   | `opacity-50`, not interactive                 |
| Loading    | Skeleton screen or inline spinner             |
| Error      | Red border + error message below field        |
| Success    | Check icon + green color                      |

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
- **Actions**: Toast/Snackbar for confirmations (e.g., "Post criado", "Membro aprovado")
- **Navigation**: Auto-redirect after successful actions

---

## Creating a New Component — Checklist

1. Create file in `components/custom/YourComponent.tsx`
2. Define a TypeScript interface for all props
3. Import base primitives from `~/components/ui/`
4. Use NativeWind classes (Tailwind) for all styling
5. Follow the color palette and spacing conventions above
6. Handle all visual states (loading, error, disabled, empty)
7. Export as default or named export consistently
8. Keep the component focused — one responsibility per component
