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
| `useMutation` (React Query) | API call with loading/error state |

---

## Standard Form Pattern

See `app/(app)/notices/create.tsx` as the canonical example.

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

Use the `useImagePicker` hook (`hooks/useImagePicker.ts`):

```ts
const { pickImage, upload, isUploading, error: uploadError } = useImagePicker();

// Pick from gallery
const uri = await pickImage();
if (uri) setLocalImageUri(uri);

// Upload to backend before submit
const uploadedUrl = await upload(localImageUri);
if (!uploadedUrl) return; // upload failed, error already set
```

`upload()` calls `POST /api/uploads` via `lib/api.ts`'s `uploadImage()` helper (multipart/form-data). Returns the server-side URL or `null` on failure.

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
| `app/(app)/notices/create.tsx` | Aviso creation — canonical form example |
| `app/(app)/business/create.tsx` | Negócio creation — includes contact + business_type |
| `hooks/useImagePicker.ts` | Image pick + upload abstraction |
| `lib/api.ts` → `uploadImage()` | Multipart upload helper |
| `components/ui/input.tsx` | Input with built-in error prop |
