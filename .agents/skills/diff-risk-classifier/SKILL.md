---
name: diff-risk-classifier
description: Use when reviewing a pull request or git diff to classify merge risk per file and list the verification required before merge. Read-only: it analyzes diffs and never modifies code.
---

# diff-risk-classifier

## Purpose

Classify the merge risk of a diff, file by file, so a reviewer knows exactly
what must be verified before merging. The output is a risk report, not a fix:
this skill ONLY analyzes diffs. It MUST NOT modify code, run migrations,
install dependencies, or change any file.

## When to use

- Reviewing a pull request or a `git diff` before merge.
- Pre-merge check on a release branch.
- Triaging a large diff to decide what needs human review first.

## When NOT to use

- When asked to fix, refactor, or write code (use a coding skill instead).
- As a replacement for running tests, a security audit, or human review.
- On anything that is not a diff (no changed files, no review target).
- To approve a merge on its own: the recommendation is advisory, a human
  still decides.

## Steps

1. **Collect diff.** Get the file list (`git diff main...HEAD --stat` or the
   PR file list) and the actual changes (`git diff main...HEAD` or the PR
   diff). If the diff is unavailable, do not invent file rows: report that
   diff collection failed and recommend human review.
2. **Detect risk triggers.** Check every changed file against the risk
   triggers list below. Flag every trigger found, even on small diffs.
3. **Assign LOW/MEDIUM/HIGH.** Give each file a level from its triggers
   (see Level definitions and Rules). The overall level is the highest
   per-file level.
4. **Define required verification.** For each file, state what must be
   checked before merge, scaled to its level.
5. **Write report.** Output the table, the verification checklist, and the
   merge recommendation in the format below. No code changes, ever.

## Risk triggers

| Trigger | What counts | Minimum level |
| --- | --- | --- |
| auth | Login, session, token, password, OAuth, permission checks | HIGH |
| DB migration | Schema change, migration file, index, seed data change | MEDIUM (HIGH if destructive or irreversible: drop column/table, data rewrite) |
| public API | New/changed/removed endpoint, request/response contract, versioned API | HIGH |
| dependency | Added, removed, or version-bumped dependency, lockfile change | MEDIUM (HIGH on major bump or security-sensitive dependency) |
| secret | Keys, tokens, credentials, secret storage, secret env values | HIGH |
| config | Config files, env handling of non-secret values (ports, paths, flags, timeouts), CLI options | MEDIUM (HIGH if security-relevant: auth behavior, TLS, CORS, permissions, secrets handling) |
| deleted tests | Any removed test file or removed test case | HIGH |
| sensitive areas | Payments, crypto, PII handling, access control, billing | HIGH |
| generated files | Generated code, snapshots, build artifacts committed to the repo | LOW if purely generated and regenerable, MEDIUM if hand-edited |
| user data changes | Any change to how user data is stored, read, exported, or deleted | HIGH |

Files with no trigger are LOW. If information is missing for a file (diff
truncated, binary file, unknown context), mark its level UNKNOWN.

Special diff cases (apply before the trigger table):

- Pure rename or whitespace-only change: LOW, after verifying no content
  changed.
- Lockfile-only change with no manifest change: MEDIUM (unexplained
  dependency change).
- Binary or otherwise unreadable diff: UNKNOWN.

## Level definitions

- **LOW** — No behavior change (docs, comments, typo, pure rename or
  whitespace-only change), or a fully reversible change with no user,
  security, or data impact.
- **MEDIUM** — Behavior change with a contained blast radius, reversible,
  and guarded by tests or review.
- **HIGH** — Irreversible change, security/auth/secret/user-data impact,
  possible data loss, or a change with no guarding test or review.

## Rules

1. **Never modify code.** Analysis only. No edits, no fixes, no commits.
2. **Never claim tests passed without evidence.** Only report tests as
   passed if they were actually run and their output was seen. Otherwise
   write "not verified" and put running them on the checklist.
3. **When in doubt, pick the higher level.** Uncertainty raises risk, never
   lowers it.
4. **Touching auth, secret, or user data means HIGH minimum.** No
   exceptions, even for one-line changes.
5. **Deleted tests mean HIGH minimum** for the deleted test file. The
   covered file gets at least MEDIUM; HIGH if the deleted test was its only
   coverage.
6. **Overall level = highest per-file level.** Any UNKNOWN file forces the
   recommendation "do not merge before human review".
7. **Every file gets a level and a trigger** (use `-` for trigger when a
   file genuinely has none).
8. **A file's level is the highest minimum among all its triggers.** A file
   hitting two triggers takes the higher one, never the average.
9. **Every checked item must cite its evidence** (test command + output,
   or CI run link). A check without evidence counts as not verified.
10. **Files outside the stated PR goal get a notes flag** (possible scope
    creep). Their level still follows their triggers.
11. **Plain config is MEDIUM minimum, not HIGH.** It rises to HIGH only
    when it changes security-relevant behavior (auth, TLS, CORS,
    permissions, or how secrets are handled). Judge by what the value is,
    not where it lives: a port number is config, an API key is secret.

## Output format

### Risk table

| file | trigger | level | required verification | notes |
| --- | --- | --- | --- |
| `path/to/file` | auth | HIGH | Run auth test suite; manual login + session-expiry check | Token expiry logic changed |

### Verification checklist

- [ ] Each HIGH file's required verification is done, with evidence cited
  (test command + output, or CI link)
- [ ] Tests run (or explicitly marked not verified, with a reason)
- [ ] No deleted test left without a replacement or a written justification
- [ ] Migration reviewed for reversibility (if any)
- [ ] Rollback plan stated for HIGH migration/API changes (if any)
- [ ] Secrets changes reviewed by a second person (if any)
- [ ] Files outside the PR goal noted as possible scope creep (if any)

### Merge recommendation

Exactly one of:

- **merge** — all files LOW, nothing unverified.
- **merge after verification** — MEDIUM or higher present, but every item
  on the checklist can be completed before merge.
- **do not merge before human review** — any HIGH, UNKNOWN, deleted test,
  or auth/secret/user-data change that needs a human decision.

## Input/output example

Input (short diff summary):

- Modified: `src/auth/session.ts` (changed token expiry logic)
- Deleted: `src/auth/session.test.ts`
- Modified: `package.json` (bump lodash 4.17.20 → 4.17.21)
- Modified: `README.md` (typo fix)

Output:

| file | trigger | level | required verification | notes |
| --- | --- | --- | --- |
| `src/auth/session.ts` | auth | HIGH | Run auth test suite; manual login + expiry check | Expiry logic changed |
| `src/auth/session.test.ts` | deleted tests | HIGH | Restore test or written justification; re-run suite | Auth coverage removed |
| `package.json` | dependency | MEDIUM | Review changelog; run affected tests | Minor bump, no breaking change |
| `README.md` | - | LOW | None | Typo only |

Verification checklist:

- [ ] Auth test suite run, output seen
- [ ] Deleted auth test restored or justified in writing
- [ ] Manual login + session-expiry check done
- [ ] Dependency changelog reviewed; affected tests run

Merge recommendation: **do not merge before human review** (HIGH files +
deleted auth test).
