# Report template

Fill every section in this order. Keep the checklist table complete (all 23 rows) even when most checks pass.

````markdown
# Attacker Audit: <project name>

<N> checks · <P> PASS · <F> FAIL · <C> CAN'T TELL · <x> CRITICAL · <y> HIGH · <z> LOW
Audited: <date> · Commit: <short sha> · attacker-audit v<version>

## Recon

- Stack: <framework / language / auth / database / ORM / hosting>
- Entry points: <count> (<API routes>, <server actions>, <webhooks>, <cron>, <functions>)
- Access rules live in: <migrations path / rules file / "not in repo">
- Git history: <full / shallow / none>
- Tools used: <gitleaks, npm audit, ... or "grep only">

## Checklist

| # | Check | Verdict | Severity | Location | Note |
|---|---|---|---|---|---|
| 1.1 | Hardcoded keys in source | | | | |
| 1.2 | Committed .env | | | | |
| 1.3 | Keys in git history | | | | |
| 1.4 | Keys in the frontend bundle | | | | |
| 2.1 | Routes with no session check | | | | |
| 2.2 | Permission checks only in the frontend | | | | |
| 2.3 | User IDs trusted from the request | | | | |
| 2.4 | Tokens in localStorage | | | | |
| 2.5 | No email verification or password rules | | | | |
| 3.1 | Tables with RLS off | | | | |
| 3.2 | Policies that let user A read user B's rows | | | | |
| 3.3 | IDs with no ownership check | | | | |
| 3.4 | Updates that save the whole request body | | | | |
| 4.1 | SQL built from strings | | | | |
| 4.2 | User content rendered as raw HTML | | | | |
| 4.3 | No server-side validation | | | | |
| 4.4 | Unchecked file uploads | | | | |
| 5.1 | No rate limits on login or AI routes | | | | |
| 5.2 | Unsigned webhooks | | | | |
| 5.3 | CORS set to * | | | | |
| 5.4 | Open /admin or debug routes | | | | |
| 5.5 | Stack traces in production | | | | |
| 5.6 | Outdated dependencies | | | | |

## Findings

### [SEVERITY] <id> <short title>
Where: `path/to/file.ext:line` (repeat for each location)
Attack: <one line: who does what, and what they get>
Fix:
```<lang>
<exact change>
```
<If severity differs from the default, one line on why.>

## Can't tell

| # | What to check | Where | A FAIL looks like |
|---|---|---|---|

## Fix order

1. **<id>**: <why it's here>. Effort: <minutes / an hour / half a day>.
2. ...

Which of these should I fix? I haven't changed anything yet.
````
