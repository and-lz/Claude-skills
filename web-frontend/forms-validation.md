# Forms & Validation

## Form fundamentals

```html
<!-- Native form — works without JavaScript -->
<form method="POST" action="/api/contact">
  <label for="email">Email</label>
  <input id="email" name="email" type="email" required />
  <button type="submit">Send</button>
</form>
```

Server Actions in Next.js extend this — `<form action={serverAction}>` works without JS and degrades gracefully.

## Server Actions

```typescript
// features/contact/actions.ts
'use server'
import { z } from 'zod'
import { redirect } from 'next/navigation'
import { revalidateTag } from 'next/cache'

const ContactSchema = z.object({
  name:    z.string().min(1, 'Name is required').max(100),
  email:   z.string().email('Invalid email address'),
  message: z.string().min(10, 'Message must be at least 10 characters'),
})

export type ContactState = {
  error?: string
  fieldErrors?: Record<string, string[]>
  success?: boolean
}

export async function submitContact(
  _prev: ContactState,
  formData: FormData
): Promise<ContactState> {
  const parsed = ContactSchema.safeParse(Object.fromEntries(formData))

  if (!parsed.success) {
    return { fieldErrors: parsed.error.flatten().fieldErrors }
  }

  try {
    await db.contact.create({ data: parsed.data })
  } catch {
    return { error: 'Failed to send message. Please try again.' }
  }

  revalidateTag('contacts')
  redirect('/contact/success')  // throws internally — don't catch
}
```

## useActionState

```tsx
'use client'
import { useActionState } from 'react'
import { submitContact, type ContactState } from './actions'

const initialState: ContactState = {}

export function ContactForm() {
  const [state, action, isPending] = useActionState(submitContact, initialState)

  return (
    <form action={action} className="space-y-4">
      <div>
        <label htmlFor="name" className="text-sm font-medium">Name</label>
        <input
          id="name"
          name="name"
          aria-invalid={!!state.fieldErrors?.name}
          aria-describedby={state.fieldErrors?.name ? 'name-error' : undefined}
          className="mt-1 w-full rounded-md border px-3 py-2 focus-visible:ring-2 focus-visible:ring-ring"
        />
        {state.fieldErrors?.name && (
          <p id="name-error" role="alert" className="mt-1 text-xs text-destructive">
            {state.fieldErrors.name[0]}
          </p>
        )}
      </div>

      <div>
        <label htmlFor="email" className="text-sm font-medium">Email</label>
        <input id="email" name="email" type="email"
          aria-invalid={!!state.fieldErrors?.email}
          aria-describedby={state.fieldErrors?.email ? 'email-error' : undefined}
          className="mt-1 w-full rounded-md border px-3 py-2 focus-visible:ring-2 focus-visible:ring-ring"
        />
        {state.fieldErrors?.email && (
          <p id="email-error" role="alert" className="mt-1 text-xs text-destructive">
            {state.fieldErrors.email[0]}
          </p>
        )}
      </div>

      {state.error && (
        <div role="alert" className="rounded-md bg-destructive/10 p-3 text-sm text-destructive">
          {state.error}
        </div>
      )}

      <SubmitButton isPending={isPending} />
    </form>
  )
}
```

## useFormStatus

```tsx
// Must be a child component of the <form> — not in the same component
'use client'
import { useFormStatus } from 'react-dom'
import { Loader2 } from 'lucide-react'

function SubmitButton({ isPending }: { isPending?: boolean }) {
  const { pending } = useFormStatus()
  const isLoading = isPending ?? pending

  return (
    <button
      type="submit"
      disabled={isLoading}
      aria-busy={isLoading}
      className="flex items-center gap-2 rounded-md bg-primary px-4 py-2 text-sm font-medium text-primary-foreground disabled:opacity-50"
    >
      {isLoading && <Loader2 className="h-4 w-4 animate-spin" aria-hidden="true" />}
      {isLoading ? 'Sending...' : 'Send message'}
    </button>
  )
}
```

## useOptimistic

```tsx
'use client'
import { useOptimistic, useTransition } from 'react'
import { addTodo, deleteTodo } from './actions'

export function TodoList({ initialTodos }: { initialTodos: Todo[] }) {
  const [todos, addOptimistic] = useOptimistic(
    initialTodos,
    (state, action: { type: 'add'; todo: Todo } | { type: 'delete'; id: string }) => {
      if (action.type === 'add') return [...state, action.todo]
      return state.filter(t => t.id !== action.id)
    }
  )
  const [, startTransition] = useTransition()

  async function handleAdd(formData: FormData) {
    const text = formData.get('text') as string
    const optimisticTodo: Todo = { id: crypto.randomUUID(), text, done: false }
    startTransition(async () => {
      addOptimistic({ type: 'add', todo: optimisticTodo })
      await addTodo(text)
    })
  }

  async function handleDelete(id: string) {
    startTransition(async () => {
      addOptimistic({ type: 'delete', id })
      await deleteTodo(id)
    })
  }

  return (
    <div>
      <ul>
        {todos.map(todo => (
          <li key={todo.id} className="flex items-center justify-between">
            <span>{todo.text}</span>
            <button onClick={() => handleDelete(todo.id)}>Delete</button>
          </li>
        ))}
      </ul>
      <form action={handleAdd}>
        <input name="text" placeholder="Add todo..." required />
        <SubmitButton />
      </form>
    </div>
  )
}
```

## Zod validation

