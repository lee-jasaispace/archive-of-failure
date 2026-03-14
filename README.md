# Museum of Lost Technologies

A deliberately vulnerable web application built for offensive security
skill development. Styled as a museum archive cataloguing historical
technologies whose security failures shaped the discipline.

> **Do not deploy this application to a public server.**
> It contains intentional security vulnerabilities. It is a local
> training environment only.

---

## What This Is

The Museum of Lost Technologies is a Node.js / Express / PostgreSQL web
application with a permanent collection of thirteen fictional artifacts —
each one assigned to a specific web security exploit class. The app exists
in two modes simultaneously:

- **Secure baseline** (`main` branch) — well-written code with proper
  input validation, parameterized queries, sanitized output, and correct
  access controls.
- **Vulnerable variants** (`feature/vuln-*` branches) — the same routes
  with the exploit surfaces active. One branch per exploit class.

The goal is to study each exploit by reading the secure code, switching to
the vulnerable branch, triggering the exploit manually, and understanding
exactly what changed and why it matters.

**Target certifications:** eWPT → BSCP  
**Primary reference:** PortSwigger Web Security Academy

---

## Exploit Coverage

| Branch | Exploit | Surface |
|---|---|---|
| `feature/vuln-sqli` | SQL Injection | `GET /search` — search query string |
| `feature/vuln-xss` | Stored XSS | `POST /artifacts/:slug/comments` |
| `feature/vuln-cmdi` | Command Injection | `POST /admin/restoration/process` |
| `feature/vuln-idor` | IDOR | `GET /profile/:id` |
| `feature/vuln-ssrf` | SSRF | `POST /submit` — external URL fetch |
| `feature/vuln-mass` | Mass Assignment | `POST /auth/register` — role field |
| `feature/vuln-proto` | Prototype Pollution | `POST /submit` — metadata merge |
| `feature/vuln-brute` | Brute Force | `POST /auth/login` — no rate limit |
| `feature/vuln-token` | Predictable Token | `POST /auth/reset-request` |
| `feature/vuln-upload` | Insecure File Upload | `POST /submit` — MIME validation |

---

## Prerequisites

- **Node.js** 20 or later
- **PostgreSQL** 16
- **ImageMagick** (`convert` must be on PATH — required by Restoration Lab)
- **npm** 10 or later

```bash
# Verify
node --version    # 20.x+
psql --version    # 16.x
convert --version # ImageMagick 7.x
```

---

## Local Setup

### 1. Clone and install

```bash
git clone https://github.com/yourhandle/museum-of-lost-technologies.git
cd museum-of-lost-technologies
npm install
```

### 2. Create the database

```bash
createdb museum_dev
```

### 3. Configure environment

```bash
cp .env.example .env
```

Open `.env` and set:

```
NODE_ENV=development
PORT=3000
DATABASE_URL=postgres://youruser:yourpassword@localhost:5432/museum_dev
SESSION_SECRET=change-this-to-at-least-32-random-characters
UPLOAD_MAX_SIZE_MB=10
UPLOAD_ALLOWED_MIMES=image/jpeg,image/png,application/pdf
LOG_LEVEL=debug
```

### 4. Initialize the database

```bash
npm run db:reset
```

This drops and recreates the schema, then seeds all thirteen artifacts and
the Curator account. Seeded credentials are in `db/seed.sql`.

### 5. Run the app

```bash
npm run dev
```

App is available at `http://localhost:3000`.

---

## Roles

Three roles. Two are obtainable through the UI; one is seeded only.

| Role | How to get it | What it unlocks |
|---|---|---|
| Visitor | No account required | Public collection, Field Guide |
| Researcher | Register at `/auth/register` | Comment, submit artifacts, view profile |
| Curator | Seeded account only — see `db/seed.sql` | Moderation queue, submission review, Restoration Lab |

There is no UI path to create a Curator account. This is intentional.
The mass assignment exploit (`feature/vuln-mass`) demonstrates how a
researcher could escalate to curator if the role field is accepted from
the request body.

---

## Working With the Dual-Branch Architecture

The project's core design is that every exploit surface has its own branch.
The workflow for studying an exploit:

```bash
# 1. Read the secure implementation on main
git checkout main
# Read src/routes/search.js — parameterized query

# 2. Switch to the vulnerable variant
git checkout feature/vuln-sqli
# Read src/routes/search.js — string concatenation

# 3. Run the app and trigger the exploit manually
npm run dev
# Try: GET /search?q=' OR '1'='1

# 4. Switch back
git checkout main
```

