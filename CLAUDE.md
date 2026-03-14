# CLAUDE.md — Museum of Lost Technologies

> Read this file at the start of every session before writing any code.
> It contains everything needed to work in this codebase without asking
> for context that should already be known.


**Workflow:** Read the secure implementation on `main` → switch to the vulnerable branch → trigger the exploit manually → diff the two branches to understand what changed and why.

When making changes, preserve the secure implementation on `main`. Vulnerable branches should only differ at the specific exploit surface — keep everything else identical to `main` to make diffs meaningful.


---

## What This Project Is

**Museum of Lost Technologies** is an intentionally vulnerable web application
built for offensive security skill development. It is a Node.js / Express /
PostgreSQL app styled as a museum archive cataloguing historical technologies
whose security failures shaped the discipline.

The application has **two modes** on a per-route basis:

- **Secure baseline** — the version that should exist in a well-written app
- **Vulnerable variant** — the version with the exploit surface active

Both branches coexist in the codebase. Feature branches prefixed with
`feature/vuln-*` contain the vulnerable variants. The `main` branch is the
secure baseline. This dual-branch structure is intentional and permanent.
Do not "fix" the vulnerable variants unless explicitly asked.

**Target certifications this project trains toward:** eWPT, then BSCP.
**Primary learning platform:** PortSwigger Web Security Academy.

---

## Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js (ESM — `"type": "module"` in package.json) |
| Framework | Express 5.x |
| Templating | EJS (`.ejs` files in `/views`) |
| Database | PostgreSQL 16 via `pg` (node-postgres) |
| Auth | Session-based — `express-session` + `connect-pg-simple` |
| File handling | `multer` for uploads, ImageMagick `convert` via `child_process.execFile` |
| Password hashing | `bcrypt` (12 rounds) |
| Rate limiting | `express-rate-limit` |
| Logging | `winston` — structured JSON to `/logs` |
| Testing | Vitest |
| Linting | ESLint with `eslint-config-prettier` |
| Formatting | Prettier |
| Process manager | `pm2` in production, `nodemon` in development |

---

## Project Structure

```
museum-of-lost-technologies/
├── CLAUDE.md                  ← you are here
├── package.json               ← "type": "module" — ESM throughout
├── .env                       ← never commit; see .env.example
├── .env.example
├── src/
│   ├── app.js                 ← Express app setup, middleware registration
│   ├── server.js              ← HTTP server entry point (import app.js)
│   ├── config/
│   │   ├── db.js              ← pg Pool instance — import this, never create pools elsewhere
│   │   └── session.js         ← session middleware config
│   ├── middleware/
│   │   ├── auth.js            ← checkAuth, requireAuth, requireRole
│   │   ├── auditLog.js        ← audit event helpers
│   │   └── rateLimit.js       ← rate limiter instances
│   ├── routes/
│   │   ├── index.js           ← GET / home
│   │   ├── artifacts.js       ← GET /artifacts, GET /artifacts/:slug
│   │   ├── search.js          ← GET /search
│   │   ├── auth.js            ← GET/POST /auth/register, /auth/login, /auth/logout
│   │   ├── profile.js         ← GET /profile, GET /profile/:id
│   │   ├── submit.js          ← GET/POST /submit
│   │   ├── comments.js        ← POST /artifacts/:slug/comments
│   │   ├── admin/
│   │   │   ├── queue.js       ← GET /admin/queue, GET/POST /admin/queue/:id
│   │   │   └── restoration.js ← GET /admin/restoration, POST /admin/restoration/process
│   │   └── fieldGuide.js      ← GET /field-guide
│   ├── services/
│   │   ├── artifactService.js
│   │   ├── userService.js
│   │   ├── submissionService.js
│   │   └── restorationService.js  ← execFile lives here
│   ├── utils/
│   │   ├── resolveCondition.js    ← NEVER inline this logic elsewhere
│   │   ├── sanitize.js            ← DOMPurify server-side wrapper
│   │   └── uuid.js                ← UUID validation regex lives here
│   └── views/
│       ├── partials/
│       │   ├── nav.ejs
│       │   ├── footer.ejs
│       │   └── conditionDot.ejs
│       ├── home.ejs
│       ├── browse.ejs
│       ├── artifact-detail.ejs
│       ├── search.ejs
│       ├── auth/
│       │   ├── register.ejs
│       │   ├── login.ejs
│       │   └── reset.ejs
│       ├── profile.ejs
│       ├── submit.ejs
│       ├── admin/
│       │   ├── queue.ejs
│       │   ├── review.ejs
│       │   └── restoration.ejs
│       ├── field-guide.ejs
│       ├── breach/
│       │   ├── xss.ejs
│       │   ├── sqli.ejs
│       │   └── cmdi.ejs
│       └── errors/
│           ├── 403.ejs
│           ├── 404.ejs
│           ├── 500.ejs
│           └── 429.ejs
├── public/
│   ├── css/
│   │   └── main.css           ← hand-written — no Tailwind, no framework
│   └── js/
│       └── breach.js          ← client-side breach overlay animation only
├── db/
│   ├── schema.sql             ← source of truth for DB structure
│   ├── seed.sql               ← 13 artifacts, seeded Curator account
│   └── migrations/            ← numbered migration files only
├── scripts/
│   └── resetDb.js             ← drops and recreates; dev only
├── uploads/                   ← multer destination; gitignored
├── processed/                 ← ImageMagick output; gitignored
└── logs/                      ← winston output; gitignored
```

