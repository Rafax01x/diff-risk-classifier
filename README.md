# diff-risk-classifier

Portable skill that classifies the merge risk of a diff — file by file —
before it gets merged. Read-only: it analyzes diffs and never modifies code.

## What it does

Given a pull request diff or `git diff`, it outputs:

- A risk table: `file | trigger | level | required verification | notes`
- A pre-merge verification checklist
- A merge recommendation: `merge` / `merge after verification` /
  `do not merge before human review`

Levels: **LOW** (no behavior change, fully reversible), **MEDIUM**
(contained, reversible, guarded), **HIGH** (irreversible, security,
data-loss, or unguarded). Touching auth, secret, or user data is HIGH
minimum. Plain config is MEDIUM — it rises to HIGH only when it changes
security-relevant behavior. Judge by what the value is, not where it
lives: a port number is config, an API key is secret.

## Layout

```text
.agents/skills/diff-risk-classifier/SKILL.md   # the skill
demo.md                                        # sample input + output
```

## Usage

One-liner:

```text
Use the diff-risk-classifier skill on this diff. Do not modify code.
Output: file/trigger/level/verification table + checklist + merge recommendation.
Touching auth/secret/user data = HIGH minimum.
```

Or collect the diff yourself first:

```text
git diff main...HEAD --stat   # file list
git diff main...HEAD          # actual changes
```

Then run the skill on it. See [demo.md](demo.md) for a worked example,
including a real PR (JetBrains/thinkrail #1047) that motivated the
config-vs-secret tuning.

## License

MIT. See [LICENSE](LICENSE).
