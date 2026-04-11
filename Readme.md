# TaskFlow

A real-time collaborative task management SaaS. Organise work into boards, track tasks with priorities and due dates, assign work to teammates, and see every update live via WebSockets.

## Features

- **Authentication** — JWT-based register/login with refresh token rotation
- **Workspaces** — multi-user workspaces with role-based access (Owner, Admin, Member, Viewer)
- **Kanban boards** — drag-and-drop task cards across columns
- **Real-time sync** — WebSocket broadcasting keeps every member's board in sync instantly
- **Task management** — priorities, due dates, labels, assignees, and soft delete
- **Comments** — threaded comments per task with live updates
- **Activity feed** — per-workspace audit log of every action
- **Soft delete** — tasks and workspaces are never hard deleted

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, TypeScript, Vite, Tailwind CSS |
| State | Zustand with persist middleware |
| Data fetching | TanStack Query v5 with optimistic updates |
| Real-time | Socket.io client |
| Backend | Node.js 20, Express 4, TypeScript |
| Auth | JWT + bcrypt (refresh token rotation) |
| Real-time | Socket.io server, room-based broadcasting |
| ORM | Prisma 5 with PostgreSQL 15 |
| Validation | Zod schemas on every route |
| Testing | Vitest + RTL (frontend), Jest + Supertest (backend) |

## Quick Start

### Prerequisites
- Node.js 20+
- PostgreSQL 15+
- npm 9+

### Backend

```bash
cd backend
cp .env.example .env
# Edit .env — set DATABASE_URL, JWT_SECRET, JWT_REFRESH_SECRET
npm install
npx prisma migrate dev --name init
npx prisma db seed
npm run dev
# Runs on http://localhost:4000
```

### Frontend

```bash
cd frontend
cp .env.example .env
# Edit .env — set VITE_API_URL and VITE_SOCKET_URL
npm install
npm run dev
# Runs on http://localhost:5173
```

### Default seed accounts

| Email | Password | Role |
|---|---|---|
| owner@example.com | password123 | Owner |
| admin@example.com | password123 | Admin |
| member@example.com | password123 | Member |

## Project Structure

```
output/
├── backend/
│   ├── prisma/
│   │   ├── schema.prisma       ← all data models
│   │   └── seed.ts             ← realistic seed data
│   └── src/
│       ├── config.ts           ← typed env config
│       ├── app.ts              ← Express app factory
│       ├── index.ts            ← HTTP server + Socket.io init
│       ├── middleware/         ← auth, validate, errorHandler
│       ├── routes/             ← one router per resource
│       ├── controllers/        ← thin — delegates to services
│       ├── services/           ← business logic + activity logging
│       ├── schemas/            ← Zod request schemas
│       ├── socket/             ← Socket.io handlers
│       └── utils/              ← AppError, asyncHandler
├── frontend/
│   └── src/
│       ├── api/                ← typed Axios clients per resource
│       ├── components/         ← UI, board, task, layout components
│       ├── hooks/              ← useAuth, useToast
│       ├── lib/                ← Axios instance, Socket.io client
│       ├── pages/              ← Login, Register, Dashboard, Workspace
│       ├── router.tsx          ← lazy-loaded React Router config
│       ├── stores/             ← Zustand: auth, workspace, ui
│       └── types/              ← TypeScript interfaces for all models
├── tests/
│   ├── backend/                ← Jest + Supertest integration tests
│   └── frontend/               ← Vitest + RTL component/store tests
└── docs/
    ├── api.md                  ← full REST API reference
    ├── websockets.md           ← WebSocket event catalogue
    ├── deployment.md           ← Docker + manual deployment guide
    └── architecture.md         ← design decisions and extension guide
```

## Environment Variables

### Backend (`.env`)

| Variable | Description | Example |
|---|---|---|
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://user:pass@localhost:5432/taskflow` |
| `JWT_SECRET` | 64-char random string for access tokens | — |
| `JWT_REFRESH_SECRET` | 64-char random string for refresh tokens | — |
| `JWT_EXPIRES_IN` | Access token lifetime | `7d` |
| `JWT_REFRESH_EXPIRES_IN` | Refresh token lifetime | `30d` |
| `PORT` | HTTP port | `4000` |
| `CLIENT_URL` | CORS allowed origin | `http://localhost:5173` |
| `NODE_ENV` | Environment | `development` |
| `BCRYPT_ROUNDS` | bcrypt work factor | `12` |

### Frontend (`.env`)

| Variable | Description | Example |
|---|---|---|
| `VITE_API_URL` | Backend REST base URL | `http://localhost:4000/api/v1` |
| `VITE_SOCKET_URL` | Backend WebSocket URL | `http://localhost:4000` |

## Running Tests

```bash
# Backend tests (requires a running PostgreSQL instance)
cd tests/backend
npx ts-jest --config jest.config.ts

# Frontend tests (no server required)
cd tests/frontend
npx vitest --config vitest.config.ts
```

## License

MIT