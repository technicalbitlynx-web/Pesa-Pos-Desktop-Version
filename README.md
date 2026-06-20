# ProfitPoint POS — Complete System Documentation

## Overview

Pesa POS is a multi-platform Point-of-Sale ecosystem consisting of five interconnected components:

| Component | Platform | Location |
|-----------|----------|----------|
| Desktop POS App | Electron (Windows) | `index.html` + `main.js` |
| Phone POS App | Android APK / PWA | `phone app/` |
| PesaPOS APK | Android (apktool-rebuilt POS app) | `Desktop\app\tools\apk-decoded\` (outside this repo) |
| PesaForce APK | Android — marketing officer field app | `Desktop\app\tools\pesaforce-decoded\` (outside this repo) |
| Admin Dashboard (POS MS) | React web app (Vercel) | `POS MS/pos-admin-frontend/` |
| Backend API (POS MS) | Node.js + Express (Render) | `POS MS/pos-admin-backend/` |

All components share the same **Turso cloud database** and the same **license key system** managed through the admin dashboard.

**PesaForce** is a separate Android app (own package `com.profitpoint.pesaforce`) used by field marketing officers to self-register, onboard new POS clients (business → subscription plan → payment), and track their own commission earnings. See [Marketing Officers & PesaForce](#marketing-officers--pesaforce) below.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   POS MS Admin Dashboard                │
│         (React + Vite — deployed on Vercel)             │
│   Manage clients, licenses, payments, subscriptions     │
└─────────────────────────┬───────────────────────────────┘
                          │ HTTPS / REST API
┌─────────────────────────▼───────────────────────────────┐
│                  POS MS Backend API                     │
│        (Express + Prisma — Vercel / Render)             │
│  /api/v1/pos/validate-license                           │
│  /api/v1/pos/sync-all   /api/v1/pos/load-all            │
└──────────┬──────────────────────────────────────────────┘
           │ LibSQL (Turso)
┌──────────▼──────────────────────────────────────────────┐
│               Turso Cloud Database                      │
│   (SQLite — libsql://pos-admin-db-*.turso.io)           │
│   Licenses · PosData (per-license JSON blobs) · etc.    │
└──────────┬──────────────────────────────────────────────┘
           │ Hybrid sync (localStorage primary, cloud secondary)
     ┌─────┴─────┐
     │           │
┌────▼────┐ ┌────▼─────────┐
│Desktop  │ │ Phone APK /  │
│POS App  │ │  PWA App     │
│Electron │ │ Android      │
└─────────┘ └──────────────┘
```

---

## 1. Desktop POS App

**Files:** `index.html`, `main.js`, `package.json`, `css/style.css`, `vendor/`, `script/`

### What it does
A full-featured Point-of-Sale application packaged as a Windows desktop app via Electron. All UI is a single self-contained HTML file — no build step required.

### Features
- Products, Sales, Stock, Credits (debt tracking), Suppliers, Expenses, Quotations
- Receipt printing and PDF export
- Multi-user login with roles (Manager / Cashier)
- Multi-theme UI (Navy Executive, Forest Corporate, Indigo Elegant, Carbon Steel)
- Barcode scanning support
- Loyalty points system
- Cloud license validation and hybrid data sync

### Tech Stack
- **Electron** 39.2.1 — desktop shell
- **React 18** — UI (loaded via Babel in-browser, no build step)
- **Tailwind CSS** — styling
- **Vendor files** (offline, no CDN): `vendor/babel.min.js`, `vendor/react.development.js`, `vendor/react-dom.development.js`, `vendor/tailwind.js`

### Running the desktop app
```bash
npm install
npm start          # launches Electron window
```

### Building the installer
```bash
npm run build      # uses electron-packager to create .exe
```

