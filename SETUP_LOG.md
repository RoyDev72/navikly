# Navikly — Setup Log

No secrets in this file. Secrets live only in `.env.local` (gitignored) and Vercel env settings.

## Step 0 — Preflight (2026-10-09)
- Node v24.11, npm 11.12, git 2.51, openssl 3.5 ✅
- GitHub CLI 2.102 installed via winget, logged in as `RoyDev72` ✅
- Supabase connector ✅ (org "Shivam Roy"), Vercel connector ✅ (personal account)

## Step 1 — Private repo
- Cloned `ArnasDon/wacrm` (MIT) → remote `upstream`
- Private repo: https://github.com/RoyDev72/navikly → remote `origin`
- `npm install` ✅

## Step 2 — Supabase
- Project: `navikly`, ref `zentwcjzttnodsqgpezt`, region ap-south-1 (Mumbai), free tier
- URL: https://zentwcjzttnodsqgpezt.supabase.co
- Migrations 001–046 applied via Supabase connector ✅ (46 rows in `supabase_migrations.schema_migrations`)
- 36 public tables, RLS enabled on all ✅
- Storage buckets: `avatars`, `chat-media`, `flow-media` ✅
- `.env.local` written: Supabase URL + anon key, `ENCRYPTION_KEY`, `AUTOMATION_CRON_SECRET`, `NEXT_PUBLIC_SITE_URL`, `NEXT_PUBLIC_APP_LOCALE`
- Advisors (upstream schema, not blocking):
  - Security WARN: 4 functions without fixed `search_path` (trigger helpers); `vector` extension in `public`; SECURITY DEFINER RPCs executable by anon/authenticated (they check `auth.uid()` internally); `automation_pending_executions` RLS on with no policy (service-role only, intended)
  - Performance: 35 unindexed FKs, 5 RLS `auth.uid()` initplan warnings, duplicate permissive SELECT policies — fine at test scale
- Gate B ✅ service_role key in `.env.local` (verified role claim only); `ENCRYPTION_KEY` saved by Shivam
- Gate C ✅ owner account created + email confirmed (role `owner`), `/dashboard` opens

## Step 3 — Signup lock, rebrand, security
- **Signup lock** (`src/middleware.ts`): `ALLOW_PUBLIC_SIGNUP` (default false). `/signup` → `/login` unless `?invite=<token>` peeks as a valid invitation. `/join/<token>` flow unchanged. 4 new middleware tests.
  - Known ceiling: page-level only — Supabase Auth signup API still accepts direct calls with the anon key.
- **Rebrand**: page title, sidebar title, meta description, signup copy, all user-visible "wacrm" in `messages/{en,es,ko,pt}.json`, 2 Meta error explanations. Kept: LICENSE, internal ids (`wacrm_live_` API key prefix, `X-Wacrm-*` webhook headers, storage keys, DB).
  - Logo files to replace (optional): `src/app/icon.tsx` (generated favicon: purple square + chat glyph), sidebar logo in `src/components/layout/sidebar.tsx` (lucide `MessageSquare` icon). No logo images in `public/`.
- **Test fix**: `vitest.config.ts` alias used `URL.pathname` → broke on Windows/paths with spaces; now `fileURLToPath`. 96/96 files, 1146 tests ✅
- **Security pass**
  - Hard-coded secrets: none ✅
  - Webhook `POST /api/whatsapp/webhook`: HMAC `x-hub-signature-256` enforced, fails closed without `META_APP_SECRET` (live: 401 without signature) ✅
  - Cron endpoints: 401 without `x-cron-secret` ✅
  - RLS on all 36 public tables ✅
  - `npm audit --omit=dev`: 6 high, all via `shadcn` CLI (braces/micromatch glob DoS). Only used as a build-time CSS import; fix = major downgrade → not applied.
  - Low (not fixed): SECURITY DEFINER RPCs executable by `anon` (all check `auth.uid()`, trigger fns can't be called as RPC); invite-URL fallback `https://wacrm.tech` is unreachable once `NEXT_PUBLIC_SITE_URL` is set (it is in prod).

## Step 4 — Deploy (Vercel)
- Vercel connector lacked access to team scope `shivam-roys-projects` (403) → used Vercel CLI 63.1 (`vercel login` by Shivam)
- Project `navikly` (prj_A6arUJpfrghfAp3EQD6Dj8Ct8JjF), team `shivam-roys-projects` (Hobby), linked to GitHub `RoyDev72/navikly` (main → production)
- Env (Production + Preview): Supabase URL/anon (config), service role, ENCRYPTION_KEY, AUTOMATION_CRON_SECRET (secret), NEXT_PUBLIC_SITE_URL=https://navikly.vercel.app, NEXT_PUBLIC_APP_LOCALE=en. META_* pending (Gate D)
- `vercel.json`: functions region `bom1` (Mumbai, next to Supabase)
- **Production URL: https://navikly.vercel.app** ✅
- Supabase Auth: Site URL + redirect URLs set by Shivam; production login ✅; no runtime errors in Vercel logs (last 1h)
- Smoke: /login 200 (title "Navikly"), /signup → /login, webhook POST w/o signature 401, cron w/o secret 401

## Step 5 — Cron (Supabase pg_cron + pg_net)
- Extensions `pg_cron`, `pg_net` enabled
- Cron secret stored in Vault as `navikly_cron_secret` (pushed via a temporary service-role-only RPC, then dropped — value never in SQL text)
- Jobs `navikly-automations-cron`, `navikly-flows-cron` every 2 min → `net.http_get` with `x-cron-secret` from Vault
- First run: both **200** ✅

## Step 6 — WhatsApp
- Webhook callback URL: https://navikly.vercel.app/api/whatsapp/webhook
- Verify token (not a high-value secret): `navikly-b84e2d3b0670d2bb012eeee5`