```typescript
import { z } from 'zod'

// Define schema once — reuse for both client and server
export const RegisterSchema = z.object({
  name:            z.string().min(1, 'Name is required').max(100),
  email:           z.string().email('Invalid email'),
  password:        z.string().min(8, 'At least 8 characters').max(100),
  confirmPassword: z.string(),
}).refine(data => data.password === data.confirmPassword, {
  message: 'Passwords do not match',
  path: ['confirmPassword'],
})

export type RegisterInput = z.infer<typeof RegisterSchema>

// Client-side validation
const result = RegisterSchema.safeParse(formData)
if (!result.success) {
  const errors = result.error.flatten().fieldErrors
  // { name: ['Name is required'], email: ['Invalid email'] }
}

// Server-side — same schema
export async function register(_prev: State, formData: FormData): Promise<State> {
  const parsed = RegisterSchema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) return { fieldErrors: parsed.error.flatten().fieldErrors }
  // ...
}
```

## React Hook Form + Zod

```tsx
'use client'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { RegisterSchema, type RegisterInput } from './schema'

export function RegisterForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
    setError,
    reset,
  } = useForm<RegisterInput>({
    resolver: zodResolver(RegisterSchema),
  })

  async function onSubmit(data: RegisterInput) {
    const result = await registerUser(data)
    if (!result.success) {
      setError('email', { message: result.error })
      return
    }
    reset()
    router.push('/dashboard')
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <div>
        <label htmlFor="name">Name</label>
        <input id="name" {...register('name')} aria-invalid={!!errors.name} />
        {errors.name && <p role="alert">{errors.name.message}</p>}
      </div>
      <div>
        <label htmlFor="email">Email</label>
        <input id="email" type="email" {...register('email')} aria-invalid={!!errors.email} />
        {errors.email && <p role="alert">{errors.email.message}</p>}
      </div>
      <div>
        <label htmlFor="password">Password</label>
        <input id="password" type="password" {...register('password')} aria-invalid={!!errors.password} />
        {errors.password && <p role="alert">{errors.password.message}</p>}
      </div>
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Creating account...' : 'Create account'}
      </button>
    </form>
  )
}
```

## Multi-step form

```tsx
'use client'
import { useState } from 'react'

type Step = 'account' | 'profile' | 'preferences' | 'review'
const STEPS: Step[] = ['account', 'profile', 'preferences', 'review']

export function MultiStepForm() {
  const [step, setStep] = useState<Step>('account')
  const [data, setData] = useState<Partial<OnboardingData>>({})

  const stepIndex = STEPS.indexOf(step)

  function next(stepData: Partial<OnboardingData>) {
    setData(prev => ({ ...prev, ...stepData }))
    setStep(STEPS[stepIndex + 1])
  }

  function back() {
    setStep(STEPS[stepIndex - 1])
  }

  return (
    <div>
      {/* Progress indicator */}
      <div role="progressbar" aria-valuenow={stepIndex + 1} aria-valuemax={STEPS.length}
        aria-label={`Step ${stepIndex + 1} of ${STEPS.length}`}
        className="flex gap-2 mb-8"
      >
        {STEPS.map((s, i) => (
          <div key={s} className={cn('h-2 flex-1 rounded-full', i <= stepIndex ? 'bg-primary' : 'bg-muted')} />
        ))}
      </div>

      {step === 'account' && <AccountStep onNext={next} />}
      {step === 'profile' && <ProfileStep onNext={next} onBack={back} />}
      {step === 'preferences' && <PrefsStep onNext={next} onBack={back} />}
      {step === 'review' && <ReviewStep data={data} onBack={back} />}
    </div>
  )
}
```

## File uploads

```tsx
'use client'
import { useState, useRef } from 'react'

export function FileUpload() {
  const [preview, setPreview] = useState<string | null>(null)
  const [uploading, setUploading] = useState(false)
  const inputRef = useRef<HTMLInputElement>(null)

  function handleFile(file: File) {
    // Validate
    if (!['image/jpeg', 'image/png', 'image/webp'].includes(file.type)) {
      toast.error('Only JPEG, PNG, and WebP images are allowed')
      return
    }
    if (file.size > 5 * 1024 * 1024) {
      toast.error('File must be smaller than 5MB')
      return
    }
    // Preview
    setPreview(URL.createObjectURL(file))
  }

  async function upload(file: File) {
    setUploading(true)
    try {
      // 1. Get presigned URL from server
      const { url, key } = await getPresignedUrl({ type: file.type, size: file.size })
      // 2. Upload directly to S3
      await fetch(url, { method: 'PUT', body: file, headers: { 'Content-Type': file.type } })
      // 3. Save key to database
      await saveFileKey(key)
      toast.success('Uploaded successfully')
    } finally {
      setUploading(false)
    }
  }

  return (
    <div
      onDragOver={e => e.preventDefault()}
      onDrop={e => { e.preventDefault(); const file = e.dataTransfer.files[0]; if (file) handleFile(file) }}
      className="border-2 border-dashed rounded-lg p-8 text-center cursor-pointer hover:border-primary transition-colors"
      onClick={() => inputRef.current?.click()}
    >
      <input
        ref={inputRef}
        type="file"
        accept="image/*"
        className="sr-only"
        aria-label="Upload image"
        onChange={e => { const file = e.target.files?.[0]; if (file) handleFile(file) }}
      />
      {preview ? (
        <img src={preview} alt="Preview" className="max-h-48 mx-auto rounded-md object-contain" />
      ) : (
        <div>
          <Upload className="mx-auto h-8 w-8 text-muted-foreground" aria-hidden="true" />
          <p className="mt-2 text-sm">Drop an image here, or click to select</p>
          <p className="text-xs text-muted-foreground">PNG, JPG, WebP up to 5MB</p>
        </div>
      )}
    </div>
  )
}
```
