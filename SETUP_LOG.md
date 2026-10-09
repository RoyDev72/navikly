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
