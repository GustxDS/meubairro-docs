---
name: Frontend Forms
description: Use when building form screens with validation, image upload, or submit state management in the meubairro React Native app
tags: [skill, frontend, forms, react-hook-form, zod, validation, how-to]
aliases: [Forms Skill, Validation Skill, Form Skill]
---

# Frontend Forms — Meu Bairro

## Stack

| Technology | Role |
|------------|------|
| `react-hook-form` | Form state and submission |
| `@hookform/resolvers/zod` | Connects Zod schema to RHF |
| `zod` | Schema definition and validation |
| `useImagePicker` hook | Image selection from device gallery |
| `ImagePickerField` component | Image picker UI (preview + remove) |
| `CreateScreenLayout` component | Reusable form screen wrapper (SafeArea + keyboard + back nav) |
| `useMutation` (React Query) | API call with loading/error state |

---

## Form Architecture

Form logic is **extracted into dedicated hooks** — screens are thin wrappers that compose the hook with UI components.

| Hook | Screen | Purpose |
|------|--------|---------|
| `hooks/use-create-notice.ts` | `app/(app)/(tabs)/notices/create.tsx` | Notice creation (title, description, category, image, expiration) |
| `hooks/use-create-business.ts` | `app/(app)/(tabs)/business/create.tsx` | Business listing (title, description, type, contact, image) |

Each hook encapsulates: Zod schema, `useForm`, `useMutation`, image upload, error state, and returns everything the screen needs.

---

## Standard Form Pattern

See `hooks/use-create-notice.ts` as the canonical example.

### 1. Define a Zod schema

```ts
import { z } from 'zod';

const noticeSchema = z.object({
  title: z.string().min(3, 'Título deve ter pelo menos 3 caracteres'),
  description: z.string().min(10, 'Descrição deve ter pelo menos 10 caracteres'),
  category: z.enum(['seguranca', 'utilidade_publica', 'eventos']),
  image_url: z.string().optional(),
});

type NoticeForm = z.infer<typeof noticeSchema>;
```

Frontend schemas mirror backend Zod schemas. Error messages should be user-facing Portuguese strings.

### 2. Initialize `useForm`

```ts
const {
  control,
  handleSubmit,
  setValue,
  formState: { errors },
} = useForm<NoticeForm>({
  resolver: zodResolver(noticeSchema),
  defaultValues: { title: '', description: '', category: 'seguranca' },
});
```

### 3. Wrap inputs with `Controller`

```tsx
<Controller
  control={control}
  name="title"
  render={({ field: { onChange, value } }) => (
    <Input
      label="Título"
      placeholder="Ex: Rua interditada"
      value={value}
      onChangeText={onChange}
      error={errors.title?.message}  // inline error below field
    />
  )}
/>
```

Pass `error={errors.field?.message}` — the `Input` component renders it as red text below the field.

### 4. Submit with `useMutation`

```ts
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
    setError(err.response?.data?.error?.message ?? 'Erro ao criar aviso');
  },
});

async function onSubmit(data: NoticeForm) {
  setError('');
  createMutation.mutate(data);
}
```

### 5. Submit button with loading state

```tsx
<Button
  label={isUploading ? 'Fazendo upload...' : 'Publicar'}
  onPress={handleSubmit(onSubmit)}
  isLoading={createMutation.isPending || isUploading}
/>
```

The button is automatically disabled while `isLoading` is true.

---

## Image Upload

### Hook: `useImagePicker` (`hooks/useImagePicker.ts`)

```ts
const { pickImage, upload, isUploading, error: uploadError } = useImagePicker();

// Pick from gallery (uses expo-image-picker, quality 0.8)
const uri = await pickImage();
if (uri) setLocalImageUri(uri);

// Upload to backend before submit
const uploadedUrl = await upload(localImageUri);
if (!uploadedUrl) return; // upload failed, error already set
```

`upload()` calls `POST /api/uploads` via `lib/api.ts`'s `uploadImage()` helper (multipart/form-data). Returns the server-side URL or `null` on failure.

### Component: `ImagePickerField` (`components/custom/image-picker-field.tsx`)

Wraps the image picker UX into a single component:

```tsx
<ImagePickerField
  uri={localImageUri}
  onPick={pickImage}                            // from useImagePicker
  onRemove={() => setLocalImageUri(null)}
/>
```

- Empty state: dashed border + "Adicionar foto" placeholder
- With image: full preview via `expo-image` + X button to remove

---

## Error Display

### Inline field errors
```tsx
error={errors.fieldName?.message}
```
Rendered by the `Input` component as `text-red-500 text-sm` below the field.

### API / top-level errors
```tsx
const [error, setError] = useState('');

{error !== '' && (
  <Text className="text-center text-sm text-red-500">{error}</Text>
)}
```

Place this just above the submit button.

---

## Keyboard Handling

Wrap the scroll view to avoid the keyboard hiding inputs:

```tsx
<KeyboardAvoidingView
  className="flex-1"
  behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
>
  <ScrollView keyboardShouldPersistTaps="handled">
    {/* form fields */}
  </ScrollView>
</KeyboardAvoidingView>
```

`keyboardShouldPersistTaps="handled"` ensures tapping a button while the keyboard is open doesn't dismiss the keyboard before the tap registers.

---

## Non-input Form Controls (e.g. category picker)

For fields that aren't text inputs (button groups, selects), use `setValue` directly:

```ts
function selectCategory(value: string) {
  setSelectedCategory(value);            // local UI state
  setValue('category', value as ...);    // sync to RHF
}
```

---

## Files

| File | Role |
|------|------|
| `hooks/use-create-notice.ts` | Notice form hook (schema, mutation, image upload, expiration) |
| `hooks/use-create-business.ts` | Business form hook (schema, mutation, image upload) |
| `app/(app)/(tabs)/notices/create.tsx` | Notice creation screen (thin UI wrapper) |
| `app/(app)/(tabs)/business/create.tsx` | Business creation screen (thin UI wrapper) |
| `hooks/useImagePicker.ts` | Image pick + upload abstraction |
| `components/custom/image-picker-field.tsx` | Image picker UI component |
| `components/custom/create-screen-layout.tsx` | Reusable form screen layout |
| `lib/api.ts` → `uploadImage()` | Multipart upload helper |
| `components/ui/input.tsx` | Input with built-in error prop |
