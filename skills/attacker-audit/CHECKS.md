# Checks

23 checks in five groups. Check IDs are stable: new checks get new IDs at the end of their group, and retired checks keep their ID with a note, so reports from different versions can be compared.

Each check lists its default severity, what to look for, how to search, and when it FAILs. Stack notes apply only when recon found that stack.

---

## 1. Secrets

### 1.1 Hardcoded keys in source
Default severity: CRITICAL

Look for: API keys, tokens, passwords, private keys, and connection strings written into source or config files.

Search:
- `gitleaks detect --no-git --redact` or `trufflehog filesystem . --no-update` if installed.
- Otherwise grep for known prefixes and patterns: `sk_live_`, `sk_test_`, `rk_live_`, `sk-`, `sk-ant-`, `sk-proj-`, `AKIA`, `ASIA`, `ghp_`, `gho_`, `github_pat_`, `xox[abpr]-`, `AIza`, `SG\.`, `-----BEGIN .*PRIVATE KEY`, `postgres(ql)?://[^ ]*:[^ ]*@`, `mongodb(\+srv)?://[^ ]*:[^ ]*@`, `service_role`, and assignments like `(api[_-]?key|secret|password|token)\s*[:=]\s*['"][^'"]{8,}`.

FAIL when: a real credential is present. Test keys are still a FAIL (default LOW) because they show where live keys will end up.

### 1.2 Committed .env
Default severity: CRITICAL

Search: `git ls-files | grep -Ei '(^|/)\.env'`. Read `.gitignore` for `.env*` coverage.

FAIL when: any tracked env file holds real values. `.env.example` with placeholders is a PASS. A `.gitignore` that misses `.env.local` or `.env.production` is a LOW FAIL even if nothing is tracked yet.

### 1.3 Keys in git history
Default severity: CRITICAL

Search:
- `gitleaks detect --redact` (scans history) if installed.
- Otherwise `git log --all -p -S '<prefix>'` for each prefix from 1.1, and `git log --all --diff-filter=D --name-only -- '*.env*'` for env files that were deleted.

FAIL when: a secret appears anywhere in history, even if HEAD is clean. Note in the fix that the key must be rotated; rewriting history alone is not enough if the repo was ever pushed.

CAN'T TELL when: the clone is shallow (`git rev-parse --is-shallow-repository` returns true). Tell the user to run the check on a full clone.

### 1.4 Keys in the frontend bundle
Default severity: CRITICAL

Look for: server secrets exposed to the browser through public env prefixes or client-side imports.

Search:
- Env vars with public prefixes that hold secrets: `NEXT_PUBLIC_`, `VITE_`, `REACT_APP_`, `PUBLIC_`, `EXPO_PUBLIC_`, `NUXT_PUBLIC_`, `GATSBY_`.
- Server-only clients or keys imported into client code (files with `'use client'`, components under `src/components`, anything bundled for the browser).
- If a build exists (`.next/static`, `dist`, `build`, `out`), grep it with the 1.1 patterns.

FAIL when: anything other than a key designed to be public reaches the browser. Publishable keys (Stripe `pk_`, Supabase anon/publishable key, Firebase web config) are fine by design, and their safety depends on checks 3.1 and 3.2.

Stack notes:
- **Supabase**: the service_role / secret key bypasses RLS entirely. Any trace of it in client code is CRITICAL.
- **Next.js**: the `server-only` package prevents a module from being imported into client code. Its absence is not a FAIL, but recommend it in fixes.

---

## 2. Auth

### 2.1 Routes with no session check
Default severity: CRITICAL

Look for: server entry points from recon that read or change user data without verifying the session on the server.

Search: list every route handler, API route, server action, and function, then confirm each one verifies identity before doing work. Middleware only counts if its matcher actually covers the route.

FAIL when: a non-public route has no server-side check.

Stack notes:
- **Next.js**: server actions (`'use server'`) are public POST endpoints. Each one needs its own auth check; a check in the page that renders the form does not protect the action.
- **Supabase**: on the server, `supabase.auth.getSession()` reads the cookie without verifying the JWT. Use `getUser()` or `getClaims()` for authorization decisions.
- **Next.js**: middleware-only auth was bypassable in some versions (CVE-2025-29927). Don't rely on middleware as the only layer.

### 2.2 Permission checks only in the frontend
Default severity: HIGH

Look for: role or admin gates that exist in components, route guards, or hidden buttons, with no matching check in the API route, server action, or database policy.

