# Demo: diff-risk-classifier

Skill ini membaca diff (PR atau `git diff`) lalu mengeluarkan tabel risiko
per file, checklist verifikasi, dan rekomendasi merge. Skill tidak mengubah
kode apa pun — analisis saja.

## Cara pakai

```text
Use the diff-risk-classifier skill on this diff. Do not modify code.
Output: file/trigger/level/verification table + checklist + merge recommendation.
Touching auth/secret/user data = HIGH minimum.
```

## Contoh 1: synthetic diff (input)

- Modified: `src/auth/session.ts` (changed token expiry logic)
- Deleted: `src/auth/session.test.ts`
- Modified: `README.md` (typo fix)
- Added: `src/api/v2/export.ts` (new public endpoint)

## Contoh 1: output skill

| file | trigger | level | required verification | notes |
| --- | --- | --- | --- | --- |
| `src/auth/session.ts` | auth | HIGH | Run auth test suite; manual login + session-expiry check | Token expiry logic changed |
| `src/auth/session.test.ts` | deleted tests | HIGH | Restore test or written justification; re-run suite | Auth coverage removed |
| `src/api/v2/export.ts` | public API | HIGH | Review request/response contract + versioning; confirm auth behavior; run API tests | New public endpoint |
| `README.md` | - | LOW | None | Typo only |

Checklist:

- [ ] Auth test suite run, output seen
- [ ] Deleted auth test restored or justified in writing
- [ ] Manual login + session-expiry check done
- [ ] New endpoint contract + versioning reviewed; rollback plan stated

Merge recommendation: **do not merge before human review** (HIGH files +
deleted auth test).

## Contoh 2: PR beneran (JetBrains/thinkrail #1047)

PR: `fix(cli): ignore a blank or out-of-range THINKRAIL_PORT` — 2 file
(`apps/cli/src/args.ts` + `args.test.ts`). PR ini yang menemukan perlunya
tuning config vs secret.

Sebelum tuning (config dan secret digabung → HIGH):

| file | trigger | level | notes |
| --- | --- | --- | --- |
| `apps/cli/src/args.ts` | config/secret | HIGH | Env handling THINKRAIL_PORT |
| `apps/cli/src/args.test.ts` | - | LOW | Test baru ditambahkan |

Sesudah tuning (secret HIGH, config biasa MEDIUM):

| file | trigger | level | notes |
| --- | --- | --- | --- |
| `apps/cli/src/args.ts` | config | MEDIUM | Port env handling, non-secret value |
| `apps/cli/src/args.test.ts` | - | LOW | Test baru ditambahkan |

Overall sesudah tuning: **MEDIUM**, rekomendasi: **merge after
verification**. Aturannya: nilai dari apa valuenya, bukan dari tempatnya —
port itu config, API key itu secret.
