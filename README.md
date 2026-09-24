# Realestic AI

Realestic AI is a web platform for real estate portals and agencies. Users upload property photos, and GPT-4o writes the listing: the description, room counts, amenities, photo tags and view. The platform also imports listings in bulk from CSV and enriches them with AI. It posts listings to Facebook and Instagram and manages partner agencies, team members and listing quotas. Access is sold as Stripe subscriptions.

![Portal dashboard](docs/screenshots/portal-dashboard.jpg)

A full illustrated walkthrough of the project is in [docs/Realestic-AI-Project-Documentation.pdf](docs/Realestic-AI-Project-Documentation.pdf).

## Tech stack

| Area | Choice |
| --- | --- |
| Frontend | React 19, React Router 7 (framework mode, SSR on), TypeScript, Vite 6 |
| UI | Tailwind CSS 4, Radix UI / shadcn-style components, lucide-react, Recharts |
| State | Redux Toolkit (`client/app/store`) |
| Backend | Node.js 18+, Express 5 (CommonJS) |
| Database | PostgreSQL through Prisma 6 (`server/prisma/schema.prisma`, 24 migrations) |
| Background jobs | Bull queues on Redis, run by a separate worker process |
| AI | OpenAI SDK pointed at OpenRouter, model `openai/gpt-4o` (vision) |
| Images and video | Cloudinary (signed direct uploads), Sharp, ffmpeg |
| Location | Google Geocoding and Places APIs, used for neighbourhood context |
| Payments | Stripe Checkout, Billing Portal and webhooks |
| Social | Facebook Graph API and Instagram Graph API |
| Email | Nodemailer with HTML templates (`server/emailTemplates`) |
| Hosting | Heroku: one app for the server (web + worker), one for the client |

## Getting started

You need Node.js 18 or newer, PostgreSQL 14 or newer, and Redis 6 or newer. The AI, image, payment and social features also need accounts with OpenRouter, Cloudinary, Stripe, Google Maps and Meta (Facebook).

```bash
git clone https://github.com/Forrentech-LTD/realestic-ai.git
cd realestic-ai
cp .env.example server/.env       # fill in the values (see "Environment variables")
npm run setup                     # installs root, server and client, then generates Prisma and runs migrations
npm --prefix server exec prisma db seed   # creates the 4 subscription plans
npm run dev                       # server on http://localhost:4000, client on http://localhost:5173
```

Scheduled social posts, CSV imports and AI enrichment run in the worker, so start it in a second terminal:

```bash
cd server && node workers/main.worker.js
```

| Script (repo root) | What it does |
| --- | --- |
| `npm run dev` | Starts the server (nodemon) and client (React Router dev server) together |
| `npm run server` / `npm run client` | Starts one side only |
| `npm run setup` | Installs everything, then runs `prisma generate` and `prisma migrate deploy` |
| `npm test` | Runs the server Jest suite, then the client Jest suite |
| `npm --prefix client run typecheck` | Generates route types and runs `tsc` |
| `npm --prefix client run build` | Production client build into `client/build/` |

> **"The column `Listing.source` does not exist" (Prisma P2022).** Your generated Prisma client doesn't match the checked-out schema, usually because it was generated on another branch. Run `npm --prefix server run prisma:generate` and restart the server.

## Environment variables

`server/.env` (see [.env.example](.env.example) for the full annotated list):

| Variable | Used for |
| --- | --- |
| `DATABASE_URL` | PostgreSQL connection string |
| `PORT`, `NODE_ENV` | Server port (default 4000) and mode |
| `JWT_SECRET` | Signing login tokens |
| `OPENROUTER_API_KEY` | AI generation. **The server will not start without it.** |
| `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET` | Image storage and signed uploads |
| `GOOGLE_PLACES_API_KEY` | Reverse geocoding and nearby places for descriptions (not in `.env.example` yet) |
| `EMAIL_HOST`, `EMAIL_PORT`, `EMAIL_USER`, `EMAIL_PASS`, `EMAIL_FROM` | OTP, invite and welcome emails |
| `CLIENT_URL` | Links in emails (e.g. invite links) and Stripe redirect URLs |
| `REDIS_URL` **or** `REDIS_HOST` + `REDIS_PORT` | Bull queues and the cache (`REDIS_URL` wins, and `rediss://` enables TLS) |
| `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` | Subscriptions and webhook verification |
| `FACEBOOK_APP_ID`, `FACEBOOK_APP_SECRET` | Facebook and Instagram connection |
| `SOCIAL_ENCRYPTION_KEY` | 64-character hex key; social access tokens are stored AES-256-GCM encrypted |

`client/.env`:

| Variable | Used for |
| --- | --- |
| `VITE_HOST_API` | Backend URL (default `http://localhost:4000`) |
| `VITE_FACEBOOK_APP_ID` | Facebook JS SDK login for connecting pages |
| `VITE_GOOGLE_MAPS_API_KEY` | Address search on the create-listing page |

Anything prefixed `VITE_` is bundled into the browser. Never put a secret such as `FACEBOOK_APP_SECRET` in `client/.env`.

## Accounts, roles and plans

There are three roles: `PORTALADMIN`, `AGENCYADMIN` and `MEMBER`. The **plan a user buys decides what kind of account they get**:

| Plan | Price / month | Account created | AI listings | Social posts | Bulk enrichment | Team | Partner agencies | API |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Starter | $99 | Standalone agency | 50 | 10 | — | 1 | — | — |
| Professional | $299 | Standalone agency | 500 | 100 | 50 | 10 | — | ✓ |
| Business | $699 | Portal | 2,000 | 500 | 500 | 50 | 5 | ✓ |
| Enterprise | $2,999 | Portal | 10,000 | 2,000 | 2,000 | 500 | 50 | ✓ |

Plans are stored in the `Plan` table, seeded from [server/prisma/seed.js](server/prisma/seed.js). The seed contains **test-mode Stripe price IDs**; replace them with your live price IDs before going to production.

**Sign-up flow:** `/register-basic` → email OTP → `/pricing` → Stripe Checkout → `/payment-success`. This page polls `/api/subscriptions/status` until the webhook lands. It then sends the user to `/complete-agency-setup` (Starter/Professional) or `/complete-portal-setup` (Business/Enterprise).

**Invitations:** A portal admin invites an agency admin, which creates the agency and gives it part of the portal's quota (300 by default). An agency admin invites members. Invite emails link to `/information-review?token=…`, where the invitee sets a password.

**Quota:** When a subscription activates, the plan's `aiListingsPerMonth` is written to the portal's `quotaLimit`, or to the agency's limit for a standalone agency. Portals share their quota out to agencies, and agencies share theirs out to members.

## Main features

| Feature | Where | Notes |
| --- | --- | --- |
| AI listing generation | Listings → Add Listing | Upload up to 15 photos (JPG/PNG/WEBP). GPT-4o returns the description, beds/baths, per-photo tags, amenities, view and parking. |
| Listings table | Listings | Search, status filter, date range, sort, edit, delete, CSV / CMS export, photo slideshow, MP4 video (ffmpeg) |
| CSV import + AI enrichment | Imported Listings | Required columns: `images`, `property_type`, `location`. Enrichment (Business+) runs in batches of 7 in the worker. |
| Original vs enriched comparison | Listings (compare icon) | Shown for listings that came from an import (`externalListingId`) |
| Partner agencies | Portal → Partner Agencies | Invite, archive/activate, per-agency quota, leaderboard |
| Members | Agency → Members | Invite, edit quota, archive/reactivate. An archived member is locked out on their next request. |
| Social media | Social Media | Connect Facebook pages and Instagram business accounts, post now or schedule, fetch insights. Twitter and LinkedIn are placeholders. |
| Dashboards | Dashboard | KPIs, listings by status, 6-month trend, top agencies or members |
| Billing | Settings → Plans / Billing | Upgrade, downgrade, cancel, payment methods, invoices, Stripe billing portal |
| API keys | Settings → API Keys (Professional+) | Public key `ak_live_…` plus a secret shown once. Permissions `read:listings` / `write:listings`. |
| In-app docs | `/docs` | Markdown guides in [client/app/docs](client/app/docs) |

## How AI listing generation works

1. **`POST /api/portal/listings/generate/init`** returns one signed Cloudinary upload per image. The browser uploads straight to a temporary Cloudinary folder, so images never pass through the server.
2. **`POST …/generate/process`** checks that every URL belongs to this upload session. It then calls GPT-4o with the images plus neighbourhood context from Google (when coordinates are given) and returns an editable preview. The prompt is in [server/services/description.service.js](server/services/description.service.js); it writes for the Pakistani market.
3. **`POST …/finalize`** saves the listing and moves the images into permanent storage. If the move fails, the listing is deleted.

Agency users use the same endpoints under `/api/agencies/listings/…`.

## Project structure