FAIL when: calling the server endpoint directly would succeed for a user who can't see the button.

### 2.3 User IDs trusted from the request
Default severity: CRITICAL

Search: `userId`, `user_id`, `ownerId`, `owner_id`, `accountId`, `orgId`, `role`, `isAdmin` read from `body`, `query`, `params`, `searchParams`, `formData`, or headers.

FAIL when: identity or privilege comes from the request instead of the verified session.

### 2.4 Tokens in localStorage
Default severity: LOW (HIGH if any 4.2 XSS FAIL exists)

Search: `localStorage.setItem` or `sessionStorage.setItem` with keys or values related to token, jwt, session, auth, or refresh.

FAIL when: long-lived auth tokens are stored in web storage. Note: several auth SDKs (including Supabase JS by default) use localStorage. That is a known tradeoff; rate it LOW unless XSS is also present, and mention cookie-based SSR helpers as the fix.

### 2.5 No email verification or password rules
Default severity: LOW

Search: auth provider config (`supabase/config.toml` `[auth.email] enable_confirmations` and `[auth] minimum_password_length` / `password_requirements`, NextAuth or Clerk config, custom signup handlers).

FAIL when: signups are not email-verified, or passwords can be shorter than 8 characters with no breach check.

CAN'T TELL when: the setting lives only in the provider's dashboard. Name the dashboard page.

---

## 3. Database

### 3.1 Tables with RLS off
Default severity: CRITICAL

Look for: tables in an API-exposed schema without row level security.

Search: every `create table` in migrations should have a matching `alter table ... enable row level security`.

FAIL when: any table in an exposed schema (Supabase: `public` by default) lacks RLS.

CAN'T TELL when: migrations aren't in the repo. Give the user this query to run:

```sql
select schemaname, tablename from pg_tables
where schemaname = 'public' and rowsecurity = false;
```

Stack notes:
- **Firebase**: the equivalent is `firestore.rules` / `storage.rules`. `allow read, write: if true;` or rules that expired test mode left open are CRITICAL.

### 3.2 Policies that let user A read user B's rows
Default severity: CRITICAL

Look for: policies that are wider than the data they guard.

Search: `using (true)`, policies granted `to anon` or `to public` on user data, `for all` policies, `select` policies missing an `auth.uid()` comparison, `insert`/`update` policies missing `with check`, and ownership read from a column the user can write.

FAIL when: a signed-in user, or an anonymous one, can read or change rows that belong to someone else.

### 3.3 IDs with no ownership check
Default severity: CRITICAL

Look for: IDOR. Server code that fetches, updates, or deletes by an ID from the request without scoping to the session user.

Search: `findUnique({ where: { id`, `.eq('id', `, `WHERE id =`, `get_object_or_404(`, `findById(` where the ID comes from the request and there's no owner filter.

FAIL when: changing the ID in the request returns or changes another user's record. This matters most where the query runs with a privileged client that bypasses RLS (service role, admin SDK, direct DB connection).

### 3.4 Updates that save the whole request body
Default severity: HIGH

Search: `.update(req.body)`, `.update(body)`, `.insert(body)`, `{...body}` or `{...data}` spread into a write, `Model.create(req.body)`, `Object.assign(record, req.body)`, Rails `permit!`, Django `ModelForm` with `fields = '__all__'`.

FAIL when: a user can set fields they shouldn't, such as `role`, `user_id`, `is_admin`, `credits`, `plan`, or `verified`.

---

## 4. Input

### 4.1 SQL built from strings
Default severity: CRITICAL

Search: template literals or concatenation passed to `query(`, `execute(`, `raw(`, `$queryRawUnsafe`, `$executeRawUnsafe`, `sql.raw(`, `sql.unsafe(`, `knex.raw(` with `${`, Python `cursor.execute(f"` or `% (`, Django `.raw(` or `.extra(`, Rails `where("... #{`. Also Postgres functions that build dynamic SQL with `execute` and string concatenation instead of `format('%I', ...)` / `%L`.

FAIL when: user input can reach SQL text instead of a bound parameter.

### 4.2 User content rendered as raw HTML
Default severity: HIGH

Search: `dangerouslySetInnerHTML`, `v-html`, `{@html`, `innerHTML =`, `outerHTML =`, `insertAdjacentHTML`, `document.write`, markdown renderers with HTML enabled and no sanitizer (look for DOMPurify, sanitize-html, rehype-sanitize), `href={` with user-supplied URLs (allowing `javascript:`), and LLM output rendered as HTML.

