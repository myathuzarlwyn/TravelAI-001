# TravelAI — AITravelBuddy

A full-stack AI-powered travel planning platform. Travellers plan trips with an AI assistant, browse packages curated by travel agents, and book via the marketplace. Platform admins manage agents and monitor activity via a dedicated admin dashboard.

---

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | React 18 + Vite, React Router v6, Context API |
| Backend | Node.js + Express |
| Database | PostgreSQL (raw `pg`, migration runner) |
| Auth | JWT (15m access token + 7d HttpOnly refresh cookie), bcrypt, Google OAuth |
| AI | OpenAI API |

---

## Project structure

```
TravelAI/
├── client/                  # React frontend (Vite)
│   └── src/
│       ├── components/
│       │   ├── admin/       # Admin dashboard UI
│       │   ├── agents/      # Public agent marketplace UI
│       │   ├── auth/        # Login, signup, password reset
│       │   ├── common/      # Navbar, shared components
│       │   ├── dashboard/   # Agent & Traveller dashboards
│       │   ├── home/        # Landing page
│       │   └── trips/       # Trip planning & detail
│       ├── context/         # AuthContext (global auth state)
│       └── services/        # API wrappers (fetch)
│           ├── adminService.js    # /api/admin/* calls (auth required)
│           ├── agentService.js    # /api/public/agents/* calls (no auth)
│           ├── authService.js     # /api/auth/* calls
│           └── tripService.js     # /api/trips/* + /api/packages/*
└── server/                  # Express backend
    ├── controllers/
    │   ├── adminController.js       # Admin management endpoints
    │   ├── publicAgentController.js # Public marketplace endpoints
    │   ├── packageController.js     # Agent package CRUD
    │   ├── profileController.js     # User profile CRUD
    │   ├── localAuthController.js   # Email/password auth
    │   ├── googleAuthController.js  # Google OAuth
    │   └── passwordResetController.js
    ├── middleware/
    │   ├── authMiddleware.js  # JWT verification → req.user
    │   └── requireRole.js     # Role-based access guard
    ├── routes/
    │   ├── adminRoutes.js     # /api/admin/* (auth + role gated)
    │   ├── publicRoutes.js    # /api/public/* (open)
    │   ├── authRoutes.js      # /api/auth/*
    │   ├── packageRoutes.js   # /api/packages/*
    │   ├── profileRoutes.js   # /api/user/*
    │   ├── tripRoutes.js      # /api/trips/*
    │   ├── chatRoutes.js      # /api/chat/*
    │   └── bookingRoutes.js   # /api/booking/*
    ├── db/
    │   ├── migrate.js         # Idempotent migration runner
    │   └── migrations/        # Numbered SQL migration files
    └── utils/
        ├── jwtHelper.js
        ├── sessionHelper.js   # writeAuditLog, initializeSession
        └── emailHelper.js
```

---

## User roles

| Role | Description |
|---|---|
| `traveler` | Default role; books trips, uses AI planner |
| `agent` | Manages travel packages via Agent Dashboard |
| `admin` | Platform admin; manages agents |
| `useradmin` | Platform admin with user management scope |
| `superadmin` | Full platform access; can assign admin roles |
| `support` | Support staff (future use) |

Account statuses: `active`, `suspended`, `pending`, `deleted`.

---

## Routing logic

### Frontend routes (`client/src/App.jsx`)

| Route | Public | Guard | Renders |
|---|---|---|---|
| `/` | ✅ | — | `HomePage` |
| `/agents` | ✅ | — | `AgentsList` (public marketplace) |
| `/login` | ✅ | Guest only | `AuthPage` |
| `/signup` | ✅ | Guest only | `AuthPage` |
| `/plan-trip` | ❌ | Auth required | `PlanTripWithTravelAI` |
| `/my-trips` | ❌ | Auth required | `MyTrips` |
| `/trip/:id` | ❌ | Auth required | `TripDetailPage` |
| `/dashboard` | ❌ | Auth required | Role-based (see below) |
| `/profile` | ❌ | Auth required | `AgentProfilePage` or `TravellerDashboard` |
| `/admin` | ❌ | Admin role required | `AdminDashboard` |

### `/dashboard` role-based redirect

```
superadmin / admin / useradmin  →  redirect to /admin
agent                           →  AgentDashboard
traveler                        →  TravellerDashboard
```