### Cloud / License setup
1. Open the app → it starts a 1-minute trial automatically
2. Click the **three-dot menu → Settings → Enter Key**
3. Enter the **Backend URL** (`https://pos-admin-backend-ten.vercel.app`) and your **license key**
4. On successful validation the app connects to cloud, loads shared data, and syncs all changes automatically (debounced 2 s)

### Data storage
- **Local (primary):** `localStorage` — products, sales, credits, suppliers, expenses, stock log, quotations
- **Cloud (secondary):** `PosData` table in Turso — synced automatically on every save

---

## 2. Phone POS App

**Files:** `phone app/index.html`, `phone app/ProfitPoint-POS.apk`

### What it does
A mobile-optimised Progressive Web App that mirrors the desktop POS functionality. Can be used in a browser or installed via the APK on Android devices.

### Features
- POS (checkout), Products, Sales history, Expenses, Customers, Suppliers, Purchase Orders, Shifts
- Loyalty points
- Receipt generation
- Multi-user login with roles
- Full-screen PWA mode
- Cloud license validation and hybrid data sync (same system as desktop)

### Tech Stack
- Vanilla JavaScript (no framework, no build step)
- Single HTML file (~470 KB)
- Font Awesome icons
- jsPDF for PDF receipts

### Installing the APK
1. Copy `ProfitPoint-POS.apk` to the Android device
2. Enable "Install from unknown sources" in Android settings
3. Open the APK file to install

### Cloud / License setup (first run)
1. Open the app → 1-minute trial starts automatically
2. Tap the **three-dot menu → Settings → Enter Key**
3. Enter **Backend URL** and **license key**
4. App validates against the server, connects to cloud, and loads shared data

### Data storage
- **Local (primary):** `localStorage` — products, sales, expenses, customers, suppliers, purchase orders, shifts
- **Cloud (secondary):** `PosData` table in Turso — synced on every save, loaded on login

### Multi-device / shared data
The same license key can be used on the desktop and the phone simultaneously. Both devices read from and write to the same `PosData` row keyed by `license_key`, so inventory and sales stay in sync across devices.

---

## 3. Admin Dashboard (POS MS Frontend)

**Location:** `POS MS/pos-admin-frontend/`

### What it does
A web-based management dashboard for the business owner / administrator to manage all clients, licenses, payments, subscriptions, and support tickets.

### Pages
| Page | Purpose |
|------|---------|
| Dashboard | Live stats: clients, revenue, devices connected, license counts |
| Clients | Create/manage client accounts |
| Subscriptions | Create and assign subscription plans |
| Licenses | Generate and track license keys |
| Payments | Record and approve payments |
| Invoices | Auto-generate and send PDF invoices |
| Reports | Revenue charts, subscription analytics |
| Tickets | Support ticket system |
| Devices | Registered POS devices — business, license key, platform, last seen (no online/offline or sales detail) |
| Marketing Officers | List of field officers with clients onboarded, pending/approved revenue, commission rate (editable by SUPER_ADMIN) and commission earned |
| Audit Logs | Admin action history |
| Settings | System configuration |

### Tech Stack
- **React 18** + **Vite 5**
- **Tailwind CSS** + **Lucide React** icons
- **Recharts** — charts and graphs
- **React Router v6** — client-side routing
- **Axios** — HTTP client
- **Socket.IO client** — real-time device monitoring

### Running locally
```bash
cd "POS MS/pos-admin-frontend"
npm install
npm run dev        # starts dev server on http://localhost:5173
```

### Building for production
```bash
npm run build      # outputs to dist/
```

### Deployed URL
`https://pos-admin-dashboard-cyan.vercel.app`

### Routing
`vercel.json` includes a catch-all rewrite to `index.html` so all React routes work on direct navigation/refresh.

---

## 4. Backend API (POS MS Backend)

**Location:** `POS MS/pos-admin-backend/`

### What it does
REST API powering both the admin dashboard and all POS devices. Handles authentication, license management, data storage, invoice generation, real-time WebSocket connections, and scheduled jobs.

