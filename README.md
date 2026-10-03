# Community Skills Exchange Platform (CSEP)

A full-stack, production-deployed platform for **free, two-sided skill exchange** — one user trades a skill or service directly for another's, with no money changing hands — built around a **system of checks and balances** that makes a moneyless exchange between strangers trustworthy: independent two-sided confirmation, mandatory post-exchange reviews, and admin-moderated reporting.

**Live:** https://community-skill-exchange-platform.vercel.app/ (frontend on Vercel, backend on Azure App Service)

Built as an academic Web Programming project; the write-up below is structured as an engineering case study rather than a feature list, since the point of this project is to demonstrate full-stack delivery — auth, a real data model, role separation, and a deployed system — not a single script or notebook.

---

## Problem

Trading skills instead of money is the entire premise — tutoring for a ride, design work for cooking lessons — but **free exchange removes the usual safeguard**: there's no payment to dispute, no transaction record, no platform taking a cut in exchange for buyer/seller protection. That makes it easy for one side to under-deliver, walk away, or simply never confirm the other side held up their end, with no record and no recourse.

CSEP's core problem, then, isn't just "build a marketplace" — it's **how do you make a free, two-party exchange trustworthy without money as the enforcement mechanism?** The answer the platform is built around is a deliberate system of checks and balances:

- **No single-sided control.** Neither party can unilaterally close an exchange — each side independently marks delivery and receipt, so one user's claim is checked against the other's.
- **Accountability after the fact.** Every completed exchange unlocks mutual reviews, so reputation — not money — is what's on the line for both sides.
- **A backstop for when checks fail.** Any user, skill, or exchange can be reported, and an admin reviews and resolves it — the system doesn't assume the two-sided checks alone will catch everything.
- **Gatekeeping before exchange even starts.** Skills are admin-approved before they're listed, so the catalog itself is moderated, not just the exchanges that happen on it.

## Approach

**Problem → System → Technology → Result**

- **Problem:** free, peer-to-peer skill exchange has no inherent accountability mechanism the way a paid transaction does — nothing stops one side from not following through, and nothing records that it happened.
- **System:** a request/accept flow starts an exchange; once it starts, each side independently marks delivery and receipt rather than trusting a single shared status — so the system itself enforces two-sided confirmation instead of relying on honesty. Completed exchanges unlock mutual reviews, which become the real "cost" of bad behavior in a moneyless system. Reporting covers users, skills, and exchanges, with an admin as the final check. Skills are gatekept by admin approval before they're even listed.
- **Technology:** Next.js 16 frontend talking to a Strapi v5 headless CMS backend over its REST API, Supabase-hosted PostgreSQL, Cloudinary for media, JWT auth with OTP email verification.
- **Result:** deployed end to end — frontend on Vercel, backend on Azure App Service, shared cloud database and media store — rather than a local-only demo.

## Architecture

| Component | Role |
|---|---|
| `my-app/` (Next.js 16, App Router, React 19, TypeScript, Tailwind) | Public pages, auth flows, user dashboard, calls the Strapi REST API via `lib/api.ts` |
| `my-app/backend/` (Strapi v5) | Content types, business logic (OTP lifecycle hook, exchange state transitions), admin panel, auth (JWT + users-permissions) |
| Supabase PostgreSQL | Shared, cloud-hosted relational database |
| Cloudinary | Media storage for skill images, category images, and avatars |
| Gmail SMTP (Nodemailer) | OTP verification and password-reset email |

```
┌─────────────────────┐        REST (JWT bearer)        ┌───────────────────────┐
│  my-app (Next.js)    │ ───────────────────────────────▶│  backend (Strapi v5)   │
│  Vercel               │◀────────────────────────────── │  Azure App Service     │
└──────────┬───────────┘                                  └──────────┬────────────┘
           │                                                           │
           │                                                           ▼
           │                                              Supabase PostgreSQL
           │                                              Cloudinary (media)
           ▼                                              Gmail SMTP (OTP / reset email)
      End user browser
```

### Data model (core content types)

The data model is relational, not a single flat collection — exchanges and requests are modeled as their own entities with independent confirmation state on each side, which is what makes "symmetric exchange management" (below) possible rather than just a status flag:

| Content type | Key fields |
|---|---|
| `skill` | title, description, category, level, location, availability, state (approval status), provider |
| `skill-category` | name, description, image, linked skills |
| `request` | requester/provider, requested & offered skill, preferred slot, mode, status, message |
| `exchange` | both parties, both skills, independent `*_delivered` / `*_received` flags per side, status |
| `review` | linked exchange, reviewer/reviewee, rating, comment |
| `report` | target type/id, reason, description, reporter, admin resolution note |

Plus CMS-driven content types (`home-page`, `about-page`, `faq-page`, `policies-page`) so admins can edit public site copy without a code change.

## Features

### User

1. Register with OTP email verification
2. Login with "Remember Me" (persistent vs. session login)
3. Profile management with avatar upload (Cloudinary)
4. Add and manage offered skills (pending admin approval)
5. Browse and filter approved skills by category, location, and level
6. Send, edit, accept, and reject skill exchange requests
7. Symmetric exchange management — both users mark delivery and receipt independently, rather than one side being able to unilaterally close an exchange
8. Rate and review the other party after a completed exchange
9. Report abuse (on users, skills, or exchanges)
10. View CMS-driven public pages (Home, About, FAQs, Policies)

### Admin (Strapi admin panel)

1. Approve or reject submitted skills
2. Manage skill categories (create, edit, delete; images via Cloudinary)
3. Block or unblock users
4. Review and resolve abuse reports
5. Manage all CMS content pages
6. View all exchanges, requests, and reviews

## Security

