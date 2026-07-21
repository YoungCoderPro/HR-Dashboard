# HR Compliance Dashboard — Operating Runbook

This is the complete, hardened version of the project. It explains what each file
is, how the weekly loop works, and the one-time GitHub setup that makes the human
review fast and safe. Committing these files once sets everything up.

## Files in this repo

| File | What it is | Who touches it |
| --- | --- | --- |
| `index.html` | The dashboard page. Fetches `data.json` at load. | You, rarely (design changes only) |
| `data.json` | All compliance data. | The routine (via PR), never hand-edited |
| `data.schema.json` | The formal contract for `data.json`'s shape. | You, only if the structure ever changes |
| `.github/workflows/validate-data.yml` | CI that validates every PR before you review it. | Set once |
| `routine-prompt.txt` | The instructions pasted into the Claude routine. | Update when you refine the routine |
| `RUNBOOK.md` | This file. | Reference |

## How the weekly loop works

1. **Monday (automatic):** the Claude routine runs on Anthropic's cloud. It verifies
   the data against official sources, edits `data.json`, and opens a pull request on
   a `claude/` branch. Nothing on the live site changes yet.
2. **CI (automatic, ~1 min):** GitHub Actions runs `validate-data.yml` on the PR. It
   confirms the file is valid JSON, matches the schema exactly (all 51 states, right
   shape, no missing/renamed keys), and passes sanity checks (no absurd wage or
   unemployment values, no duplicate news IDs, no future date). A green check means
   the file is *structurally* safe.
3. **Review (human, ~5 min):** a person opens the PR, sees the green check, and only
   has to judge the *numbers*, not hunt for broken JSON. The PR body lists every
   change with its source, what was removed, what couldn't be verified, and what
   needs payroll sign-off. Spot-check the flagged items against their linked sources.
4. **Merge (human, one click):** merge the PR. GitHub Pages redeploys in ~1 minute.
   The dashboard's trust bar updates to show the new "Last verified" date, version,
   and any review notes.

The site is hosted by **GitHub Pages**, not Claude. If a routine run ever fails, the
site keeps serving last week's verified data — it never goes down.

## Why the merge stays human (and why that's the whole point)

Auto-merging was considered and deliberately rejected. Across the first runs the
routine (correctly) surfaced figures it could not fully verify, removed fabricated
entries, and flagged payroll-affecting changes for sign-off. If PRs merged
themselves, the one week a wrong wage or a bot-blocked-but-stale figure slips
through would flow straight to the live dashboard and, downstream, to payroll. The
CI check guarantees the file is well-formed; only a human can confirm the numbers
are *true*. Keeping a person on the merge is what lets you tell an auditor every
published figure was verified against an official source.

## One-time GitHub setup

### 1. Enable the validation workflow
Commit `.github/workflows/validate-data.yml`. It runs automatically on the next PR.
No secrets or config needed.

### 2. Turn on branch protection for `main`
Settings → Branches (or Rules → Rulesets) → add a rule targeting `main`:
- **Require a pull request before merging** — with **1 approval**.
- **Require status checks to pass** → select **"Validate compliance data"**.
- (Optional) **Require branches to be up to date before merging.**

Now it is impossible for anyone — routine or human — to change the live site without
a PR that passes CI and gets one approval. Your weekly merge click becomes the
enforced, audited sign-off.

### 3. Point the routine at the updated prompt
Open the routine (Claude Desktop → Routines, or claude.ai/code/routines), and replace
its Instructions with the contents of `routine-prompt.txt`.

### 4. Set routine notifications
Turn on the routine's completion notification so you know each Monday when a PR is
waiting. No PR waiting = nothing to do.

## The weekly human ritual (give this to whoever reviews)

When the "routine finished" notification arrives:
1. Open the PR. Confirm the green **Validate compliance data** check.
2. Read the PR body top to bottom. It's organized as CHANGED / REMOVED / COULD NOT
   VERIFY / NEEDS SIGN-OFF.
3. Spot-check the **NEEDS SIGN-OFF** items (wage + PFML changes) against their linked
   official sources. Loop in the ArmHR/payroll compliance owner for these.
4. Skim the **COULD NOT VERIFY** list — decide if any needs manual follow-up.
5. If it looks right, **Merge**. If anything's off, comment or **Close** the PR;
   the live site stays untouched.
6. After merge, open the live URL, hard-refresh, and confirm the trust bar shows
   today's date.

Total time in a normal week: about five minutes.

## What the dashboard now shows the end user

- A **trust bar** under the hero: last-verified date (green dot if fresh, amber if
  the data is more than ~10 days old), data version, and the unemployment period.
- A **Review notes** button (when the latest update flagged anything) that expands
  the reviewer's caveats — so HR staff can see which figures are fully confirmed vs.
  corroborated-only, instead of assuming everything is equally certain.
- A **loading state** while data.json loads, and a **clear error screen** (not a raw
  red banner) if it can't, with plain guidance.
- A standing **disclaimer**: internal reference, verified weekly, not legal advice,
  payroll figures need sign-off.

## If something breaks

- **Live site shows the error screen:** `data.json` failed to load or parse. If it
  just merged, the JSON was likely malformed — but CI should have caught that, so
  check the Actions tab. Revert the merge commit to restore the prior good data.
- **CI check fails on a PR:** the routine changed the shape or produced an insane
  value. Do not merge. Read the CI error, and if the routine keeps doing it, tighten
  the prompt's "keep the EXACT structure" rule and re-run.
- **Trust bar dot is amber:** data is older than ~10 days — a routine run was missed
  or a PR is sitting unreviewed. Check for an open PR.

## Recommended next hardening (optional)

- Move the routine to a **shared service account** so it survives staff changes,
  rather than running under one person's identity.
- Add a **monthly deeper-scan routine** for the slower-moving sections (workers'
  comp, posters, EEOC) that the weekly run leaves alone.