```text
realestic-ai/
├── client/                       React Router 7 app
│   └── app/
│       ├── routes.ts             Every URL → route module
│       ├── routes/               Thin route modules (meta + ProtectedRoute + page)
│       ├── pages/                Page components (portals/, agencies/, sections/ = settings tabs)
│       ├── components/           Layout (sidebar, topbar), charts, modals, ui/ primitives
│       ├── services/             Axios API clients (apiClient.ts adds the JWT, handles 401/403)
│       ├── store/                Redux slices and thunks
│       └── docs/                 Markdown shown at /docs
├── server/                       Express API
│   ├── index.js                  App setup, CORS, route mounting, Stripe webhook (raw body)
│   ├── routes/                   /auth, /api (portal, agencies, social, invites, api-keys, quota), /api/v1, /api/plans, /api/subscriptions
│   ├── middleware/               JWT auth, portal/agency guards, subscription + plan gates, rate limits, uploads
│   ├── controllers/  services/   Request handling / business logic (services/social = Facebook & Instagram providers)
│   ├── queues/                   Bull queues: import, enrichment, scheduler (social posts), video
│   ├── workers/main.worker.js    Worker entry: loads the import, enrichment and scheduler queues
│   ├── prisma/                   schema.prisma, migrations, seed.js (plans)
│   └── tests/                    Jest + Supertest
├── .github/workflows/            test.yml (PRs), deploy.yml (push to main → Heroku)
└── docs/                         Project documentation PDF and screenshots
```

## API overview

All app routes use `Authorization: Bearer <JWT>` from `POST /auth/login`.

| Prefix | Who | Contents |
| --- | --- | --- |
| `/auth` | Public / any user | register-basic, login, verify-email (OTP), password reset, `/me`, profile, complete-portal-setup, complete-agency-setup |
| `/api/portal` | Portal admins | dashboard, listings (+ generate, import, enrich, export, comparison, slideshow, video), agencies, invites |
| `/api/agencies` | Agency admins and members | dashboard, members, listings (same features as the portal), invites |
| `/api/social` | Portal or agency users | Facebook and Instagram connect, post, accounts, scheduled posts, insights |
| `/api/quota` | Portal admins | status, breakdown, can-create |
| `/api/api-keys` | Any user (create needs Professional+) | create, list, usage, revoke, regenerate, delete |
| `/api/subscriptions` | Any user | checkout, status, cancel, update, billing, invoices, payment methods, portal session |
| `/api/plans` | Public `GET /`; admin routes below | Plan list for the pricing page |
| `/api/webhook` | Stripe | Signed webhook events |
| `/api/v1` | External, with `x-api-key` + `x-api-secret` | `POST /generate`, `GET/POST /listings`, `GET/DELETE /listings/:id` (60 req/min, 100 AI calls/hour) |

Paid features return **402** without an active subscription and **403** (`INSUFFICIENT_PLAN` or `FEATURE_NOT_AVAILABLE`) when the plan doesn't include them.

## Testing

```bash
npm --prefix server test     # Jest + Supertest, 61 suites
npm --prefix client test     # Jest + Testing Library (jsdom)
```

CI ([.github/workflows/test.yml](.github/workflows/test.yml)) runs both suites on pull requests into `develop` and `main`.

## Deployment

On every push to `main`, [.github/workflows/deploy.yml](.github/workflows/deploy.yml) deploys two Heroku apps:

- **Server:** the whole repo is pushed. The server's `Procfile` runs `web: node index.js` and `worker: node workers/main.worker.js`, and `heroku-postbuild` runs the Prisma migrations. Scale up the worker dyno, or scheduled posts, imports and enrichment will never run.
- **Client:** `client/` is pushed as a git subtree and served with `react-router-serve`.

The workflow needs these secrets: `HEROKU_API_KEY`, `HEROKU_SERVER_APP` and `HEROKU_CLIENT_APP`. `deploy.sh` is an older manual script that pushes `develop` instead.

Allowed CORS origins are hard-coded in [server/index.js](server/index.js). Add your domain there when it changes.

## Known issues

- **The listing quota isn't checked when listings are created through the UI.** `QuotaService.canCreateListing` runs only for `POST /listings`. The `generate/process` → `finalize` flow checks only that the plan includes AI listings, so users can go over their monthly quota.
- **Any portal admin can edit the global plans.** `/api/plans/admin/*` allows every `PORTALADMIN`, and portal admins are paying customers. It needs a separate super-admin role.
- `helmet`, `hpp` and `xss-clean` are installed but never applied in `index.js`.
- Video generation runs in the web process and writes to the local temp folder, which Heroku wipes on restart.
- The sidebar "Usage" card shows 0 on every page except the dashboard.
- `bull` and `bullmq` are both installed, as are `bcrypt` and `bcryptjs`; only `bull` and `bcryptjs` are used.
- On this branch (`new-test-cases`), 225 of 1,855 server tests fail (21 of 61 suites), partly because some tests cover code that isn't on the branch yet (e.g. `autopost`). On the client, 254 of 2,602 tests fail (13 of 153 suites).