---

## Code Style

### Modules
- **ESM throughout.** `import`/`export` — never `require()` or `module.exports`.
- Named exports preferred. Default exports only for Express routers.
- Keep imports grouped: Node built-ins → third-party → local. One blank line between groups.

### Naming
- Files and directories: `camelCase.js` for modules, `kebab-case` for routes and views.
- Variables and functions: `camelCase`.
- Constants: `SCREAMING_SNAKE_CASE` only for true compile-time constants (e.g. `MAX_COMMENT_LENGTH`).
- Database columns: `snake_case` in SQL. Map to `camelCase` in JS at the service layer.

### Functions
- Arrow functions for callbacks and utility functions.
- Named function declarations for route handlers and middleware — makes stack traces readable.
- Async/await throughout — no `.then()` chains.
- Always `try/catch` in async route handlers. Pass errors to `next(err)`.

### SQL
- **All queries must use parameterized placeholders (`$1`, `$2`…) on the secure branch.**
- Queries live in service files (`src/services/`), never inline in route handlers.
- No ORM. Raw SQL with `pg`. This is intentional — the app needs legible query strings for teaching.
- Multi-line template literals for queries. Align keywords:

```js
const { rows } = await db.query(
  `SELECT id, name, slug, condition, module_override
     FROM artifacts
    WHERE is_published = true
      AND era = $1
    ORDER BY name ASC
    LIMIT $2 OFFSET $3`,
  [era, limit, offset]
);
```

### Error Handling
- Express error handler is in `src/app.js`. Route errors go to `next(err)`.
- 4xx errors from middleware use `res.status(code).render('errors/NNN.ejs')` directly.
- Never expose stack traces in rendered views. Log them with winston, show generic message to user.
- Audit log every security event using `auditLog.js` helpers. See the list of event types below.

### Audit Event Types (use these exact strings)
```
SEC_ROLE_CHECK_FAILED
SEC_AUTH_FAILED
SEC_RATE_LIMIT_EXCEEDED
SEC_SQLI_PAYLOAD_DETECTED
SEC_XSS_PAYLOAD_DETECTED
SEC_CMD_INJECTION_ATTEMPT
SEC_MASS_ASSIGNMENT_ATTEMPT
SEC_IDOR_ATTEMPT
SEC_SSRF_ATTEMPT
SEC_FILE_UPLOAD_REJECTED
ARTIFACT_CREATED
ARTIFACT_UPDATED
ARTIFACT_DELETED
SUBMISSION_CREATED
SUBMISSION_STATUS_CHANGED
COMMENT_CREATED
COMMENT_DELETED
USER_REGISTERED
USER_LOGIN
USER_LOGOUT
USER_PASSWORD_RESET
```

### Views (EJS)
- Role-scoped rendering: check `res.locals.user.role` in templates. Never send fields to the
  template that shouldn't reach a given role — filter in the service/route, not in the view.
- `technical_notes` is **never** passed to `res.locals` for Visitor-role requests. Enforce in route.
- Use `<%- sanitizedContent %>` (unescaped) only for fields that have been through `sanitize.js`.
  Use `<%= %>` (escaped) everywhere else. Getting this backwards is an XSS vulnerability.

---

## Roles and Access

| Role | How obtained | Access |
|---|---|---|
| `visitor` | Unauthenticated (null session) | Public pages only |
| `researcher` | Self-registration via `/auth/register` | + profile, submit, comment |
| `curator` | Seeded only — never self-registerable | + admin queue, review, restoration lab |

**Curator accounts are seeded, not self-registered.** There is no UI path to create a
curator account. If you find yourself adding one, stop. The seeded Curator credentials
are in `db/seed.sql` — not in `.env`.

`requireRole('curator')` middleware must be on **every** `/admin/*` route. Check the route file
before adding new admin routes.

---

## Database Schema (summary)

Eight tables. Full schema in `db/schema.sql`.

