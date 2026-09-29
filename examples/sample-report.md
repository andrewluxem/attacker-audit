# Sample report

A shortened example of what the skill produces, run against a fictional Next.js and Supabase app called Tidepool. Paths, keys, and findings are invented for illustration. Only a few rows and findings are shown.

---

# Attacker Audit: tidepool

23 checks · 14 PASS · 6 FAIL · 3 CAN'T TELL · 3 CRITICAL · 2 HIGH · 1 LOW
Audited: 2026-09-28 · Commit: 4f2a9c1 · attacker-audit v0.1.0

## Recon

- Stack: Next.js 15 app router / TypeScript / Supabase Auth / Supabase Postgres / supabase-js / Vercel
- Entry points: 19 (11 route handlers, 6 server actions, 1 Stripe webhook, 1 cron)
- Access rules live in: `supabase/migrations/`
- Git history: full
- Tools used: gitleaks, npm audit

## Checklist (excerpt)

| # | Check | Verdict | Severity | Location | Note |
|---|---|---|---|---|---|
| 1.3 | Keys in git history | FAIL | CRITICAL | commit `b81e0d2` `.env.local` | OpenAI key, removed from HEAD in a later commit |
| 2.1 | Routes with no session check | FAIL | CRITICAL | `app/actions/notes.ts:12` | server action `deleteNote` |
| 3.1 | Tables with RLS off | FAIL | CRITICAL | `supabase/migrations/0003_shares.sql:1` | `public.shares` |
| 3.4 | Updates that save the whole request body | FAIL | HIGH | `app/api/profile/route.ts:18` | |
| 5.1 | No rate limits on login or AI routes | FAIL | HIGH | `app/api/summarize/route.ts` | no limiter anywhere |
| 5.5 | Stack traces in production | FAIL | LOW | `app/api/summarize/route.ts:41` | returns `err.message` |
| 5.2 | Unsigned webhooks | PASS | | `app/api/stripe/route.ts:9` | `constructEvent` on `req.text()` |
| 5.4 | Open /admin or debug routes | CAN'T TELL | | `app/api/cron/digest/route.ts` | checks `CRON_SECRET`; confirm it's set in Vercel |

## Findings (excerpt)

### [CRITICAL] 3.1 RLS off on public.shares
Where: `supabase/migrations/0003_shares.sql:1`
Attack: anyone with the public anon key (shipped in the bundle by design) runs `GET /rest/v1/shares?select=*` and gets every share link, including private notes.
Fix:
```sql
alter table public.shares enable row level security;
create policy "owners read their shares" on public.shares
  for select using (auth.uid() = owner_id);
```

### [HIGH] 3.4 Profile update saves the whole body
Where: `app/api/profile/route.ts:18`
Attack: a signed-in user sends `{"display_name":"x","plan":"pro"}` and upgrades themselves for free.
Fix:
```ts
const Profile = z.object({ display_name: z.string().max(80), bio: z.string().max(500).optional() });
const data = Profile.parse(await req.json());
await supabase.from('profiles').update(data).eq('id', user.id);
```

## Fix order

1. **1.3**: the key is in public history, and scanners find those within minutes. Rotate it at OpenAI first, then purge history. Effort: minutes.
2. **3.1**: anonymous, one request, whole table. Effort: minutes.
3. **5.1**: the summarize route is a direct billing risk for anyone who finds it. Effort: an hour.
4. **2.1**: needs a signed-in user but deletes anyone's notes. Effort: minutes.
5. **3.4**: self-serve plan upgrades. Effort: an hour.
6. **5.5**: hygiene. Effort: minutes.

Which of these should I fix? I haven't changed anything yet.
