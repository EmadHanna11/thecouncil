[DEPLOYMENT-README.md](https://github.com/user-attachments/files/32430223/DEPLOYMENT-README.md)
# The Council — Deployment and Recovery Guide

Backup date: 2026-09-20  
Source commit: `6f56dbb27e9b36dce92fd3b35c7866da9b993554`

This archive contains the complete current source code for The Council. It is a portable source backup, not a universal one-click deployment package. The application currently targets ChatGPT Sites and Cloudflare-compatible runtime services, so another host must provide or replace the database and runtime integrations described below.

## What is included

- The React/Vinext application and bilingual interface
- All server API routes
- The 250-persona library currently committed with the application
- Persona-selection and council-discussion prompts
- Daily bilingual question generation and fallback questions
- Account, session, usage, and discussion-history logic
- Database migrations in `.openai/drizzle/`
- Package manifest and exact dependency lockfile
- Hosting and build configuration

Local dependency folders, build caches, Git history, user data, passwords, API keys, and other secrets are deliberately excluded.

## Runtime requirements

- Node.js 22.13 or newer
- pnpm
- A Cloudflare Workers-compatible runtime, or adaptations for the chosen host
- A D1-compatible SQL database exposed to the application as `COUNCIL_DB`
- HTTPS in production, because the session cookie is marked `Secure`
- A Moonshot/Kimi API key for live AI generation

## Environment variables

Configure these only in the hosting provider's secure server-side environment. Never place API keys in browser code or commit them to source control.

| Variable | Required | Purpose |
| --- | --- | --- |
| `MOONSHOT_API_KEY` | For live AI | Server-side Moonshot/Kimi API credential. Without it, the application uses limited fallback/demo behavior. |
| `KIMI_MODEL` | Optional | Model name. The application defaults to `kimi-k2.6`. |
| `KIMI_THINKING` | Optional | Set to `enabled` only if Kimi thinking mode is intentionally required. It is disabled by default to control latency and cost. |

The database binding name must be `COUNCIL_DB`. On a non-Cloudflare host, replace the `cloudflare:workers` database access in `lib/server-account.ts` and the API routes with that provider's database client.

## Install and build

From the extracted project directory:

```bash
corepack enable
pnpm install --frozen-lockfile
pnpm build
```

The production build should complete successfully and create `dist/server/index.js`. The current development command is:

```bash
pnpm dev
```

The packaged Cloudflare-compatible local start command is:

```bash
pnpm start
```

## Database setup

Create a database and apply the SQL migrations in filename order:

1. `.openai/drizzle/0001_personal_workspace.sql`
2. `.openai/drizzle/0002_owner_premium.sql`
3. `.openai/drizzle/0003_remove_auth_test.sql`
4. `.openai/drizzle/0004_daily_questions.sql`

Do not skip or reorder migrations. The database stores user accounts, hashed credentials, sessions, usage events, saved discussions, and one shared generated question batch per day.

The backup contains schema and application code only. It does not contain the live production database or existing user records. Export the live database separately if historical accounts and discussions must be migrated.

## Authentication and plans

Authentication is implemented inside the application using the database, PBKDF2 password hashing, and a 30-day secure HTTP-only session cookie. Free, paid, and premium usage behavior is defined in `lib/server-account.ts`.

Before commercial deployment, review:

- The owner/premium account rule
- Free and paid discussion limits
- Password-reset and email-verification requirements
- Privacy policy and account-deletion process
- Rate limiting, abuse controls, monitoring, and backups

If the new platform supplies managed authentication, the internal account/session implementation can be replaced while preserving the interface and discussion APIs.

## Moving to another hosting provider

For a Cloudflare-compatible provider, retain the Vinext build and connect the `COUNCIL_DB` binding. For Vercel, Netlify, or a conventional Node host, an engineer will normally need to:

1. Replace `cloudflare:workers` bindings with the host's environment and database client.
2. Migrate the D1 SQL schema to the chosen SQL database.
3. Confirm that all API routes run server-side and can reach `https://api.moonshot.ai/v1`.
4. Configure the three environment variables above.
5. Preserve secure cookie behavior behind HTTPS.
6. Rebuild and test registration, sign-in, session persistence, usage limits, saved discussions, persona selection, daily questions, contributions, and synthesis.

## Cost behavior

The application generates one shared bilingual question batch per Amman calendar day and stores it in the database. Visitors reuse that batch instead of triggering a new question-generation request. The council prompt also keeps stable instructions separate from changing conversation content to improve provider-side prompt caching where supported.

Actual cost still depends on model pricing, discussion length, number of personas, output length, and the provider's caching rules. Configure usage limits and monitor API consumption before opening the service broadly.

## Verification checklist after deployment

- The home page loads correctly in English and Arabic.
- Layout direction changes correctly for Arabic.
- Registration, sign-in, sign-out, and session persistence work.
- User profile and plan information are correct.
- The persona library contains the intended 250 personas.
- Persona preparation returns five relevant selections with a position, tension, and reconsideration condition.
- Daily questions show five questions per category and remain stable until the next Amman day.
- A discussion can start, all personas contribute, and synthesis completes.
- Lists, headings, emphasis, and tables render correctly in messages.
- Usage limits and premium behavior work as intended.
- Saved discussions appear in history.
- No API keys or private data appear in browser responses, logs, or source files.
- Automated database backups and recovery testing are enabled.

## Recommended backup strategy

Keep at least three independent copies:

1. This dated ZIP on a local or external drive.
2. A private GitHub repository.
3. A separate encrypted cloud backup.

Also export the live database on a schedule. Source-code backups alone do not preserve user accounts or discussion history.