```
users              id, username, email, password_hash, role, is_active, created_at
artifacts          id, name, slug, era, technology_class, origin_story, security_lesson,
                   technical_notes, condition, is_published, module_override, decommission_date
submissions        id, artifact_id, submitted_by, status, claimed_by, created_at, updated_at
                   status ENUM: draft | submitted | under_review | approved |
                                revision_requested | rejected
comments           id, artifact_id, user_id, content_raw, content_sanitized, is_deleted
uploads            id, artifact_id, original_filename, stored_filename, mime_type,
                   scan_status, uploaded_by
artifact_versions  id, artifact_id, changed_by, field_changed, old_value, new_value,
                   change_reason, changed_at
audit_log          id, event_type, user_id, target_id, request_path, ip_address,
                   payload_sample, severity, outcome, created_at
```

**`resolveCondition(artifact, user)`** — always use this utility to determine what
condition to display. Never compute condition display logic inline in routes or views.
It handles `module_override` for Curators and returns `{ displayed, showOverrideNotice }`.

**Submission status transitions** are managed in `submissionService.js`. Do not update
`submissions.status` directly from route handlers — call the service method, which
enforces valid transitions and writes the audit log.

Valid transitions:
```
draft            → submitted         (researcher)
submitted        → under_review      (curator — claim)
under_review     → approved          (curator)
under_review     → revision_requested (curator)
under_review     → rejected          (curator)
revision_requested → draft           (researcher — re-edit)
approved         → [lab processing]  → artifact.condition = 'restored'
```

---

## Commands

```bash
# Development
npm run dev              # nodemon src/server.js — hot reload

# Database
npm run db:reset         # drops and recreates schema + seeds (DEV ONLY)
npm run db:migrate       # runs pending migrations in db/migrations/
npm run db:seed          # seeds artifacts and curator account only

# Test
npm test                 # vitest run
npm run test:watch       # vitest watch
npm run test:coverage    # vitest run --coverage

# Lint / Format
npm run lint             # eslint src/
npm run lint:fix         # eslint src/ --fix
npm run format           # prettier --write src/ views/

# Production
npm run build            # no build step — Node runs src/ directly
npm start                # pm2 start ecosystem.config.js
npm run logs             # pm2 logs museum
```

---

## Environment Variables

All required vars are documented in `.env.example`. Never hard-code any of these.

```
NODE_ENV
PORT
DATABASE_URL            # postgres://user:pass@host:port/dbname
SESSION_SECRET          # min 32 chars, random
UPLOAD_MAX_SIZE_MB
UPLOAD_ALLOWED_MIMES    # comma-separated: image/jpeg,image/png,application/pdf
LOG_LEVEL               # debug | info | warn | error
```

---

## Gotchas and Hard Rules

### The dual-branch exploit architecture
Vulnerable routes are in `feature/vuln-*` branches. The structure is:
- Secure baseline: parameterized queries, `execFile`, UUID validation, sanitized output
- Vulnerable variant: string concatenation, `exec()`, missing validation, raw output

**Do not merge vulnerable branches into main.** Do not "fix" vuln branches without
being asked. When working on a vuln branch, preserve the exploit surface exactly —
that is the point.

### `resolveCondition()` is the only source of truth for condition display
Never compute `showOverrideNotice` or displayed condition inline anywhere else.
This has been deliberately centralised. Inline logic will silently break the Curator
module override behaviour.

### `technical_notes` must never reach Visitor-role responses
This is enforced in the route, not the template. The route handler selects different
fields from the database depending on role. Do not restructure this so the template
decides — that model is one template-render bug away from a data leak.

### `execFile` — never change to `exec`
In `restorationService.js`, the `execFile` with an argument array is the secure baseline.
The corresponding `exec()` with string interpolation is the vulnerable variant, living
in the vuln branch only. If you ever see `exec()` on main, that is a bug.

### UUID validation is an allowlist, not a sanitizer
`src/utils/uuid.js` exports a regex. It must be called before any DB query or shell
invocation that accepts user-controlled IDs. The regex must not be loosened. If a value
does not match, reject it and write a `SEC_CMD_INJECTION_ATTEMPT` audit log entry.

### Comment sanitization is two fields, not one
Comments are stored with both `content_raw` (what the user typed) and `content_sanitized`
(DOMPurify output). The raw field is for audit purposes only and is never rendered.
The sanitized field is rendered with `<%-`. Do not collapse these to a single field.
On the XSS vuln branch, `content_raw` is rendered with `<%-` instead — this is intentional.

### Session store is PostgreSQL, not in-memory
`connect-pg-simple` stores sessions in the `session` table (auto-created). Never swap this
to MemoryStore even in development — the session behaviour under load matters for the
brute-force exploit testing.

### Rate limiter instances
`src/middleware/rateLimit.js` exports named instances: `loginLimiter`, `registerLimiter`,
`apiLimiter`. Apply the correct one per route. Do not create ad-hoc rate limiters in route
files.

