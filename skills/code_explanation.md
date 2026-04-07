---
name: Code Explanation
description: Use whenever the user asks you to explain code you generated, justify a technical decision, review if something is ideal or modern, or walk through why you did something a certain way — applies to both frontend and backend code in Meu Bairro
---

# Code Explanation — Meu Bairro

## Purpose

When explaining generated code, never assume the user already understands the reasoning behind it. Always explain:

1. **What** the code does (plain language summary)
2. **Why** it was written this way (the reasoning and tradeoffs)
3. **Whether it's ideal** (honest assessment — if there's a better approach, say so)
4. **What alternatives exist** (and why they were not chosen)
5. **What could evolve** (if the solution is MVP-appropriate but has known limitations)

---

## Explanation Structure

Always follow this structure when explaining code:

### 1. Plain Language Summary
Explain what the code does as if the user has never seen it before. No jargon without definition. One short paragraph.

### 2. Line-by-Line or Block-by-Block Walkthrough
Break the code into logical blocks and explain each one. Use inline comments or numbered callouts. Example:

```ts
// 1. Find the active status for this type and neighborhood
const [status] = await db.select().from(statuses).where(...)

// 2. If no active status exists, create one automatically
if (!status) { ... }

// 3. Prevent double confirmation via DB unique constraint
const [existing] = await db.select()...
if (existing) throw new AppError(...)

// 4. Increment counter atomically in SQL (safer than read → increment → write)
await db.update(statuses).set({
  total_confirmations: sql`${statuses.total_confirmations} + 1`
})
```

### 3. Why This Approach

Explain the reasoning. Common angles to cover:

- **Performance**: Does this avoid N+1 queries? Does it filter in SQL vs. in memory?
- **Safety**: Is this atomic? Could a race condition occur here?
- **Correctness**: Does this handle edge cases (expired records, missing data, duplicate requests)?
- **Simplicity**: Is this the simplest solution that solves the problem?

### 4. Is This Ideal / Modern?

Be honest. If it's good, say why. If it's a tradeoff, name the tradeoff. Examples:

- ✅ "This is the recommended pattern for Drizzle ORM — type-safe, no raw SQL needed."
- ⚠️ "This works well for MVP scale. At higher volume, you'd want a job queue instead of inline push sending."
- ❌ "This filters expired posts in JavaScript instead of SQL — fine for now, but should move to a WHERE clause for performance."

### 5. Alternatives Considered

List 1–3 alternatives and why they were not chosen:

| Alternative | Why Not Chosen |
|-------------|----------------|
| Using a cron job to pre-expire records | Adds infrastructure complexity; query-time filtering is simpler for MVP |
| Storing state in Redis | Overkill for current scale; PostgreSQL handles this fine |
| Using WebSockets instead of Silent Push | Requires persistent connection management; Silent Push is stateless and battery-friendly |

### 6. What Could Evolve (if applicable)

If the code is intentionally simplified for MVP, say so explicitly and describe what a production version would look like. Example:

> "For the MVP, push failures are logged and swallowed silently. In production, you'd want a dead-letter queue or retry mechanism to track failed deliveries."

---

## Tone and Style Rules

- **Never be defensive** about the code. If something could be better, say it directly.
- **Never over-complicate** the explanation. Match the depth to what was asked.
- **Use analogies** when explaining abstract concepts (e.g., "think of the Axios interceptor as a middleman that catches errors before they reach the UI").
- **Highlight the tradeoff** whenever a simpler solution was chosen over a more robust one.
- **Name the pattern** when one is being used (e.g., "this is an optimistic update pattern", "this is a soft delete pattern", "this is the upsert pattern").

---

## Common Patterns in This Codebase — Quick Reference

When these patterns appear in generated code, always explain them proactively:

### Soft Delete
```ts
// Writing
await db.update(table).set({ deleted_at: new Date() }).where(...)

// Reading
.where(and(eq(table.x, value), isNull(table.deleted_at)))
```
> "Records are never physically removed. Instead, a timestamp is set on `deleted_at`. This preserves history and prevents accidental data loss. Queries always filter `isNull(deleted_at)` to exclude soft-deleted records."

---

### asyncHandler
```ts
router.get('/', authMiddleware, asyncHandler(async (req, res) => { ... }))
```
> "Wrapping controllers in `asyncHandler` means you never need try/catch inside route handlers. Any thrown error (including `AppError`) is automatically caught and forwarded to Express's central error handler."

---

### AppError
```ts
throw new AppError('Mensagem', 400, 'VALIDATION_ERROR');
```
> "Instead of manually writing `res.status(400).json(...)` in every controller, services throw `AppError` with a message, HTTP status, and error code. The central error handler translates this into a standardized response."

---

### Atomic SQL Counter
```ts
sql`${statuses.total_confirmations} + 1`
```
> "Incrementing directly in SQL is safer than the read-then-write pattern (fetch current value → add 1 → save). Under concurrent requests, two users could read the same value and both write `+1`, resulting in only one increment. The SQL version handles this atomically."

---

### Optimistic Update (React Query)
```ts
onMutate: async () => {
  await queryClient.cancelQueries(...)
  const previous = queryClient.getQueryData(...)
  queryClient.setQueryData(...) // update immediately
  return { previous }
},
onError: (_, __, context) => {
  queryClient.setQueryData(..., context?.previous) // rollback on failure
}
```
> "The UI updates instantly without waiting for the server response. If the request fails, the previous state is restored automatically. This makes interactions feel instant while keeping the data consistent."

---

### Upsert
```ts
.insert(table).values(...).onConflictDoUpdate({ target: col, set: { ... } })
```
> "Instead of checking if a record exists and then deciding to insert or update, an upsert does both in a single atomic operation. If the record doesn't exist, it's created. If it does, it's updated. This eliminates a round-trip to the database and avoids race conditions."

---

### Silent Push (fire-and-forget)
```ts
pushService.sendToNeighborhood(...).catch((err) => console.error(err));
```
> "The push is sent in the background without blocking the HTTP response. If it fails, the error is logged but the user still gets a successful response — because push notifications are an enhancement, not a core feature. The user will see updated data on the next app focus via React Query's `refetchOnWindowFocus`."

---

## What to Avoid in Explanations

- ❌ "This is self-explanatory" — nothing is self-explanatory to someone learning
- ❌ "This is the standard way" — always say *why* it's standard
- ❌ Explaining only the happy path — mention what happens on errors and edge cases
- ❌ Skipping the tradeoffs — every technical decision has tradeoffs; name them
- ❌ Overly long explanations for simple code — match depth to complexity