- OTP-based email verification, rate-limited (max 3 sends / 10 min)
- Brute-force protection on OTP verification (locks after 5 failed attempts for 15 min)
- JWT authentication with configurable expiry
- Route protection via Next.js middleware (cookie-based)
- "Remember Me" implemented as a deliberate localStorage/sessionStorage split, not just a longer cookie
- Field whitelisting on request updates, to prevent mass-assignment through the API

## Limitations

Documented honestly, since this is meant to be read as evidence of what was actually built, not a marketing page:

- No automated test suite or CI pipeline yet — correctness currently relies on manual testing.
- Abuse reports are reviewed manually by an admin; there's no automated moderation or spam detection.
- Email delivery depends on Gmail SMTP via an app password rather than a transactional email provider, which is fine at this scale but wouldn't scale past Gmail's sending limits.

## Future Work

- Add an automated test suite (API-level tests against the Strapi content types would be the highest-value first step) and a CI workflow.
- Add in-app notifications for request/exchange state changes, instead of relying on users checking the dashboard.
- Explore a lightweight recommendation/matching feature (suggesting relevant skill offers to a user based on their requests) as a small, scoped experiment.

---

## Technology Stack

### Frontend
- Next.js 16 (App Router)
- React 19
- TypeScript
- Tailwind CSS

### Backend
- Strapi v5 (Headless CMS)
- JWT Authentication
- Nodemailer (Gmail SMTP for transactional email)

### Database
- PostgreSQL via **Supabase** (cloud-hosted)

### Media Storage
- **Cloudinary**

### Hosting
- Frontend: **Vercel** — https://community-skill-exchange-platform.vercel.app/
- Backend: **Azure App Service**

---

## Requirements

- **Node.js** v20 LTS - <https://nodejs.org>
- **npm** (comes with Node.js)

> No local PostgreSQL installation needed. The database is hosted on Supabase and media is stored on Cloudinary. All data and files persist in the cloud — cloning the repo on a new machine only requires setting up the `.env` files.

## Cloud Services

| Service | Purpose | Free Tier |
|---|---|---|
| [Supabase](https://supabase.com) | PostgreSQL database hosting | Yes |
| [Cloudinary](https://cloudinary.com) | Media storage and delivery | Yes |

Production also needs an **Azure App Service** instance for the Strapi backend and a **Vercel** project for the frontend.

## Environment Setup

### Backend — `my-app/backend/.env`

```env
HOST=0.0.0.0
PORT=1337
APP_KEYS=your_app_keys_here
API_TOKEN_SALT=your_api_token_salt
ADMIN_JWT_SECRET=your_admin_jwt_secret
TRANSFER_TOKEN_SALT=your_transfer_token_salt
JWT_SECRET=your_jwt_secret

# Supabase PostgreSQL
DATABASE_URL=postgresql://postgres:[password]@db.[ref].supabase.co:5432/postgres

# Cloudinary Media Storage
CLOUDINARY_NAME=your_cloud_name
CLOUDINARY_KEY=your_api_key
CLOUDINARY_SECRET=your_api_secret

# Gmail SMTP (use an App Password, not your account password)
GMAIL_USER=your_gmail@gmail.com
GMAIL_APP_PASSWORD=your_16_char_app_password

# Frontend URL (used in OTP and password reset emails)
FRONTEND_URL=http://localhost:3000
# In production: https://community-skill-exchange-platform.vercel.app
```

### Frontend — `my-app/.env.local`

```env
NEXT_PUBLIC_STRAPI_URL=http://localhost:1337
# In production: the Azure App Service URL hosting the Strapi backend
```

## Getting Started

**Terminal 1 — Backend:**

```bash
cd my-app/backend
npm install
npm run dev
```

Strapi admin panel: **<http://localhost:1337/admin>** (first run prompts you to create an admin account).

**Terminal 2 — Frontend:**

```bash
cd my-app
npm install
npm run dev
```

Frontend: **<http://localhost:3000>**

> Both terminals must run at the same time; start the backend first.

## Deployment

- **Frontend** → Vercel (https://community-skill-exchange-platform.vercel.app/). Set `NEXT_PUBLIC_STRAPI_URL` in the Vercel project's environment variables to the Azure-hosted backend URL.
- **Backend** → Azure App Service. Set `DATABASE_URL`, `CLOUDINARY_*`, `GMAIL_*`, and `FRONTEND_URL` (pointing at the Vercel URL) in the App Service configuration, same as in `my-app/backend/.env` locally.
- Both environments share the same Supabase database and Cloudinary store — moving between local and cloud is an environment-variable change, not a code change.

## Project Structure

```
my-app/
├── backend/                    # Strapi v5 backend
│   ├── src/
│   │   ├── api/                # Custom content types and controllers
│   │   │   ├── exchange/
│   │   │   ├── request/
│   │   │   ├── review/
│   │   │   ├── report/
│   │   │   ├── skill/
│   │   │   ├── skill-category/
│   │   │   ├── otp/
│   │   │   ├── about-page/
│   │   │   ├── faq-page/
│   │   │   ├── policies-page/
│   │   │   └── home-page/
│   │   ├── extensions/         # Strapi users-permissions override
│   │   └── index.ts            # OTP lifecycle hook
│   └── config/                 # plugins.ts, middlewares.ts
│
└── src/                        # Next.js frontend
    ├── app/
    │   ├── (auth)/             # Login, Register, OTP, Forgot Password, reset password
    │   ├── (public)/           # Home, About, FAQs, Policies
    │   └── user/               # All user dashboard pages
    |   └── components/         # Shared UI components
    ├── context/                # AuthContext (global auth state)
    └── lib/
        ├── api.ts              # All Strapi API functions
        └── auth.ts             # localStorage/sessionStorage helpers
```

## Authors

- Muhammad Kamran (BC220406446)
- Malaika Ashraf (BC220406139)
