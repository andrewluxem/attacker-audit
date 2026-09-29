# Contributing

Thanks for helping make this better. The skill is only as good as its checks, and the checks get better when people report what they missed or got wrong on real code.

## Before you open anything

**Never include real secrets, private repo code, or production URLs in an issue or PR.** Reduce the problem to a minimal snippet with fake values. If you found a vulnerability in someone else's project using this skill, report it to them, not here.

## Ways to contribute

### Report a false negative or false positive
Use the **Wrong result** issue template. Include:
- the check ID (for example `3.2`)
- a minimal code snippet that reproduces it
- what the skill said and what it should have said
- the stack

These are the highest-value reports. Most improvements start here.

### Add stack notes
Stack notes live under the check they belong to in `CHECKS.md`, in this format:

```markdown
Stack notes:
- **Framework name**: the specific pattern, API, or gotcha, in one or two sentences.
```

Keep them concrete: an API name, a config key, a known footgun. Link official docs in the PR description, not in the file.

### Propose a new check
Open a **New check** issue first. A new check should:
- be something an agent can verify by reading code or config (not by attacking a live system)
- be common in real apps, not a rare edge case
- have a clear FAIL condition and a concrete fix
- not overlap an existing check (extend that one instead)

Once agreed, the new check gets the next ID in its group (for example `2.6`). IDs never get reused or renumbered.

## Rules for editing the skill

- **Read-only stays read-only.** No check may instruct the agent to modify files, install into the project, or send requests to a live app.
- **Stable IDs.** Don't renumber. To retire a check, leave the heading with `(retired in vX.Y: reason)` and remove it from the report template.
- **Keep the three files in sync.** A new or retired check touches `CHECKS.md`, the table in `report-template.md`, the check count in `SKILL.md` frontmatter and body, and the table in `README.md`.
- **Default severity follows the definitions in `SKILL.md`.** If you think a default is wrong, argue it in the PR.
- **Version-specific facts rot.** When citing a CVE or a fixed version, tell the agent to confirm it against the advisory rather than hardcoding a version as the only test.
- **Plain markdown.** No scripts, no remote includes, no instructions to fetch other files at run time.
- **Writing style.** Short sentences, concrete examples, no em dashes.

## Pull requests

1. One change per PR (one new check, one set of stack notes, one fix).
2. Update `CHANGELOG.md` under `Unreleased`.
3. If behavior changes, test it: run the skill on a small repo that exercises the change and paste the relevant report rows (with fake data) into the PR.

## Versioning

Semver on the skill's `metadata.version` in `SKILL.md`:
- **Patch**: wording, stack notes, search patterns
- **Minor**: new checks, severity changes
- **Major**: changes to the report format or verdict model
