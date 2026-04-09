# Skill: Database Schema Design

## Purpose
This skill defines Prisma schema conventions and PostgreSQL best practices
for the database agent. All schema output must follow these rules.

---

## Model conventions

### ID fields — always CUID, never auto-increment
```prisma
id  String  @id @default(cuid())
```

### Timestamps — always present
```prisma
createdAt  DateTime  @default(now())
updatedAt  DateTime  @updatedAt
```

### Soft delete fields (on Task, Workspace, Board)
```prisma
deletedAt  DateTime?
```
Every query on soft-deletable models must include `WHERE deleted_at IS NULL`
(in Prisma: `where: { deletedAt: null }`).

### Table naming — always snake_case via @@map
```prisma
model WorkspaceMember {
  @@map("workspace_members")
}
```

### Column naming — camelCase in schema, snake_case in DB via @map
```prisma
model Task {
  dueDate   DateTime?  @map("due_date")
  createdAt DateTime   @default(now()) @map("created_at")
}
```

---

## Relation conventions

### One-to-many (User has many Tasks)
```prisma
model User {
  tasks  Task[]
}

model Task {
  userId  String
  user    User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@index([userId])
}
```

### Many-to-many (Task has many Labels, via explicit junction)
```prisma
// Always use explicit junction tables, never implicit @relation
model TaskLabel {
  taskId   String
  labelId  String
  task     Task    @relation(fields: [taskId], references: [id], onDelete: Cascade)
  label    Label   @relation(fields: [labelId], references: [id], onDelete: Cascade)

  @@id([taskId, labelId])
  @@map("task_labels")
}

model Task {
  labels  TaskLabel[]
}

model Label {
  tasks  TaskLabel[]
}
```

### Optional relation (Task may have assignee)
```prisma
model Task {
  assigneeId  String?
  assignee    User?   @relation("AssignedTasks", fields: [assigneeId], references: [id], onDelete: SetNull)
}
```

---

## onDelete strategy

| Relationship | Strategy | Reasoning |
|---|---|---|
| Comment on Task | Cascade | Comments don't exist without the task |
| Task on Column | Cascade | Tasks belong to a column |
| WorkspaceMember on User | Cascade | Membership gone when user deleted |
| WorkspaceMember on Workspace | Cascade | Membership gone when workspace deleted |
| Task.assignee on User | SetNull | Task survives user deletion, just unassigned |
| Activity.user on User | SetNull | Keep activity history, anonymise user |
| Board on Workspace | Cascade | |
| Column on Board | Cascade | |

Never use `Restrict` — it causes confusing errors and blocks deletions.

---

## Indexing strategy

Index every foreign key:
```prisma
@@index([userId])
@@index([workspaceId])
@@index([boardId])
```

Index fields used in WHERE clauses:
```prisma
@@index([deletedAt])          // soft delete filter
@@index([status])             // task filtering
@@index([priority])           // task filtering
@@index([dueDate])            // sorting/filtering
@@index([createdAt])          // sorting
```

Compound indexes for common query patterns:
```prisma
@@index([workspaceId, deletedAt])   // list non-deleted workspace tasks
@@index([columnId, createdAt])      // list tasks in column by date
@@index([assigneeId, status])       // assigned tasks by status
```

Unique constraints:
```prisma
@@unique([workspaceId, slug])   // workspace-scoped slugs
@@unique([userId, workspaceId]) // one membership record per user per workspace
```

---

## Enum conventions

Always define enums in schema (not as string fields):
```prisma
enum TaskStatus {
  BACKLOG
  TODO
  IN_PROGRESS
  IN_REVIEW
  DONE
  CANCELLED
}
```

Enums generate TypeScript types automatically — use them in service code.

---

## Prisma query best practices

```typescript
// Always select only needed fields in list queries
const tasks = await prisma.task.findMany({
  where: { columnId, deletedAt: null },
  select: {
    id: true,
    title: true,
    priority: true,
    dueDate: true,
    assignee: { select: { id: true, name: true, avatar: true } }
    // Don't select: description, large text fields in list views
  }
})

// Use transactions for multi-step writes
const [task, activity] = await prisma.$transaction([
  prisma.task.create({ data: taskData }),
  prisma.activity.create({ data: activityData })
])

// Upsert pattern for idempotent operations
await prisma.workspaceMember.upsert({
  where: { userId_workspaceId: { userId, workspaceId } },
  create: { userId, workspaceId, role: 'MEMBER' },
  update: { role }
})
```

---

## Seed data quality standards

- Passwords: always bcrypt-hashed (never plaintext in seed)
- IDs: let Prisma generate them (use `cuid()`) — don't hardcode IDs
- Dates: use relative dates from `new Date()` — don't hardcode year strings
- Volume: enough data to make the UI non-trivial:
  - ≥ 3 users, 2 workspaces, 3 boards each, 4 columns each, 8 tasks per board
- Variety: mix all enum values across the seed data