### Tech Stack
- **Node.js** + **Express 4**
- **Prisma 5** ORM with `@prisma/adapter-libsql`
- **Turso** (LibSQL) cloud SQLite database
- **Socket.IO 4** — real-time device connections
- **JWT** — admin authentication
- **Nodemailer** — email notifications
- **PDFKit** — invoice PDF generation
- **node-cron** — subscription expiry jobs

### API Endpoints

#### Authentication
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/auth/login` | Admin login |
| POST | `/api/v1/auth/refresh` | Refresh JWT token |
| POST | `/api/v1/auth/logout` | Logout |

#### POS (Public — license-key authenticated)
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/pos/validate-license` | Validate license key + device |
| POST | `/api/v1/pos/sync-sales` | Push daily sales aggregates |
| POST | `/api/v1/pos/sync-all` | Push full data blobs to cloud |
| GET  | `/api/v1/pos/load-all` | Pull full data blobs from cloud |
| GET  | `/api/v1/pos/status` | Check license status |

#### Admin (JWT protected)
| Method | Path | Description |
|--------|------|-------------|
| GET/POST | `/api/v1/clients` | Manage clients |
| GET/POST | `/api/v1/licenses` | Manage licenses |
| GET/POST | `/api/v1/subscriptions` | Manage subscriptions |
| GET/POST | `/api/v1/payments` | Manage payments — approving one auto-generates/activates the client's license |
| GET/POST | `/api/v1/invoices` | Generate/manage invoices |
| GET/POST | `/api/v1/tickets` | Support tickets |
| GET | `/api/v1/reports/*` | Analytics and reports |
| GET | `/api/v1/pos/devices` | Connected POS devices |

#### Marketing (PesaForce officers)
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/marketing/register` | Public | Officer self-registration → creates a `SALES_MANAGER` admin user |
| GET | `/api/v1/marketing/my-stats` | Officer (JWT) | Officer's own clients/revenue/commission summary |
| GET | `/api/v1/marketing/my-clients` | Officer (JWT) | Paginated list of clients the officer onboarded, incl. license key once issued |
| GET | `/api/v1/marketing/officers` | Admin (`admin:read`) | List all officers with earnings, shown on the POS MS Marketing Officers page |
| PATCH | `/api/v1/marketing/officers/:id/commission` | Admin (`admin:update`) | Update an officer's commission rate |

### Running locally
```bash
cd "POS MS/pos-admin-backend"
npm install
npm run dev        # starts on http://localhost:3000
```

### Environment variables (`.env`)
```env
NODE_ENV=production
PORT=3000

# Turso cloud database
TURSO_DATABASE_URL=libsql://pos-admin-db-*.turso.io
TURSO_AUTH_TOKEN=<token>

# JWT
JWT_SECRET=<secret>
JWT_ACCESS_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=
SMTP_PASS=

# CORS — comma-separated allowed origins
ALLOWED_ORIGINS=https://pos-admin-dashboard-cyan.vercel.app,...

# Rate limiting
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX=100
```

### Deployed URL
- **Render:** `https://pos-admin-backend.onrender.com` — auto-deploys on push to `master` of the `technicalbitlynx-web/pos-admin-backend` GitHub repo
- A `pos-keep-alive` Render cron job pings `/health` every 14 minutes to avoid the free-tier cold-start sleep (`render.yaml`)
- A legacy Vercel deployment of the same backend may still exist but is not the one actively maintained — treat Render as the source of truth

### Database schema (key models)
```
License        — license keys, status, device binding, expiry
Client         — client/business accounts
Subscription   — subscription plans and assignments
Payment        — payment records
PosData        — per-license JSON blobs for all POS data
PosSalesReport — daily aggregated sales totals
Ticket         — support tickets
AuditLog       — admin action history
```