FAIL when: content a user or a model controls reaches the DOM as HTML without sanitizing.

### 4.3 No server-side validation
Default severity: HIGH

Search: schema validation at server entry points (zod, valibot, yup, joi, ajv, class-validator, pydantic, marshmallow, Rails strong params).

FAIL when: a server route or action uses request data with no schema check. Client-side validation alone is a FAIL.

### 4.4 Unchecked file uploads
Default severity: HIGH

Look for: upload handlers and storage bucket config.

FAIL when any of these is true:
- No size limit or no type allowlist on the server.
- Type is trusted from the client's `Content-Type` or file extension alone.
- User-controlled filenames or paths are used in storage keys or disk paths (path traversal).
- Private files go into a public bucket.
- SVG or HTML uploads are served from the app's own origin (stored XSS).

---

## 5. Exposure

### 5.1 No rate limits on login or AI routes
Default severity: HIGH

Search: `@upstash/ratelimit`, `express-rate-limit`, `rate-limiter-flexible`, `slowapi`, `django-ratelimit`, `rack-attack`, custom counters in middleware, provider-side limits in config.

FAIL when: login, signup, password reset, OTP, or any route that calls a paid API (LLM, email, SMS) has no limit. Paid-API routes without limits are a direct billing risk: HIGH at minimum, CRITICAL if reachable without auth.

CAN'T TELL when: limits might exist at the edge (hosting firewall, WAF, API gateway) and aren't visible in the repo. Name where to check.

### 5.2 Unsigned webhooks
Default severity: CRITICAL for payment webhooks, HIGH otherwise

Look for: every webhook handler found in recon.

FAIL when: the handler acts on the payload without verifying the signature against the raw body with a timing-safe compare.

Stack notes:
- **Stripe**: `stripe.webhooks.constructEvent(rawBody, sig, secret)`. In the Next.js app router, read the body with `await req.text()`; parsing JSON first breaks verification, and developers sometimes "fix" that by skipping it.
- **Shopify**: verify `X-Shopify-Hmac-Sha256`.
- **GitHub**: verify `X-Hub-Signature-256`.
- **Clerk, Resend, and other Svix-based senders**: `svix` `Webhook.verify`.

### 5.3 CORS set to *
Default severity: HIGH

Search: `Access-Control-Allow-Origin` headers in code and hosting config (`vercel.json`, `next.config.*`, `netlify.toml`, `_headers`), `cors()` with no options, `origin: true`, `origin: '*'`, origin reflection (`res.setHeader('Access-Control-Allow-Origin', req.headers.origin)`).

FAIL when: authenticated or private endpoints allow any origin, and especially when combined with `Access-Control-Allow-Credentials: true`. A wildcard on a truly public, read-only endpoint is a PASS with a note.

### 5.4 Open /admin or debug routes
Default severity: CRITICAL

Search: route paths containing `admin`, `debug`, `test`, `internal`, `seed`, `reset`, `migrate`, `cron`, `health` that return more than a status, GraphQL playground or introspection enabled in production, `productionBrowserSourceMaps: true`, Vite `build.sourcemap: true` for public deploys, Swagger/OpenAPI UIs without auth.

FAIL when: any of these is reachable in production without auth or a secret. Cron endpoints must check a secret header (for example `CRON_SECRET` on Vercel).

### 5.5 Stack traces in production
Default severity: LOW

Search: error handlers that return `err.stack`, `error.message` from database or provider errors, `console.log` of request bodies, headers, or env, `DEBUG = True`, and environment checks that default to development when the env var is missing.

FAIL when: production responses or logs can expose internals, queries, or secrets.

### 5.6 Outdated dependencies
Default severity: set per advisory

Search: `npm audit --omit=dev`, `pnpm audit --prod`, `yarn npm audit`, `pip-audit`, `bundle audit`, `osv-scanner -r .`. These read the lockfile and query advisory databases; they don't change the project.

FAIL when: a production dependency has a HIGH or CRITICAL advisory with a fix available. List package, installed version, fixed version, and whether the vulnerable code path is reachable if you can tell.

Also check the core framework version against its own security advisories. Recent examples worth checking by name: Next.js CVE-2025-29927 (middleware bypass) and the React Server Components RCE disclosed in December 2025 (CVE-2025-55182). Confirm fixed versions from the advisory rather than from memory.

CAN'T TELL when: there's no lockfile or the audit tool can't reach its advisory source.