**Rules:**
- Never merge a `feature/vuln-*` branch into `main`.
- Never "fix" a vulnerable branch — the broken code is the point.
- Never deploy anything other than `main`, and even then only locally.

---

## Commands

```bash
npm run dev           # Start with nodemon (hot reload)
npm test              # Run test suite (Vitest)
npm run test:watch    # Vitest in watch mode
npm run test:coverage # Coverage report

npm run db:reset      # Drop, recreate, seed (dev only)
npm run db:migrate    # Run pending migrations
npm run db:seed       # Seed artifacts and curator only

npm run lint          # ESLint
npm run lint:fix      # ESLint with auto-fix
npm run format        # Prettier
```

---

## Project Structure

```
src/
├── app.js              Express setup and middleware registration
├── server.js           HTTP server entry point
├── config/             Database pool, session config
├── middleware/         Auth guards, audit logging, rate limiters
├── routes/             One file per route group
│   └── admin/          Queue, review, restoration lab (Curator only)
├── services/           Database queries and business logic
├── utils/              resolveCondition, sanitize, UUID validation
└── views/              EJS templates
    ├── partials/       Nav, footer, condition dot
    ├── admin/          Curator-only views
    ├── breach/         XSS, SQLi, CMDi overlay screens
    └── errors/         403, 404, 500, 429

db/
├── schema.sql          Source of truth for database structure
├── seed.sql            Thirteen artifacts, seeded Curator account
└── migrations/         Numbered migration files

public/
├── css/main.css        Hand-written — no framework
└── js/breach.js        Breach screen animation only
```

---

## The Artifact Collection

Thirteen artifacts, each assigned to an exploit class:

| # | Artifact | Era | Exploit |
|---|---|---|---|
| I | Vanthorpe Tabulating Engine | 1943 | SQL Injection |
| II | Caduceus Radiological Sequencer | 1985 | Narrative |
| III | Meridian SSL Entropy Engine | 1994 | Brute Force / Session |
| IV | Zephyr Personal Media Device | 2006 | DoS (exhibit only) |
| V | Prometheus Logging Framework | 2021 | Command Injection |
| VI | Palmetto Digital Assistant | 1997 | Narrative |
| VII | Laminar Optical Disc | 1992 | Narrative |
| VIII | Cascade Interactive Plugin Runtime | 1996 | XSS |
| IX | Aurelian Data Vault | 2012 | File Upload |
| X | Helios Object Prototype Library | 2019 | Prototype Pollution |
| XI | Heliograph Presence Network | 2003 | SSRF |
| XII | Aether Wireless Encryption Standard | 1999 | Brute Force |
| XIII | Meridian Trading Relay v8 | 2018 | Mass Assignment |

Each artifact page has three rendering states depending on role:
- **Visitor** — origin story, security lesson placard, read-only comments
- **Researcher** — adds technical notes panel, comment form
- **Curator** — adds module override notice if a vuln module is active

---

## Documentation

Full reference documents are in `/docs`:

| Document | What it covers |
|---|---|
| `CLAUDE.md` | Instructions for Claude Code — read this before starting a session |
| `ui_specification.docx` | Every route, every role state, every security surface |
| `data_model.html` | Interactive ERD — all 8 tables with columns and relationships |
| `architecture_sketch.html` | System architecture diagram |
| `nav_map.html` | Screen-to-screen navigation map |
| `state_machine.html` | Submission lifecycle state machine |
| `flow-0*.mmd` | Six Mermaid sequence diagrams, one per major user flow |
| `artifact_collection.docx` | All 13 artifacts with exploit assignments and narrative |
| `field_guide.html` | Plain-language exploit reference (also served at `/field-guide`) |
| `phase1_build_plan.docx` | 15-session build plan with session goals and order |
| `ui_design_deck.pptx` | Visual design reference — every page, every role |

---

## Security Warning

This application contains working exploit surfaces. On the vulnerable
branches, you can trigger SQL injection, stored XSS, command injection,
and SSRF against the local instance. This is the intended use.

Keep it on localhost. Do not put it behind a public IP. Do not use real
credentials anywhere in the app. The seeded accounts in `db/seed.sql`
are for development only.