### Schema changes
`npx prisma db push` / `prisma migrate` do **not** reliably work against Turso through the driver adapter. In practice, schema changes are applied with small one-off Node scripts using `@libsql/client` directly (see `migrate-marketing.js`, `migrate-license-device.js` in the backend root):
```js
const { createClient } = require('@libsql/client');
const client = createClient({ url: process.env.TURSO_DATABASE_URL, authToken: process.env.TURSO_AUTH_TOKEN });
await client.execute(`ALTER TABLE Client ADD COLUMN onboarded_by TEXT REFERENCES AdminUser(id)`);
```
Run with `node migrate-xxx.js` against the production Turso DB, then update `prisma/schema.prisma` to match and run `npx prisma generate`. SQLite/LibSQL only supports additive `ALTER TABLE ADD COLUMN` — there's no `ALTER COLUMN`, so an existing `NOT NULL` constraint (e.g. `Client.email`) can't be relaxed without a full table rebuild; the pragmatic workaround used here is generating a placeholder value in the controller when the field is optional in a client app but required in the DB.

---

## License Key System

### How licenses work

**Admin-created path** (manual, via dashboard):
1. Admin creates a **Client** in the dashboard
2. Admin creates a **Subscription** (plan + duration) for that client
3. Admin generates a **License Key** linked to that subscription
4. License key is given to the client
5. Client enters the key in the POS app (desktop or phone) along with the backend URL
6. Backend validates the key, activates it (PENDING → ACTIVE), and binds the first device
7. Additional devices can use the same key — they share the same cloud data