### `/admin` access guard (`AdminRoute`)

```
Unauthenticated                 →  redirect to /login
Authenticated, non-admin role   →  redirect to /dashboard
admin / useradmin / superadmin  →  AdminDashboard ✅
```

### Navbar links by role

```
Admin roles   →  Home | Agents | AI Planner | My Trips | Admin | Profile | Logout
Agent/Traveller → Home | Agents | AI Planner | My Trips | Dashboard | Profile | Logout
Unauthenticated → Home | Agents | AI Planner | Pricing | About Us | [Login button]
```

---

## API routes

### Public (no auth)
```
GET  /api/public/agents          List active agents (search, specialty filter, pagination)
GET  /api/public/agents/:id      Agent public profile + active packages
GET  /api/health                 Health check
```

### Authenticated (`Authorization: Bearer <token>`)
```
POST /api/auth/signup            Register
POST /api/auth/login             Login → returns accessToken + sets refreshToken cookie
POST /api/auth/google            Google OAuth
POST /api/auth/forgot-password
POST /api/auth/reset-password
POST /api/auth/verify-email

GET  /api/user/profile           Authenticated user's profile
PUT  /api/user/profile           Update profile

GET  /api/trips                  Saved trips
POST /api/trips/plan             AI trip planning
POST /api/trips/save
PUT  /api/trips/:id
DELETE /api/trips/:id

GET  /api/packages               Packages matching destinations
GET  /api/packages/my            Agent's own packages
POST /api/packages/single        Create package (agent+)
POST /api/packages/bulk/validate Validate bulk import file
POST /api/packages/bulk/confirm  Confirm bulk import
PUT  /api/packages/:id           Update package (agent+)
```

### Admin only (`admin | useradmin | superadmin`)
```
GET    /api/admin/stats                    Platform analytics
GET    /api/admin/agents                   List/search agents
POST   /api/admin/agents                   Provision new agent
GET    /api/admin/agents/:id               Agent detail + package stats + activity
PATCH  /api/admin/agents/:id/status        Suspend / activate
PATCH  /api/admin/agents/:id/role          Change role *
GET    /api/admin/agents/:id/packages      Agent's packages (read-only)
GET    /api/admin/audit-logs               Audit log (filterable, paginated)
```
> \* Only `superadmin` can assign elevated roles (`admin`, `useradmin`, `superadmin`).
> Self-suspend and self-demote are blocked server-side.

---

## Setup

### Prerequisites
- Node.js 18+
- PostgreSQL 14+

### 1. Install dependencies
```bash
npm run install:all
```

### 2. Configure environment

**`server/.env`**
```
DB_HOST=localhost
DB_PORT=5432
DB_NAME=your_db_name
DB_USER=your_db_user
DB_PASSWORD=your_db_password
JWT_SECRET=your_jwt_secret
JWT_REFRESH_SECRET=your_refresh_secret
CLIENT_ORIGIN=http://localhost:5173
```

**`client/.env`**
```
VITE_API_BASE_URL=/api
VITE_GOOGLE_CLIENT_ID=your_google_client_id
```

### 3. Run migrations
```bash
cd server && node db/migrate.js
```

### 4. Seed superadmin
Generate a bcrypt hash for your chosen password:
```bash
node -e "require('bcrypt').hash('YourPassword',12).then(console.log)"
```
Then run the seed SQL in `server/db/seeds/001_seed_superadmin.sql` in pgAdmin, replacing the hash placeholder.

### 5. Start development servers
```bash
# From project root — runs both client and server concurrently
npm run dev
```

- Frontend: http://localhost:5173
- Backend API: http://localhost:5000

---

## Admin Dashboard

Access at `/admin` (requires `admin`, `useradmin`, or `superadmin` role).

**Tabs:**
- **Agents** — search/filter agents, view detail, suspend/activate, change role, create new agent
- **Audit Logs** — filterable log of all security events (logins, failures, agent actions)
- **Analytics** — agent counts by status, package summary, signup trend (last 30 days), login activity

**Agent detail modal tabs:**
- Profile, Packages (read-only), Activity (last 10 audit events), Actions (status/role changes)

---

## Database migrations

Migrations live in `server/db/migrations/` and are tracked in the `schema_migrations` table. Run with:
```bash
cd server && node db/migrate.js
```
Files are applied in alphabetical order; already-applied files are skipped.

---
