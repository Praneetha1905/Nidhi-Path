# Nidhi Path SmartCRM Intelligence

Role-based CRM for an education-loan consultancy. A single Node/Express
service serves both the JSON API (`/api/*`) and the static frontend, backed
by **Supabase** (Postgres + Auth).

- **Frontend:** static HTML/CSS/vanilla JS (no build step) in `frontend/`
- **Backend:** Node.js + Express in `backend/`
- **Database / Auth:** Supabase (Postgres + Supabase Auth)
- **Roles:** `admin`, `employee`, `board_member`, `client`, `student`

---

## 1. Directory structure

```
nidhipath-crm/
├── render.yaml                 # Render Blueprint (deploy config)
├── .nvmrc                      # Node version pin (22.17.0)
├── .gitignore
├── backend/
│   ├── server.js               # Express app: API + static frontend + SPA fallback
│   ├── package.json
│   ├── .env.example            # copy to .env and fill in
│   ├── routes/                 # auth, users, crm, commissions, reports, site
│   ├── middleware/             # verifyToken + role guards
│   ├── supabase/client.js      # supabaseAdmin (service role) + supabasePublic (anon)
│   └── scripts/create-admin.js # first-admin bootstrap CLI
├── frontend/
│   ├── index.html, login.html
│   ├── admin/  employee/  board/  client/  student/   # per-role dashboards
│   └── assets/{css,js}
└── supabase/
    └── migrations/001_initial_schema.sql   # run this in Supabase
```

---

## 2. Prerequisites

- **Node.js ≥ 18** (the repo pins **22.17.0** via `.nvmrc`)
- A **Supabase** project (free tier is fine)

---

## 3. Supabase setup

1. Create a project at <https://supabase.com>.
2. Open **SQL Editor → New query**, paste the full contents of
   [`supabase/migrations/001_initial_schema.sql`](supabase/migrations/001_initial_schema.sql),
   and **Run**. The script is idempotent (`IF NOT EXISTS`), so it is safe to
   re-run.
3. Verify the tables were created (SQL Editor):
   ```sql
   select table_name from information_schema.tables
   where table_schema = 'public' order by table_name;
   -- expect: application_id_seq, client_replies, commissions, crm_updates,
   --         employee_incentives, profiles, site_settings, student_details
   ```
4. Grab your keys from **Project Settings → API**:
   - `Project URL` → `SUPABASE_URL`
   - `anon` `public` key → `SUPABASE_ANON_KEY`
   - `service_role` `secret` key → `SUPABASE_SERVICE_ROLE_KEY` (server-only!)

---

## 4. Local development

```bash
cd backend
cp .env.example .env        # then edit .env with your Supabase values
npm install
npm run dev                 # nodemon, or: npm start
```

Open <http://localhost:3000>. Health check: <http://localhost:3000/api/health>.

---

## 5. Create the first admin (required once)

User-creation API routes require an existing admin token, so the very first
admin must be bootstrapped directly. Run this once (locally or in a Render
shell), using the **same env vars** as the server:

```bash
cd backend
node scripts/create-admin.js admin@nidhipath.in "StrongPassw0rd!" "Nidhi Path Admin"
# or set ADMIN_EMAIL / ADMIN_PASSWORD / ADMIN_NAME in the env and run:
npm run create-admin
```

The script creates the Supabase Auth user **and** the matching `profiles`
row with `role=admin`. It is safe to re-run (it reuses an existing auth user
and resets the password). Then sign in at `/login.html` → **Employee Login**.

---

## 6. Deploy to Render

The backend serves the frontend too, so you only need **one Web Service**.

### Option A — Blueprint (recommended)

1. Push the **contents of this `nidhipath-crm/` folder** as the **root** of a
   GitHub repo (so `render.yaml` sits at the repo root).
2. In Render: **New + → Blueprint →** select the repo. Render reads
   [`render.yaml`](render.yaml):
   - `rootDir: backend`, build `npm install`, start `npm start`
   - health check `GET /api/health`
3. When prompted, set the four secrets (they are `sync: false` in the blueprint):
   `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`,
   `ALLOWED_ORIGINS`. `NODE_ENV=production` is set automatically.
4. After the first deploy, open a **Render Shell** and run the
   [first-admin bootstrap](#5-create-the-first-admin-required-once).

### Option B — Manual Web Service

Create a Web Service from the repo with:

| Setting | Value |
|---|---|
| Root Directory | `backend` (or `nidhipath-crm/backend` if you pushed the whole tree) |
| Build Command | `npm install` |
| Start Command | `npm start` |
| Health Check Path | `/api/health` |

Add the environment variables below, then bootstrap the admin.

### Environment variables

| Variable | Required | Notes |
|---|:--:|---|
| `NODE_ENV` | ✓ | `production` on Render |
| `SUPABASE_URL` | ✓ | Project URL from Supabase API settings |
| `SUPABASE_ANON_KEY` | ✓ | anon public key |
| `SUPABASE_SERVICE_ROLE_KEY` | ✓ | **secret** — server only |
| `ALLOWED_ORIGINS` | ✓ (prod) | Comma-separated origins, e.g. `https://nidhipath-crm.onrender.com` |
| `PORT` | — | Render injects this automatically |

> CORS note: in `production` only origins in `ALLOWED_ORIGINS` are allowed.
> Since the frontend is served from the same origin as the API, list your own
> Render URL (and any custom domain). In `development` all origins are allowed.

---

## 7. Post-deploy smoke test

1. `GET /api/health` → `{ "success": true, "status": "ok" }`
2. `GET /login.html` loads the three login cards.
3. `GET /api/users` without a token → **401** (auth is enforced).
4. Sign in via **Employee Login** with the bootstrapped admin → lands on
   `/admin/dashboard.html` and the stat cards / funnel populate.
5. Create a test student in CRM and confirm an Application ID like
   `NP-YYYYMMDD-0001` is generated, then remove test data before launch.

---

## 8. Security notes

- **Helmet** sets HSTS, `X-Frame-Options`, `X-Content-Type-Options`, etc.
  Content-Security-Policy is intentionally disabled because the frontend uses
  inline scripts/handlers; re-enable it if the frontend is refactored.
- **Login rate limiting** throttles failed `/api/auth/login` attempts
  (10 per IP per 15 min; successful logins are not counted).
- Auth uses a **Bearer token** in the `Authorization` header (stored in
  `sessionStorage`), not cookies — so the API is not exposed to CSRF.
- The `service_role` key bypasses Row Level Security and must **never** reach
  the browser; it is used only by `backend/supabase/client.js`.
- RLS policies are enabled as defense-in-depth (see the migration). The
  `get_my_role()` `SECURITY DEFINER` helper is the canonical Supabase pattern
  and is safe (no recursion).

> Optional: there's no `frontend/assets/img/logo.jpg`; the pages reference it
> with an `onerror` fallback that hides the image, so it's harmless. Drop a
> `logo.jpg` there to show a logo.