**Marketing-officer path** (field onboarding via PesaForce, auto-generated):
1. Officer onboards a client + plan + payment through PesaForce — payment is recorded `PENDING` (officers can't self-approve)
2. Admin reviews and approves the payment in POS MS → Payments
3. On approval, `payments.service.js` activates the subscription and client, then **auto-generates a license** if none exists yet for that subscription (or activates an existing `PENDING` one)
4. The license appears immediately in POS MS → Licenses, and the **license key shows up in the officer's PesaForce app** (My Clients → tap client → License Key row) so they can hand it to the client on the spot

### License states
| Status | Meaning |
|--------|---------|
| PENDING | Generated, not yet activated by a device |
| ACTIVE | In use — valid for POS operations |
| EXPIRED | Past expiry date |
| SUSPENDED | Manually disabled by admin |

### Multi-device support
One license key works on multiple devices simultaneously (desktop + phone). All devices share the same `PosData` record in the cloud, so inventory, sales, and customer data stay in sync.

---

## Marketing Officers & PesaForce

PesaForce is a standalone Android app (package `com.profitpoint.pesaforce`) for field marketing officers — built from `apktool`-decoded sources, not from this repo's web code. It's a self-contained vanilla-JS single-file app (`assets/index.html`), same pattern as the desktop/phone POS apps.

### What officers can do
- **Sign up** (`POST /api/v1/marketing/register`) — creates a `SALES_MANAGER` admin user with a default 10% `commission_rate`, then auto-logs in
- **Onboard a client** via a 3-step wizard: client details → subscription plan → payment (recorded `PENDING`)
- **My Clients** — paginated list of clients they onboarded, with subscription plan, payment status, and license key once issued
- **My Earnings** — total clients, pending/approved revenue, commission rate, commission earned, month-by-month breakdown

### Backend module
`POS MS/pos-admin-backend/src/modules/marketing/` — `register`, `myStats`, `myClients`, `getOfficers` (admin), `updateCommission` (admin). RBAC: `SALES_MANAGER` is scoped to `clients:*`, `subscriptions:*`, `licenses:create`, `payments:create`, `reports:read` (`src/middleware/rbac.js`).

### Gotchas already hit and fixed
- `Client.email` is `NOT NULL UNIQUE` in the DB but optional in the PesaForce form — missing email used to 500; the clients controller now generates a placeholder (`client.<timestamp>.<rand>@pos.local`) when none is given
- Payment dates come back from LibSQL as strings, not `Date` objects — any `.toISOString()` call on a raw DB date field must be wrapped in `new Date(...)` first (hit in both `marketing.controller.js` and `reports.service.js`)
- Deleting an admin user (officer) can hit a `FOREIGN KEY constraint failed` if they have recorded payments/onboarded clients/audit logs — `admin.service.js`'s `remove()` nullifies those FK references in a transaction before deleting
- The POS MS frontend's axios interceptor already unwraps `res.data`; don't add a second `.then((r) => r.data)` in an `api/*.js` file or you'll silently drop `pagination` and get an empty table (see `pos-admin-frontend/README.md`)

### Building/signing the APK
Toolchain lives outside this repo at `Desktop\app\tools\`:
```
java -jar apktool.jar b pesaforce-decoded -o PesaForce.apk
"<Android SDK>\build-tools\<ver>\zipalign.exe" -v -p 4 PesaForce.apk PesaForce-aligned.apk
"<Android SDK>\build-tools\<ver>\apksigner.bat" sign --ks pesapos-debug.keystore \
  --ks-key-alias pesapos --ks-pass pass:pesapos123 --key-pass pass:pesapos123 \
  --out PesaForce_signed.apk PesaForce-aligned.apk
```
Output is copied to `Desktop\PesaPos Apk\PesaForce_signed.apk` for distribution. The same keystore/process builds `PesaPOS_signed.apk` from `apk-decoded/` (package `com.profitpoint.pos`). Always zipalign before signing — skipping it can cause "app not installed" errors on some devices, and never leave `android:debuggable="true"` in `AndroidManifest.xml` for distributed builds.

---

## Hybrid Storage System

All POS apps (desktop and phone) use a hybrid storage pattern:

```
User action → Save to localStorage (instant, offline-safe)
           → syncAllToCloud() debounced 2 s (fire-and-forget)
                  ↓
           POST /api/v1/pos/sync-all
           → Upsert PosData row for license_key
```

On login / license activation:
```
loadFromCloud() → GET /api/v1/pos/load-all
               → Merge cloud data into localStorage
               → App re-renders with cloud data
```

**Cloud sync fields per license key:**

| Field | Desktop POS | Phone App |
|-------|------------|-----------|
| products | ✓ | ✓ |
| sales | ✓ (last 90 days) | ✓ (last 90 days) |
| credits | ✓ | — |
| suppliers | ✓ | ✓ |
| expenses | ✓ | ✓ |
| stock_log | ✓ | — |
| quotations | ✓ | — |
| customers | — | ✓ |
| purchase_orders | — | ✓ |
| shifts | — | ✓ |

---

## File Structure

```
app/
├── index.html                    # Desktop POS app (Electron frontend, ~510 KB)
├── main.js                       # Electron entry point
├── package.json                  # Electron app config (ProfitPointPOS v2.1.0)
├── package-lock.json
├── logo.png                      # App icon
├── logo-bg.png                   # Background logo
├── README.md                     # This file
│
├── css/
│   └── style.css                 # Global styles + 4 colour themes
│
├── vendor/                       # Offline vendor libraries (no CDN)
│   ├── babel.min.js
│   ├── react.development.js
│   ├── react-dom.development.js
│   └── tailwind.js
│
├── script/                       # Alternate vendor copies
│   └── (same as vendor/)
│
├── phone app/
│   ├── index.html                # Phone PWA (~470 KB single-file app)
│   └── ProfitPoint-POS.apk       # Android APK (6.2 MB)
│
├── POS MS/
│   ├── start.bat                 # Quick-start script
│   │
│   ├── pos-admin-frontend/       # Admin dashboard (React + Vite)
│   │   ├── src/
│   │   │   ├── App.jsx
│   │   │   ├── main.jsx
│   │   │   ├── api/              # Axios API modules (11 files)
│   │   │   ├── components/       # Shared UI components
│   │   │   │   ├── charts/
│   │   │   │   ├── common/
│   │   │   │   └── layout/
│   │   │   ├── context/          # AuthContext, ThemeContext
│   │   │   ├── hooks/
│   │   │   └── pages/            # 16 page components
│   │   ├── vercel.json           # SPA catch-all rewrite
│   │   ├── vite.config.js
│   │   └── package.json
│   │
│   └── pos-admin-backend/        # REST API (Express + Prisma)
│       ├── src/
│       │   ├── app.js            # Express app + CORS + middleware
│       │   ├── server.js         # HTTP server + Socket.IO
│       │   ├── config/           # database, mailer, redis
│       │   ├── middleware/       # auth, rbac, rateLimiter, errorHandler
│       │   ├── modules/          # 10 business modules
│       │   │   ├── auth/
│       │   │   ├── clients/
│       │   │   ├── licenses/
│       │   │   ├── payments/
│       │   │   ├── subscriptions/
│       │   │   ├── invoices/
│       │   │   ├── tickets/
│       │   │   ├── reports/
│       │   │   ├── admin/
│       │   │   └── pos/          # License validation + cloud sync
│       │   ├── jobs/             # Cron: subscription expiry checker
│       │   ├── websocket/        # Socket.IO POS device manager
│       │   └── utils/
│       ├── prisma/
│       │   ├── schema.prisma
│       │   └── migrations/
│       ├── .env                  # Environment variables (not committed)
│       ├── vercel.json
│       ├── render.yaml
│       ├── Dockerfile
│       └── package.json
│
├── PREVIOUS VERSION/             # Archived legacy version
│   ├── index.html
│   ├── launcher.py
│   ├── build_exe.spec
│   └── build/
│
└── node_modules/                 # Electron + packager dependencies
```

---

## Quick Start

### Start everything locally
```bash
# 1. Backend API
cd "POS MS/pos-admin-backend"
npm install
npm run dev          # → http://localhost:3000

# 2. Admin dashboard
cd "POS MS/pos-admin-frontend"
npm install
npm run dev          # → http://localhost:5173

# 3. Desktop POS
cd ../..             # back to app root
npm install
npm start            # opens Electron window

# 4. Phone app
# Open "phone app/index.html" in a browser
# or install "phone app/ProfitPoint-POS.apk" on Android
```

### Deploy backend to Vercel
```bash
cd "POS MS/pos-admin-backend"
vercel --prod
```

### Deploy dashboard to Vercel
```bash
cd "POS MS/pos-admin-frontend"
vercel --prod
```

---

## Deployment Summary

| Component | Platform | URL |
|-----------|----------|-----|
| Admin Dashboard | Vercel | `https://pos-admin-dashboard-cyan.vercel.app` |
| Backend API | Render | `https://pos-admin-backend.onrender.com` |
| Database | Turso | `libsql://pos-admin-db-technicalbitlynx-web.aws-ap-northeast-1.turso.io` |
| PesaForce APK | Manual distribution | `Desktop\PesaPos Apk\PesaForce_signed.apk` |
| PesaPOS APK | Manual distribution | `Desktop\PesaPos Apk\PesaPOS_signed.apk` |

---

## Technology Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Desktop shell | Electron | 39.2.1 |
| Desktop/phone UI | React | 18.2.0 |
| Admin dashboard | React + Vite | 18.2.0 + 5.0.8 |
| Styling | Tailwind CSS | 3.3.5 |
| Backend | Node.js + Express | 4.18.2 |
| ORM | Prisma | 5.7.0 |
| Database | Turso (LibSQL) | cloud |
| Real-time | Socket.IO | 4.6.2 |
| Icons | Lucide React / Font Awesome | — |
| Charts | Recharts | — |
| PDF | PDFKit / jsPDF | — |
| Auth | JWT | — |