### `db/schema.sql` is the source of truth
If schema and migrations disagree, fix the migration. If you alter a table, update
`schema.sql` and write a migration file. Never run `ALTER TABLE` manually against the
dev database without also updating both files.

### Breach screens are triggered, not routed
The breach screen overlays (XSS, SQLi, CMDi) are not standalone pages with their own
routes. They are triggered by exploit detection middleware and rendered as overlays.
Do not add direct routes to `/breach/*`.

### `public/js/breach.js` is client-only
This file drives the character-rain animation on breach screens. It has no server-side
equivalent. Do not `import` it in any Node module.

### `content_sanitized` is stored, not generated at render time
Sanitization happens on write (POST /artifacts/:slug/comments), not at render time.
Do not add sanitization logic to the EJS template or the route's GET handler.

### File upload stored filenames are UUIDs
`multer` is configured with a `filename` function that generates a UUID + original
extension. The `original_filename` is stored for display; `stored_filename` is what
the filesystem uses. Never construct a file path from `original_filename`.

---

## Exploit Map (what is intentionally vulnerable and where)

| Exploit | Branch | Location | Secure fix |
|---|---|---|---|
| SQL Injection | `feature/vuln-sqli` | `GET /search` — string concat in query | Parameterized `$1` |
| XSS (stored) | `feature/vuln-xss` | `POST /comments` — raw stored, raw rendered | `content_sanitized` + `<%-` only |
| Command Injection | `feature/vuln-cmdi` | `POST /admin/restoration/process` | `execFile` + UUID validation |
| IDOR | `feature/vuln-idor` | `GET /profile/:id` — no ownership check | Ownership check against `req.user.id` |
| SSRF | `feature/vuln-ssrf` | `POST /submit` — external URL fetched | Blocklist + allowlist on URL |
| Mass Assignment | `feature/vuln-mass` | `POST /auth/register` — role from body | Hardcode `role = 'researcher'` server-side |
| Prototype Pollution | `feature/vuln-proto` | `POST /submit` — `Object.assign` on metadata | `Object.create(null)` merge |
| Brute Force | `feature/vuln-brute` | `POST /auth/login` — no rate limit | `loginLimiter` middleware |
| Predictable Token | `feature/vuln-token` | `POST /auth/reset-request` — `Date.now()` token | `crypto.randomBytes(32)` |
| Insecure Upload | `feature/vuln-upload` | `POST /submit` — MIME from `Content-Type` only | Read magic bytes + extension check |

---

## Design Reference

The visual design system uses the following palette. CSS variables are defined in
`public/css/main.css` and must be used — no hardcoded hex values in templates or CSS.

```css
--cream:       #f7f4ef;   /* page background */
--cream-deep:  #ede9e1;   /* card background */
--cream-dark:  #e0dbd0;   /* borders, badges */
--ink:         #2a2820;   /* primary text */
--ink-mid:     #5a5548;   /* secondary text */
--ink-faint:   #9a9488;   /* captions, labels */
--ink-ghost:   #c8c3b8;   /* disabled, placeholders */
--slate:       #7a8fa6;   /* Researcher accent */
--slate-pale:  #dce6ed;   /* Researcher panel bg */
--rust:        #b87c5a;   /* exploit/breach accent */
--rust-pale:   #f5ede6;   /* exploit callout bg */
--sage:        #7a9688;   /* success/restored */
--sage-pale:   #e8f0ed;   /* success bg */
--gold:        #a89060;   /* Curator accent */
--gold-pale:   #f2ede0;   /* Curator panel bg */
```

Typography: `'Cormorant Garamond'` for display/headings, `'Jost'` for UI/labels,
`'Courier New'` for code and monospace contexts. Do not introduce other typefaces.

---

## Documentation Produced (reference files)

All in `/docs` or previously generated — ask if you need to see any of these:

- `ui_specification.docx` — full UI spec, all routes, all roles, security surfaces
- `data_model.html` — interactive ERD with all 8 tables
- `architecture_sketch.html` — system architecture diagram
- `flow-0*.mmd` — 6 Mermaid sequence diagrams, one per major user flow
- `nav_map.html` — screen-to-screen navigation map
- `state_machine.html` — submission lifecycle state machine
- `artifact_collection.docx` — all 13 artifacts with exploit assignments
- `field_guide.html` — plain-language exploit reference (deployed as `/field-guide`)
- `ai_governance_v1.0.docx` — AI usage policy for this project
- `phase1_build_plan.docx` — 15-session build plan with session goals and order

---

## Session Start Checklist

Before writing any code in a new session:

1. Confirm which branch you are on (`git branch`)
2. If on a `feature/vuln-*` branch, confirm the exploit surface that branch owns
3. Check the open TODO in the session notes if provided
4. Do not install new dependencies without asking first — the dependency surface is intentionally minimal
5. Run `npm run lint` before declaring any work done
