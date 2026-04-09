# Skill: React Component Generation

## Purpose
This skill defines the standards and patterns for generating React + TypeScript
components in PromptForge. All frontend agent output must follow these rules.

---

## File structure (one component = one file)

```typescript
// 1. Imports — external first, then internal
import { useState, useCallback } from 'react'
import { useNavigate } from 'react-router-dom'
import { clsx } from 'clsx'

import { useTaskStore } from '@/stores/taskStore'
import { updateTask } from '@/api/tasks'
import type { Task } from '@/types'

// 2. Props interface — always exported
export interface TaskCardProps {
  task: Task
  onStatusChange?: (id: string, status: string) => void
  className?: string
}

// 3. Component — always default export
export default function TaskCard({ task, onStatusChange, className }: TaskCardProps) {
  // 3a. Hooks first (state, then effects, then custom hooks)
  const [isLoading, setIsLoading] = useState(false)
  const navigate = useNavigate()

  // 3b. Derived values
  const priorityColor = PRIORITY_COLORS[task.priority]

  // 3c. Handlers — named handle<Event>
  const handleStatusChange = useCallback(async (status: string) => {
    setIsLoading(true)
    try {
      await updateTask(task.id, { status })
      onStatusChange?.(task.id, status)
    } catch {
      // show toast
    } finally {
      setIsLoading(false)
    }
  }, [task.id, onStatusChange])

  // 3d. Render
  return (
    <div className={clsx('rounded-lg border border-gray-200 bg-white p-4', className)}>
      {/* content */}
    </div>
  )
}
```

---

## State management rules

- Local UI state (loading, open/closed, hover): `useState`
- Cross-component or persisted state: Zustand store
- Server state (fetched data): `@tanstack/react-query`
- Never use prop drilling beyond 2 levels — use store instead

### Zustand store pattern

```typescript
import { create } from 'zustand'
import { persist } from 'zustand/middleware'
import { immer } from 'zustand/middleware/immer'

interface TaskState {
  tasks: Task[]
  isLoading: boolean
  addTask: (task: Task) => void
  updateTask: (id: string, changes: Partial<Task>) => void
  removeTask: (id: string) => void
}

export const useTaskStore = create<TaskState>()(
  immer(
    persist(
      (set) => ({
        tasks: [],
        isLoading: false,
        addTask: (task) => set((state) => { state.tasks.push(task) }),
        updateTask: (id, changes) => set((state) => {
          const task = state.tasks.find(t => t.id === id)
          if (task) Object.assign(task, changes)
        }),
        removeTask: (id) => set((state) => {
          state.tasks = state.tasks.filter(t => t.id !== id)
        }),
      }),
      { name: 'task-store', partialize: (s) => ({ tasks: s.tasks }) }
    )
  )
)
```

---

## Data fetching pattern

```typescript
// Query
const { data: tasks, isLoading, error } = useQuery({
  queryKey: ['tasks', workspaceId],
  queryFn: () => getTasks(workspaceId),
  staleTime: 30_000,
})

// Mutation with optimistic update
const mutation = useMutation({
  mutationFn: (data: UpdateTaskData) => updateTask(task.id, data),
  onMutate: async (data) => {
    await queryClient.cancelQueries({ queryKey: ['tasks', workspaceId] })
    const previous = queryClient.getQueryData(['tasks', workspaceId])
    queryClient.setQueryData(['tasks', workspaceId], (old: Task[]) =>
      old.map(t => t.id === task.id ? { ...t, ...data } : t)
    )
    return { previous }
  },
  onError: (_, __, context) => {
    queryClient.setQueryData(['tasks', workspaceId], context?.previous)
  },
})
```

---

## Tailwind class conventions

- Layout: `flex`, `grid`, `items-center`, `justify-between`
- Spacing: `p-4`, `gap-3`, `space-y-2` (not arbitrary values)
- Typography: `text-sm font-medium text-gray-900`
- States: `hover:bg-gray-50`, `focus:ring-2 focus:ring-blue-500`
- Responsive: `sm:grid-cols-2 lg:grid-cols-3` (mobile-first)
- Dark mode: `dark:bg-gray-800 dark:text-white`

Never use `style={{ }}` inline styles. Never use arbitrary Tailwind values
like `text-[13px]` — stick to the design system scale.

---

## Loading and error patterns

Every page that fetches data must implement:

```typescript
if (isLoading) return <SkeletonLoader count={3} />
if (error) return <ErrorState message="Failed to load tasks" onRetry={refetch} />
if (!tasks?.length) return <EmptyState title="No tasks yet" action={<CreateTaskButton />} />
return <TaskList tasks={tasks} />
```

---

## Accessibility checklist

Every interactive element must have:
- Meaningful `aria-label` if no visible text
- `role` attribute when using non-semantic elements
- Keyboard event handler (`onKeyDown`) alongside `onClick` for custom elements
- `tabIndex={0}` for focusable custom elements
- Focus visible styles (Tailwind: `focus:outline-none focus:ring-2`)

---

## Component size rules

- If a component exceeds 200 lines: split into sub-components
- Sub-components live in a folder: `components/TaskCard/TaskCard.tsx` +
  `components/TaskCard/TaskCard.header.tsx` + `components/TaskCard/index.ts`
- No god components — one responsibility per component
