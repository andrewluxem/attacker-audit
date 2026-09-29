---
name: attacker-audit
description: Read-only security audit of a web app codebase from an attacker's view. Runs 23 checks across secrets, auth, database, input, and exposure, gives each PASS / FAIL / CAN'T TELL with file:line evidence, then ranks the fails into a fix order. Use when asked to security-audit, pen-test review, or "hack" a codebase before launch.
license: MIT
metadata:
  version: "0.1.0"
  checks: "23"
---

# Attacker Audit

Audit the codebase like someone trying to hack it. Report first. Do not modify, delete, commit, or rotate anything during the audit, even if a fix is trivial. Fixes happen only after the user reviews the report and says go.

The checks live in [`CHECKS.md`](CHECKS.md). The report format lives in [`report-template.md`](report-template.md). Read both before starting.

## Ground rules

- Every check gets exactly one verdict: PASS, FAIL, or CAN'T TELL.
- PASS requires evidence you actually looked at: a file, a config, or command output. Not finding a problem is only a PASS if you searched for it in the places it would be.
- CAN'T TELL is for things that live outside the repo (database dashboard settings, hosting env config, edge or WAF rate limits, production error settings) or code you could not reach. Say what the user needs to check and where.
- Every FAIL cites `path/to/file.ext:line`, gets a severity (CRITICAL, HIGH, LOW), states the one-line attack, and gives a concrete fix: a code snippet or exact config change, not general advice.
- Never print a full secret. Show the first 4 and last 4 characters and the location.
- Commands you run must be read-only. No installs into the project, no writes, no network calls to the app's production services.
- In a monorepo, audit every deployable app and label each finding with its app.
- If a check does not apply to the stack (for example RLS on a project with no Postgres), mark it PASS with the note "N/A: <reason>" rather than skipping it, so the report always has 23 rows.

## Severity

- **CRITICAL**: exploitable now by an anonymous user, or exposes secrets, other users' data, or money. Examples: live keys in the repo or bundle, RLS off on user tables, IDOR on user records, unsigned payment webhooks.
- **HIGH**: exploitable by a signed-in user or with modest effort. Examples: mass assignment, missing server-side validation, XSS, no rate limit on routes that bill you.
- **LOW**: hardening and hygiene. Examples: weak password rules, verbose errors, stale dependencies with no reachable exploit.

A check's default severity is listed in `CHECKS.md`. Raise or lower it based on what you find, and say why when you do (for example, tokens in localStorage go from LOW to HIGH when an XSS finding also exists).

## Step 1: Recon

Before checking anything, map the attack surface and write it at the top of the report:

- Stack: framework and router style, language, auth provider, database, ORM, hosting.
- Every server entry point: API routes, route handlers, server actions, edge or serverless functions, webhooks, cron endpoints, middleware and its matcher.
- Where schema and access rules live: SQL migrations, ORM schema, security rules files.
- Whether git history is available, and whether a build output exists.
- Which security tools are installed locally (gitleaks, trufflehog, semgrep, osv-scanner). Use them if present; fall back to grep if not.

## Step 2: Run the checks

Work through `CHECKS.md` in order. Each check lists what to look for, how to search, and what counts as a FAIL. Stack-specific notes are marked with the stack name; apply the ones that match recon.

## Step 3: Report

Use `report-template.md`. The order is fixed:

1. Summary line
2. Recon
3. Checklist table, all 23 rows in check order
4. Findings, one block per FAIL, most severe first
5. CAN'T TELL list with what to check and where
6. Fix order (step 4 below)

## Step 4: Rank by likelihood of hurting the user

Severity says how bad a finding is. The fix order says what is most likely to actually hurt, soonest. Rank FAILs by these factors, in order of weight:

1. **Discoverability.** Bots scan public repos and JS bundles for keys within minutes, and anyone with the public anon key can probe open RLS. Exposed admin routes get found by path scanners. These go first.
2. **Direct cost.** Anything that lets a stranger run up AI, email, or SMS bills, or fake payment events.
3. **Other users' data.** IDOR, policy leaks, role escalation via mass assignment.
4. **Effort.** Anonymous beats signed-in beats insider.
5. **Blast radius.** Whole table beats single record.

For each item in the fix order give the finding ID, one sentence on why it sits at that position, and rough effort (minutes, an hour, half a day). Group fixes that belong together.

If any secret leaked, step 1 is always: rotate it at the provider, then remove it from code and history. Removing without rotating fixes nothing, because the old value is already public.

End by asking the user which fixes to apply. Do not start fixing until they answer.
